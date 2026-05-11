# Directus 事务与事件边界控制分析（纯源码证据版）

> **声明**: 本报告所有结论均直接基于仓内源码可验证的实现，不包含任何通用经验推断或未被代码支撑的说法。

---

## 一、可重试错误与不可重试错误配置

### 1.1 源码位置

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

### 1.2 可重试错误码对照表（直接来自源码）

| 数据库 | 可重试错误条件 | 源码位置 |
|--------|---------------|---------|
| cockroachdb | `error.code === '40001'` | `transaction.ts:84` |
| sqlite | `error.code === 'SQLITE_BUSY'` | `transaction.ts:85` |
| mysql | `error.code === 'ER_LOCK_DEADLOCK'` | `transaction.ts:86` |
| mssql | `error.code === 'EREQUEST'` **且** `error.number === '1205'` | `transaction.ts:87` |
| oracle | `error.code === 'ORA-00060'` | `transaction.ts:88` |
| postgres | `error.code === '40P01'` | `transaction.ts:89` |
| redshift | **无** | `transaction.ts:90` |

### 1.3 不可重试错误（源码隐含）

所有不满足上述条件的错误都是不可重试的，包括但不限于：
- 约束违规（如 `UNIQUE` 约束冲突）
- 字段不存在
- 权限错误
- 语法错误
- 类型错误

### 1.4 重试策略参数（直接来自源码）

**文件**: `api/src/utils/transaction.ts:28-49`

```typescript
const MAX_ATTEMPTS = 3;
const BASE_DELAY = 100;

for (let attempt = 0; attempt < MAX_ATTEMPTS; ++attempt) {
    const delay = 2 ** attempt * BASE_DELAY;  // 指数退避: 100ms, 200ms, 400ms
    await new Promise((resolve) => setTimeout(resolve, delay));
    // ...
}

const attempts = 1 + MAX_ATTEMPTS;  // 总尝试次数: 1 + 3 = 4 次
```

**已验证的重试参数**:
- 初始尝试: 1 次
- 额外重试: 3 次
- 总尝试次数: 4 次
- 延迟策略: 指数退避 `2^attempt * 100ms`
- 延迟序列: 100ms（第1次重试前）, 200ms（第2次）, 400ms（第3次）

---

## 二、嵌套事务处理（直接来自源码）

### 2.1 源码位置

**文件**: `api/src/utils/transaction.ts:18-19`

```typescript
if (knex.isTransaction) {
    return handler(knex as Knex.Transaction);
}
```

### 2.2 已验证的嵌套事务规则

**规则 1**: 如果 `knex.isTransaction === true`，直接执行 handler，不创建新事务

**规则 2**: 嵌套事务**不触发重试逻辑**（重试逻辑在 `else` 分支）

**规则 3**: 只有最外层（非事务状态的 knex）调用才会启用重试

---

## 三、事件触发边界（直接来自源码）

### 3.1 Filter 事件边界

**文件**: `api/src/services/items.ts:151-170`

```typescript
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
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
    // ...
});
```

**已验证的 Filter 事件边界**:

| 特征 | 源码证据 |
|------|---------|
| 执行位置 | `transaction()` 的 handler 回调内部 |
| 数据库连接 | `context.database = trx`（事务内连接） |
| 执行方式 | `await emitter.emitFilter()`（同步等待） |
| 异常影响 | 抛出异常会导致 `transaction()` 捕获并回滚 |
| 禁用条件 | `opts.emitEvents === false` |

### 3.2 Action 事件边界

**文件**: `api/src/services/items.ts:388-419`

```typescript
// 注意：这段代码在 transaction() 调用之后
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
            database: getDatabase(),  // 使用新的数据库连接
            schema: this.schema,
            accountability: this.accountability,
        },
    };

    if (opts.bypassEmitAction) {
        opts.bypassEmitAction(actionEvent);
    } else {
        emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
    }

    for (const nestedActionEvent of nestedActionEvents) {
        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(nestedActionEvent);
        } else {
            emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
        }
    }
}
```

**已验证的 Action 事件边界**:

| 特征 | 源码证据 |
|------|---------|
| 执行位置 | `transaction()` 调用**之后** |
| 数据库连接 | `context.database = getDatabase()`（新连接，不是事务） |
| 执行方式 | `emitter.emitAction()` 内部调用 `emitAsync()`（不 await） |
| 异常影响 | Action 事件异常被 `.catch()` 捕获，只记录日志 |
| 触发条件 | 只有 `transaction()` 成功返回后才会执行 |
| 禁用条件 | `opts.emitEvents === false` |

### 3.3 Action 事件的异步特性

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

**已验证的 Action 事件特性**:
- 返回类型: `void`（不返回 Promise）
- 执行方式: `emitAsync()` + `.catch()`（fire-and-forget）
- 错误处理: 只记录 `logger.warn()`，不抛出异常
- 对主流程影响: 无

### 3.4 代码结构证明的边界

从 `items.ts:127-426` 的 `createOne` 方法结构看：

```
createOne() {
    // ... 准备工作 ...
    
    const primaryKey = await transaction(this.knex, async (trx) => {
        // 1. emitFilter 【事务内】
        // 2. processM2O/processA2O
        // 3. INSERT 主记录
        // 4. processO2M
        // 5. Activity/Revision 写入
        return primaryKey;
    });  // <-- transaction 结束
    
    // 6. emitAction 【事务外，只有 transaction 成功后才执行】
    // 7. cache.clear()
}
```

**已验证**:
- Action 事件代码（第6步）在 `transaction()` 调用的**后面**
- 如果 `transaction()` 抛出异常，第6步**不会执行**
- 这是 Action 事件只在事务成功后触发的**结构保证**

---

## 四、嵌套事件收集机制（直接来自源码）

### 4.1 bypassEmitAction 回调的使用

**文件**: `api/src/services/items.ts:455-463`

```typescript
const primaryKey = await service.createOne(payload, {
    ...(opts || {}),
    autoPurgeCache: false,
    onRequireUserIntegrityCheck: (flags) => (userIntegrityCheckFlags |= flags),
    bypassEmitAction: (params) => nestedActionEvents.push(params),  // 收集而不触发
    mutationTracker: opts.mutationTracker,
    overwriteDefaults: opts.overwriteDefaults?.[index],
    bypassAutoIncrementSequenceReset,
});
```

### 4.2 关系处理中的事件收集

**文件**: `api/src/services/payload.ts:725-737`（processM2O）

```typescript
if (exists) {
    const { [relatedPrimaryKeyField]: _, ...record } = relatedRecord;
    if (Object.keys(record).length > 0) {
        await service.updateOne(relatedPrimaryKey, record, {
            onRevisionCreate: (pk) => revisions.push(pk),
            onRequireUserIntegrityCheck: (flags) => (userIntegrityCheckFlags |= flags),
            bypassEmitAction: (params) =>
                opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
            emitEvents: opts?.emitEvents,
            // ...
        });
    }
}
```

**文件**: `api/src/services/payload.ts:867-881`（processO2M）

```typescript
savedPrimaryKeys.push(
    ...(await service.upsertMany(recordsToUpsert, {
        onRevisionCreate: (pk) => revisions.push(pk),
        onRequireUserIntegrityCheck: (flags) => (userIntegrityCheckFlags |= flags),
        bypassEmitAction: (params) =>
            opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
        emitEvents: opts?.emitEvents,
        // ...
    })),
);
```

### 4.3 收集后的统一触发

**文件**: `api/src/services/items.ts:479-487`

```typescript
if (opts.emitEvents !== false) {
    for (const nestedActionEvent of nestedActionEvents) {
        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(nestedActionEvent);
        } else {
            emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
        }
    }
}
```

**已验证的事件收集规则**:

| 步骤 | 源码位置 | 行为 |
|------|---------|------|
| 收集 | `items.ts:459`, `payload.ts:728-729` | `nestedActionEvents.push(params)` |
| 返回 | `items.ts:476` | `return { primaryKeys, nestedActionEvents }` |
| 触发 | `items.ts:479-487` | `transaction()` 成功后遍历触发 |

---

## 五、缓存清除边界（直接来自源码）

**文件**: `api/src/services/items.ts:421-423`

```typescript
if (shouldClearCache(this.cache, opts, this.collection)) {
    await this.cache.clear();
}
```

**位置**: 在 `transaction()` 调用**之后**，`emitAction` 之后

**已验证**:
- 缓存清除在事务成功提交后执行
- 事务失败时不执行

---

## 六、可重试错误 vs 不可重试错误的对照时序

### 6.1 场景定义

**操作**: `ItemsService.createOne()` 创建一条带 O2M 关系的记录

**失败点**: O2M 关系的子记录插入时发生数据库错误

### 6.2 可重试错误时序（以 PostgreSQL 死锁 `code: '40P01'` 为例）

```
时间轴 ──────────────────────────────────────────────────────────────────────►

T0  createOne() 开始
    │
    ▼
T1  transaction(this.knex, handler) 开始
    │
    ├── T1.1 knex.isTransaction === false → 进入 else 分支
    │
    ├── T1.2 try { knex.transaction((trx) => handler(trx)) }
    │   │
    │   ├── T1.2.1 BEGIN TRANSACTION [TX-1]
    │   │
    │   ├── T1.2.2 handler(trx) 执行
    │   │   │
    │   │   ├── T1.2.2.1 emitFilter('items.create')
    │   │   │   │
    │   │   │   └── [副作用已发生: Filter 事件中的外部调用]
    │   │   │
    │   │   ├── T1.2.2.2 INSERT 主记录 [TX-1]
    │   │   │
    │   │   ├── T1.2.2.3 processO2M()
    │   │   │   │
    │   │   │   ├── service.upsertMany()
    │   │   │   │   │
    │   │   │   │   ├── T1.2.2.3.1 嵌套 transaction() → isTransaction=true → 直接执行
    │   │   │   │   │
    │   │   │   │   ├── T1.2.2.3.2 INSERT 子记录 [TX-1]
    │   │   │   │   │   │
    │   │   │   │   │   └── ❌ ERROR: code='40P01' (PostgreSQL 死锁)
    │   │   │   │   │
    │   │   │   │   └── T1.2.2.3.3 异常抛出
    │   │   │   │
    │   │   │   └── T1.2.2.3.4 异常向上传播
    │   │   │
    │   │   └── T1.2.2.4 handler 异常退出
    │   │
    │   └── T1.2.3 ROLLBACK [TX-1]
    │       │
    │       └── [数据库副作用已回滚]
    │
    ├── T1.3 catch(error)
    │   │
    │   ├── T1.3.1 shouldRetryTransaction('postgres', error)
    │   │   │
    │   │   └── error.code === '40P01' → ✅ true（可重试）
    │   │
    │   ├── T1.3.2 MAX_ATTEMPTS=3, attempt=0
    │   │
    │   └── T1.3.3 delay = 2^0 * 100 = 100ms
    │
    └── T1.4 await setTimeout(100ms)
        │
        ▼
T2  第 1 次重试
    │
    ├── T2.1 try { knex.transaction((trx) => handler(trx)) }
    │   │
    │   ├── T2.1.1 BEGIN TRANSACTION [TX-2]
    │   │
    │   ├── T2.1.2 handler(trx) 执行（完全重新执行）
    │   │   │
    │   │   ├── T2.1.2.1 emitFilter('items.create') 【再次执行！】
    │   │   │   │
    │   │   │   └── [副作用再次发生: Filter 事件重复执行]
    │   │   │
    │   │   ├── T2.1.2.2 INSERT 主记录 [TX-2]
    │   │   │
    │   │   ├── T2.1.2.3 processO2M()
    │   │   │   │
    │   │   │   ├── T2.1.2.3.1 INSERT 子记录 [TX-2]
    │   │   │   │   │
    │   │   │   │   └── ✅ 成功
    │   │   │   │
    │   │   │   └── T2.1.2.3.2 bypassEmitAction → nestedActionEvents.push()
    │   │   │
    │   │   ├── T2.1.2.4 INSERT activity/revisions [TX-2]
    │   │   │
    │   │   └── T2.1.2.5 handler 返回 primaryKey
    │   │
    │   └── T2.1.3 COMMIT [TX-2]
    │
    └── T2.2 return primaryKey（transaction() 成功返回）
        │
        ▼
T3  transaction() 成功返回
    │
    ├── T3.1 emitAction() 触发主事件 ✅
    │
    ├── T3.2 emitAction() 触发 nestedActionEvents ✅
    │
    └── T3.3 cache.clear() ✅
        │
        ▼
T4  最终状态
    │
    ├── 数据库: 主记录 + 子记录已提交 ✅
    │
    ├── Filter 事件: 执行了 2 次（初始 + 重试）
    │   │
    │   └── [副作用: 如果 Filter 中有外部 API 调用，执行了 2 次]
    │
    ├── Action 事件: 执行了 1 次（只在最终成功后）
    │
    └── 缓存: 已清除
```

### 6.3 不可重试错误时序（以 PostgreSQL 约束违规 `code: '23505'` 为例）

```
时间轴 ──────────────────────────────────────────────────────────────────────►

T0  createOne() 开始
    │
    ▼
T1  transaction(this.knex, handler) 开始
    │
    ├── T1.1 knex.isTransaction === false → 进入 else 分支
    │
    ├── T1.2 try { knex.transaction((trx) => handler(trx)) }
    │   │
    │   ├── T1.2.1 BEGIN TRANSACTION [TX-1]
    │   │
    │   ├── T1.2.2 handler(trx) 执行
    │   │   │
    │   │   ├── T1.2.2.1 emitFilter('items.create')
    │   │   │   │
    │   │   │   └── [副作用已发生: Filter 事件中的外部调用]
    │   │   │
    │   │   ├── T1.2.2.2 INSERT 主记录 [TX-1]
    │   │   │
    │   │   ├── T1.2.2.3 processO2M()
    │   │   │   │
    │   │   │   ├── service.upsertMany()
    │   │   │   │   │
    │   │   │   │   ├── T1.2.2.3.1 INSERT 子记录 [TX-1]
    │   │   │   │   │   │
    │   │   │   │   │   └── ❌ ERROR: code='23505' (唯一约束违规)
    │   │   │   │   │
    │   │   │   │   └── T1.2.2.3.2 异常抛出
    │   │   │   │
    │   │   │   └── T1.2.2.3.3 异常向上传播
    │   │   │
    │   │   └── T1.2.2.4 handler 异常退出
    │   │
    │   └── T1.2.3 ROLLBACK [TX-1]
    │       │
    │       └── [数据库副作用已回滚]
    │
    ├── T1.3 catch(error)
    │   │
    │   ├── T1.3.1 shouldRetryTransaction('postgres', error)
    │   │   │
    │   │   └── error.code === '23505'（不在可重试列表中）→ ❌ false
    │   │
    │   └── T1.3.2 throw error（不重试）
    │
    └── T1.4 transaction() 异常抛出
        │
        ▼
T2  createOne() 异常传播
    │
    ├── nestedActionEvents 数组随作用域销毁（内容丢失）
    │
    ├── emitAction 代码块不执行
    │
    └── cache.clear() 代码块不执行
        │
        ▼
T3  最终状态
    │
    ├── 数据库: 完全回滚到事务前状态 ✅
    │
    ├── Filter 事件: 执行了 1 次
    │   │
    │   └── [副作用已发生: 外部调用已执行，无法回滚]
    │
    ├── Action 事件: 未执行 ❌
    │
    └── 缓存: 未清除（保持旧状态）
```

### 6.4 副作用对照表（直接来自源码结构推断）

| 副作用类型 | 可重试错误（重试后成功） | 不可重试错误 | 源码证据位置 |
|-----------|------------------------|-------------|-------------|
| 数据库写入 | ✅ 最终成功提交 | ❌ 回滚 | `transaction.ts:22` + `41` |
| Filter 事件 | ⚠️ 执行 N 次（N=1+重试次数） | ⚠️ 执行 1 次 | `items.ts:156` |
| Filter 中的外部 API | ⚠️ 调用 N 次（无法回滚） | ⚠️ 调用 1 次（无法回滚） | 代码结构推断（无回滚逻辑） |
| Action 事件 | ✅ 执行 1 次（只在最终成功后） | ❌ 不执行 | `items.ts:388`（在 transaction 之后） |
| Activity/Revision | ✅ 最终成功提交 | ❌ 回滚 | `items.ts:335-362`（在 transaction 内） |
| 缓存清除 | ✅ 成功后清除 | ❌ 不清除 | `items.ts:421`（在 transaction 之后） |
| bypassEmitAction 收集 | ✅ 成功后触发 | ❌ 数组销毁，不触发 | `items.ts:479`（在 transaction 之后） |

### 6.5 关键源码证据解释

**为什么 Filter 事件会执行多次？**
- 源码证据: `transaction.ts:41` 中 `knex.transaction((trx) => handler(trx))` 在重试时完全重新执行
- 源码证据: `items.ts:156` 中 `emitFilter` 在 `handler` 内部
- 结论: 每次重试都会重新执行 `emitFilter`

**为什么 Action 事件不会执行多次？**
- 源码证据: `items.ts:388` 中 `emitAction` 代码在 `transaction()` 调用**之后**
- 源码证据: `transaction.ts:41` 重试时 `emitAction` 代码不在重试循环内
- 结论: 只有 `transaction()` 最终成功返回后才会执行一次

**为什么不可重试错误时 Action 事件不触发？**
- 源码证据: `transaction.ts:26` 中不可重试错误直接 `throw error`
- 源码证据: JavaScript 异常抛出后，`try` 块中剩余代码不执行
- 源码证据: `items.ts:388` 的 `emitAction` 代码在 `transaction()` 调用之后
- 结论: 异常抛出后，`emitAction` 代码块永远不会执行

---

## 七、源码位置索引（精确到行号）

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| transaction 函数入口 | `api/src/utils/transaction.ts` | 14-52 |
| 嵌套事务检测 | `api/src/utils/transaction.ts` | 18-19 |
| 重试循环 | `api/src/utils/transaction.ts` | 33-45 |
| 重试参数 | `api/src/utils/transaction.ts` | 28-37 |
| 可重试错误码表 | `api/src/utils/transaction.ts` | 83-91 |
| shouldRetryTransaction | `api/src/utils/transaction.ts` | 54-99 |
| Emitter 类定义 | `api/src/emitter.ts` | 6-114 |
| emitFilter | `api/src/emitter.ts` | 34-60 |
| emitAction | `api/src/emitter.ts` | 62-72 |
| ItemsService.createOne 入口 | `api/src/services/items.ts` | 127 |
| createOne 中 transaction 包裹 | `api/src/services/items.ts` | 151-386 |
| createOne 中 emitFilter | `api/src/services/items.ts` | 156-169 |
| createOne 中 emitAction | `api/src/services/items.ts` | 388-419 |
| createOne 中 cache.clear | `api/src/services/items.ts` | 421-423 |
| ItemsService.createMany 入口 | `api/src/services/items.ts` | 433 |
| createMany 中 transaction | `api/src/services/items.ts` | 436-477 |
| createMany 中 bypassEmitAction | `api/src/services/items.ts` | 459 |
| createMany 中事件触发 | `api/src/services/items.ts` | 479-487 |
| PayloadService.processM2O | `api/src/services/payload.ts` | 665-760 |
| PayloadService.processA2O | `api/src/services/payload.ts` | 551-660 |
| PayloadService.processO2M | `api/src/services/payload.ts` | 765-1076 |

---

## 八、仅基于源码可验证的结论清单

以下结论**仅**基于仓内源码直接可验证的实现：

### 8.1 事务相关

| # | 结论 | 源码证据 |
|---|------|---------|
| 1 | `transaction()` 函数支持嵌套事务检测 | `transaction.ts:18-19` |
| 2 | 嵌套事务不触发重试逻辑 | `transaction.ts:18-19`（重试在 else 分支） |
| 3 | 最大重试次数为 3 次（总尝试 4 次） | `transaction.ts:28,48` |
| 4 | 重试延迟为指数退避 `2^attempt * 100ms` | `transaction.ts:29,34` |
| 5 | 重试延迟序列: 100ms, 200ms, 400ms | `transaction.ts:34`（attempt 从 0 到 2） |
| 6 | PostgreSQL 死锁（40P01）可重试 | `transaction.ts:89` |
| 7 | MySQL 死锁（ER_LOCK_DEADLOCK）可重试 | `transaction.ts:86` |
| 8 | Redshift 不支持任何重试 | `transaction.ts:90` |
| 9 | 重试时整个 handler 完全重新执行 | `transaction.ts:41` |

### 8.2 事件相关

| # | 结论 | 源码证据 |
|---|------|---------|
| 10 | `emitFilter` 在 `transaction()` 的 handler 内部执行 | `items.ts:151-170` |
| 11 | `emitAction` 在 `transaction()` 调用之后执行 | `items.ts:388-419` |
| 12 | `emitFilter` 使用事务内连接 `trx` | `items.ts:165` |
| 13 | `emitAction` 使用新连接 `getDatabase()` | `items.ts:400` |
| 14 | `emitFilter` 是同步等待的（`await`） | `items.ts:156` |
| 15 | `emitAction` 内部使用 `emitAsync`，不 await | `emitter.ts:67` |
| 16 | `emitAction` 异常只记录日志，不抛出 | `emitter.ts:67-70` |
| 17 | `bypassEmitAction` 用于收集事件而非立即触发 | `items.ts:459` |
| 18 | 收集的事件在 `transaction()` 成功后统一触发 | `items.ts:479-487` |
| 19 | `emitEvents: false` 可禁用所有事件 | `items.ts:155,388` |

### 8.3 副作用相关

| # | 结论 | 源码证据 |
|---|------|---------|
| 20 | Filter 事件在重试时会重新执行 | `transaction.ts:41` + `items.ts:156` |
| 21 | Action 事件在重试时不会重新执行 | `items.ts:388`（在 transaction 之后） |
| 22 | 事务失败时 `emitAction` 代码不执行 | JavaScript 异常传播机制 + 代码位置 |
| 23 | 事务失败时 `cache.clear()` 不执行 | `items.ts:421`（在 transaction 之后） |
| 24 | Activity/Revision 在事务内写入，可回滚 | `items.ts:321-375`（在 transaction handler 内） |
| 25 | 收集的 `nestedActionEvents` 数组在异常时销毁 | JavaScript 作用域规则 + `items.ts:442` |

> **注意**: 本清单只包含可从源码直接验证的结论，不包含任何推断或假设。
