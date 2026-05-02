# Directus 流程与事件系统嵌入写入链路深度分析

## 1. 核心写入链路概览

### 1.1 事务边界与事件触发时序

Directus 的数据写入操作（Create/Update/Delete）遵循统一的事件触发模式，但在事务边界和上下文传递上存在细微差异。

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         数据写入链路统一时序                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  调用入口: createOne / updateOne / updateMany / deleteOne / deleteMany          │
│                                    │                                              │
│                                    ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 1: emitFilter (前置拦截)                                              │   │
│  │ ┌──────────────────────────────────────────────────────────────────────┐ │   │
│  │ │ 触发条件: opts.emitEvents !== false (默认 undefined 即 true)           │ │   │
│  │ │                                                                      │ │   │
│  │ │ 事务边界:                                                              │ │   │
│  │ │  - createOne: ✅ 在 transaction() 回调内部                              │ │   │
│  │ │  - updateMany: ❌ 在 transaction() 回调外部                              │ │   │
│  │ │  - deleteMany: ❌ 在 transaction() 回调外部                              │ │   │
│  │ │                                                                      │ │   │
│  │ │ 上下文传递 (context.database):                                          │ │   │
│  │ │  - createOne: trx (事务连接)                                           │ │   │
│  │ │  - updateMany: this.knex (非事务)                                       │ │   │
│  │ │  - deleteMany: this.knex (非事务)                                       │ │   │
│  │ └──────────────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                              │
│                                    ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 2: 权限校验与数据处理                                                  │   │
│  │  - validateAccess: 权限校验                                                 │   │
│  │  - processPayload: 数据预处理                                               │   │
│  │  - preMutationError: 前置错误抛出                                           │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                              │
│                                    ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 3: transaction() - 实际数据库操作                                     │   │
│  │ ┌──────────────────────────────────────────────────────────────────────┐ │   │
│  │ │ - 实际 INSERT/UPDATE/DELETE                                            │ │   │
│  │ │ - 关系处理 (M2O/A2O/O2M)                                               │ │   │
│  │ │ - Activity/Revision 记录 (如果开启 accountability)                       │ │   │
│  │ │ - 嵌套事件收集 (nestedActionEvents)                                     │ │   │
│  │ └──────────────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                    │                                              │
│                                    ▼ (事务提交)                                    │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 4: emitAction (后置通知)                                              │   │
│  │ ┌──────────────────────────────────────────────────────────────────────┐ │   │
│  │ │ 触发条件: opts.emitEvents !== false                                     │ │   │
│  │ │                                                                      │ │   │
│  │ │ 执行位置: 事务提交后                                                      │ │   │
│  │ │                                                                      │ │   │
│  │ │ 上下文传递:                                                              │ │   │
│  │ │  - database: getDatabase() (新的独立连接)                                │ │   │
│  │ │  - 不参与原事务                                                          │ │   │
│  │ │                                                                      │ │   │
│  │ │ 参数分支:                                                                │ │   │
│  │ │  - bypassEmitAction 存在: 调用回调收集事件                               │ │   │
│  │ │  - bypassEmitAction 不存在: 直接 emitter.emitAction()                   │ │   │
│  │ │                                                                      │ │   │
│  │ │ 嵌套事件:                                                                │ │   │
│  │ │  - nestedActionEvents 中的事件同步发出或收集                             │ │   │
│  │ └──────────────────────────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 各操作类型的事务边界详解

### 2.1 Create (创建)

**源码位置**: `api/src/services/items.ts:127-426`

```typescript
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    // ... 准备工作 ...

    // ⚠️ 关键点: emitFilter 在事务内部
    const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
        // ===== Filter 事件: 事务内 =====
        const payloadAfterHooks =
            opts.emitEvents !== false
                ? await emitter.emitFilter(
                        this.eventScope === 'items'
                            ? ['items.create', `${this.collection}.items.create`]
                            : `${this.eventScope}.create`,
                        payload,
                        { collection: this.collection },
                        {
                            database: trx,           // ✅ 事务连接
                            schema: this.schema,
                            accountability: this.accountability,
                        },
                    )
                : payload;

        // ... 实际 INSERT ...

        return primaryKey;
    });

    // ===== Action 事件: 事务提交后 =====
    if (opts.emitEvents !== false) {
        const actionEvent = {
            event: ...,
            meta: { payload: actionHookPayload, key: primaryKey, collection: this.collection },
            context: {
                database: getDatabase(),    // ❌ 独立连接
                schema: this.schema,
                accountability: this.accountability,
            },
        };

        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(actionEvent);
        } else {
            emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
        }

        // 嵌套事件同样处理
        for (const nestedActionEvent of nestedActionEvents) {
            if (opts.bypassEmitAction) {
                opts.bypassEmitAction(nestedActionEvent);
            } else {
                emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
            }
        }
    }

    return primaryKey;
}
```

**Create 事务边界总结**:

| 阶段 | 事件类型 | 事务位置 | database 连接 |
|------|----------|----------|---------------|
| 前置拦截 | Filter | ✅ 事务内 | `trx` (事务连接) |
| 后置通知 | Action | ❌ 事务外 | `getDatabase()` (独立) |

---

### 2.2 Update (更新)

**源码位置**: `api/src/services/items.ts:709-974`

```typescript
async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    // ... 准备工作 ...

    // ⚠️ 关键点: emitFilter 在事务外部
    // ===== Filter 事件: 事务外 =====
    const payloadAfterHooks =
        opts.emitEvents !== false
            ? await emitter.emitFilter(
                    this.eventScope === 'items'
                        ? ['items.update', `${this.collection}.items.update`]
                        : `${this.eventScope}.update`,
                    payload,
                    {
                        keys,
                        collection: this.collection,
                    },
                    {
                        database: this.knex,      // ❌ 非事务连接
                        schema: this.schema,
                        accountability: this.accountability,
                    },
                )
            : payload;

    // ... 权限校验 ...

    // ===== 实际操作: 事务内 =====
    await transaction(this.knex, async (trx) => {
        // ... 实际 UPDATE ...
        // ... 关系处理 ...
        // ... 嵌套事件收集 ...
    });

    // ===== Action 事件: 事务提交后 =====
    if (opts.emitEvents !== false) {
        const actionEvent = {
            event: ...,
            meta: { payload: payloadWithPresets, keys, collection: this.collection },
            context: {
                database: getDatabase(),    // ❌ 独立连接
                schema: this.schema,
                accountability: this.accountability,
            },
        };

        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(actionEvent);
        } else {
            emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
        }

        // 嵌套事件处理
        for (const nestedActionEvent of nestedActionEvents) {
            if (opts.bypassEmitAction) {
                opts.bypassEmitAction(nestedActionEvent);
            } else {
                emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
            }
        }
    }

    return keys;
}
```

**Update 事务边界总结**:

| 阶段 | 事件类型 | 事务位置 | database 连接 |
|------|----------|----------|---------------|
| 前置拦截 | Filter | ❌ 事务外 | `this.knex` (非事务) |
| 后置通知 | Action | ❌ 事务外 | `getDatabase()` (独立) |

---

### 2.3 Delete (删除)

**源码位置**: `api/src/services/items.ts:1070-1185`

```typescript
async deleteMany(keys: PrimaryKey[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    // ... 准备工作 ...

    // ⚠️ 关键点: emitFilter 在事务外部
    // ===== Filter 事件: 事务外 =====
    const keysAfterHooks =
        opts.emitEvents !== false
            ? await emitter.emitFilter(
                    this.eventScope === 'items'
                        ? ['items.delete', `${this.collection}.items.delete`]
                        : `${this.eventScope}.delete`,
                    keys,
                    {
                        collection: this.collection,
                    },
                    {
                        database: this.knex,      // ❌ 非事务连接
                        schema: this.schema,
                        accountability: this.accountability,
                    },
                )
            : keys;

    // ... 权限校验 ...

    // ===== 实际操作: 事务内 =====
    await transaction(this.knex, async (trx) => {
        // ... 实际 DELETE ...
        // ... Activity 记录 ...
    });

    // ===== Action 事件: 事务提交后 =====
    if (opts.emitEvents !== false) {
        const actionEvent = {
            event: ...,
            meta: { payload: keysAfterHooks, keys: keysAfterHooks, collection: this.collection },
            context: {
                database: getDatabase(),    // ❌ 独立连接
                schema: this.schema,
                accountability: this.accountability,
            },
        };

        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(actionEvent);
        } else {
            emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
        }
    }

    return keysAfterHooks;
}
```

**Delete 事务边界总结**:

| 阶段 | 事件类型 | 事务位置 | database 连接 |
|------|----------|----------|---------------|
| 前置拦截 | Filter | ❌ 事务外 | `this.knex` (非事务) |
| 后置通知 | Action | ❌ 事务外 | `getDatabase()` (独立) |

---

### 2.4 事务边界对比总表

| 操作类型 | Filter 事件位置 | Filter database 连接 | Action 事件位置 | Action database 连接 |
|----------|-----------------|----------------------|-----------------|----------------------|
| **Create** | ✅ 事务内 | `trx` (事务连接) | 事务提交后 | `getDatabase()` |
| **Update** | ❌ 事务外 | `this.knex` (非事务) | 事务提交后 | `getDatabase()` |
| **Delete** | ❌ 事务外 | `this.knex` (非事务) | 事务提交后 | `getDatabase()` |

**⚠️ 重要差异说明**:

1. **Create 的 Filter 在事务内**:
   - Filter handler 中的数据库操作可以参与原事务
   - Filter 抛出错误会导致事务回滚
   - 这是为了保证数据一致性（生成的主键在事务内）

2. **Update/Delete 的 Filter 在事务外**:
   - Filter handler 中的数据库操作在独立连接
   - Filter 抛出错误不会影响已开启的事务（因为事务还没开始）
   - 注意：权限校验 `validateAccess` 在 Filter 之后、事务之前

---

## 3. 事件开关参数分支行为

### 3.1 MutationOptions 参数总览

**源码位置**: `packages/types/src/items.ts:39-109`

```typescript
export type MutationOptions = {
    /** 禁用事件触发 (Filter + Action) */
    emitEvents?: boolean | undefined;

    /** 绕过 Action 事件直接触发，用于收集嵌套事件 */
    bypassEmitAction?: ((params: ActionEventParams) => void) | undefined;

    /** 绕过批量限制 */
    bypassLimits?: boolean | undefined;

    /** 跳过 accountability/revision 记录 */
    skipTracking?: boolean | undefined;

    /** 覆盖默认值 */
    overwriteDefaults?: DefaultOverwrite | undefined;

    /** 变异追踪器 */
    mutationTracker?: MutationTracker | undefined;

    /** 变异前置错误 */
    preMutationError?: DirectusError<unknown> | undefined;

    /** 用户完整性检查标志 */
    userIntegrityCheckFlags?: UserIntegrityCheckFlag;

    /** 用户完整性检查回调 */
    onRequireUserIntegrityCheck?: ((flags: UserIntegrityCheckFlag) => void) | undefined;

    /** 自动清理缓存 */
    autoPurgeCache?: false | undefined;

    /** 自动清理系统缓存 */
    autoPurgeSystemCache?: false | undefined;

    /** 项创建回调 */
    onItemCreate?: ((collection: string, pk: PrimaryKey) => void) | undefined;

    /** 修订创建回调 */
    onRevisionCreate?: ((pk: PrimaryKey) => void) | undefined;
};
```

### 3.2 emitEvents 参数分支

**参数语义**: `emitEvents !== false` 是判断条件

| emitEvents 值 | Filter 事件 | Action 事件 | 说明 |
|---------------|-------------|-------------|------|
| `undefined` (默认) | ✅ 触发 | ✅ 触发 | 正常行为 |
| `true` | ✅ 触发 | ✅ 触发 | 显式启用 |
| `false` | ❌ 跳过 | ❌ 跳过 | 完全禁用事件 |

**源码实现 (以 createOne 为例)**:

```typescript
// Filter 事件检查
const payloadAfterHooks =
    opts.emitEvents !== false        // ⚠️ 关键: 只有明确 false 才跳过
        ? await emitter.emitFilter(...)
        : payload;

// Action 事件检查
if (opts.emitEvents !== false) {
    // 触发 Action 事件
    if (opts.bypassEmitAction) {
        opts.bypassEmitAction(actionEvent);
    } else {
        emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
    }
    
    // 嵌套事件同样检查
    for (const nestedActionEvent of nestedActionEvents) {
        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(nestedActionEvent);
        } else {
            emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
        }
    }
}
```

**使用场景示例**:

```typescript
// 场景1: 完全禁用事件 (如内部数据迁移)
itemsService.createOne(data, { emitEvents: false });

// 场景2: 自定义事件触发 (如文件上传需要特殊事件)
itemsService.createOne(data, { emitEvents: false });
// 后续手动触发自定义事件
emitter.emitAction('files.upload', { key, payload }, context);
```

---

## 4. bypassEmitAction 与嵌套通知机制

### 4.1 参数设计意图

**源码注释** (`packages/types/src/items.ts:65-69`):

```typescript
/**
 * To bypass the emitting of action events if emitEvents is enabled
 * Can be used to queue up the nested events from item service's create, update and delete
 */
bypassEmitAction?: ((params: ActionEventParams) => void) | undefined;
```

**核心目的**:
- 在批量操作或嵌套关系操作中，统一收集所有事件
- 延迟触发，等待所有操作完成后再一起发送
- 用于需要原子性事件触发的场景

### 4.2 嵌套事件来源

嵌套事件来自关系字段的处理，在 `PayloadService` 中生成：

**源码位置**: `api/src/services/payload.ts` (processM2O, processA2O, processO2M)

```typescript
// 以 processM2O 为例
async processM2O(data: Partial<Item>, opts?: MutationOptions) {
    const nestedActionEvents: ActionEventParams[] = [];
    
    // ... 处理多对一关系 ...
    
    for (const relation of relationsToProcess) {
        // 递归创建/更新关联项时，传递 bypassEmitAction 收集事件
        const relatedItemsService = new ItemsService(relation.related_collection, {
            knex: this.knex,
            schema: this.schema,
            accountability: this.accountability,
        });

        // 场景: 嵌套创建关联项
        const key = await relatedItemsService.createOne(nestedData, {
            bypassEmitAction: (params) => nestedActionEvents.push(params),
            // 注意: 没有设置 emitEvents: false
            // 所以 Filter 事件仍会触发，只有 Action 被收集
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

**嵌套事件触发的关系类型**:

| 关系类型 | 处理方法 | 触发的操作 |
|----------|----------|-----------|
| **M2O** (Many-to-One) | `processM2O` | 创建/更新关联项 |
| **A2O** (Any-to-One) | `processA2O` | 创建/更新关联项 |
| **O2M** (One-to-Many) | `processO2M` | 创建/更新/删除关联项 |

### 4.3 bypassEmitAction 分支行为

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    bypassEmitAction 分支行为                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Action 事件触发逻辑 (createOne/updateMany/deleteMany 中相同):                    │
│                                                                                  │
│  if (opts.emitEvents !== false) {                                               │
│      // 主事件处理                                                                │
│      if (opts.bypassEmitAction) {                                               │
│          // 分支1: 收集事件，不立即发出                                           │
│          opts.bypassEmitAction(actionEvent);                                    │
│      } else {                                                                    │
│          // 分支2: 立即发出事件                                                   │
│          emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
│      }                                                                           │
│                                                                                  │
│      // 嵌套事件同样处理                                                          │
│      for (const nestedActionEvent of nestedActionEvents) {                     │
│          if (opts.bypassEmitAction) {                                           │
│              opts.bypassEmitAction(nestedActionEvent);                          │
│          } else {                                                                │
│              emitter.emitAction(nestedActionEvent.event, ...);                  │
│          }                                                                       │
│      }                                                                           │
│  }                                                                               │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  bypassEmitAction 存在时的行为:                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │ 顶层调用                                                                   │  │
│  │ createOne(data, { bypassEmitAction: (params) => queue.push(params) })   │  │
│  │     │                                                                     │  │
│  │     ▼                                                                     │  │
│  │ processM2O() - 处理关系字段                                               │  │
│  │     │                                                                     │  │
│  │     ▼                                                                     │  │
│  │ 嵌套 createOne(nestedData, {                                             │  │
│  │     bypassEmitAction: (params) => nestedActionEvents.push(params)       │  │
│  │ })                                                                        │  │
│  │     │                                                                     │  │
│  │     ▼                                                                     │  │
│  │ 嵌套 Action 事件被收集到 nestedActionEvents                               │  │
│  │     │                                                                     │  │
│  │     ▼                                                                     │  │
│  │ 返回到顶层 createOne                                                       │  │
│  │     │                                                                     │  │
│  │     ▼                                                                     │  │
│  │ 顶层 Action 事件 + nestedActionEvents 全部通过顶层的 bypassEmitAction 收集 │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 实际使用案例: CollectionsService

**源码位置**: `api/src/services/collections.ts:199-202, 251-258`

```typescript
// 创建集合时，内部创建字段需要收集事件
await fieldItemsService.createMany(sortedFieldPayloads, {
    bypassEmitAction: (params) =>
        opts?.bypassEmitAction 
            ? opts.bypassEmitAction(params)  // 上层提供则透传
            : nestedActionEvents.push(params), // 否则本地收集
    bypassLimits: true,
});

// ... 事务提交后 ...

// 处理收集的嵌套事件
if (opts?.emitEvents !== false && nestedActionEvents.length > 0) {
    const updatedSchema = await getSchema();

    for (const nestedActionEvent of nestedActionEvents) {
        nestedActionEvent.context.schema = updatedSchema;  // 更新 schema
        emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
    }
}
```

### 4.5 注意事项: Filter 事件不受 bypassEmitAction 影响

**关键点**: `bypassEmitAction` 只影响 **Action** 事件，**Filter** 事件仍然正常触发。

```typescript
// 即使设置了 bypassEmitAction，Filter 事件仍会触发
relatedItemsService.createOne(nestedData, {
    bypassEmitAction: (params) => nestedActionEvents.push(params),
    // emitEvents 未设置，默认为 undefined (即 true)
});

// 内部执行:
// 1. emitter.emitFilter(...) → ✅ 正常触发
// 2. 实际数据库操作
// 3. if (bypassEmitAction) → 收集事件，不 emitAction
//    else → emitter.emitAction(...)
```

---

## 5. Filter 事件拒绝分支错误传播

### 5.1 传播链路总览

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    Filter 事件拒绝分支错误传播链路                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  场景: 配置了一个 Event Trigger，类型为 Filter，流程中有节点抛出错误                │
│                                                                                  │
│  1. ItemsService.emitFilter()                                                   │
│     ┌─────────────────────────────────────────────────────────────────────┐   │
│     │  for (const listener of listeners) {                                  │   │
│     │      result = await listener(payload, meta, context);                 │   │
│     │      // 如果 listener 抛出错误，这里会直接抛出，中断循环                │   │
│     │  }                                                                     │   │
│     └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                                   │
│                              ▼                                                   │
│  2. FlowManager 的 Filter Handler                                                │
│     (注册时绑定)                                                                  │
│     ┌─────────────────────────────────────────────────────────────────────┐   │
│     │  const handler: FilterHandler = (payload, meta, context) =>         │   │
│     │      this.executeFlow(                                                │   │
│     │          flow,                                                         │   │
│     │          { payload, ...meta },                                        │   │
│     │          {                                                             │   │
│     │              accountability: context['accountability'],               │   │
│     │              database: context['database'],    // ⚠️ 透传连接        │   │
│     │              getSchema: ...,                                           │   │
│     │          },                                                             │   │
│     │      );                                                                 │   │
│     └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                                   │
│                              ▼                                                   │
│  3. FlowManager.executeFlow()                                                    │
│     ┌─────────────────────────────────────────────────────────────────────┐   │
│     │  // 执行所有操作节点...                                                 │   │
│     │  while (nextOperation !== null) {                                     │   │
│     │      const { successor, data, status, options } =                    │   │
│     │          await this.executeOperation(nextOperation, keyedData, context);
│     │                                                                       │   │
│     │      lastOperationStatus = status;  // 'resolve' | 'reject'          │   │
│     │      nextOperation = successor;                                        │   │
│     │  }                                                                     │   │
│     │                                                                       │   │
│     │  // ⚠️ 关键: 流程结束后的错误传播判断                                    │   │
│     │  if (                                                                 │   │
│     │      flow.trigger === 'event' &&                                      │   │
│     │      flow.options['type'] === 'filter' &&                             │   │
│     │      lastOperationStatus === 'reject'                                 │   │
│     │  ) {                                                                   │   │
│     │      throw keyedData[LAST_KEY];  // 抛出最后一个操作的结果             │   │
│     │  }                                                                     │   │
│     └─────────────────────────────────────────────────────────────────────┘   │
│                              │                                                   │
│                              ▼                                                   │
│  4. 错误冒泡回 ItemsService                                                       │
│     ┌─────────────────────────────────────────────────────────────────────┐   │
│     │  // CreateOne: Filter 在事务内                                         │   │
│     │  transaction(this.knex, async (trx) => {                             │   │
│     │      const payloadAfterHooks = await emitter.emitFilter(...);         │   │
│     │      // 如果抛出错误，事务自动回滚                                       │   │
│     │  });                                                                   │   │
│     │                                                                       │   │
│     │  // UpdateMany/DeleteMany: Filter 在事务外                             │   │
│     │  const payloadAfterHooks = await emitter.emitFilter(...);             │   │
│     │  // 如果抛出错误，后续代码不执行，事务还没开始                            │   │
│     └─────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 错误传播条件详解

**源码位置**: `api/src/flows.ts:484-495`

```typescript
// executeFlow 末尾的错误传播逻辑

// 条件1: Manual 或 Webhook 触发器 + 同步 + error_on_reject + 最后状态 reject
if (
    (flow.trigger === 'manual' || flow.trigger === 'webhook') &&
    flow.options['async'] !== true &&
    flow.options['error_on_reject'] === true &&
    lastOperationStatus === 'reject'
) {
    throw keyedData[LAST_KEY];
}

// 条件2: ⚠️ 关键 - Event 触发器 + Filter 类型 + 最后状态 reject
if (flow.trigger === 'event' && flow.options['type'] === 'filter' && lastOperationStatus === 'reject') {
    throw keyedData[LAST_KEY];
}
```

**错误传播条件对比表**:

| 触发器类型 | 子类型/配置 | 最后状态 | 是否抛出错误 | 错误影响 |
|------------|-------------|----------|--------------|----------|
| **Event** | `type: 'filter'` | reject | ✅ **抛出** | 中断原操作 |
| **Event** | `type: 'filter'` | resolve | ❌ 不抛 | 正常继续 |
| **Event** | `type: 'action'` | reject | ❌ 不抛 | 静默忽略 |
| **Event** | `type: 'action'` | resolve | ❌ 不抛 | 正常完成 |
| **Manual** | `async: false` + `error_on_reject: true` | reject | ✅ 抛出 | 返回错误响应 |
| **Manual** | `async: true` 或 `error_on_reject: false` | reject | ❌ 不抛 | 静默忽略 |
| **Webhook** | `async: false` + `error_on_reject: true` | reject | ✅ 抛出 | 返回错误响应 |
| **Webhook** | `async: true` 或 `error_on_reject: false` | reject | ❌ 不抛 | 静默忽略 |
| **Operation** | 任意 | 任意 | ❌ 不抛 | 返回结果 |
| **Schedule** | 任意 | 任意 | ❌ 不抛 | 仅记录日志 |

### 5.3 操作节点如何进入 reject 状态

**源码位置**: `api/src/flows.ts:506-600`

```typescript
private async executeOperation(operation, keyedData, context) {
    try {
        // 解析配置中的模板变量
        options = applyOptionsData(options, optionData);

        // 执行操作 handler
        let result = await handler(options, {
            services,
            env: useEnv(),
            database: getDatabase(),  // ⚠️ 注意: 这里是新连接！
            logger,
            getSchema,
            data: keyedData,
            accountability: null,
            ...context,  // ⚠️ 但 context 可能覆盖 database
        });

        // 验证结果可序列化
        JSON.stringify(result ?? null);

        // 处理 undefined 值
        if (typeof result === 'object' && result !== null) {
            result = deepMap(result, (value) => (value === undefined ? null : value));
        }

        // ===== Resolve 分支 =====
        return { 
            successor: operation.resolve, 
            status: 'resolve', 
            data: result ?? null, 
            options 
        };

    } catch (error) {
        // ===== Reject 分支 =====
        let data;

        if (error instanceof Error) {
            delete error.stack;  // 移除堆栈
            data = error;
        } else if (typeof error === 'string') {
            data = isValidJSON(error) ? parseJSON(error) : error;
        } else {
            data = error ?? null;
        }

        return {
            successor: operation.reject,  // 走 reject 分支
            status: 'reject',
            data,
            options,
        };
    }
}
```

**进入 reject 状态的情况**:

| 场景 | 示例 | 结果 |
|------|------|------|
| **Handler 抛出错误** | `throw new Error('validation failed')` | reject |
| **Handler 返回 rejected Promise** | `return Promise.reject('error')` | reject |
| **Condition 操作验证失败** | `validatePayload()` 返回 errors | reject |
| **Throw Error 操作** | 执行 `throw-error` 操作 | reject |

**特殊的 Condition 操作**:

**源码位置**: `api/src/operations/condition/index.ts`

```typescript
export default defineOperationApi<Options>({
    id: 'condition',

    handler: ({ filter }, { data, accountability }) => {
        const parsedFilter = parseFilter(filter, accountability, undefined, true);

        if (!parsedFilter) {
            return null;  // 空 filter = resolve
        }

        const errors = validatePayload(parsedFilter, data, { requireAll: true });

        if (errors.length > 0) {
            // 验证失败 = 抛出错误 = reject
            const validationErrors = errors
                .map((error) =>
                    error.details.map((details) => new FailedValidationError(...)),
                )
                .flat();

            throw validationErrors;  // ⚠️ 抛出错误进入 reject
        } else {
            return null;  // 验证通过 = resolve
        }
    },
});
```

### 5.4 数据库连接传递的关键细节

**⚠️ 重要发现: Filter Handler 和 Flow 操作的 database 连接可能不同**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  database 连接传递路径 (Event Filter 触发器)                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ItemsService.createOne:                                                         │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ transaction(this.knex, async (trx) => {                                   │  │
│  │     emitter.emitFilter(                                                    │  │
│  │         'items.create',                                                    │  │
│  │         payload,                                                           │  │
│  │         { collection },                                                    │  │
│  │         { database: trx, ... }  // ⚠️ 事务连接                            │  │
│  │     );                                                                     │  │
│  │ });                                                                        │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                   │
│                              ▼                                                   │
│  Emitter.emitFilter:                                                            │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ for (const listener of listeners) {                                        │  │
│  │     result = await listener(payload, meta, context);  // context 有 trx   │  │
│  │ }                                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                   │
│                              ▼                                                   │
│  FlowManager 注册的 Filter Handler:                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ const handler: FilterHandler = (payload, meta, context) =>               │  │
│  │     this.executeFlow(                                                       │  │
│  │         flow,                                                               │  │
│  │         { payload, ...meta },                                              │  │
│  │         {                                                                   │  │
│  │             accountability: context['accountability'],                     │  │
│  │             database: context['database'],  // ⚠️ 透传 trx (Filter 特有)  │  │
│  │             getSchema: ...,                                                 │  │
│  │         },                                                                   │  │
│  │     );                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                   │
│                              ▼                                                   │
│  ⚠️ 对比: Action 触发器的 database 连接                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ const handler: ActionHandler = (meta, context) =>                          │  │
│  │     this.executeFlow(flow, meta, {                                         │  │
│  │         accountability: context['accountability'],                          │  │
│  │         database: getDatabase(),  // ⚠️ 新连接！不是透传                   │  │
│  │         getSchema: ...,                                                     │  │
│  │     });                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                              │                                                   │
│                              ▼                                                   │
│  executeFlow 调用 executeOperation:                                              │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ let result = await handler(options, {                                       │  │
│  │     services,                                                                │  │
│  │     env: useEnv(),                                                           │  │
│  │     database: getDatabase(),  // ⚠️ 默认新连接                              │  │
│  │     ...                                                                      │  │
│  │     ...context,  // ⚠️ 但 context 会覆盖！                                  │  │
│  │ });                                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│  结论:                                                                            │
│  - Event Filter 触发器: 操作节点可以通过 context.database 访问事务连接           │
│  - Event Action 触发器: 操作节点使用独立的 getDatabase() 连接                   │
│  - 但 executeOperation 默认用 getDatabase()，依赖 context 覆盖                   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 5.5 错误传播对事务的影响

**Create 操作 (Filter 在事务内)**:

```
时序:
┌─────────────────────────────────────────────────────────────────┐
│  transaction(trx) {                                              │
│      emitFilter() ──► 执行 Flow ──► 操作 reject ──► throw ◄───┼── 错误抛出
│      // 后续代码不执行                                            │
│  }                                                                │
│  ▲                                                                │
│  └── 事务自动回滚 ◄──────────────────────────────────────────────┘
│
结果:
- 数据库操作: ❌ 回滚，无变更
- Flow 中使用 context.database 的操作: ❌ 回滚
- Flow 中使用 getDatabase() 的操作: ⚠️ 独立事务，可能已提交
```

**Update/Delete 操作 (Filter 在事务外)**:

```
时序:
┌─────────────────────────────────────────────────────────────────┐
│  emitFilter() ──► 执行 Flow ──► 操作 reject ──► throw ◄───────┼── 错误抛出
│                                                                  │
│  // 后续代码不执行，transaction() 还没开始                         │
│  ─────────────────────────────────────────────────────────────  │
│  transaction() { ... }  // 不执行                                 │
│
结果:
- 数据库操作: ❌ 未开始
- Flow 中的操作: ⚠️ 取决于用了哪个 database 连接
```

### 5.6 错误传播条件汇总表

| 维度 | Filter 类型 Event Trigger | Action 类型 Event Trigger |
|------|---------------------------|---------------------------|
| **注册时 database 传递** | `context['database']` (透传) | `getDatabase()` (新建) |
| **操作节点可访问事务连接** | ✅ 是 (通过 context) | ❌ 否 (独立连接) |
| **操作 reject 后是否抛错** | ✅ 是 | ❌ 否 |
| **对原操作的影响** | 中断 + 回滚 | 无影响 |
| **Action 事件是否触发** | ❌ 不触发 (原操作中断) | ✅ 触发 |

---

## 6. 完整参数行为决策树

### 6.1 写入操作事件决策树

```
开始写入操作 (create/update/delete)
         │
         ▼
┌─────────────────────────┐
│ opts.emitEvents !== false? │
└───────────┬─────────────┘
            │
      ┌─────┴─────┐
      │           │
    false       true/undefined
      │           │
      ▼           ▼
  跳过事件    ┌─────────────────────────────────────┐
              │         emitFilter (前置拦截)        │
              │  (位置: create=事务内, update/delete=事务外)
              └───────────────────┬─────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │   Filter Handler 执行结果    │
                    └─────────────┬─────────────┘
                                  │
              ┌───────────────────┼───────────────────┐
              │                   │                   │
           抛出错误            返回值              返回 undefined
              │                   │                   │
              ▼                   ▼                   ▼
        中断原操作          更新 payload         保持原 payload
        (事务回滚)              │                   │
              │                   ▼                   │
              │           ┌─────────────────────────────────┐
              │           │       继续执行实际写入            │
              │           │    (create/update/delete)       │
              │           └───────────────┬─────────────────┘
              │                           │
              │                           ▼
              │           ┌─────────────────────────────────┐
              │           │       事务提交 (如果有)          │
              │           └───────────────┬─────────────────┘
              │                           │
              │                           ▼
              │           ┌─────────────────────────────────┐
              │           │       emitAction (后置通知)      │
              │           │  (位置: 始终在事务提交后)         │
              │           └───────────────┬─────────────────┘
              │                           │
              │              ┌────────────┴────────────┐
              │              │  opts.bypassEmitAction   │
              │              └────────────┬────────────┘
              │                           │
              │                    ┌──────┴──────┐
              │                    │             │
              │                 存在          不存在
              │                    │             │
              │                    ▼             ▼
              │            调用回调收集    立即 emitter.emitAction()
              │            不立即发出         异步不阻塞
              │
              └──────► 主流程中断，Action 事件不触发
```

### 6.2 关键参数组合行为表

| emitEvents | bypassEmitAction | Filter 事件 | Action 事件 | 嵌套 Action 事件 | 典型场景 |
|------------|------------------|-------------|-------------|------------------|----------|
| `undefined` | `undefined` | ✅ 触发 | ✅ 立即发出 | ✅ 立即发出 | 正常 API 请求 |
| `true` | `undefined` | ✅ 触发 | ✅ 立即发出 | ✅ 立即发出 | 显式启用 |
| `false` | 任意 | ❌ 跳过 | ❌ 跳过 | ❌ 跳过 | 内部数据迁移 |
| `undefined` | `(params) => queue.push(params)` | ✅ 触发 | 📦 收集 | 📦 收集 | 批量操作/嵌套关系 |

---

## 7. 总结与最佳实践

### 7.1 事务边界关键发现

| 操作 | Filter 事件位置 | database 连接 | 错误影响 |
|------|-----------------|---------------|----------|
| **Create** | ✅ 事务内 | `trx` | 回滚 |
| **Update** | ❌ 事务外 | `this.knex` | 不影响未开始的事务 |
| **Delete** | ❌ 事务外 | `this.knex` | 不影响未开始的事务 |

**⚠️ 重要差异**: Create 操作的 Filter 在事务内，这是为了保证新生成的主键在同一事务中可见。

### 7.2 事件参数最佳实践

1. **禁用所有事件**:
   ```typescript
   itemsService.createOne(data, { emitEvents: false });
   ```
   - 使用场景: 数据迁移、批量导入、内部维护操作

2. **收集事件延迟触发**:
   ```typescript
   const eventQueue: ActionEventParams[] = [];
   
   await itemsService.createOne(data, {
       bypassEmitAction: (params) => eventQueue.push(params),
   });
   
   // 后续统一处理
   for (const event of eventQueue) {
       // 自定义处理或延迟发出
   }
   ```
   - 使用场景: 批量操作、需要原子性事件通知、关系嵌套操作

3. **Filter 事件中使用事务连接**:
   ```typescript
   // 在 Event Filter 类型的 Flow 操作中
   // context.database 是原事务连接，可以参与回滚
   // 但要注意不是所有操作都暴露这个连接
   ```

### 7.3 错误传播注意事项

1. **Filter 类型 Event Trigger**:
   - 操作节点 `reject` 状态会抛出错误
   - Create 操作会回滚事务
   - Update/Delete 操作会中断后续流程

2. **Action 类型 Event Trigger**:
   - 操作节点 `reject` 状态静默忽略
   - 不影响原操作
   - 错误仅记录日志

3. **手动/ Webhook 触发器**:
   - 需要显式配置 `error_on_reject: true` + `async: false` 才会抛出错误
   - 适用于需要验证输入的场景

### 7.4 嵌套事件处理

嵌套事件来自 `PayloadService.processM2O/A2O/O2M` 处理关系字段时的递归操作：

- 关系字段中的嵌套创建/更新/删除会产生事件
- 通过 `bypassEmitAction` 可以统一收集
- 主操作完成后统一发出或处理

**设计意图**: 保证关系操作的事件能够被追踪，但又能灵活控制触发时机。

---

## 8. 相关源码索引

| 文件路径 | 说明 |
|----------|------|
| `api/src/services/items.ts:127-426` | `createOne` 完整实现 |
| `api/src/services/items.ts:709-974` | `updateMany` 完整实现 |
| `api/src/services/items.ts:1070-1185` | `deleteMany` 完整实现 |
| `api/src/flows.ts:196-228` | Event 触发器注册 (Filter vs Action) |
| `api/src/flows.ts:484-495` | 错误传播条件判断 |
| `api/src/flows.ts:506-600` | `executeOperation` resolve/reject 分支 |
| `api/src/emitter.ts:34-72` | `emitFilter` / `emitAction` 实现 |
| `packages/types/src/items.ts:39-109` | `MutationOptions` 类型定义 |
| `api/src/services/collections.ts:199-258` | `bypassEmitAction` 使用示例 |
| `api/src/operations/condition/index.ts` | Condition 操作 reject 逻辑 |

---

*报告生成时间: 2026-05-02*
*基于 Directus 代码库深度分析*
