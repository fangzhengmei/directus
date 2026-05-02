# Directus Schema 推断机制深度分析

## 1. 概述

Directus 的 Schema 推断机制是一个设计精良的多层架构系统，它不仅能够从现有数据库中自动推断表结构、字段类型和关系，还包含了复杂的缓存并发控制、重试机制、跨实例同步以及完整的元数据合并刷新路径。

本报告深入分析 Directus Schema 推断的完整运行时机制，重点包括：
- 缓存机制与并发控制
- 重试机制与容错处理
- 跨实例同步信号机制
- 推断结果与元数据合并的完整路径
- 多数据库差异抹平的正确层次

## 2. 架构层次重新梳理

### 2.1 完整架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              业务层 (Business Layer)                               │
│  - SchemaOverview: 包含 collections、relations、fields 的完整业务视图               │
│  - API 中间件: schema.ts 中间件，每个请求获取 schema                                │
│  - 权限系统: 基于 schema 进行权限检查                                               │
│  - 查询构建器: 使用 schema 构建正确的 SQL 查询                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Schema 缓存与并发控制层 (Cache & Concurrency)              │
│  - getMemorySchemaCache() / setMemorySchemaCache(): 内存级缓存                     │
│  - localSchemaCache: Keyv 本地缓存（支持内存或 Redis）                              │
│  - useLock(): 分布式锁（本地或 Redis）                                              │
│  - useBus(): 消息总线（本地或 Redis Pub/Sub）                                       │
│  - 并发控制: 单进程构建 + 多进程等待                                                 │
│  - 重试机制: 最多 3 次重试 + 超时保护                                               │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           元数据合并层 (Metadata Merge)                             │
│  - getDatabaseSchema(): 核心的 schema 构建函数                                      │
│  - schemaInspector.overview(): 从数据库获取原始 schema                               │
│  - 查询 directus_collections: 获取集合元数据                                        │
│  - 查询 directus_fields: 获取字段元数据                                             │
│  - RelationsService.readAll(): 获取并合并关系数据                                   │
│  - stitchRelations(): 合并 schema 外键与 meta 关系                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Schema Inspector 层 (@directus/schema)                     │
│  - SchemaInspector 接口: 统一的 schema 访问接口                                     │
│  - 方言实现: MySQL、PostgreSQL、SQLite、MSSQL、Oracle、CockroachDB                 │
│  - 类型定义: Column、Table、ForeignKey、SchemaOverview                              │
│  - 差异抹平: 第一层，将不同数据库的系统表查询转换为统一类型                            │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              数据库层 (Database Layer)                              │
│  - Knex.js: 数据库连接和查询构建器                                                  │
│  - 各种 SQL 数据库: MySQL、PostgreSQL、SQLite、MSSQL、Oracle、CockroachDB          │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 关键组件职责

| 层次 | 组件 | 主要职责 |
|------|------|----------|
| 业务层 | `SchemaOverview` | 完整的业务 schema 视图，包含集合、字段、关系 |
| 业务层 | `schema` 中间件 | 每个 HTTP 请求获取 schema 并附加到 `req.schema` |
| 缓存层 | `getSchema()` | schema 获取的入口函数，处理缓存、并发、重试 |
| 缓存层 | `memorySchemaCache` | 进程内内存缓存，最快的缓存层 |
| 缓存层 | `localSchemaCache` | Keyv 缓存，支持内存或 Redis |
| 缓存层 | `useLock()` | 分布式锁，控制并发构建 |
| 缓存层 | `useBus()` | 消息总线，跨实例同步 |
| 元数据层 | `getDatabaseSchema()` | 实际构建 schema，合并数据库与元数据 |
| 元数据层 | `RelationsService` | 关系数据的获取与合并 |
| Inspector层 | `SchemaInspector` | 方言适配，数据库差异抹平 |

## 3. 核心入口：getSchema() 函数

`getSchema()` 函数是 Directus Schema 推断的核心入口，它协调了缓存、并发控制、重试机制和实际的 schema 构建。

### 3.1 完整执行流程

```typescript
export async function getSchema(
  options?: {
    database?: Knex;
    bypassCache?: boolean;  // 绕过缓存，强制重新构建
  },
  attempt = 0,  // 当前重试次数
): Promise<SchemaOverview> {
  const MAX_ATTEMPTS = 3;  // 最大重试次数

  const env = useEnv();

  // 步骤 1: 检查是否需要绕过缓存
  if (options?.bypassCache || env['CACHE_SCHEMA'] === false) {
    const database = options?.database || getDatabase();
    const schemaInspector = createInspector(database);
    return await getDatabaseSchema(database, schemaInspector);
  }

  // 步骤 2: 检查内存缓存
  const cached = getMemorySchemaCache();
  if (cached) {
    return cached;
  }

  // 步骤 3: 检查重试次数，防止无限循环
  if (attempt >= MAX_ATTEMPTS) {
    throw new Error(`Failed to get Schema information: hit infinite loop`);
  }

  // 步骤 4: 获取分布式锁和消息总线
  const lock = useLock();
  const bus = useBus();

  const lockKey = 'schemaCache--preparing';
  const messageKey = 'schemaCache--done';
  const processId = await lock.increment(lockKey);  // 原子递增

  // 步骤 5: 防止锁计数器溢出
  if (processId >= (env['CACHE_SCHEMA_MAX_ITERATIONS'] as number)) {
    await lock.delete(lockKey);
  }

  // 步骤 6: 判断当前进程是否应该处理 schema 构建
  const currentProcessShouldHandleOperation = processId === 1;

  if (currentProcessShouldHandleOperation === false) {
    // 非处理进程：等待其他进程完成
    logger.trace('Schema cache is prepared in another process, waiting for result.');

    // 超时机制
    const timeout: Promise<any> = new Promise((_, reject) =>
      setTimeout(reject, env['CACHE_SCHEMA_SYNC_TIMEOUT'] as number),
    );

    // 订阅消息总线，等待构建完成的通知
    const subscription = new Promise<SchemaOverview>((resolve, reject) => {
      bus.subscribe(messageKey, busListener).catch(reject);

      function busListener(options: { schema: SchemaOverview | null }) {
        cleanup();

        if (options.schema === null) {
          return reject();
        }

        try {
          setMemorySchemaCache(options.schema);
          resolve(options.schema);
        } catch (e) {
          reject(e);
        }
      }

      function cleanup() {
        bus.unsubscribe(messageKey, busListener).catch(reject);
      }
    });

    // 竞争：超时或收到消息
    return Promise.race([timeout, subscription]).catch(() => getSchema(options, attempt + 1));
  }

  // 步骤 7: 处理进程：实际构建 schema
  let schema: SchemaOverview | null = null;

  try {
    const database = options?.database || getDatabase();
    const schemaInspector = createInspector(database);

    schema = await getDatabaseSchema(database, schemaInspector);  // 核心构建
    setMemorySchemaCache(schema);  // 设置内存缓存
    return schema;
  } finally {
    // 步骤 8: 通知其他进程构建完成
    await bus.publish(messageKey, { schema });
    await lock.delete(lockKey);  // 释放锁
  }
}
```
[api/src/utils/get-schema.ts:22-114](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/get-schema.ts)

### 3.2 关键机制解析

#### 3.2.1 缓存层次

Directus 实现了多层缓存机制：

1. **内存缓存 (`memorySchemaCache`)**：
   - 最快的缓存层，存储在进程内存中
   - 可选的冻结机制（`CACHE_SCHEMA_FREEZE_ENABLED`）
   - 通过 `getMemorySchemaCache()` 和 `setMemorySchemaCache()` 访问

```typescript
let memorySchemaCache: Readonly<SchemaOverview> | null = null;

export function setMemorySchemaCache(schema: SchemaOverview) {
  if (Object.isFrozen(schema)) {
    memorySchemaCache = schema;
  } else {
    memorySchemaCache = freezeSchema(schema);  // 冻结防止意外修改
  }
}

export function getMemorySchemaCache(): Readonly<SchemaOverview> | undefined {
  if (env['CACHE_SCHEMA_FREEZE_ENABLED']) {
    return memorySchemaCache ?? undefined;
  } else if (memorySchemaCache) {
    return unfreezeSchema(memorySchemaCache);  // 解冻返回副本
  }
  return undefined;
}
```
[api/src/cache.ts:133-149](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

2. **本地 Schema 缓存 (`localSchemaCache`)**：
   - 使用 Keyv 抽象，支持内存或 Redis 存储
   - 用于缓存外键信息等中间数据
   - 通过 `getCacheValue()` 和 `setCacheValue()` 访问

```typescript
// 在 RelationsService.foreignKeys() 中的使用
async foreignKeys(collection?: string) {
  const schemaCacheIsEnabled = Boolean(env['CACHE_SCHEMA']);
  let foreignKeys: ForeignKey[] | null = null;

  if (schemaCacheIsEnabled) {
    foreignKeys = await getCacheValue(this.schemaCache, 'foreignKeys');
  }

  if (!foreignKeys) {
    foreignKeys = await this.schemaInspector.foreignKeys();

    if (schemaCacheIsEnabled) {
      setCacheValue(this.schemaCache, 'foreignKeys', foreignKeys);
    }
  }
  // ...
}
```
[api/src/services/relations.ts:65-87](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/relations.ts)

#### 3.2.2 并发控制机制

Directus 使用**分布式锁 + 消息总线**的组合来处理并发：

**锁机制 (`useLock`)**：
- 支持本地锁（单实例）或 Redis 锁（多实例）
- 使用原子递增操作 (`lock.increment()`) 来分配"处理者"
- 只有 `processId === 1` 的进程是实际的构建者

```typescript
// useLock 的实现
export const useLock = () => {
  if (_cache.lock) {
    return _cache.lock;
  }

  if (redisConfigAvailable()) {
    // 多实例场景：使用 Redis 锁
    const env = useEnv();
    _cache.lock = createKv({
      type: 'redis',
      redis: useRedis(),
      namespace: (env['REDIS_LOCK_NAMESPACE'] as string) ?? 'directus:lock',
    });
  } else {
    // 单实例场景：使用本地锁
    _cache.lock = createKv({ type: 'local' });
  }

  return _cache.lock;
};
```
[api/src/lock/lib/use-lock.ts:12-30](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/lock/lib/use-lock.ts)

**消息总线 (`useBus`)**：
- 支持本地事件（单实例）或 Redis Pub/Sub（多实例）
- 用于构建完成后的通知机制

```typescript
// useBus 的实现
export const useBus = () => {
  if (_cache.bus) {
    return _cache.bus;
  }

  if (redisConfigAvailable()) {
    // 多实例场景：使用 Redis Pub/Sub
    const env = useEnv();
    _cache.bus = createBus({
      type: 'redis',
      redis: useRedis(),
      namespace: (env['REDIS_BUS_NAMESPACE'] as string) ?? 'directus:bus',
    });
  } else {
    // 单实例场景：使用本地事件总线
    _cache.bus = createBus({ type: 'local' });
  }

  return _cache.bus;
};
```
[api/src/bus/lib/use-bus.ts:13-31](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/bus/lib/use-bus.ts)

**并发流程示意**：

```
时间线 ──────────────────────────────────────────────────────────────────►

进程 A (先到达)                    进程 B, C, D (后到达)
     │                                   │
     ▼                                   ▼
┌─────────┐                         ┌─────────┐
│ 检查缓存 │                         │ 检查缓存 │
│  未命中  │                         │  未命中  │
└────┬────┘                         └────┬────┘
     │                                   │
     ▼                                   ▼
┌─────────────┐                     ┌─────────────┐
│ lock.incre- │                     │ lock.incre- │
│ ment() = 1  │                     │ ment() = 2,3,4│
└──────┬──────┘                     └──────┬──────┘
       │                                   │
       ▼                                   ▼
┌─────────────────┐                 ┌─────────────────┐
│ processId === 1 │                 │ processId !== 1 │
│  是构建者       │                 │  是等待者       │
└────────┬────────┘                 └────────┬────────┘
         │                                   │
         ▼                                   ▼
┌──────────────────┐                ┌──────────────────┐
│ getDatabaseSchema│                │ 订阅消息总线     │
│  实际构建 schema  │                │ 等待 'done' 消息 │
└────────┬─────────┘                └────────┬─────────┘
         │                                   │
         ▼                                   │
┌──────────────────┐                         │
│ 设置内存缓存      │                         │
│ 发布 'done' 消息 │◄────────────────────────┘
│ 释放锁           │
└────────┬─────────┘
         │
         ▼
    返回 schema
```

#### 3.2.3 重试机制

Directus 实现了完善的重试机制：

1. **最大重试次数**：`MAX_ATTEMPTS = 3`
2. **重试触发条件**：
   - 超时（`CACHE_SCHEMA_SYNC_TIMEOUT`）
   - 消息总线收到 `schema: null`（构建失败）
   - 其他异常
3. **重试保护**：防止无限循环的检查

```typescript
// 重试入口
return Promise.race([timeout, subscription]).catch(() => getSchema(options, attempt + 1));

// 防止无限循环
if (attempt >= MAX_ATTEMPTS) {
  throw new Error(`Failed to get Schema information: hit infinite loop`);
}
```
[api/src/utils/get-schema.ts:51-98](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/get-schema.ts)

## 4. 跨实例同步机制

在多实例部署场景下，Directus 需要确保所有实例的 schema 缓存保持一致。

### 4.1 同步触发点

Schema 缓存的清除和同步在以下情况触发：

1. **Schema 变更操作**：
   - 创建/更新/删除集合
   - 创建/更新/删除字段
   - 创建/更新/删除关系

2. **手动触发**：
   - 调用 `clearSystemCache()`

### 4.2 同步流程

```typescript
// 缓存清除和同步的核心函数
export async function clearSystemCache(opts?: {
  forced?: boolean | undefined;
  autoPurgeCache?: false | undefined;
}): Promise<void> {
  const { systemCache, localSchemaCache, lockCache } = getCache();

  // 步骤 1: 刷新系统缓存（带锁保护）
  if (opts?.forced || !(await lockCache.get('system-cache-lock'))) {
    await lockCache.set('system-cache-lock', true, 10000);  // 加锁 10 秒
    await systemCache.clear();
    await lockCache.delete('system-cache-lock');
  }

  // 步骤 2: 清除本地 schema 缓存
  await localSchemaCache.clear();
  memorySchemaCache = null;  // 清除内存缓存

  // 步骤 3: 清除权限缓存（依赖 schema）
  await clearPermissionCache();

  // 步骤 4: 发布消息，通知其他实例
  messenger.publish<CacheMessage>('schemaChanged', { autoPurgeCache: opts?.autoPurgeCache });
}
```
[api/src/cache.ts:97-117](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

### 4.3 跨实例消息监听

每个实例在启动时订阅 `schemaChanged` 消息：

```typescript
// 在 cache.ts 中的订阅逻辑
if (redisConfigAvailable() && !messengerSubscribed) {
  messengerSubscribed = true;

  messenger.subscribe<CacheMessage>('schemaChanged', async (opts) => {
    // 如果使用内存缓存且配置了自动清除
    if (env['CACHE_STORE'] === 'memory' && env['CACHE_AUTO_PURGE'] && cache && opts?.['autoPurgeCache'] !== false) {
      await cache.clear();
    }

    // 关键：清除所有实例的 schema 缓存
    await localSchemaCache?.clear();
    memorySchemaCache = null;
  });
}
```
[api/src/cache.ts:41-52](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

### 4.4 同步机制总结

| 场景 | 同步方式 |
|------|----------|
| 单实例部署 | 本地事件总线 + 内存缓存直接清除 |
| 多实例 + Redis | Redis Pub/Sub 发布 `schemaChanged` 消息 |
| 并发构建时 | Redis 锁 + Redis Pub/Sub 通知构建结果 |

## 5. 元数据合并：getDatabaseSchema() 详解

`getDatabaseSchema()` 是实际构建 schema 的核心函数，它将数据库原始 schema 与 Directus 元数据合并。

### 5.1 完整构建流程

```typescript
async function getDatabaseSchema(database: Knex, schemaInspector: SchemaInspector): Promise<SchemaOverview> {
  const env = useEnv();

  // 步骤 1: 初始化结果对象
  const result: SchemaOverview = {
    collections: {},
    relations: [],
  };

  // 步骤 2: 获取系统字段行（内置字段配置）
  const systemFieldRows = getSystemFieldRowsWithAuthProviders();

  // 步骤 3: 从数据库获取原始 schema 概览
  // 这会调用 schemaInspector.overview()，底层是方言实现
  const schemaOverview = await schemaInspector.overview();

  // 步骤 4: 获取集合元数据（从 directus_collections 表 + 系统内置集合）
  const collections = [
    ...(await database
      .select('collection', 'singleton', 'note', 'sort_field', 'accountability')
      .from('directus_collections')),
    ...systemCollectionRows,  // 系统内置集合，如 directus_users
  ];

  // 步骤 5: 遍历所有数据库表，构建集合信息
  for (const [collection, info] of Object.entries(schemaOverview)) {
    // 过滤：排除配置的表
    if (toArray(env['DB_EXCLUDE_TABLES']).includes(collection)) {
      logger.trace(`Collection "${collection}" is configured to be excluded and will be ignored`);
      continue;
    }

    // 过滤：排除没有主键的表
    if (!info.primary) {
      logger.warn(`Collection "${collection}" doesn't have a primary key column and will be ignored`);
      continue;
    }

    // 过滤：排除名称含空格的表
    if (collection.includes(' ')) {
      logger.warn(`Collection "${collection}" has a space in the name and will be ignored`);
      continue;
    }

    // 查找对应的元数据
    const collectionMeta = collections.find((collectionMeta) => collectionMeta.collection === collection);

    // 构建集合信息，合并数据库 schema 和元数据
    result.collections[collection] = {
      collection,
      primary: info.primary,  // 来自数据库 schema
      singleton: toBoolean(collectionMeta?.singleton),  // 来自元数据
      note: collectionMeta?.note || null,  // 来自元数据
      sortField: collectionMeta?.sort_field || null,  // 来自元数据
      accountability: collectionMeta ? collectionMeta.accountability : 'all',  // 来自元数据
      fields: mapValues(schemaOverview[collection]?.columns, (column) => {
        // 构建字段信息，从数据库列推断
        return {
          field: column.column_name,
          defaultValue: getDefaultValue(column) ?? null,
          nullable: column.is_nullable ?? true,
          generated: column.is_generated ?? false,
          type: getLocalType(column),  // 关键：从数据库类型推断 Directus 类型
          dbType: column.data_type,
          precision: column.numeric_precision || null,
          scale: column.numeric_scale || null,
          special: [],
          note: null,
          validation: null,
          alias: false,
          searchable: true,
        };
      }),
    };
  }

  // 步骤 6: 获取字段元数据（从 directus_fields 表 + 系统内置字段）
  const fields = [
    ...(await database
      .select<{/* 字段类型 */}>('id', 'collection', 'field', 'special', 'note', 'validation', 'searchable')
      .from('directus_fields')),
    ...systemFieldRows,
  ].filter((field) => (field.special ? toArray(field.special) : []).includes('no-data') === false);

  // 步骤 7: 用元数据更新字段信息
  for (const field of fields) {
    if (!result.collections[field.collection]) continue;

    const existing = result.collections[field.collection]?.fields[field.field];
    const column = schemaOverview[field.collection]?.columns[field.field];
    const special = field.special ? toArray(field.special) : [];

    // 跳过非别名字段且不存在的字段
    if (ALIAS_TYPES.some((type) => special.includes(type)) === false && !existing) continue;

    // 推断字段类型
    const type = (existing && getLocalType(column, { special })) || 'alias';
    let validation = field.validation ?? null;

    if (validation && typeof validation === 'string') validation = parseJSON(validation);

    // 更新字段信息，元数据覆盖默认值
    result.collections[field.collection]!.fields[field.field] = {
      field: field.field,
      defaultValue: existing?.defaultValue ?? null,
      nullable: existing?.nullable ?? true,
      generated: existing?.generated ?? false,
      type: type,  // 可能被元数据修改
      dbType: existing?.dbType || null,
      precision: existing?.precision || null,
      scale: existing?.scale || null,
      special: special,  // 来自元数据，如 'alias', 'file', 'm2o' 等
      note: field.note,  // 来自元数据
      alias: existing?.alias ?? true,
      validation: (validation as Filter) ?? null,  // 来自元数据
      searchable: toBoolean(field.searchable) ?? true,  // 来自元数据
    };
  }

  // 步骤 8: 获取并构建关系
  const relationsService = new RelationsService({ knex: database, schema: result });
  result.relations = await relationsService.readAll(undefined, undefined, true);

  return result;
}
```
[api/src/utils/get-schema.ts:116-232](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/get-schema.ts)

### 5.2 关系合并机制

关系的合并在 `RelationsService` 中处理，它将数据库外键与 Directus 关系元数据合并。

```typescript
// RelationsService.stitchRelations() - 核心合并函数
private stitchRelations(metaRows: RelationMeta[], schemaRows: ForeignKey[]) {
  // 步骤 1: 从数据库外键构建关系
  const results = schemaRows.map((foreignKey): Relation => {
    return {
      collection: foreignKey.table,
      field: foreignKey.column,
      related_collection: foreignKey.foreign_key_table,
      schema: foreignKey,  // 数据库外键信息
      meta:  // 查找对应的元数据
        metaRows.find((meta) => {
          if (meta.many_collection !== foreignKey.table) return false;
          if (meta.many_field !== foreignKey.column) return false;
          if (meta.one_collection && meta.one_collection !== foreignKey.foreign_key_table) return false;
          return true;
        }) || null,
    };
  });

  // 步骤 2: 处理没有对应外键的元数据（如别名关系、M2M 的 O2M 端）
  const remainingMetaRows = metaRows
    .filter((meta) => {
      return !results.find((relation) => relation.meta === meta);
    })
    .map((meta): Relation => {
      return {
        collection: meta.many_collection,
        field: meta.many_field,
        related_collection: meta.one_collection ?? null,
        schema: null,  // 没有数据库外键
        meta: meta,
      };
    });

  results.push(...remainingMetaRows);
  return results;
}
```
[api/src/services/relations.ts:526-563](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/relations.ts)

### 5.3 类型推断机制

`getLocalType()` 函数负责从数据库列类型推断 Directus 字段类型：

```typescript
// 这是一个关键的类型映射点
// 从 Column.data_type 映射到 Directus 的 Type

// 示例：MySQL 的 tinyint(1) 被映射为 boolean
// 在 MySQLSchemaInspector.rawColumnToColumn() 中：
if (rawColumn.COLUMN_TYPE.startsWith('tinyint(1)')) {
  dataType = 'boolean';
}

// 然后在 getLocalType() 中进一步映射到 Directus 类型
```

## 6. 多数据库差异抹平的正确层次

### 6.1 差异抹平的三层架构

之前的描述有偏差，正确的差异抹平发生在三个层次：

| 层次 | 组件 | 抹平的差异 | 输出 |
|------|------|-----------|------|
| 第一层 | `SchemaInspector` 方言实现 | SQL 语法、系统表结构、类型命名 | `Column`、`Table`、`ForeignKey` |
| 第二层 | `getLocalType()` 等函数 | 数据库类型到 Directus 类型的映射 | Directus `Type` |
| 第三层 | `sanitize-*` 函数 | 移除数据库特有属性，标准化结构 | 用于快照/比较的标准化对象 |

### 6.2 第一层：SchemaInspector 方言层

这是最底层的差异抹平，每个方言实现处理：

1. **不同的系统表查询**：
   - MySQL: `INFORMATION_SCHEMA.COLUMNS`、`INFORMATION_SCHEMA.KEY_COLUMN_USAGE` 等
   - PostgreSQL: `pg_class`、`pg_attribute`、`pg_constraint` 等系统目录
   - SQLite: `sqlite_master`、`PRAGMA` 命令

2. **不同的类型命名**：
   - MySQL: `tinyint(1)` → `boolean`
   - PostgreSQL: `character varying` → `varchar`（在某些语境下）

3. **不同的元数据获取方式**：
   - 自增列检测
   - 生成列检测
   - 注释获取

### 6.3 第二层：类型映射层

`getLocalType()` 函数（在 `get-local-type.ts` 中）将标准化的 `Column.data_type` 进一步映射到 Directus 的业务类型。

这一层处理：
- `varchar` → `string`
- `int` → `integer`
- `boolean` → `boolean`
- 等等

### 6.4 第三层：标准化层

`sanitize-*` 函数用于**快照和比较**场景，移除所有数据库特有属性：

```typescript
export function sanitizeColumn(column: Column) {
  return pick(column, [
    'name', 'table', 'data_type', 'default_value',
    'max_length', 'numeric_precision', 'numeric_scale',
    'is_nullable', 'is_unique', 'is_indexed',
    'is_primary_key', 'is_generated', 'generation_expression',
    'has_auto_increment', 'foreign_key_table', 'foreign_key_column',
    // 注意：这里没有包含：
    // - schema (PostgreSQL 特有)
    // - foreign_key_schema (PostgreSQL 特有)
    // - comment (不是所有数据库都支持)
  ]);
}
```
[api/src/utils/sanitize-schema.ts:59-78](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/sanitize-schema.ts)

**重要**：这一层不是用于"抹平差异供业务层使用"，而是用于**schema 快照和版本比较**。业务层使用的是完整的 `Column` 对象，包括所有数据库特有属性。

## 7. 完整的 Schema 刷新路径

### 7.1 从请求到响应的完整路径

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           HTTP 请求进入                                    │
│                         (GET /items/users)                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      schema 中间件 (middleware/schema.ts)                 │
│                                                                           │
│  const schema: RequestHandler = asyncHandler(async (req, _res, next) => {│
│    req.schema = await getSchema();  // 关键调用                            │
│    return next();                                                         │
│  });                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         getSchema() 入口                                   │
│                                                                           │
│  1. 检查 bypassCache 或 CACHE_SCHEMA=false                                │
│  2. 检查 memorySchemaCache (最快路径)                                      │
│  3. 未命中 → 获取锁 → 竞争处理权                                           │
│  4. 处理者 → 调用 getDatabaseSchema()                                      │
│  5. 等待者 → 订阅消息总线等待                                              │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     getDatabaseSchema() 核心构建                           │
│                                                                           │
│  1. schemaInspector.overview() → 数据库原始 schema                         │
│     └─→ 触发具体方言实现（MySQL/PostgreSQL 等）                             │
│                                                                           │
│  2. 查询 directus_collections → 集合元数据                                 │
│  3. 查询 directus_fields → 字段元数据                                      │
│  4. 合并 schema + 元数据 → 构建 collections 和 fields                      │
│                                                                           │
│  5. RelationsService.readAll() → 关系数据                                  │
│     └─→ schemaInspector.foreignKeys() → 数据库外键                         │
│     └─→ 查询 directus_relations → 关系元数据                               │
│     └─→ stitchRelations() → 合并外键与元数据                               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         缓存设置与通知                                      │
│                                                                           │
│  处理者:                                                                   │
│  1. setMemorySchemaCache(schema) → 设置内存缓存                            │
│  2. bus.publish('schemaCache--done', { schema }) → 通知等待者              │
│  3. lock.delete(lockKey) → 释放锁                                         │
│                                                                           │
│  等待者:                                                                   │
│  1. 收到消息 → setMemorySchemaCache(options.schema)                        │
│  2. 取消订阅 → 返回 schema                                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         业务层使用 schema                                   │
│                                                                           │
│  req.schema 现在包含:                                                      │
│  - collections: { [collectionName]: CollectionInfo }                      │
│    └─→ fields: { [fieldName]: FieldInfo }                                 │
│    └─→ primary, singleton, note, 等                                       │
│                                                                           │
│  - relations: Relation[]                                                   │
│    └─→ collection, field, related_collection                               │
│    └─→ schema: ForeignKey (数据库外键)                                     │
│    └─→ meta: RelationMeta (Directus 元数据)                               │
│                                                                           │
│  使用场景:                                                                 │
│  - 权限检查: 验证用户对 collection/field 的访问权限                         │
│  - 查询构建: 构建正确的 SQL，处理 JOIN 和关系                               │
│  - 响应格式化: 按照字段配置格式化输出数据                                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Schema 变更触发的刷新路径

当 schema 发生变更（如添加字段、创建关系）时：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      变更操作执行                                          │
│  (如: POST /fields, PATCH /relations/:collection/:field)                  │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      服务层处理                                            │
│  (FieldsService.createOne, RelationsService.createOne 等)                 │
│                                                                           │
│  在 finally 块中:                                                          │
│  if (opts?.autoPurgeSystemCache !== false) {                             │
│    await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });     │
│  }                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   clearSystemCache() 执行                                  │
│                                                                           │
│  1. 获取锁 (防止并发清除)                                                  │
│  2. systemCache.clear() → 清除系统缓存                                     │
│  3. localSchemaCache.clear() → 清除本地 schema 缓存                        │
│  4. memorySchemaCache = null → 清除内存缓存                                │
│  5. clearPermissionCache() → 清除权限缓存（依赖 schema）                    │
│  6. messenger.publish('schemaChanged', {...}) → 发布同步消息               │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    跨实例同步 (多实例场景)                                  │
│                                                                           │
│  Redis Pub/Sub:                                                           │
│  实例 A ──publish('schemaChanged')──► Redis                              │
│                                    │                                       │
│  实例 B ◄──subscribe('schemaChanged')──┘                                  │
│  实例 C ◄──subscribe('schemaChanged')──┘                                  │
│  实例 D ◄──subscribe('schemaChanged')──┘                                  │
│                                                                           │
│  收到消息后的处理:                                                          │
│  await localSchemaCache?.clear();                                         │
│  memorySchemaCache = null;                                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    下次请求时重新构建                                       │
│                                                                           │
│  下一个 HTTP 请求:                                                         │
│  1. schema 中间件调用 getSchema()                                          │
│  2. getMemorySchemaCache() 返回 null (已被清除)                            │
│  3. 触发完整的构建流程                                                      │
│  4. 获取最新的 schema                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

## 8. 关键设计模式与架构决策

### 8.1 设计模式应用

| 设计模式 | 应用场景 | 实现位置 |
|---------|---------|---------|
| **适配器模式** | 方言适配，统一不同数据库的 schema 访问 | `SchemaInspector` 接口 + 各方言实现 |
| **工厂模式** | 根据数据库类型创建对应的 Inspector | `createInspector()` 函数 |
| **观察者模式** | 消息总线，跨实例通知 | `useBus()` + Redis Pub/Sub |
| **双重检查锁** | 并发控制，单进程构建多进程等待 | `getSchema()` 中的锁 + 等待机制 |
| **策略模式** | 不同存储后端的选择 | `useLock()`、`useBus()` 中的 Redis/本地选择 |

### 8.2 关键架构决策

1. **多层缓存策略**：
   - 内存缓存（最快）+ Keyv 缓存（持久化）
   - 权衡：速度 vs 内存使用 vs 跨实例共享

2. **单进程构建 + 多进程等待**：
   - 避免多个进程同时构建相同的 schema
   - 权衡：锁开销 vs 重复计算浪费

3. **消息总线解耦**：
   - 构建者完成后通知等待者
   - 变更时通知所有实例清除缓存
   - 权衡：消息传递开销 vs 轮询开销

4. **SchemaInspector 接口抽象**：
   - 完全隔离数据库差异
   - 新增数据库支持只需添加新的方言实现
   - 权衡：抽象层开销 vs 可维护性

5. **元数据与 schema 分离但合并使用**：
   - 数据库 schema 是基础
   - Directus 元数据扩展 schema
   - 运行时合并为完整的 `SchemaOverview`
   - 权衡：合并开销 vs 灵活性

## 9. 配置参数

Schema 推断和缓存机制受以下环境变量影响：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `CACHE_SCHEMA` | boolean | true | 是否启用 schema 缓存 |
| `CACHE_SCHEMA_FREEZE_ENABLED` | boolean | - | 是否启用 schema 冻结（防止意外修改） |
| `CACHE_SCHEMA_MAX_ITERATIONS` | number | - | 锁计数器的最大值，防止溢出 |
| `CACHE_SCHEMA_SYNC_TIMEOUT` | number | - | 等待其他进程构建 schema 的超时时间（毫秒） |
| `CACHE_STORE` | string | 'memory' | 缓存存储类型：'memory' 或 'redis' |
| `CACHE_NAMESPACE` | string | - | 缓存键的命名空间 |
| `REDIS_LOCK_NAMESPACE` | string | 'directus:lock' | Redis 锁的命名空间 |
| `REDIS_BUS_NAMESPACE` | string | 'directus:bus' | Redis 消息总线的命名空间 |
| `SYNCHRONIZATION_STORE` | string | - | 同步管理器存储类型（用于 `SynchronizedClock`） |
| `DB_EXCLUDE_TABLES` | array | - | 排除的表名列表，不会被 Directus 管理 |

## 10. 总结

### 10.1 核心机制回顾

Directus 的 Schema 推断机制包含以下核心组件：

1. **SchemaInspector 方言层**：
   - 统一接口，多种数据库实现
   - 第一层差异抹平，输出 `Column`、`Table`、`ForeignKey`

2. **缓存与并发控制**：
   - 多层缓存：内存缓存 + Keyv 缓存
   - 分布式锁：控制并发构建
   - 消息总线：跨进程/实例通知
   - 重试机制：最多 3 次重试 + 超时保护

3. **元数据合并**：
   - `getDatabaseSchema()` 合并数据库 schema 与 Directus 元数据
   - `stitchRelations()` 合并数据库外键与关系元数据
   - `getLocalType()` 类型映射

4. **跨实例同步**：
   - `schemaChanged` 消息通知所有实例清除缓存
   - Redis Pub/Sub 实现多实例通信
   - 单实例时使用本地事件总线

### 10.2 之前描述的偏差纠正

1. **关于"差异抹平"**：
   - ❌ 之前：`sanitize-*` 函数是第二层差异抹平
   - ✅ 正确：`sanitize-*` 函数是用于**快照和比较**的标准化，不是运行时的差异抹平
   - 运行时业务层使用的是完整的 `Column` 对象，包括数据库特有属性

2. **关于"缓存层次"**：
   - ❌ 之前：没有详细描述缓存机制
   - ✅ 正确：Directus 有多层缓存，内存缓存最快，Keyv 缓存支持持久化

3. **关于"并发控制"**：
   - ❌ 之前：没有描述并发机制
   - ✅ 正确：使用分布式锁 + 消息总线实现"单进程构建，多进程等待"

4. **关于"跨实例同步"**：
   - ❌ 之前：没有描述同步机制
   - ✅ 正确：使用 `schemaChanged` 消息 + Redis Pub/Sub 实现多实例缓存同步

### 10.3 代码位置索引

| 功能 | 文件位置 |
|------|---------|
| Schema 获取入口 | [api/src/utils/get-schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/get-schema.ts) |
| Schema 中间件 | [api/src/middleware/schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/middleware/schema.ts) |
| 缓存管理 | [api/src/cache.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts) |
| 分布式锁 | [api/src/lock/lib/use-lock.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/lock/lib/use-lock.ts) |
| 消息总线 | [api/src/bus/lib/use-bus.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/bus/lib/use-bus.ts) |
| 关系服务 | [api/src/services/relations.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/relations.ts) |
| Schema 标准化 | [api/src/utils/sanitize-schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/sanitize-schema.ts) |
| SchemaInspector 工厂 | [packages/schema/src/index.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/index.ts) |
| MySQL 方言 | [packages/schema/src/dialects/mysql.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/mysql.ts) |
| PostgreSQL 方言 | [packages/schema/src/dialects/postgres.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts) |
| 同步管理器 | [api/src/synchronization.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/synchronization.ts) |
