# Directus Schema 推断机制深度分析（纠偏版）

## 1. 概述

本报告针对 Directus Schema 推断机制进行了深入的源码级分析，重点纠正了两处关键描述偏差：

1. **本地 Schema 缓存后端能力**：`localSchemaCache` 硬编码使用 `memory` 存储，不遵循 `CACHE_STORE` 环境变量
2. **Schema 刷新链路**：缓存不是跨实例共享的，跨实例同步的是**失效信号**而非数据本身，重建发生在下次请求时

## 2. 缓存机制的纠偏分析

### 2.1 关键发现：localSchemaCache 硬编码为 Memory

**源码证据**（`api/src/cache.ts` 第 78-81 行）：

```typescript
if (localSchemaCache === null) {
  localSchemaCache = getKeyvInstance('memory', getMilliseconds(env['CACHE_SYSTEM_TTL']), '_schema');
  // 第一个参数是硬编码的 'memory'！
}
```
[api/src/cache.ts:78-81](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

**对比其他缓存的初始化**：

| 缓存 | 初始化代码 | 遵循 `CACHE_STORE` |
|------|-----------|---------------------|
| `systemCache` | `getKeyvInstance(env['CACHE_STORE'] as Store, ...)` | ✅ 是 |
| `deploymentCache` | `getKeyvInstance(env['CACHE_STORE'] as Store, ...)` | ✅ 是 |
| `lockCache` | `getKeyvInstance(env['CACHE_STORE'] as Store, ...)` | ✅ 是 |
| **`localSchemaCache`** | `getKeyvInstance('memory', ...)` | ❌ **硬编码 `memory`** |

### 2.2 各缓存的真实职责澄清

#### 2.2.1 memorySchemaCache（进程内内存缓存）

**定义**：
```typescript
let memorySchemaCache: Readonly<SchemaOverview> | null = null;
```
[api/src/cache.ts:27](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

**特点**：
- 最简单的进程内变量
- 最快的缓存层（无需序列化/反序列化）
- **每个实例独立拥有**，不跨实例共享

**访问方式**：
```typescript
// 设置
export function setMemorySchemaCache(schema: SchemaOverview) {
  if (Object.isFrozen(schema)) {
    memorySchemaCache = schema;
  } else {
    memorySchemaCache = freezeSchema(schema);
  }
}

// 获取
export function getMemorySchemaCache(): Readonly<SchemaOverview> | undefined {
  if (env['CACHE_SCHEMA_FREEZE_ENABLED']) {
    return memorySchemaCache ?? undefined;
  } else if (memorySchemaCache) {
    return unfreezeSchema(memorySchemaCache);
  }
  return undefined;
}
```
[api/src/cache.ts:133-149](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

#### 2.2.2 localSchemaCache（Keyv 封装，但硬编码 Memory）

**定义**：
```typescript
let localSchemaCache: Keyv | null = null;
```
[api/src/cache.ts:26](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

**初始化**（硬编码 `'memory'`）：
```typescript
if (localSchemaCache === null) {
  localSchemaCache = getKeyvInstance('memory', getMilliseconds(env['CACHE_SYSTEM_TTL']), '_schema');
  localSchemaCache.on('error', (err) => logger.warn(err, `[schema-cache] ${err}`));
}
```
[api/src/cache.ts:78-81](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

**真实用途**：
- 不用于 `getSchema()` 的主缓存路径
- 主要用于 `RelationsService.foreignKeys()` 缓存外键信息

```typescript
// 在 RelationsService 中的使用
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

#### 2.2.3 缓存访问路径澄清

**`getSchema()` 的缓存检查路径**：

```typescript
export async function getSchema(...) {
  // 步骤 1: 检查是否绕过缓存
  if (options?.bypassCache || env['CACHE_SCHEMA'] === false) {
    // 直接构建，不使用缓存
    return await getDatabaseSchema(database, schemaInspector);
  }

  // 步骤 2: 只检查 memorySchemaCache！
  const cached = getMemorySchemaCache();
  if (cached) {
    return cached;
  }

  // 步骤 3: 未命中，进入并发构建流程
  // ...
}
```
[api/src/utils/get-schema.ts:38-49](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/get-schema.ts)

**关键结论**：
- `getSchema()` **只检查 `memorySchemaCache`**，不检查 `localSchemaCache`
- `localSchemaCache` 只用于缓存**外键信息**等中间数据
- 两个缓存都是**实例本地**的，不跨实例共享

### 2.3 真正跨实例的组件

虽然 `memorySchemaCache` 和 `localSchemaCache` 都是实例本地的，但以下组件支持跨实例：

| 组件 | 单实例场景 | 多实例 + Redis 场景 |
|------|-----------|---------------------|
| `useLock()` | 本地内存锁 | Redis 分布式锁 |
| `useBus()` | 本地事件总线 | Redis Pub/Sub |
| `lockCache` | 本地 Keyv | Redis Keyv |
| `systemCache` | 本地 Keyv | Redis Keyv |
| `memorySchemaCache` | 本地变量 | **仍为本地变量** |
| `localSchemaCache` | 本地 Keyv | **仍为本地 Keyv（硬编码 memory）** |

## 3. Schema 刷新链路的纠偏分析

### 3.1 触发条件：何时会清除缓存？

`clearSystemCache()` 是清除 schema 缓存的核心函数，它在以下场景被调用：

#### 3.1.1 Schema 相关操作

| 服务 | 操作 | 调用位置 |
|------|------|---------|
| `CollectionsService` | `createOne`, `updateOne`, `deleteOne`, `updateField`, `deleteField` 等 | `finally` 块 |
| `FieldsService` | `createOne`, `updateOne`, `deleteOne`, `createField`, `updateField`, `deleteField` 等 | `finally` 块 |
| `RelationsService` | `createOne`, `updateOne`, `deleteOne` | `finally` 块 |

#### 3.1.2 权限相关操作

| 服务 | 操作 | 调用位置 |
|------|------|---------|
| `PermissionsService` | 权限变更 | `finally` 块 |
| `RolesService` | 角色变更 | `finally` 块 |
| `PoliciesService` | 策略变更 | `finally` 块 |
| `AccessService` | 访问控制变更 | `finally` 块 |
| `UsersService` | 用户变更（影响权限） | `finally` 块 |

#### 3.1.3 手动/程序化触发

| 触发方式 | 位置 |
|---------|------|
| `flushCaches()` 函数 | `cache.ts` |
| CLI 命令 `cache:clear` | `cli/commands/cache/clear.ts` |
| API 端点 `POST /utils/cache/clear` | `controllers/utils.ts` |
| AI 工具字段操作 | `ai/tools/fields/index.ts` |

#### 3.1.4 调用模式

所有调用都遵循以下模式：

```typescript
try {
  // 执行实际操作（创建/更新/删除）
  await doTheActualWork();
} finally {
  // 默认清除缓存，除非显式传入 autoPurgeSystemCache: false
  if (opts?.autoPurgeSystemCache !== false) {
    await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });
  }
}
```

**关键点**：
- 在 `finally` 块中调用，确保**无论操作成功还是失败**都会执行
- `autoPurgeSystemCache` 默认值为 `true`（检查 `!== false`）

### 3.2 失效机制：缓存如何被清除？

#### 3.2.1 clearSystemCache() 执行流程

```typescript
export async function clearSystemCache(opts?: {
  forced?: boolean | undefined;
  autoPurgeCache?: false | undefined;
}): Promise<void> {
  const { systemCache, localSchemaCache, lockCache } = getCache();

  // 步骤 1: 清除 systemCache（带锁保护，防止并发清除）
  if (opts?.forced || !(await lockCache.get('system-cache-lock'))) {
    await lockCache.set('system-cache-lock', true, 10000);  // 加锁 10 秒
    await systemCache.clear();
    await lockCache.delete('system-cache-lock');
  }

  // 步骤 2: 清除 schema 相关缓存（关键！无锁保护，总是执行）
  await localSchemaCache.clear();  // 清除 Keyv 缓存（外键等）
  memorySchemaCache = null;         // 清除进程内内存缓存！

  // 步骤 3: 清除权限缓存（依赖 schema）
  await clearPermissionCache();

  // 步骤 4: 发布失效信号到消息总线
  messenger.publish<CacheMessage>('schemaChanged', { autoPurgeCache: opts?.autoPurgeCache });
}
```
[api/src/cache.ts:97-117](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

#### 3.2.2 跨实例同步的本质

**❌ 错误理解**：
- 缓存是跨实例共享的
- 一个实例更新缓存，其他实例自动看到新数据

**✅ 正确理解**：

1. **各实例缓存独立**：
   - `memorySchemaCache` 是**进程内变量**，每个实例有自己的一份
   - `localSchemaCache` 硬编码为 `memory` 存储，也是实例本地的
   - 缓存**从不跨实例共享**

2. **同步的是"失效信号"，不是数据**：
   - `schemaChanged` 消息只是一个通知："你的缓存可能过时了"
   - 各实例收到信号后，**自己清除自己的缓存**

**订阅逻辑源码**（`cache.ts` 第 41-52 行）：

```typescript
if (redisConfigAvailable() && !messengerSubscribed) {
  messengerSubscribed = true;

  messenger.subscribe<CacheMessage>('schemaChanged', async (opts) => {
    // 如果使用内存缓存且配置了自动清除，清除 API 响应缓存
    if (env['CACHE_STORE'] === 'memory' && env['CACHE_AUTO_PURGE'] && cache && opts?.['autoPurgeCache'] !== false) {
      await cache.clear();
    }

    // 关键：每个实例独立清除自己的 schema 缓存
    await localSchemaCache?.clear();  // 清除 Keyv 缓存
    memorySchemaCache = null;           // 清除内存缓存！
  });
}
```
[api/src/cache.ts:41-52](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts)

### 3.3 重建衔接：失效后如何重建？

#### 3.3.1 重建的触发时机

**重建不是立即发生的**，而是发生在**下次请求**时：

```
┌─────────────────────────────────────────────────────────────────────┐
│  时间线                                                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  T0: 实例 A 执行字段更新操作                                         │
│      └─→ 调用 clearSystemCache()                                    │
│      └─→ 实例 A: memorySchemaCache = null                          │
│      └─→ 发布 'schemaChanged' 消息                                  │
│                                                                      │
│  T1: 实例 B、C、D 收到消息                                           │
│      └─→ 各自执行: memorySchemaCache = null                         │
│      └─→ 此时所有实例的缓存都已失效，但还没有重建                    │
│                                                                      │
│  T2: 用户发起 HTTP 请求到实例 B                                      │
│      └─→ 触发 schema 中间件                                         │
│      └─→ 调用 getSchema()                                            │
│      └─→ getMemorySchemaCache() 返回 null                           │
│      └─→ 触发实际的重建流程！                                        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3.3.2 完整的失效→重建链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 1: 触发清除（实例 A 执行变更操作）                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  实例 A:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. 执行字段/集合/关系变更操作                                      │    │
│  │    (FieldsService.createOne 等)                                  │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                                    ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 2. finally 块中调用 clearSystemCache()                            │    │
│  │                                                                    │    │
│  │    2.1 localSchemaCache.clear()  ──→ 清除 Keyv 缓存              │    │
│  │    2.2 memorySchemaCache = null   ──→ 清除内存缓存！              │    │
│  │    2.3 clearPermissionCache()     ──→ 清除权限缓存                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                                    ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 3. 发布 'schemaChanged' 消息到消息总线                            │    │
│  │                                                                    │    │
│  │    messenger.publish('schemaChanged', { autoPurgeCache })        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 2: 失效信号传播（多实例 + Redis 场景）                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Redis Pub/Sub:                                                          │
│                                                                           │
│  ┌──────────┐                                                            │
│  │ 实例 A  │──── publish('schemaChanged') ────┐                        │
│  └──────────┘                                  │                        │
│                                                ▼                        │
│                                        ┌───────────┐                    │
│                                        │   Redis   │                    │
│                                        │  Pub/Sub  │                    │
│                                        └───────────┘                    │
│                                                │                        │
│                    ┌──────────────┬───────────┼───────────┐           │
│                    ▼              ▼           ▼           ▼           │
│              ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│              │ 实例 B   │  │ 实例 C   │  │ 实例 D   │  │ 实例 ...  │  │
│              └──────────┘  └──────────┘  └──────────┘  └──────────┘  │
│                                                                           │
│  各实例收到消息后的处理：                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ // 每个实例独立执行！                                              │    │
│  │ await localSchemaCache?.clear();  // 清除自己的 Keyv 缓存        │    │
│  │ memorySchemaCache = null;          // 清除自己的内存缓存！        │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
│  此时状态：所有实例的缓存都已失效，但还没有重建                            │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  阶段 3: 下次请求时重建（实例 B 收到请求）                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. HTTP 请求到达实例 B                                            │    │
│  │    GET /items/users                                               │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                                    ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 2. schema 中间件执行                                              │    │
│  │                                                                    │    │
│  │    const schema: RequestHandler = asyncHandler(async (req, ...) │    │
│  │      req.schema = await getSchema();  // 关键调用                │    │
│  │      return next();                                               │    │
│  │    });                                                            │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                                    ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 3. getSchema() 执行                                               │    │
│  │                                                                    │    │
│  │    3.1 检查 bypassCache 或 CACHE_SCHEMA=false                    │    │
│  │    3.2 检查 memorySchemaCache                                     │    │
│  │        └─→ 返回 null（已被清除）                                  │    │
│  │    3.3 检查重试次数（防止无限循环）                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                                    ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 4. 并发控制流程（多实例场景下使用 Redis 锁）                       │    │
│  │                                                                    │    │
│  │    4.1 获取锁: const lock = useLock();                            │    │
│  │        └─→ 多实例场景: Redis 锁                                   │    │
│  │        └─→ 单实例场景: 本地锁                                     │    │
│  │                                                                    │    │
│  │    4.2 原子递增获取 processId:                                     │    │
│  │        const processId = await lock.increment('schemaCache--preparing'); │
│  │                                                                    │    │
│  │    4.3 判断是否为处理者:                                          │    │
│  │        const isHandler = processId === 1;                         │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                    ┌───────────────┴───────────────┐                    │
│                    ▼                               ▼                    │
│        ┌─────────────────────┐       ┌─────────────────────┐           │
│        │ processId === 1     │       │ processId !== 1     │           │
│        │ (我是处理者)         │       │ (我是等待者)         │           │
│        └──────────┬──────────┘       └──────────┬──────────┘           │
│                   │                               │                       │
│                   ▼                               ▼                       │
│        ┌─────────────────────┐       ┌─────────────────────┐           │
│        │ 实际构建            │       │ 订阅消息等待        │           │
│        │                     │       │                     │           │
│        │ 1. getDatabaseSchema│       │ 1. 订阅 'schemaCache│           │
│        │    (核心构建函数)    │       │    --done' 消息     │           │
│        │                     │       │                     │           │
│        │ 2. setMemorySchema  │       │ 2. 超时保护         │           │
│        │    Cache(schema)    │       │    (CACHE_SCHEMA_   │           │
│        │                     │       │    SYNC_TIMEOUT)     │           │
│        │ 3. 发布 'done' 消息 │       │                     │           │
│        │ 4. 释放锁           │       │ 3. 收到消息后:       │           │
│        │                     │       │    setMemorySchema   │           │
│        │ 返回 schema         │       │    Cache()           │           │
│        │                     │       │    返回 schema        │           │
│        └─────────────────────┘       └─────────────────────┘           │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 3.3.3 多实例场景下的锁竞争

**关键点**：多实例场景下，`useLock()` 返回 Redis 锁，确保**只有一个实例实际执行构建**。

```typescript
// useLock 的实现
export const useLock = () => {
  if (_cache.lock) {
    return _cache.lock;
  }

  if (redisConfigAvailable()) {
    // 多实例场景：使用 Redis 分布式锁
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

**锁竞争流程**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│  多实例场景（实例 B、C、D 同时收到请求）                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  锁键: 'schemaCache--preparing'                                          │
│                                                                           │
│  时间 T0:                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                          │
│  │ 实例 B   │    │ 实例 C   │    │ 实例 D   │                          │
│  │ lock.incre│    │ lock.incre│    │ lock.incre│                          │
│  │ ment()   │    │ ment()   │    │ ment()   │                          │
│  └────┬─────┘    └────┬─────┘    └────┬─────┘                          │
│       │               │               │                                 │
│       ▼               ▼               ▼                                 │
│  ┌─────────────────────────────────────────────┐                       │
│  │ Redis 锁原子递增:                             │                       │
│  │ - 实例 B 先到达: processId = 1               │                       │
│  │ - 实例 C 后到达: processId = 2               │                       │
│  │ - 实例 D 最后到: processId = 3               │                       │
│  └─────────────────────────────────────────────┘                       │
│                                                                           │
│  时间 T1:                                                                 │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐ │
│  │ 实例 B           │    │ 实例 C           │    │ 实例 D           │ │
│  │ processId === 1  │    │ processId !== 1  │    │ processId !== 1  │ │
│  │                  │    │                  │    │                  │ │
│  │ 我是处理者！     │    │ 我是等待者...    │    │ 我是等待者...    │ │
│  │                  │    │                  │    │                  │ │
│  │ 执行构建:        │    │ 订阅消息:        │    │ 订阅消息:        │ │
│  │ getDatabaseSchema│    │ 'schemaCache--   │    │ 'schemaCache--   │ │
│  │ ()               │    │ done'            │    │ done'            │ │
│  └────────┬─────────┘    └────────┬─────────┘    └────────┬─────────┘ │
│           │                        │                        │           │
│           ▼                        │                        │           │
│  ┌──────────────────┐              │                        │           │
│  │ 构建完成          │              │                        │           │
│  │                  │              │                        │           │
│  │ 1. setMemorySchema│              │                        │           │
│  │    Cache()       │              │                        │           │
│  │ 2. 发布 'done'   │◄─────────────┼────────────────────────┘           │
│  │ 3. 释放锁         │              │                                    │
│  └────────┬─────────┘              │                                    │
│           │                         │                                    │
│           ▼                         ▼                                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │ 实例 C、D 收到 'done' 消息                                        │   │
│  │                                                                    │   │
│  │ 1. setMemorySchemaCache(options.schema)  ──→ 设置自己的缓存       │   │
│  │ 2. 返回 schema                                                │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  结果：只有实例 B 实际执行了构建，实例 C、D 直接使用构建结果               │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## 4. 关键纠偏总结

### 4.1 第一处纠偏：localSchemaCache 的后端能力

| 项目 | 之前错误描述 | 正确描述（按源码） |
|------|-------------|-------------------|
| `localSchemaCache` 存储 | "支持内存或 Redis" | **硬编码 `'memory'` 存储**，不遵循 `CACHE_STORE` |
| 初始化代码 | `getKeyvInstance(env['CACHE_STORE'], ...)` | `getKeyvInstance('memory', ...)` |
| 主要用途 | 主 schema 缓存 | **只用于缓存外键信息**（`RelationsService.foreignKeys()`） |
| `getSchema()` 检查 | 可能暗示检查 | **不检查**，只检查 `memorySchemaCache` |

### 4.2 第二处纠偏：Schema 刷新链路

| 项目 | 之前可能的误解 | 正确描述（按源码） |
|------|---------------|-------------------|
| 缓存是否跨实例共享 | 可能认为是共享的 | **各实例独立拥有**，从不共享 |
| 跨实例同步的内容 | 同步新的 schema 数据 | **同步"失效信号"**，不是数据 |
| 同步后是否立即重建 | 可能认为立即重建 | **不立即重建**，重建发生在**下次请求**时 |
| 多实例重建方式 | 可能认为各实例各自重建 | **使用 Redis 锁确保只有一个实例重建**，其他实例等待结果 |
| 触发清除的位置 | 可能认为只在成功时清除 | **在 `finally` 块中清除**，无论操作成功或失败 |

### 4.3 完整的刷新链路总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    完整的 Schema 刷新链路（多实例场景）                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  触发条件:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ - 集合/字段/关系变更（CollectionsService, FieldsService,         │    │
│  │   RelationsService）                                              │    │
│  │ - 权限/角色/策略变更                                              │    │
│  │ - 手动触发（CLI、API、程序化调用）                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                                    ▼                                      │
│  失效机制:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. 执行实例（实例 A）调用 clearSystemCache()                      │    │
│  │    ├─→ localSchemaCache.clear()                                  │    │
│  │    ├─→ memorySchemaCache = null（实例 A 自己的）                 │    │
│  │    └─→ 发布 'schemaChanged' 消息到 Redis Pub/Sub                 │    │
│  │                                                                    │    │
│  │ 2. 其他实例（B、C、D）收到消息                                     │    │
│  │    ├─→ 各自执行 localSchemaCache?.clear()                        │    │
│  │    └─→ 各自执行 memorySchemaCache = null                          │    │
│  │                                                                    │    │
│  │ 结果：所有实例的缓存都已失效，但还没有重建                          │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│                                    ▼                                      │
│  重建衔接:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 1. 下次 HTTP 请求到达某个实例（如实例 B）                          │    │
│  │                                                                    │    │
│  │ 2. schema 中间件调用 getSchema()                                  │    │
│  │    ├─→ getMemorySchemaCache() 返回 null                          │    │
│  │    └─→ 进入并发构建流程                                            │    │
│  │                                                                    │    │
│  │ 3. 并发控制（使用 Redis 锁）                                       │    │
│  │    ├─→ 只有 processId === 1 的实例是处理者                        │    │
│  │    ├─→ 处理者：调用 getDatabaseSchema() 实际构建                   │    │
│  │    ├─→ 处理者：发布 'schemaCache--done' 消息                      │    │
│  │    └─→ 等待者：订阅消息，收到后直接使用结果                        │    │
│  │                                                                    │    │
│  │ 结果：只有一个实例实际执行构建，其他实例等待结果                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

## 5. 代码位置索引

| 功能 | 文件位置 | 关键行 |
|------|---------|--------|
| `localSchemaCache` 硬编码初始化 | [api/src/cache.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts) | 78-81 |
| `memorySchemaCache` 定义 | [api/src/cache.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts) | 27 |
| `getMemorySchemaCache()` 函数 | [api/src/cache.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts) | 141-149 |
| `clearSystemCache()` 函数 | [api/src/cache.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts) | 97-117 |
| `schemaChanged` 消息订阅 | [api/src/cache.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/cache.ts) | 41-52 |
| `getSchema()` 函数 | [api/src/utils/get-schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/get-schema.ts) | 22-114 |
| `useLock()` 函数 | [api/src/lock/lib/use-lock.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/lock/lib/use-lock.ts) | 12-30 |
| `RelationsService.foreignKeys()` | [api/src/services/relations.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/relations.ts) | 65-87 |
