# Directus 流程与事件系统核对矩阵

> 本文档为可直接核对的分析报告，所有结论均有源码位置标注

---

## 第一部分：拒绝分支错误传播条件矩阵

### 1.1 Flow 触发器类型错误传播判定表

**源码位置**: `api/src/flows.ts:484-495`

```typescript
// executeFlow 末尾的错误传播逻辑
if (
    (flow.trigger === 'manual' || flow.trigger === 'webhook') &&
    flow.options['async'] !== true &&
    flow.options['error_on_reject'] === true &&
    lastOperationStatus === 'reject'
) {
    throw keyedData[LAST_KEY];
}

if (flow.trigger === 'event' && flow.options['type'] === 'filter' && lastOperationStatus === 'reject') {
    throw keyedData[LAST_KEY];
}
```

| 触发器类型 | 子类型/配置 | 最后操作状态 | 是否抛出错误 | 判定条件 |
|------------|-------------|--------------|--------------|----------|
| **Event** | `type: 'filter'` | `reject` | ✅ **抛出** | `trigger === 'event' && type === 'filter' && status === 'reject'` |
| **Event** | `type: 'filter'` | `resolve` | ❌ 不抛 | 条件不满足 |
| **Event** | `type: 'action'` | `reject` | ❌ 不抛 | 缺少 `type === 'filter'` 条件 |
| **Event** | `type: 'action'` | `resolve` | ❌ 不抛 | 条件不满足 |
| **Manual** | `async: false` + `error_on_reject: true` | `reject` | ✅ **抛出** | `(manual OR webhook) && async !== true && error_on_reject === true && status === 'reject'` |
| **Manual** | `async: true` | 任意 | ❌ 不抛 | 缺少 `async !== true` 条件 |
| **Manual** | `error_on_reject: false` | 任意 | ❌ 不抛 | 缺少 `error_on_reject === true` 条件 |
| **Webhook** | `async: false` + `error_on_reject: true` | `reject` | ✅ **抛出** | 同 Manual |
| **Webhook** | `async: true` | 任意 | ❌ 不抛 | 缺少 `async !== true` 条件 |
| **Operation** | 任意 | 任意 | ❌ 不抛 | 无对应条件分支 |
| **Schedule** | 任意 | 任意 | ❌ 不抛 | try-catch 包裹，仅记录日志 |

### 1.2 Schedule 触发器特殊处理

**源码位置**: `api/src/flows.ts:231-237`

```typescript
const job = scheduleSynchronizedJob(flow.id, flow.options['cron'], async () => {
    try {
        await this.executeFlow(flow);
    } catch (error: any) {
        logger.error(error);  // ⚠️ 错误被捕获，仅记录日志
    }
});
```

**结论**: Schedule 触发器的错误**永远不会传播**，被 try-catch 完全捕获。

### 1.3 扩展 Hook 错误传播（非 Flow）

**源码位置**: `api/src/emitter.ts:34-72`

```typescript
// emitFilter - 同步执行，错误冒泡
public async emitFilter<T>(event, payload, meta, context) {
    for (const { event, listeners } of eventListeners) {
        for (const listener of listeners) {
            const result = await listener(...);  // ⚠️ await，错误会冒泡
            if (result !== undefined) updatedPayload = result;
        }
    }
    return updatedPayload;
}

// emitAction - 异步执行，错误被 catch
public emitAction(event, meta, context) {
    for (const event of events) {
        this.actionEmitter.emitAsync(event, ...).catch((err) => {
            logger.warn(`An error was thrown while executing action "${event}"`);
            logger.warn(err);  // ⚠️ 错误被捕获
        });
    }
}
```

| 扩展 Hook 类型 | 执行方式 | 错误处理 | 是否影响主流程 |
|----------------|----------|----------|----------------|
| **Filter Hook** | 同步 `await` | 错误冒泡 | ✅ **影响**（中断） |
| **Action Hook** | 异步 `emitAsync + .catch()` | 错误被捕获 | ❌ 不影响 |

### 1.4 错误传播条件汇总决策树

```
                    事件触发
                        │
            ┌───────────┴───────────┐
            │                       │
        Flow Trigger          Extension Hook
            │                       │
     ┌──────┴──────┐           ┌────┴────┐
     │             │           │         │
   Event     Other Triggers  Filter   Action
     │             │           │         │
  ┌──┴──┐          │           │         │
Filter Action    Manual    同步await  异步+catch
  │      │      Webhook        │         │
  │      │         │           ▼         ▼
  ▼      │    ┌────┴────┐   抛出错误  错误捕获
抛出错误  │    │         │   (影响)   (不影响)
 (影响)   │    │    ┌────┴────┐
         ▼    │    │         │
      不抛错  │ async  error_on_reject
              │  true    false  true
              │   │       │      │
              │   ▼       ▼      ▼
              │ 不抛错   不抛错  检查 status
              │                     │
              │                 ┌───┴───┐
              │                 │       │
              │              resolve  reject
              │                 │       │
              │                 ▼       ▼
              │              不抛错   抛出错误
              │                        (影响)
              │
         Operation, Schedule
              │
              ▼
           不抛错
```

---

## 第二部分：上下文来源对账

### 2.1 Flow 触发器注册时的上下文传递

**源码位置**: `api/src/flows.ts:196-228`

```typescript
// Event Filter 触发器注册
if (flow.options['type'] === 'filter') {
    const handler: FilterHandler = (payload, meta, context) =>
        this.executeFlow(
            flow,
            { payload, ...meta },
            {
                accountability: context['accountability'],
                database: context['database'],      // ⚠️ 透传来源 context
                getSchema: context['schema'] ? () => context['schema'] : getSchema,
            },
        );
    events.forEach((event) => emitter.onFilter(event, handler));
}

// Event Action 触发器注册
else if (flow.options['type'] === 'action') {
    const handler: ActionHandler = (meta, context) =>
        this.executeFlow(flow, meta, {
            accountability: context['accountability'],
            database: getDatabase(),                // ⚠️ 新建连接！不是透传
            getSchema: context['schema'] ? () => context['schema'] : getSchema,
        });
    events.forEach((event) => emitter.onAction(event, handler));
}
```

| 触发器类型 | `database` 来源 | 说明 |
|------------|-----------------|------|
| **Event Filter** | `context['database']` | ✅ 透传事件触发时的 context |
| **Event Action** | `getDatabase()` | ❌ 新建独立连接 |
| **Manual/Webhook/Operation** | 取决于调用方 | 由 `runWebhookFlow`/`runOperationFlow` 的调用者传入 |

### 2.2 executeOperation 中的上下文覆盖

**源码位置**: `api/src/flows.ts:554-566`

```typescript
try {
    options = applyOptionsData(options, optionData);

    let result = await handler(options, {
        services,
        env: useEnv(),
        database: getDatabase(),        // ⚠️ 默认新建连接
        logger,
        getSchema,
        data: keyedData,
        accountability: null,
        ...context,                      // ⚠️ 但会被传入的 context 覆盖！
    });
```

**关键对账**: `database: getDatabase()` 是默认值，但 `...context` 展开会覆盖它。

| 场景 | 最终 `database` 值 | 原因 |
|------|---------------------|------|
| **Event Filter** | 触发时传入的 `context.database` | 注册时透传，executeFlow 传入，被 `...context` 覆盖 |
| **Event Action** | `getDatabase()` 新连接 | 注册时新建，无覆盖来源 |
| **调用方传入 context.database** | 调用方传入的值 | `...context` 覆盖默认值 |

### 2.3 前置拦截 (Filter) 上下文来源按操作类型

**源码位置**: `api/src/services/items.ts:154-169 (createOne)`, `730-746 (updateMany)`, `1080-1095 (deleteMany)`

```typescript
// createOne - Filter 在事务内
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
    const payloadAfterHooks =
        opts.emitEvents !== false
            ? await emitter.emitFilter(
                    event,
                    payload,
                    meta,
                    {
                        database: trx,           // ✅ 事务连接
                        schema: this.schema,
                        accountability: this.accountability,
                    },
                )
            : payload;
    // ...
});

// updateMany - Filter 在事务外
const payloadAfterHooks =
    opts.emitEvents !== false
        ? await emitter.emitFilter(
                event,
                payload,
                { keys, collection },
                {
                    database: this.knex,     // ⚠️ 非事务连接！
                    schema: this.schema,
                    accountability: this.accountability,
                },
            )
        : payload;
// 之后才开始 transaction
await transaction(this.knex, async (trx) => { /* ... */ });

// deleteMany - Filter 在事务外（同 updateMany）
```

| 操作类型 | Filter 事件位置 | `database` 连接 | 事务可见性 |
|----------|-----------------|-----------------|------------|
| **Create** | ✅ `transaction()` **内部** | `trx` (事务连接) | 可看到同一事务内的变更 |
| **Update** | ❌ `transaction()` **外部** | `this.knex` (非事务) | 独立连接，无事务上下文 |
| **Delete** | ❌ `transaction()` **外部** | `this.knex` (非事务) | 独立连接，无事务上下文 |

### 2.4 后置通知 (Action) 上下文来源

**源码位置**: `api/src/services/items.ts:399-403 (createOne)`, `951-955 (updateMany)`, `1170-1174 (deleteMany)`

```typescript
// 所有操作的 Action 事件 - 统一使用 getDatabase()
if (opts.emitEvents !== false) {
    const actionEvent = {
        event: ...,
        meta: ...,
        context: {
            database: getDatabase(),    // ⚠️ 总是新建独立连接！
            schema: this.schema,
            accountability: this.accountability,
        },
    };
    // ...
}
```

| 操作类型 | Action 事件位置 | `database` 连接 | 事务可见性 |
|----------|-----------------|-----------------|------------|
| **Create** | 事务提交后 | `getDatabase()` 新连接 | 独立连接 |
| **Update** | 事务提交后 | `getDatabase()` 新连接 | 独立连接 |
| **Delete** | 事务提交后 | `getDatabase()` 新连接 | 独立连接 |

### 2.5 上下文传递完整链路

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           Event Filter 触发器链路                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  ItemsService.createOne()                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ transaction(this.knex, async (trx) => {                                      │  │
│  │     emitter.emitFilter(                                                        │  │
│  │         event,                                                                  │  │
│  │         payload,                                                                │  │
│  │         meta,                                                                   │  │
│  │         { database: trx, ... }  ◄────── ① 传入事务连接                        │  │
│  │     )                                                                           │  │
│  │ })                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  Emitter.emitFilter()                                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ for (const listener of listeners) {                                            │  │
│  │     await listener(updatedPayload, { event, ...meta }, context);  ◄── ② 透传 │  │
│  │ }                                                                               │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  FlowManager 注册的 Filter Handler                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ const handler: FilterHandler = (payload, meta, context) =>                    │  │
│  │     this.executeFlow(                                                           │  │
│  │         flow,                                                                    │  │
│  │         { payload, ...meta },                                                    │  │
│  │         {                                                                        │  │
│  │             accountability: context['accountability'],                          │  │
│  │             database: context['database'],  ◄────── ③ 透传 trx                │  │
│  │             getSchema: ...,                                                       │  │
│  │         },                                                                        │  │
│  │     );                                                                            │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  FlowManager.executeFlow()                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ context['flow'] ??= flow;                                                       │  │
│  │ // ... 执行操作节点循环 ...                                                      │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                               │
│                                      ▼                                               │
│  FlowManager.executeOperation()                                                     │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ let result = await handler(options, {                                          │  │
│  │     database: getDatabase(),        // 默认值：新建连接                         │  │
│  │     ...context,                    // ◄────── ④ 覆盖！使用 trx                 │  │
│  │ });                                                                              │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  结论: Event Filter 触发器的操作节点可以通过 context.database 访问原事务连接        │
└─────────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           Event Action 触发器链路                                     │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  FlowManager 注册的 Action Handler                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ const handler: ActionHandler = (meta, context) =>                              │  │
│  │     this.executeFlow(flow, meta, {                                             │  │
│  │         accountability: context['accountability'],                              │  │
│  │         database: getDatabase(),  ◄────── ① 直接新建！不是透传                 │  │
│  │         getSchema: ...,                                                          │  │
│  │     });                                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
│  结论: Event Action 触发器的操作节点永远使用独立连接，不参与原事务                   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 第三部分：事件开关参数分支行为

### 3.1 emitEvents 参数

**源码位置**: `api/src/services/items.ts` (多处)

```typescript
// 统一判断条件: opts.emitEvents !== false
const payloadAfterHooks =
    opts.emitEvents !== false        // ⚠️ 只有明确 false 才跳过
        ? await emitter.emitFilter(...)
        : payload;

// Action 事件同样
if (opts.emitEvents !== false) {
    // emitAction 或 bypassEmitAction
}
```

| `emitEvents` 值 | Filter 事件 | Action 事件 | 说明 |
|-----------------|-------------|-------------|------|
| `undefined` (默认) | ✅ 触发 | ✅ 触发 | `undefined !== false` 为 true |
| `true` | ✅ 触发 | ✅ 触发 | `true !== false` 为 true |
| `false` | ❌ 跳过 | ❌ 跳过 | `false !== false` 为 false |

### 3.2 bypassEmitAction 参数

**源码位置**: `packages/types/src/items.ts:65-69`, `api/src/services/items.ts:406-418`

```typescript
// 类型定义
export type MutationOptions = {
    /**
     * To bypass the emitting of action events if emitEvents is enabled
     * Can be used to queue up the nested events from item service's create, update and delete
     */
    bypassEmitAction?: ((params: ActionEventParams) => void) | undefined;
};

// 使用方式
if (opts.emitEvents !== false) {
    const actionEvent = { event, meta, context };
    
    if (opts.bypassEmitAction) {
        opts.bypassEmitAction(actionEvent);           // 分支1: 回调收集
    } else {
        emitter.emitAction(event, meta, context);      // 分支2: 直接发出
    }
    
    // 嵌套事件同样处理
    for (const nestedActionEvent of nestedActionEvents) {
        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(nestedActionEvent);
        } else {
            emitter.emitAction(...);
        }
    }
}
```

| `bypassEmitAction` 值 | 主事件行为 | 嵌套事件行为 | 说明 |
|------------------------|-----------|--------------|------|
| `undefined` (默认) | `emitter.emitAction()` | `emitter.emitAction()` | 直接异步发出 |
| `(params) => void` 回调 | 调用回调 | 调用回调 | 收集事件，不立即发出 |

### 3.3 bypassEmitAction 只影响 Action，不影响 Filter

**关键对账**:

```typescript
// ⚠️ bypassEmitAction 只在 Action 事件部分检查
// Filter 事件部分完全不检查 bypassEmitAction

// Filter 事件: 没有 bypassEmitAction 检查
const payloadAfterHooks =
    opts.emitEvents !== false
        ? await emitter.emitFilter(...)  // 直接触发，无视 bypassEmitAction
        : payload;

// Action 事件: 检查 bypassEmitAction
if (opts.emitEvents !== false) {
    if (opts.bypassEmitAction) {
        opts.bypassEmitAction(actionEvent);
    } else {
        emitter.emitAction(...);
    }
}
```

| 事件类型 | 受 `bypassEmitAction` 影响 | 说明 |
|----------|---------------------------|------|
| **Filter** | ❌ 不受影响 | 代码中无相关检查 |
| **Action** | ✅ 受影响 | 代码中有显式分支 |

### 3.4 嵌套事件来源

**源码位置**: `api/src/services/payload.ts` (processM2O/A2O/O2M)

```typescript
// PayloadService 处理关系字段时递归调用 ItemsService
async processM2O(data, opts) {
    const nestedActionEvents: ActionEventParams[] = [];
    
    // ... 处理多对一关系 ...
    
    for (const relation of relationsToProcess) {
        const relatedItemsService = new ItemsService(relation.related_collection, {...});
        
        // 递归创建关联项，传递 bypassEmitAction 收集事件
        const key = await relatedItemsService.createOne(nestedData, {
            bypassEmitAction: (params) => nestedActionEvents.push(params),
            // ⚠️ 注意: emitEvents 未设置为 false，所以 Filter 事件仍会触发！
        });
    }
    
    return {
        payload,
        revisions,
        nestedActionEvents,  // 返回收集的事件
        userIntegrityCheckFlags,
    };
}
```

**嵌套事件触发场景**:

| 关系类型 | 处理方法 | 可能触发的嵌套操作 |
|----------|----------|-------------------|
| **M2O** (Many-to-One) | `processM2O` | 创建/更新关联项 |
| **A2O** (Any-to-One) | `processA2O` | 创建/更新关联项 |
| **O2M** (One-to-Many) | `processO2M` | 创建/更新/删除关联项 |

### 3.5 参数组合行为表

| `emitEvents` | `bypassEmitAction` | Filter 事件 | 主 Action 事件 | 嵌套 Action 事件 |
|--------------|---------------------|-------------|----------------|------------------|
| `undefined` | `undefined` | ✅ 触发 | ✅ `emitAction()` | ✅ `emitAction()` |
| `undefined` | `(params) => queue.push(params)` | ✅ 触发 | 📦 收集到回调 | 📦 收集到回调 |
| `true` | `undefined` | ✅ 触发 | ✅ `emitAction()` | ✅ `emitAction()` |
| `true` | `(params) => queue.push(params)` | ✅ 触发 | 📦 收集到回调 | 📦 收集到回调 |
| `false` | 任意 | ❌ 跳过 | ❌ 跳过 | ❌ 跳过 |

---

## 第四部分：同步与异步场景对账

### 4.1 Emitter 层的同步/异步

**源码位置**: `api/src/emitter.ts:34-72`

```typescript
// emitFilter - 完全同步
public async emitFilter<T>(event, payload, meta, context) {
    let updatedPayload = payload;
    
    for (const { event, listeners } of eventListeners) {
        for (const listener of listeners) {
            const result = await listener(updatedPayload, ...);  // ⚠️ await，同步等待
            
            if (result !== undefined) {
                updatedPayload = result;  // 链式修改
            }
        }
    }
    
    return updatedPayload;
}

// emitAction - 完全异步
public emitAction(event, meta, context) {
    for (const event of events) {
        this.actionEmitter.emitAsync(event, ...).catch((err) => {
            // ⚠️ 异步发出，错误被 catch
            logger.warn(`An error was thrown while executing action "${event}"`);
            logger.warn(err);
        });
    }
    // 无 await，立即返回
}
```

| 特性 | `emitFilter` | `emitAction` |
|------|--------------|--------------|
| **执行方式** | 同步 `await` 每个 listener | 异步 `emitAsync`，不等待 |
| **错误处理** | 错误冒泡，中断主流程 | 错误被 `.catch()` 捕获，仅记录日志 |
| **返回值** | 返回修改后的 payload | `void`，无返回值 |
| **监听器顺序** | 按注册顺序执行，前一个返回值影响后一个 | 并发执行，无顺序保证 |

### 4.2 写入链路中的时序

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              createOne 完整时序                                        │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  调用 createOne(data, opts)                                                          │
│           │                                                                          │
│           ▼                                                                          │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ transaction(this.knex, async (trx) => {                                      │  │
│  │                                                                                │  │
│  │  ① emitFilter (同步阻塞，事务内)                                               │  │
│  │  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │  │ await emitter.emitFilter(                                                 │  │
│  │  │     ['items.create', 'collection.items.create'],                         │  │
│  │  │     payload,                                                              │  │
│  │  │     { collection },                                                       │  │
│  │  │     { database: trx, schema, accountability }                            │  │
│  │  │ );                                                                        │  │
│  │  │                                                                           │  │
│  │  │ 内部执行:                                                                 │  │
│  │  │ - 按顺序 await 每个 Filter Listener                                       │  │
│  │  │ - Flow Filter Trigger: executeFlow (同步)                                │  │
│  │  │ - Extension Filter Hook: 直接调用 (同步)                                  │  │
│  │  │ - 任一错误 → 冒泡 → 中断 → 事务回滚                                       │  │
│  │  └────────────────────────────────────────────────────────────────────────┘  │
│  │                                      │                                         │
│  │                                      ▼                                         │
│  │  ② 权限校验、processPayload、实际 INSERT                                       │
│  │                                      │                                         │
│  │                                      ▼                                         │
│  │  ③ 关系处理 (processM2O/A2O/O2M)                                             │
│  │     - 递归调用 ItemsService                                                    │
│  │     - 传递 bypassEmitAction 收集嵌套事件                                      │
│  │     - ⚠️ 嵌套操作的 Filter 事件仍会触发！                                      │
│  │                                      │                                         │
│  │                                      ▼                                         │
│  │  ④ Activity/Revision 记录 (如果开启 accountability)                           │
│  │                                                                                │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│           │ 事务提交                                                                 │
│           ▼                                                                          │
│  ⑤ emitAction (异步非阻塞，事务后)                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ if (opts.emitEvents !== false) {                                               │  │
│  │     if (opts.bypassEmitAction) {                                               │  │
│  │         opts.bypassEmitAction(actionEvent);  // 收集                          │  │
│  │     } else {                                                                    │  │
│  │         emitter.emitAction(event, meta, context);  // 异步发出                 │  │
│  │     }                                                                           │  │
│  │                                                                                 │  │
│  │     // 嵌套事件同样处理                                                         │  │
│  │     for (const nestedActionEvent of nestedActionEvents) {                     │  │
│  │         if (opts.bypassEmitAction) {                                           │  │
│  │             opts.bypassEmitAction(nestedActionEvent);                         │  │
│  │         } else {                                                                │  │
│  │             emitter.emitAction(...);                                            │  │
│  │         }                                                                       │  │
│  │     }                                                                           │  │
│  │ }                                                                                │  │
│  │                                                                                 │  │
│  │ 内部执行:                                                                        │  │
│  │ - emitAsync 异步发出，不等待                                                    │
│  │ - Flow Action Trigger: executeFlow (同步执行，但调用方不 await)                │
│  │ - Extension Action Hook: 直接调用 (同步执行，但被 emitAsync 包裹)             │
│  │ - 错误被 catch，不影响主流程                                                    │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                      │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 updateMany/deleteMany 时序差异

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           updateMany/deleteMany 时序差异                              │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                      │
│  关键差异: Filter 事件在事务外部！                                                   │
│                                                                                      │
│  调用 updateMany(keys, data, opts)                                                  │
│           │                                                                          │
│           ▼                                                                          │
│  ① emitFilter (同步阻塞，⚠️ 事务外)                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────────┐  │
│  │ const payloadAfterHooks =                                                       │  │
│  │     opts.emitEvents !== false                                                   │  │
│  │         ? await emitter.emitFilter(                                             │  │
│  │             event,                                                               │  │
│  │             payload,                                                             │  │
│  │             { keys, collection },                                                │  │
│  │             {                                                                   │  │
│  │                 database: this.knex,     // ⚠️ 非事务连接！                   │  │
│  │                 schema: this.schema,                                             │  │
│  │                 accountability: this.accountability,                            │  │
│  │             },                                                                   │  │
│  │         )                                                                         │  │
│  │         : payload;                                                                │
│  └──────────────────────────────────────────────────────────────────────────────┘  │
│           │                                                                          │
│           ▼ (Filter 错误会中断后续，但事务还没开始)                                  │
│                                                                                      │
│  ② 权限校验 validateAccess                                                           │
│           │                                                                          │
│           ▼                                                                          │
│  ③ transaction(this.knex, async (trx) => {                                         │
│         - 实际 UPDATE/DELETE                                                         │
│         - 关系处理                                                                   │
│         - Activity/Revision 记录                                                     │
│     })                                                                               │
│           │ 事务提交                                                                 │
│           ▼                                                                          │
│  ④ emitAction (异步非阻塞，事务后) ←── 同 createOne                                 │
│                                                                                      │
├─────────────────────────────────────────────────────────────────────────────────────┤
│  关键结论:                                                                            │
│  - createOne 的 Filter 在事务内，错误可回滚                                          │
│  - updateMany/deleteMany 的 Filter 在事务外，错误不影响未开始的事务                 │
│  - 三种操作的 Action 都在事务提交后，使用独立连接                                    │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 同步/异步场景汇总

| 维度 | Filter (前置拦截) | Action (后置通知) |
|------|-------------------|-------------------|
| **执行时机** | 数据操作前 | 数据操作后 (事务提交后) |
| **执行方式** | 同步 `await` | 异步 `emitAsync` |
| **错误影响** | 中断主流程 | 不影响主流程 |
| **Create 事务** | ✅ 事务内 | ❌ 事务外 |
| **Update 事务** | ❌ 事务外 | ❌ 事务外 |
| **Delete 事务** | ❌ 事务外 | ❌ 事务外 |
| **database 连接** | Create: `trx`<br>Update/Delete: `this.knex` | 总是 `getDatabase()` 新连接 |
| **可修改数据** | ✅ 返回值修改 payload | ❌ 无返回值 |

---

## 第五部分：快速核对清单

### 5.1 错误传播快速判断

**问题**: 我的 Flow 在某个操作 reject 了，会中断原操作吗？

**判断步骤**:
1. 确定触发器类型
2. 检查以下条件:

```
如果是 Event 触发器:
    ├── type === 'filter' 且 lastOperationStatus === 'reject'
    │   └── ✅ 会中断
    └── type === 'action'
        └── ❌ 不会中断

如果是 Manual 或 Webhook 触发器:
    ├── async !== true (即同步)
    │   ├── error_on_reject === true
    │   │   └── lastOperationStatus === 'reject'
    │   │       └── ✅ 会中断
    │   └── error_on_reject !== true
    │       └── ❌ 不会中断
    └── async === true (异步)
        └── ❌ 不会中断

如果是 Operation 或 Schedule 触发器:
    └── ❌ 不会中断
```

### 5.2 上下文快速判断

**问题**: 我的 Flow 操作中 `context.database` 是什么连接？

**判断步骤**:
1. 确定触发器类型
2. 检查注册时的传递方式:

```
如果是 Event Filter 触发器:
    └── 透传事件触发时的 context.database
        ├── Create 操作: trx (事务连接)
        ├── Update/Delete 操作: this.knex (非事务)
        └── 其他触发方式: 取决于触发源

如果是 Event Action 触发器:
    └── getDatabase() (总是新建独立连接)

如果是 Manual/Webhook/Operation 触发器:
    └── 取决于调用方传入的 context
```

### 5.3 参数行为快速判断

| 参数 | 值 | 行为 |
|------|-----|------|
| `emitEvents` | `undefined` / `true` | 触发 Filter + Action |
| `emitEvents` | `false` | 跳过所有事件 |
| `bypassEmitAction` | `undefined` | Action 直接 `emitAction()` |
| `bypassEmitAction` | 回调函数 | Action 被收集到回调 |
| `bypassEmitAction` | 任意 | **不影响** Filter 事件 |

---

## 附录：关键源码位置索引

| 功能 | 文件路径 | 行号范围 |
|------|----------|----------|
| Flow 错误传播逻辑 | `api/src/flows.ts` | 484-495 |
| Event Filter 触发器注册 | `api/src/flows.ts` | 196-213 |
| Event Action 触发器注册 | `api/src/flows.ts` | 214-228 |
| Schedule 触发器 try-catch | `api/src/flows.ts` | 231-237 |
| executeOperation 上下文覆盖 | `api/src/flows.ts` | 554-566 |
| Emitter.emitFilter 同步实现 | `api/src/emitter.ts` | 34-60 |
| Emitter.emitAction 异步实现 | `api/src/emitter.ts` | 62-72 |
| createOne Filter 事务内 | `api/src/services/items.ts` | 154-169 |
| createOne Action 事务后 | `api/src/services/items.ts` | 388-419 |
| updateMany Filter 事务外 | `api/src/services/items.ts` | 730-746 |
| deleteMany Filter 事务外 | `api/src/services/items.ts` | 1080-1095 |
| MutationOptions 类型定义 | `packages/types/src/items.ts` | 39-109 |
| bypassEmitAction 注释 | `packages/types/src/items.ts` | 65-69 |
| Extension Hook 注册 | `api/src/extensions/manager.ts` | 892-906 |

---

*报告生成时间: 2026-05-02*
*基于 Directus 源码深度对账分析*
