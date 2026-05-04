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

### 设计亮点

1. **职责分离**：写入权限和读取权限完全分离
2. **延迟过滤**：权限裁剪延迟到事件分发阶段，不阻塞写入操作
3. **按需读取**：每个订阅客户端独立读取，确保权限隔离
4. **多实例友好**：通过消息总线实现跨实例事件分发
5. **细粒度控制**：支持 collection 级别、item 级别、event 类型级别的过滤
