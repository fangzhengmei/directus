# Directus 写入链路 Filter 与 Action 事务边界修订分析（R4）

## 核心修正（R4）：delete 路径不是恒不在事务内

经过重新核对代码，发现 R3 的分析存在重要偏差。**delete 路径不是恒不在事务内**。当 deleteMany/deleteOne 被上层服务通过以下方式调用时，其 filter 阶段的 `this.knex` 可以是事务连接：

1. **直接构造 ItemsService 并传入 `knex: trx`**（如 CollectionsService）
2. **通过继承 ItemsService 的子服务的 `this.knex` 继承事务上下文**

关键机制：
- `ItemsService` 的构造函数使用 `options.knex || getDatabase()`
- 上层服务在 `transaction()` 内部构造 `ItemsService` 时传入 `knex: trx`
- deleteMany 的 filter 阶段使用 `this.knex`，即事务连接
- deleteMany 内部的 `transaction(this.knex, ...)` 发现 `knex.isTransaction === true`，直接执行

---

## 1. 关键发现：deleteMany 可以参与事务

### 1.1 实际源码证据：CollectionsService.deleteOne

**位置**: `api/src/services/collections.ts:637-681`

```typescript
async deleteOne(collectionKey: string, opts?: MutationOptions): Promise<string> {
    // ... 前置检查 ...
    
    await transaction(this.knex, async (trx) => {
        // trx 是事务连接
        
        if (collectionToBeDeleted!.schema) {
            await trx.schema.dropTable(collectionKey);
        }
        
        // ...
        
        if (collectionToBeDeleted!.meta) {
            // ⚠️ 直接构造 ItemsService，传入 knex: trx
            const collectionsItemsService = new ItemsService('directus_collections', {
                knex: trx,              // ✅ 事务连接！
                accountability: this.accountability,
                schema: this.schema,
            });
            
            // 调用 deleteOne → deleteMany
            await collectionsItemsService.deleteOne(collectionKey, {
                bypassEmitAction: (params) =>
                    opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
            });
        }
        
        if (collectionToBeDeleted!.schema) {
            const fieldsService = new FieldsService({
                knex: trx,              // ✅ 事务连接！
                accountability: this.accountability,
                schema: this.schema,
            });
            
            // ⚠️ 直接构造 ItemsService，传入 knex: trx
            const fieldItemsService = new ItemsService('directus_fields', {
                knex: trx,              // ✅ 事务连接！
                accountability: this.accountability,
                schema: this.schema,
            });
            
            // 调用 deleteByQuery → deleteMany
            await fieldItemsService.deleteByQuery(
                {
                    filter: {
                        collection: { _eq: collectionKey },
                    },
                },
                {
                    bypassEmitAction: (params) =>
                        opts?.bypassEmitAction ? opts.bypassEmitAction(params) : nestedActionEvents.push(params),
                },
            );
        }
        
        // ... 更多操作
    });
    
    // ...
}
```

### 1.2 调用链分析

**CollectionsService.deleteOne → ItemsService.deleteOne**

```
CollectionsService.deleteOne(collectionKey)
  ↓
transaction(this.knex, async (trx) => {   // 创建外层事务
  ↓
  const collectionsItemsService = new ItemsService('directus_collections', {
      knex: trx,        // ✅ 传入事务连接
      accountability: this.accountability,
      schema: this.schema,
  });
  ↓
  collectionsItemsService.deleteOne(collectionKey, {...})
    ↓
    ItemsService.deleteOne(key, {...})
      ↓
      ItemsService.deleteMany([key], {...})
        ↓
        emitter.emitFilter(
            ...,
            {
                database: this.knex,  // ⚠️ this.knex 是 trx（事务连接）！
                schema: this.schema,
                accountability: this.accountability,
            },
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
- 之前的分析错误地认为 delete 路径的 filter 钩子总是在事务外
- 实际上，当被上层服务通过 `new ItemsService(..., { knex: trx })` 方式调用时：
  1. `collectionsItemsService` 的 `this.knex = trx`
  2. deleteMany 中的 `emitFilter` 使用 `this.knex`（即 trx）
  3. deleteMany 内部的 `transaction(this.knex, ...)` 发现 `this.knex.isTransaction === true`，直接执行
- **deleteMany 的 filter 钩子实际上参与外层事务**

### 1.3 CollectionsService.deleteMany 进一步确认

**位置**: `api/src/services/collections.ts:804-847`

```typescript
async deleteMany(collectionKeys: string[], opts?: MutationOptions): Promise<string[]> {
    // ... 前置检查 ...
    
    const nestedActionEvents: ActionEventParams[] = [];
    
    try {
        await transaction(this.knex, async (trx) => {
            // ⚠️ 构造 CollectionsService，传入 knex: trx
            const service = new CollectionsService({
                schema: this.schema,
                accountability: this.accountability,
                knex: trx,              // ✅ 事务连接！
            });
            
            for (const collectionKey of collectionKeys) {
                // 内部调用 deleteOne，其中会构造 ItemsService 并调用 deleteOne
                await service.deleteOne(collectionKey, {
                    autoPurgeCache: false,
                    autoPurgeSystemCache: false,
                    bypassEmitAction: (params) => nestedActionEvents.push(params),
                });
            }
        });
        
        return collectionKeys;
    } finally {
        // ...
    }
}
```

**调用链**：
```
CollectionsService.deleteMany(keys)
  ↓
transaction(this.knex, async (trx) => {
  ↓
  const service = new CollectionsService({ knex: trx, ... })
  ↓
  service.deleteOne(collectionKey)
    ↓
    // 同上，deleteOne 内部会构造 ItemsService 并调用 deleteOne/deleteByQuery
    // 这些 delete 操作的 filter 阶段使用的 this.knex 都是 trx
})
```

---

## 2. 完整的调用路径分类

### 2.1 路径分类标准

Filter 钩子的事务边界取决于 **`this.knex` 是否是事务连接**，判断标准：

| 判断条件 | 是否参与事务 |
|---------|-------------|
| `this.knex.isTransaction === true` | ✅ 参与 |
| `this.knex.isTransaction === false` | ❌ 不参与 |

### 2.2 使 deleteMany 参与事务的两种方式

#### 方式一：上层服务直接构造 ItemsService 传入 `knex: trx`

```typescript
// CollectionsService.deleteOne 中的模式
await transaction(this.knex, async (trx) => {
    const itemsService = new ItemsService('some_collection', {
        knex: trx,              // ✅ 事务连接
        accountability: this.accountability,
        schema: this.schema,
    });
    
    await itemsService.deleteOne(key);  // filter 阶段 this.knex = trx
});
```

**实际存在的例子**：
- `CollectionsService.deleteOne` 调用 `ItemsService('directus_collections').deleteOne()`
- `CollectionsService.deleteOne` 调用 `ItemsService('directus_fields').deleteByQuery()`

#### 方式二：通过继承传递事务上下文

如果上层服务继承自 ItemsService，并且其 `this.knex` 已经是事务连接：

```typescript
// 假设有一个继承自 ItemsService 的服务
class CustomService extends ItemsService {
    async customDelete(keys: PrimaryKey[]) {
        // this.knex 可能已经是事务连接（取决于构造时传入的选项）
        await this.deleteMany(keys);  // filter 阶段使用 this.knex
    }
}

// 上层调用
await transaction(this.knex, async (trx) => {
    const service = new CustomService('collection', {
        knex: trx,              // ✅ 事务连接
        ...
    });
    
    await service.customDelete(keys);  // deleteMany 的 filter 阶段 this.knex = trx
});
```

**实际存在的例子**：
- `FilesService`、`UsersService`、`RolesService` 等都继承自 `ItemsService`
- 如果这些服务的方法在 transaction 内部被调用，且 `this.knex` 是 trx，那么 deleteMany 的 filter 也会参与事务

### 2.3 使 updateMany 参与事务的两种方式

#### 方式一：通过 updateBatch 的 fork({knex}) 传递

**位置**: `api/src/services/items.ts:657-704`

```typescript
async updateBatch(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    try {
        await transaction(this.knex, async (knex) => {
            const service = this.fork({ knex });  // ✅ fork 传递事务连接
            // service.this.knex = trx
            
            for (const index in data) {
                keys.push(await service.updateOne(primaryKey, omit(item, primaryKeyField), combinedOpts));
            }
        });
    } finally {
        // ...
    }
}
```

#### 方式二：上层服务直接构造 ItemsService 传入 `knex: trx`

与 deleteMany 相同的模式。

### 2.4 使 createOne 参与事务的方式

#### 方式一：单独调用（createOne 内部自己创建事务）

```typescript
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
        // filter 阶段在 transaction 内部，传入 trx
        const payloadAfterHooks = await emitter.emitFilter(
            ...,
            { database: trx, ... },  // ✅ 事务连接
        );
        // ...
    });
}
```

#### 方式二：通过 createMany 的 fork({knex}) 传递

**位置**: `api/src/services/items.ts:433-494`

```typescript
async createMany(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    const { primaryKeys, nestedActionEvents } = await transaction(this.knex, async (knex) => {
        const service = this.fork({ knex });  // ✅ fork 传递事务连接
        
        for (const [index, payload] of data.entries()) {
            const primaryKey = await service.createOne(payload, { ... });
        }
        // ...
    });
}
```

---

## 3. 完整事务边界对照表（R4 修订版）

### 3.1 按调用路径分类

| 调用路径 | 外层事务 | 事务传递方式 | Filter 钩子 database | 事务边界 |
|---------|---------|-------------|---------------------|----------|
| **单独调用 createOne** | 自己创建 | 内部传入 | `trx`（事务内传入） | ✅ 参与事务 |
| **单独调用 updateMany** | 自己创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |
| **单独调用 deleteMany** | 自己创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |
| **调用 createMany** | createMany 创建 | ✅ `fork({ knex: trx })` | `trx`（createOne 内传入） | ✅ 参与外层事务 |
| **调用 updateBatch** | updateBatch 创建 | ✅ `fork({ knex: trx })` | `this.knex = trx` | ✅ 参与外层事务 |
| **调用 upsertMany** | upsertMany 创建 | ✅ `fork({ knex: trx })` | `this.knex = trx` | ✅ 参与外层事务 |
| **调用 updateByQuery** | updateMany 创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |
| **调用 deleteByQuery** | deleteMany 创建（filter 后） | 无 | `this.knex`（普通连接） | ❌ 不参与事务 |
| **CollectionsService.deleteOne 内的 ItemsService.deleteOne** | CollectionsService 创建 | ✅ 直接构造传入 `knex: trx` | `this.knex = trx` | ✅ 参与外层事务 |
| **CollectionsService.deleteOne 内的 ItemsService.deleteByQuery** | CollectionsService 创建 | ✅ 直接构造传入 `knex: trx` | `this.knex = trx` | ✅ 参与外层事务 |

### 3.2 delete 方法详细拆解（R4 修订版）

| 方法 | 调用场景 | this.knex 来源 | database 参数 | 事务参与 |
|------|---------|---------------|--------------|----------|
| `deleteOne`（单独） | 直接调用 | `getDatabase()` | `this.knex`（普通） | ❌ |
| `deleteMany`（单独） | 直接调用 | `getDatabase()` | `this.knex`（普通） | ❌ |
| `deleteByQuery`（单独） | 直接调用 | `getDatabase()` | `this.knex`（普通） | ❌ |
| `deleteOne`（被 CollectionsService 调用） | `new ItemsService(..., { knex: trx })` | `trx` | `this.knex = trx` | ✅ |
| `deleteMany`（被 CollectionsService 调用） | `new ItemsService(..., { knex: trx })` | `trx` | `this.knex = trx` | ✅ |
| `deleteByQuery`（被 CollectionsService 调用） | `new ItemsService(..., { knex: trx })` | `trx` | `this.knex = trx` | ✅ |

**R3 错误修正**：
- ❌ R3 结论："Directus 没有 deleteBatch 方法，所以 delete 的批量操作没有 fork 传递事务的路径"
- ✅ R4 修正：虽然没有 `deleteBatch`，但上层服务可以通过直接构造 `ItemsService` 并传入 `knex: trx` 来使 deleteMany 的 filter 参与事务

### 3.3 update 方法详细拆解

| 方法 | 调用场景 | this.knex 来源 | database 参数 | 事务参与 |
|------|---------|---------------|--------------|----------|
| `updateOne`（单独） | 直接调用 | `getDatabase()` | `this.knex`（普通） | ❌ |
| `updateMany`（单独） | 直接调用 | `getDatabase()` | `this.knex`（普通） | ❌ |
| `updateByQuery`（单独） | 直接调用 | `getDatabase()` | `this.knex`（普通） | ❌ |
| `updateOne`（被 updateBatch 调用） | `fork({ knex: trx })` | `trx` | `this.knex = trx` | ✅ |
| `updateMany`（被 updateBatch 调用） | `fork({ knex: trx })` | `trx` | `this.knex = trx` | ✅ |

### 3.4 create 方法详细拆解

| 方法 | 调用场景 | this.knex 来源 | database 参数 | 事务参与 |
|------|---------|---------------|--------------|----------|
| `createOne`（单独） | 直接调用 | transaction 内部传入 | `trx` | ✅ |
| `createOne`（被 createMany 调用） | `fork({ knex: trx })` | `trx` | `trx` | ✅ |
| `createMany` | 直接调用 | 外层事务 | - | ✅ |

---

## 4. 流程触发器在 delete 路径下的行为（R4 修订版）

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

**行为取决于传入的 `context.database`**：

| 调用路径 | context.database | 流程执行环境 | 异常影响 |
|---------|-----------------|-------------|----------|
| 单独 deleteOne/deleteMany | `this.knex`（普通） | ❌ 不参与事务 | 阻止事务开始，但流程内操作已提交 |
| CollectionsService 内的 deleteOne/deleteByQuery | `trx`（事务） | ✅ 参与事务 | 触发回滚 |
| 单独 createOne | `trx`（事务） | ✅ 参与事务 | 触发回滚 |
| 单独 updateMany | `this.knex`（普通） | ❌ 不参与事务 | 阻止事务开始，但流程内操作已提交 |
| createMany 内的 createOne | `trx`（事务） | ✅ 参与事务 | 触发回滚 |
| updateBatch 内的 updateMany | `trx`（事务） | ✅ 参与事务 | 触发回滚 |

### 4.2 delete 路径的实际流程示例

**场景：删除一个 collection**

```
用户请求删除 collection "test_collection"
  ↓
CollectionsService.deleteOne("test_collection")
  ↓
transaction(this.knex, async (trx) => {   // 创建外层事务
  ↓
  const collectionsItemsService = new ItemsService('directus_collections', {
      knex: trx,        // ✅ 事务连接
      ...
  })
  ↓
  collectionsItemsService.deleteOne("test_collection")
    ↓
    ItemsService.deleteMany(["test_collection"], {...})
      ↓
      emitter.emitFilter(
          'directus_collections.items.delete',
          keys,
          { collection: 'directus_collections' },
          {
              database: this.knex,  // this.knex = trx ✅
              ...
          },
      )
      ↓
      // 如果有 Filter 类型流程绑定到 'directus_collections.items.delete' 事件
      // 流程内的 database 操作使用 trx，参与事务
      ↓
      transaction(this.knex, async (innerTrx) => {
          // this.knex 是 trx，直接执行
          await trx('directus_collections').where('collection', 'test_collection').delete();
      })
})
```

**如果 Filter 流程失败**：
1. 流程执行失败，抛出异常
2. 外层 `transaction()` 捕获异常
3. **整个事务回滚**（包括流程内的数据库操作）
4. collection 的元数据不会被删除

---

## 5. 自定义扩展在 delete 路径下的回滚与副作用（R4 修订版）

### 5.1 两种 delete 路径的对比

#### 路径 A: 单独调用 deleteMany（无事务上下文）

```typescript
// 单独调用
const service = new ItemsService('articles', { schema, accountability });
await service.deleteMany([1, 2]);
```

**扩展行为**：
```typescript
register.filter('items.delete', async (keys, meta, context) => {
    const { database } = context;  // this.knex，普通连接
    
    // 扩展内的数据库操作使用独立连接
    await database('audit_log').insert({
        action: 'delete',
        collection: meta.collection,
        keys: JSON.stringify(keys),
    });  // ⚠️ 自动提交！
    
    if (validationFailed) {
        throw new Error('Validation failed');  // 阻止主操作，但 audit_log 已写入
    }
    
    return keys;
});
```

**风险**：
- ❌ 扩展内数据库操作不参与后续事务
- ❌ 异常阻止主操作，但扩展内操作已提交
- ⚠️ 可能造成数据不一致

#### 路径 B: 被上层服务通过 transaction 包裹调用（有事务上下文）

```typescript
// 通过 CollectionsService 调用（间接）
const collectionsService = new CollectionsService({ schema, accountability });
await collectionsService.deleteOne('test_collection');

// 内部会调用：
// new ItemsService('directus_collections', { knex: trx, ... }).deleteOne('test_collection')
```

**扩展行为**：
```typescript
register.filter('directus_collections.items.delete', async (keys, meta, context) => {
    const { database } = context;  // trx，事务连接
    
    // 扩展内的数据库操作参与同一事务
    await database('audit_log').insert({
        action: 'delete',
        collection: meta.collection,
        keys: JSON.stringify(keys),
    });  // 未提交，在事务内
    
    if (validationFailed) {
        throw new Error('Validation failed');  // 触发回滚，audit_log 也被撤销
    }
    
    return keys;
});
```

**保证**：
- ✅ 扩展内数据库操作参与事务
- ✅ 异常触发完整回滚
- ✅ 数据一致性有保障

### 5.2 delete 路径的副作用一致性矩阵（R4 修订版）

| 副作用类型 | 单独 deleteMany | CollectionsService 内的 delete |
|-----------|-----------------|-------------------------------|
| 扩展内 DB 操作 | ❌ 独立提交 | ✅ 可回滚 |
| 外部 API 调用 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |
| 文件系统操作 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |
| 缓存操作 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |

---

## 6. 实际风险场景（R4 修订版）

### 场景 1: 单独调用 deleteMany - 有风险

```typescript
// 单独调用
await itemsService.deleteMany([1], { emitEvents: true });
```

**流程**:
1. Filter 钩子执行，`context.database` 是普通连接
2. 扩展写入审计日志 → 已提交
3. 验证失败抛出异常
4. 事务未开始，主操作被阻止
5. **审计日志已存在，但记录未删除** → 数据不一致

### 场景 2: 通过 CollectionsService 调用 deleteOne - 安全

```typescript
// 通过 CollectionsService 调用（删除 collection 元数据）
await collectionsService.deleteOne('test_collection');
```

**流程**:
1. 外层 `transaction()` 创建事务
2. 构造 `ItemsService('directus_collections', { knex: trx, ... })`
3. Filter 钩子执行，`context.database` 是 `trx`
4. 扩展写入审计日志 → 在事务内，未提交
5. 验证失败抛出异常
6. 外层事务回滚 → **审计日志也被撤销**
7. ✅ 数据一致

### 场景 3: 用户自定义上层服务中的 delete 操作

```typescript
// 用户自定义的上层服务
class CustomService {
    async deleteWithDependencies(collection: string, keys: PrimaryKey[]) {
        await transaction(this.knex, async (trx) => {
            // 构造 ItemsService，传入事务连接
            const itemsService = new ItemsService(collection, {
                knex: trx,              // ✅ 事务连接
                accountability: this.accountability,
                schema: this.schema,
            });
            
            // 先删除关联数据
            const relatedService = new ItemsService('related_table', {
                knex: trx,
                accountability: this.accountability,
                schema: this.schema,
            });
            await relatedService.deleteMany(relatedKeys);  // filter 阶段 this.knex = trx ✅
            
            // 再删除主数据
            await itemsService.deleteMany(keys);  // filter 阶段 this.knex = trx ✅
        });
    }
}
```

**行为**:
- `relatedService.deleteMany()` 的 filter 钩子使用 `trx`
- `itemsService.deleteMany()` 的 filter 钩子使用 `trx`
- 任何一个 filter 钩子失败 → 外层事务回滚
- ✅ 所有 filter 钩子内的数据库操作都被撤销

---

## 7. 关键机制总结（R4 修订版）

### 7.1 三个核心机制

#### 机制 1: transaction() 的嵌套检查

**位置**: `api/src/utils/transaction.ts:14-27`

```typescript
export const transaction = async <T = unknown>(
    knex: Knex,
    handler: (knex: Knex.Transaction) => Promise<T>,
): Promise<T> => {
    if (knex.isTransaction) {
        // 如果已经在事务中，直接执行，不创建新事务
        return handler(knex as Knex.Transaction);
    } else {
        return await knex.transaction((trx) => handler(trx));
    }
};
```

**作用**:
- 避免嵌套事务
- 使"嵌套"的 transaction 调用实际上加入同一个外层事务

#### 机制 2: ItemsService 的构造函数

**位置**: `api/src/services/items.ts:53-63`

```typescript
constructor(collection: Collection, options: AbstractServiceOptions) {
    this.collection = collection;
    this.knex = options.knex || getDatabase();  // 使用传入的 knex 或获取新连接
    this.accountability = options.accountability || null;
    // ...
}
```

**作用**:
- 允许上层服务传入 `knex` 参数
- 如果传入 `trx`（事务连接），`this.knex` 就是事务连接

#### 机制 3: filter 阶段使用 this.knex

**位置**: `api/src/services/items.ts:1080-1095` (deleteMany)

```typescript
async deleteMany(keys: PrimaryKey[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    // ...
    
    const keysAfterHooks =
        opts.emitEvents !== false
            ? await emitter.emitFilter(
                    ...,
                    {
                        database: this.knex,  // ⚠️ 使用 this.knex
                        schema: this.schema,
                        accountability: this.accountability,
                    },
                )
            : keys;
    
    // ...
    
    await transaction(this.knex, async (trx) => {  // ⚠️ 使用 this.knex
        // ...
    });
}
```

**作用**:
- filter 阶段和后续的 transaction 都使用 `this.knex`
- 如果 `this.knex` 是事务连接，两者都参与同一事务

### 7.2 两种事务传递方式对比

| 方式 | 使用方法 | 适用场景 | 示例 |
|------|---------|---------|------|
| **fork({ knex: trx })** | `this.fork({ knex: trx })` | 同一服务的批量操作 | `createMany`、`updateBatch`、`upsertMany` |
| **直接构造传入** | `new ItemsService(..., { knex: trx })` | 不同服务间的协作 | `CollectionsService.deleteOne` 内调用 `ItemsService` |

**共同点**:
- 都使子服务的 `this.knex = trx`
- 都使子服务的 filter 阶段参与外层事务

---

## 8. 最佳实践建议（R4 修订版）

### 8.1 了解你的调用路径

**对于 delete 操作**：

```typescript
// ❌ 单独调用 deleteMany - Filter 钩子不参与事务
const service = new ItemsService('articles', { schema, accountability });
await service.deleteMany([1, 2, 3]);

// ✅ 在上层服务的 transaction 内构造 ItemsService 并传入 knex: trx
await transaction(this.knex, async (trx) => {
    const service = new ItemsService('articles', {
        knex: trx,              // ✅ 事务连接
        schema: this.schema,
        accountability: this.accountability,
    });
    await service.deleteMany([1, 2, 3]);  // Filter 钩子参与事务
});
```

### 8.2 对于自定义上层服务

如果你的服务需要调用 ItemsService 的 delete 方法并保证事务一致性：

```typescript
class MyCustomService {
    private knex: Knex;
    private schema: SchemaOverview;
    private accountability: Accountability | null;
    
    constructor(options: AbstractServiceOptions) {
        this.knex = options.knex || getDatabase();
        this.schema = options.schema;
        this.accountability = options.accountability || null;
    }
    
    async safeDelete(collection: string, keys: PrimaryKey[]) {
        // 如果 this.knex 已经是事务连接，直接使用
        if (this.knex.isTransaction) {
            const itemsService = new ItemsService(collection, {
                knex: this.knex,
                schema: this.schema,
                accountability: this.accountability,
            });
            return await itemsService.deleteMany(keys);
        }
        
        // 否则，创建事务并传入
        return await transaction(this.knex, async (trx) => {
            const itemsService = new ItemsService(collection, {
                knex: trx,
                schema: this.schema,
                accountability: this.accountability,
            });
            return await itemsService.deleteMany(keys);
        });
    }
}
```

### 8.3 副作用处理策略（R4 修订版）

| 操作类型 | 推荐调用方式 | 推荐钩子 | 理由 |
|---------|-------------|---------|------|
| 数据验证 | 任意方式 | Filter | 需要阻止无效操作 |
| 数据转换 | 任意方式 | Filter | 需要在写入前修改 |
| 审计日志（关键） | transaction 内构造服务 | Filter | 需要事务一致性 |
| 审计日志（非关键） | 任意方式 | Action | 容忍最终一致 |
| 外部 API 调用 | 任意方式 | Action | 失败不影响主流程 |

### 8.4 对于系统集合的特殊注意

Directus 的系统集合（如 `directus_collections`、`directus_fields`、`directus_relations` 等）的删除操作通常通过专门的服务（如 `CollectionsService`、`FieldsService`、`RelationsService`）进行，这些服务内部会：

1. 在 `transaction()` 内执行
2. 构造 `ItemsService` 时传入 `knex: trx`
3. 调用 `ItemsService` 的 delete 方法

**结论**：系统集合的 delete 操作的 Filter 钩子通常参与事务。

---

## 9. 代码位置索引（R4 修订版）

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| transaction 嵌套检查 | `api/src/utils/transaction.ts` | 14-27 |
| ItemsService 构造函数 | `api/src/services/items.ts` | 53-63 |
| fork 方法实现 | `api/src/services/items.ts` | 68-88 |
| createOne Filter 钩子 | `api/src/services/items.ts` | 151-170 |
| updateMany Filter 钩子 | `api/src/services/items.ts` | 730-747 |
| deleteMany Filter 钩子 | `api/src/services/items.ts` | 1080-1095 |
| createMany 批量处理 | `api/src/services/items.ts` | 433-494 |
| updateBatch 批量处理 | `api/src/services/items.ts` | 657-704 |
| upsertMany 批量处理 | `api/src/services/items.ts` | 1010-1038 |
| CollectionsService.deleteOne | `api/src/services/collections.ts` | 621-799 |
| CollectionsService.deleteMany | `api/src/services/collections.ts` | 804-847 |
| 流程 Filter 触发器 | `api/src/flows.ts` | 196-213 |
| 扩展注册 Filter | `api/src/extensions/manager.ts` | 892-899 |
| Emitter.emitFilter | `api/src/emitter.ts` | 34-60 |

---

## 10. 修订版核心结论（R4）

### 10.1 R4 相对于 R3 的修正

#### ❌ R3 错误结论
> "Directus 没有 deleteBatch 方法，所以 delete 的批量操作没有 fork 传递事务的路径。"

#### ✅ R4 修正结论
> 虽然没有 `deleteBatch` 方法，但 deleteMany 可以通过以下方式参与事务：
> 1. 上层服务直接构造 `ItemsService` 并传入 `knex: trx`（如 CollectionsService）
> 2. 继承 ItemsService 的子服务的 `this.knex` 继承事务上下文
> 
> 实际上，delete、update、create 三种操作的 Filter 钩子都**可以**参与事务，取决于调用路径。

### 10.2 统一的事务边界判断标准

Filter 钩子是否参与事务，**不是**由操作类型（create/update/delete）决定的，而是由**调用时的 `this.knex` 是否是事务连接**决定的。

判断标准：
```typescript
// 对于 updateMany 和 deleteMany
if (this.knex.isTransaction) {
    // ✅ filter 阶段的 database 参数是事务连接
    // ✅ 内部的 transaction() 调用直接执行，不创建新事务
    // ✅ Filter 钩子参与事务
} else {
    // ❌ filter 阶段的 database 参数是普通连接
    // ❌ 内部的 transaction() 调用创建新事务
    // ❌ Filter 钩子不参与事务
}

// 对于 createOne
// filter 阶段在 transaction 内部，传入的是 trx
// 所以总是参与事务（除非被上层 fork 传递的 trx 覆盖）
```

### 10.3 三种操作的事务参与方式

| 操作 | 使 filter 参与事务的方式 | 示例 |
|------|---------------------|------|
| **create** | 1. 单独调用（内部创建事务）<br>2. createMany 的 fork({knex}) | `createOne()`、`createMany()` |
| **update** | 1. updateBatch 的 fork({knex})<br>2. 上层服务直接构造传入 `knex: trx` | `updateBatch()`、自定义上层服务 |
| **delete** | 1. 上层服务直接构造传入 `knex: trx`<br>2. 继承 ItemsService 的子服务 | `CollectionsService.deleteOne()`、自定义上层服务 |

### 10.4 对开发者的最终建议

1. **不要假设 Filter 钩子的事务边界**：
   - 检查 `context.database.isTransaction` 来判断当前是否在事务中
   - 或者了解你的代码被调用的路径

2. **对于 delete 操作的一致性保证**：
   - 在自定义上层服务中，使用 `transaction()` 包裹并构造 `ItemsService` 时传入 `knex: trx`
   - 或者使用 Action 钩子来处理副作用

3. **对于系统集合**：
   - Directus 自己的服务（CollectionsService、FieldsService 等）内部已经正确处理了事务
   - 监听系统集合的 delete 事件的 Filter 钩子通常参与事务

4. **测试建议**：
   - 测试单独调用和通过上层服务调用两种场景
   - 特别注意自定义上层服务中的事务传递
