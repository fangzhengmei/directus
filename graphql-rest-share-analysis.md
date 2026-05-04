# Directus GraphQL Schema 动态生成与 REST 共享 ItemsService 机制分析

## 一、概述

Directus 通过一套统一的架构设计，实现了 GraphQL 和 REST API 共享同一套业务逻辑层（ItemsService）。这种设计保证了两种 API 接口在数据操作、权限验证、事件触发等方面的一致性。

核心架构层次：
- **API 层**：REST Controllers + GraphQL Resolvers
- **服务层**：统一的 ItemsService（及专用系统服务）
- **数据层**：Knex.js + 数据库

---

## 二、GraphQL Schema 动态生成机制

### 2.1 核心入口与流程

GraphQL Schema 的动态生成由 `GraphQLService` 的 `getSchema()` 方法触发，最终调用 `generateSchema()` 函数。

**文件位置**：`api/src/services/graphql/schema/index.ts`

```typescript
export async function generateSchema(
    gql: GraphQLService,
    type: 'schema' | 'sdl' = 'schema',
): Promise<GraphQLSchema | string> {
    // 1. 缓存检查
    const key = `${gql.scope}_${type}_${gql.accountability?.role}_${gql.accountability?.user}`;
    const cachedSchema = cache.get(key);
    if (cachedSchema) return cachedSchema;

    // 2. 创建 SchemaComposer
    const schemaComposer = new SchemaComposer<GraphQLParams['contextValue']>();

    // 3. 处理权限过滤
    const schema: Schema = { read, create, update, delete };
    
    // 4. 生成可读/可写类型
    const { ReadCollectionTypes, VersionCollectionTypes } = await getReadableTypes(...);
    const { CreateCollectionTypes, UpdateCollectionTypes, DeleteCollectionTypes } = getWritableTypes(...);

    // 5. 注册 Query 和 Mutation 字段
    schemaComposer.Query.addFields(...);
    schemaComposer.Mutation.addFields(...);

    // 6. 构建并缓存 Schema
    const gqlSchema = schemaComposer.buildSchema();
    cache.set(key, gqlSchema);
    return gqlSchema;
}
```

### 2.2 Schema 的数据来源

Schema 生成依赖于 `SchemaOverview` 对象，该对象包含：
- `collections`：所有集合的元数据（字段、主键、单例标记等）
- `relations`：所有关系定义
- `fields`：每个集合的字段定义

**权限感知的 Schema 过滤**：
```typescript
// 非管理员用户需要根据权限过滤 schema
schema = {
    read: reduceSchema(
        sanitizedSchema,
        await fetchAllowedFieldMap({ accountability: gql.accountability, action: 'read' }, ...),
    ),
    // create, update, delete 同理...
};
```

这意味着不同角色的用户看到的 GraphQL Schema 是不同的，只包含他们有权限访问的集合和字段。

### 2.3 类型生成机制

#### 2.3.1 基础类型生成 (`getTypes`)

**文件位置**：`api/src/services/graphql/schema/get-types.ts`

该函数为每个 collection 生成对应的 GraphQL ObjectType：

```typescript
export function getTypes(
    schemaComposer: SchemaComposer,
    scope: GQLScope,
    schema: Schema,
    inconsistentFields: InconsistentFields,
    action: 'read' | 'create' | 'update' | 'delete',
) {
    const CollectionTypes: Record<string, ObjectTypeComposer> = {};

    for (const collection of Object.values(schema[action].collections)) {
        // 1. 创建 ObjectTypeComposer
        CollectionTypes[collection.collection] = schemaComposer.createObjectTC({
            name: action === 'read' ? collection.collection : `${action}_${collection.collection}`,
            fields: Object.values(collection.fields).reduce((acc, field) => {
                // 2. 将 Directus 字段类型映射到 GraphQL 类型
                let type = getGraphQLType(field.type, field.special);
                
                // 3. 处理非空约束
                if (field.nullable === false && action !== 'update') {
                    type = new GraphQLNonNull(type);
                }

                acc[field.field] = {
                    type,
                    description: field.note,
                    resolve: (obj) => obj[field.field],
                };
                return acc;
            }, {}),
        });
    }

    return { CollectionTypes, VersionTypes };
}
```

#### 2.3.2 可读类型增强 (`getReadableTypes`)

**文件位置**：`api/src/services/graphql/schema/read.ts`

在基础类型之上，添加查询能力：

1. **过滤器类型**：为每个字段创建 `_eq`, `_neq`, `_contains`, `_in` 等过滤操作符
2. **聚合查询**：为数值字段添加 `count`, `sum`, `avg`, `min`, `max` 等聚合函数
3. **排序/分页参数**：`sort`, `limit`, `offset`, `page`, `search`
4. **Resolver 注册**：
   ```typescript
   ReadCollectionTypes[collection.collection]!.addResolver({
       name: collection.collection,
       type: new GraphQLList(new GraphQLNonNull(ReadCollectionTypes[collection.collection]!.getType())),
       args: {
           filter: ReadableCollectionFilterTypes[collection.collection]!,
           sort: new GraphQLList(GraphQLString),
           limit: GraphQLInt,
           offset: GraphQLInt,
           page: GraphQLInt,
           search: GraphQLString,
       },
       resolve: dedupeRelationalResolver(({ info }) => resolveQuery(gql, info)),
   });
   ```

#### 2.3.3 关系字段处理

关系字段通过遍历 `schema.relations` 动态添加：

```typescript
for (const relation of schema[action].relations) {
    if (relation.related_collection) {
        // M2O (Many-to-One) 关系
        CollectionTypes[relation.collection]?.addFields({
            [relation.field]: {
                type: CollectionTypes[relation.related_collection]!,
                resolve: (obj, _, __, info) => {
                    return obj[info?.path?.key ?? relation.field];
                },
            },
        });

        // O2M (One-to-Many) 反向关系
        if (relation.meta?.one_field) {
            CollectionTypes[relation.related_collection]?.addFields({
                [relation.meta.one_field]: {
                    type: [CollectionTypes[relation.collection]!],
                    resolve: (obj, _, __, info) => {
                        return obj[info?.path?.key ?? relation.meta!.one_field];
                    },
                },
            });
        }
    } else if (relation.meta?.one_allowed_collections && action === 'read') {
        // A2O (Any-to-One) 关系 - 使用 GraphQL Union Type
        CollectionTypes[relation.collection]?.addFields({
            [relation.field]: {
                type: new GraphQLUnionType({
                    name: `${relation.collection}_${relation.field}_union`,
                    types: relation.meta.one_allowed_collections.map(
                        (collection) => CollectionTypes[collection]!.getType()
                    ),
                    resolveType(_value, context, info) {
                        // 运行时根据 one_collection_field 判断实际类型
                        const collection = parent[relation.meta!.one_collection_field!]!;
                        return CollectionTypes[collection]!.getType().name;
                    },
                }),
                resolve: (obj, _, __, info) => obj[info?.path?.key ?? relation.field],
            },
        });
    }
}
```

### 2.4 Schema 缓存策略

Schema 生成是一个相对耗时的操作，Directus 使用了多层缓存：

1. **内存缓存**：使用 `lru-cache` 缓存生成的 Schema
2. **缓存 Key**：基于 `scope + type + role + user` 生成
3. **并发控制**：使用 `Semaphore` 限制同时生成 Schema 的并发数（默认 5）

```typescript
const semaphore = new Semaphore((env['GRAPHQL_SCHEMA_GENERATION_MAX_CONCURRENT'] as number) ?? 5);

export async function generateSchema(gql: GraphQLService, type: 'schema' | 'sdl') {
    const key = `${gql.scope}_${type}_${gql.accountability?.role}_${gql.accountability?.user}`;
    
    const cachedSchema = cache.get(key);
    if (cachedSchema) return cachedSchema;

    return semaphore.runExclusive(async () => {
        // 双重检查锁
        const cachedSchema = cache.get(key);
        if (cachedSchema) return cachedSchema;
        
        // ... 生成逻辑
        
        cache.set(key, gqlSchema);
        return gqlSchema;
    });
}
```

---

## 三、REST 如何使用 ItemsService

### 3.1 REST 控制器结构

**文件位置**：`api/src/controllers/items.ts`

REST 控制器是 Express.js 路由处理器，直接实例化并调用 ItemsService：

```typescript
const router = express.Router();

// POST - 创建
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
}), respond);

// GET - 查询列表
router.get('/:collection', collectionExists, asyncHandler(async (req, res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    const result = await service.readByQuery(req.sanitizedQuery);
    res.locals['payload'] = { data: result };
}), respond);

// GET/:pk - 查询单条
router.get('/:collection/:pk', collectionExists, asyncHandler(async (req, res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    const result = await service.readOne(req.params['pk']!, req.sanitizedQuery);
    res.locals['payload'] = { data: result };
}), respond);

// PATCH - 更新
router.patch('/:collection', collectionExists, validateBatch('update'), asyncHandler(async (req, res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    if (Array.isArray(req.body)) {
        keys = await service.updateBatch(req.body);
    } else if (req.body.keys) {
        keys = await service.updateMany(req.body.keys, req.body.data);
    } else {
        keys = await service.updateByQuery(sanitizedQuery, req.body.data);
    }
}), respond);

// DELETE - 删除
router.delete('/:collection/:pk', collectionExists, asyncHandler(async (req, _res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    await service.deleteOne(req.params['pk']!);
}), respond);
```

### 3.2 REST 与 ItemsService 方法映射

| HTTP 方法 | REST 端点 | ItemsService 方法 |
|-----------|----------|------------------|
| POST | `/items/:collection` | `createOne` / `createMany` |
| GET | `/items/:collection` | `readByQuery` / `readSingleton` |
| GET | `/items/:collection/:pk` | `readOne` |
| SEARCH | `/items/:collection` | `readMany` (by keys) |
| PATCH | `/items/:collection` | `updateBatch` / `updateMany` / `updateByQuery` |
| PATCH | `/items/:collection/:pk` | `updateOne` |
| DELETE | `/items/:collection` | `deleteMany` / `deleteByQuery` |
| DELETE | `/items/:collection/:pk` | `deleteOne` |

---

## 四、GraphQL 如何使用 ItemsService

### 4.1 GraphQLService 封装

**文件位置**：`api/src/services/graphql/index.ts`

GraphQLService 是 GraphQL 与业务逻辑层之间的桥梁，它通过 `getService()` 函数获取正确的服务实例：

```typescript
export class GraphQLService {
    accountability: Accountability | null;
    knex: Knex;
    schema: SchemaOverview;
    scope: GQLScope;

    /**
     * 执行读取操作 - 根据 collection 类型选择正确的服务
     */
    async read(collection: string, query: Query, id?: PrimaryKey): Promise<Partial<Item>> {
        const service = getService(collection, {
            knex: this.knex,
            accountability: this.accountability,
            schema: this.schema,
        });

        // 单例集合特殊处理
        if (this.schema.collections[collection]!.singleton)
            return await service.readSingleton(query, { stripNonRequested: false });

        // 单条查询
        if (id) return await service.readOne(id, query, { stripNonRequested: false });

        // 列表查询
        return await service.readByQuery(query, { stripNonRequested: false });
    }

    /**
     * 单例集合的 upsert 操作
     */
    async upsertSingleton(
        collection: string,
        body: Record<string, any> | Record<string, any>[],
        query: Query,
    ): Promise<Partial<Item> | boolean> {
        const service = getService(collection, {
            knex: this.knex,
            accountability: this.accountability,
            schema: this.schema,
        });

        await service.upsertSingleton(body);
        
        if ((query.fields || []).length > 0) {
            return await service.readSingleton(query);
        }
        return true;
    }
}
```

### 4.2 getService 函数：服务路由

**文件位置**：`api/src/utils/get-service.ts`

这个函数是实现"共享同一套 ItemsService"的关键，它根据 collection 名称路由到不同的服务实现：

```typescript
export function getService(collection: string, opts: AbstractServiceOptions): ItemsService {
    switch (collection) {
        // 系统集合使用专用服务（都继承自 ItemsService）
        case 'directus_users':
            return new UsersService(opts);
        case 'directus_files':
            return new FilesService(opts);
        case 'directus_roles':
            return new RolesService(opts);
        case 'directus_permissions':
            return new PermissionsService(opts);
        case 'directus_activity':
            return new ActivityService(opts);
        case 'directus_revisions':
            return new RevisionsService(opts);
        // ... 更多系统服务

        default:
            // 其他系统集合禁止直接访问
            if (collection.startsWith('directus_')) throw new ForbiddenError();

            // 用户自定义集合使用基础 ItemsService
            return new ItemsService(collection, opts);
    }
}
```

**设计要点**：
1. 所有系统服务（UsersService, FilesService 等）都继承自 ItemsService
2. 它们可以重写特定方法来添加业务逻辑（如 UsersService 处理密码哈希）
3. 这保证了多态性：GraphQL 和 REST 都通过统一的接口调用，但实际执行的可能是专用逻辑

### 4.3 GraphQL Resolver 流程

**文件位置**：`api/src/services/graphql/resolvers/query.ts`

Resolver 的核心职责是将 GraphQL 查询转换为 Directus 的 Query 结构，然后调用 GraphQLService：

```typescript
export async function resolveQuery(gql: GraphQLService, info: GraphQLResolveInfo): Promise<Partial<Item> | null> {
    // 1. 解析 collection 名称
    let collection = info.fieldName;
    if (gql.scope === 'system') collection = `directus_${collection}`;

    // 2. 处理特殊查询类型
    const isAggregate = collection.endsWith('_aggregated');
    if (isAggregate) {
        collection = collection.slice(0, -11);
    } else if (collection.endsWith('_by_id')) {
        collection = collection.slice(0, -6);
    }

    // 3. 解析 GraphQL 参数
    const args: Record<string, any> = parseArgs(info.fieldNodes[0]!.arguments || [], info.variableValues);

    // 4. 将 GraphQL 选择集转换为 Directus Query
    let query: Query;
    if (isAggregate) {
        query = await getAggregateQuery(args, selections, gql.schema, gql.accountability, collection);
    } else {
        query = await getQuery(args, gql.schema, selections, info.variableValues, gql.accountability, collection);
    }

    // 5. 调用统一的读取方法（最终使用 ItemsService）
    const result = await gql.read(collection, query, args['id']);

    return result;
}
```

### 4.4 查询转换：GraphQL AST → Directus Query

**文件位置**：`api/src/services/graphql/schema/parse-query.ts`

这是 GraphQL 与 REST 共享同一套查询逻辑的关键转换层：

```typescript
export async function getQuery(
    args: Record<string, any>,
    schema: SchemaOverview,
    selections: readonly SelectionNode[],
    variableValues: { [variable: string]: unknown },
    accountability: Accountability | null,
    collection: string,
): Promise<Query> {
    const query: Query = {};

    // 1. 字段选择 - 从 GraphQL selections 提取
    query.fields = extractFields(selections, schema, collection);

    // 2. 过滤参数 - GraphQL filter 参数 → Directus filter
    if (args['filter']) {
        query.filter = args['filter'];
    }

    // 3. 排序 - GraphQL sort 参数 → Directus sort
    if (args['sort']) {
        query.sort = args['sort'];
    }

    // 4. 分页 - limit/offset/page
    if (args['limit'] !== undefined) query.limit = args['limit'];
    if (args['offset'] !== undefined) query.offset = args['offset'];
    if (args['page'] !== undefined) query.page = args['page'];

    // 5. 搜索
    if (args['search']) query.search = args['search'];

    // 6. 深度解析嵌套关系字段
    // 递归处理 selections 中的关系字段，构建嵌套的 Query 结构

    return query;
}
```

---

## 五、共享机制深度分析

### 5.1 架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        客户端请求                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
┌───────────────────────┐         ┌─────────────────────────────┐
│    REST Controller    │         │      GraphQL Resolver       │
│  (items.ts)           │         │  (query.ts, mutation.ts)    │
└───────────┬───────────┘         └─────────────┬───────────────┘
            │                                     │
            │  直接实例化                          │  通过 getService()
            │  ItemsService                       │  获取服务实例
            ▼                                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                    getService() 服务路由层                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ UsersService│  │ FilesService│  │     ItemsService        │ │
│  │ (继承自...) │  │ (继承自...) │  │  (用户自定义集合基础服务)   │ │
│  └─────────────┘  └─────────────┘  └─────────────────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    统一的业务逻辑层                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    ItemsService 核心                       │   │
│  │  - createOne/createMany    (创建)                         │   │
│  │  - readOne/readByQuery     (读取)                         │   │
│  │  - updateOne/updateMany    (更新)                         │   │
│  │  - deleteOne/deleteMany    (删除)                         │   │
│  │  - readSingleton/upsertSingleton (单例)                    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                      │
│  ┌─────────────────────────┼─────────────────────────────────┐  │
│  ▼                         ▼                                 ▼  │
│  ┌─────────────┐  ┌─────────────────┐  ┌───────────────────┐ │
│  │  权限检查    │  │   事件系统       │  │   活动/修订追踪     │ │
│  │ validateAccess│  │   emitter      │  │  ActivityService  │ │
│  │ processPayload│  │ (Hooks系统)    │  │  RevisionsService │ │
│  │  processAst  │  │                 │  │                   │ │
│  └─────────────┘  └─────────────────┘  └───────────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        数据访问层                                  │
│  ┌─────────────┐  ┌─────────────────┐  ┌───────────────────┐   │
│  │   Knex.js   │  │  getAstFromQuery │  │     runAst        │   │
│  │  (Query Builder) │  (AST 构建)    │  │  (AST 执行)       │   │
│  └─────────────┘  └─────────────────┘  └───────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 核心共享点

#### 1. 统一的 Query 结构

无论是 REST 还是 GraphQL，最终都转换为相同的 `Query` 类型传递给 ItemsService：

```typescript
// 两者使用相同的 Query 接口
interface Query {
    fields?: string[];
    filter?: Record<string, any>;
    sort?: string[];
    limit?: number;
    offset?: number;
    page?: number;
    search?: string;
    group?: string[];
    aggregate?: Record<string, string[]>;
    deep?: Record<string, Query>;  // 嵌套关系查询
}
```

**REST 路径**：
```
HTTP 请求 → Express 中间件 → sanitizeQuery() → Query 对象 → ItemsService.readByQuery(query)
```

**GraphQL 路径**：
```
GraphQL 查询 → parseArgs() + parseQuery() → Query 对象 → GraphQLService.read() → ItemsService.readByQuery(query)
```

#### 2. 相同的服务实例化参数

REST 和 GraphQL 传递给 ItemsService 的参数完全一致：

```typescript
// REST Controller
const service = new ItemsService(req.collection, {
    accountability: req.accountability,  // 用户身份、角色、IP 等
    schema: req.schema,                    // SchemaOverview
    knex: req.knex,                        // 数据库连接（可选，默认使用全局）
});

// GraphQLService (通过 getService)
const service = getService(collection, {
    accountability: this.accountability,   // 相同的 accountability
    schema: this.schema,                    // 相同的 schema
    knex: this.knex,                        // 相同的数据库连接
});
```

#### 3. 统一的权限检查

ItemsService 内部的权限检查对 REST 和 GraphQL 完全透明：

```typescript
// ItemsService.readByQuery 中的权限检查
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
    // 1. 构建 AST
    let ast = await getAstFromQuery(
        { collection: this.collection, query, accountability: this.accountability },
        { schema: this.schema, knex: this.knex },
    );

    // 2. 权限处理 AST
    ast = await processAst(
        { ast, action: 'read', accountability: this.accountability },
        { knex: this.knex, schema: this.schema },
    );

    // 3. 执行
    const records = await runAst(ast, this.schema, this.accountability, { ... });
    return records;
}
```

关键的权限模块：
- `validateAccess`：验证用户对特定字段/记录的操作权限
- `processPayload`：在创建/更新前处理字段预设和权限过滤
- `processAst`：在查询 AST 中注入权限条件

#### 4. 统一的事件系统

所有数据操作都会触发相同的事件，供 Hooks 系统使用：

```typescript
// ItemsService.createOne 中的事件触发
const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
    // Filter Hook (操作前)
    const payloadAfterHooks = await emitter.emitFilter(
        this.eventScope === 'items'
            ? ['items.create', `${this.collection}.items.create`]
            : `${this.eventScope}.create`,
        payload,
        { collection: this.collection },
        { database: trx, schema: this.schema, accountability: this.accountability },
    );
    // ... 执行创建 ...
});

// Action Hook (操作后)
emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
```

事件命名规范：
- 自定义集合：`items.create`, `{collection}.items.create`
- 系统集合：`users.create`, `files.create` 等

#### 5. 统一的活动/修订追踪

数据变更的审计追踪也是共享的：

```typescript
// ItemsService 中创建活动记录
if (this.accountability && this.schema.collections[this.collection]!.accountability !== null) {
    const activityService = new ActivityService({ knex: trx, schema: this.schema });
    
    const activity = await activityService.createOne({
        action: Action.CREATE,  // or UPDATE, DELETE
        user: this.accountability!.user,
        collection: this.collection,
        ip: this.accountability!.ip,
        user_agent: this.accountability!.userAgent,
        origin: this.accountability!.origin,
        item: primaryKey,
    });

    // 如果启用了完整修订追踪
    if (this.schema.collections[this.collection]!.accountability === 'all') {
        const revisionsService = new RevisionsService({ knex: trx, schema: this.schema });
        await revisionsService.createOne({
            activity: activity,
            collection: this.collection,
            item: primaryKey,
            data: revisionPayload,
            delta: revisionPayload,
        });
    }
}
```

### 5.3 系统服务的多态性

Directus 为系统集合提供了专用服务，它们都继承自 ItemsService，可以重写特定方法：

```
                    ┌──────────────────┐
                    │   ItemsService   │  ← 基础服务
                    │  (核心 CRUD)      │
                    └─────────┬────────┘
                              │
          ┌───────────┬───────┼───────┬───────────┐
          │           │       │       │           │
          ▼           ▼       ▼       ▼           ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
    │UsersService│ │FilesService│ │RolesService│ │...更多    │
    │  (密码处理)  │ │ (文件处理)  │ │ (权限关联)  │ │ 系统服务   │
    └──────────┘ └──────────┘ └──────────┘ └──────────┘
```

**示例：UsersService 重写 createOne**

```typescript
export class UsersService extends ItemsService {
    async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
        // 密码特殊处理：在创建前进行哈希
        if (data.password) {
            data.password = await hashPassword(data.password);
        }
        
        // 调用父类方法（共享的创建逻辑）
        return super.createOne(data, opts);
    }
    
    // 同样重写 updateOne 处理密码更新
    async updateOne(key: PrimaryKey, data: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey> {
        if (data.password) {
            data.password = await hashPassword(data.password);
        }
        return super.updateOne(key, data, opts);
    }
}
```

这种设计的好处：
1. **REST 和 GraphQL 自动受益**：无需在两个 API 层分别实现密码哈希逻辑
2. **保持接口一致性**：`getService()` 返回的都是 ItemsService 类型
3. **易于扩展**：添加新的系统集合只需创建新的服务子类

---

## 六、关键代码位置汇总

| 功能模块 | 文件路径 |
|---------|---------|
| GraphQL 服务入口 | `api/src/services/graphql/index.ts` |
| Schema 生成核心 | `api/src/services/graphql/schema/index.ts` |
| 类型生成 | `api/src/services/graphql/schema/get-types.ts` |
| 可读类型与 Resolver | `api/src/services/graphql/schema/read.ts` |
| 可写类型 | `api/src/services/graphql/schema/write.ts` |
| 查询 Resolver | `api/src/services/graphql/resolvers/query.ts` |
| 变更 Resolver | `api/src/services/graphql/resolvers/mutation.ts` |
| 查询转换 | `api/src/services/graphql/schema/parse-query.ts` |
| REST 控制器 | `api/src/controllers/items.ts` |
| ItemsService 核心 | `api/src/services/items.ts` |
| 服务路由 | `api/src/utils/get-service.ts` |
| GraphQL 控制器 | `api/src/controllers/graphql.ts` |

---

## 七、总结

Directus 通过以下设计实现了 GraphQL 和 REST 共享同一套业务逻辑：

1. **统一的服务层抽象**：`ItemsService` 封装了所有数据操作的核心逻辑
2. **服务路由机制**：`getService()` 根据 collection 类型返回正确的服务实例
3. **标准化的 Query 结构**：REST 和 GraphQL 都转换为相同的 `Query` 对象
4. **共享的上下文参数**：`accountability`, `schema`, `knex` 在两个 API 层完全一致
5. **内部化的权限和事件**：权限检查、事件触发、审计追踪都在 ItemsService 内部完成
6. **多态的系统服务**：系统集合通过继承 ItemsService 来扩展专用逻辑

这种架构的优势：
- **一致性**：REST 和 GraphQL 的行为完全一致
- **可维护性**：业务逻辑只需修改一次
- **可扩展性**：添加新功能只需在服务层实现
- **安全性**：权限检查集中管理，不易遗漏
