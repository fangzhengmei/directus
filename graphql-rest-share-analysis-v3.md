# Directus GraphQL Schema 动态生成与 REST 共享 ItemsService 机制分析（事实纠偏版）

> **版本说明**：本报告是 v2 版本的事实纠偏版，修正了以下关键错误：
> 1. ItemsService 实例化参数的错误描述
> 2. REST 查询入口对 sanitizedQuery 的错误表述
>
> 所有修正均基于实际代码证据。

---

## 一、概述

Directus 通过一套统一的架构设计，实现了 GraphQL 和 REST API 共享同一套业务逻辑层（ItemsService）。这种设计保证了两种 API 接口在数据操作、权限验证、事件触发等方面的一致性。

本文档深入分析以下核心机制：
1. GraphQL Schema 的动态生成（含缓存、Scope 过滤、System Resolver 注入）
2. Mutation 写操作复用 ItemsService 的完整链路
3. Parse-query 中 alias、deep 参数处理与 M2A 过滤重写
4. GraphQL 与 REST 共享边界的完整说明（**事实纠偏**）

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

    // 2. 并发控制（Semaphore）
    return semaphore.runExclusive(async () => {
        // 双重检查锁
        const cachedSchema = cache.get(key);
        if (cachedSchema) return cachedSchema;

        // 3. 创建 SchemaComposer
        const schemaComposer = new SchemaComposer<GraphQLParams['contextValue']>();

        // 4. 处理权限过滤（read/create/update/delete 四种操作的独立 schema）
        let schema: Schema;
        const sanitizedSchema = sanitizeGraphqlSchema(gql.schema);

        if (!gql.accountability || gql.accountability.admin) {
            schema = {
                read: sanitizedSchema,
                create: sanitizedSchema,
                update: sanitizedSchema,
                delete: sanitizedSchema,
            };
        } else {
            schema = {
                read: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap({ action: 'read' }, ...)),
                create: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap({ action: 'create' }, ...)),
                update: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap({ action: 'update' }, ...)),
                delete: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap({ action: 'delete' }, ...)),
            };
        }

        // 5. 生成可读/可写类型
        const { ReadCollectionTypes, VersionCollectionTypes } = await getReadableTypes(...);
        const { CreateCollectionTypes, UpdateCollectionTypes, DeleteCollectionTypes } = getWritableTypes(...);

        // 6. Scope 过滤与 System Resolver 注入
        const scopeFilter = (collection: SchemaOverview['collections'][string]) => {
            if (gql.scope === 'items' && isSystemCollection(collection.collection)) return false;

            if (gql.scope === 'system') {
                if (isSystemCollection(collection.collection) === false) return false;
                if (SYSTEM_DENY_LIST.includes(collection.collection)) return false;
            }

            return true;
        };

        if (gql.scope === 'system') {
            injectSystemResolvers(gql, schemaComposer, CollectionTypes, schema);
        }

        // 7. 注册 Query 和 Mutation 字段
        // ... 详见后续章节

        // 8. 构建并缓存 Schema
        const gqlSchema = schemaComposer.buildSchema();
        cache.set(key, gqlSchema);
        return gqlSchema;
    });
}
```

### 2.2 Schema 缓存机制

**文件位置**：`api/src/services/graphql/schema-cache.ts`

Schema 缓存使用 `mnemonist` 库的 `LRUMap` 实现：

```typescript
import { LRUMap } from 'mnemonist';
import { useBus } from '../../bus/index.js';

const env = useEnv();
const bus = useBus();

// 缓存容量由环境变量控制，默认 100
export const cache = new LRUMap<string, GraphQLSchema | string>(
    Number(env['GRAPHQL_SCHEMA_CACHE_CAPACITY'] ?? 100)
);

// 监听 schema 变更事件，自动清除缓存
bus.subscribe('schemaChanged', () => {
    cache.clear();
});
```

**缓存 Key 构成**：
```typescript
const key = `${gql.scope}_${type}_${gql.accountability?.role}_${gql.accountability?.user}`;
```

Key 包含四个维度：
1. `scope`：`items` 或 `system`
2. `type`：`schema` 或 `sdl`
3. `role`：用户角色 ID（用于权限过滤的 schema 差异化）
4. `user`：用户 ID（用于更细粒度的缓存）

**并发控制**：
```typescript
const semaphore = new Semaphore(
    (env['GRAPHQL_SCHEMA_GENERATION_MAX_CONCURRENT'] as number) ?? 5
);
```
使用 Semaphore 限制同时生成 Schema 的并发数（默认 5），防止高并发下的性能问题。

### 2.3 Scope 过滤机制

Scope 是 GraphQL 接口的重要概念，用于区分两类不同的访问端点：

| Scope | 端点路径 | 包含的集合 | 主要用途 |
|-------|---------|-----------|---------|
| `items` | `/graphql` | 用户自定义集合 | 业务数据操作 |
| `system` | `/graphql/system` | 系统集合（排除 `SYSTEM_DENY_LIST`） | 系统管理操作 |

**Scope 过滤实现**：
```typescript
const scopeFilter = (collection: SchemaOverview['collections'][string]) => {
    // items scope：只包含非系统集合
    if (gql.scope === 'items' && isSystemCollection(collection.collection)) return false;

    // system scope：只包含系统集合，且排除 SYSTEM_DENY_LIST
    if (gql.scope === 'system') {
        if (isSystemCollection(collection.collection) === false) return false;
        if (SYSTEM_DENY_LIST.includes(collection.collection)) return false;
    }

    return true;
};
```

**SYSTEM_DENY_LIST 定义**：
```typescript
export const SYSTEM_DENY_LIST = [
    'directus_collections',
    'directus_fields',
    'directus_relations',
    'directus_migrations',
    'directus_sessions',
    'directus_extensions',
];
```
这些集合通过专门的 System Resolvers 访问，而不是通用的 items 模式。

### 2.4 System Resolver 注入

当 `scope === 'system'` 时，会调用 `injectSystemResolvers()` 注入系统专用的 resolvers。

**文件位置**：`api/src/services/graphql/resolvers/system.ts`

```typescript
export function injectSystemResolvers(
    gql: GraphQLService,
    schemaComposer: SchemaComposer<GraphQLParams['contextValue']>,
    { CreateCollectionTypes, ReadCollectionTypes, UpdateCollectionTypes }: CollectionTypes,
    schema: Schema,
): SchemaComposer<any> {
    // 1. 全局 Resolvers（server_info, server_ping, server_health 等）
    globalResolvers(gql, schemaComposer);

    // 2. Server Info 类型定义
    const ServerInfo = schemaComposer.createObjectTC({
        name: 'server_info',
        fields: {
            project: { type: ... },
            rateLimit: { type: ... },
            websocket: { type: ... },
            queryLimit: { type: ... },
        },
    });

    // 3. Query 字段注册
    schemaComposer.Query.addFields({
        server_ping: { type: GraphQLString, resolve: () => 'pong' },
        server_info: { type: ServerInfo, resolve: ... },
        server_health: { type: GraphQLJSON, resolve: ... },
    });

    // 4. 元数据查询（collections, fields, relations）
    // 5. 当前用户相关（users_me, permissions_me, roles_me）
    // 6. 系统专用 Mutation（update_users_me, import_file, users_invite）

    return schemaComposer;
}
```

---

## 三、Query 解析与转换机制

### 3.1 Parse-Query 核心逻辑

**文件位置**：`api/src/services/graphql/schema/parse-query.ts`

这是 GraphQL 查询转换为 Directus Query 的核心模块：

```typescript
export async function getQuery(
    rawQuery: Query,
    schema: SchemaOverview,
    selections: readonly SelectionNode[],
    variableValues: GraphQLResolveInfo['variableValues'],
    accountability?: Accountability | null,
    collection?: string,
): Promise<Query> {
    // 1. 基础 sanitize（与 REST 共享的工具函数）
    const query: Query = await sanitizeQuery(rawQuery, schema, accountability);

    // 2. 解析别名
    const parseAliases = (selections: readonly SelectionNode[]) => {
        const aliases: Record<string, string> = {};
        for (const selection of selections) {
            if (selection.kind !== 'Field') continue;
            if (selection.alias?.value) {
                aliases[selection.alias.value] = selection.name.value;
            }
        }
        return aliases;
    };

    // 3. 解析字段（核心递归函数）
    const parseFields = async (
        selections: readonly SelectionNode[],
        parent?: string,
        currentCollection?: string,
        parentFieldName?: string,
    ): Promise<string[]> => {
        const fields: string[] = [];

        for (let selection of selections) {
            if ((selection.kind === 'Field' || selection.kind === 'InlineFragment') !== true) continue;

            // ... 字段解析逻辑

            // 关键：嵌套字段参数写入 deep 参数
            if (selection.kind === 'Field' && selection.arguments && selection.arguments.length > 0) {
                if (!query.deep) query.deep = {};

                const args: Record<string, any> = parseArgs(selection.arguments, variableValues);
                const path = (currentAlias ?? current).replaceAll(':', '__');

                set(
                    query.deep,
                    path,
                    merge(
                        {},
                        get(query.deep, path),
                        mapKeys(await sanitizeQuery(args, schema, accountability), (_value, key) => `_${key}`),
                    ),
                );
            }
        }

        return uniq(fields);
    };

    // 4. 执行解析
    query.alias = parseAliases(selections);
    query.fields = await parseFields(selections, undefined, collection);

    // 5. 函数替换
    if (query.filter) query.filter = replaceFuncs(query.filter);
    query.deep = replaceFuncs(query.deep as any) as any;

    // 6. M2A 过滤重写
    if (collection) {
        if (query.filter) {
            query.filter = filterReplaceM2A(query.filter, collection, schema, { aliasMap: query.alias });
        }
        query.deep = filterReplaceM2ADeep(query.deep, collection, schema, { aliasMap: query.alias });
    }

    validateQuery(query);
    return query;
}
```

### 3.2 Alias 与 Deep 参数处理

**Alias 机制**：
GraphQL 支持字段别名，Directus 通过 `query.alias` 和 `query.deep._alias` 处理：

```graphql
query {
    articles {
        originalAuthor: author {  # 别名
            name
        }
    }
}
```

转换后的 Query：
```typescript
{
    fields: ['author.name'],
    alias: { 'originalAuthor': 'author' },
    deep: {
        author: {
            _alias: { 'originalAuthor': 'author' }
        }
    }
}
```

**Deep 参数结构**：
`query.deep` 用于嵌套关系的查询参数，格式为：
```typescript
{
    "relation_field": {
        "_filter": { ... },
        "_sort": [...],
        "_limit": 10,
        "_alias": { "alias_name": "actual_field" }
    },
    "a2o_field__target_collection": {  // M2A 使用 __ 分隔
        "_filter": { ... }
    }
}
```

### 3.3 M2A 过滤重写机制

**文件位置**：`api/src/services/graphql/utils/filter-replace-m2a.ts`

由于 GraphQL 不支持在 filter 中使用 `field:collection` 语法（冒号有特殊含义），Directus 使用 `field__collection` 格式，然后在解析时重写。

```typescript
export function filterReplaceM2A(
    filter_arg: Filter,
    collection: string,
    schema: SchemaOverview,
    options?: { aliasMap?: Query['alias'] },
): any {
    const filter: any = filter_arg;

    for (const key in filter) {
        const parts = key.split('__');
        let field = parts[0];
        const any_collection = parts[1];  // __ 后面的是目标集合

        if (!field) continue;

        // 应用别名映射
        field = options?.aliasMap?.[field] ?? field;

        const relation = getRelation(schema.relations, collection, field);
        const type = relation ? getRelationType({ relation, collection, field, useA2O: true }) : null;

        if (type === 'a2o' && any_collection && relation.meta?.one_allowed_collections?.includes(any_collection)) {
            // 关键：M2A 重写 - 将 field__collection 转换为 field:collection
            filter[`${field}:${any_collection}`] = filterReplaceM2A(filter[key], any_collection, schema, options);
            delete filter[key];
        } else if (Array.isArray(filter[key])) {
            // 递归处理数组（_and, _or）
            filter[key] = filter[key].map((item) => filterReplaceM2A(item, collection, schema, options));
        } else if (typeof filter[key] === 'object') {
            // 递归处理嵌套对象
            filter[key] = filterReplaceM2A(filter[key], collection, schema, options);
        }
    }

    return filter;
}
```

**示例**：

GraphQL 查询（使用 `field__collection` 格式）：
```graphql
query {
    notes {
        related__articles: related {  # M2A 关系，目标集合是 articles
            ... on articles {
                title
            }
        }
    }
}
```

转换后的 Query（重写为 `field:collection` 格式）：
```typescript
{
    fields: ['related:articles.title'],
    alias: { 'related__articles': 'related' },
    deep: {
        'related:articles': {
            _alias: { 'related__articles': 'related' }
        }
    }
}
```

---

## 四、Mutation 写操作复用 ItemsService 链路

### 4.1 Mutation Resolver 核心逻辑

**文件位置**：`api/src/services/graphql/resolvers/mutation.ts`

```typescript
export async function resolveMutation(
    gql: GraphQLService,
    args: Record<string, any>,
    info: GraphQLResolveInfo,
): Promise<Partial<Item> | boolean | undefined> {
    // 1. 从字段名解析 action 和 collection
    const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
    let collection = info.fieldName.substring(action.length + 1);
    if (gql.scope === 'system') collection = `directus_${collection}`;

    // 2. 解析返回字段的 Query（用于读取创建/更新后的结果）
    const selections = replaceFragmentsInSelections(
        info.fieldNodes[0]?.selectionSet?.selections, info.fragments
    );
    const query = await getQuery(
        args, gql.schema, selections || [], info.variableValues, gql.accountability, collection
    );

    // 3. 判断操作类型
    const singleton =
        collection.endsWith('_batch') === false &&
        collection.endsWith('_items') === false &&
        collection.endsWith('_item') === false &&
        collection in gql.schema.collections;

    const single = collection.endsWith('_items') === false && collection.endsWith('_batch') === false;
    const batchUpdate = action === 'update' && collection.endsWith('_batch');

    // 4. 清理 collection 名称后缀
    if (collection.endsWith('_batch')) collection = collection.slice(0, -6);
    if (collection.endsWith('_items')) collection = collection.slice(0, -6);
    if (collection.endsWith('_item')) collection = collection.slice(0, -5);

    // 5. 单例集合的 update 特殊处理
    if (singleton && action === 'update') {
        return await gql.upsertSingleton(collection, args['data'], query);
    }

    // 6. 获取服务实例
    const service = getService(collection, {
        knex: gql.knex,
        accountability: gql.accountability,
        schema: gql.schema,
    });

    const hasQuery = (query.fields || []).length > 0;

    try {
        // 7. 根据操作类型调用不同的服务方法

        if (single) {
            // 单条操作
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
        } else {
            // 批量操作
            if (action === 'create') {
                const keys = await service.createMany(args['data']);
                return hasQuery ? await service.readMany(keys, query) : true;
            }

            if (action === 'update') {
                const keys: PrimaryKey[] = [];

                if (batchUpdate) {
                    keys.push(...(await service.updateBatch(args['data'])));
                } else {
                    keys.push(...(await service.updateMany(args['ids'], args['data'])));
                }

                return hasQuery ? await service.readMany(keys, query) : true;
            }

            if (action === 'delete') {
                const keys = await service.deleteMany(args['ids']);
                return { ids: keys };
            }
        }

        return undefined;
    } catch (err: any) {
        return formatError(err);
    }
}
```

### 4.2 Mutation 字段命名规范

| 字段名模式 | 操作类型 | 服务方法 | 参数 |
|-----------|---------|---------|------|
| `create_{collection}_item` | 创建单条 | `service.createOne(data)` | `data` |
| `create_{collection}_items` | 创建多条 | `service.createMany(data[])` | `data` (数组) |
| `update_{collection}_item` | 更新单条 | `service.updateOne(id, data)` | `id`, `data` |
| `update_{collection}_items` | 更新多条 | `service.updateMany(ids[], data)` | `ids`, `data` |
| `update_{collection}_batch` | 批量更新 | `service.updateBatch(data[])` | `data` (含主键的数组) |
| `update_{collection}` | 更新单例 | `gql.upsertSingleton()` | `data` |
| `delete_{collection}_item` | 删除单条 | `service.deleteOne(id)` | `id` |
| `delete_{collection}_items` | 删除多条 | `service.deleteMany(ids[])` | `ids` |

### 4.3 GraphQLService 中的单例操作

**文件位置**：`api/src/services/graphql/index.ts`

```typescript
export class GraphQLService {
    accountability: Accountability | null;
    knex: Knex;
    schema: SchemaOverview;
    scope: GQLScope;

    constructor(options: AbstractServiceOptions & { scope: GQLScope }) {
        this.accountability = options?.accountability || null;
        this.knex = options?.knex || getDatabase();  // 显式获取或使用默认数据库
        this.schema = options.schema;
        this.scope = options.scope;
    }

    /**
     * 执行读取操作
     */
    async read(collection: string, query: Query, id?: PrimaryKey): Promise<Partial<Item>> {
        const service = getService(collection, {
            knex: this.knex,           // 显式传递 knex
            accountability: this.accountability,
            schema: this.schema,
        });

        if (this.schema.collections[collection]!.singleton)
            return await service.readSingleton(query, { stripNonRequested: false });

        if (id) return await service.readOne(id, query, { stripNonRequested: false });

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
            knex: this.knex,           // 显式传递 knex
            accountability: this.accountability,
            schema: this.schema,
        });

        try {
            await service.upsertSingleton(body);

            if ((query.fields || []).length > 0) {
                const result = await service.readSingleton(query);
                return result;
            }

            return true;
        } catch (err: any) {
            throw formatError(err);
        }
    }
}
```

---

## 五、GraphQL 与 REST 共享边界完整说明（事实纠偏）

### 5.1 架构总览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              客户端请求                                   │
│  ┌─────────────────────┐              ┌─────────────────────────────┐  │
│  │   REST API 请求     │              │      GraphQL 请求           │  │
│  │  GET /items/articles│              │  query { articles { id } } │  │
│  └──────────┬──────────┘              └─────────────┬───────────────┘  │
└─────────────┼────────────────────────────────────────┼──────────────────┘
              │                                        │
              ▼                                        ▼
┌──────────────────────────────┐        ┌───────────────────────────────────┐
│      REST Controller         │        │         GraphQL Layer               │
│  (api/src/controllers/items.ts)│        │                                   │
│                              │        │  ┌─────────────────────────────┐   │
│  职责：                       │        │  │ GraphQL Controller          │   │
│  - Express 路由绑定           │        │  │ (controllers/graphql.ts)    │   │
│  - HTTP 方法处理              │        │  │  - 解析 POST body           │   │
│  - 参数初步验证               │        │  │  - 区分 items/system scope   │   │
│  └──────────────────────────────┘        │  └─────────────┬───────────────┘   │
              │                             │                │                    │
              │                             │                ▼                    │
              │                             │  ┌─────────────────────────────┐   │
              │                             │  │ GraphQLService               │   │
              │                             │  │ (services/graphql/index.ts) │   │
              │                             │  │  - 封装 ItemsService 调用    │   │
              │                             │  │  - read()/upsertSingleton() │   │
              │                             │  │  - 显式传递 knex             │   │
              │                             │  └─────────────┬───────────────┘   │
              │                             │                │                    │
              │                             │                ▼                    │
              │                             │  ┌─────────────────────────────┐   │
              │                             │  │ Resolvers                    │   │
              │                             │  │  - resolveQuery()            │   │
              │                             │  │  - resolveMutation()         │   │
              │                             │  │  - System Resolvers          │   │
              │                             │  └─────────────┬───────────────┘   │
              │                             │                │                    │
              │                             │                ▼                    │
              │                             │  ┌─────────────────────────────┐   │
              │                             │  │ Query 转换层                 │   │
              │                             │  │  - getQuery()               │   │
              │                             │  │  - parse-query.ts           │   │
              │                             │  │  - 别名处理、Deep 参数、M2A  │   │
              │                             │  │    过滤重写                   │   │
              │                             │  └─────────────┬───────────────┘   │
              │                             └────────────────┼────────────────────┘
              │                                              │
              ▼                                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           共享层（Shared Layer）                                  │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         getService() 路由层                               │   │
│  │                    (api/src/utils/get-service.ts)                        │   │
│  │                                                                             │   │
│  │  根据 collection 名称返回正确的服务实例：                                   │   │
│  │  - 'directus_users'    → UsersService                                    │   │
│  │  - 'directus_files'    → FilesService                                    │   │
│  │  - 'directus_roles'    → RolesService                                    │   │
│  │  - ...其他系统集合       → 专用服务                                        │   │
│  │  - 用户自定义集合         → ItemsService                                  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                      │                                            │
│                                      ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         业务服务层（Services）                             │   │
│  │                                                                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────────┐  │   │
│  │  │ UsersService │  │ FilesService │  │      ItemsService           │  │   │
│  │  │  (继承自...) │  │ (继承自...)  │  │  (用户自定义集合基础服务)     │  │   │
│  │  │              │  │              │  │                             │  │   │
│  │  │ - 密码哈希   │  │ - 文件上传   │  │  - createOne/createMany     │  │   │
│  │  │ - TFA 处理   │  │ - 图片处理   │  │  - readOne/readByQuery      │  │   │
│  │  │ - 邀请用户   │  │ - 导入文件   │  │  - updateOne/updateMany      │  │   │
│  │  │              │  │              │  │  - deleteOne/deleteMany      │  │   │
│  │  └──────┬───────┘  └──────┬───────┘  │  - readSingleton/upsertSingleton│ │
│  │         │                 │           └───────────────┬─────────────┘  │   │
│  │         └─────────────────┼───────────────────────────┘                 │   │
│  │                           │                                             │   │
│  │                           ▼                                             │   │
│  │              ┌───────────────────────────────┐                        │   │
│  │              │      ItemsService (基类)       │                        │   │
│  │              │  (api/src/services/items.ts)   │                        │   │
│  │              │                               │                        │   │
│  │              │  核心功能：                    │                        │   │
│  │              │  - 权限检查（validateAccess）  │                        │   │
│  │              │  - 事件触发（emitter）         │                        │   │
│  │              │  - 活动/修订追踪               │                        │   │
│  │              │  - 事务管理                    │                        │   │
│  │              │  - 关系处理（PayloadService）  │                        │   │
│  │              └───────────────┬───────────────┘                        │   │
│  └──────────────────────────────┼──────────────────────────────────────────┘   │
│                                 │                                                │
│                                 ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                      共享工具模块（Shared Utilities）                      │   │
│  │                                                                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────────┐  │   │
│  │  │ sanitizeQuery│  │ validateQuery│  │     PayloadService          │  │   │
│  │  │ (REST+GraphQL)│  │              │  │  - 关系处理（M2O/O2M/A2O） │  │   │
│  │  │              │  │              │  │  - 类型转换                  │  │   │
│  │  └──────────────┘  └──────────────┘  └─────────────────────────────┘  │   │
│  │                                                                             │   │
│  │  ┌──────────────────┐  ┌──────────────────────────────────────────────┐  │   │
│  │  │  权限模块         │  │              事件系统                        │  │   │
│  │  │ - validateAccess │  │  - emitter.emitFilter (操作前钩子)          │  │   │
│  │  │ - processPayload │  │  - emitter.emitAction (操作后钩子)          │  │   │
│  │  │ - processAst     │  │                                              │  │   │
│  │  └──────────────────┘  └──────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                 │                                                │
│                                 ▼                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           数据访问层（Data Layer）                          │
│                                                                           │
│  ┌──────────────┐  ┌──────────────────┐  ┌───────────────────────────┐ │
│  │   Knex.js    │  │  getAstFromQuery │  │        runAst             │ │
│  │ (Query Builder)│  │   (AST 构建)    │  │      (AST 执行)          │ │
│  └──────────────┘  └──────────────────┘  └───────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键事实纠偏

#### 5.2.1 ItemsService 实例化参数纠偏

**❌ v2 版本中的错误描述**：
```typescript
// REST Controller (items.ts)
const service = new ItemsService(req.collection, {
    accountability: req.accountability,
    schema: req.schema,
    knex: req.knex,                        // ❌ 错误：不存在
    nested: req.nested,                    // ❌ 错误：不存在
});
```

**✅ 实际代码证据**（`api/src/controllers/items.ts:25-28`）：
```typescript
const service = new ItemsService(req.collection, {
    accountability: req.accountability,
    schema: req.schema,
});
```

**代码验证**：
- 在整个代码库中搜索 `req.knex`，**无任何匹配结果**
- `req.nested` 也不存在于 Express Request 类型定义中

---

**GraphQL 中的实际参数传递**（`api/src/services/graphql/index.ts:104-108`）：
```typescript
const service = getService(collection, {
    knex: this.knex,           // ✅ 显式传递 knex
    accountability: this.accountability,
    schema: this.schema,
});
```

**关键差异**：

| 参数 | REST Controller | GraphQLService | 说明 |
|-----|----------------|-----------------|------|
| `accountability` | ✅ 传递 | ✅ 传递 | 用户身份信息 |
| `schema` | ✅ 传递 | ✅ 传递 | SchemaOverview |
| `knex` | ❌ 不传递 | ✅ 显式传递 | 数据库连接 |
| `nested` | ❌ 不传递 | ❌ 不传递 | 不存在此参数 |

**技术说明**：
- ItemsService 构造函数中，`knex` 是可选的。如果不传递，ItemsService 内部会通过 `getDatabase()` 获取默认连接
- GraphQL 显式传递 `knex` 是为了确保使用同一个数据库连接实例
- REST 中不传递 `knex`，依赖 ItemsService 内部的默认机制

#### 5.2.2 sanitizedQuery 使用纠偏

**❌ v2 版本中的错误描述**：
```
"通过 `sanitizeQuery(req.query)` 转换为 Query 对象"
```

**✅ 实际代码证据**：

1. **中间件预处理**（`api/src/middleware/sanitize-query.ts:10-38`）：
```typescript
const sanitizeQueryMiddleware: RequestHandler = async (req, _res, next) => {
    req.sanitizedQuery = {};
    if (!req.query) return;

    // Skip sanitization and validation if query is empty
    if (Object.keys(req.query).length === 0) {
        Object.freeze(req.sanitizedQuery);
        return next();
    }

    try {
        req.sanitizedQuery = await sanitizeQuery(
            {
                fields: req.query['fields'] || '*',
                ...req.query,
            },
            req.schema,
            req.accountability || null,
        );

        Object.freeze(req.sanitizedQuery);
        validateQuery(req.sanitizedQuery);
    } catch (error) {
        return next(error);
    }

    return next();
};
```

2. **REST Controller 使用**（`api/src/controllers/items.ts:61-92`）：
```typescript
const readHandler = asyncHandler(async (req, res, next) => {
    // ...
    let result;

    if (req.singleton) {
        result = await service.readSingleton(req.sanitizedQuery);  // ✅ 直接使用
    } else if (req.body.keys) {
        result = await service.readMany(req.body.keys, req.sanitizedQuery);  // ✅ 直接使用
    } else {
        result = await service.readByQuery(req.sanitizedQuery);  // ✅ 直接使用
    }
    // ...
});
```

**特殊情况：updateByQuery 和 deleteByQuery**（`api/src/controllers/items.ts:146-147` 和 `216-217`）：

```typescript
// updateByQuery 时，query 来自 req.body.query 而不是 URL 参数
} else {
    const sanitizedQuery = await sanitizeQuery(req.body.query, req.schema, req.accountability);  // 手动调用
    keys = await service.updateByQuery(sanitizedQuery, req.body.data);
}

// deleteByQuery 同理
} else {
    const sanitizedQuery = await sanitizeQuery(req.body.query, req.schema, req.accountability);  // 手动调用
    await service.deleteByQuery(sanitizedQuery);
}
```

**正确的表述**：

| 场景 | 数据来源 | 处理方式 |
|-----|---------|---------|
| 正常读取操作（GET/SEARCH） | URL 查询参数（`req.query`） | 中间件预处理 → `req.sanitizedQuery` |
| updateByQuery | 请求体（`req.body.query`） | 手动调用 `sanitizeQuery()` |
| deleteByQuery | 请求体（`req.body.query`） | 手动调用 `sanitizeQuery()` |

**技术说明**：
- `sanitizeQueryMiddleware` 是一个全局中间件，在路由处理之前执行
- 它将 Express 的 `req.query`（字符串形式的 URL 参数）转换为类型安全的 `Query` 对象
- 结果存储在 `req.sanitizedQuery` 中，并被 `Object.freeze()` 冻结，防止意外修改
- Controller 直接使用 `req.sanitizedQuery`，而不是调用 `sanitizeQuery(req.query)`

### 5.3 共享组件详细说明

#### 5.3.1 完全共享的组件

| 组件 | 位置 | REST 使用方式 | GraphQL 使用方式 | 说明 |
|-----|------|--------------|-----------------|------|
| **ItemsService** | `services/items.ts` | 直接实例化 | 通过 `getService()` 获取 | 核心 CRUD 业务逻辑 |
| **系统服务** | `services/*.ts` | 直接实例化 | 通过 `getService()` 获取 | UsersService, FilesService 等 |
| **getService()** | `utils/get-service.ts` | 不直接使用 | 必用 | 服务路由 |
| **sanitizeQuery()** | `utils/sanitize-query.ts` | 中间件调用 | `getQuery()` 内部调用 | Query 参数净化 |
| **validateQuery()** | `utils/validate-query.ts` | 中间件调用 | `getQuery()` 内部调用 | Query 验证 |
| **权限模块** | `permissions/modules/*` | ItemsService 内部 | ItemsService 内部 | 权限检查 |
| **事件系统** | `emitter.ts` | ItemsService 内部 | ItemsService 内部 | Hooks 触发 |
| **PayloadService** | `services/payload.ts` | ItemsService 内部 | ItemsService 内部 | 关系处理 |
| **ActivityService** | `services/activity.ts` | ItemsService 内部 | ItemsService 内部 | 活动追踪 |
| **RevisionsService** | `services/revisions.ts` | ItemsService 内部 | ItemsService 内部 | 修订追踪 |

#### 5.3.2 GraphQL 独有的组件

| 组件 | 位置 | 说明 |
|-----|------|------|
| **GraphQLService** | `services/graphql/index.ts` | GraphQL 服务封装，显式管理 knex 连接 |
| **Schema 生成** | `services/graphql/schema/*` | 动态 Schema 生成 |
| **Resolvers** | `services/graphql/resolvers/*` | Query/Mutation 解析器 |
| **Query 转换** | `services/graphql/schema/parse-query.ts` | GraphQL AST → Directus Query |
| **M2A 过滤重写** | `services/graphql/utils/filter-replace-m2a.ts` | M2A 关系过滤格式转换 |
| **Scope 机制** | N/A | `items`/`system` 双端点 |
| **System Resolvers** | `services/graphql/resolvers/system.ts` | 系统专用 resolvers |

#### 5.3.3 REST 独有的组件

| 组件 | 位置 | 说明 |
|-----|------|------|
| **Items Controller** | `controllers/items.ts` | Express 路由绑定 |
| **sanitizeQueryMiddleware** | `middleware/sanitize-query.ts` | 预处理 URL 查询参数 |
| **HTTP 中间件** | `middleware/*` | 认证、缓存、CORS 等 |
| **MetaService** | `services/meta.ts` | REST 响应的 meta 字段 |
| **respond 中间件** | `middleware/respond.ts` | 统一响应格式化 |

### 5.4 Query 对象的统一结构

无论是 REST 还是 GraphQL，最终都转换为相同的 `Query` 结构传递给 ItemsService：

```typescript
interface Query {
    // 基础字段
    fields?: string[];           // 要返回的字段
    filter?: Filter;             // 过滤条件
    sort?: string[];             // 排序
    limit?: number;              // 限制数量
    offset?: number;             // 偏移量
    page?: number;               // 页码
    search?: string;             // 搜索关键词

    // 聚合查询
    group?: string[];            // 分组字段
    aggregate?: Aggregate;       // 聚合函数

    // 嵌套查询（GraphQL 特有，但 ItemsService 支持）
    deep?: DeepQuery;            // 关系字段的嵌套查询参数
    alias?: Record<string, string>; // 字段别名映射

    // 版本控制
    version?: string;            // 版本号
    versionRaw?: boolean;        // 是否原始版本数据

    // 其他
    export?: 'json' | 'csv';     // 导出格式
}
```

**REST 传递方式**：
```
GET /items/articles?fields=id,title&filter[status][_eq]=published&limit=10
```
- 中间件 `sanitizeQueryMiddleware` 预处理
- 结果存入 `req.sanitizedQuery`
- Controller 直接使用 `req.sanitizedQuery`

**GraphQL 传递方式**：
```graphql
query {
    articles(filter: { status: { _eq: "published" } }, limit: 10) {
        id
        title
    }
}
```
- Resolver 调用 `getQuery()` 解析
- 从 GraphQL AST 中提取字段、参数、别名、关系等
- 生成完整的 Query 对象

### 5.5 服务实例化参数对比

#### 5.5.1 REST Controller 中的实例化

**文件位置**：`api/src/controllers/items.ts:25-28`

```typescript
const service = new ItemsService(req.collection, {
    accountability: req.accountability,
    schema: req.schema,
});
```

**参数说明**：
- `accountability`：用户身份信息（用户 ID、角色 ID、IP、User-Agent 等）
- `schema`：数据库 SchemaOverview（包含 collections、fields、relations 等元数据）

**注意**：不传递 `knex` 参数，ItemsService 内部会使用默认的数据库连接。

#### 5.5.2 GraphQLService 中的实例化

**文件位置**：`api/src/services/graphql/index.ts:37-42`

```typescript
constructor(options: AbstractServiceOptions & { scope: GQLScope }) {
    this.accountability = options?.accountability || null;
    this.knex = options?.knex || getDatabase();  // 显式获取
    this.schema = options.schema;
    this.scope = options.scope;
}
```

**文件位置**：`api/src/services/graphql/index.ts:104-108`

```typescript
const service = getService(collection, {
    knex: this.knex,           // 显式传递
    accountability: this.accountability,
    schema: this.schema,
});
```

**参数说明**：
- `knex`：显式传递的数据库连接实例
- `accountability`：用户身份信息
- `schema`：数据库 SchemaOverview

#### 5.5.3 为何存在差异？

ItemsService 的构造函数中，`knex` 是可选参数：

```typescript
// 抽象服务基类
constructor(options: AbstractServiceOptions) {
    this.knex = options?.knex || getDatabase();
    // ...
}
```

**设计意图**：
- **REST**：简化调用，依赖默认机制
- **GraphQL**：显式管理，确保在同一执行上下文中使用相同的连接实例

**实际效果**：
- 两者最终使用的是同一个数据库连接池
- GraphQL 的显式传递只是确保在复杂执行场景下的一致性

### 5.6 操作流程对比

#### 5.6.1 读取操作流程对比

**REST 流程**：
```
1. HTTP 请求 → Express 路由
   GET /items/articles?fields=id,title,author.name&filter[status][_eq]=published

2. 中间件预处理
   → sanitizeQueryMiddleware(req, res, next)
     • 从 req.query 中提取参数
     • 调用 sanitizeQuery() 转换
     • 结果存入 req.sanitizedQuery（已冻结）
     • 调用 validateQuery() 验证

3. Controller 处理
   → 直接使用 req.sanitizedQuery
   → new ItemsService(collection, { accountability, schema })
     注意：不传递 knex
   → service.readByQuery(req.sanitizedQuery)

4. ItemsService 处理（与 GraphQL 共享）
   → emitter.emitFilter('items.query', ...)  // 前置钩子
   → getAstFromQuery()                        // 构建 AST
   → processAst()                              // 权限注入
   → runAst()                                  // 执行查询
   → emitter.emitFilter('items.read', ...)    // 后置钩子
   → emitter.emitAction('items.read', ...)    // 事件触发

5. 响应格式化
   → MetaService.getMetaForQuery()
   → respond 中间件 → { data: [...], meta: {...} }
```

**GraphQL 流程**：
```
1. GraphQL 请求
   POST /graphql
   {
       "query": "query { articles(filter: { status: { _eq: \"published\" } }) { id title author { name } } }"
   }

2. GraphQL Controller 处理
   → parseGraphQL 中间件 → 解析 document, variables
   → new GraphQLService({ scope: 'items', accountability, schema, knex })
     注意：显式传递 knex
   → service.execute(graphqlParams)

3. Schema 获取与执行
   → getSchema() → 生成或获取缓存的 GraphQLSchema
   → validate(schema, document)
   → execute() → 调用 resolver

4. Resolver 处理 (resolveQuery)
   → 解析 collection 名称
   → getQuery(args, schema, selections, ...)  → Query 对象
     • parseAliases() → 别名映射
     • parseFields() → 字段解析，生成 fields 数组
     • 处理嵌套字段参数 → 写入 query.deep
     • filterReplaceM2A() → M2A 过滤重写
   → gql.read(collection, query, args['id'])

5. GraphQLService.read()
   → getService(collection, { knex, accountability, schema })
     注意：显式传递 knex

6. ItemsService 处理（与 REST 完全相同）
   → emitter.emitFilter('items.query', ...)
   → getAstFromQuery()
   → processAst()
   → runAst()
   → ...

7. GraphQL 响应
   → { data: { articles: [...] }, errors?: [...] }
```

#### 5.6.2 写入操作流程对比

**REST 创建流程**：
```
POST /items/articles
Body: { "title": "New Article", "author": 1 }

1. Controller 处理
   → new ItemsService('articles', { accountability, schema })
     注意：不传递 knex
   → service.createOne(req.body)  // 或 createMany for array

2. ItemsService.createOne
   → 事务开始
   → emitter.emitFilter('items.create', payload)
   → processPayload() → 权限检查、字段预设
   → PayloadService.processM2O/A2O/O2M → 关系处理
   → Knex.insert() → 数据库插入
   → ActivityService.createOne() → 活动记录
   → RevisionsService.createOne() → 修订记录（如果启用）
   → 事务提交
   → emitter.emitAction('items.create', ...)

3. 读取创建的结果
   → service.readOne(primaryKey, req.sanitizedQuery)
   → 响应: { data: { id: 1, title: "New Article", ... } }
```

**GraphQL 创建流程**：
```
POST /graphql
{
    "query": "mutation { create_articles_item(data: { title: \"New Article\", author: 1 }) { id title } }"
}

1. Resolver 处理 (resolveMutation)
   → 解析 action='create', collection='articles'
   → getQuery(args, ...) → 解析返回字段
   → getService('articles', { knex, accountability, schema })
     注意：显式传递 knex

2. 执行创建
   → service.createOne(args['data'])
   // ItemsService 内部流程与 REST 完全相同

3. 读取结果
   → service.readOne(primaryKey, query)
   → 返回: { id: 1, title: "New Article" }
```

---

## 六、关键代码位置汇总

### 6.1 GraphQL 相关

| 功能模块 | 文件路径 |
|---------|---------|
| GraphQL 服务入口 | `api/src/services/graphql/index.ts` |
| Schema 生成核心 | `api/src/services/graphql/schema/index.ts` |
| Schema 缓存 | `api/src/services/graphql/schema-cache.ts` |
| 类型生成 | `api/src/services/graphql/schema/get-types.ts` |
| 可读类型与 Resolver | `api/src/services/graphql/schema/read.ts` |
| 可写类型 | `api/src/services/graphql/schema/write.ts` |
| Query 解析 | `api/src/services/graphql/schema/parse-query.ts` |
| 参数解析 | `api/src/services/graphql/schema/parse-args.ts` |
| 查询 Resolver | `api/src/services/graphql/resolvers/query.ts` |
| 变更 Resolver | `api/src/services/graphql/resolvers/mutation.ts` |
| 系统 Resolvers | `api/src/services/graphql/resolvers/system.ts` |
| M2A 过滤重写 | `api/src/services/graphql/utils/filter-replace-m2a.ts` |
| GraphQL 控制器 | `api/src/controllers/graphql.ts` |

### 6.2 REST 相关

| 功能模块 | 文件路径 |
|---------|---------|
| Items 控制器 | `api/src/controllers/items.ts` |
| Query 净化中间件 | `api/src/middleware/sanitize-query.ts` |
| Query 净化工具 | `api/src/utils/sanitize-query.ts` |
| Query 验证 | `api/src/utils/validate-query.ts` |
| 响应中间件 | `api/src/middleware/respond.ts` |
| Express 类型扩展 | `api/src/types/express.d.ts` |

### 6.3 共享层

| 功能模块 | 文件路径 |
|---------|---------|
| ItemsService 核心 | `api/src/services/items.ts` |
| 服务路由 | `api/src/utils/get-service.ts` |
| 负载服务（关系处理） | `api/src/services/payload.ts` |
| 权限检查 | `api/src/permissions/modules/validate-access/validate-access.ts` |
| 负载处理 | `api/src/permissions/modules/process-payload/process-payload.ts` |
| AST 处理 | `api/src/permissions/modules/process-ast/process-ast.ts` |
| 事件系统 | `api/src/emitter.ts` |
| 活动服务 | `api/src/services/activity.ts` |
| 修订服务 | `api/src/services/revisions.ts` |

---

## 七、事实纠偏总结

### 7.1 已修正的错误

| 错误类型 | v2 版本错误 | v3 版本正确 | 证据位置 |
|---------|------------|------------|---------|
| ItemsService 参数 | 传递了 `knex: req.knex` 和 `nested: req.nested` | 只传递了 `accountability` 和 `schema` | `api/src/controllers/items.ts:25-28` |
| GraphQL 参数对比 | 说 REST 和 GraphQL 参数"完全一致" | 指出了 `knex` 传递的差异 | `api/src/services/graphql/index.ts:104-108` |
| sanitizedQuery 使用 | "通过 `sanitizeQuery(req.query)` 转换" | 中间件预处理存入 `req.sanitizedQuery`，Controller 直接使用 | `api/src/middleware/sanitize-query.ts:10-38` |

### 7.2 补充的细节

1. **updateByQuery/deleteByQuery 的特殊处理**：
   - 这两个操作中，query 来自 `req.body.query` 而不是 URL 参数
   - Controller 手动调用 `sanitizeQuery(req.body.query, ...)`

2. **req.sanitizedQuery 的冻结机制**：
   - 中间件使用 `Object.freeze(req.sanitizedQuery)` 防止意外修改
   - 这是一个重要的设计细节

3. **GraphQL 显式传递 knex 的原因**：
   - 确保在复杂执行场景下使用相同的连接实例
   - 但两者最终使用的是同一个数据库连接池

### 7.3 设计哲学

Directus 的架构体现了以下设计哲学：

1. **协议无关的业务逻辑**：
   - 业务逻辑（ItemsService）不依赖任何特定的传输协议
   - REST 和 GraphQL 只是不同的"适配器"，将各自的协议格式转换为统一的 Query 对象和服务调用

2. **配置驱动的动态性**：
   - GraphQL Schema 完全根据数据库 schema 动态生成
   - 新增 collection 或修改 field 无需修改代码
   - 权限配置直接影响可见的 Schema 结构

3. **多态服务设计**：
   - 通过 `getService()` 实现服务路由
   - 系统服务通过继承 ItemsService 扩展功能
   - 调用方无需知道具体是哪个服务类

4. **上下文贯穿**：
   - Accountability 对象在整个调用链中保持一致
   - 这是实现权限检查、活动追踪、事件触发的基础
   - 任何操作都可以追溯到具体的用户和请求

5. **关注点分离**：
   - **中间件**：协议解析、参数预处理（sanitizeQueryMiddleware）
   - **Controller/Resolver**：路由绑定、参数验证、结果格式化
   - **Service**：业务逻辑、权限检查、事务管理
   - **PayloadService**：关系处理、类型转换
   - **Data Layer**：数据库操作
