# Directus 钩子、流程触发器与自定义扩展分析

## 1. 数据写入事件监听机制

### 1.1 核心服务层事件触发

Directus 的数据写入操作主要通过 `ItemsService` 类处理，位于 `api/src/services/items.ts`。该类实现了完整的 CRUD 操作，并在关键节点触发事件。

#### 创建操作 (createOne)
- **位置**: `api/src/services/items.ts:127-426`
- **事件触发流程**:
  1. **Filter 钩子** (`items.create` / `${collection}.items.create`): 在事务开始前执行，用于修改即将保存的数据
  2. **事务包装**: 使用 `transaction()` 包装所有数据库操作
  3. **Action 钩子**: 在事务成功提交后执行

#### 更新操作 (updateMany)
- **位置**: `api/src/services/items.ts:709-974`
- **事件触发流程**:
  1. **Filter 钩子** (`items.update` / `${collection}.items.update`): 在事务外执行
  2. **权限验证**: 验证用户是否有权限执行更新
  3. **事务包装**: 包含数据更新、关系处理、活动记录
  4. **Action 钩子**: 事务提交后触发

#### 删除操作 (deleteMany)
- **位置**: `api/src/services/items.ts:1070-1185`
- **事件触发流程**:
  1. **Filter 钩子** (`items.delete` / `${collection}.items.delete`): 在事务外执行
  2. **权限验证**
  3. **事务包装**: 包含删除操作和活动记录
  4. **Action 钩子**: 事务提交后触发

### 1.2 事件系统架构

#### Emitter 类
- **位置**: `api/src/emitter.ts`
- **两种核心事件类型**:

##### Filter 钩子
```typescript
// 定义: packages/types/src/events.ts:12-16
export type FilterHandler<T = unknown> = (
    payload: T,
    meta: Record<string, any>,
    context: EventContext,
) => T | Promise<T>;
```

- **执行方式**: 同步/异步执行 (`await listener(...)`)
- **特点**:
  - 可以修改传入的 payload
  - 支持链式处理（多个 filter 依次执行）
  - 抛出异常会中断操作
  - 接收事务上下文（可参与同一事务）

##### Action 钩子
```typescript
// 定义: packages/types/src/events.ts:17
export type ActionHandler = (meta: Record<string, any>, context: EventContext) => void;
```

- **执行方式**: 异步执行 (`emitAsync`, 不等待)
- **特点**:
  - 不返回值，也不修改数据
  - 异常被捕获并记录日志，不会影响主流程
  - 在事务提交后执行
  - 使用新的数据库连接（不参与原事务）

### 1.3 流程触发器集成

#### FlowManager 类
- **位置**: `api/src/flows.ts`
- **事件触发器类型**:

##### Filter 类型流程
```typescript
// api/src/flows.ts:196-213
if (flow.options['type'] === 'filter') {
    const handler: FilterHandler = (payload, meta, context) =>
        this.executeFlow(flow, { payload, ...meta }, {
            accountability: context['accountability'],
            database: context['database'],  // 使用传入的事务连接
            getSchema: context['schema'] ? () => context['schema'] : getSchema,
        });
    events.forEach((event) => emitter.onFilter(event, handler));
}
```

##### Action 类型流程
```typescript
// api/src/flows.ts:214-228
} else if (flow.options['type'] === 'action') {
    const handler: ActionHandler = (meta, context) =>
        this.executeFlow(flow, meta, {
            accountability: context['accountability'],
            database: getDatabase(),  // 使用新的数据库连接
            getSchema: context['schema'] ? () => context['schema'] : getSchema,
        });
    events.forEach((event) => emitter.onAction(event, handler));
}
```

### 1.4 自定义扩展机制

#### ExtensionManager 类
- **位置**: `api/src/extensions/manager.ts`
- **Hook 扩展注册**:

```typescript
// api/src/extensions/manager.ts:885-972
private registerHook(hookRegistrationCallback: HookConfig, name: string): PromiseCallback[] {
    const hookRegistrationContext = {
        filter: <T = unknown>(event: string, handler: FilterHandler<T>) => {
            emitter.onFilter(event, handler);
            // ... 注册清理函数
        },
        action: (event: string, handler: ActionHandler) => {
            emitter.onAction(event, handler);
            // ... 注册清理函数
        },
        // ... 其他钩子类型
    };

    hookRegistrationCallback(hookRegistrationContext, {
        services,
        env,
        database: getDatabase(),
        emitter: this.localEmitter,
        logger,
        getSchema,
    });
}
```

## 2. 同步钩子与异步操作的执行边界

### 2.1 事务边界分析

#### 事务包装机制
- **位置**: `api/src/utils/transaction.ts`
- **核心实现**:

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
            // 支持特定数据库的重试机制
            // ...
        }
    }
};
```

#### Filter 钩子的执行时机

以 `createOne` 为例 (`api/src/services/items.ts:151-170`):

```typescript
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
    // Filter 钩子在事务内部执行
    const payloadAfterHooks =
        opts.emitEvents !== false
            ? await emitter.emitFilter(
                    this.eventScope === 'items'
                        ? ['items.create', `${this.collection}.items.create`]
                        : `${this.eventScope}.create`,
                    payload,
                    { collection: this.collection },
                    {
                        database: trx,  // 传入事务连接
                        schema: this.schema,
                        accountability: this.accountability,
                    },
                )
            : payload;
    // ... 后续数据库操作
});
```

**关键点**:
- Filter 钩子在事务内部执行
- 接收的 `database` 参数是事务连接 (`trx`)
- Filter 钩子中的异常会导致事务回滚
- Filter 钩子可以修改 payload，影响后续数据写入

#### Action 钩子的执行时机

```typescript
// api/src/services/items.ts:388-419
if (opts.emitEvents !== false) {
    const actionEvent = {
        event: [...],
        meta: {...},
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
}
```

**关键点**:
- Action 钩子在事务成功提交后执行
- 接收的 `database` 参数是新的数据库连接（不是事务）
- 使用 `emitAsync` 异步执行，不等待完成
- Action 钩子的异常不会影响原事务

### 2.2 执行边界总结

| 特性 | Filter 钩子 | Action 钩子 |
|------|-------------|-------------|
| 执行时机 | 事务内部（数据写入前/中） | 事务提交后 |
| 数据库连接 | 参与原事务 (`trx`) | 新连接 (`getDatabase()`) |
| 执行方式 | 同步等待 (`await`) | 异步执行 (`emitAsync`) |
| 能否修改数据 | 可以（返回修改后的 payload） | 不可以 |
| 异常影响 | 中断操作，触发事务回滚 | 仅记录日志，不影响主流程 |
| 流程触发器支持 | Filter 类型流程 | Action 类型流程 |
| 自定义扩展支持 | `register.filter()` | `register.action()` |

## 3. 扩展失败对数据一致性的影响

### 3.1 Filter 钩子失败场景

#### 情况 1: 钩子抛出异常
```typescript
// Filter 钩子示例
register.filter('items.create', async (payload, meta, context) => {
    if (payload.name === 'invalid') {
        throw new Error('Invalid name');  // 抛出异常
    }
    return payload;
});
```

**影响**:
- 异常向上传播
- `transaction()` 捕获异常并触发回滚
- 数据库操作全部撤销
- 请求返回错误响应

#### 情况 2: 流程触发器执行失败
```typescript
// api/src/flows.ts:493-495
if (flow.trigger === 'event' && flow.options['type'] === 'filter' && lastOperationStatus === 'reject') {
    throw keyedData[LAST_KEY];  // 抛出错误数据
}
```

**影响**:
- 流程执行失败（reject 状态）
- 异常被抛出
- 事务回滚
- 数据不会写入

### 3.2 Action 钩子失败场景

#### Emitter 中的异常处理
```typescript
// api/src/emitter.ts:62-72
public emitAction(event: string | string[], meta: Record<string, any>, context: EventContext | null = null): void {
    const logger = useLogger();
    const events = Array.isArray(event) ? event : [event];

    for (const event of events) {
        this.actionEmitter.emitAsync(event, { event, ...meta }, context ?? this.getDefaultContext()).catch((err) => {
            logger.warn(`An error was thrown while executing action "${event}"`);
            logger.warn(err);  // 仅记录日志
        });
    }
}
```

**影响**:
- 异常被 `.catch()` 捕获
- 仅记录警告日志
- 不影响主流程
- 数据已成功写入

#### Action 类型流程失败
```typescript
// api/src/flows.ts:214-220
const handler: ActionHandler = (meta, context) =>
    this.executeFlow(flow, meta, {
        accountability: context['accountability'],
        database: getDatabase(),  // 新连接
        getSchema: context['schema'] ? () => context['schema'] : getSchema,
    });
```

**影响**:
- 流程失败被 `emitAction` 的 `.catch()` 捕获
- 原事务已提交，数据已存在
- 可能导致副作用不一致（如外部 API 调用失败）

### 3.3 事务重试机制

#### 自动重试条件
```typescript
// api/src/utils/transaction.ts:54-99
function shouldRetryTransaction(client: DatabaseClient, error: unknown): boolean {
    const COCKROACH_RETRY_ERROR_CODE = '40001';
    const SQLITE_BUSY_ERROR_CODE = 'SQLITE_BUSY';
    const MYSQL_DEADLOCK_CODE = 'ER_LOCK_DEADLOCK';
    const POSTGRES_DEADLOCK_CODE = '40P01';
    const ORACLE_DEADLOCK_CODE = 'ORA-00060';
    const MSSQL_DEADLOCK_CODE = 'EREQUEST';
    // ...
}
```

**重试场景**:
- CockroachDB: 事务冲突 (40001)
- SQLite: 数据库忙 (SQLITE_BUSY)
- MySQL/MariaDB: 死锁 (ER_LOCK_DEADLOCK)
- PostgreSQL: 死锁 (40P01)
- Oracle: 死锁 (ORA-00060)
- MSSQL: 死锁 (EREQUEST, number=1205)

**重试策略**:
- 最多重试 3 次
- 指数退避延迟 (100ms, 200ms, 400ms)
- Filter 钩子会在每次重试时重新执行

### 3.4 扩展加载失败处理

```typescript
// api/src/extensions/manager.ts:1032-1044
private handleExtensionError({ error, reason }: { error?: unknown; reason: string }): void {
    const logger = useLogger();

    if (toBoolean(env['EXTENSIONS_MUST_LOAD'])) {
        logger.error('EXTENSION_MUST_LOAD is enabled and an extension failed to load.');
        logger.error(reason);
        if (error) logger.error(error);
        process.exit(1);  // 强制退出
    } else {
        logger.warn(reason);
        if (error) logger.warn(error);  // 仅警告，继续运行
    }
}
```

**影响**:
- `EXTENSIONS_MUST_LOAD=true`: 扩展加载失败导致进程退出
- `EXTENSIONS_MUST_LOAD=false`（默认）: 仅记录警告，其他功能正常

## 4. 数据一致性保证与风险

### 4.1 保证一致性的场景

#### Filter 钩子
- ✅ 钩子在事务内执行
- ✅ 异常导致事务回滚
- ✅ 可以验证和修改数据
- ✅ 流程触发器的 filter 类型同样受事务保护

#### 核心数据库操作
- ✅ 所有写入操作包装在 `transaction()` 中
- ✅ 嵌套操作使用同一事务连接
- ✅ 活动记录和修订记录也在同一事务中
- ✅ 支持数据库特定的死锁重试

### 4.2 潜在一致性风险

#### Action 钩子的副作用
- ❌ 在事务提交后执行
- ❌ 失败不会影响已提交的数据
- ❌ 异步执行，无法保证顺序
- ❌ 可能导致"数据已写入但副作用未完成"的情况

**示例风险**:
```typescript
register.action('items.create', async (meta, context) => {
    // 发送通知邮件 - 可能失败
    await sendEmail(meta.payload);
    
    // 调用外部 API - 可能超时
    await externalApi.create(meta.payload);
});
```

如果邮件发送或 API 调用失败：
- 数据已成功写入数据库
- 但外部系统未同步
- 造成数据不一致

#### Filter 钩子中的非事务操作
- ❌ Filter 钩子内的外部操作不会随事务回滚

**示例风险**:
```typescript
register.filter('items.create', async (payload, meta, context) => {
    // 外部操作 - 不会回滚
    await externalApi.preprocess(payload.id);
    
    // 后续数据库操作失败
    throw new Error('DB error');  // 事务回滚，但外部 API 已调用
});
```

### 4.3 最佳实践建议

#### 对于关键业务逻辑
- **使用 Filter 钩子**: 在事务内验证和处理数据
- **保持幂等性**: 考虑事务重试的情况
- **避免外部副作用**: 不要在 Filter 钩子中调用外部 API

#### 对于非关键副作用
- **使用 Action 钩子**: 不影响主事务
- **实现补偿机制**: 考虑失败后的重试或回滚策略
- **记录详细日志**: 便于问题排查

#### 对于流程设计
- **Filter 类型流程**: 用于数据验证、权限检查、数据转换
- **Action 类型流程**: 用于通知、索引更新、缓存清除等
- **避免长耗时操作**: 特别是 Filter 类型，会阻塞请求

#### 扩展错误处理
- **设置合理的 `EXTENSIONS_MUST_LOAD`**: 生产环境建议启用
- **实现健壮的错误处理**: 在自定义扩展中捕获和处理异常
- **监控扩展执行**: 关注日志中的警告和错误

## 5. 代码位置索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| ItemsService.createOne | `api/src/services/items.ts` | 127-426 |
| ItemsService.updateMany | `api/src/services/items.ts` | 709-974 |
| ItemsService.deleteMany | `api/src/services/items.ts` | 1070-1185 |
| Emitter 实现 | `api/src/emitter.ts` | 1-120 |
| 事务工具 | `api/src/utils/transaction.ts` | 1-99 |
| 流程管理器 | `api/src/flows.ts` | 1-601 |
| 扩展管理器 | `api/src/extensions/manager.ts` | 1-1045 |
| 事件类型定义 | `packages/types/src/events.ts` | 1-20 |
| Hook 扩展类型 | `packages/types/src/extensions/hooks.ts` | 1-16 |
