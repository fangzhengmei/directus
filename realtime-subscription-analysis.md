# Directus 实时订阅机制分析

## 概览

本文档分析 Directus 中客户端订阅 collection 变化后，items 写入、权限过滤、事件 payload 和 WebSocket 客户端的协作流程，以及变更事件和订阅权限裁剪的顺序。

## 核心组件

### 1. Items 服务 (`api/src/services/items.ts`)

负责处理数据的 CRUD 操作，是整个流程的起点。

**关键方法：**
- `createOne()` / `createMany()` - 创建数据
- `updateOne()` / `updateMany()` - 更新数据
- `deleteOne()` / `deleteMany()` - 删除数据
- `readOne()` / `readMany()` / `readByQuery()` - 读取数据

### 2. 权限处理模块

#### 2.1 `processPayload` (`api/src/permissions/modules/process-payload/process-payload.ts`)

处理写入时的权限过滤：
- 验证用户是否有该 collection 的操作权限
- 验证字段级别的权限
- 应用权限预设（presets）
- 执行验证规则（validation rules）

#### 2.2 `validateAccess`

验证用户对特定主键的访问权限，用于更新和删除操作。

#### 2.3 `processAst`

处理读取时的权限过滤，在 `readByQuery()` 中被调用。

### 3. WebSocket 事件钩子 (`api/src/websocket/controllers/hooks.ts`)

注册和处理 WebSocket 相关的事件钩子。

**关键函数：**
- `registerWebSocketEvents()` - 注册各种 action hooks
- `registerAction()` - 包装 emitter.onAction，将事件发布到消息总线

### 4. 订阅处理器 (`api/src/websocket/handlers/subscribe.ts`)

管理 WebSocket 客户端的订阅和事件分发。

**关键组件：**
- `SubscribeHandler` 类
- `subscriptions` - 按 collection 存储订阅
- `dispatch()` - 分发事件到订阅者
- `getPayload()` - 获取事件 payload（关键：这里会重新读取数据）

### 5. Payload 获取工具 (`api/src/websocket/utils/items.ts`)

根据订阅信息和事件获取实际数据。

**关键函数：**
- `getPayload()` - 入口函数，根据 collection 类型分发
- `getItemsPayload()` - 获取普通 items 的数据
- `getCollectionPayload()` - 获取 directus_collections 的数据
- `getFieldsPayload()` - 获取 directus_fields 的数据

---

## 完整流程分析

### 阶段一：Items 写入与权限过滤

**流程：**

```
客户端请求
    ↓
ItemsService.createOne/updateMany/deleteMany
    ↓
[权限过滤 - 使用操作发起者的 accountability]
    ↓
emitFilter (可选的 filter hooks)
    ↓
processPayload / validateAccess (权限验证)
    ↓
数据库写入
    ↓
emitAction (触发 action 事件)
```

**关键代码位置：**

1. **创建操作** (`items.ts:127-426`):
   ```typescript
   // 1. 先执行 filter hooks
   const payloadAfterHooks = await emitter.emitFilter('items.create', payload, ...);
   
   // 2. 权限过滤和预设应用
   const payloadWithPresets = await processPayload({
       accountability: this.accountability,  // 操作发起者的权限
       action: 'create',
       collection: this.collection,
       payload: payloadAfterHooks,
       nested: this.nested,
   }, { knex: trx, schema: this.schema });
   
   // 3. 数据库写入
   await trx.insert(payloadWithoutAliases).into(this.collection)...
   
   // 4. 触发 action 事件
   emitter.emitAction(['items.create', `${this.collection}.items.create`], {
       payload: actionHookPayload,
       key: primaryKey,
       collection: this.collection,
   }, ...);
   ```

2. **更新操作** (`items.ts:709-974`):
   ```typescript
   // 1. filter hooks
   const payloadAfterHooks = await emitter.emitFilter('items.update', payload, ...);
   
   // 2. 验证访问权限
   await validateAccess({
       accountability: this.accountability,
       action: 'update',
       collection: this.collection,
       primaryKeys: keys,
       fields: Object.keys(payloadAfterHooks),
   }, ...);
   
   // 3. processPayload 权限过滤
   const payloadWithPresets = await processPayload(...);
   
   // 4. 数据库更新
   await trx(this.collection).update(payloadWithTypeCasting).whereIn(primaryKeyField, keys);
   
   // 5. 触发 action 事件
   emitter.emitAction(['items.update', `${this.collection}.items.update`], {
       payload: payloadWithPresets,
       keys,
       collection: this.collection,
   }, ...);
   ```

3. **删除操作** (`items.ts:1070-1185`):
   ```typescript
   // 1. filter hooks
   const keysAfterHooks = await emitter.emitFilter('items.delete', keys, ...);
   
   // 2. 验证访问权限
   await validateAccess({
       accountability: this.accountability,
       action: 'delete',
       collection: this.collection,
       primaryKeys: keysAfterHooks,
   }, ...);
   
   // 3. 数据库删除
   await trx(this.collection).whereIn(primaryKeyField, keysAfterHooks).delete();
   
   // 4. 触发 action 事件
   emitter.emitAction(['items.delete', `${this.collection}.items.delete`], {
       payload: keysAfterHooks,
       keys: keysAfterHooks,
       collection: this.collection,
   }, ...);
   ```

**阶段一要点：**
- 权限过滤发生在**写入之前**
- 使用**操作发起者**的 `accountability` 进行权限验证
- 写入完成后才触发 `emitAction` 事件

---

### 阶段二：事件触发与消息总线发布

**流程：**

```
emitter.emitAction (来自 ItemsService)
    ↓
hooks.ts 中注册的 onAction 回调
    ↓
transform(data) 转换为 WebSocketEvent 格式
    ↓
messenger.publish('websocket.event', transformedData)
    ↓
Redis Pub/Sub (或内存消息总线)
```

**关键代码位置：**

`hooks.ts:148-154`:
```typescript
function registerAction(event: string, transform: (args: Record<string, any>) => WebSocketEvent) {
    const messenger = useBus();

    emitter.onAction(event, (data: Record<string, any>) => {
        // push the event through the Redis pub/sub
        messenger.publish('websocket.event', transform(data) as Record<string, any>);
    });
}
```

**事件转换示例：**

`hooks.ts:39-59`:
```typescript
// create 事件转换
registerAction(module + '.create', ({ key, collection, payload = {} }) => ({
    collection,
    action: 'create',
    key,
    payload,  // 注意：这里的 payload 是原始写入的 payload，不是完整数据
}));

// update 事件转换
registerAction(module + '.update', ({ keys, collection, payload = {} }) => ({
    collection,
    action: 'update',
    keys,
    payload,
}));

// delete 事件转换
registerAction(module + '.delete', ({ keys, collection, payload = [] }) => ({
    collection,
    action: 'delete',
    keys,
    payload,
}));
```

**阶段二要点：**
- 事件 payload 只包含**最小信息**（keys/key, collection, action, 原始 payload）
- **不包含完整的数据库记录**
- 通过消息总线发布，支持多实例场景

---

### 阶段三：订阅管理

**流程：**

```
WebSocket 客户端发送 subscribe 消息
    ↓
emitter.emitAction('websocket.message', ...)
    ↓
SubscribeHandler.onMessage() 处理
    ↓
验证 collection 是否存在/可访问
    ↓
sanitizeQuery 处理订阅查询参数
    ↓
调用 getPayload() 获取初始数据（测试权限）
    ↓
this.subscribe(subscription) 注册订阅
    ↓
发送初始响应给客户端
```

**关键代码位置：**

`subscribe.ts:140-199`:
```typescript
async onMessage(client: WebSocketClient, message: WebSocketSubscribeMessage) {
    if (getMessageType(message) === 'subscribe') {
        // 1. 验证 collection
        if (!accountability?.admin && !schema.collections[collection]) {
            throw new WebSocketError('subscribe', 'INVALID_COLLECTION', ...);
        }

        // 2. 构建订阅对象
        const subscription: Subscription = {
            client,
            collection,
        };

        if ('event' in message) subscription.event = message.event as SubscriptionEvent;
        if (message.query) subscription.query = await sanitizeQuery(message.query, schema, accountability);
        if ('item' in message) subscription.item = String(message.item);
        if ('uid' in message) subscription.uid = String(message.uid);

        // 3. 测试获取初始数据（验证权限）
        const data = subscription.event === undefined 
            ? await getPayload(subscription, accountability, schema) 
            : { event: 'init' };

        // 4. 注册订阅
        this.subscribe(subscription);

        // 5. 发送初始响应
        client.send(fmtMessage('subscription', data, subscription.uid));
    }
}
```

**阶段三要点：**
- 订阅时会验证客户端对 collection 的访问权限
- 通过调用 `getPayload()` 测试权限（如果没有指定 event 类型）
- 订阅信息存储在 `subscriptions` 对象中，按 collection 分组

---

### 阶段四：事件分发与订阅权限裁剪

**这是最关键的阶段，权限裁剪发生在这里。**

**流程：**

```
messenger.subscribe('websocket.event', callback) 接收事件
    ↓
SubscribeHandler.dispatch(event)
    ↓
遍历该 collection 的所有订阅
    ↓
[事件类型过滤] 如果订阅指定了 event 类型，不匹配则跳过
    ↓
[item 过滤] 如果订阅指定了 item，不匹配则跳过
    ↓
[关键步骤：权限裁剪] 
调用 getPayload(subscription, client.accountability, schema, event)
    ↓
使用订阅客户端的 accountability 重新读取数据
    ↓
processAst 进行读取权限过滤
    ↓
[结果判断] 如果返回空数组，跳过该订阅
    ↓
发送消息给订阅客户端
```

**关键代码位置：**

1. **事件接收** (`subscribe.ts:31-37`):
   ```typescript
   this.messenger.subscribe('websocket.event', (message: Record<string, any>) => {
       try {
           this.dispatch(message as WebSocketEvent);
       } catch {
           // don't error on an invalid event from the messenger
       }
   });
   ```

2. **事件分发** (`subscribe.ts:108-135`):
   ```typescript
   async dispatch(event: WebSocketEvent) {
       const subscriptions = this.subscriptions[event.collection];
       if (!subscriptions || subscriptions.size === 0) return;
       const schema = await getSchema();

       for (const subscription of subscriptions) {
           const { client } = subscription;

           // 1. 事件类型过滤
           if (subscription.event !== undefined && event.action !== subscription.event) {
               continue;
           }

           // 2. item 过滤
           if ('item' in subscription) {
               if ('keys' in event && !event.keys.includes(subscription.item)) continue;
               if ('key' in event && event.key !== subscription.item) continue;
           }

           // 3. 关键：权限裁剪 - 使用订阅客户端的 accountability
           try {
               const result = await getPayload(subscription, client.accountability, schema, event);

               // 4. 如果权限过滤后没有数据，跳过
               if (Array.isArray(result?.['data']) && result?.['data']?.length === 0) continue;

               // 5. 发送消息
               client.send(fmtMessage('subscription', result, subscription.uid));
           } catch (err) {
               handleWebSocketError(client, err, 'subscribe');
           }
       }
   }
   ```

3. **Payload 获取（权限裁剪的核心）** (`utils/items.ts:19-168`):
   ```typescript
   export async function getPayload(
       subscription: PSubscription,
       accountability: Accountability | null,  // 订阅客户端的 accountability！
       schema: SchemaOverview,
       event?: WebSocketEvent,
   ): Promise<Record<string, any>> {
       // ...
       switch (subscription.collection) {
           // ...
           default:
               result['data'] = await getItemsPayload(subscription, accountability, schema, event);
               break;
       }
       // ...
   }

   export async function getItemsPayload(
       subscription: PSubscription,
       accountability: Accountability | null,
       schema: SchemaOverview,
       event?: WebSocketEvent,
   ) {
       const query = subscription.query ?? {};
       // 使用订阅客户端的 accountability 创建 ItemsService
       const service = getService(subscription.collection, { schema, accountability });

       if ('item' in subscription) {
           if (event?.action === 'delete') {
               return subscription.item;
           } else {
               // 重新读取数据，会经过权限过滤
               return await service.readOne(subscription.item, query);
           }
       }

       switch (event?.action) {
           case 'create':
               // 重新读取数据，会经过权限过滤
               return await service.readMany([event.key], query);
           case 'update':
               // 重新读取数据，会经过权限过滤
               return await service.readMany(event.keys, query);
           case 'delete':
               return event.keys;
           case undefined:
           default:
               return await service.readByQuery(query);
       }
   }
   ```

4. **读取时的权限过滤** (`items.ts:499-580`):
   ```typescript
   async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
       // ...
       let ast = await getAstFromQuery({
           collection: this.collection,
           query: updatedQuery,
           accountability: this.accountability,  // 订阅客户端的 accountability
       }, { schema: this.schema, knex: this.knex });

       // 关键：processAst 进行权限过滤
       ast = await processAst(
           { ast, action: 'read', accountability: this.accountability },
           { knex: this.knex, schema: this.schema },
       );

       const records = await runAst(ast, this.schema, this.accountability, ...);
       // ...
   }
   ```

**阶段四要点：**
- 权限裁剪发生在**事件分发阶段**
- 使用**订阅客户端**的 `accountability`，而不是操作发起者的
- 通过**重新读取数据库**来应用权限过滤
- `getPayload()` 中的 `service.readOne/readMany` 会调用 `processAst` 进行读取权限过滤
- 如果权限过滤后返回空数组，该订阅客户端**不会收到通知**

---

## 变更事件与订阅权限裁剪的顺序

### 完整时间线

```
时间轴：

T0: 操作发起者执行写入请求
     ↓
T1: ItemsService 执行写入前权限过滤 (processPayload/validateAccess)
     ↓ 使用操作发起者的 accountability
T2: 数据库写入完成
     ↓
T3: emitter.emitAction 触发变更事件
     ↓
T4: hooks.ts 接收事件，转换为 WebSocketEvent，发布到消息总线
     ↓
T5: SubscribeHandler 接收消息总线事件，调用 dispatch()
     ↓
T6: 遍历该 collection 的所有订阅
     ↓
T7: 对每个订阅，调用 getPayload()
     ↓ 使用订阅客户端的 accountability
T8: getPayload() 创建 ItemsService，调用 readOne/readMany
     ↓
T9: readOne/readMany → readByQuery → processAst 权限过滤
     ↓
T10: 检查返回数据，如果为空则跳过该订阅
     ↓
T11: 发送 WebSocket 消息给订阅客户端
```

### 关键顺序总结

| 阶段 | 操作 | 使用的 accountability | 权限类型 |
|------|------|----------------------|----------|
| 1 | 写入前权限过滤 | 操作发起者 | 写入权限 (create/update/delete) |
| 2 | 触发变更事件 | - | - |
| 3 | 事件分发到订阅者 | - | - |
| 4 | **重新读取数据** | **订阅客户端** | **读取权限 (read)** |
| 5 | 权限裁剪后的消息发送 | - | - |

### 核心结论

**变更事件先触发，订阅权限裁剪后进行。**

具体来说：
1. **变更事件触发**是在写入完成后立即发生的（T3）
2. **订阅权限裁剪**是在事件分发阶段，针对每个订阅客户端重新读取数据时进行的（T7-T9）

这种设计的优势：
- **写入性能**：写入操作不需要等待所有订阅者的权限检查
- **权限隔离**：每个订阅客户端使用自己的权限上下文
- **数据新鲜**：重新读取确保发送的是最新的完整数据
- **多实例支持**：通过消息总线，事件可以跨实例分发

---

## 关键代码文件索引

| 文件路径 | 功能 |
|---------|------|
| `api/src/services/items.ts` | Items CRUD 服务，写入权限过滤 |
| `api/src/services/websocket.ts` | WebSocket 服务封装 |
| `api/src/permissions/modules/process-payload/process-payload.ts` | 写入时权限过滤 |
| `api/src/permissions/modules/validate-access/validate-access.ts` | 访问权限验证 |
| `api/src/permissions/modules/process-ast/process-ast.ts` | 读取时权限过滤 |
| `api/src/websocket/controllers/base.ts` | WebSocket 控制器基类 |
| `api/src/websocket/controllers/rest.ts` | REST WebSocket 控制器 |
| `api/src/websocket/controllers/hooks.ts` | WebSocket 事件钩子注册 |
| `api/src/websocket/handlers/subscribe.ts` | 订阅管理和事件分发 |
| `api/src/websocket/utils/items.ts` | Payload 获取和权限裁剪 |
| `api/src/websocket/messages.ts` | WebSocket 消息类型定义 |

---

## 数据流示例

假设有两个用户：
- **User A**（操作发起者）：有 `articles`  collection 的写入权限
- **User B**（订阅者）：有 `articles` collection 的读取权限，但只能看到 `status = 'published'` 的文章
- **User C**（订阅者）：没有 `articles` collection 的读取权限

**场景：User A 创建一篇 status = 'draft' 的文章**

```
User A 调用 POST /items/articles
     ↓
[阶段一：写入权限过滤 - User A 的 accountability]
  ✓ User A 有写入权限
     ↓
数据库插入文章 { id: 1, title: 'Test', status: 'draft' }
     ↓
[阶段二：触发事件]
emitter.emitAction('items.create', { key: 1, collection: 'articles', payload: {...} })
     ↓
messenger.publish('websocket.event', { collection: 'articles', action: 'create', key: 1, payload: {...} })
     ↓
[阶段三：事件分发]
SubscribeHandler.dispatch() 遍历 articles 的订阅
     ↓
遍历到 User B 的订阅：
  [阶段四：权限裁剪 - User B 的 accountability]
  getPayload() → ItemsService.readOne(1)
    ↓
  processAst 应用 User B 的权限过滤器 (status = 'published')
    ↓
  查询返回空数组（因为文章是 draft）
    ↓
  跳过，User B 收不到通知
     ↓
遍历到 User C 的订阅：
  [阶段四：权限裁剪 - User C 的 accountability]
  getPayload() → ItemsService.readOne(1)
    ↓
  processAst 发现 User C 没有读取权限
    ↓
  抛出 ForbiddenError
    ↓
  跳过，User C 收不到通知
```

**结果：User B 和 User C 都收不到这篇 draft 文章的创建通知。**

---

---

## 附录 A：权限过滤与错误处理的两条路径

### 关键发现：`readOne` vs `readMany` 的行为差异

在深入分析代码后，发现权限过滤有**两条不同的路径**，取决于订阅方式和事件类型。

#### 1. `readOne` 的行为（抛出异常）

**代码位置**：`api/src/services/items.ts:587-607`

```typescript
async readOne(key: PrimaryKey, query: Query = {}, opts?: QueryOptions): Promise<Item> {
    // ...
    let results: Item[] = [];

    if (query.version && query.version !== 'main') {
        results = [await handleVersion(this, key, queryWithKey, opts)];
    } else {
        results = await this.readByQuery(queryWithKey, opts);
    }

    // 关键：如果结果为空，抛出 ForbiddenError
    if (results.length === 0) {
        throw new ForbiddenError();
    }

    return results[0]!;
}
```

**行为特点**：
- 如果 `readByQuery` 返回空数组（数据被权限过滤掉）
- `readOne` 会**抛出 `ForbiddenError`**
- 无论是"没有权限"还是"数据被过滤"，都会抛出相同的异常

#### 2. `readMany` 的行为（不抛出异常）

**代码位置**：`api/src/services/items.ts:614-629`

```typescript
async readMany(keys: PrimaryKey[], query: Query = {}, opts?: QueryOptions): Promise<Item[]> {
    // ...
    const results = await this.readByQuery(queryWithKey, opts);

    // 关键：直接返回结果，不检查是否为空
    return results;
}
```

**行为特点**：
- 如果 `readByQuery` 返回空数组（数据被权限过滤掉）
- `readMany` 会**直接返回空数组**，不抛出异常
- 调用者需要自行检查结果是否为空

#### 3. `getItemsPayload` 中的调用路径

**代码位置**：`api/src/websocket/utils/items.ts:139-168`

```typescript
export async function getItemsPayload(
    subscription: PSubscription,
    accountability: Accountability | null,
    schema: SchemaOverview,
    event?: WebSocketEvent,
) {
    const query = subscription.query ?? {};
    const service = getService(subscription.collection, { schema, accountability });

    if ('item' in subscription) {
        // 路径 1：订阅特定 item
        if (event?.action === 'delete') {
            return subscription.item;  // delete 事件特殊处理
        } else {
            // create/update 事件 → readOne → 可能抛出 ForbiddenError
            return await service.readOne(subscription.item, query);
        }
    }

    switch (event?.action) {
        case 'create':
            // 路径 2：create 事件 → readMany → 返回空数组不抛异常
            return await service.readMany([event.key], query);
        case 'update':
            // 路径 2：update 事件 → readMany → 返回空数组不抛异常
            return await service.readMany(event.keys, query);
        case 'delete':
            // 路径 3：delete 事件 → 直接返回 keys，不查数据库
            return event.keys;
        case undefined:
        default:
            // 路径 4：初始数据加载 → readByQuery → 返回空数组不抛异常
            return await service.readByQuery(query);
    }
}
```

---

### 路径 A：无数据被过滤（静默跳过）

#### 触发条件

| 订阅方式 | 事件类型 | 调用的方法 | 行为 |
|---------|---------|-----------|------|
| 非特定 item 订阅 | create | `readMany([event.key])` | 返回空数组，不抛异常 |
| 非特定 item 订阅 | update | `readMany(event.keys)` | 返回空数组，不抛异常 |
| 任意订阅 | 初始加载（无事件） | `readByQuery()` | 返回空数组，不抛异常 |

#### 发生的场景

**场景 1：数据被权限过滤器过滤**
- 数据存在于数据库中
- 用户有该 collection 的读取权限
- 但权限条件（如 `status = 'published'`）不匹配该数据
- `processAst` 中的 `injectCases` 注入权限过滤条件
- SQL 查询返回空结果
- `readMany` / `readByQuery` 返回空数组

**场景 2：用户完全没有读取权限**
- `fetchPermissions` 返回空数组
- `validatePathPermissions` 可能抛出异常（取决于具体实现）
- 或者 `injectCases` 注入的条件导致没有结果
- `readMany` / `readByQuery` 返回空数组

#### 处理流程

```
getPayload() 被调用
    ↓
readMany() / readByQuery() 执行
    ↓
processAst 注入权限过滤条件
    ↓
SQL 查询返回空结果
    ↓
readMany() / readByQuery() 返回空数组 []
    ↓
getPayload() 返回 { event: 'create', data: [] }
    ↓
dispatch() 中的检查：
if (Array.isArray(result?.['data']) && result?.['data']?.length === 0) continue;
    ↓
continue 跳过该订阅
    ↓
[结果] 不发送任何消息给客户端
```

**代码位置**：`api/src/websocket/handlers/subscribe.ts:108-135`

```typescript
async dispatch(event: WebSocketEvent) {
    // ...
    for (const subscription of subscriptions) {
        try {
            const result = await getPayload(subscription, client.accountability, schema, event);

            // 路径 A 的关键检查：空数组则跳过
            if (Array.isArray(result?.['data']) && result?.['data']?.length === 0) continue;

            client.send(fmtMessage('subscription', result, subscription.uid));
        } catch (err) {
            // 路径 B：异常处理
            handleWebSocketError(client, err, 'subscribe');
        }
    }
}
```

#### 客户端收到的消息

**无消息** - 客户端完全收不到任何通知。

---

### 路径 B：权限错误（发送错误消息）

#### 触发条件

| 订阅方式 | 事件类型 | 调用的方法 | 行为 |
|---------|---------|-----------|------|
| 特定 item 订阅 (`'item' in subscription`) | create | `readOne(subscription.item)` | 空结果抛出 `ForbiddenError` |
| 特定 item 订阅 (`'item' in subscription`) | update | `readOne(subscription.item)` | 空结果抛出 `ForbiddenError` |

#### 发生的场景

**场景 1：数据不存在**
- 数据在数据库中不存在（可能已被删除）
- `readByQuery` 返回空数组
- `readOne` 检查 `results.length === 0` → 抛出 `ForbiddenError`

**场景 2：用户没有读取权限**
- `fetchPermissions` 返回空数组
- `validatePathPermissions` 抛出异常
- 或者 `injectCases` 注入的条件导致没有结果
- `readOne` 检查 `results.length === 0` → 抛出 `ForbiddenError`

**场景 3：数据被权限过滤器过滤**
- 数据存在，但权限条件不匹配
- `readByQuery` 返回空数组
- `readOne` 检查 `results.length === 0` → 抛出 `ForbiddenError`

**重要**：`readOne` 无法区分这三种场景，都会抛出相同的 `ForbiddenError`。

#### 处理流程

```
getPayload() 被调用
    ↓
readOne() 执行
    ↓
processAst 注入权限过滤条件
    ↓
SQL 查询返回空结果（或抛出异常）
    ↓
readOne() 检查 results.length === 0
    ↓
抛出 ForbiddenError
    ↓
dispatch() 中的 try-catch 捕获异常
    ↓
进入 catch 块：handleWebSocketError(client, err, 'subscribe')
    ↓
ForbiddenError 转换为 WebSocket 错误消息
    ↓
[结果] 发送错误消息给客户端
```

**错误处理代码**：`api/src/websocket/errors.ts:51-71`

```typescript
export function handleWebSocketError(client: WebSocketClient | WebSocket, error: unknown, type?: string): void {
    const logger = useLogger();

    if (isDirectusError(error)) {
        // ForbiddenError 是 DirectusError，走这个分支
        client.send(WebSocketError.fromError(error, type).toMessage());
        return;
    }

    if (error instanceof WebSocketError) {
        client.send(error.toMessage());
        return;
    }
    // ...
}
```

**WebSocketError 格式**：`api/src/websocket/errors.ts:9-48`

```typescript
export class WebSocketError extends Error {
    type: string;
    code: string;
    uid: string | number | undefined;

    toJSON(): WebSocketResponse {
        const message: WebSocketResponse = {
            type: this.type,
            status: 'error',
            error: {
                code: this.code,
                message: this.message,
            },
        };

        if (this.uid !== undefined) {
            message.uid = this.uid;
        }

        return message;
    }
    // ...
}
```

#### 客户端收到的消息

**ForbiddenError 转换后的消息格式**：

```json
{
    "type": "subscribe",
    "status": "error",
    "error": {
        "code": "FORBIDDEN",
        "message": "You don't have permission to access this."
    },
    "uid": "subscription-uid-123"  // 如果订阅有 uid
}
```

**消息字段说明**：

| 字段 | 值 | 说明 |
|-----|---|------|
| `type` | `"subscribe"` | 错误类型，来自 `handleWebSocketError` 的第三个参数 |
| `status` | `"error"` | 固定值，表示这是错误消息 |
| `error.code` | `"FORBIDDEN"` | 来自 `ForbiddenError` 的 `ErrorCode.Forbidden` |
| `error.message` | `"You don't have permission to access this."` | 来自 `ForbiddenError` 的默认消息 |
| `uid` | 订阅的 uid | 只有订阅时指定了 uid 才会有 |

---

### 路径 A vs 路径 B 对比总结

| 维度 | 路径 A：无数据被过滤 | 路径 B：权限错误 |
|-----|---------------------|-----------------|
| **触发条件** | 非特定 item 订阅 + create/update/初始加载 | 特定 item 订阅 + create/update |
| **调用方法** | `readMany()` / `readByQuery()` | `readOne()` |
| **空结果处理** | 返回空数组 `[]` | 抛出 `ForbiddenError` |
| **dispatch 处理** | `if (data.length === 0) continue` | `catch` 块捕获异常 |
| **客户端行为** | 收不到任何消息（静默跳过） | 收到错误消息 |
| **错误消息格式** | 无 | `{ type: "subscribe", status: "error", error: { code: "FORBIDDEN", ... } }` |

---

### 特殊情况：delete 事件

**代码位置**：`api/src/websocket/utils/items.ts:143-145, 159-161`

```typescript
if ('item' in subscription) {
    if (event?.action === 'delete') {
        return subscription.item;  // 直接返回，不查数据库
    }
    // ...
}

switch (event?.action) {
    case 'delete':
        return event.keys;  // 直接返回 keys，不查数据库
    // ...
}
```

**delete 事件的特殊行为**：
- 不经过 `readOne` / `readMany` / `readByQuery`
- 直接返回 `subscription.item` 或 `event.keys`
- **不会触发权限过滤**
- 订阅者会收到 delete 事件，即使他们可能没有权限读取该数据

**delete 事件的消息格式**：

```json
{
    "type": "subscription",
    "event": "delete",
    "data": [1, 2, 3],  // 被删除的主键
    "uid": "subscription-uid-123"
}
```

---

### 完整决策流程图

```
                    事件到达 dispatch()
                           ↓
                    遍历所有订阅
                           ↓
                    事件类型匹配？
                    /           \
                  否             是
                  ↓              ↓
              [跳过]        item 过滤？
                            /         \
                          否           是
                          ↓            ↓
                      [继续]        匹配？
                                    /    \
                                  否      是
                                  ↓       ↓
                              [跳过]   事件类型？
                                      /   |   \
                                create update delete
                                    \   |   /
                                     ↓ ↓ ↓
                               订阅特定 item？
                               /             \
                             是               否
                             ↓                 ↓
                        readOne()        readMany()/readByQuery()
                             ↓                 ↓
                        空结果？           空数组？
                        /      \          /       \
                      是        否       否         是
                      ↓         ↓       ↓           ↓
                ForbiddenError  发送消息  发送消息   continue
                      ↓                              ↓
                catch 块                        [静默跳过]
                      ↓
              handleWebSocketError
                      ↓
                发送错误消息
```

---

## 附录 B：消息格式完整参考

### 1. 正常订阅消息

**Create 事件**：
```json
{
    "type": "subscription",
    "event": "create",
    "data": [
        {
            "id": 1,
            "title": "Test Article",
            "status": "published",
            "created_at": "2026-05-04T10:00:00Z"
        }
    ],
    "uid": "my-subscription-1"
}
```

**Update 事件**：
```json
{
    "type": "subscription",
    "event": "update",
    "data": [
        {
            "id": 1,
            "title": "Updated Title",
            "status": "published"
        }
    ],
    "uid": "my-subscription-1"
}
```

**Delete 事件**：
```json
{
    "type": "subscription",
    "event": "delete",
    "data": [1, 2, 3],
    "uid": "my-subscription-1"
}
```

**初始数据（订阅时未指定 event 类型）**：
```json
{
    "type": "subscription",
    "event": "init",
    "data": [
        { "id": 1, "title": "Article 1" },
        { "id": 2, "title": "Article 2" }
    ],
    "uid": "my-subscription-1"
}
```

### 2. 错误消息

**ForbiddenError（路径 B）**：
```json
{
    "type": "subscribe",
    "status": "error",
    "error": {
        "code": "FORBIDDEN",
        "message": "You don't have permission to access this."
    },
    "uid": "my-subscription-1"
}
```

**其他可能的错误**：

| 错误码 | 场景 | 消息示例 |
|-------|------|---------|
| `FORBIDDEN` | 没有权限访问 | `"You don't have permission to access this."` |
| `INVALID_COLLECTION` | 订阅的 collection 不存在 | `"The provided collection does not exists or is not accessible."` |
| `INVALID_PAYLOAD` | 消息格式错误 | `"Unable to parse the incoming message."` |
| `REQUESTS_EXCEEDED` | 触发限流 | `"Too many messages, retry after 1000ms."` |

---

## 总结

### 核心协作机制

1. **Items 写入**：使用操作发起者的权限进行写入权限过滤（`processPayload` / `validateAccess`）
2. **事件触发**：写入完成后通过 `emitter.emitAction` 触发事件
3. **事件发布**：通过消息总线（Redis Pub/Sub）发布事件
4. **订阅管理**：客户端订阅时验证 collection 访问权限
5. **事件分发**：遍历订阅者，对每个订阅者进行独立处理
6. **权限裁剪**：使用订阅者的权限重新读取数据（`processAst`），实现基于读取权限的消息过滤

### 变更事件与权限裁剪的顺序

**变更事件先触发，订阅权限裁剪后进行。**

- **T1-T3**：写入操作和事件触发（使用操作发起者权限）
- **T4**：事件通过消息总线发布
- **T5-T11**：事件分发和权限裁剪（使用订阅客户端权限）

### 权限过滤的两条关键路径

| 路径 | 触发条件 | 客户端行为 | 消息格式 |
|-----|---------|-----------|---------|
| **路径 A** | 非特定 item 订阅 + create/update/初始加载 | 静默跳过，收不到任何消息 | 无 |
| **路径 B** | 特定 item 订阅 + create/update | 收到错误消息 | `{ type: "subscribe", status: "error", error: { code: "FORBIDDEN" } }` |

### 设计亮点

1. **职责分离**：写入权限和读取权限完全分离
2. **延迟过滤**：权限裁剪延迟到事件分发阶段，不阻塞写入操作
3. **按需读取**：每个订阅客户端独立读取，确保权限隔离
4. **多实例友好**：通过消息总线实现跨实例事件分发
5. **细粒度控制**：支持 collection 级别、item 级别、event 类型级别的过滤

### 潜在的设计权衡

1. **readOne vs readMany 行为不一致**：
   - `readOne` 空结果抛出异常
   - `readMany` 空结果返回空数组
   - 这种不一致可能导致调试困难

2. **delete 事件不进行权限过滤**：
   - 订阅者可能收到他们无权读取的数据的删除通知
   - 这是一个潜在的信息泄露风险

3. **静默跳过 vs 错误消息**：
   - 路径 A 的静默跳过可能让客户端困惑（为什么收不到通知？）
   - 路径 B 的错误消息可能暴露敏感信息（虽然 `ForbiddenError` 消息比较通用）
