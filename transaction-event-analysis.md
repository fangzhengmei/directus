# Directus 事务处理与事件副作用边界控制分析

## 概述

Directus 采用了清晰的事务边界与事件副作用隔离机制，确保数据库操作的原子性与事件触发的可靠性。本文档详细分析了 Directus 中事务处理通过数据库适配器执行时，事件副作用的边界控制策略。

## 核心架构组件

### 1. 事务处理机制

**文件**: `api/src/utils/transaction.ts`

```typescript
export const transaction = async <T = unknown>(
    knex: Knex,
    handler: (knex: Knex.Transaction) => Promise<T>,
): Promise<T> => {
    if (knex.isTransaction) {
        return handler(knex as Knex.Transaction);
    } else {
        try {
            return await knex.transaction((trx) => handler(trx));
        } catch (error) {
            // 重试逻辑...
        }
    }
};
```

**关键特性**:
- 支持嵌套事务检测（`knex.isTransaction`）
- 自动事务重试机制（针对死锁、SQLITE_BUSY 等错误）
- 最大重试 3 次，使用指数退避策略

### 2. 事件系统

**文件**: `api/src/emitter.ts`

Directus 的事件系统分为三种类型：

| 事件类型 | 触发方式 | 执行特性 |
|---------|---------|---------|
| **Filter** | `emitFilter` | 同步执行，可修改 payload，可抛出错误阻止操作 |
| **Action** | `emitAction` | 异步执行（`emitAsync`），不等待完成，错误不影响主流程 |
| **Init** | `emitInit` | 系统初始化时触发 |

## 事件副作用边界控制策略

### 核心原则：两阶段事件模型

Directus 采用**两阶段事件模型**来严格控制事件副作用与事务的边界：

#### 阶段一：事务内的 Filter 事件

**触发时机**: 事务提交前，数据库操作执行前后
**数据库上下文**: 使用事务内的 `trx` 对象

**示例** (`api/src/services/items.ts:151-170`):

```typescript
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
    // Filter 事件在事务内部执行
    const payloadAfterHooks =
        opts.emitEvents !== false
            ? await emitter.emitFilter(
                    this.eventScope === 'items'
                        ? ['items.create', `${this.collection}.items.create`]
                        : `${this.eventScope}.create`,
                    payload,
                    {
                        collection: this.collection,
                    },
                    {
                        database: trx,  // 使用事务连接
                        schema: this.schema,
                        accountability: this.accountability,
                    },
                )
            : payload;
    // ... 后续数据库操作
});
```

**边界控制**:
- Filter 事件抛出错误会导致事务回滚
- Filter 事件可以访问事务内的未提交数据
- Filter 事件可以修改即将写入数据库的 payload

#### 阶段二：事务外的 Action 事件

**触发时机**: 事务提交成功后
**数据库上下文**: 使用新的数据库连接 `getDatabase()`，而非事务对象

**示例** (`api/src/services/items.ts:388-419`):

```typescript
// 事务结束后才触发 Action 事件
if (opts.emitEvents !== false) {
    const actionEvent = {
        event:
            this.eventScope === 'items'
                ? ['items.create', `${this.collection}.items.create`]
                : `${this.eventScope}.create`,
        meta: {
            payload: actionHookPayload,
            key: primaryKey,
            collection: this.collection,
        },
        context: {
            database: getDatabase(),  // 使用新的数据库连接，不是事务！
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
```

**边界控制**:
- Action 事件只在事务成功提交后触发
- Action 事件使用独立的数据库连接，无法访问事务内状态
- Action 事件是异步的，不会阻塞主流程
- Action 事件的错误不会影响已提交的事务

### 嵌套操作的事件收集机制

对于批量操作或嵌套关系操作，Directus 使用 `bypassEmitAction` 选项来收集事件，确保所有事件在最外层事务提交后统一触发。

**示例** (`api/src/services/items.ts:433-494`):

```typescript
async createMany(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    const { primaryKeys, nestedActionEvents } = await transaction(this.knex, async (knex) => {
        const service = this.fork({ knex });
        const nestedActionEvents: ActionEventParams[] = [];

        for (const [index, payload] of data.entries()) {
            const primaryKey = await service.createOne(payload, {
                ...(opts || {}),
                // 收集嵌套事件，而不是立即触发
                bypassEmitAction: (params) => nestedActionEvents.push(params),
                // ...
            });
            primaryKeys.push(primaryKey);
        }

        return { primaryKeys, nestedActionEvents };
    });

    // 事务提交后，统一触发所有事件
    if (opts.emitEvents !== false) {
        for (const nestedActionEvent of nestedActionEvents) {
            if (opts.bypassEmitAction) {
                opts.bypassEmitAction(nestedActionEvent);
            } else {
                emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
            }
        }
    }
}
```

**关键机制**:
1. **事件收集**: 使用 `bypassEmitAction` 回调将 Action 事件参数收集到数组中
2. **延迟触发**: 在事务成功提交后，遍历数组统一触发所有事件
3. **上下文隔离**: 每个收集的事件都有独立的上下文，在触发时使用新的数据库连接

### 跨服务的事件传播

其他服务（如 `CollectionsService`、`RelationsService`）也遵循相同的模式：

**示例** (`api/src/services/collections.ts:199-259`):

```typescript
await fieldItemsService.createMany(sortedFieldPayloads, {
    bypassEmitAction: (params) =>
        opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
    bypassLimits: true,
});

// 事务结束后
if (opts?.emitEvents !== false && nestedActionEvents.length > 0) {
    const updatedSchema = await getSchema();

    for (const nestedActionEvent of nestedActionEvents) {
        nestedActionEvent.context.schema = updatedSchema;
        emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
    }
}
```

**特殊处理**: 对于涉及 Schema 变更的操作（如创建/删除集合），会在事件触发前重新获取更新后的 Schema。

## 边界控制流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                      数据库操作请求                              │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    开始事务 (transaction)                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  阶段一: 事务内操作                                       │  │
│  │                                                          │  │
│  │  1. emitFilter (同步，可修改 payload，可阻止操作)         │  │
│  │     └── 使用 trx 数据库连接                              │  │
│  │                                                          │  │
│  │  2. 数据库操作 (INSERT/UPDATE/DELETE)                    │  │
│  │     └── 使用 trx 数据库连接                              │  │
│  │                                                          │  │
│  │  3. 嵌套操作 (关系字段处理)                               │  │
│  │     └── bypassEmitAction 收集事件                       │  │
│  │                                                          │  │
│  │  4. 活动记录/修订记录写入                                │  │
│  │     └── 使用 trx 数据库连接                              │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
    ┌─────────────────┐     ┌─────────────────┐
    │   事务成功提交    │     │   事务回滚       │
    │  (COMMIT)       │     │  (ROLLBACK)     │
    └────────┬────────┘     └────────┬────────┘
             │                       │
             ▼                       ▼
    ┌─────────────────┐     ┌─────────────────┐
    │  阶段二: 事务外   │     │   不触发任何    │
    │  Action 事件     │     │   Action 事件   │
    │  (异步触发)      │     │                 │
    │                 │     │                 │
    │  1. 遍历收集的    │     │                 │
    │     事件数组     │     │                 │
    │                 │     │                 │
    │  2. emitAction  │     │                 │
    │     └── 使用     │     │                 │
    │         getDatabase() │                 │
    │         新连接   │     │                 │
    └─────────────────┘     └─────────────────┘
```

## 关键技术决策分析

### 1. 为什么 Filter 事件在事务内？

- **可中断性**: Filter 事件可以通过抛出错误来阻止数据库操作，保证业务规则验证
- **数据一致性**: Filter 事件可以访问和修改即将写入的数据
- **原子性**: 如果 Filter 事件失败，整个事务回滚，不会产生部分写入

### 2. 为什么 Action 事件在事务外？

- **可靠性**: Action 事件只在事务成功后触发，避免了"事件已发送但数据回滚"的不一致情况
- **隔离性**: Action 事件使用独立的数据库连接，不会持有事务锁
- **异步性**: Action 事件是异步的，不会阻塞主请求流程
- **可观测性**: Action 事件触发时数据已持久化，便于外部系统集成

### 3. 为什么使用 bypassEmitAction 收集嵌套事件？

- **批量操作一致性**: 确保批量操作的所有事件在整个批量事务完成后才触发
- **嵌套关系处理**: 处理 M2O、A2O、O2M 等关系时，避免部分触发
- **事件顺序保证**: 可以控制事件触发的顺序和时机
- **Schema 更新同步**: 对于 Schema 变更操作，可以先更新 Schema 再触发事件

## 代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 事务工具函数 | `api/src/utils/transaction.ts` | 14-52 |
| 事件发射器 | `api/src/emitter.ts` | 6-118 |
| ItemsService.createOne | `api/src/services/items.ts` | 127-426 |
| ItemsService.createMany | `api/src/services/items.ts` | 433-494 |
| ItemsService.updateMany | `api/src/services/items.ts` | 709-974 |
| ItemsService.deleteMany | `api/src/services/items.ts` | 1070-1185 |
| CollectionsService | `api/src/services/collections.ts` | 63-847 |
| RelationsService | `api/src/services/relations.ts` | 191-520 |

## 总结

Directus 通过以下策略实现了严格的事件副作用边界控制：

1. **两阶段事件模型**: Filter 在事务内，Action 在事务外
2. **数据库连接隔离**: 事务内使用 `trx`，事务外使用 `getDatabase()`
3. **嵌套事件收集**: 使用 `bypassEmitAction` 延迟触发嵌套操作的事件
4. **异步事件执行**: Action 事件使用 `emitAsync`，不阻塞主流程

这种设计确保了：
- 数据库操作的原子性
- 事件触发的可靠性（只在事务成功后触发）
- 系统的可扩展性（异步事件处理）
- 数据的最终一致性
