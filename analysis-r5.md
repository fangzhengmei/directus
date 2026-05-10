# Directus 写入链路 Filter 与 Action 事务边界修订分析（R5）

## 核心修正（R5）：可回滚结论需要明确前提

经过最后一轮校准，发现 R4 的"可回滚"结论不够严谨。**Filter 钩子、流程和扩展内的数据库操作是否可回滚，不仅取决于调用路径，还取决于开发者使用的数据库连接获取方式**。

关键发现：
1. **使用 `context.database`**：可能参与事务（取决于调用路径）
2. **使用扩展注册时的 `database` 参数**：永远不参与事务
3. **自行调用 `getDatabase()` 或构造 ItemsService 不传入 knex**：永远不参与事务

---

## 1. 扩展开发者获取数据库连接的方式

### 1.1 扩展注册时的上下文

**位置**: `api/src/extensions/manager.ts:962-969`

```typescript
hookRegistrationCallback(hookRegistrationContext, {
    services,
    env,
    database: getDatabase(),  // ⚠️ 这是普通连接，永远不是事务连接！
    emitter: this.localEmitter,
    logger,
    getSchema,
});
```

**关键点**：
- 扩展注册时传入的 `database` 参数是 `getDatabase()` 返回的**普通连接**
- 这个连接**永远不是事务连接**
- 如果扩展开发者通过闭包捕获这个 `database` 并在钩子中使用，**永远不会参与事务**

### 1.2 Filter 钩子的 context 参数

**类型定义**: `packages/types/src/events.ts:6-16`

```typescript
export type EventContext = {
    database: Knex;          // ⚠️ 这个可能是事务连接，也可能是普通连接
    schema: SchemaOverview | null;
    accountability: Accountability | null;
};

export type FilterHandler<T = unknown> = (
    payload: T,
    meta: Record<string, any>,
    context: EventContext,    // ⬅️ 从这里获取 database
) => T | Promise<T>;
```

**关键点**：
- `context.database` 是 ItemsService 调用 `emitFilter` 时传入的
- 它可能是 `trx`（事务连接），也可能是 `this.knex`（普通连接）
- 取决于调用路径

### 1.3 三种获取方式的对比

| 获取方式 | 来源 | 事务可能性 | 说明 |
|---------|------|-----------|------|
| **`context.database`** | Filter 钩子的 `context` 参数 | ✅ 可能 | 取决于调用路径 |
| **扩展注册时的 `database`** | `registerHook` 回调的第二个参数 | ❌ 不可能 | 总是 `getDatabase()` 普通连接 |
| **自行调用 `getDatabase()`** | 从 `@directus/api` 导入 | ❌ 不可能 | 总是获取新的普通连接 |
| **构造 ItemsService 不传入 knex** | `new ItemsService(collection, options)` | ❌ 不可能 | 内部使用 `options.knex \|\| getDatabase()` |
| **构造 ItemsService 传入 `context.database`** | `new ItemsService(collection, { knex: context.database, ... })` | ✅ 可能 | 取决于 `context.database` |

---

## 2. 源码证据对比

### 2.1 证据一：Action 类型流程总是使用新连接

**位置**: `api/src/flows.ts:214-220`

```typescript
} else if (flow.options['type'] === 'action') {
    const handler: ActionHandler = (meta, context) =>
        this.executeFlow(flow, meta, {
            accountability: context['accountability'],
            database: getDatabase(),  // ⚠️ 总是获取新连接，忽略 context.database！
            getSchema: context['schema'] ? () => context['schema'] : getSchema,
        });
    events.forEach((event) => emitter.onAction(event, handler));
}
```

**对比 Filter 类型流程** (`api/src/flows.ts:196-213`)：

```typescript
if (flow.options['type'] === 'filter') {
    const handler: FilterHandler = (payload, meta, context) =>
        this.executeFlow(
            flow,
            { payload, ...meta },
            {
                accountability: context['accountability'],
                database: context['database'],  // ✅ 透传 context.database
                getSchema: context['schema'] ? () => context['schema'] : getSchema,
            },
        );
    events.forEach((event) => emitter.onFilter(event, handler));
}
```

**关键差异**：
| 流程类型 | database 来源 | 事务可能性 |
|---------|-------------|-----------|
| **Filter 类型流程** | `context['database']` | ✅ 可能参与事务 |
| **Action 类型流程** | `getDatabase()` | ❌ 永远不参与事务 |

### 2.2 证据二：扩展注册时的 database 永远是普通连接

**位置**: `api/src/extensions/manager.ts:962-969`

```typescript
hookRegistrationCallback(hookRegistrationContext, {
    services,
    env,
    database: getDatabase(),  // ⚠️ 总是普通连接
    emitter: this.localEmitter,
    logger,
    getSchema,
});
```

**扩展代码示例**：

```typescript
// 扩展注册文件 src/index.ts
export default ({ filter, action }, { services, database, env, logger, getSchema }) => {
    // ⚠️ 这里的 database 是 getDatabase() 返回的普通连接
    // 即使在事务上下文中，这个 database 也不会参与事务
    
    filter('items.create', async (payload, meta, context) => {
        // ❌ 错误方式：使用扩展注册时的 database
        await database('audit_log').insert({ ... });  // 永远不参与事务！
        
        // ✅ 正确方式：使用 context.database
        const { database: ctxDb } = context;
        await ctxDb('audit_log').insert({ ... });  // 可能参与事务
    });
};
```

### 2.3 证据三：构造 ItemsService 时的 knex 参数

**位置**: `api/src/services/items.ts:53-63`

```typescript
constructor(collection: Collection, options: AbstractServiceOptions) {
    this.collection = collection;
    this.knex = options.knex || getDatabase();  // ⚠️ 如果不传入 knex，使用 getDatabase()
    this.accountability = options.accountability || null;
    // ...
}
```

**扩展代码示例**：

```typescript
filter('items.create', async (payload, meta, context) => {
    const { database: ctxDb, schema, accountability } = context;
    const { ItemsService } = services;
    
    // ❌ 错误方式：不传入 knex
    const badService = new ItemsService('other_collection', {
        schema,
        accountability,
        // 没有传入 knex → 内部使用 getDatabase()
    });
    await badService.createOne({ ... });  // 永远不参与事务！
    
    // ✅ 正确方式：传入 context.database 作为 knex
    const goodService = new ItemsService('other_collection', {
        knex: ctxDb,  // ⬅️ 关键！
        schema,
        accountability,
    });
    await goodService.createOne({ ... });  // 可能参与事务
});
```

---

## 3. 修正后的可回滚结论

### 3.1 Filter 钩子中的数据库操作

#### 前提条件（必须同时满足）

1. **调用路径使 `context.database` 是事务连接**：
   - 单独调用 `createOne`
   - 通过 `createMany`/`updateBatch`/`upsertMany` 调用
   - 通过上层服务（如 `CollectionsService`）在 transaction 内构造 ItemsService 并传入 `knex: trx`

2. **开发者使用正确的数据库连接**：
   - 使用 `context.database` 进行数据库操作
   - 或者构造 ItemsService 时传入 `context.database` 作为 knex 参数

#### 可回滚的情况

```typescript
filter('items.create', async (payload, meta, context) => {
    const { database } = context;  // 这是事务连接（如果调用路径正确）
    
    // ✅ 可回滚：使用 context.database
    await database('audit_log').insert({
        action: 'create',
        collection: meta.collection,
        data: JSON.stringify(payload),
    });
    
    // ✅ 可回滚：构造 ItemsService 时传入 context.database
    const { ItemsService } = services;
    const relatedService = new ItemsService('related_collection', {
        knex: database,  // ⬅️ 关键！
        schema: context.schema,
        accountability: context.accountability,
    });
    await relatedService.createOne({ ... });
    
    // 如果这里抛出异常，上面的操作会回滚
    if (validationFailed) {
        throw new Error('Validation failed');
    }
    
    return payload;
});
```

#### 不可回滚的情况（即使调用路径正确）

```typescript
// 扩展注册时捕获的 database
export default ({ filter }, { services, database: extDb, ... }) => {
    filter('items.create', async (payload, meta, context) => {
        // ❌ 不可回滚：使用扩展注册时的 database
        await extDb('audit_log').insert({ ... });  // 永远不参与事务！
        
        // ❌ 不可回滚：自行调用 getDatabase()
        const knex = getDatabase();
        await knex('audit_log').insert({ ... });  // 永远不参与事务！
        
        // ❌ 不可回滚：构造 ItemsService 不传入 knex
        const { ItemsService } = services;
        const badService = new ItemsService('related_collection', {
            schema: context.schema,
            accountability: context.accountability,
            // 没有传入 knex → 内部使用 getDatabase()
        });
        await badService.createOne({ ... });  // 永远不参与事务！
        
        // 如果这里抛出异常，上面的操作不会回滚！
        if (validationFailed) {
            throw new Error('Validation failed');
        }
        
        return payload;
    });
};
```

### 3.2 Filter 类型流程中的数据库操作

#### 前提条件

1. **调用路径使 `context.database` 是事务连接**（同 Filter 钩子）
2. **流程操作使用传入的 database 连接**

#### 流程中的数据库操作

流程中的数据库操作通过"Run Script"操作或自定义操作实现。关键是要看操作是否使用传入的 `database` 连接。

**"Run Script"操作的上下文**：
```typescript
// 流程脚本中的可用变量
// - $trigger: 触发数据
// - $last: 上一个操作的结果
// - $accountability: 用户权限信息
// - $database: 数据库连接（取决于流程类型）
// - $schema: 数据库结构
```

**关键差异**：
| 流程类型 | `$database` 来源 | 事务可能性 |
|---------|-----------------|-----------|
| **Filter 类型流程** | `context['database']`（透传） | ✅ 可能参与事务 |
| **Action 类型流程** | `getDatabase()`（新连接） | ❌ 永远不参与事务 |

---

## 4. 完整一致性边界对照表

### 4.1 Filter 钩子中的操作

| 操作类型 | 使用 context.database | 使用扩展注册的 database | 自行调用 getDatabase() |
|---------|---------------------|-----------------------|----------------------|
| 直接数据库操作 | ⚠️ 可能可回滚 | ❌ 不可回滚 | ❌ 不可回滚 |
| 构造 ItemsService 传入 knex | ⚠️ 可能可回滚 | ❌ 不可回滚 | ❌ 不可回滚 |
| 构造 ItemsService 不传入 knex | ❌ 不可回滚 | ❌ 不可回滚 | ❌ 不可回滚 |
| 外部 API 调用 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |
| 文件系统操作 | ⚠️ 不会回滚 | ⚠️ 不会回滚 | ⚠️ 不会回滚 |

**注**："可能可回滚"表示还需要满足调用路径的前提条件。

### 4.2 调用路径对 context.database 的影响

| 调用路径 | context.database 类型 | 事务参与 |
|---------|---------------------|---------|
| 单独调用 createOne | 事务连接（trx） | ✅ 参与 |
| 单独调用 updateMany | 普通连接（this.knex） | ❌ 不参与 |
| 单独调用 deleteMany | 普通连接（this.knex） | ❌ 不参与 |
| 调用 createMany | 事务连接（trx） | ✅ 参与 |
| 调用 updateBatch | 事务连接（trx） | ✅ 参与 |
| 调用 upsertMany | 事务连接（trx） | ✅ 参与 |
| CollectionsService 内的 delete | 事务连接（trx） | ✅ 参与 |

### 4.3 流程触发器

| 流程类型 | 触发时机 | database 来源 | 事务参与 | 异常影响 |
|---------|---------|-------------|---------|---------|
| **Filter 类型** | 事务前或事务内 | `context['database']` | ⚠️ 取决于调用路径 | 可能阻止操作或触发回滚 |
| **Action 类型** | 事务提交后 | `getDatabase()` | ❌ 永远不参与 | 仅记录日志，不影响主流程 |

---

## 5. 实际风险场景（R5 修订版）

### 场景 1: 使用错误的数据库连接（即使调用路径正确）

**调用路径**：通过 updateBatch 调用（context.database 是事务连接）

**扩展代码**：
```typescript
// 扩展注册时捕获的 database
export default ({ filter }, { services, database: extDb, ... }) => {
    filter('items.update', async (payload, meta, context) => {
        // context.database 是 trx（事务连接）✅
        // 但开发者使用了错误的连接 ❌
        
        // ❌ 使用扩展注册时的 database
        await extDb('audit_log').insert({
            action: 'update',
            collection: meta.collection,
            keys: JSON.stringify(meta.keys),
        });  // ⚠️ 已独立提交！
        
        // ❌ 构造 ItemsService 不传入 knex
        const { ItemsService } = services;
        const relatedService = new ItemsService('related_collection', {
            schema: context.schema,
            accountability: context.accountability,
            // 没有传入 knex
        });
        await relatedService.updateMany(relatedKeys, { status: 'updated' });  // ⚠️ 已独立提交！
        
        // 验证失败
        if (validationFailed) {
            throw new Error('Validation failed');
        }
        
        return payload;
    });
};
```

**结果**：
1. 调用路径正确（updateBatch），`context.database` 是事务连接
2. 但开发者使用了 `extDb` 和不传入 knex 的 ItemsService
3. 这些操作使用独立连接，已提交
4. 验证失败抛出异常，主操作回滚
5. **审计日志和关联表更新已存在，但主记录未更新** → 数据不一致

### 场景 2: 使用正确的数据库连接（调用路径正确）

**调用路径**：通过 updateBatch 调用（context.database 是事务连接）

**扩展代码**：
```typescript
export default ({ filter }, { services, ... }) => {
    filter('items.update', async (payload, meta, context) => {
        const { database } = context;  // trx（事务连接）✅
        
        // ✅ 使用 context.database
        await database('audit_log').insert({ ... });  // 在事务内
        
        // ✅ 构造 ItemsService 时传入 context.database
        const { ItemsService } = services;
        const relatedService = new ItemsService('related_collection', {
            knex: database,  // ⬅️ 关键！
            schema: context.schema,
            accountability: context.accountability,
        });
        await relatedService.updateMany(relatedKeys, { status: 'updated' });  // 在事务内
        
        // 验证失败
        if (validationFailed) {
            throw new Error('Validation failed');
        }
        
        return payload;
    });
};
```

**结果**：
1. 调用路径正确，`context.database` 是事务连接
2. 开发者使用了正确的连接
3. 所有操作都在同一事务内
4. 验证失败抛出异常，**所有操作回滚**
5. ✅ 数据一致

### 场景 3: Filter 类型流程 vs Action 类型流程

**Filter 类型流程**：
```
调用路径正确（updateBatch）
  ↓
Filter 类型流程触发
  ↓
流程内的 database 操作使用 context.database（trx）
  ↓
流程执行失败
  ↓
外层事务回滚 → 流程内的数据库操作也回滚 ✅
```

**Action 类型流程**：
```
调用路径正确（updateBatch）
  ↓
主操作提交成功
  ↓
Action 类型流程触发
  ↓
流程内的 database 操作使用 getDatabase()（新连接）
  ↓
流程执行失败
  ↓
主操作已提交，流程内的操作可能部分提交 ❌
```

---

## 6. 关键机制总结（R5 修订版）

### 6.1 数据库连接的来源层级

```
┌─────────────────────────────────────────────────────────────────┐
│                    数据库连接来源层级                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Level 1: 调用路径决定 ItemsService.this.knex                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ 单独调用: this.knex = getDatabase() (普通连接)           │   │
│  │ 批量调用: this.knex = trx (通过 fork({ knex: trx }))     │   │
│  │ 上层服务: this.knex = trx (通过 new ItemsService(...,    │   │
│  │                              { knex: trx }))            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           ↓                                     │
│  Level 2: ItemsService 调用 emitFilter 时传入的 context.database │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ createOne: database = trx (在 transaction 回调内传入)    │   │
│  │ updateMany: database = this.knex (可能是 trx 或普通连接) │   │
│  │ deleteMany: database = this.knex (可能是 trx 或普通连接) │   │
│  └─────────────────────────────────────────────────────────┘   │
│                           ↓                                     │
│  Level 3: 开发者选择使用的连接                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ ✅ context.database: 透传的连接（可能参与事务）           │   │
│  │ ❌ 扩展注册时的 database: 普通连接（永远不参与事务）       │   │
│  │ ❌ 自行调用 getDatabase(): 新的普通连接                   │   │
│  │ ❌ 构造 ItemsService 不传入 knex: getDatabase()          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 三层决策模型

是否可回滚 = Level 1（调用路径）× Level 2（ItemsService 传入）× Level 3（开发者选择）

| Level 1 调用路径 | Level 2 context.database | Level 3 开发者选择 | 可回滚？ |
|----------------|------------------------|------------------|---------|
| 单独 updateMany | 普通连接 | 任意方式 | ❌ 否 |
| updateBatch | 事务连接 (trx) | context.database | ✅ 是 |
| updateBatch | 事务连接 (trx) | 扩展注册的 database | ❌ 否 |
| updateBatch | 事务连接 (trx) | ItemsService 传入 knex | ✅ 是 |
| updateBatch | 事务连接 (trx) | ItemsService 不传入 knex | ❌ 否 |
| CollectionsService.deleteOne | 事务连接 (trx) | context.database | ✅ 是 |
| CollectionsService.deleteOne | 事务连接 (trx) | 其他方式 | ❌ 否 |

---

## 7. 最佳实践建议（R5 修订版）

### 7.1 Filter 钩子中的数据库操作

#### ✅ 推荐模式

```typescript
export default ({ filter }, { services, ... }) => {
    filter('items.create', async (payload, meta, context) => {
        // 1. 从 context 获取连接
        const { database, schema, accountability } = context;
        
        // 2. 直接使用 context.database 进行数据库操作
        await database('audit_log').insert({ ... });
        
        // 3. 如果需要使用 ItemsService，传入 context.database 作为 knex
        const { ItemsService } = services;
        const relatedService = new ItemsService('related_collection', {
            knex: database,  // ⬅️ 关键！
            schema,
            accountability,
        });
        await relatedService.createOne({ ... });
        
        return payload;
    });
};
```

#### ❌ 不推荐模式

```typescript
export default ({ filter }, { services, database: extDb, ... }) => {
    filter('items.create', async (payload, meta, context) => {
        // ❌ 不要使用扩展注册时的 database
        await extDb('audit_log').insert({ ... });
        
        // ❌ 不要自行调用 getDatabase()
        const knex = getDatabase();
        await knex('audit_log').insert({ ... });
        
        // ❌ 构造 ItemsService 时不要忘记传入 knex
        const { ItemsService } = services;
        const badService = new ItemsService('related_collection', {
            schema: context.schema,
            accountability: context.accountability,
            // 缺少 knex 参数
        });
        await badService.createOne({ ... });
        
        return payload;
    });
};
```

### 7.2 检查是否在事务中

如果需要在运行时判断是否在事务中：

```typescript
filter('items.update', async (payload, meta, context) => {
    const { database } = context;
    
    // 检查是否在事务中
    const inTransaction = database.isTransaction;
    
    if (inTransaction) {
        // 在事务中，可以安全地进行需要一致性的操作
        await database('audit_log').insert({ ... });
    } else {
        // 不在事务中，考虑使用 Action 钩子或其他机制
        logger.warn('Not in transaction, audit log may be inconsistent');
    }
    
    return payload;
});
```

### 7.3 副作用处理策略（R5 修订版）

| 副作用类型 | 推荐钩子 | 推荐连接方式 | 一致性保证 |
|-----------|---------|-------------|-----------|
| 数据验证 | Filter | 不需要数据库 | ✅ 强一致 |
| 数据转换 | Filter | 不需要数据库 | ✅ 强一致 |
| 审计日志（关键） | Filter | context.database | ⚠️ 条件一致（需调用路径正确） |
| 审计日志（非关键） | Action | getDatabase() | ⚠️ 最终一致 |
| 外部 API 调用 | Action | 不需要 | ⚠️ 最终一致 |
| 缓存清除 | Action | 不需要 | ✅ 事务后执行 |

### 7.4 对于需要强一致性的场景

如果必须保证数据库操作的一致性：

1. **使用批量方法**：
   - `createMany`、`updateBatch`、`upsertMany`
   - 这些方法内部使用 `transaction()` + `fork({ knex: trx })`

2. **自定义上层服务**：
   ```typescript
   class MyCustomService {
       async safeUpdate(collection: string, keys: PrimaryKey[], data: Partial<Item>) {
           return await transaction(this.knex, async (trx) => {
               const itemsService = new ItemsService(collection, {
                   knex: trx,  // ⬅️ 关键！
                   schema: this.schema,
                   accountability: this.accountability,
               });
               return await itemsService.updateMany(keys, data);
           });
       }
   }
   ```

3. **在 Filter 钩子中**：
   - 始终使用 `context.database`
   - 构造 ItemsService 时传入 `context.database` 作为 knex

---

## 8. 代码位置索引（R5 修订版）

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| 扩展注册时的 database 参数 | `api/src/extensions/manager.ts` | 965 |
| Filter 类型流程透传 database | `api/src/flows.ts` | 196-213 |
| Action 类型流程使用 getDatabase() | `api/src/flows.ts` | 218 |
| FilterHandler 类型定义 | `packages/types/src/events.ts` | 12-16 |
| EventContext 类型定义 | `packages/types/src/events.ts` | 6-10 |
| ItemsService 构造函数 | `api/src/services/items.ts` | 53-63 |
| transaction 嵌套检查 | `api/src/utils/transaction.ts` | 14-27 |
| updateMany Filter 钩子 | `api/src/services/items.ts` | 730-747 |
| deleteMany Filter 钩子 | `api/src/services/items.ts` | 1080-1095 |
| CollectionsService.deleteOne | `api/src/services/collections.ts` | 621-799 |

---

## 9. 修订版核心结论（R5）

### 9.1 R5 相对于 R4 的修正

#### ❌ R4 不严谨的结论
> "扩展内数据库操作参与事务" / "Filter 类型流程参与事务"

#### ✅ R5 修正结论
> **只有同时满足以下条件，Filter 钩子和流程内的数据库操作才能参与事务并可回滚：**
> 
> 1. **调用路径正确**：
>    - 单独调用 createOne
>    - 通过 createMany/updateBatch/upsertMany 调用
>    - 通过上层服务（如 CollectionsService）在 transaction 内构造 ItemsService 并传入 `knex: trx`
> 
> 2. **开发者使用正确的连接获取方式**：
>    - 使用 `context.database` 进行数据库操作
>    - 或者构造 ItemsService 时传入 `context.database` 作为 knex 参数
> 
> **以下方式永远不参与事务：**
> - 使用扩展注册时的 `database` 参数
> - 自行调用 `getDatabase()`
> - 构造 ItemsService 时不传入 knex 参数

### 9.2 三层决策模型

是否可回滚 = 调用路径 × ItemsService 传入 × 开发者选择

| 层级 | 决定因素 | 影响 |
|------|---------|------|
| **Level 1** | 调用路径 | 决定 `context.database` 是事务连接还是普通连接 |
| **Level 2** | ItemsService 实现 | 决定 `emitFilter` 时传入的 `context.database` |
| **Level 3** | 开发者选择 | 决定实际使用哪个连接进行数据库操作 |

### 9.3 关键源码证据对比

| 证据 | 文件路径 | 关键代码 | 说明 |
|------|----------|---------|------|
| **扩展注册时的 database** | `api/src/extensions/manager.ts:965` | `database: getDatabase()` | 永远是普通连接 |
| **Filter 流程透传** | `api/src/flows.ts:196-213` | `database: context['database']` | 透传，可能是事务连接 |
| **Action 流程新建** | `api/src/flows.ts:218` | `database: getDatabase()` | 永远是新连接 |
| **ItemsService 构造** | `api/src/services/items.ts:59` | `this.knex = options.knex \|\| getDatabase()` | 不传入 knex 则用普通连接 |

### 9.4 对开发者的最终建议

1. **Filter 钩子中始终使用 `context.database`**：
   - 不要使用扩展注册时的 `database` 参数
   - 不要自行调用 `getDatabase()`

2. **构造 ItemsService 时传入 `context.database`**：
   ```typescript
   const relatedService = new ItemsService('other_collection', {
       knex: context.database,  // ⬅️ 必须传入！
       schema: context.schema,
       accountability: context.accountability,
   });
   ```

3. **了解你的调用路径**：
   - 单独调用 updateMany/deleteMany 时，`context.database` 是普通连接
   - 通过批量方法或上层服务调用时，`context.database` 可能是事务连接

4. **对于需要强一致性的副作用**：
   - 确保调用路径正确
   - 确保使用正确的连接获取方式
   - 考虑在运行时检查 `context.database.isTransaction`

5. **对于不需要强一致性的副作用**：
   - 使用 Action 钩子
   - 行为更一致，更容易推理
