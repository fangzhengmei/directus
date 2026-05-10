# Directus 写入链路 Filter 与 Action 事务边界修订分析

## 核心发现：Create 与 Update/Delete 的 Filter 钩子事务边界不一致

经过重新核对代码，发现了一个**关键的设计差异**：

| 操作 | Filter 钩子位置 | 传入的 database 参数 | 事务边界 |
|------|----------------|---------------------|----------|
| **createOne** | 事务**内部** | `trx`（事务连接） | ✅ 参与事务 |
| **updateMany** | 事务**外部** | `this.knex`（普通连接） | ❌ 不参与事务 |
| **deleteMany** | 事务**外部** | `this.knex`（普通连接） | ❌ 不参与事务 |

---

## 1. 详细代码分析

### 1.1 createOne - Filter 钩子在事务内部

**位置**: `api/src/services/items.ts:151-170`

```typescript
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
    // ⚠️ Filter 钩子在事务内部执行
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
                        database: trx,        // ✅ 传入的是事务连接
                        schema: this.schema,
                        accountability: this.accountability,
                    },
                )
            : payload;
    
    // ... 后续数据库操作
});
```

**关键特征**:
- `emitFilter` 被包裹在 `transaction()` 的回调函数内
- 传入的 `database` 参数是 `trx`（当前事务连接）
- Filter 钩子中的数据库操作会参与同一事务
- Filter 钩子抛出异常会触发事务回滚

### 1.2 updateMany - Filter 钩子在事务外部

**位置**: `api/src/services/items.ts:730-747`

```typescript
// ⚠️ Filter 钩子在事务外部执行
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
                    database: this.knex,  // ❌ 传入的是普通连接（非事务）
                    schema: this.schema,
                    accountability: this.accountability,
                },
            )
        : payload;

// ... 权限验证 ...

await transaction(this.knex, async (trx) => {
    // 实际的数据库更新在事务内
    await trx(this.collection).update(payloadWithTypeCasting).whereIn(primaryKeyField, keys);
    // ...
});
```

**关键特征**:
- `emitFilter` 在 `transaction()` 调用**之前**执行
- 传入的 `database` 参数是 `this.knex`（普通数据库连接）
- Filter 钩子中的数据库操作**不参与**后续事务
- Filter 钩子抛出异常会阻止事务开始，但不会回滚钩子内的操作

### 1.3 deleteMany - Filter 钩子在事务外部

**位置**: `api/src/services/items.ts:1080-1096`

```typescript
// ⚠️ Filter 钩子在事务外部执行
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
                    database: this.knex,  // ❌ 传入的是普通连接（非事务）
                    schema: this.schema,
                    accountability: this.accountability,
                },
            )
        : keys;

// ... 权限验证 ...

await transaction(this.knex, async (trx) => {
    await trx(this.collection).whereIn(primaryKeyField, keysAfterHooks).delete();
    // ...
});
```

**关键特征**: 与 updateMany 一致，Filter 钩子在事务外部

---

## 2. 批量操作的影响分析

### 2.1 createMany - 继承 createOne 的行为

**位置**: `api/src/services/items.ts:433-477`

```typescript
async createMany(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    const { primaryKeys, nestedActionEvents } = await transaction(this.knex, async (knex) => {
        const service = this.fork({ knex });  // 用事务连接 fork 新服务
        
        for (const [index, payload] of data.entries()) {
            // 内部调用 createOne，Filter 钩子会使用 fork 后的事务连接
            const primaryKey = await service.createOne(payload, {
                ...
                bypassEmitAction: (params) => nestedActionEvents.push(params),
                ...
            });
        }
        // ...
    });
    
    // Action 钩子在事务提交后触发
    if (opts.emitEvents !== false) {
        for (const nestedActionEvent of nestedActionEvents) {
            emitter.emitAction(...);
        }
    }
}
```

**行为**:
- 外层 `transaction()` 创建一个事务
- `fork({ knex })` 创建的新 service 使用事务连接
- 内部 `createOne` 中的 Filter 钩子接收的是 `trx`（事务连接）
- 所有 create 操作的 Filter 钩子都参与同一个大事务

### 2.2 updateBatch - 继承 updateMany 的行为

**位置**: `api/src/services/items.ts:657-704`

```typescript
async updateBatch(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    try {
        await transaction(this.knex, async (knex) => {
            const service = this.fork({ knex });
            
            for (const index in data) {
                // 内部调用 updateOne → updateMany
                // 但 updateMany 的 Filter 钩子在 updateMany 自己的逻辑中，不在 transaction 内
                keys.push(await service.updateOne(primaryKey, omit(item, primaryKeyField), combinedOpts));
            }
        });
    } finally {
        // ...
    }
}
```

**关键问题**:
- 虽然外层有 `transaction()`，但 `updateMany` 内部的 Filter 钩子仍然在自己的事务外部
- `updateMany` 的代码结构是：
  ```
  updateMany() {
      emitFilter(...)  // 使用 this.knex，不是 trx
      transaction(this.knex, async (trx) => { ... })
  }
  ```
- 所以即使 `service.fork({ knex })` 传入了事务连接，Filter 钩子仍然不参与事务

### 2.3 upsertOne - 根据路径不同行为不同

**位置**: `api/src/services/items.ts:981-1002`

```typescript
async upsertOne(payload: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey> {
    // 检查是否存在
    const exists = ...;
    
    if (exists) {
        // 更新路径 - Filter 钩子在事务外部
        return await this.updateOne(primaryKey as PrimaryKey, data as Partial<Item>, opts);
    } else {
        // 创建路径 - Filter 钩子在事务内部
        return await this.createOne(payload, opts);
    }
}
```

**行为差异**:
- 同一集合的 upsert 操作，根据是创建还是更新，Filter 钩子的事务边界不同
- 这可能导致不一致的行为

---

## 3. 流程触发器（Flow Triggers）在不同路径下的行为

### 3.1 Filter 类型流程

**位置**: `api/src/flows.ts:196-213`

```typescript
if (flow.options['type'] === 'filter') {
    const handler: FilterHandler = (payload, meta, context) =>
        this.executeFlow(
            flow,
            { payload, ...meta },
            {
                accountability: context['accountability'],
                database: context['database'],  // 透传传入的 database
                getSchema: context['schema'] ? () => context['schema'] : getSchema,
            },
        );
    events.forEach((event) => emitter.onFilter(event, handler));
}
```

**行为分析**:

| 操作 | 传入的 database | 流程执行环境 | 异常影响 |
|------|----------------|-------------|----------|
| create | `trx`（事务） | ✅ 参与事务 | 触发事务回滚 |
| update | `this.knex`（普通） | ❌ 不参与事务 | 阻止事务开始，但不回滚流程内操作 |
| delete | `this.knex`（普通） | ❌ 不参与事务 | 阻止事务开始，但不回滚流程内操作 |

**关键差异示例**:

**场景 1: create 路径下的 Filter 流程**
```
用户请求 create
  ↓
transaction() 开始
  ↓
emitFilter() 触发流程
  ↓
流程内数据库操作 → 使用 trx，参与事务
  ↓
流程执行失败 → 抛出异常
  ↓
transaction() 捕获异常 → 回滚所有操作（包括流程内的）
  ↓
用户收到错误，数据库无变化
```

**场景 2: update 路径下的 Filter 流程**
```
用户请求 update
  ↓
emitFilter() 触发流程（在事务外）
  ↓
流程内数据库操作 → 使用 this.knex，独立连接
  ↓
流程内操作提交（自动提交模式）
  ↓
流程执行失败 → 抛出异常
  ↓
transaction() 未开始
  ↓
用户收到错误，但流程内的数据库操作已永久提交！
```

### 3.2 Action 类型流程

**位置**: `api/src/flows.ts:214-228`

```typescript
} else if (flow.options['type'] === 'action') {
    const handler: ActionHandler = (meta, context) =>
        this.executeFlow(flow, meta, {
            accountability: context['accountability'],
            database: getDatabase(),  // ⚠️ 总是使用新连接，忽略传入的 context
            getSchema: context['schema'] ? () => context['schema'] : getSchema,
        });
    events.forEach((event) => emitter.onAction(event, handler));
}
```

**行为一致性**:
- Action 类型流程**总是**使用 `getDatabase()` 获取新连接
- 不区分 create/update/delete 路径
- 总是在事务提交后异步执行
- 异常被捕获，不影响主流程

---

## 4. 自定义扩展在不同路径下的回滚与副作用影响

### 4.1 Filter 钩子扩展的事务边界差异

扩展通过 `ExtensionManager.registerHook()` 注册：

**位置**: `api/src/extensions/manager.ts:892-906`

```typescript
const hookRegistrationContext = {
    filter: <T = unknown>(event: string, handler: FilterHandler<T>) => {
        emitter.onFilter(event, handler);
        // ...
    },
    action: (event: string, handler: ActionHandler) => {
        emitter.onAction(event, handler);
        // ...
    },
};
```

扩展的 Filter 钩子接收的 `context.database` 完全取决于调用路径：

#### Create 路径 - 完全事务保护

```typescript
// 扩展代码示例
register.filter('items.create', async (payload, meta, context) => {
    const { database } = context;  // 这是 trx（事务连接）
    
    // 扩展内的数据库操作参与事务
    await database('audit_log').insert({
        action: 'create',
        collection: meta.collection,
        data: JSON.stringify(payload),
    });
    
    // 如果这里抛出异常，事务回滚，audit_log 也会被回滚
    if (payload.invalid) {
        throw new Error('Validation failed');  // 触发完整回滚
    }
    
    return payload;
});
```

**保证**:
- ✅ 扩展内数据库操作参与事务
- ✅ 异常触发完整回滚
- ✅ 数据一致性有保障

#### Update/Delete 路径 - 无事务保护

```typescript
// 扩展代码示例
register.filter('items.update', async (payload, meta, context) => {
    const { database } = context;  // 这是 this.knex（普通连接）
    
    // 扩展内的数据库操作使用独立连接，自动提交
    await database('audit_log').insert({
        action: 'update',
        collection: meta.collection,
        data: JSON.stringify(payload),
    });  // ⚠️ 这里已经提交了！
    
    // 后续验证失败
    if (payload.invalid) {
        throw new Error('Validation failed');  // 阻止更新，但 audit_log 已永久写入！
    }
    
    return payload;
});
```

**风险**:
- ❌ 扩展内数据库操作**不参与**后续事务
- ❌ 异常阻止主操作，但扩展内操作已提交
- ⚠️ **可能造成数据不一致**

### 4.2 Action 钩子扩展的一致性

Action 钩子的行为在所有路径下是一致的：

**位置**: `api/src/emitter.ts:62-72`

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

**特征**:
- 总是使用 `emitAsync` 异步执行
- 异常被 `.catch()` 捕获，仅记录日志
- 不影响主流程
- 数据库连接取决于 ItemsService 传入的 context（都是 `getDatabase()` 新连接）

### 4.3 副作用一致性分析

| 副作用类型 | Create Filter | Update/Delete Filter | 所有 Action |
|-----------|---------------|---------------------|-------------|
| 扩展内数据库操作 | ✅ 参与事务，可回滚 | ❌ 独立提交，不可回滚 | ❌ 独立提交，不可回滚 |
| 外部 API 调用 | ⚠️ 不会随事务回滚 | ⚠️ 不会随事务回滚 | ⚠️ 不会随事务回滚 |
| 文件系统操作 | ⚠️ 不会随事务回滚 | ⚠️ 不会随事务回滚 | ⚠️ 不会随事务回滚 |
| 缓存操作 | ⚠️ 不会随事务回滚 | ⚠️ 不会随事务回滚 | ✅ 事务后执行，通常安全 |

---

## 5. 完整事务边界对比表

### 5.1 Filter 钩子对比

| 维度 | createOne/createMany | updateMany/updateBatch/deleteMany |
|------|---------------------|-----------------------------------|
| **执行位置** | `transaction()` 回调内 | `transaction()` 调用前 |
| **database 参数** | `trx`（事务连接） | `this.knex`（普通连接） |
| **能否参与事务** | ✅ 能 | ❌ 不能 |
| **钩子内 DB 操作** | 与主操作同一事务 | 独立连接，自动提交 |
| **异常对主操作影响** | 回滚主操作 | 阻止主操作开始 |
| **异常对钩子内操作影响** | 回滚钩子内操作 | ❌ 钩子内操作已提交 |
| **事务重试时重新执行** | ✅ 每次重试都执行 | ⚠️ 只执行一次（重试不影响） |

### 5.2 Action 钩子对比（所有路径一致）

| 维度 | 行为 |
|------|------|
| **执行位置** | `transaction()` 提交后 |
| **database 参数** | `getDatabase()` 新连接 |
| **能否参与事务** | ❌ 不能（事务已提交） |
| **异常对主操作影响** | ❌ 无影响（已提交） |
| **执行方式** | `emitAsync` 异步，不等待 |
| **异常处理** | `.catch()` 捕获，仅记录日志 |

---

## 6. 实际风险场景分析

### 场景 1: Update 路径下的审计日志

```typescript
// 扩展代码 - 有风险！
register.filter('orders.update', async (payload, meta, context) => {
    const { database } = context;
    
    // 记录变更前状态（查询操作，影响不大）
    const before = await database('orders').where('id', meta.keys[0]).first();
    
    // 写入审计日志 - ⚠️ 这里已经提交了！
    await database('audit_log').insert({
        table_name: 'orders',
        record_id: meta.keys[0],
        action: 'update',
        before_data: JSON.stringify(before),
        after_data: JSON.stringify(payload),
    });
    
    // 后续业务验证
    if (payload.status === 'cancelled' && before.paid) {
        // 已支付订单不能取消
        throw new Error('Paid order cannot be cancelled');
    }
    
    return payload;
});
```

**问题**:
- 用户尝试取消已支付订单
- 审计日志已写入数据库
- 验证失败抛出异常
- 订单未更新，但审计日志显示"已更新"
- **数据不一致**

### 场景 2: Create 路径下的库存扣减

```typescript
// 扩展代码 - 相对安全
register.filter('order_items.create', async (payload, meta, context) => {
    const { database } = context;
    
    // 扣减库存 - 参与同一事务
    await database('inventory')
        .where('product_id', payload.product_id)
        .decrement('stock', payload.quantity);
    
    // 检查库存是否足够
    const inventory = await database('inventory')
        .where('product_id', payload.product_id)
        .first();
    
    if (inventory.stock < 0) {
        throw new Error('Insufficient stock');  // 触发回滚，库存扣减也被撤销
    }
    
    return payload;
});
```

**优势**:
- 库存扣减与订单创建在同一事务
- 库存不足时，扣减操作被回滚
- **数据一致性有保障**

### 场景 3: 批量操作中的混合行为

```typescript
// 用户调用 upsertMany
await itemsService.upsertMany([
    { id: 1, name: 'Existing Item Updated' },  // 走 update 路径
    { name: 'New Item Created' },              // 走 create 路径
]);
```

**行为**:
- 第一条记录（update）: Filter 钩子在事务外，独立连接
- 第二条记录（create）: Filter 钩子在事务内，参与事务
- 如果第二条的 Filter 钩子失败并回滚:
  - 第二条的所有操作被回滚
  - 第一条的 Filter 钩子内的操作**不会**被回滚（已独立提交）
- **潜在不一致**

---

## 7. 设计意图与可能原因

### 7.1 为什么 create 与 update/delete 不一致？

可能的设计考量：

1. **创建时需要事务内验证**:
   - 创建操作可能需要检查关联数据、生成序列号等
   - 这些操作需要在同一事务内，避免竞态条件

2. **更新时需要事务外预览**:
   - 更新操作可能需要读取当前数据进行对比
   - 在事务外执行可以避免长事务
   - 但代价是失去了事务保护

3. **历史遗留**:
   - 可能是不同时期由不同开发者实现
   - 缺乏统一的设计规范

### 7.2 代码中的证据

查看 `createOne` 和 `updateMany` 的注释：

**createOne** (`api/src/services/items.ts:145-150`):
```typescript
/**
 * By wrapping the logic in a transaction, we make sure we automatically roll back all the
 * changes in the DB if any of the parts contained within throws an error. This also means
 * that any errors thrown in any nested relational changes will bubble up and cancel the whole
 * update tree
 */
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
```

注释明确说明"所有部分"包括在事务内，暗示 Filter 钩子应该是这些"部分"之一。

**updateMany** (`api/src/services/items.ts:728-729`):
```typescript
// Run all hooks that are attached to this event so the end user has the chance to augment the
// item that is about to be saved
```

注释只说"修改即将保存的数据"，没有提到事务参与。

---

## 8. 最佳实践建议

### 8.1 对于 Filter 钩子开发

#### 在 create 路径下（相对安全）
```typescript
register.filter('items.create', async (payload, meta, context) => {
    const { database } = context;
    
    // 可以安全地进行数据库操作
    await database('related_table').insert({ ... });
    
    // 验证失败时，所有操作会回滚
    if (!isValid(payload)) {
        throw new Error('Validation failed');
    }
    
    return payload;
});
```

#### 在 update/delete 路径下（需谨慎）
```typescript
register.filter('items.update', async (payload, meta, context) => {
    const { database } = context;
    
    // 1. 只进行只读操作
    const existing = await database('items').where('id', meta.keys[0]).first();
    
    // 2. 所有验证放在写入操作之前
    if (!isValidUpdate(existing, payload)) {
        throw new Error('Validation failed');  // 先验证，再考虑写入
    }
    
    // 3. 如果必须写入，使用补偿机制
    // 或者考虑改用 Action 钩子
    
    return payload;
});
```

### 8.2 副作用处理策略

| 操作类型 | 推荐钩子类型 | 理由 |
|---------|-------------|------|
| 数据验证 | Filter | 需要阻止无效操作 |
| 数据转换 | Filter | 需要在写入前修改 |
| 审计日志（关键） | Filter（仅 create） | 需要事务一致性 |
| 审计日志（非关键） | Action | 容忍最终一致 |
| 外部 API 调用 | Action | 失败不影响主流程 |
| 缓存清除 | Action | 事务后执行更合理 |
| 通知发送 | Action | 异步，不阻塞请求 |

### 8.3 确保一致性的模式

#### 模式 1: 两阶段验证（Update 路径）
```typescript
register.filter('items.update', async (payload, meta, context) => {
    // 第一阶段：只读验证（在事务外）
    const { database } = context;
    const existing = await database('items').whereIn('id', meta.keys);
    
    for (const item of existing) {
        if (!canUpdate(item, payload)) {
            throw new Error('Validation failed');
        }
    }
    
    // 验证通过，允许操作继续
    // 注意：此时还没写入任何东西
    return payload;
});

// 副作用放在 Action 钩子中
register.action('items.update', async (meta, context) => {
    const { database } = context;
    // 现在可以安全写入审计日志等
    await database('audit_log').insert({ ... });
});
```

#### 模式 2: 仅在 create 中使用事务依赖逻辑
```typescript
// 对于需要强一致性的操作，只在 create 路径实现
register.filter('items.create', async (payload, meta, context) => {
    const { database } = context;
    
    // 事务内的原子操作
    await database('counter').increment('value');
    const counter = await database('counter').first();
    
    payload.sequence_number = counter.value;
    
    return payload;
});

// update 路径使用其他机制（如乐观锁）
register.filter('items.update', async (payload, meta, context) => {
    // 使用版本号而非事务依赖
    if (!payload.version) {
        throw new Error('Version required');
    }
    return payload;
});
```

---

## 9. 代码位置索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| createOne Filter 钩子 | `api/src/services/items.ts` | 151-170 |
| updateMany Filter 钩子 | `api/src/services/items.ts` | 730-747 |
| deleteMany Filter 钩子 | `api/src/services/items.ts` | 1080-1096 |
| createMany 批量处理 | `api/src/services/items.ts` | 433-477 |
| updateBatch 批量处理 | `api/src/services/items.ts` | 657-704 |
| upsertOne 分支逻辑 | `api/src/services/items.ts` | 981-1002 |
| Emitter.emitFilter | `api/src/emitter.ts` | 34-60 |
| Emitter.emitAction | `api/src/emitter.ts` | 62-72 |
| 流程 Filter 触发器 | `api/src/flows.ts` | 196-213 |
| 流程 Action 触发器 | `api/src/flows.ts` | 214-228 |
| 扩展注册 Filter | `api/src/extensions/manager.ts` | 892-899 |
| 扩展注册 Action | `api/src/extensions/manager.ts` | 900-906 |
| 事务工具函数 | `api/src/utils/transaction.ts` | 14-52 |

---

## 10. 修订版核心结论

### 10.1 主要发现

1. **Filter 钩子的事务边界不一致**是本分析最重要的发现
   - create 路径：Filter 钩子在事务内，传入 `trx`
   - update/delete 路径：Filter 钩子在事务外，传入 `this.knex`

2. **这种不一致导致行为差异**
   - create 路径：Filter 钩子内的数据库操作可回滚
   - update/delete 路径：Filter 钩子内的数据库操作独立提交，可能造成不一致

3. **批量操作继承各自的行为**
   - createMany：所有子操作的 Filter 钩子参与同一大事务
   - updateBatch：子操作的 Filter 钩子仍在各自事务外

4. **流程触发器的行为取决于触发路径**
   - Filter 类型流程：create 路径参与事务，update/delete 路径不参与
   - Action 类型流程：所有路径行为一致，总是异步、无事务

5. **自定义扩展需要路径感知**
   - 开发者需要知道自己的钩子在哪个路径下执行
   - 不同路径需要不同的错误处理和一致性策略

### 10.2 对开发者的建议

1. **假设 Filter 钩子不在事务内**：除非明确知道是 create 路径
2. **在 Filter 钩子中最小化副作用**：特别是 update/delete 路径
3. **关键副作用使用 Action 钩子**：虽然不参与事务，但行为一致
4. **考虑使用补偿机制**：对于 update/delete 路径的 Filter 钩子
5. **测试不同路径**：确保 create 和 update/delete 下行为符合预期
