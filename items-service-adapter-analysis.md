# Directus ItemsService 与数据库适配架构分析

## 一、整体架构概览

Directus 通过精心设计的多层架构实现了 REST 和 GraphQL 请求的统一处理，以及多数据库的无缝适配。核心架构如下：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API Entry Layer                                   │
│  ┌─────────────────────┐          ┌─────────────────────┐                   │
│  │   REST Controller   │          │   GraphQL Service   │                   │
│  │  (controllers/*)    │          │  (services/graphql) │                   │
│  └──────────┬──────────┘          └──────────┬──────────┘                   │
└─────────────┼────────────────────────────────┼────────────────────────────────┘
              │                                │
              │  ┌───────────────────────────┐ │
              │  │    getService() 工具      │ │
              │  │  (utils/get-service.ts)   │ │
              │  └─────────────┬─────────────┘ │
              │                │                 │
              ▼                ▼                 │
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Unified Service Layer                                 │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        ItemsService (核心)                              │   │
│  │  - createOne/createMany    │  - readByQuery/readOne/readMany         │   │
│  │  - updateOne/updateMany    │  - deleteOne/deleteMany                 │   │
│  │  - upsertOne/upsertMany    │  - readSingleton/upsertSingleton        │   │
│  └────────────────────────────┬──────────────────────────────────────────┘   │
│                               │                                                │
│              ┌────────────────┼────────────────┐                             │
│              ▼                ▼                ▼                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  PayloadService │  │   MetaService   │  │  ActivityService │             │
│  │  (数据处理/转换) │  │  (元数据计算)   │  │  (操作审计)     │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
└───────────────────────┬────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Query Execution Layer                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    AST 转换与执行管道                                   │   │
│  │                                                                         │   │
│  │  Query Object ──► getAstFromQuery() ──► AST ──► runAst()            │   │
│  │                         │                              │               │   │
│  │                         ▼                              ▼               │   │
│  │                  ┌─────────────┐              ┌─────────────┐       │   │
│  │                  │ parseFields │              │  getDBQuery │       │   │
│  │                  │ (字段解析)   │              │ (构建Knex)  │       │   │
│  │                  └─────────────┘              └─────────────┘       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└───────────────────────┬────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Database Abstraction Layer                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                           Knex.js 核心                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │   │
│  │  │ getDatabase │  │getDatabase- │  │      getHelpers()           │  │   │
│  │  │  (连接管理)  │  │ Client      │  │  (方言特定的辅助函数)        │  │   │
│  │  │             │  │ (类型检测)   │  │                             │  │   │
│  │  └─────────────┘  └─────────────┘  │  ┌───────────────────────┐  │  │   │
│  │                                      │  │ date: 日期函数        │  │  │   │
│  │                                      │  │ st: 空间几何函数      │  │  │   │
│  │                                      │  │ schema: 模式操作      │  │  │   │
│  │                                      │  │ sequence: 序列管理    │  │  │   │
│  │                                      │  │ number: 数值处理      │  │  │   │
│  │                                      │  └───────────────────────┘  │  │   │
│  │                                      └─────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Postgres │ │  MySQL   │ │  SQLite  │ │  MSSQL   │ │  Oracle  │       │
│  │CockroachDB│ │ MariaDB  │ │          │ │          │ │          │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、REST 与 GraphQL 统一到 ItemsService

### 2.1 统一入口设计

Directus 的核心设计理念是**将所有数据操作统一收敛到 ItemsService**，无论请求来自 REST 还是 GraphQL。

#### REST 控制器调用方式 (`api/src/controllers/items.ts`)

REST 控制器直接实例化 ItemsService：

```typescript
// 创建操作
router.post('/:collection', collectionExists, asyncHandler(async (req, res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    if (Array.isArray(req.body)) {
        const keys = await service.createMany(req.body);
        // ...
    } else {
        const key = await service.createOne(req.body);
        // ...
    }
}));

// 查询操作
const readHandler = asyncHandler(async (req, res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    if (req.singleton) {
        result = await service.readSingleton(req.sanitizedQuery);
    } else if (req.body.keys) {
        result = await service.readMany(req.body.keys, req.sanitizedQuery);
    } else {
        result = await service.readByQuery(req.sanitizedQuery);
    }
});
```

**特点**：
- 每个 REST 路由处理器都创建新的 ItemsService 实例
- 直接调用 `createOne/createMany/readByQuery/readOne` 等方法
- 通过 `req.sanitizedQuery` 传递查询参数（已通过中间件处理）

#### GraphQL 解析器调用方式 (`api/src/services/graphql/resolvers/`)

GraphQL 使用 `getService()` 工具函数获取服务实例：

**查询解析器** (`resolvers/query.ts`):
```typescript
export async function resolveQuery(gql: GraphQLService, info: GraphQLResolveInfo) {
    // 1. 解析 GraphQL 字段名和参数
    let collection = info.fieldName;
    if (gql.scope === 'system') collection = `directus_${collection}`;
    
    // 2. 解析参数为 Directus Query 对象
    const args = parseArgs(info.fieldNodes[0]!.arguments, info.variableValues);
    const query = await getQuery(args, gql.schema, selections, ...);
    
    // 3. 通过 GraphQLService.read() 间接调用 ItemsService
    const result = await gql.read(collection, query, args['id']);
    return result;
}
```

**变更解析器** (`resolvers/mutation.ts`):
```typescript
export async function resolveMutation(gql: GraphQLService, args, info) {
    // 解析动作和集合
    const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
    let collection = info.fieldName.substring(action.length + 1);
    
    // 获取 Query 对象
    const query = await getQuery(args, gql.schema, selections, ...);
    
    // 获取服务实例
    const service = getService(collection, {
        knex: gql.knex,
        accountability: gql.accountability,
        schema: gql.schema,
    });
    
    // 根据动作调用不同方法
    if (single) {
        if (action === 'create') {
            const key = await service.createOne(args['data']);
            return hasQuery ? await service.readOne(key, query) : true;
        }
        if (action === 'update') {
            const key = await service.updateOne(args['id'], args['data']);
            return hasQuery ? await service.readOne(key, query) : true;
        }
        if (action === 'delete') {
            await service.deleteOne(args['id']);
            return { id: args['id'] };
        }
    }
}
```

**GraphQLService 中的 read 方法** (`services/graphql/index.ts`):
```typescript
async read(collection: string, query: Query, id?: PrimaryKey) {
    const service = getService(collection, {
        knex: this.knex,
        accountability: this.accountability,
        schema: this.schema,
    });

    if (this.schema.collections[collection]!.singleton)
        return await service.readSingleton(query, { stripNonRequested: false });

    if (id) return await service.readOne(id, query, { stripNonRequested: false });

    return await service.readByQuery(query, { stripNonRequested: false });
}
```

### 2.2 getService() 服务工厂

`utils/get-service.ts` 是一个智能服务工厂，根据集合名称返回对应的服务：

```typescript
export function getService(collection: string, opts: AbstractServiceOptions): ItemsService {
    switch (collection) {
        // 系统集合使用专门的 Service（继承自 ItemsService）
        case 'directus_users':
            return new UsersService(opts);
        case 'directus_files':
            return new FilesService(opts);
        case 'directus_activity':
            return new ActivityService(opts);
        // ... 更多系统集合
        
        // 自定义集合使用通用 ItemsService
        default:
            if (collection.startsWith('directus_')) throw new ForbiddenError();
            return new ItemsService(collection, opts);
    }
}
```

**设计优势**：
1. **统一接口**：所有 Service 都继承或兼容 ItemsService 接口
2. **系统集合增强**：系统集合（如 `directus_users`）有特殊业务逻辑
3. **权限控制**：通过 switch-case 精确控制哪些集合可访问

### 2.3 统一调用模式对比

| 维度 | REST API | GraphQL |
|------|---------|---------|
| **服务获取** | `new ItemsService(collection, opts)` | `getService(collection, opts)` |
| **参数来源** | Express Request (query params, body) | GraphQL AST + variables |
| **Query 对象** | 由 `sanitizeQuery` 中间件生成 | 由 `getQuery()` 解析器生成 |
| **方法调用** | 直接调用 service 方法 | 解析后调用对应方法 |
| **权限上下文** | `req.accountability` | `gql.accountability` |

---

## 三、ItemsService 核心实现分析

### 3.1 类结构与依赖

```typescript
export class ItemsService<Item extends AnyItem = AnyItem> implements AbstractService<Item> {
    collection: Collection;           // 当前操作的集合名
    knex: Knex;                       // 数据库连接实例
    accountability: Accountability | null;  // 用户权限上下文
    eventScope: string;               // 事件作用域（用于 hooks）
    schema: SchemaOverview;           // 数据库元数据
    cache: Keyv<any> | null;          // 缓存实例
    nested: string[];                 // 嵌套关系追踪

    constructor(collection: Collection, options: AbstractServiceOptions) {
        this.collection = collection;
        this.knex = options.knex || getDatabase();  // 使用传入的或全局连接
        this.accountability = options.accountability || null;
        this.eventScope = isSystemCollection(this.collection) 
            ? this.collection.substring(9)  // 去掉 'directus_' 前缀
            : 'items';
        this.schema = options.schema;
        this.cache = getCache().cache;
        this.nested = options.nested ?? [];
    }
}
```

### 3.2 CRUD 方法实现模式

#### 查询操作 (`readByQuery`)

```typescript
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
    // 1. 触发 query filter hook（允许修改查询）
    const updatedQuery = opts?.emitEvents !== false
        ? await emitter.emitFilter(
            this.eventScope === 'items'
                ? ['items.query', `${this.collection}.items.query`]
                : `${this.eventScope}.query`,
            query,
            { collection: this.collection },
            { database: this.knex, schema: this.schema, accountability: this.accountability },
        )
        : query;

    // 2. 将 Query 对象转换为 AST
    let ast = await getAstFromQuery(
        {
            collection: this.collection,
            query: updatedQuery,
            accountability: this.accountability,
        },
        {
            schema: this.schema,
            knex: this.knex,
        },
    );

    // 3. 应用权限处理（注入权限过滤条件）
    ast = await processAst(
        { ast, action: 'read', accountability: this.accountability },
        { knex: this.knex, schema: this.schema },
    );

    // 4. 执行 AST 获取数据
    const records = await runAst(ast, this.schema, this.accountability, {
        knex: this.knex,
        stripNonRequested: opts?.stripNonRequested !== undefined ? opts.stripNonRequested : true,
    });

    // 5. 触发 read filter hook（允许修改返回数据）
    const filteredRecords = opts?.emitEvents !== false
        ? await emitter.emitFilter(
            this.eventScope === 'items' 
                ? ['items.read', `${this.collection}.items.read`] 
                : `${this.eventScope}.read`,
            records,
            { query: updatedQuery, collection: this.collection },
            { database: this.knex, schema: this.schema, accountability: this.accountability },
        )
        : records;

    // 6. 触发 read action hook（审计、日志等）
    if (opts?.emitEvents !== false) {
        emitter.emitAction(
            this.eventScope === 'items' 
                ? ['items.read', `${this.collection}.items.read`] 
                : `${this.eventScope}.read`,
            { payload: filteredRecords, query: updatedQuery, collection: this.collection },
            { database: getDatabase(), schema: this.schema, accountability: this.accountability },
        );
    }

    return filteredRecords as Item[];
}
```

#### 创建操作 (`createOne`)

```typescript
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    const primaryKeyField = this.schema.collections[this.collection]!.primary;
    
    // 使用事务保证原子性
    const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
        // 1. 触发 create filter hook
        const payloadAfterHooks = opts.emitEvents !== false
            ? await emitter.emitFilter(
                this.eventScope === 'items'
                    ? ['items.create', `${this.collection}.items.create`]
                    : `${this.eventScope}.create`,
                payload,
                { collection: this.collection },
                { database: trx, schema: this.schema, accountability: this.accountability },
            )
            : payload;

        // 2. 处理权限预设值（如 created_by 等自动填充字段）
        const payloadWithPresets = this.accountability
            ? await processPayload(
                {
                    accountability: this.accountability,
                    action: 'create',
                    collection: this.collection,
                    payload: payloadAfterHooks,
                    nested: this.nested,
                },
                { knex: trx, schema: this.schema },
            )
            : payloadAfterHooks;

        // 3. 使用 PayloadService 处理复杂关系数据
        const payloadService = new PayloadService(this.collection, {
            accountability: this.accountability,
            knex: trx,
            schema: this.schema,
            nested: this.nested,
            overwriteDefaults: opts.overwriteDefaults,
        });

        // 处理多对一关系
        const { payload: payloadWithM2O, ... } = await payloadService.processM2O(payloadWithPresets, opts);
        
        // 处理任对一关系
        const { payload: payloadWithA2O, ... } = await payloadService.processA2O(payloadWithM2O, opts);
        
        // 类型转换
        const payloadWithTypeCasting = await payloadService.processValues('create', payloadWithoutAliases);

        // 4. 执行数据库插入
        try {
            const result = await trx
                .insert(payloadWithoutAliases)
                .into(this.collection)
                .returning(primaryKeyField, returningOptions)
                .then((result) => result[0]);
            // ...
        } catch (err: any) {
            const dbError = await translateDatabaseError(err, data);
            throw dbError;
        }

        // 5. 处理一对多关系（需要先有主键）
        const { ... } = await payloadService.processO2M(payloadWithPresets, primaryKey, opts);

        // 6. 创建活动记录（如果启用了 accountability）
        if (opts.skipTracking !== true && this.accountability && ...) {
            const activityService = new ActivityService({ knex: trx, schema: this.schema });
            const activity = await activityService.createOne({
                action: Action.CREATE,
                user: this.accountability!.user,
                collection: this.collection,
                // ...
            });

            // 创建修订记录
            if (this.schema.collections[this.collection]!.accountability === 'all') {
                const revisionsService = new RevisionsService({ knex: trx, schema: this.schema });
                // ...
            }
        }

        return primaryKey;
    });

    // 7. 事务外触发 action hook
    if (opts.emitEvents !== false) {
        emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
    }

    // 8. 清除缓存
    if (shouldClearCache(this.cache, opts, this.collection)) {
        await this.cache.clear();
    }

    return primaryKey;
}
```

### 3.3 方法调用层级关系

```
readByQuery (最通用)
  ├── readOne (单条查询)
  │     └── 构建 { filter: { id: { _eq: key } } } 后调用 readByQuery
  │
  ├── readMany (批量查询)
  │     └── 构建 { filter: { _and: [{ id: { _in: keys } }] } } 后调用 readByQuery
  │
  └── readSingleton (单例读取)
        └── 设置 limit: 1 后调用 readByQuery

createOne (单条创建)
  └── createMany (批量创建)
        └── 循环调用 createOne，在同一事务中

updateOne (单条更新)
  ├── updateMany (按主键批量更新)
  │     └── 核心更新逻辑
  │
  ├── updateBatch (批量不同更新)
  │     └── 循环调用 updateOne
  │
  └── updateByQuery (按查询更新)
        ├── getKeysByQuery (获取符合条件的主键)
        │     └── 调用 readByQuery 只查询主键
        │
        └── updateMany (按获取的主键更新)

deleteOne (单条删除)
  ├── deleteMany (批量删除)
  │     └── 核心删除逻辑
  │
  └── deleteByQuery (按查询删除)
        ├── getKeysByQuery
        └── deleteMany

upsertOne (单条 upsert)
  ├── 查询是否存在
  ├── 存在则 updateOne
  └── 不存在则 createOne

upsertSingleton (单例 upsert)
  └── 逻辑同上，但针对单例集合
```

---

## 四、数据库适配层分析

### 4.1 数据库连接管理 (`database/index.ts`)

#### 连接初始化

```typescript
export function getDatabase(): Knex {
    if (database) return database;  // 单例模式

    const env = useEnv();
    
    // 从环境变量提取配置
    const {
        client,
        version,
        searchPath,
        connectionString,
        pool: poolConfig = {},
        ...connectionConfig
    } = getConfigFromEnv('DB_', { omitPrefix: 'DB_EXCLUDE_TABLES' });

    // 构建 Knex 配置
    const knexConfig: Knex.Config = {
        client,
        version,
        searchPath,
        connection: connectionString || connectionConfig,
        log: { /* 日志处理 */ },
        pool: poolConfig,
    };

    // 数据库特定配置
    if (client === 'sqlite3') {
        knexConfig.useNullAsDefault = true;
        poolConfig.afterCreate = (conn: any, callback: any) => {
            // SQLite: 启用外键约束
            conn.run('PRAGMA foreign_keys = ON');
            callback(null, conn);
        };
    }

    if (client === 'cockroachdb') {
        poolConfig.afterCreate = (conn: any, callback: any) => {
            // CockroachDB: 设置序列和整数类型
            conn.query('SET serial_normalization = "sql_sequence"');
            conn.query('SET default_int_size = 4');
            callback(null, conn);
        };
    }

    if (client === 'oracledb') {
        poolConfig.afterCreate = (conn: any, callback: any) => {
            // Oracle: 设置日期格式
            conn.execute('ALTER SESSION SET NLS_TIMESTAMP_FORMAT = \'YYYY-MM-DD"T"HH24:MI:SS.FF3"Z"\'');
            conn.execute("ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD'");
            callback(null, conn);
        };
    }

    if (client === 'mysql') {
        // MySQL: 使用 mysql2 驱动
        Object.assign(knexConfig, { client: 'mysql2' });
    }

    if (client === 'mssql') {
        // MSSQL: 禁用自动时区转换
        merge(knexConfig, { connection: { options: { useUTC: false } } });
    }

    database = knex.default(knexConfig);
    
    // 添加查询监控
    database
        .on('query', ({ __knexUid }) => {
            times.set(__knexUid, performance.now());
        })
        .on('query-response', (_response, queryInfo) => {
            const delta = performance.now() - times.get(queryInfo.__knexUid);
            metrics?.getDatabaseResponseMetric()?.observe(delta);
            logger.trace(`[${delta.toFixed(3)}ms] ${queryInfo.sql} [...]`);
        });

    return database;
}
```

#### 数据库类型检测

```typescript
export function getDatabaseClient(database?: Knex): DatabaseClient {
    database = database ?? getDatabase();

    // 根据 Knex 客户端构造函数名判断
    switch (database.client.constructor.name) {
        case 'Client_MySQL2':
            return 'mysql';
        case 'Client_PG':
            return 'postgres';
        case 'Client_CockroachDB':
            return 'cockroachdb';
        case 'Client_SQLite3':
            return 'sqlite';
        case 'Client_Oracledb':
        case 'Client_Oracle':
            return 'oracle';
        case 'Client_MSSQL':
            return 'mssql';
        case 'Client_Redshift':
            return 'redshift';
    }

    throw new Error(`Couldn't extract database client`);
}
```

### 4.2 方言辅助系统 (`database/helpers/`)

#### 辅助函数工厂

```typescript
// database/helpers/index.ts
export function getHelpers(database: Knex) {
    const client = getDatabaseClient(database);

    return {
        date: new dateHelpers[client](database),           // 日期函数
        st: new geometryHelpers[client](database),          // 空间几何函数
        schema: new schemaHelpers[client](database),        // 模式操作
        sequence: new sequenceHelpers[client](database),    // 序列管理
        number: new numberHelpers[client](database),        // 数值处理
        capabilities: new capabilitiesHelpers[client](database),  // 能力检测
    };
}
```

#### 方言实现目录结构

```
database/helpers/
├── index.ts              # 工厂函数
├── types.ts              # 类型定义
├── capabilities/         # 数据库能力检测
│   ├── index.ts
│   └── dialects/
│       ├── default.ts
│       ├── mysql.ts
│       ├── postgres.ts
│       ├── sqlite.ts
│       └── ...
├── date/                 # 日期函数
│   ├── index.ts
│   └── dialects/
│       ├── default.ts
│       ├── mysql.ts
│       ├── oracle.ts
│       ├── postgres.ts
│       ├── sqlite.ts
│       └── mssql.ts
├── fn/                   # 通用函数（JSON 等）
│   ├── index.ts
│   └── dialects/
├── geometry/             # 空间几何函数
│   ├── index.ts
│   └── dialects/
│       ├── mysql.ts      # MySQL: ST_* 函数
│       ├── postgres.ts   # PostGIS
│       ├── sqlite.ts     # SpatiaLite
│       └── ...
├── number/               # 数值处理（大数、精度）
│   ├── index.ts
│   └── dialects/
├── schema/               # 模式操作
│   ├── index.ts
│   ├── types.ts
│   └── dialects/
│       ├── default.ts
│       ├── cockroachdb.ts
│       ├── mssql.ts
│       ├── mysql.ts
│       ├── oracle.ts
│       ├── postgres.ts
│       └── sqlite.ts
└── sequence/             # 序列管理（自增重置等）
    ├── index.ts
    ├── types.ts
    └── dialects/
        ├── default.ts
        └── postgres.ts
```

#### 示例：日期函数方言差异

```typescript
// PostgreSQL 实现 (date/dialects/postgres.ts)
export class DateHelperPostgres extends DateHelper {
    dateField(field: string): Knex.Raw {
        // PostgreSQL: 日期转换
        return this.knex.raw(`??::date`, [field]);
    }

    dateFieldAsISO(field: string): Knex.Raw {
        // ISO 格式转换
        return this.knex.raw(`to_char(??::date, 'YYYY-MM-DD')`, [field]);
    }

    timestampFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`to_char(?? at time zone 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"')`, [field]);
    }
}

// MySQL 实现 (date/dialects/mysql.ts)
export class DateHelperMySQL extends DateHelper {
    dateField(field: string): Knex.Raw {
        // MySQL: 日期转换
        return this.knex.raw(`date(??)`, [field]);
    }

    dateFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`date_format(??, '%Y-%m-%d')`, [field]);
    }

    timestampFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`date_format(convert_tz(??, @@session.time_zone, '+00:00'), '%Y-%m-%dT%H:%i:%sZ')`, [field]);
    }
}

// SQLite 实现 (date/dialects/sqlite.ts)
export class DateHelperSQLite extends DateHelper {
    dateField(field: string): Knex.Raw {
        // SQLite: 使用内置日期函数
        return this.knex.raw(`date(??)`, [field]);
    }

    dateFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`strftime('%Y-%m-%d', ??)`, [field]);
    }

    timestampFieldAsISO(field: string): Knex.Raw {
        // SQLite: 处理 Unix 时间戳和 ISO 字符串
        return this.knex.raw(
            `case
                when typeof(??) = 'integer' then strftime('%Y-%m-%dT%H:%M:%SZ', datetime(??, 'unixepoch'))
                else strftime('%Y-%m-%dT%H:%M:%SZ', ??)
            end`,
            [field, field, field]
        );
    }
}
```

#### 示例：序列重置

```typescript
// PostgreSQL 特定实现 (sequence/dialects/postgres.ts)
export class SequenceHelperPostgres extends SequenceHelper {
    async resetAutoIncrementSequence(collection: string, pkField: string): Promise<void> {
        // 获取序列名
        const [row] = await this.knex
            .select(this.knex.raw('pg_get_serial_sequence(?, ?) as sequence'), [collection, pkField])
            .from(this.knex.raw('(VALUES (1)) as temp(val)'));

        if (!row.sequence) return;

        // 计算新的序列值（max(pk) + 1）
        const maxValueResult = await this.knex.max(pkField, { as: 'max' }).from(collection);
        const maxValue = maxValueResult[0]?.max;

        const newSequenceValue = typeof maxValue === 'number' ? maxValue + 1 : 1;

        // 重置序列
        await this.knex.raw('SELECT setval(?, ?, false)', [row.sequence, newSequenceValue]);
    }
}

// 默认实现（大多数数据库不需要或有其他方式）
export class SequenceHelperDefault extends SequenceHelper {
    async resetAutoIncrementSequence(): Promise<void> {
        // 默认不做任何操作
        return;
    }
}
```

### 4.3 查询执行管道 (`run-ast/`)

#### AST 到 SQL 的转换流程

```
┌────────────────────────────────────────────────────────────────────────┐
│                           runAst() 执行流程                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 解析当前层级字段                                                     │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  parseCurrentLevel(schema, collection, children, query)    │   │
│     │  ├── fieldNodes: 要查询的字段列表                            │   │
│     │  ├── primaryKeyField: 主键字段名                            │   │
│     │  └── nestedCollectionNodes: 关联关系节点                     │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  2. 获取权限（非管理员）                                                 │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  if (accountability && !accountability.admin) {             │   │
│     │      policies = fetchPolicies(accountability)                │   │
│     │      permissions = fetchPermissions({ action: 'read' })     │   │
│     │  }                                                            │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  3. 构建 Knex 查询                                                       │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  getDBQuery({                                                 │   │
│     │      table, fieldNodes, o2mNodes, query, cases, permissions │   │
│     │  }, { schema, knex })                                         │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  4. 执行查询获取原始数据                                                 │
│     const rawItems: Item | Item[] = await dbQuery;                    │
│                                                                          │
│  5. 数据类型转换                                                         │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  payloadService.processValues('read', rawItems, aliasMap)   │   │
│     │  ├── 日期格式转换                                             │   │
│     │  ├── JSON 解析                                                │   │
│     │  └── 特殊字段类型处理                                         │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  6. 递归处理嵌套关系                                                     │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  for (const nestedNode of nestedNodes) {                     │   │
│     │      if (nestedNode.type === 'o2m') {                        │   │
│     │          // 批量处理一对多关系（支持分页）                    │   │
│     │          while (hasMore) {                                    │   │
│     │              nestedItems = runAst(node, ...)                 │   │
│     │              items = mergeWithParentItems(...)               │   │
│     │          }                                                     │   │
│     │      } else {                                                  │   │
│     │          // 处理多对一、任对一关系                            │   │
│     │          nestedItems = runAst(node, ...)                     │   │
│     │          items = mergeWithParentItems(...)                   │   │
│     │      }                                                         │   │
│     │  }                                                             │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  7. 移除临时字段（如关联查询所需的外键）                                 │
│     items = removeTemporaryFields(schema, items, originalAST)         │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

#### getDBQuery 核心实现

```typescript
// database/run-ast/lib/get-db-qy.ts
export function getDBQuery(
    options: {
        table: string;
        fieldNodes: FieldNode[];
        o2mNodes: O2MNode[];
        query: Query;
        cases: Filter[];
        permissions: Permission[];
    },
    context: { schema: SchemaOverview; knex: Knex },
) {
    const { table, fieldNodes, o2mNodes, query, cases, permissions } = options;
    const { schema, knex } = context;

    // 创建基础查询
    let dbQuery = knex.select().from(table);

    // 应用各种查询子句
    // 1. 应用过滤器
    applyFilter(dbQuery, query.filter, { table, cases, permissions, schema, knex });
    
    // 2. 应用排序
    applySort(dbQuery, query.sort, { table, schema, knex });
    
    // 3. 应用分页
    applyPagination(dbQuery, { limit: query.limit, offset: query.offset, page: query.page });
    
    // 4. 应用聚合
    if (query.aggregate) {
        applyAggregate(dbQuery, query.aggregate, { table, schema, knex });
    }
    
    // 5. 应用分组
    if (query.group) {
        applyGroup(dbQuery, query.group, { table, schema, knex });
    }

    // 6. 处理搜索
    if (query.search) {
        applySearch(dbQuery, query.search, { table, schema, knex });
    }

    return dbQuery;
}
```

---

## 五、关键设计模式与架构优势

### 5.1 设计模式应用

| 设计模式 | 应用场景 | 实现位置 |
|---------|---------|---------|
| **Repository 模式** | ItemsService 作为数据访问仓库 | `services/items.ts` |
| **Service 模式** | 业务逻辑封装（ItemsService 及子类） | `services/*` |
| **Factory 模式** | getService() 根据集合类型创建对应服务 | `utils/get-service.ts` |
| **Adapter 模式** | Knex.js 适配不同数据库驱动 | `database/index.ts` |
| **Strategy 模式** | 方言辅助函数（每种数据库一个策略） | `database/helpers/*/dialects/` |
| **Template Method** | readByQuery 固定的 Hook + 执行流程 | `services/items.ts` |
| **Unit of Work** | Transaction 包装多个操作 | `utils/transaction.ts` |
| **Builder 模式** | AST 构建、Knex 查询构建 | `getAstFromQuery`, `getDBQuery` |

### 5.2 架构优势

#### 1. 统一数据访问层
- **单一真相源**：所有数据操作通过 ItemsService，保证业务逻辑一致性
- **易于维护**：修改数据逻辑只需修改一处
- **权限集中**：权限检查在 Service 层统一实施

#### 2. 灵活的数据库适配
- **Knex 抽象**：SQL 查询构建与具体数据库分离
- **方言策略**：数据库特定功能通过 Helper 类实现
- **连接初始化**：每个数据库有专门的初始化逻辑

#### 3. 强大的扩展能力
- **Hooks 系统**：`emitFilter` 和 `emitAction` 允许在操作前后注入逻辑
- **事件驱动**：基于事件的架构便于解耦
- **服务继承**：系统集合可通过继承 ItemsService 添加特殊逻辑

#### 4. 查询灵活性
- **Query 对象**：统一的查询描述格式，支持 REST 和 GraphQL
- **AST 中间表示**：Query → AST → SQL 的多级转换，便于优化和扩展
- **权限注入**：`processAst` 在执行前注入权限条件，数据安全有保障

### 5.3 数据流示例

#### REST 查询流程

```
GET /items/articles?fields=id,title,author(name)&filter[status]=published&limit=10

  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  Express Middleware Chain                                        │
│  ├── collectionExists()                                          │
│  └── sanitizeQuery()  ──► 生成 Query 对象                       │
│         {                                                         │
│           fields: ['id', 'title', { author: ['name'] }],        │
│           filter: { status: { _eq: 'published' } },             │
│           limit: 10                                               │
│         }                                                         │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  REST Controller                                                  │
│  const service = new ItemsService('articles', opts)             │
│  const result = service.readByQuery(sanitizedQuery)             │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  ItemsService.readByQuery()                                      │
│  ├── emitFilter('items.query', query)      [Hook]               │
│  ├── getAstFromQuery()  ──►  AST                               │
│  │         {                                                       │
│  │           type: 'root',                                         │
│  │           name: 'articles',                                     │
│  │           query: { ... },                                       │
│  │           children: [                                            │
│  │               { type: 'field', fieldKey: 'id' },               │
│  │               { type: 'field', fieldKey: 'title' },            │
│  │               {                                                  │
│  │                   type: 'm2o',                                  │
│  │                   fieldKey: 'author',                           │
│  │                   children: [{ type: 'field', fieldKey: 'name' }]│
│  │               }                                                  │
│  │           ]                                                      │
│  │         }                                                        │
│  ├── processAst(ast)  ──► 注入权限条件                          │
│  └── runAst(ast)  ──► 执行查询                                   │
│         ├── getDBQuery() ──► Knex Query Builder                 │
│         │       knex('articles')                                   │
│         │         .select('id', 'title', 'author_id')            │
│         │         .where('status', '=', 'published')              │
│         │         .limit(10)                                       │
│         ├── 执行获取 rawItems                                     │
│         ├── 递归处理 author 关联                                 │
│         │       runAst(author AST)                                │
│         │       ┌─────────────────────────────────────┐          │
│         │       │  knex('directus_users')              │          │
│         │       │    .select('name')                   │          │
│         │       │    .whereIn('id', [1, 2, 3, ...])  │          │
│         │       └─────────────────────────────────────┘          │
│         └── mergeWithParentItems()  ──► 合并关联数据            │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  返回结果                                                         │
│  {                                                                │
│    data: [                                                        │
│      { id: 1, title: 'Article 1', author: { name: 'User A' } },│
│      { id: 2, title: 'Article 2', author: { name: 'User B' } },│
│      ...                                                          │
│    ]                                                              │
│  }                                                                │
└─────────────────────────────────────────────────────────────────┘
```

#### GraphQL 等价查询流程

```graphql
query {
  articles(
    filter: { status: { _eq: "published" } }
    limit: 10
  ) {
    id
    title
    author {
      name
    }
  }
}

  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  GraphQLService.execute()                                        │
│  ├── validate()  ──► 验证查询和变量                            │
│  └── execute()  ──► 执行 GraphQL 解析                          │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  resolveQuery()                                                  │
│  ├── collection = 'articles'                                     │
│  ├── parseArgs() ──► { filter: ..., limit: 10 }                │
│  └── getQuery() ──► 生成 Query 对象                            │
│         {                                                         │
│           fields: ['id', 'title', { author: ['name'] }],        │
│           filter: { status: { _eq: 'published' } },             │
│           limit: 10                                               │
│         }                              ┌────────────────────────┐
│  └── gql.read('articles', query) ─────┤  ** 与 REST 相同路径  │
│                                         │  const service =       │
│                                         │    getService(...)     │
│                                         │  service.readByQuery() │
│                                         └────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、文件索引

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| **核心服务** | | |
| ItemsService | `api/src/services/items.ts` | 统一数据访问层核心 |
| PayloadService | `api/src/services/payload.ts` | 数据处理、关系处理 |
| ActivityService | `api/src/services/activity.ts` | 操作审计 |
| RevisionsService | `api/src/services/revisions.ts` | 数据版本控制 |
| UsersService | `api/src/services/users.ts` | 用户服务（继承 ItemsService） |
| FilesService | `api/src/services/files.ts` | 文件服务（继承 ItemsService） |
| | | |
| **REST 控制器** | | |
| Items Controller | `api/src/controllers/items.ts` | 集合 REST 端点 |
| | | |
| **GraphQL 层** | | |
| GraphQLService | `api/src/services/graphql/index.ts` | GraphQL 服务入口 |
| Query Resolver | `api/src/services/graphql/resolvers/query.ts` | 查询解析器 |
| Mutation Resolver | `api/src/services/graphql/resolvers/mutation.ts` | 变更解析器 |
| Schema Generator | `api/src/services/graphql/schema/` | 动态 Schema 生成 |
| | | |
| **服务工厂** | | |
| getService | `api/src/utils/get-service.ts` | 服务选择工厂 |
| | | |
| **数据库层** | | |
| 连接管理 | `api/src/database/index.ts` | Knex 连接、客户端检测 |
| Helpers 工厂 | `api/src/database/helpers/index.ts` | 方言辅助函数工厂 |
| 日期函数 | `api/src/database/helpers/date/` | 各数据库日期处理 |
| 几何函数 | `api/src/database/helpers/geometry/` | 空间数据处理 |
| 模式操作 | `api/src/database/helpers/schema/` | 数据库模式操作 |
| 序列管理 | `api/src/database/helpers/sequence/` | 自增序列管理 |
| 数值处理 | `api/src/database/helpers/number/` | 数值精度处理 |
| | | |
| **查询执行** | | |
| AST 构建 | `api/src/database/get-ast-from-query/` | Query 转 AST |
| AST 执行 | `api/src/database/run-ast/` | AST 转 SQL 并执行 |
| SQL 构建 | `api/src/database/run-ast/lib/get-db-query.ts` | Knex 查询构建器 |
| 过滤器 | `api/src/database/run-ast/lib/apply-query/filter/` | 过滤条件应用 |
| | | |
| **权限处理** | | |
| 权限处理 | `api/src/permissions/modules/process-ast/` | 权限条件注入 |
| 访问验证 | `api/src/permissions/modules/validate-access/` | 权限验证 |

---

## 七、总结

Directus 的架构设计体现了以下核心理念：

### 1. 接口统一
- REST 和 GraphQL 虽然入口不同，但最终都调用相同的 `ItemsService` 方法
- `Query` 对象作为中间表示，抹平了两种 API 的查询语法差异

### 2. 分层清晰
- **API 层**：处理 HTTP/GraphQL 协议
- **Service 层**：业务逻辑、权限、Hooks
- **Query 层**：AST 构建与执行
- **Database 层**：Knex 抽象 + 方言适配

### 3. 数据库无关性
- Knex.js 提供基础 SQL 抽象
- Helper 类处理数据库特定功能
- 连接初始化处理各数据库的特殊需求

### 4. 扩展友好
- 基于事件的 Hooks 系统
- 服务继承机制（系统集合可定制）
- AST 中间层便于查询优化和扩展

这种架构使得 Directus 能够同时支持 REST 和 GraphQL 两种 API，并能适配 8+ 种数据库，同时保持代码的可维护性和扩展性。
