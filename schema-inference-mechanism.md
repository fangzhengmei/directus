# Directus Schema 推断机制深度分析

## 1. 概述

Directus 的 Schema 推断机制是一个设计精良的多层架构系统，它不仅能够从现有数据库中自动推断表结构、字段类型和关系，还包含了复杂的缓存并发控制、重试机制、跨实例同步以及完整的元数据合并刷新路径。

本报告深入分析 Directus Schema 推断的完整运行时机制，重点包括：
- 缓存机制与并发控制（**已纠正**）
- 重试机制与容错处理
- 跨实例同步信号机制（**已纠正**）
- 推断结果与元数据合并的完整路径
- Schema 刷新链路的触发条件、失效与重建衔接（**已纠正**）
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
│  - getMemorySchemaCache() / setMemorySchemaCache(): 进程内内存缓存（最快）          │
│  - localSchemaCache: Keyv 缓存（**硬编码 memory，不支持 Redis**）                  │
│  - useLock(): 分布式锁（遵循 CACHE_STORE，支持本地或 Redis）                        │
│  - useBus(): 消息总线（遵循 Redis 配置，支持本地事件或 Redis Pub/Sub）               │
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

### 2.2 关键组件职责（已纠正）

| 层次 | 组件 | 主要职责 | 存储后端 |
|------|------|----------|----------|
| 业务层 | `SchemaOverview` | 完整的业务 schema 视图，包含集合、字段、关系 | - |
| 业务层 | `schema` 中间件 | 每个 HTTP 请求获取 schema 并附加到 `req.schema` | - |
| 缓存层 | `getSchema()` | schema 获取的入口函数，处理缓存、并发、重试 | - |
| 缓存层 | `memorySchemaCache` | 进程内内存缓存，最快的缓存层 | **进程内存** |
| 缓存层 | `localSchemaCache` | Keyv 缓存，用于缓存外键等中间数据 | **硬编码 memory（不支持 Redis）** |
| 缓存层 | `useLock()` | 分布式锁，控制并发构建 | 遵循 `CACHE_STORE` |
| 缓存层 | `useBus()` | 消息总线，跨进程/实例通知 | 遵循 Redis 配置 |
| 元数据层 | `getDatabaseSchema()` | 实际构建 schema，合并数据库与元数据 | - |
| 元数据层 | `RelationsService` | 关系数据的获取与合并 | - |
| Inspector层 | `SchemaInspector` | 方言适配，数据库差异抹平 | - |

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

  // 步骤 2: 检查内存缓存（最快路径）
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

#### 3.2.1 缓存层次（已纠正）

Directus 实现了多层缓存机制，但各层的存储后端不同：

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

2. **本地 Schema 缓存 (`localSchemaCache`) - 重要纠偏**：

   **⚠️ 关键发现：`localSchemaCache` 硬编码使用 `'memory'` 存储，不遵循 `CACHE_STORE` 环境变量！**

   看源码 `cache.ts` 第 78-81 行：
   ```typescript
   if (localSchemaCache === null) {
     localSchemaCache = getKeyvInstance('memory', getMilliseconds(env['CACHE_SYSTEM_TTL']), '_schema');
     // 第一个参数是硬编码的 'memory'，不是 env['CACHE_STORE']！
     localSchemaCache.on('error', (err) => logger.warn(err, `[schema-cache] ${err}`));
   }
   ```

   对比其他缓存（都遵循 `CACHE_STORE`）：
   - `systemCache`: 第 68 行：`getKeyvInstance(env['CACHE_STORE'] as Store, ...)`
   - `deploymentCache`: 第 74 行：`getKeyvInstance(env['CACHE_STORE'] as Store, ...)`
   - `lockCache`: 第 84 行：`getKeyvInstance(env['CACHE_STORE'] as Store, ...)`
   - **`localSchemaCache`**: 第 79 行：`getKeyvInstance('memory', ...)` - **硬编码 memory！**

   `localSchemaCache` 的实际用途：
   - 用于缓存 `foreignKeys` 等中间数据（见 `RelationsService.foreignKeys()`）
   - 是**进程内的本地缓存**，不用于跨实例共享
   - 跨实例同步是通过 `schemaChanged` 消息总线实现的，不是通过共享缓存

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
- 支持本地锁（单实例）或 Redis 锁（多实例）- **遵循 `CACHE_STORE`**
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

## 4. Schema 刷新链路详解（已纠正）

### 4.1 触发条件：谁调用了 clearSystemCache？

Schema 缓存的清除由 `clearSystemCache()` 函数触发，该函数被以下服务调用：

| 服务 | 文件位置 | 触发场景 |
|------|----------|----------|
| **CollectionsService** | [api/src/services/collections.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/collections.ts) | 创建/更新/删除集合 (248, 299, 494, 553, 603, 787, 835行) |
| **FieldsService** | [api/src/services/fields.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/fields.ts) | 创建/更新/删除字段 (489, 644, 683, 891行) |
| **RelationsService** | [api/src/services/relations.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/relations.ts) | 创建/更新/删除关系 (295, 422, 508行) |
| **PermissionsService** | [api/src/services/permissions.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/permissions.ts) | 权限变更 (27行) |
| **AccessService** | [api/src/services/access.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/access.ts) | 访问控制变更 (12行) |
| **RolesService** | [api/src/services/roles.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/roles.ts) | 角色变更 (122行) |
| **UsersService** | [api/src/services/users.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/users.ts) | 用户变更 (651行) |
| **PoliciesService** | [api/src/services/policies.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/policies.ts) | 策略变更 (15行) |
| **UtilsService** | [api/src/services/utils.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/utils.ts) | 手动清除缓存端点 (167行) |
| **CLI** | [api/src/cli/commands/cache/clear.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cli/commands/cache/clear.ts) | `cache:clear` 命令 (20行) |
| **GraphQL Resolvers** | [api/src/services/graphql/resolvers/system-global.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/graphql/resolvers/system-global.ts) | GraphQL 系统变更 (431行) |
| **AI Tools** | [api/src/ai/tools/fields/index.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/ai/tools/fields/index.ts) | AI 工具操作 (116, 180行) |

### 4.2 失效机制：clearSystemCache() 做了什么？

```typescript
export async function clearSystemCache(opts?: {
  forced?: boolean | undefined;
  autoPurgeCache?: false | undefined;
}): Promise<void> {
  const { systemCache, localSchemaCache, lockCache } = getCache();

  // 步骤 1: 刷新系统缓存（带锁保护，防止并发清除）
  // 只有当 forced=true 或 system-cache-lock 不存在时才执行
  if (opts?.forced || !(await lockCache.get('system-cache-lock'))) {
    await lockCache.set('system-cache-lock', true, 10000);  // 加锁 10 秒
    await systemCache.clear();  // 清除系统缓存
    await lockCache.delete('system-cache-lock');  // 释放锁
  }

  // 步骤 2: 清除本地 schema 缓存（Keyv 内存缓存）
  await localSchemaCache.clear();

  // 步骤 3: 清除内存缓存（进程内最快缓存）
  memorySchemaCache = null;

  // 步骤 4: 清除权限缓存（因为权限检查依赖 schema）
  await clearPermissionCache();

  // 步骤 5: 发布消息，通知其他实例（关键：跨实例同步）
  messenger.publish<CacheMessage>('schemaChanged', { autoPurgeCache: opts?.autoPurgeCache });
}
```
[api/src/cache.ts:97-117](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

### 4.3 跨实例同步机制（重要纠偏）

**⚠️ 关键发现：缓存不是跨实例共享的，而是通过消息总线同步"失效信号"！**

之前的描述可能暗示缓存是共享的，但实际架构是：

1. **每个实例有自己独立的缓存**：
   - `memorySchemaCache`：每个进程自己的内存
   - `localSchemaCache`：每个实例自己的 Keyv 内存缓存（硬编码 memory）

2. **同步的是"失效信号"，不是缓存数据**：
   - 实例 A 变更 schema → 清除**自己的**缓存 → 发布 `schemaChanged` 消息
   - 其他实例收到消息 → 清除**各自的**缓存
   - 下次请求时，每个实例**各自**重建缓存

**完整的跨实例同步流程**：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           实例 A (执行变更的实例)                                   │
│                                                                                   │
│  1. 用户操作: POST /fields (创建新字段)                                           │
│                    │                                                              │
│                    ▼                                                              │
│  2. FieldsService.createOne() 执行数据库变更                                      │
│                    │                                                              │
│                    ▼                                                              │
│  3. 调用 clearSystemCache():                                                      │
│     ├─→ await localSchemaCache.clear()     ← 清除自己的 Keyv 缓存                │
│     ├─→ memorySchemaCache = null           ← 清除自己的内存缓存                   │
│     └─→ messenger.publish('schemaChanged', {...})  ← 发布消息！                  │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                              │
                              │ Redis Pub/Sub
                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Redis (消息中间件)                                           │
│                                                                                   │
│  PUBLISH directus:bus:schemaChanged {autoPurgeCache: undefined}                  │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│    实例 B        │ │    实例 C        │ │    实例 D        │
│                  │ │                  │ │                  │
│  订阅者收到消息:  │ │  订阅者收到消息:  │ │  订阅者收到消息:  │
│                  │ │                  │ │                  │
│  messenger.subscribe('schemaChanged', │ │ messenger.subscribe('schemaChanged', │ │ messenger.subscribe('schemaChanged', │
│    async (opts) => {                 │ │   async (opts) => {                 │ │   async (opts) => {                 │
│      await localSchemaCache?.clear();│ │     await localSchemaCache?.clear();│ │     await localSchemaCache?.clear();│
│      memorySchemaCache = null;       │ │     memorySchemaCache = null;       │ │     memorySchemaCache = null;       │
│    })                                 │ │   })                                 │ │   })                                 │
│                  │ │                  │ │                  │
│  各自清除自己的缓存 │ │  各自清除自己的缓存 │ │  各自清除自己的缓存 │
└──────────────────┘ └──────────────────┘ └──────────────────┘
```

**消息订阅的源码**（cache.ts 第 41-51 行）：

```typescript
// 只有当 Redis 可用时才订阅（单实例时使用本地事件总线）
if (redisConfigAvailable() && !messengerSubscribed) {
  messengerSubscribed = true;

  messenger.subscribe<CacheMessage>('schemaChanged', async (opts) => {
    // 如果使用内存缓存且配置了自动清除，清除 API 响应缓存
    if (env['CACHE_STORE'] === 'memory' && env['CACHE_AUTO_PURGE'] && cache && opts?.['autoPurgeCache'] !== false) {
      await cache.clear();
    }

    // 关键：每个实例各自清除自己的 schema 缓存
    await localSchemaCache?.clear();
    memorySchemaCache = null;
  });
}
```
[api/src/cache.ts:41-52](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

### 4.4 失效与重建的衔接

缓存失效后，重建发生在**下一次请求**时，衔接流程如下：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        缓存失效阶段 (已执行 clearSystemCache)                      │
│                                                                                   │
│  所有实例的状态:                                                                   │
│  - memorySchemaCache = null                                                       │
│  - localSchemaCache 已被 clear()                                                  │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ 下一个 HTTP 请求到达
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        重建触发阶段                                                │
│                                                                                   │
│  1. 请求进入 → schema 中间件执行                                                  │
│                                                                                   │
│     const schema: RequestHandler = asyncHandler(async (req, _res, next) => {  │
│       req.schema = await getSchema();  // 关键调用                               │
│       return next();                                                              │
│     });                                                                           │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        getSchema() 执行流程                                        │
│                                                                                   │
│  1. 检查 bypassCache 或 CACHE_SCHEMA=false → 否                                  │
│                                                                                   │
│  2. 检查内存缓存:                                                                  │
│     const cached = getMemorySchemaCache();                                       │
│     // 返回 undefined（因为 memorySchemaCache = null）                            │
│                                                                                   │
│  3. 获取锁和消息总线:                                                              │
│     const lock = useLock();                                                       │
│     const bus = useBus();                                                         │
│     const processId = await lock.increment('schemaCache--preparing');            │
│                                                                                   │
│  4. 竞争构建权:                                                                    │
│     - 如果 processId === 1 → 我是构建者                                          │
│     - 如果 processId !== 1 → 我是等待者                                          │
│                                                                                   │
└─────────────────────────────────────────────────────────────────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │ 构建者 (processId=1)  │   等待者 (processId>1) │
            ▼                       ▼
┌──────────────────────┐  ┌────────────────────────────────┐
│ 调用 getDatabaseSchema│  │ 订阅 'schemaCache--done' 消息  │
│ 实际构建 schema       │  │ 等待构建者完成通知             │
│                      │  │                                │
│ 1. schemaInspector.  │  │ 超时: CACHE_SCHEMA_SYNC_TIMEOUT│
│    overview()        │  │                                │
│                      │  │ 收到消息后:                    │
│ 2. 查询 directus_    │  │ setMemorySchemaCache(schema)  │
│    collections       │  │ 返回 schema                    │
│                      │  │                                │
│ 3. 查询 directus_    │  │ 超时/失败: 重试 getSchema()   │
│    fields            │  │                                │
│                      │  └────────────────────────────────┘
│ 4. 合并 fields 元数据│
│                      │
│ 5. RelationsService. │
│    readAll()         │
│                      │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────────┐
│  构建完成后:                                                   │
│                                                               │
│  1. setMemorySchemaCache(schema)  ← 设置自己的内存缓存        │
│                                                               │
│  2. bus.publish('schemaCache--done', { schema })             │
│     ↓                                                         │
│     通知其他等待者（同一实例内的其他请求或其他实例？）          │
│                                                               │
│     ⚠️ 注意：这里的消息总线用途不同！                          │
│     - 'schemaCache--done': 用于同一实例内并发请求的协调        │
│     - 'schemaChanged': 用于跨实例的缓存失效通知               │
│                                                               │
│  3. lock.delete('schemaCache--preparing')  ← 释放锁          │
│                                                               │
│  4. 返回 schema                                               │
└──────────────────────────────────────────────────────────────┘
```

### 4.5 两种消息总线的区别

| 消息键 | 发布时机 | 用途 | 范围 |
|--------|----------|------|------|
| `schemaCache--done` | `getSchema()` 构建完成后 | 同一实例内并发请求的协调：构建者通知等待者 | **进程内/同实例** |
| `schemaChanged` | `clearSystemCache()` 执行后 | 跨实例同步：通知所有实例清除缓存 | **多实例（Redis Pub/Sub）** |

**关键区别**：
- `schemaCache--done`：用于**并发构建**场景，多个请求同时到达时，一个构建，其他等待
- `schemaChanged`：用于**缓存失效**场景，一个实例变更后，通知所有实例清除缓存

### 4.6 完整的刷新链路流程图

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              用户发起 Schema 变更操作                                  │
│                    (如: POST /fields, PATCH /relations)                               │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              服务层执行数据库变更                                        │
│  (CollectionsService / FieldsService / RelationsService 等)                           │
│                                                                                       │
│  在 finally 块中:                                                                      │
│  if (opts?.autoPurgeSystemCache !== false) {                                         │
│    await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });                 │
│  }                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         clearSystemCache() 执行（当前实例）                            │
│                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 清除系统缓存 (带锁保护)                                                      │   │
│  │    if (opts?.forced || !(await lockCache.get('system-cache-lock'))) {         │   │
│  │      await lockCache.set('system-cache-lock', true, 10000);                   │   │
│  │      await systemCache.clear();                                                 │   │
│  │      await lockCache.delete('system-cache-lock');                              │   │
│  │    }                                                                            │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                            │
│                                          ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ 2. 清除 Schema 缓存（当前实例）                                                 │   │
│  │    await localSchemaCache.clear();    ← Keyv 内存缓存                         │   │
│  │    memorySchemaCache = null;           ← 进程内存缓存                          │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                            │
│                                          ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ 3. 清除依赖缓存                                                                │   │
│  │    await clearPermissionCache();  ← 权限缓存依赖 schema                       │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                            │
│                                          ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │ 4. 发布跨实例同步消息                                                          │   │
│  │    messenger.publish<CacheMessage>('schemaChanged', {                         │   │
│  │      autoPurgeCache: opts?.autoPurgeCache                                     │   │
│  │    });                                                                         │   │
│  │                                                                                 │   │
│  │    ↓ 如果是 Redis 环境，这会发布到 Redis Pub/Sub                                │   │
│  │    ↓ 所有订阅的实例都会收到                                                     │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          │ Redis Pub/Sub (多实例场景)
                                          │ 或 本地事件总线 (单实例场景)
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         其他实例收到 'schemaChanged' 消息                            │
│                         (通过 messenger.subscribe 监听)                                │
│                                                                                       │
│  每个实例各自执行:                                                                     │
│  messenger.subscribe<CacheMessage>('schemaChanged', async (opts) => {              │
│                                                                                       │
│    // 可选：清除 API 响应缓存                                                          │
│    if (env['CACHE_STORE'] === 'memory' && env['CACHE_AUTO_PURGE'] && ...) {        │
│      await cache.clear();                                                             │
│    }                                                                                  │
│                                                                                       │
│    // 关键：清除自己的 schema 缓存                                                     │
│    await localSchemaCache?.clear();    ← 自己的 Keyv 缓存                           │
│    memorySchemaCache = null;           ← 自己的内存缓存                              │
│  });                                                                                  │
│                                                                                       │
│  ⚠️ 注意：没有重建，只是清除！重建发生在下一次请求时。                                   │
│                                                                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼ 下一个 HTTP 请求到达任意实例
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                         缓存重建阶段（懒加载）                                          │
│                                                                                       │
│  1. schema 中间件调用 getSchema()                                                    │
│                                                                                       │
│  2. getMemorySchemaCache() 返回 undefined (缓存已失效)                                │
│                                                                                       │
│  3. 获取锁 → 竞争构建权 → 一个构建，其他等待                                          │
│                                                                                       │
│  4. 调用 getDatabaseSchema() 实际重建:                                                │
│     ├─→ schemaInspector.overview()     ← 从数据库获取原始 schema                      │
│     ├─→ 查询 directus_collections     ← 集合元数据                                    │
│     ├─→ 查询 directus_fields          ← 字段元数据                                    │
│     ├─→ 合并 fields 元数据            ← 用元数据更新默认字段信息                       │
│     └─→ RelationsService.readAll()   ← 获取并合并关系                                │
│                                                                                       │
│  5. 设置内存缓存: setMemorySchemaCache(schema)                                        │
│                                                                                       │
│  6. 返回 schema                                                                       │
│                                                                                       │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

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

## 6. 多数据库差异抹平的正确层次

### 6.1 差异抹平的三层架构

之前的描述有偏差，正确的差异抹平发生在三个层次：

| 层次 | 组件 | 抹平的差异 | 输出 | 使用场景 |
|------|------|-----------|------|----------|
| 第一层 | `SchemaInspector` 方言实现 | SQL 语法、系统表结构、类型命名 | `Column`、`Table`、`ForeignKey` | **运行时**：所有需要访问数据库 schema 的场景 |
| 第二层 | `getLocalType()` 等函数 | 数据库类型到 Directus 类型的映射 | Directus `Type` | **运行时**：构建业务 schema 时 |
| 第三层 | `sanitize-*` 函数 | 移除数据库特有属性，标准化结构 | 用于快照/比较的标准化对象 | **快照/版本比较**：不是运行时使用 |

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

### 6.4 第三层：标准化层（重要纠偏）

`sanitize-*` 函数用于**快照和比较**场景，**不是运行时的差异抹平**！

```typescript
export function sanitizeColumn(column: Column) {
  return pick(column, [
    'name', 'table', 'data_type', 'default_value',
    'max_length', 'numeric_precision', 'numeric_scale',
    'is_nullable', 'is_unique', 'is_indexed',
    'is_primary_key', 'is_generated', 'generation_expression',
    'has_auto_increment', 'foreign_key_table', 'foreign_key_column',
    // 注意：这里移除了数据库特有属性：
    // - schema (PostgreSQL 特有)
    // - foreign_key_schema (PostgreSQL 特有)
    // - comment (不是所有数据库都支持)
  ]);
}
```
[api/src/utils/sanitize-schema.ts:59-78](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/sanitize-schema.ts)

**⚠️ 关键纠偏**：
- ❌ 之前的描述：`sanitize-*` 是运行时的差异抹平层
- ✅ 正确理解：`sanitize-*` 函数是用于**schema 快照和版本比较**的标准化工具
- 运行时业务层使用的是**完整的 `Column` 对象**，包括所有数据库特有属性

## 7. 配置参数

Schema 推断和缓存机制受以下环境变量影响：

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `CACHE_SCHEMA` | boolean | true | 是否启用 schema 缓存 |
| `CACHE_SCHEMA_FREEZE_ENABLED` | boolean | - | 是否启用 schema 冻结（防止意外修改） |
| `CACHE_SCHEMA_MAX_ITERATIONS` | number | - | 锁计数器的最大值，防止溢出 |
| `CACHE_SCHEMA_SYNC_TIMEOUT` | number | - | 等待其他进程构建 schema 的超时时间（毫秒） |
| `CACHE_STORE` | string | 'memory' | 缓存存储类型：'memory' 或 'redis'（**不影响 localSchemaCache**） |
| `CACHE_NAMESPACE` | string | - | 缓存键的命名空间 |
| `REDIS_LOCK_NAMESPACE` | string | 'directus:lock' | Redis 锁的命名空间 |
| `REDIS_BUS_NAMESPACE` | string | 'directus:bus' | Redis 消息总线的命名空间 |
| `DB_EXCLUDE_TABLES` | array | - | 排除的表名列表，不会被 Directus 管理 |

### 7.1 关于 CACHE_STORE 的重要说明

| 组件 | 是否遵循 CACHE_STORE | 存储后端 |
|------|----------------------|----------|
| `memorySchemaCache` | ❌ 否 | **硬编码：进程内存** |
| `localSchemaCache` | ❌ 否 | **硬编码：Keyv 内存** |
| `systemCache` | ✅ 是 | 遵循 `CACHE_STORE` |
| `deploymentCache` | ✅ 是 | 遵循 `CACHE_STORE` |
| `lockCache` (useLock) | ✅ 是 | 遵循 `CACHE_STORE` |
| `useBus()` 消息总线 | ⚠️ 部分 | 遵循 Redis 配置（通过 `redisConfigAvailable()` 检测） |

## 8. 关键纠偏总结

### 8.1 第一处纠偏：localSchemaCache 的后端能力

| 之前的描述 | 纠正后的事实 |
|-----------|-------------|
| `localSchemaCache` 支持内存或 Redis | `localSchemaCache` **硬编码使用 `'memory'`**，源码第 79 行：`getKeyvInstance('memory', ...)` |
| 与其他缓存一样遵循 `CACHE_STORE` | 其他缓存（`systemCache`、`deploymentCache`、`lockCache`）都遵循 `CACHE_STORE`，但 `localSchemaCache` **不遵循** |
| 用于跨实例共享 | `localSchemaCache` 是**进程内缓存**，不用于跨实例共享。跨实例同步通过 `schemaChanged` 消息总线实现 |

### 8.2 第二处纠偏：刷新链路的触发、失效与重建

#### 8.2.1 触发条件

`clearSystemCache()` 被以下场景触发：
- **Schema 变更**：Collections、Fields、Relations 的 CUD 操作
- **权限变更**：Permissions、Access、Roles、Users、Policies 的变更
- **手动触发**：`/utils/cache` 端点、`cache:clear` CLI 命令
- **其他**：GraphQL 系统变更、AI 工具操作

#### 8.2.2 失效机制

`clearSystemCache()` 执行以下步骤：
1. 清除 `systemCache`（带锁保护）
2. 清除 `localSchemaCache`（当前实例的 Keyv 内存缓存）
3. 清除 `memorySchemaCache`（当前实例的进程内存缓存）
4. 清除权限缓存（依赖 schema）
5. **发布 `schemaChanged` 消息**（跨实例同步的关键）

#### 8.2.3 跨实例同步的正确理解

| 之前的可能误解 | 纠正后的事实 |
|---------------|-------------|
| 缓存是跨实例共享的 | **每个实例有自己独立的缓存** |
| 同步的是缓存数据 | **同步的是"失效信号"**，不是数据 |
| 一个实例重建，其他实例直接用 | 每个实例**各自清除、各自重建** |

**完整同步流程**：
1. 实例 A 变更 → 清除自己的缓存 → 发布 `schemaChanged` 消息
2. 实例 B、C、D 收到消息 → **各自**清除自己的缓存
3. 下次请求时，每个实例**各自**通过 `getSchema()` 重建

#### 8.2.4 两种消息总线的区别

| 消息键 | 发布者 | 订阅者 | 用途 | 范围 |
|--------|--------|--------|------|------|
| `schemaCache--done` | `getSchema()` 中的构建者 | `getSchema()` 中的等待者 | 并发请求协调：一个构建，其他等待 | **同实例/进程内** |
| `schemaChanged` | `clearSystemCache()` | 所有实例的订阅回调 | 缓存失效通知：一个变更，所有实例清除 | **多实例（Redis）** |

#### 8.2.5 失效与重建的衔接

```
失效阶段：clearSystemCache()
    │
    ├─→ 只清除，不重建
    └─→ 发布失效信号（schemaChanged）

等待阶段：缓存已失效，但没有重建
    │
    └─→ 懒加载：重建发生在下一次请求

重建阶段：下一次请求到达
    │
    ├─→ schema 中间件调用 getSchema()
    ├─→ getMemorySchemaCache() 返回 undefined
    ├─→ 竞争锁 → 一个构建，其他等待
    ├─→ getDatabaseSchema() 实际重建
    └─→ 设置缓存，返回 schema
```

## 9. 代码位置索引

| 功能 | 文件位置 |
|------|---------|
| Schema 获取入口 | [api/src/utils/get-schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/get-schema.ts) |
| Schema 中间件 | [api/src/middleware/schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/middleware/schema.ts) |
| 缓存管理（含 localSchemaCache 硬编码） | [api/src/cache.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts) |
| 分布式锁 | [api/src/lock/lib/use-lock.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/lock/lib/use-lock.ts) |
| 消息总线 | [api/src/bus/lib/use-bus.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/bus/lib/use-bus.ts) |
| 关系服务 | [api/src/services/relations.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/relations.ts) |
| Schema 标准化（用于快照/比较） | [api/src/utils/sanitize-schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/sanitize-schema.ts) |
| SchemaInspector 工厂 | [packages/schema/src/index.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/index.ts) |
| MySQL 方言 | [packages/schema/src/dialects/mysql.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/mysql.ts) |
| PostgreSQL 方言 | [packages/schema/src/dialects/postgres.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts) |

## 10. 关键发现总结

### 10.1 localSchemaCache 的硬编码问题

**源码位置**：`api/src/cache.ts` 第 78-81 行

```typescript
if (localSchemaCache === null) {
  localSchemaCache = getKeyvInstance('memory', getMilliseconds(env['CACHE_SYSTEM_TTL']), '_schema');
  // 第一个参数是硬编码的 'memory'！
}
```

对比其他缓存（都遵循 `CACHE_STORE`）：
- `systemCache`: `getKeyvInstance(env['CACHE_STORE'] as Store, ...)`
- `deploymentCache`: `getKeyvInstance(env['CACHE_STORE'] as Store, ...)`
- `lockCache`: `getKeyvInstance(env['CACHE_STORE'] as Store, ...)`

**结论**：`localSchemaCache` 是设计为进程内缓存，不支持 Redis 共享。

### 10.2 跨实例同步的正确模式

Directus 采用的是**"失效信号广播 + 各自重建"**的模式：

1. **不共享缓存数据**：每个实例有自己独立的 `memorySchemaCache` 和 `localSchemaCache`
2. **只同步失效信号**：通过 `schemaChanged` 消息通知所有实例"该清除缓存了"
3. **懒加载重建**：每个实例在下次请求时各自重建自己的缓存

**优点**：
- 简单可靠：不需要复杂的缓存同步机制
- 一致性：每个实例都从数据库获取最新数据
- 可扩展性：不依赖共享缓存的性能

**缺点**：
- 缓存失效后，第一次请求会变慢（需要重建）
- 高并发场景下，可能出现多个实例同时重建（但通过 `useLock()` 缓解）

### 10.3 两种消息总线的分工

| 场景 | 使用的消息 | 作用 |
|------|-----------|------|
| 多个请求同时到达，缓存已失效 | `schemaCache--done` | 协调并发请求：一个构建，其他等待结果 |
| 一个实例变更了 schema | `schemaChanged` | 通知所有实例清除缓存，准备重建 |

这两种消息配合使用，实现了完整的并发控制和跨实例同步。
