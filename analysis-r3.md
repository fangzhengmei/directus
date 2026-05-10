# Directus 写入链路 Filter 与 Action 事务边界修订分析（R3）

## 核心修正：事务边界取决于调用路径，而非操作类型

经过重新核对代码，发现之前的分析存在重要偏差。**Filter 钩子的事务边界不是由操作类型（create/update/delete）决定的，而是由调用路径决定的**。

关键机制：
1. `transaction()` 函数检查 `knex.isTransaction`，避免嵌套事务
2. `fork({ knex })` 可以传递事务上下文到子服务
3. 批量方法（createMany/updateBatch/upsertMany）通过 `fork({ knex })` 传递事务连接

---

## 1. 关键机制分析

### 1.1 transaction() 的嵌套事务检查

**位置**: `api/src/utils/transaction.ts:14-52`

```typescript
export const transaction = async <T = unknown>(
    knex: Knex,
    handler: (knex: Knex.Transaction) => Promise<T>,
): Promise<T> => {
    if (knex.isTransaction) {
        // ⚠️ 如果已经在事务中，直接执行，不创建新事务
        return handler(knex as Knex.Transaction);
    } else {
        try {
            return await knex.transaction((trx) => handler(trx));
        } catch (error) {
            // ... 重试逻辑
        }
    }
};
```

**关键行为**:
- 如果传入的 `knex` 已经是事务（`isTransaction === true`），直接执行 handler
- 这意味着"嵌套"的 `transaction()` 调用实际上是加入同一个外层事务
- 只有最外层的 `transaction()` 调用才会真正创建事务

### 1.2 fork() 方法的上下文传递

**位置**: `api/src/services/items.ts:68-88`

```typescript
private fork(options?: Partial<AbstractServiceOptions>): ItemsService<AnyItem> {
    const Service = this.constructor;
    const isItemsService = Service.length === 2;

    const newOptions = {
        knex: this.knex,        // 默认继承当前的 knex
        accountability: this.accountability,
        schema: this.schema,
        nested: this.nested,
        ...options,              // 可以覆盖 knex
    };

    if (isItemsService) {
        return new ItemsService(this.collection, newOptions);
    }
    return new (Service as new (options: AbstractServiceOptions) => this)(newOptions);
}
```

**关键行为**:
- `fork({ knex: trx })` 会创建一个新的 service 实例，其 `this.knex` 被设置为传入的事务连接
- 新 service 的所有方法都使用这个事务连接

### 1.3 ItemsService 的构造函数

**位置**: `api/src/services/items.ts:53-63`

```typescript
constructor(collection: Collection, options: AbstractServiceOptions) {
    this.collection = collection;
    this.knex = options.knex || getDatabase();  // 使用传入的 knex 或获取新连接
    this.accountability = options.accountability || null;
    // ...
}
```

---

## 2. 调用路径与事务边界分析

### 2.1 路径一：单独调用（无外层事务）

#### 场景 A: 单独调用 createOne

```typescript
// 调用方式
const service = new ItemsService('articles', { schema, accountability });
await service.createOne({ title: 'Test' });
```

**执行流程**:

```
createOne(data)
  ↓
transaction(this.knex, async (trx) => {   // this.knex 是普通连接 → 创建新事务
  ↓
  emitter.emitFilter(
    ...,
    { database: trx, ... }  // ✅ 传入事务连接
  )
  ↓
  // 数据库操作使用 trx
})
```

**Filter 钩子状态**: ✅ 参与事务

#### 场景 B: 单独调用 updateMany

```typescript
// 调用方式
const service = new ItemsService('articles', { schema, accountability });
await service.updateMany([1, 2], { status: 'published' });
```

**执行流程**:

```
updateMany(keys, data)
  ↓
emitter.emitFilter(
  ...,
  { database: this.knex, ... }  // ❌ this.knex 是普通连接
)
  ↓
transaction(this.knex, async (trx) => {   // 创建新事务
  ↓
  // 数据库操作使用 trx
})
```

**Filter 钩子状态**: ❌ 不参与事务（在事务外执行）

#### 场景 C: 单独调用 deleteMany

```typescript
// 调用方式
const service = new ItemsService('articles', { schema, accountability });
await service.deleteMany([1, 2]);
```

**执行流程**：与 updateMany 相同

**Filter 钩子状态**: ❌ 不参与事务

---

### 2.2 路径二：批量调用（有外层事务 + fork）

#### 场景 D: 调用 createMany

**位置**: `api/src/services/items.ts:433-494`

```typescript
async createMany(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    const { primaryKeys, nestedActionEvents } = await transaction(this.knex, async (knex) => {
        // knex 是事务连接
        const service = this.fork({ knex });  // ⚠️ fork 传递事务连接
        
        for (const [index, payload] of data.entries()) {
            // 内部调用 createOne
            const primaryKey = await service.createOne(payload, { ... });
        }
        // ...
    });
    // ...
}
```

**执行流程**:

```
createMany(data)
  ↓
transaction(this.knex, async (trx) => {   // 创建外层事务
  ↓
  const service = this.fork({ knex: trx })  // 新 service 的 this.knex = trx
  ↓
  service.createOne(payload)
    ↓
    transaction(this.knex, async (innerTrx) => {
      // this.knex 是 trx，且 trx.isTransaction === true
      // ⚠️ 直接执行，不创建新事务！
      ↓
      emitter.emitFilter(
        ...,
        { database: innerTrx, ... }  // innerTrx 就是 trx
      )
      ↓
      // 数据库操作使用 trx
    })
})
```

**Filter 钩子状态**: ✅ 参与外层事务

#### 场景 E: 调用 updateBatch

**位置**: `api/src/services/items.ts:657-704`

```typescript
async updateBatch(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    try {
        await transaction(this.knex, async (knex) => {
            // knex 是事务连接
            const service = this.fork({ knex });  // ⚠️ fork 传递事务连接
            
            for (const index in data) {
                // 内部调用 updateOne → updateMany
                keys.push(await service.updateOne(primaryKey, omit(item, primaryKeyField), combinedOpts));
            }
            // ...
        });
    } finally {
        // ...
    }
}
```

**执行流程**:

```
updateBatch(data)
  ↓
transaction(this.knex, async (trx) => {   // 创建外层事务
  ↓
  const service = this.fork({ knex: trx })  // 新 service 的 this.knex = trx
  ↓
  service.updateOne(key, data)
    ↓
    service.updateMany([key], data)
      ↓
      emitter.emitFilter(
        ...,
        { database: this.knex, ... }  // ⚠️ this.knex 是 trx（事务连接）！
      )
      ↓
      transaction(this.knex, async (innerTrx) => {
        // this.knex 是 trx，且 trx.isTransaction === true
        // ⚠️ 直接执行，不创建新事务！
        ↓
        // 数据库操作使用 trx
      })
})
```

**关键修正**：
- 之前的分析错误地认为 updateMany 的 filter 钩子总是在事务外
- 实际上，当通过 updateBatch 调用时：
  1. `fork({ knex: trx })` 使 service 的 `this.knex = trx`
  2. updateMany 中的 `emitFilter` 使用 `this.knex`（即 trx）
  3. updateMany 内部的 `transaction(this.knex, ...)` 发现 `this.knex.isTransaction === true`，直接执行
- **Filter 钩子实际上参与外层事务**

**Filter 钩子状态**: ✅ 参与外层事务

#### 场景 F: 调用 upsertMany

**位置**: `api/src/services/items.ts:1010-1038`

```typescript
async upsertMany(payloads: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    const primaryKeys = await transaction(this.knex, async (knex) => {
        const service = this.fork({ knex });  // ⚠️ fork 传递事务连接
        
        for (const index in payloads) {
            const primaryKey = await service.upsertOne(payload, { ... });
        }
        // ...
    });
    // ...
}
```

**upsertOne 的分支逻辑** (`api/src/services/items.ts:981-1002`):

```typescript
async upsertOne(payload: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey> {
    const exists = ...; // 检查是否存在
    
    if (exists) {
        // 更新路径
        return await this.updateOne(primaryKey as PrimaryKey, data as Partial<Item>, opts);
    } else {
        // 创建路径
        return await this.createOne(payload, opts);
    }
}
```

**执行流程**:

```
upsertMany(payloads)
  ↓
transaction(this.knex, async (trx) => {   // 创建外层事务
  ↓
  const service = this.fork({ knex: trx })  // 新 service 的 this.knex = trx
  ↓
  service.upsertOne(payload)
    ↓
    if (exists) {
      // 更新路径
      service.updateOne(...) → updateMany(...)
        ↓
        emitter.emitFilter(..., { database: this.knex, ... })  // this.knex = trx ✅
        ↓
        transaction(this.knex, ...)  // 直接执行，不新建事务
    } else {
      // 创建路径
      service.createOne(...)
        ↓
        transaction(this.knex, ...)  // 直接执行，不新建事务
          ↓
          emitter.emitFilter(..., { database: trx, ... })  // ✅
    }
})
```

**Filter 钩子状态**: 
- 更新路径：✅ 参与外层事务
- 创建路径：✅ 参与外层事务

---

## 3. 完整事务边界对照表

### 3.1 按调用路径分类

| 调用路径 | 外层事务 | fork 传递 | Filter 钩子 database | 事务边界 |
|---------|---------|----------|---------------------|----------|
| **单独调用 createOne** | 自己创建 | 无 | `trx`（事务内传入） | ✅ 参与事务 |
| **单独调用 updateMany** | 自己创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |
| **单独调用 deleteMany** | 自己创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |
| **调用 createMany** | createMany 创建 | ✅ `fork({ knex: trx })` | `trx`（createOne 内传入） | ✅ 参与外层事务 |
| **调用 updateBatch** | updateBatch 创建 | ✅ `fork({ knex: trx })` | `this.knex = trx` | ✅ 参与外层事务 |
| **调用 upsertMany** | upsertMany 创建 | ✅ `fork({ knex: trx })` | `this.knex = trx` | ✅ 参与外层事务 |
| **调用 updateByQuery** | updateMany 创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |
| **调用 deleteByQuery** | deleteMany 创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |

### 3.2 按方法详细拆解

#### createOne / createMany

| 方法 | Filter 位置 | database 参数 | 事务参与 |
|------|------------|--------------|----------|
| `createOne`（单独） | 事务内 | `trx` | ✅ |
| `createOne`（被 createMany 调用） | 事务内 | `trx`（fork 传递的） | ✅ |
| `createMany` | 无（委托给 createOne） | - | ✅（外层事务） |

#### updateOne / updateMany / updateBatch / updateByQuery

| 方法 | Filter 位置 | database 参数 | 事务参与 |
|------|------------|--------------|----------|
| `updateOne`（单独） | 事务外（updateMany 内） | `this.knex`（普通） | ❌ |
| `updateOne`（被 updateBatch 调用） | 事务外但 database 是 trx | `this.knex = trx`（fork 传递的） | ✅ |
| `updateMany`（单独） | 事务外 | `this.knex`（普通） | ❌ |
| `updateMany`（被 updateBatch 调用） | 事务外但 database 是 trx | `this.knex = trx`（fork 传递的） | ✅ |
| `updateBatch` | 无（委托给 updateOne） | - | ✅（外层事务） |
| `updateByQuery` | 事务外（updateMany 内） | `this.knex`（普通） | ❌ |

#### deleteOne / deleteMany / deleteByQuery

| 方法 | Filter 位置 | database 参数 | 事务参与 |
|------|------------|--------------|----------|
| `deleteOne`（单独） | 事务外（deleteMany 内） | `this.knex`（普通） | ❌ |
| `deleteMany`（单独） | 事务外 | `this.knex`（普通） | ❌ |
| `deleteByQuery` | 事务外（deleteMany 内） | `this.knex`（普通） | ❌ |

**注意**：Directus 没有 `deleteBatch` 方法，所以 delete 的批量操作没有 fork 传递事务的路径。

#### upsertOne / upsertMany

| 方法 | 分支 | Filter 位置 | database 参数 | 事务参与 |
|------|------|------------|--------------|----------|
| `upsertOne`（单独，创建） | create | 事务内 | `trx` | ✅ |
| `upsertOne`（单独，更新） | update | 事务外 | `this.knex`（普通） | ❌ |
| `upsertOne`（被 upsertMany 调用，创建） | create | 事务内 | `trx` | ✅ |
| `upsertOne`（被 upsertMany 调用，更新） | update | 事务外但 database 是 trx | `this.knex = trx` | ✅ |
| `upsertMany` | - | 无（委托给 upsertOne） | - | ✅（外层事务） |

---

## 4. 流程触发器在不同路径下的行为

### 4.1 Filter 类型流程

**位置**: `api/src/flows.ts:196-213`

```typescript
if (flow.options['type'] === 'filter') {
    const handler: FilterHandler = (payload, meta, context) =>
        this.executeFlow(
            flow,
            { payload, ...meta },
            {
                accountability: context['accountability'],
                database: context['database'],  // ⚠️ 透传传入的 database
                getSchema: context['schema'] ? () => context['schema'] : getSchema,
            },
        );
    events.forEach((event) => emitter.onFilter(event, handler));
}
```

**行为取决于传入的 `context.database`**:

| 调用路径 | context.database | 流程执行环境 | 异常影响 |
|---------|-----------------|-------------|----------|
| 单独 createOne | `trx`（事务） | ✅ 参与事务 | 触发回滚 |
| 单独 updateMany | `this.knex`（普通） | ❌ 不参与事务 | 阻止事务开始，但流程内操作已提交 |
| 单独 deleteMany | `this.knex`（普通） | ❌ 不参与事务 | 阻止事务开始，但流程内操作已提交 |
| createMany 内的 createOne | `trx`（事务） | ✅ 参与事务 | 触发回滚 |
| updateBatch 内的 updateMany | `trx`（事务，fork 传递） | ✅ 参与事务 | 触发回滚 |
| upsertMany 内的 update | `trx`（事务，fork 传递） | ✅ 参与事务 | 触发回滚 |
| upsertMany 内的 create | `trx`（事务） | ✅ 参与事务 | 触发回滚 |

### 4.2 Action 类型流程

**位置**: `api/src/flows.ts:214-228`

```typescript
} else if (flow.options['type'] === 'action') {
    const handler: ActionHandler = (meta, context) =>
        this.executeFlow(flow, meta, {
            accountability: context['accountability'],
            database: getDatabase(),  // ⚠️ 总是使用新连接
            getSchema: context['schema'] ? () => context['schema'] : getSchema,
        });
    events.forEach((event) => emitter.onAction(event, handler));
}
```

**行为一致性**：
- 总是使用 `getDatabase()` 获取新连接
- 忽略传入的 `context.database`
- 总是在事务提交后异步执行
- 异常被捕获，不影响主流程

---

## 5. 自定义扩展在不同路径下的回滚与副作用

### 5.1 Filter 钩子扩展

扩展的 Filter 钩子接收的 `context.database` 取决于调用路径：

#### 路径 A: 单独调用 updateMany/deleteMany

```typescript
register.filter('items.update', async (payload, meta, context) => {
    const { database } = context;  // this.knex，普通连接
    
    // 扩展内的数据库操作使用独立连接
    await database('audit_log').insert({ ... });  // ⚠️ 自动提交！
    
    if (validationFailed) {
        throw new Error('Validation failed');  // 阻止主操作，但 audit_log 已写入
    }
    
    return payload;
});
```

**风险**：
- ❌ 扩展内数据库操作不参与后续事务
- ❌ 异常阻止主操作，但扩展内操作已提交
- ⚠️ 可能造成数据不一致

#### 路径 B: 单独调用 createOne / 通过批量方法调用

```typescript
register.filter('items.update', async (payload, meta, context) => {
    const { database } = context;  // trx，事务连接
    
    // 扩展内的数据库操作参与同一事务
    await database('audit_log').insert({ ... });  // 未提交，在事务内
    
    if (validationFailed) {
        throw new Error('Validation failed');  // 触发回滚，audit_log 也被撤销
    }
    
    return payload;
});
```

**保证**：
- ✅ 扩展内数据库操作参与事务
- ✅ 异常触发完整回滚
- ✅ 数据一致性有保障

### 5.2 副作用一致性矩阵

| 副作用类型 | 单独 createOne | 单独 updateMany | 单独 deleteMany | createMany | updateBatch | upsertMany |
|-----------|---------------|-----------------|-----------------|-----------|-------------|-----------|
| 扩展内 DB 操作 | ✅ 可回滚 | ❌ 独立提交 | ❌ 独立提交 | ✅ 可回滚 | ✅ 可回滚 | ✅ 可回滚 |
| 外部 API 调用 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |
| 文件系统操作 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |
| 缓存操作 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |

---

## 6. 实际风险场景（修订版）

### 场景 1: 单独调用 updateMany - 有风险

```typescript
// 单独调用
await itemsService.updateMany([1], { status: 'cancelled' });
```

**流程**:
1. Filter 钩子执行，`context.database` 是普通连接
2. 扩展写入审计日志 → 已提交
3. 验证失败抛出异常
4. 事务未开始，主操作被阻止
5. **审计日志已存在，但订单状态未更新** → 数据不一致

### 场景 2: 通过 updateBatch 调用 - 安全

```typescript
// 通过 updateBatch 调用
await itemsService.updateBatch([{ id: 1, status: 'cancelled' }]);
```

**流程**:
1. 外层 `transaction()` 创建事务
2. `fork({ knex: trx })` 传递事务连接
3. Filter 钩子执行，`context.database` 是 `trx`
4. 扩展写入审计日志 → 在事务内，未提交
5. 验证失败抛出异常
6. 外层事务回滚 → **审计日志也被撤销**
7. ✅ 数据一致

### 场景 3: upsertMany 中的混合路径

```typescript
await itemsService.upsertMany([
    { id: 1, status: 'updated' },  // 存在，走 update 路径
    { title: 'new item' },         // 不存在，走 create 路径
]);
```

**流程**:
1. 外层 `transaction()` 创建事务
2. `fork({ knex: trx })` 传递事务连接
3. 第一条（update）: Filter 钩子的 `context.database` 是 `trx` ✅
4. 第二条（create）: Filter 钩子的 `context.database` 是 `trx` ✅
5. 任何一条的 Filter 钩子失败 → 外层事务回滚
6. ✅ 两条记录的 Filter 钩子内操作都被撤销

---

## 7. 设计意图分析

### 7.1 为什么单独调用 updateMany/deleteMany 的 Filter 在事务外？

可能的设计考量：

1. **避免长事务**:
   - Filter 钩子可能执行复杂的验证或查询
   - 在事务外执行可以减少锁竞争
   - 但代价是失去事务保护

2. **代码结构差异**:
   - `createOne`: 所有逻辑（包括 filter）都在 `transaction()` 回调内
   - `updateMany`/`deleteMany`: filter 在 `transaction()` 调用前执行

3. **历史原因**:
   - 可能是不同时期的实现
   - 缺乏统一的设计规范

### 7.2 为什么批量方法使用 fork({knex})？

1. **原子性保证**:
   - 批量操作需要所有子操作要么全部成功，要么全部失败
   - 通过 `fork({ knex: trx })` 确保所有子操作在同一事务内

2. **避免嵌套事务**:
   - `transaction()` 的 `isTransaction` 检查确保不会创建嵌套事务
   - 子操作的 `transaction()` 调用实际上加入外层事务

3. **副作用一致性**:
   - 批量操作中的 Filter 钩子内的数据库操作也参与事务
   - 任何子操作失败都能完整回滚

---

## 8. 最佳实践建议（修订版）

### 8.1 了解你的调用路径

**推荐使用批量方法**（如果适用）:

```typescript
// ❌ 单独调用 updateMany - Filter 钩子不参与事务
await itemsService.updateMany([1, 2, 3], { status: 'published' });

// ✅ 使用 updateBatch - Filter 钩子参与外层事务
await itemsService.updateBatch([
    { id: 1, status: 'published' },
    { id: 2, status: 'published' },
    { id: 3, status: 'published' },
]);
```

**注意**：updateBatch 需要每条记录都有主键，且可以有不同的更新值。

### 8.2 对于单独调用的 update/delete

#### 策略 1: 只做只读操作

```typescript
register.filter('items.update', async (payload, meta, context) => {
    const { database } = context;
    
    // ✅ 只读操作是安全的
    const existing = await database('items').where('id', meta.keys[0]).first();
    
    // 验证逻辑
    if (!canUpdate(existing, payload)) {
        throw new Error('Validation failed');
    }
    
    return payload;
});

// 副作用放在 Action 钩子中
register.action('items.update', async (meta, context) => {
    const { database } = context;
    await database('audit_log').insert({ ... });
});
```

#### 策略 2: 手动管理事务（高级）

如果必须在 Filter 钩子中写入数据库，可以手动创建事务：

```typescript
import { transaction } from '../utils/transaction.js';

register.filter('items.update', async (payload, meta, context) => {
    const { database } = context;
    
    // 手动创建事务来保护 Filter 钩子内的操作
    return await transaction(database, async (trx) => {
        await trx('audit_log').insert({ ... });
        
        if (validationFailed) {
            throw new Error('Validation failed');  // 回滚 audit_log
        }
        
        return payload;
    });
});
```

**注意**：这只能保证 Filter 钩子内的操作一致，不能保证与主操作的一致性（因为主操作的事务在 Filter 之后才开始）。

### 8.3 副作用处理策略表

| 操作类型 | 推荐路径 | 推荐钩子 | 理由 |
|---------|---------|---------|------|
| 数据验证 | 任意路径 | Filter | 需要阻止无效操作 |
| 数据转换 | 任意路径 | Filter | 需要在写入前修改 |
| 审计日志（关键） | 批量方法 | Filter | 需要事务一致性 |
| 审计日志（非关键） | 任意路径 | Action | 容忍最终一致 |
| 外部 API 调用 | 任意路径 | Action | 失败不影响主流程 |
| 缓存清除 | 任意路径 | Action | 事务后执行更合理 |

### 8.4 测试建议

1. **测试不同调用路径**:
   - 单独调用 createOne/updateMany/deleteMany
   - 通过批量方法调用
   - 通过 upsertMany 调用（测试 create 和 update 分支）

2. **测试 Filter 钩子失败场景**:
   - 验证数据库操作是否正确回滚
   - 验证外部副作用的处理

3. **测试事务重试**:
   - 模拟死锁场景
   - 验证 Filter 钩子是否被正确重试

---

## 9. 代码位置索引

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| transaction 嵌套检查 | `api/src/utils/transaction.ts` | 14-20 |
| fork 方法实现 | `api/src/services/items.ts` | 68-88 |
| createOne Filter 钩子 | `api/src/services/items.ts` | 151-170 |
| updateMany Filter 钩子 | `api/src/services/items.ts` | 730-747 |
| deleteMany Filter 钩子 | `api/src/services/items.ts` | 1080-1096 |
| createMany 批量处理 | `api/src/services/items.ts` | 433-494 |
| updateBatch 批量处理 | `api/src/services/items.ts` | 657-704 |
| upsertMany 批量处理 | `api/src/services/items.ts` | 1010-1038 |
| upsertOne 分支逻辑 | `api/src/services/items.ts` | 981-1002 |
| 流程 Filter 触发器 | `api/src/flows.ts` | 196-213 |
| 流程 Action 触发器 | `api/src/flows.ts` | 214-228 |
| 扩展注册 Filter | `api/src/extensions/manager.ts` | 892-899 |
| Emitter.emitFilter | `api/src/emitter.ts` | 34-60 |

---

## 10. 修订版核心结论

### 10.1 修正后的关键发现

1. **事务边界取决于调用路径，而非操作类型**:
   - ❌ 之前错误：认为 create 的 Filter 总是在事务内，update/delete 总是在事务外
   - ✅ 正确：Filter 的事务边界取决于是否通过批量方法（使用 fork({knex})）调用

2. **单独调用路径**:
   - `createOne`: Filter 在事务内（因为 createOne 的 filter 在 transaction 回调内）
   - `updateMany`/`deleteMany`: Filter 在事务外（filter 在 transaction 调用前）

3. **批量调用路径（createMany/updateBatch/upsertMany）**:
   - 外层 `transaction()` 创建事务
   - `fork({ knex: trx })` 传递事务连接到子服务
   - 子服务的 `this.knex` 被设置为事务连接
   - `transaction()` 的 `isTransaction` 检查避免嵌套事务
   - ✅ **所有路径的 Filter 钩子都参与外层事务**

4. **transaction() 的嵌套检查是关键**:
   - `if (knex.isTransaction) return handler(knex)`
   - 这使得"嵌套"的 transaction 调用实际上加入同一个事务

### 10.2 对开发者的修正建议

1. **优先使用批量方法**:
   - `createMany`、`updateBatch`、`upsertMany` 提供一致的事务保护
   - Filter 钩子内的数据库操作可以回滚

2. **了解单独调用的风险**:
   - 单独调用 `updateMany`/`deleteMany` 时，Filter 钩子不参与事务
   - 避免在这些路径的 Filter 钩子中进行写入操作

3. **Action 钩子行为一致**:
   - 所有路径下的 Action 钩子行为相同
   - 总是异步，总是使用新连接，总是在事务后执行

4. **测试覆盖不同路径**:
   - 确保测试单独调用和批量调用两种场景
   - 特别注意 upsert 的 create/update 分支差异

### 10.3 潜在的代码改进方向

如果要统一行为，可以考虑：

1. **将 updateMany/deleteMany 的 filter 移入 transaction**:
   - 与 createOne 保持一致
   - 但可能会增加锁竞争

2. **或者将 createOne 的 filter 移出 transaction**:
   - 与 updateMany/deleteMany 保持一致
   - 但会失去事务保护

3. **文档明确说明**:
   - 在文档中明确不同调用路径的事务边界
   - 提供最佳实践指南
