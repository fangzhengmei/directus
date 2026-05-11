# Directus 事务与事件边界控制深度分析（第二轮）

## 概述

本文档深入分析 Directus 在不同数据库适配器下的事务与事件边界差异，重点探讨：
1. 不同数据库的死锁重试机制差异
2. 事务回滚后事件触发的严格边界控制
3. 批量/嵌套写入失败时的副作用隔离策略
4. 从写入到事件投递的完整失败路径时序分析

---

## 一、不同数据库适配器的死锁重试机制差异

### 1.1 事务重试核心逻辑

**文件**: `api/src/utils/transaction.ts:14-52`

```typescript
export const transaction = async <T = unknown>(
    knex: Knex,
    handler: (knex: Knex.Transaction) => Promise<T>,
): Promise<T> => {
    if (knex.isTransaction) {
        return handler(knex as Knex.Transaction);  // 嵌套事务，直接执行不重试
    } else {
        try {
            return await knex.transaction((trx) => handler(trx));
        } catch (error) {
            const client = getDatabaseClient(knex);

            if (!shouldRetryTransaction(client, error)) throw error;

            const MAX_ATTEMPTS = 3;
            const BASE_DELAY = 100;
            const logger = useLogger();

            for (let attempt = 0; attempt < MAX_ATTEMPTS; ++attempt) {
                const delay = 2 ** attempt * BASE_DELAY;
                await new Promise((resolve) => setTimeout(resolve, delay));
                logger.trace(`Restarting failed transaction (attempt ${attempt + 1}/${MAX_ATTEMPTS})`);
                try {
                    return await knex.transaction((trx) => handler(trx));
                } catch (error) {
                    if (!shouldRetryTransaction(client, error)) throw error;
                }
            }

            const attempts = 1 + MAX_ATTEMPTS;
            throw new Error(`Transaction failed after ${attempts} attempts`, { cause: error });
        }
    }
};
```

### 1.2 各数据库重试错误码配置

**文件**: `api/src/utils/transaction.ts:54-99`

```typescript
function shouldRetryTransaction(client: DatabaseClient, error: unknown): boolean {
    const COCKROACH_RETRY_ERROR_CODE = '40001';
    const SQLITE_BUSY_ERROR_CODE = 'SQLITE_BUSY';
    const MYSQL_DEADLOCK_CODE = 'ER_LOCK_DEADLOCK';
    const POSTGRES_DEADLOCK_CODE = '40P01';
    const ORACLE_DEADLOCK_CODE = 'ORA-00060';
    const MSSQL_DEADLOCK_CODE = 'EREQUEST';
    const MSSQL_DEADLOCK_NUMBER = '1205';

    const codes: Record<DatabaseClient, Record<string, any>[]> = {
        cockroachdb: [{ code: COCKROACH_RETRY_ERROR_CODE }],
        sqlite: [{ code: SQLITE_BUSY_ERROR_CODE }],
        mysql: [{ code: MYSQL_DEADLOCK_CODE }],
        mssql: [{ code: MSSQL_DEADLOCK_CODE, number: MSSQL_DEADLOCK_NUMBER }],
        oracle: [{ code: ORACLE_DEADLOCK_CODE }],
        postgres: [{ code: POSTGRES_DEADLOCK_CODE }],
        redshift: [],
    };

    return (
        isObject(error) &&
        codes[client].some((code) => {
            return Object.entries(code).every(([key, value]) => String(error[key]) === value);
        })
    );
}
```

### 1.3 各数据库重试策略对比表

| 数据库 | 错误码/条件 | 重试触发条件 | 最大重试次数 | 延迟策略 | 特殊说明 |
|--------|------------|------------|------------|---------|---------|
| **CockroachDB** | `code: '40001'` | 序列化失败/事务冲突 | 4次（1次初始 + 3次重试） | 指数退避：100ms, 200ms, 400ms | 分布式数据库特有的事务重试机制 |
| **PostgreSQL** | `code: '40P01'` | 死锁检测 | 4次 | 指数退避 | 标准死锁检测 |
| **MySQL/MariaDB** | `code: 'ER_LOCK_DEADLOCK'` | InnoDB 死锁 | 4次 | 指数退避 | 仅支持 DML 事务回滚 |
| **SQLite** | `code: 'SQLITE_BUSY'` | 数据库文件锁定 | 4次 | 指数退避 | 文件级锁冲突 |
| **Oracle** | `code: 'ORA-00060'` | 死锁检测 | 4次 | 指数退避 | 标准 ORA 错误码 |
| **MS SQL Server** | `code: 'EREQUEST'` + `number: '1205'` | 死锁受害者 | 4次 | 指数退避 | 需要同时检查 code 和 number |
| **Redshift** | 无 | 不重试 | 0次 | - | 数据仓库场景不支持重试 |

### 1.4 嵌套事务的特殊处理

**关键代码**: `transaction.ts:18-19`

```typescript
if (knex.isTransaction) {
    return handler(knex as Knex.Transaction);  // 不创建新事务，不重试
}
```

**边界控制规则**:
1. 如果当前已经在事务中（`knex.isTransaction === true`），直接执行 handler
2. **嵌套事务不触发重试逻辑**，由最外层事务负责重试
3. 这确保了重试的原子性——整个事务链要么全部成功，要么全部回滚

### 1.5 MySQL DDL 事务限制

**代码注释**: `api/src/services/collections.ts:99-101`

```typescript
// Create the collection/fields in a transaction so it'll be reverted in case of errors or
// permission problems. This might not work reliably in MySQL, as it doesn't support DDL in
// transactions.
```

**数据库 DDL 事务支持对比**:

| 数据库 | DDL 事务支持 | 说明 |
|--------|-------------|------|
| **PostgreSQL** | ✅ 完全支持 | DDL 可以回滚 |
| **CockroachDB** | ✅ 完全支持 | 分布式 DDL 事务 |
| **SQLite** | ✅ 基本支持 | 简单 DDL 可回滚 |
| **MySQL** | ❌ 不支持 | DDL 执行会隐式提交当前事务 |
| **Oracle** | ⚠️ 部分支持 | 部分 DDL 支持 |
| **MS SQL Server** | ⚠️ 部分支持 | 依赖版本和配置 |

**MySQL 隐式提交问题**（`database/index.ts:109-110`）:

```typescript
// Ignore warning about MySQL not supporting TRX for DDL
if (msg.startsWith('Transaction was implicitly committed, do not mix transactions and DDL with MySQL')) return;
```

**风险**: 在 MySQL 中，如果事务中包含 DDL 语句（如 `CREATE TABLE`、`ALTER TABLE`），执行 DDL 时会隐式提交之前的所有操作。如果后续操作失败，已执行的 DDL **无法回滚**。

---

## 二、事务回滚后事件触发的严格边界控制

### 2.1 两阶段事件模型的边界保证

Directus 采用严格的"两阶段事件模型"，确保事件只在事务成功提交后触发。

**核心原则**:
1. **Filter 事件**（`emitFilter`）在事务内执行，可中断事务
2. **Action 事件**（`emitAction`）在事务外执行，**只在事务成功后触发**

### 2.2 Filter 事件的事务内边界

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
- ✅ Filter 事件抛出错误 → 事务回滚 → **不触发任何 Action 事件**
- ✅ Filter 事件可以修改 payload → 修改后的值用于数据库写入
- ✅ Filter 事件可以访问事务内的未提交数据
- ❌ Filter 事件的副作用（如外部 API 调用）**不会随事务回滚**

### 2.3 Action 事件的事务外边界

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

**关键边界保证**:

| 场景 | 事务状态 | Action 事件触发 | 原因 |
|------|---------|----------------|------|
| 事务成功提交 | COMMITTED | ✅ 触发 | 代码在 `transaction()` 之后执行 |
| 事务回滚（Filter 抛错） | ROLLED BACK | ❌ 不触发 | 异常抛出，后续代码不执行 |
| 事务回滚（数据库错误） | ROLLED BACK | ❌ 不触发 | 异常抛出，后续代码不执行 |
| 事务重试中 | RETRYING | ❌ 不触发 | 每次重试都是独立的事务 |
| 重试全部失败 | ROLLED BACK | ❌ 不触发 | 最终异常抛出 |

### 2.4 Action 事件的异步特性

**文件**: `api/src/emitter.ts:62-72`

```typescript
public emitAction(event: string | string[], meta: Record<string, any>, context: EventContext | null = null): void {
    const logger = useLogger();
    const events = Array.isArray(event) ? event : [event];

    for (const event of events) {
        this.actionEmitter.emitAsync(event, { event, ...meta }, context ?? this.getDefaultContext()).catch((err) => {
            logger.warn(`An error was thrown while executing action "${event}"`);
            logger.warn(err);
        });
    }
}
```

**异步边界特性**:
1. **Fire-and-forget**: `emitAsync` 返回 Promise 但不 await
2. **错误隔离**: Action 事件错误只记录日志，不影响主流程
3. **独立上下文**: 使用 `getDatabase()` 新连接，不持有事务锁
4. **最终一致性**: Action 事件可能在事务提交后的任意时间执行

### 2.5 事件禁用选项

Directus 提供了多种控制事件触发的选项：

| 选项 | 作用范围 | 说明 |
|------|---------|------|
| `emitEvents: false` | Filter + Action | 完全禁用事件 |
| `bypassEmitAction: callback` | Action | 收集事件而不触发 |
| `skipTracking: true` | Activity/Revision | 跳过活动追踪 |

**示例** (`api/src/utils/versioning/handle-version.ts:59-63`):

```typescript
await itemsServiceAdmin.updateOne(key, rawDelta, {
    emitEvents: false,      // 禁用所有事件
    autoPurgeCache: false,
    skipTracking: true,      // 跳过活动追踪
    // ...
});
```

---

## 三、批量/嵌套写入失败时的副作用隔离

### 3.1 嵌套事件收集机制

**核心模式**: 使用 `bypassEmitAction` 收集嵌套操作的事件，确保在最外层事务提交后统一触发。

**批量创建示例** (`api/src/services/items.ts:433-494`):

```typescript
async createMany(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    const { primaryKeys, nestedActionEvents } = await transaction(this.knex, async (knex) => {
        const service = this.fork({ knex });
        const nestedActionEvents: ActionEventParams[] = [];

        for (const [index, payload] of data.entries()) {
            const primaryKey = await service.createOne(payload, {
                ...(opts || {}),
                // 关键：收集事件而不是立即触发
                bypassEmitAction: (params) => nestedActionEvents.push(params),
                mutationTracker: opts.mutationTracker,
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

### 3.2 关系字段嵌套处理

**文件**: `api/src/services/payload.ts:551-1076`

Directus 支持三种关系类型的嵌套写入，每种都有独立的事件收集机制：

#### 3.2.1 Many-to-One (M2O) 处理

**代码**: `payload.ts:665-760`

```typescript
async processM2O(data: Partial<Item>, opts?: MutationOptions): Promise<...> {
    const nestedActionEvents: ActionEventParams[] = [];

    for (const relation of relationsToProcess) {
        const service = getService(relation.related_collection, {
            accountability: this.accountability,
            knex: this.knex,  // 使用当前事务连接
            schema: this.schema,
            nested: [...this.nested, relation.field],
        });

        if (exists) {
            await service.updateOne(relatedPrimaryKey, record, {
                bypassEmitAction: (params) =>
                    opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
                // ...
            });
        } else {
            relatedPrimaryKey = await service.createOne(relatedRecord, {
                bypassEmitAction: (params) =>
                    opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
                // ...
            });
        }
    }

    return { payload, revisions, nestedActionEvents, userIntegrityCheckFlags };
}
```

#### 3.2.2 Any-to-One (A2O) 处理

**代码**: `payload.ts:551-660`

```typescript
async processA2O(data: Partial<Item>, opts?: MutationOptions): Promise<...> {
    const nestedActionEvents: ActionEventParams[] = [];
    // 与 M2O 类似的模式，需要根据 one_collection_field 动态确定目标集合
    // ...
}
```

#### 3.2.3 One-to-Many (O2M) 处理

**代码**: `payload.ts:765-1076`

O2M 关系支持两种更新模式：

1. **完整数组替换**: 提供完整的子项数组
2. **差异更新对象**: `{ create: [...], update: [...], delete: [...] }`

```typescript
async processO2M(data: Partial<Item>, parent: PrimaryKey, opts?: MutationOptions): Promise<...> {
    const nestedActionEvents: ActionEventParams[] = [];

    for (const relation of relationsToProcess) {
        const service = getService(relation.collection, {
            knex: this.knex,  // 事务连接
            // ...
        });

        // 模式 1: 完整数组替换
        if (!field || Array.isArray(field)) {
            savedPrimaryKeys.push(
                ...(await service.upsertMany(recordsToUpsert, {
                    bypassEmitAction: (params) =>
                        opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
                    // ...
                })),
            );

            // 删除/解除关联不在数组中的子项
            if (relation.meta.one_deselect_action === 'delete') {
                await service.deleteByQuery(query, {
                    bypassEmitAction: (params) =>
                        opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
                    // ...
                });
            } else {
                await service.updateByQuery(query, { [relation.field]: null }, {
                    bypassEmitAction: (params) =>
                        opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
                    // ...
                });
            }
        }
        // 模式 2: 差异更新
        else {
            if (alterations.create) {
                await service.createMany(createPayload, {
                    bypassEmitAction: (params) =>
                        opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
                });
            }
            if (alterations.update) { /* ... */ }
            if (alterations.delete) { /* ... */ }
        }
    }

    return { revisions, nestedActionEvents, userIntegrityCheckFlags };
}
```

### 3.3 嵌套事件的层级传播

**事件收集链路**（以 `createOne` 为例）:

```
ItemsService.createOne()
├── transaction() 开始
│   ├── emitFilter() → 事务内，可中断
│   ├── processM2O()
│   │   └── ItemsService.createOne/updateOne()
│   │       └── bypassEmitAction: push to nestedActionEventsM2O
│   ├── processA2O()
│   │   └── ItemsService.createOne/updateOne()
│   │       └── bypassEmitAction: push to nestedActionEventsA2O
│   ├── INSERT 主记录
│   └── processO2M()
│       ├── ItemsService.upsertMany()
│       │   └── transaction() [嵌套，不重试]
│       │       └── bypassEmitAction: push to nestedActionEventsO2M
│       ├── ItemsService.deleteByQuery/updateByQuery()
│       │   └── bypassEmitAction: push to nestedActionEventsO2M
│       └── Activity/Revision 写入
├── transaction() 结束（COMMIT 或 ROLLBACK）
│
├── [成功] 统一触发事件
│   ├── emitter.emitAction(主事件)
│   └── emitter.emitAction(每个 nestedActionEvent)
│
└── [失败] 不触发任何事件
    └── 异常向上抛出
```

### 3.4 失败时的副作用隔离保证

**隔离层级**:

| 层级 | 失败场景 | 已执行操作 | 事件状态 | 回滚保证 |
|------|---------|-----------|---------|---------|
| **最外层事务** | `createMany` 第 3 项失败 | 第 1、2 项的数据库写入 | 未触发 | ✅ 全部回滚 |
| **M2O 嵌套** | 子项创建失败 | 父项 M2O 字段处理 | 未触发 | ✅ 全部回滚 |
| **O2M 嵌套** | 第 5 个子项失败 | 前 4 个子项写入 | 未触发 | ✅ 全部回滚 |
| **A2O 嵌套** | 目标集合不存在 | 无写入 | 未触发 | ✅ 无操作 |
| **Activity 记录** | 活动记录写入失败 | 主记录已写入 | 未触发 | ✅ 全部回滚 |

**关键保证**: 所有数据库操作（主记录 + 关系记录 + 活动记录 + 修订记录）都在**同一个事务**中，任何失败都会导致**全部回滚**，且**不触发任何 Action 事件**。

### 3.5 Schema 变更操作的特殊处理

对于涉及 Schema 变更的操作（如 `CollectionsService`、`RelationsService`），事件触发前会刷新 Schema：

**示例** (`api/src/services/collections.ts:251-258`):

```typescript
if (opts?.emitEvents !== false && nestedActionEvents.length > 0) {
    const updatedSchema = await getSchema();  // 刷新 Schema

    for (const nestedActionEvent of nestedActionEvents) {
        nestedActionEvent.context.schema = updatedSchema;  // 使用更新后的 Schema
        emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
    }
}
```

**原因**: Schema 变更（如创建表、添加字段）需要事件处理器能访问到最新的 Schema 信息。

---

## 四、完整失败路径时序分析

### 4.1 场景：批量创建带 O2M 关系，第 3 项的子项失败

**请求示例**:
```json
{
    "collection": "articles",
    "data": [
        { "title": "Article 1", "comments": [{ "text": "Comment 1" }] },
        { "title": "Article 2", "comments": [{ "text": "Comment 2" }] },
        { "title": "Article 3", "comments": [{ "text": "Comment 3 - WILL FAIL" }] }
    ]
}
```

### 4.2 详细时序图

```
时间轴 ──────────────────────────────────────────────────────────────────────►

T0  客户端请求
    │
    ▼
T1  ItemsService.createMany() 开始
    │
    ▼
T2  transaction() 开始 [最外层事务 TX-1]
    │
    ├── T2.1 创建数据库连接池中的新连接
    │
    ├── T2.2 执行 BEGIN TRANSACTION
    │
    └── T2.3 调用 handler(trx)
        │
        ▼
T3  第 1 个 article 处理
    │
    ├── T3.1 ItemsService.createOne(article-1)
    │   │
    │   ├── T3.1.1 transaction() 检测到 isTransaction=true，直接执行
    │   │
    │   ├── T3.1.2 emitFilter('items.create')
    │   │   │   └── [事务内执行，可修改 payload]
    │   │
    │   ├── T3.1.3 processM2O() → 无 M2O 字段
    │   │
    │   ├── T3.1.4 processA2O() → 无 A2O 字段
    │   │
    │   ├── T3.1.5 INSERT INTO articles (title='Article 1')
    │   │   │   └── [TX-1 中执行，未提交]
    │   │
    │   ├── T3.1.6 processO2M(comments)
    │   │   │
    │   │   ├── T3.1.6.1 ItemsService.createMany(comments)
    │   │   │   │
    │   │   │   ├── T3.1.6.1.1 transaction() 检测到嵌套，直接执行
    │   │   │   │
    │   │   │   ├── T3.1.6.1.2 emitFilter('items.create') [comment]
    │   │   │   │
    │   │   │   ├── T3.1.6.1.3 INSERT INTO comments (text='Comment 1')
    │   │   │   │   │   └── [TX-1 中执行]
    │   │   │   │
    │   │   │   └── T3.1.6.1.4 bypassEmitAction → 收集到 nestedActionEvents
    │   │   │
    │   │   └── T3.1.6.2 返回 nestedActionEventsO2M
    │   │
    │   ├── T3.1.7 INSERT INTO directus_activity
    │   │   │   └── [TX-1 中执行]
    │   │
    │   ├── T3.1.8 INSERT INTO directus_revisions
    │   │   │   └── [TX-1 中执行]
    │   │
    │   └── T3.1.9 bypassEmitAction → 收集到 nestedActionEvents (createMany)
    │
    ▼
T4  第 2 个 article 处理（与第 1 个相同）
    │
    ├── T4.1 INSERT article-2
    ├── T4.2 INSERT comments for article-2
    ├── T4.3 INSERT activity/revisions
    └── T4.4 收集事件
    │
    ▼
T5  第 3 个 article 处理
    │
    ├── T5.1 ItemsService.createOne(article-3)
    │   │
    │   ├── T5.1.1 emitFilter('items.create')
    │   │
    │   ├── T5.1.2 INSERT article-3 [TX-1 中]
    │   │
    │   ├── T5.1.3 processO2M(comments)
    │   │   │
    │   │   ├── T5.1.3.1 ItemsService.createMany(comments)
    │   │   │   │
    │   │   │   ├── T5.1.3.1.1 emitFilter('items.create') [comment-3]
    │   │   │   │
    │   │   │   ├── T5.1.3.1.2 INSERT comment-3
    │   │   │   │   │
    │   │   │   │   └── ❌ DATABASE ERROR: e.g. constraint violation
    │   │   │   │
    │   │   │   └── T5.1.3.1.3 异常抛出
    │   │   │
    │   │   └── T5.1.3.2 异常向上传播
    │   │
    │   └── T5.1.4 异常向上传播
    │
    └── T5.2 handler(trx) 抛出异常
    │
    ▼
T6  transaction() 捕获异常
    │
    ├── T6.1 检查是否为可重试错误
    │   │
    │   └── 例如：非死锁错误 → 不重试
    │
    ├── T6.2 执行 ROLLBACK [TX-1]
    │   │
    │   ├── 撤销 article-1 的 INSERT
    │   ├── 撤销 article-1 comments 的 INSERT
    │   ├── 撤销 article-2 的 INSERT
    │   ├── 撤销 article-2 comments 的 INSERT
    │   ├── 撤销 article-3 的 INSERT（部分）
    │   ├── 撤销所有 activity/revisions 记录
    │   └── 释放所有行锁/表锁
    │
    └── T6.3 异常重新抛出
    │
    ▼
T7  createMany() 异常传播
    │
    ├── nestedActionEvents 数组随作用域销毁
    │   └── [所有收集的事件丢失]
    │
    └── 没有执行到 emitAction 代码块
    │
    ▼
T8  异常到达控制器/错误处理中间件
    │
    ├── 错误响应返回给客户端
    │
    └── 记录错误日志
    │
    ▼
T9  最终状态
    │
    ├── 数据库状态：回到事务开始前（完全一致）
    │
    ├── 事件状态：
    │   ├── article-1.create → ❌ 未触发
    │   ├── article-1.comment-1.create → ❌ 未触发
    │   ├── article-2.create → ❌ 未触发
    │   ├── article-2.comment-2.create → ❌ 未触发
    │   └── article-3.* → ❌ 未触发
    │
    ├── 副作用状态：
    │   ├── Filter 事件中执行的外部操作 → ⚠️ 已执行，无法回滚
    │   └── Action 事件 → ❌ 完全未执行
    │
    └── 缓存状态：
        └── cache.clear() 未调用 → 缓存可能包含旧数据（下次读取时刷新）
```

### 4.3 失败路径的关键保证

**保证 1：数据库原子性**
- 所有写入（主记录 + 关系 + 活动 + 修订）在同一事务中
- 任何失败导致全部回滚
- 数据库始终保持一致状态

**保证 2：事件不提前触发**
- Action 事件代码在 `transaction()` 之后
- 事务失败时，`try` 块剩余代码不执行
- 收集的事件数组随异常丢失

**保证 3：重试时的事件隔离**
- 每次重试都是全新的事务
- 重试之间的事件收集完全独立
- 只有最终成功的那次重试才会触发事件

**保证 4：异步事件不影响主流程**
- `emitAsync` 不等待完成
- 事件处理器错误不影响已提交的数据
- 事件队列故障不影响业务操作

### 4.4 边缘情况分析

#### 情况 1：Filter 事件中的外部副作用

```typescript
// 危险：Filter 事件中的外部 API 调用
emitter.onFilter('items.create', async (payload, meta, context) => {
    // 调用外部服务
    await externalApi.createResource(payload);  // ⚠️ 这个调用不会随事务回滚！
    
    return payload;
});
```

**风险**: 如果后续数据库操作失败，外部 API 调用**无法回滚**，导致数据不一致。

**建议**: 
- Filter 事件只做数据验证和转换
- 外部副作用放在 Action 事件中
- 或使用补偿事务模式

#### 情况 2：MySQL DDL 隐式提交

```typescript
// 在 MySQL 中
await transaction(this.knex, async (trx) => {
    await trx.schema.createTable('new_table', ...);  // DDL → 隐式提交！
    // 此时之前的操作已提交
    
    await trx('other_table').insert(...);  // 这是新事务的开始
    
    // 如果这里失败，new_table 不会被回滚！
});
```

**风险**: MySQL 中 DDL 语句会隐式提交当前事务，后续操作在新事务中。如果后续失败，DDL 操作无法回滚。

**建议**:
- MySQL 中避免在同一事务中混合 DDL 和 DML
- Schema 变更操作单独处理
- 注意 Directus 代码中的相关警告注释

#### 情况 3：事件处理器失败

```typescript
emitter.onAction('items.create', async (meta, context) => {
    await criticalExternalSystem.notify(meta);  // 外部系统不可用
    throw new Error('Notification failed');  // Action 事件错误
});
```

**行为**:
- `emitAsync` 的 `.catch()` 只记录日志
- 数据库数据已提交，不会回滚
- 事件处理器的失败不影响业务数据一致性

**建议**:
- Action 事件处理器应包含重试逻辑
- 使用死信队列处理失败事件
- 监控事件处理器的失败率

---

## 五、关键代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 事务重试逻辑 | `api/src/utils/transaction.ts` | 14-52 |
| 数据库错误码配置 | `api/src/utils/transaction.ts` | 54-99 |
| 事件发射器 | `api/src/emitter.ts` | 6-118 |
| ItemsService.createOne | `api/src/services/items.ts` | 127-426 |
| ItemsService.createMany | `api/src/services/items.ts` | 433-494 |
| ItemsService.updateMany | `api/src/services/items.ts` | 709-974 |
| ItemsService.deleteMany | `api/src/services/items.ts` | 1070-1185 |
| PayloadService.processM2O | `api/src/services/payload.ts` | 665-760 |
| PayloadService.processA2O | `api/src/services/payload.ts` | 551-660 |
| PayloadService.processO2M | `api/src/services/payload.ts` | 765-1076 |
| CollectionsService | `api/src/services/collections.ts` | 63-847 |
| RelationsService | `api/src/services/relations.ts` | 191-520 |
| MySQL DDL 警告 | `api/src/database/index.ts` | 109-110 |
| 版本控制禁用事件 | `api/src/utils/versioning/handle-version.ts` | 59-63 |

---

## 六、总结

Directus 通过多层次的边界控制机制，确保了事务与事件的可靠隔离：

### 数据库层面
1. **统一事务包装**: 所有写入操作通过 `transaction()` 工具函数
2. **数据库特定重试**: 针对不同数据库的错误码进行智能重试
3. **嵌套事务检测**: 防止重复创建事务和重试循环
4. **MySQL DDL 警告**: 明确标记不支持事务回滚的场景

### 事件层面
1. **两阶段分离**: Filter 在内，Action 在外
2. **延迟触发**: Action 事件代码物理上在 `transaction()` 之后
3. **嵌套收集**: `bypassEmitAction` 确保批量操作的事件统一触发
4. **异步执行**: `emitAsync` 确保事件不阻塞主流程

### 失败保证
1. **全部或全不**: 事务失败时，所有数据库操作回滚，所有 Action 事件不触发
2. **重试隔离**: 每次重试独立，只有最终成功的那次触发事件
3. **错误隔离**: 事件处理器错误不影响业务数据

这种设计在保证数据一致性的同时，提供了灵活的事件驱动能力，是 Directus 架构可靠性的关键基石。
