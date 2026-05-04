# Directus Schema Introspection 机制分析

## 概述

Directus 通过一套完整的 Schema Introspection 机制来读取不同数据库的结构信息。这套机制采用了**抽象接口 + 方言实现**的设计模式，在保持统一接口的同时，处理不同数据库之间的差异。

## 核心架构

### 1. 架构层次

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           请求入口层 (Request Entry)                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Express Middleware: schema.ts                                       │   │
│  │  - 每个请求自动加载 schema 到 req.schema                              │   │
│  │  - 调用 getSchema() 获取/缓存结构信息                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      缓存与并发控制层 (Cache & Concurrency)                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  getSchema() - api/src/utils/get-schema.ts                          │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 1. 缓存检查: getMemorySchemaCache()                         │    │   │
│  │  │    - 检查内存缓存 (freezeSchema / unfreezeSchema)          │    │   │
│  │  │    - 支持 CACHE_SCHEMA 环境变量控制                         │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                         │   │
│  │                              ▼ (缓存未命中)                            │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 2. 分布式锁: useLock()                                       │    │   │
│  │  │    - lockKey = 'schemaCache--preparing'                     │    │   │
│  │  │    - 防止多进程/多实例重复构建                                │    │   │
│  │  │    - processId === 1 时才真正执行构建                       │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                         │   │
│  │                              ▼ (非首个进程)                            │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 3. 进程间通信: useBus()                                      │    │   │
│  │  │    - messageKey = 'schemaCache--done'                       │    │   │
│  │  │    - 非首个进程订阅消息，等待首个进程完成                     │    │   │
│  │  │    - 超时机制: CACHE_SCHEMA_SYNC_TIMEOUT                    │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         方言分派层 (Dialect Dispatch)                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  createInspector() - packages/schema/src/index.ts                   │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 根据 knex.client.constructor.name 选择方言:                  │    │   │
│  │  │  - Client_PG          → PostgresSchemaInspector             │    │   │
│  │  │  - Client_MySQL/2     → MySQLSchemaInspector                │    │   │
│  │  │  - Client_SQLite3     → SqliteSchemaInspector               │    │   │
│  │  │  - Client_MSSQL       → MSSQLSchemaInspector                │    │   │
│  │  │  - Client_CockroachDB → CockroachDBSchemaInspector         │    │   │
│  │  │  - Client_Oracledb    → OracleDBSchemaInspector             │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          结构回填层 (Schema Hydration)                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  getDatabaseSchema() - api/src/utils/get-schema.ts                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 1. 原生结构读取                                               │    │   │
│  │  │    - inspector.overview() → 获取表/列/主键的原始结构        │    │   │
│  │  │    - 返回 SchemaOverview (方言层输出格式)                    │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                         │   │
│  │                              ▼                                         │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 2. Directus 元数据合并                                       │    │   │
│  │  │    - directus_collections → 集合配置 (singleton, note 等)  │    │   │
│  │  │    - directus_fields → 字段配置 (special, validation 等)   │    │   │
│  │  │    - systemCollectionRows → 系统集合默认配置                 │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                         │   │
│  │                              ▼                                         │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 3. 类型归一化                                                │    │   │
│  │  │    - getLocalType() → 将数据库类型映射为 Directus 类型      │    │   │
│  │  │    - getDefaultValue() → 解析并转换默认值格式               │    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  │                              │                                         │   │
│  │                              ▼                                         │   │
│  │  ┌─────────────────────────────────────────────────────────────┐    │   │
│  │  │ 4. 关系信息补充                                              │    │   │
│  │  │    - RelationsService.readAll() → 从 directus_relations 读取│    │   │
│  │  └─────────────────────────────────────────────────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. 统一层与方言层的边界定义

| 层级 | 职责范围 | 输入 | 输出 | 关键文件 |
|------|---------|------|------|---------|
| **方言层** | 数据库特有元数据查询 | Knex 连接 | `SchemaOverview` (原生结构) | `packages/schema/src/dialects/*.ts` |
| **统一层** | 类型归一、元数据合并、缓存管理 | `SchemaOverview` + Directus 元数据 | `SchemaOverview` (完整 Directus schema) | `api/src/utils/get-schema.ts`, `api/src/utils/get-local-type.ts` |

#### 方言层的职责边界

方言层**只负责**：
1. 执行数据库特有的 SQL 查询获取元数据
2. 将查询结果转换为统一的 `Column`/`Table`/`ForeignKey` 接口格式
3. 处理数据库特有的格式差异（如 PostgreSQL 的 `::type` 后缀）

方言层**不负责**：
1. 类型归一化（不将 `varchar` 映射为 `string`）
2. 与 Directus 元数据合并
3. 缓存管理
4. 并发控制

#### 统一层的职责边界

统一层**负责**：
1. 缓存读取与写入
2. 分布式锁与进程同步
3. 从 `directus_collections`/`directus_fields` 读取元数据
4. 类型归一化 (`getLocalType()`)
5. 默认值解析 (`getDefaultValue()`)
6. 关系信息补充

### 3. 关键文件位置

| 组件 | 文件路径 | 说明 |
|------|---------|------|
| 请求入口 | `api/src/middleware/schema.ts` | Express 中间件，自动加载 schema |
| 缓存与并发 | `api/src/utils/get-schema.ts` | `getSchema()` 核心逻辑 |
| 方言工厂 | `packages/schema/src/index.ts` | `createInspector()` |
| 类型归一 | `api/src/utils/get-local-type.ts` | `getLocalType()` |
| 默认值处理 | `api/src/utils/get-default-value.ts` | `getDefaultValue()` |
| 核心接口 | `packages/schema/src/types/schema-inspector.ts` | SchemaInspector 接口 |

---

## 完整调用链详解

### 阶段 1: 请求入口

**文件**: `api/src/middleware/schema.ts:1-10`

每个 HTTP 请求到达时，Express 中间件自动触发 schema 加载：

```typescript
const schema: RequestHandler = asyncHandler(async (req, _res, next) => {
    req.schema = await getSchema();  // 核心入口
    return next();
});
```

**调用时机**:
- 每个 API 请求都会执行
- `req.schema` 在后续的 Service 层和 Controller 层中使用

### 阶段 2: 缓存与并发控制

**文件**: `api/src/utils/get-schema.ts:22-114`

这是最复杂的一层，处理多进程/多实例场景下的缓存与并发：

```
┌─────────────────────────────────────────────────────────────────┐
│                    getSchema() 执行流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐                                               │
│  │ 1. 检查配置   │                                               │
│  │ bypassCache?  │ ──Yes──→ 跳过缓存，直接查询数据库            │
│  │ CACHE_SCHEMA? │                                               │
│  └──────┬───────┘                                               │
│         │ No                                                    │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │ 2. 内存缓存   │                                               │
│  │              │ ──命中──→ 直接返回缓存数据                    │
│  │ getMemory    │                                               │
│  │ SchemaCache  │                                               │
│  └──────┬───────┘                                               │
│         │ Miss                                                  │
│         ▼                                                       │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 3. 分布式锁竞争 (useLock)                                  │ │
│  │                                                           │ │
│  │   lockKey = 'schemaCache--preparing'                     │ │
│  │   processId = lock.increment(lockKey)                    │ │
│  │                                                           │ │
│  │   ┌─────────────────┐      ┌─────────────────────────┐  │ │
│  │   │ processId === 1 │      │ processId >= 2          │  │ │
│  │   │ (首个进程)       │      │ (等待进程)               │  │ │
│  │   └────────┬────────┘      └───────────┬─────────────┘  │ │
│  │            │                            │                  │ │
│  │            ▼                            ▼                  │ │
│  │   ┌────────────────┐         ┌───────────────────────┐   │ │
│  │   │ 实际执行构建   │         │ 订阅消息等待完成      │   │ │
│  │   │                │         │                       │   │ │
│  │   │ getDatabase    │         │ bus.subscribe(        │   │ │
│  │   │ Schema()       │         │   'schemaCache--done' │   │ │
│  │   │                │         │ )                      │   │ │
│  │   │ setMemory      │         │ 超时: CACHE_SCHEMA_   │   │ │
│  │   │ SchemaCache()  │         │ SYNC_TIMEOUT          │   │ │
│  │   └────────┬───────┘         └───────────┬───────────┘   │ │
│  │            │                            │                  │ │
│  │            └────────────┬───────────────┘                  │ │
│  │                         ▼                                  │ │
│  │            ┌────────────────────────┐                      │ │
│  │            │ 4. 广播完成消息        │                      │ │
│  │            │ bus.publish(           │                      │ │
│  │            │   'schemaCache--done', │                      │ │
│  │            │   { schema }           │                      │ │
│  │            │ )                      │                      │ │
│  │            └────────────┬───────────┘                      │ │
│  │                         ▼                                  │ │
│  │            ┌────────────────────────┐                      │ │
│  │            │ 5. 释放锁              │                      │ │
│  │            │ lock.delete(lockKey)   │                      │ │
│  │            └────────────────────────┘                      │ │
│  └──────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**关键代码片段**:

```typescript
export async function getSchema(options?, attempt = 0): Promise<SchemaOverview> {
    // 1. 缓存检查
    if (options?.bypassCache || env['CACHE_SCHEMA'] === false) {
        // 跳过缓存，直接查询
        const schemaInspector = createInspector(database);
        return await getDatabaseSchema(database, schemaInspector);
    }

    const cached = getMemorySchemaCache();
    if (cached) return cached;  // 缓存命中

    // 2. 分布式锁
    const lock = useLock();
    const bus = useBus();
    const lockKey = 'schemaCache--preparing';
    const messageKey = 'schemaCache--done';
    const processId = await lock.increment(lockKey);

    // 3. 非首个进程：等待消息
    if (processId !== 1) {
        const subscription = new Promise((resolve, reject) => {
            bus.subscribe(messageKey, (options) => {
                // 收到首个进程完成的消息
                setMemorySchemaCache(options.schema);
                resolve(options.schema);
            });
        });
        // 超时后重试
        return Promise.race([timeout, subscription])
            .catch(() => getSchema(options, attempt + 1));
    }

    // 4. 首个进程：实际执行构建
    try {
        const schemaInspector = createInspector(database);
        schema = await getDatabaseSchema(database, schemaInspector);
        setMemorySchemaCache(schema);
        return schema;
    } finally {
        // 5. 广播完成消息
        await bus.publish(messageKey, { schema });
        await lock.delete(lockKey);
    }
}
```

### 阶段 3: 方言分派

**文件**: `packages/schema/src/index.ts:16-46`

根据 Knex 客户端类型选择对应的方言实现：

```typescript
export const createInspector = (knex: Knex): SchemaInspector => {
    let constructor: SchemaInspectorConstructor;

    switch (knex.client.constructor.name) {
        case 'Client_MySQL':
        case 'Client_MySQL2':
            constructor = MySQLSchemaInspector;
            break;
        case 'Client_PG':
            constructor = PostgresSchemaInspector;
            break;
        // ... 其他方言
    }

    return new constructor(knex);
};
```

**分派依据**:
- `knex.client.constructor.name` - Knex 客户端的构造函数名称
- 例如 PostgreSQL 使用 `Client_PG`，MySQL 使用 `Client_MySQL`

### 阶段 4: 结构回填

**文件**: `api/src/utils/get-schema.ts:116-232`

这是将数据库原生结构转换为 Directus 可用结构的关键步骤：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    getDatabaseSchema() 执行流程                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Step 1: 读取原生数据库结构                                    │  │
│  │                                                              │  │
│  │   const schemaOverview = await schemaInspector.overview();  │  │
│  │                                                              │  │
│  │   输出格式 (SchemaOverview):                                 │  │
│  │   {                                                          │  │
│  │     "users": {                                               │  │
│  │       primary: "id",                                         │  │
│  │       columns: {                                             │  │
│  │         "id": {                                              │  │
│  │           data_type: "integer",      // 数据库原生类型       │  │
│  │           default_value: "nextval(...)", // 原生格式         │  │
│  │           is_nullable: false,                                │  │
│  │           ...                                                │  │
│  │         }                                                    │  │
│  │       }                                                      │  │
│  │     }                                                        │  │
│  │   }                                                          │  │
│  └───────────────────────┬─────────────────────────────────────┘  │
│                          │                                           │
│                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Step 2: 合并 Directus 元数据                                 │  │
│  │                                                              │  │
│  │   // 从 directus_collections 读取集合配置                    │  │
│  │   const collections = [                                      │  │
│  │     ...await database.select(...).from('directus_collections'),│
│  │     ...systemCollectionRows  // 系统集合默认配置              │  │
│  │   ];                                                         │  │
│  │                                                              │  │
│  │   // 从 directus_fields 读取字段配置                         │  │
│  │   const fields = [                                           │  │
│  │     ...await database.select(...).from('directus_fields'), │  │
│  │     ...systemFieldRows                                       │  │
│  │   ];                                                         │  │
│  └───────────────────────┬─────────────────────────────────────┘  │
│                          │                                           │
│                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Step 3: 类型归一化 (核心转换)                                │  │
│  │                                                              │  │
│  │   遍历每个集合和字段，执行以下转换：                         │  │
│  │                                                              │  │
│  │   fields: mapValues(columns, (column) => {                 │  │
│  │     return {                                                 │  │
│  │       // 数据库原生值                                        │  │
│  │       dbType: column.data_type,        // "character varying"│
│  │       precision: column.numeric_precision, // null          │  │
│  │       scale: column.numeric_scale,      // null             │  │
│  │                                                              │  │
│  │       // 归一化后的值                                        │  │
│  │       type: getLocalType(column),       // "string"         │  │
│  │       defaultValue: getDefaultValue(column), // 解析后的值  │  │
│  │       nullable: column.is_nullable,      // true/false      │  │
│  │       generated: column.is_generated,    // true/false      │  │
│  │       ...                                                    │  │
│  │     };                                                        │  │
│  │   })                                                          │  │
│  └───────────────────────┬─────────────────────────────────────┘  │
│                          │                                           │
│                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Step 4: 补充 Directus 字段配置                               │  │
│  │                                                              │  │
│  │   遍历 directus_fields 中的记录，覆盖或补充字段配置：        │  │
│  │                                                              │  │
│  │   for (const field of fields) {                             │  │
│  │     const special = toArray(field.special); // ["cast-json"]│
│  │                                                              │  │
│  │     // special 影响类型判断                                  │  │
│  │     const type = getLocalType(column, { special });         │  │
│  │                                                              │  │
│  │     result.collections[field.collection]!.fields[field.field] = {│
│  │       special: special,              // ["cast-json"]       │  │
│  │       note: field.note,              // 用户添加的备注       │  │
│  │       validation: field.validation,   // JSON 验证规则       │  │
│  │       alias: !existing?.dbType,       // 是否为虚拟字段      │  │
│  │       ...                                                      │  │
│  │     };                                                          │  │
│  │   }                                                             │  │
│  └───────────────────────┬─────────────────────────────────────┘  │
│                          │                                           │
│                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Step 5: 补充关系信息                                          │  │
│  │                                                              │  │
│  │   const relationsService = new RelationsService({            │  │
│  │     knex: database,                                          │  │
│  │     schema: result                                           │  │
│  │   });                                                         │  │
│  │                                                              │  │
│  │   // 从 directus_relations 表读取所有关系                    │  │
│  │   result.relations = await relationsService.readAll();       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**关键代码片段**:

```typescript
async function getDatabaseSchema(database: Knex, schemaInspector: SchemaInspector): Promise<SchemaOverview> {
    const result: SchemaOverview = {
        collections: {},
        relations: [],
    };

    // Step 1: 读取原生结构
    const schemaOverview = await schemaInspector.overview();

    // Step 2: 读取 Directus 元数据
    const collections = [
        ...(await database.select(...).from('directus_collections')),
        ...systemCollectionRows,
    ];

    const fields = [
        ...(await database.select(...).from('directus_fields')),
        ...systemFieldRows,
    ];

    // Step 3: 遍历集合，构建完整结构
    for (const [collection, info] of Object.entries(schemaOverview)) {
        // 跳过排除的表
        if (toArray(env['DB_EXCLUDE_TABLES']).includes(collection)) continue;

        const collectionMeta = collections.find(c => c.collection === collection);

        result.collections[collection] = {
            collection,
            primary: info.primary,
            singleton: toBoolean(collectionMeta?.singleton),
            note: collectionMeta?.note || null,
            accountability: collectionMeta?.accountability || 'all',
            
            // 类型归一化的核心位置
            fields: mapValues(info.columns, (column) => {
                return {
                    field: column.column_name,
                    // 数据库原生类型
                    dbType: column.data_type,
                    precision: column.numeric_precision || null,
                    scale: column.numeric_scale || null,
                    // 归一化后的 Directus 类型
                    type: getLocalType(column),
                    // 解析后的默认值
                    defaultValue: getDefaultValue(column) ?? null,
                    // 其他属性
                    nullable: column.is_nullable ?? true,
                    generated: column.is_generated ?? false,
                    special: [],
                    alias: false,
                    searchable: true,
                };
            }),
        };
    }

    // Step 4: 应用 Directus 字段配置
    for (const field of fields) {
        if (!result.collections[field.collection]) continue;
        
        const existing = result.collections[field.collection]?.fields[field.field];
        const column = schemaOverview[field.collection]?.columns[field.field];
        const special = field.special ? toArray(field.special) : [];

        // special 会影响类型判断
        const type = (existing && getLocalType(column, { special })) || 'alias';

        result.collections[field.collection]!.fields[field.field] = {
            ...existing,
            special: special,
            note: field.note,
            validation: parseJSON(field.validation) ?? null,
            alias: existing?.alias ?? true,
            searchable: toBoolean(field.searchable) ?? true,
        };
    }

    // Step 5: 补充关系信息
    const relationsService = new RelationsService({ knex: database, schema: result });
    result.relations = await relationsService.readAll();

    return result;
}
```

---

## PostgreSQL 与 MySQL 对照示例

### 场景说明

假设我们有一张 `users` 表，在两个数据库中的定义如下：

**PostgreSQL DDL**:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL DEFAULT 'unknown@example.com',
    is_active BOOLEAN DEFAULT true,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    metadata JSONB DEFAULT '{}'::jsonb,
    score NUMERIC(10,2) DEFAULT 0.00
);
```

**MySQL DDL**:
```sql
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL DEFAULT 'unknown@example.com',
    is_active TINYINT(1) DEFAULT 1,
    status VARCHAR(20) DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    metadata JSON DEFAULT '{}',
    score DECIMAL(10,2) DEFAULT 0.00
);
```

### 对照 1: 方言层输出 (SchemaInspector.overview())

#### PostgreSQL 方言层输出

**文件**: `packages/schema/src/dialects/postgres.ts:86-230`

```typescript
// schemaInspector.overview() 返回的原始数据
{
  "users": {
    "primary": "id",
    "columns": {
      "id": {
        "table_name": "users",
        "column_name": "id",
        "data_type": "integer",
        "default_value": "AUTO_INCREMENT",  // 方言层已转换: 原来是 nextval('users_id_seq'::regclass)
        "is_nullable": false,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": 32,
        "numeric_scale": 0
      },
      "email": {
        "table_name": "users",
        "column_name": "email",
        "data_type": "character varying",
        "default_value": "unknown@example.com",  // 方言层已解析: 原来是 'unknown@example.com'::character varying
        "is_nullable": false,
        "is_generated": false,
        "max_length": 255,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "is_active": {
        "table_name": "users",
        "column_name": "is_active",
        "data_type": "boolean",
        "default_value": "true",  // 方言层直接返回
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "status": {
        "table_name": "users",
        "column_name": "status",
        "data_type": "character varying",
        "default_value": "active",
        "is_nullable": true,
        "is_generated": false,
        "max_length": 20,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "created_at": {
        "table_name": "users",
        "column_name": "created_at",
        "data_type": "timestamp with time zone",
        "default_value": "now()",  // 方言层解析后
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "metadata": {
        "table_name": "users",
        "column_name": "metadata",
        "data_type": "jsonb",
        "default_value": "{}",  // 原来是 '{}'::jsonb
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "score": {
        "table_name": "users",
        "column_name": "score",
        "data_type": "numeric",
        "default_value": "0.00",
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": 10,
        "numeric_scale": 2
      }
    }
  }
}
```

**PostgreSQL 方言层的关键转换**:

```typescript
// 1. 自增列检测: 检查 default_value 是否以 nextval( 开头
if (column.is_identity || column.default_value?.startsWith('nextval(')) {
    column.default_value = 'AUTO_INCREMENT';
}

// 2. 默认值解析: 移除 PostgreSQL 特有的类型转换后缀
export function parseDefaultValue(value: string | null): string | null {
    if (value === null) return null;
    if (value.startsWith('nextval(')) return value;  // 保留序列引用
    
    // 移除 :: 类型后缀: 'text'::character varying => 'text'
    value = value.split('::')[0] ?? null;
    
    if (value?.trim().toLowerCase() === 'null') return null;
    return stripQuotes(value);
}
```

#### MySQL 方言层输出

**文件**: `packages/schema/src/dialects/mysql.ts:83-137`

```typescript
// schemaInspector.overview() 返回的原始数据
{
  "users": {
    "primary": "id",
    "columns": {
      "id": {
        "table_name": "users",
        "column_name": "id",
        "data_type": "int",
        "default_value": "AUTO_INCREMENT",  // 方言层检测: EXTRA = 'auto_increment'
        "is_nullable": false,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": null,
        "numeric_scale": null,
        "column_key": "PRI",
        "extra": "auto_increment"
      },
      "email": {
        "table_name": "users",
        "column_name": "email",
        "data_type": "varchar",
        "default_value": "unknown@example.com",
        "is_nullable": false,
        "is_generated": false,
        "max_length": 255,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "is_active": {
        "table_name": "users",
        "column_name": "is_active",
        "data_type": "boolean",  // 方言层已转换: 原来是 tinyint(1)
        "default_value": "1",
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": null,
        "numeric_scale": null,
        "column_type": "tinyint(1)"  // 原始类型
      },
      "status": {
        "table_name": "users",
        "column_name": "status",
        "data_type": "varchar",
        "default_value": "active",
        "is_nullable": true,
        "is_generated": false,
        "max_length": 20,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "created_at": {
        "table_name": "users",
        "column_name": "created_at",
        "data_type": "timestamp",
        "default_value": "CURRENT_TIMESTAMP",
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "metadata": {
        "table_name": "users",
        "column_name": "metadata",
        "data_type": "json",
        "default_value": "{}",
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": null,
        "numeric_scale": null
      },
      "score": {
        "table_name": "users",
        "column_name": "score",
        "data_type": "decimal",
        "default_value": "0.00",
        "is_nullable": true,
        "is_generated": false,
        "max_length": null,
        "numeric_precision": 10,
        "numeric_scale": 2
      }
    }
  }
}
```

**MySQL 方言层的关键转换**:

```typescript
// 1. 自增列检测: 检查 EXTRA 字段
if (column.extra === 'auto_increment') {
    column.default_value = 'AUTO_INCREMENT';
}

// 2. 布尔类型约定: tinyint(1) => boolean
let dataType = column.data_type.replace(/\(.*?\)/, '');
if (column.data_type.startsWith('tinyint(1)')) {
    dataType = 'boolean';
}

// 3. 默认值解析 (较简单，没有类型后缀)
export function parseDefaultValue(value: string | null): string | null {
    if (value === null || value.trim().toLowerCase() === 'null') return null;
    return stripQuotes(value);
}
```

### 对照 2: 类型归一化 (getLocalType)

**文件**: `api/src/utils/get-local-type.ts:105-153`

这是统一层的核心功能，将数据库原生类型映射为 Directus 标准类型。

#### 类型映射对照表

| 字段 | PostgreSQL 原生类型 | MySQL 原生类型 | getLocalType 输出 |
|------|---------------------|----------------|-------------------|
| id | `integer` | `int` | `integer` |
| email | `character varying` | `varchar` | `string` |
| is_active | `boolean` | `boolean` (从 tinyint(1) 转换) | `boolean` |
| status | `character varying` | `varchar` | `string` |
| created_at | `timestamp with time zone` | `timestamp` | `timestamp` |
| metadata | `jsonb` | `json` | `json` |
| score | `numeric` (precision=10, scale=2) | `decimal` (precision=10, scale=2) | `decimal` |

#### 归一化代码流程

```typescript
// getLocalType 的核心逻辑
const localTypeMap: Record<string, Type | 'unknown'> = {
    // 共享类型
    boolean: 'boolean',
    integer: 'integer',
    int: 'integer',
    varchar: 'string',
    timestamp: 'timestamp',
    json: 'json',
    decimal: 'decimal',
    numeric: 'integer',  // 默认映射为 integer，但有特殊处理
    
    // PostgreSQL 特有
    'character varying': 'string',
    bool: 'boolean',
    jsonb: 'json',
    'timestamp with time zone': 'timestamp',
    int4: 'integer',
    
    // MySQL 特有
    tinyint: 'integer',
    text: 'text',
    datetime: 'dateTime',
};

export default function getLocalType(column?, field?): Type | 'unknown' {
    if (!column) return 'alias';
    
    const dataType = column.data_type.toLowerCase();
    // 移除括号中的长度信息: varchar(255) => varchar
    const type = localTypeMap[dataType.split('(')[0]!];
    
    // 特殊情况 1: PostgreSQL numeric 带 precision/scale => decimal
    if (dataType === 'numeric' && 
        column.numeric_precision !== null && 
        column.numeric_scale !== null) {
        return 'decimal';
    }
    
    // 特殊情况 2: special 字段覆盖类型
    const special = field?.special;
    if (special) {
        if (special.includes('cast-json')) return 'json';
        if (special.includes('uuid')) return 'uuid';
        if (special.includes('cast-timestamp')) return 'timestamp';
        // ...
    }
    
    return type ?? 'unknown';
}
```

#### 关键差异点

**1. 字符串类型**
- PostgreSQL: `character varying` → `string`
- MySQL: `varchar` → `string`
- 两者最终都映射为 `string`

**2. 布尔类型**
- PostgreSQL: 原生 `boolean` → `boolean`
- MySQL: 方言层先将 `tinyint(1)` 转换为 `boolean`，然后统一层映射为 `boolean`
- 注意: MySQL 的转换发生在**方言层**，而不是统一层

**3. 时间戳类型**
- PostgreSQL: `timestamp with time zone` → `timestamp`
- PostgreSQL: `timestamp without time zone` → `dateTime`
- MySQL: `timestamp` → `timestamp`
- MySQL: `datetime` → `dateTime`

**4. 数值类型 (最复杂)**
- PostgreSQL:
  - `numeric` 无 precision/scale → `integer` (默认)
  - `numeric(10,2)` → `decimal` (特殊处理)
- MySQL:
  - `decimal` → `decimal`
  - `decimal(10,2)` → `decimal`

### 对照 3: 默认值解析 (getDefaultValue)

**文件**: `api/src/utils/get-default-value.ts:8-65`

这是统一层的另一个核心功能，根据归一化后的类型转换默认值格式。

#### 默认值对照

| 字段 | PostgreSQL 方言层输出 | MySQL 方言层输出 | getDefaultValue 输出 |
|------|----------------------|-----------------|---------------------|
| id | `"AUTO_INCREMENT"` | `"AUTO_INCREMENT"` | `"AUTO_INCREMENT"` (特殊值) |
| email | `"unknown@example.com"` | `"unknown@example.com"` | `"unknown@example.com"` |
| is_active | `"true"` | `"1"` | `true` (布尔值) |
| status | `"active"` | `"active"` | `"active"` |
| created_at | `"now()"` | `"CURRENT_TIMESTAMP"` | `"now()"` / `"CURRENT_TIMESTAMP"` (保留原始) |
| metadata | `"{}"` | `"{}"` | `{}` (解析为对象) |
| score | `"0.00"` | `"0.00"` | `0.00` (数字) |

#### 解析代码流程

```typescript
export default function getDefaultValue(column, field?): any {
    const type = getLocalType(column, field);  // 先获取归一化类型
    const defaultValue = column.default_value ?? null;
    
    // 特殊情况: MySQL 的零日期
    if (defaultValue === '0000-00-00 00:00:00') return null;
    
    // 根据类型转换
    switch (type) {
        case 'bigInteger':
        case 'integer':
        case 'decimal':
        case 'float':
            // 字符串转数字
            return Number.isNaN(Number(defaultValue)) === false 
                ? Number(defaultValue) 
                : defaultValue;
        
        case 'boolean':
            // 各种格式转布尔
            return castToBoolean(defaultValue);
        
        case 'json':
            // 字符串解析为对象
            return castToObject(defaultValue);
        
        default:
            // 其他类型保持原样
            return defaultValue;
    }
}

// 布尔值转换 (处理多种格式)
function castToBoolean(value: any): boolean {
    if (typeof value === 'boolean') return value;
    
    // 数字格式
    if (value === 0 || value === '0') return false;
    if (value === 1 || value === '1') return true;
    
    // 字符串格式
    if (value === 'false' || value === false) return false;
    if (value === 'true' || value === true) return true;
    
    return Boolean(value);
}

// JSON 解析
function castToObject(value: any): any {
    if (typeof value === 'object') return value;
    if (typeof value === 'string') {
        try {
            return parseJSON(value);
        } catch (err) {
            return value;
        }
    }
    return {};
}
```

#### 关键差异点

**1. 自增列标记**
- 两个数据库的方言层都将自增列的 default_value 设为 `"AUTO_INCREMENT"`
- 统一层保留这个特殊值，不进行类型转换

**2. 布尔值差异**
- PostgreSQL 方言层输出 `"true"` 或 `"false"` (字符串)
- MySQL 方言层输出 `"1"` 或 `"0"` (字符串)
- 统一层的 `castToBoolean` 函数处理这两种格式：
  - `"true"` → `true`
  - `"1"` → `true`
  - `"false"` → `false`
  - `"0"` → `false`

**3. JSON 类型**
- 两个数据库的方言层都输出 `"{}"` (字符串)
- 统一层解析为实际对象 `{}`

**4. 数值类型**
- PostgreSQL: `"0.00"` (字符串) → `0.00` (数字)
- MySQL: `"0.00"` (字符串) → `0.00` (数字)
- 两者处理方式相同

**5. 函数默认值**
- PostgreSQL: `"now()"`
- MySQL: `"CURRENT_TIMESTAMP"`
- 统一层不解析这些函数调用，保持原样返回

### 对照 4: 最终输出对比

经过统一层处理后，两个数据库的 `users` 表结构几乎完全一致：

```typescript
// PostgreSQL 和 MySQL 处理后的输出 (几乎相同)
{
  "collections": {
    "users": {
      "collection": "users",
      "primary": "id",
      "singleton": false,
      "note": null,
      "accountability": "all",
      "fields": {
        "id": {
          "field": "id",
          "type": "integer",        // 归一化类型
          "dbType": "integer",      // PostgreSQL: "integer", MySQL: "int"
          "defaultValue": "AUTO_INCREMENT",
          "nullable": false,
          "generated": false,
          "precision": 32,          // PostgreSQL: 32, MySQL: null
          "scale": 0,               // PostgreSQL: 0, MySQL: null
          "special": [],
          "alias": false,
          "searchable": true
        },
        "email": {
          "field": "email",
          "type": "string",         // 归一化类型
          "dbType": "character varying",  // PostgreSQL
          // "dbType": "varchar",         // MySQL (唯一的差异点)
          "defaultValue": "unknown@example.com",
          "nullable": false,
          "generated": false,
          "maxLength": 255,
          "special": [],
          "alias": false,
          "searchable": true
        },
        "is_active": {
          "field": "is_active",
          "type": "boolean",        // 归一化类型
          "dbType": "boolean",      // PostgreSQL: "boolean", MySQL: "boolean" (从 tinyint(1) 转换)
          "defaultValue": true,     // 统一后的布尔值
          "nullable": true,
          "generated": false,
          "special": [],
          "alias": false,
          "searchable": true
        },
        "metadata": {
          "field": "metadata",
          "type": "json",           // 归一化类型
          "dbType": "jsonb",        // PostgreSQL: "jsonb", MySQL: "json"
          "defaultValue": {},       // 解析后的对象
          "nullable": true,
          "generated": false,
          "special": [],
          "alias": false,
          "searchable": true
        },
        "score": {
          "field": "score",
          "type": "decimal",        // 归一化类型
          "dbType": "numeric",      // PostgreSQL: "numeric", MySQL: "decimal"
          "defaultValue": 0.00,     // 解析后的数字
          "nullable": true,
          "generated": false,
          "precision": 10,
          "scale": 2,
          "special": [],
          "alias": false,
          "searchable": true
        }
      }
    }
  },
  "relations": []
}
```

### 对照总结

| 转换阶段 | PostgreSQL | MySQL | 统一后 |
|---------|-----------|-------|--------|
| **方言层 - 自增列** | `nextval('seq'::regclass)` → `"AUTO_INCREMENT"` | `EXTRA='auto_increment'` → `"AUTO_INCREMENT"` | `"AUTO_INCREMENT"` |
| **方言层 - 布尔类型** | 原生 `boolean` | `tinyint(1)` → `boolean` | `boolean` |
| **方言层 - 默认值** | `'text'::varchar` → `"text"` | `'text'` → `"text"` | `"text"` |
| **统一层 - 类型映射** | `character varying` → `string` | `varchar` → `string` | `string` |
| **统一层 - 类型映射** | `timestamp with time zone` → `timestamp` | `timestamp` → `timestamp` | `timestamp` |
| **统一层 - 类型映射** | `numeric(10,2)` → `decimal` | `decimal(10,2)` → `decimal` | `decimal` |
| **统一层 - 默认值解析** | `"true"` → `true` | `"1"` → `true` | `true` |
| **统一层 - 默认值解析** | `"{}"` → `{}` | `"{}"` → `{}` | `{}` |
| **统一层 - 默认值解析** | `"0.00"` → `0.00` | `"0.00"` → `0.00` | `0.00` |

**关键设计原则**:
1. **方言层**处理数据库特有的语法和格式差异
2. **统一层**负责将方言层输出映射为 Directus 标准类型
3. `dbType` 字段保留原始数据库类型用于调试
4. `type` 字段是 Directus 内部使用的标准类型

---

## (以下为原有内容，保持不变)

## SchemaInspector 接口定义

### 接口契约

**文件**: `packages/schema/src/types/schema-inspector.ts:7-36`

```typescript
export interface SchemaInspector {
    knex: Knex;

    // 获取完整的 schema 概览（最常用）
    overview: () => Promise<SchemaOverview>;

    // 表操作
    tables(): Promise<string[]>;
    tableInfo(): Promise<Table[]>;
    tableInfo(table: string): Promise<Table>;
    hasTable(table: string): Promise<boolean>;

    // 列操作
    columns(table?: string): Promise<{ table: string; column: string }[]>;
    columnInfo(): Promise<Column[]>;
    columnInfo(table?: string): Promise<Column[]>;
    columnInfo(table: string, column: string): Promise<Column>;
    hasColumn(table: string, column: string): Promise<boolean>;

    // 约束操作
    primary(table: string): Promise<string | null>;
    foreignKeys(table?: string): Promise<ForeignKey[]>;

    // 可选方法（部分数据库支持）
    withSchema?(schema: string): void;
}
```

### 统一类型定义

**Column 接口** (`packages/schema/src/types/column.ts`):

```typescript
export interface Column {
    name: string;
    table: string;
    data_type: string;
    default_value: string | number | boolean | null;
    max_length: number | null;
    numeric_precision: number | null;
    numeric_scale: number | null;

    // 布尔属性
    is_nullable: boolean;
    is_unique: boolean;
    is_indexed: boolean;
    is_primary_key: boolean;
    is_generated: boolean;
    has_auto_increment: boolean;

    // 外键信息
    foreign_key_table: string | null;
    foreign_key_column: string | null;

    // 数据库特定的可选属性
    comment?: string | null;           // SQLite/MSSQL 不支持
    schema?: string;                    // Postgres 独有
    foreign_key_schema?: string | null; // Postgres 独有
    generation_expression?: string | null;
}
```

**Table 接口** (`packages/schema/src/types/table.ts`):

```typescript
export interface Table {
    name: string;

    // 可选属性
    comment?: string | null;    // SQLite 不完整支持
    schema?: string;

    // MySQL 独有
    collation?: string;
    engine?: string;

    // Postgres 独有
    owner?: string;

    // SQLite 独有
    sql?: string;

    // MSSQL 独有
    catalog?: string;
}
```

## 工厂函数：createInspector

**文件**: `packages/schema/src/index.ts:16-46`

这是方言选择的核心逻辑，根据 Knex 客户端的构造函数名称来选择对应的 SchemaInspector 实现：

```typescript
export const createInspector = (knex: Knex): SchemaInspector => {
    let constructor: SchemaInspectorConstructor;

    switch (knex.client.constructor.name) {
        case 'Client_MySQL':
        case 'Client_MySQL2':
            constructor = MySQLSchemaInspector;
            break;
        case 'Client_PG':
            constructor = PostgresSchemaInspector;
            break;
        case 'Client_CockroachDB':
            constructor = CockroachDBSchemaInspector;
            break;
        case 'Client_SQLite3':
            constructor = SqliteSchemaInspector;
            break;
        case 'Client_Oracledb':
        case 'Client_Oracle':
            constructor = OracleDBSchemaInspector;
            break;
        case 'Client_MSSQL':
            constructor = MSSQLSchemaInspector;
            break;
        default:
            throw Error('Unsupported driver used: ' + knex.client.constructor.name);
    }

    return new constructor(knex);
};
```

## 各数据库方言实现差异

### 1. PostgreSQL

**核心特点**:
- 使用 PostgreSQL 系统目录表 (`pg_class`, `pg_attribute`, `pg_constraint` 等)
- 支持 `withSchema()` 方法切换 schema
- 特殊处理 PostGIS 几何类型

**关键实现示例** (`packages/schema/src/dialects/postgres.ts:86-230`):

```typescript
async overview(): Promise<SchemaOverview> {
    // 使用 information_schema + 系统表组合查询
    const [columnsResult, primaryKeysResult] = await Promise.all([
        this.knex.raw(
            `
            SELECT c.table_name, c.column_name, ...
            FROM information_schema.columns c
            LEFT JOIN information_schema.tables t
              ON c.table_name = t.table_name
            WHERE t.table_type = 'BASE TABLE'
              AND c.table_schema IN (${bindings});
            `,
            this.explodedSchema,
        ),
        this.knex.raw(
            `
            SELECT relname as table_name, pg_attribute.attname as column_name
            FROM pg_index, pg_class, pg_attribute, pg_namespace
            WHERE indrelid = pg_class.oid
              AND nspname IN (${bindings})
              ...
              AND indisprimary
            `,
            this.explodedSchema,
        ),
    ]);

    // 特殊处理: 检查 PostGIS 扩展
    const hasPostGIS = (await this.knex.raw(
        `SELECT oid FROM pg_proc WHERE proname = 'postgis_version'`
    )).rows.length > 0;

    if (hasPostGIS) {
        // 读取 geometry_columns 和 geography_columns
    }
    // ...
}
```

**默认值解析** (`packages/schema/src/dialects/postgres.ts:39-48`):

```typescript
export function parseDefaultValue(value: string | null): string | null {
    if (value === null) return null;
    if (value.startsWith('nextval(')) return value; // 保留序列引用

    value = value.split('::')[0] ?? null; // 移除类型转换: 'text'::character varying => 'text'

    if (value?.trim().toLowerCase() === 'null') return null;
    return stripQuotes(value);
}
```

### 2. MySQL

**核心特点**:
- 使用 `INFORMATION_SCHEMA` 系统视图
- 特殊处理 `tinyint(1)` 为 boolean 类型
- 使用 `SHOW KEYS` 等 MySQL 特有语句

**关键实现示例** (`packages/schema/src/dialects/mysql.ts:83-137`):

```typescript
async overview(): Promise<SchemaOverview> {
    const columns = await this.knex.raw(
        `
        SELECT
            C.TABLE_NAME as table_name,
            C.COLUMN_NAME as column_name,
            C.COLUMN_DEFAULT as default_value,
            C.IS_NULLABLE as is_nullable,
            C.COLUMN_TYPE as data_type,
            C.COLUMN_KEY as column_key,
            C.CHARACTER_MAXIMUM_LENGTH as max_length,
            C.EXTRA as extra
        FROM INFORMATION_SCHEMA.COLUMNS AS C
        LEFT JOIN INFORMATION_SCHEMA.TABLES AS T 
            ON C.TABLE_NAME = T.TABLE_NAME AND C.TABLE_SCHEMA = T.TABLE_SCHEMA
        WHERE
            T.TABLE_TYPE = 'BASE TABLE' AND
            C.TABLE_SCHEMA = ?;
        `,
        [this.knex.client.database()],
    );

    // 特殊处理: tinyint(1) => boolean
    let dataType = column.data_type.replace(/\(.*?\)/, '');
    if (column.data_type.startsWith('tinyint(1)')) {
        dataType = 'boolean';
    }
}
```

**主键查询** (`packages/schema/src/dialects/mysql.ts:333-341`):

```typescript
async primary(table: string): Promise<string | null> {
    const results = await this.knex.raw(
        `SHOW KEYS FROM ?? WHERE Key_name = 'PRIMARY'`, 
        table
    );
    // MySQL 特有语法
}
```

### 3. SQLite

**核心特点**:
- 使用 SQLite 的 `PRAGMA` 命令而非标准 SQL
- 不支持 `INFORMATION_SCHEMA`
- 类型系统非常灵活（动态类型）

**关键实现示例** (`packages/schema/src/dialects/sqlite.ts:38-77`):

```typescript
async overview(): Promise<SchemaOverview> {
    // 1. 查找包含 AUTOINCREMENT 的表
    const tablesWithAutoIncrementPrimaryKeys = (
        await this.knex.select('name')
            .from('sqlite_master')
            .whereRaw(`sql LIKE "%AUTOINCREMENT%"`)
    ).map(({ name }) => name);

    const tables = await this.tables();

    for (const table of tables) {
        // 2. 使用 PRAGMA 获取表结构
        const columns = await this.knex.raw<RawColumn[]>(
            `PRAGMA table_xinfo(??)`, 
            table
        );
        // ...
    }
}
```

**列信息查询** (`packages/schema/src/dialects/sqlite.ts:159-240`):

```typescript
async columnInfo(table?: string, column?: string) {
    const getColumnsForTable = async (table: string): Promise<Column[]> => {
        // 使用多个 PRAGMA 命令组合信息
        const columns: RawColumn[] = await this.knex.raw(
            `PRAGMA table_xinfo(??)`, 
            table
        );
        
        const foreignKeys = await this.knex.raw(
            `PRAGMA foreign_key_list(??)`, 
            table
        );
        
        const indexList = await this.knex.raw(
            `PRAGMA index_list(??)`, 
            table
        );
        
        // 逐个获取索引的详细信息
        const indexInfoList = await Promise.all(
            indexList.map((index) =>
                this.knex.raw(`PRAGMA index_info(??)`, index.name),
            ),
        );
        // ...
    };
}
```

**类型提取工具** (`packages/schema/src/utils/extract-type.ts`):

```typescript
// SQLite 的类型是动态的，需要从 CREATE TABLE 语句中解析
export default function extractType(type: string): string {
    if (!type) return 'unknown';
    
    // 提取类型名称，忽略括号内的长度/精度
    const match = type.match(/^([a-zA-Z]+)/);
    return match ? match[1].toLowerCase() : 'unknown';
}
```

### 4. Microsoft SQL Server (MSSQL)

**核心特点**:
- 混合使用 `INFORMATION_SCHEMA` 和 `sys.*` 系统视图
- 支持 `withSchema()` 方法（默认 schema 为 `dbo`）
- 特殊处理 `nvarchar` 等 Unicode 类型的长度计算

**关键实现示例** (`packages/schema/src/dialects/mssql.ts:287-402`):

```typescript
async columnInfo(table?: string, column?: string) {
    const dbName = this.knex.client.database();
    const schemaIdResult = await this.knex.select('schema_id')
        .from('sys.schemas')
        .where({ name: this.schema })
        .first();

    // 使用临时表存储索引信息
    const dbResult = await this.knex.transaction(async (trx) => {
        await trx.raw(`IF OBJECT_ID('tempdb..##IndexInfo') IS NOT NULL DROP TABLE ##IndexInfo;`);

        // 将索引信息插入临时表
        await trx.raw(`
            SELECT [ic].[object_id], [ic].[column_id], ...
            INTO ##IndexInfo
            FROM [sys].[index_columns] ic
            JOIN [sys].[indexes] ix ON [ix].[object_id] = [ic].[object_id];
        `);

        // 复杂的查询组合
        const query = trx
            .with('FilteredIndexInfo', this.knex.raw(`...`))
            .select(trx.raw(`
                [o].[name] AS [table],
                [c].[name] AS [name],
                [t].[name] AS [data_type],
                ...
                OBJECT_NAME ([fk].[referenced_object_id]) AS [foreign_key_table],
                COL_NAME ([fk].[referenced_object_id], [fk].[referenced_column_id]) AS [foreign_key_column]
            `))
            .from(trx.raw(`??.[sys].[columns] [c]`, [dbName]))
            .joinRaw(`JOIN [sys].[types] [t] ON [c].[user_type_id] = [t].[user_type_id]`)
            // ... 更多 JOIN
            .where({ 's.schema_id': schemaId });
        // ...
    });
}
```

**Unicode 类型长度处理** (`packages/schema/src/dialects/mssql.ts:54-72`):

```typescript
function parseMaxLength(rawColumn: RawColumn) {
    const max_length = Number(rawColumn.max_length);
    // ...
    
    // n-* 类型每个字符占 2 字节
    // varchar(100)  => max_length = 100
    // nvarchar(100) => max_length = 200 (需要除以 2)
    if (['nvarchar', 'nchar', 'ntext'].includes(rawColumn.data_type)) {
        return max_length === -1 ? null : max_length / 2;
    }
    return max_length === -1 ? null : max_length;
}
```

## 各数据库差异汇总表

| 特性 | PostgreSQL | MySQL | SQLite | MSSQL |
|------|-----------|-------|--------|-------|
| **元数据来源** | 系统表 (`pg_*`) + `information_schema` | `INFORMATION_SCHEMA` | `PRAGMA` 命令 | `sys.*` + `INFORMATION_SCHEMA` |
| **Schema 支持** | ✅ 支持 (默认 `public`) | ✅ (数据库即 schema) | ❌ 不支持 | ✅ (默认 `dbo`) |
| **withSchema() 方法** | ✅ | ❌ | ❌ | ✅ |
| **几何类型** | ✅ PostGIS 扩展 | ❌ | ❌ | ✅ (内置) |
| **注释支持** | ✅ | ✅ | ❌ | 部分 |
| **自增列检测** | `nextval()` 序列 | `EXTRA = 'auto_increment'` | `AUTOINCREMENT` 关键字 | `is_identity` 属性 |
| **外键信息** | `pg_constraint` | `REFERENTIAL_CONSTRAINTS` | `PRAGMA foreign_key_list` | `sys.foreign_keys` |
| **默认值格式** | `'value'::type` | 原始值 | 原始值 | `((value))` 嵌套括号 |

## API 层集成

### SchemaService

**文件**: `api/src/services/schema.ts`

SchemaService 是 Directus API 暴露给外部的 schema 操作服务：

```typescript
export class SchemaService {
    knex: Knex;
    accountability: Accountability | null;

    // 获取当前数据库快照
    async snapshot(): Promise<Snapshot> {
        if (this.accountability?.admin !== true) throw new ForbiddenError();
        const currentSnapshot = await getSnapshot({ database: this.knex });
        return currentSnapshot;
    }

    // 应用 schema 差异
    async apply(payload: SnapshotDiffWithHash, options?: { force?: boolean }): Promise<void> {
        // 验证 + 应用
        await applyDiff(currentSnapshot, payload.diff, { database: this.knex });
    }

    // 比较两个 schema
    async diff(snapshot: Snapshot, options?: ...): Promise<SnapshotDiff | null> {
        const diff = getSnapshotDiff(currentSnapshot, snapshot);
        return diff;
    }
}
```

### getSnapshot 流程

**文件**: `api/src/utils/get-snapshot.ts`

```
getSnapshot()
    │
    ├──> getSchema()           # 使用 SchemaInspector 获取数据库原生结构
    │       │
    │       └──> createInspector(knex)
    │            └──> inspector.overview()
    │
    ├──> CollectionsService    # 读取 Directus 集合元数据
    ├──> FieldsService         # 读取 Directus 字段元数据
    └──> RelationsService      # 读取 Directus 关系元数据
```

### 调用链示例

```typescript
// 1. 创建 Inspector (工厂模式)
import { createInspector } from '@directus/schema';
const inspector = createInspector(knex);

// 2. 获取 schema 概览
const overview = await inspector.overview();
// 返回 SchemaOverview 类型:
// {
//   "users": {
//     "primary": "id",
//     "columns": {
//       "id": { data_type: "integer", default_value: "AUTO_INCREMENT", ... },
//       "email": { data_type: "varchar", max_length: 255, ... }
//     }
//   }
// }

// 3. 获取详细列信息
const columns = await inspector.columnInfo('users');

// 4. 获取外键信息
const foreignKeys = await inspector.foreignKeys('articles');
```

## 设计模式总结

### 1. 策略模式 (Strategy Pattern)
- **SchemaInspector** 接口定义了策略契约
- 每个数据库方言是一个具体策略实现
- **createInspector()** 工厂函数根据上下文选择策略

### 2. 适配器模式 (Adapter Pattern)
- 每个方言实现将数据库特有的 API 适配到统一的 SchemaInspector 接口
- 例如:
  - PostgreSQL 的 `pg_class` 查询 → `tables()` 方法
  - SQLite 的 `PRAGMA table_xinfo()` → `columnInfo()` 方法

### 3. 空对象/可选属性模式
- **Column** 和 **Table** 接口使用可选属性处理数据库差异
- 例如:
  - `schema?: string` - 只有 Postgres/MSSQL 有
  - `engine?: string` - 只有 MySQL 有
  - `comment?: string` - SQLite/MSSQL 不完整支持

### 4. 工厂模式
- **createInspector()** 根据 Knex 客户端类型动态创建对应的 Inspector
- 解耦了调用者和具体实现

## 关键差异处理技巧

### 1. 默认值解析
不同数据库返回的默认值格式差异很大:

| 数据库 | 默认值格式示例 | 解析策略 |
|--------|--------------|---------|
| PostgreSQL | `'example'::character varying` | 移除 `::type` 后缀，解引号 |
| PostgreSQL | `nextval('seq'::regclass)` | 保留序列引用 |
| MySQL | `'example'` | 直接解引号 |
| MSSQL | `((N'example'))` | 移除多层括号 |
| SQLite | `'example'` | 直接解引号 |

### 2. 自增列检测
| 数据库 | 检测方式 |
|--------|---------