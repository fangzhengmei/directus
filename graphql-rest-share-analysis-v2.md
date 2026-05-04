# Directus GraphQL Schema 动态生成与 REST 共享 ItemsService 机制分析（修订版）

## 一、概述

Directus 通过一套统一的架构设计，实现了 GraphQL 和 REST API 共享同一套业务逻辑层（ItemsService）。这种设计保证了两种 API 接口在数据操作、权限验证、事件触发等方面的一致性。

本文档深入分析以下核心机制：
1. GraphQL Schema 的动态生成（含缓存、Scope 过滤、System Resolver 注入）
2. Mutation 写操作复用 ItemsService 的完整链路
3. Parse-query 中 alias、deep 参数处理与 M2A 过滤重写
4. GraphQL 与 REST 共享边界的完整说明

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

### 2.2 Schema 缓存机制（修正）

**文件位置**：`api/src/services/graphql/schema-cache.ts`

Schema 缓存使用 `mnemonist` 库的 `LRUMap` 实现，而非 `lru-cache`：

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
            project: {
                type: new GraphQLObjectType({
                    name: 'server_info_project',
                    fields: {
                        project_name: { type: GraphQLString },
                        project_descriptor: { type: GraphQLString },
                        // ... 更多项目配置字段
                    },
                }),
            },
            // rateLimit, websocket, queryLimit 等
        },
    });

    // 3. Query 字段注册

    // 3.1 服务器相关
    schemaComposer.Query.addFields({
        server_ping: {
            type: GraphQLString,
            resolve: () => 'pong',
        },
        server_info: {
            type: ServerInfo,
            resolve: dedupeResolver(async () => {
                const service = new ServerService({
                    accountability: gql.accountability,
                    schema: gql.schema,
                });
                return await service.serverInfo();
            }, 'server_info'),
        },
        server_health: {
            type: GraphQLJSON,
            resolve: dedupeResolver(async () => {
                const service = new ServerService({ ... });
                return await service.health();
            }, 'server_health'),
        },
    });

    // 3.2 Collections 元数据查询
    if ('directus_collections' in schema.read.collections) {
        const Collection = getCollectionType(schemaComposer, schema, 'read');

        schemaComposer.Query.addFields({
            collections: {
                type: new GraphQLNonNull(new GraphQLList(new GraphQLNonNull(Collection.getType()))),
                resolve: dedupeResolver(async () => {
                    const collectionsService = new CollectionsService({
                        accountability: gql.accountability,
                        schema: gql.schema,
                    });
                    return await collectionsService.readByQuery();
                }, 'directus_collections'),
            },
            collections_by_name: {
                type: Collection,
                args: { name: new GraphQLNonNull(GraphQLString) },
                resolve: dedupeResolver(async (_, args) => {
                    const collectionsService = new CollectionsService({ ... });
                    return await collectionsService.readOne(args['name']);
                }),
            },
        });
    }

    // 3.3 Fields 元数据查询
    if ('directus_fields' in schema.read.collections) {
        const Field = getFieldType(schemaComposer, schema, 'read');

        schemaComposer.Query.addFields({
            fields: {
                type: new GraphQLNonNull(new GraphQLList(new GraphQLNonNull(Field.getType()))),
                resolve: dedupeResolver(async () => {
                    const service = new FieldsService({ ... });
                    return await service.readAll();
                }, 'directus_fields'),
            },
            fields_in_collection: { /* ... */ },
            fields_by_name: { /* ... */ },
        });
    }

    // 3.4 Relations 元数据查询
    if ('directus_relations' in schema.read.collections) {
        const Relation = getRelationType(schemaComposer, schema, 'read');

        schemaComposer.Query.addFields({
            relations: { /* ... */ },
            relations_in_collection: { /* ... */ },
            relations_by_name: { /* ... */ },
        });
    }

    // 4. 管理员专用 Resolvers
    resolveSystemAdmin(gql, schema, schemaComposer);

    // 5. 当前用户相关 Query

    // 5.1 users_me - 获取当前用户信息
    if ('directus_users' in schema.read.collections) {
        schemaComposer.Query.addFields({
            users_me: {
                type: ReadCollectionTypes['directus_users']!,
                resolve: dedupeResolver(async (_, args, __, info) => {
                    if (!gql.accountability?.user) return null;
                    const service = new UsersService({ schema: gql.schema, accountability: gql.accountability });

                    const selections = replaceFragmentsInSelections(
                        info.fieldNodes[0]?.selectionSet?.selections, info.fragments
                    );

                    const query = await getQuery(
                        args, gql.schema, selections || [], info.variableValues,
                        gql.accountability, 'directus_users'
                    );

                    return await service.readOne(gql.accountability.user, query);
                }),
            },
        });
    }

    // 5.2 permissions_me - 获取当前用户权限
    if ('directus_permissions' in schema.read.collections) {
        schemaComposer.Query.addFields({
            permissions_me: {
                type: schemaComposer.createScalarTC<CollectionAccess>({ /* ... */ }),
                resolve: dedupeResolver(async (_, _args, __, _info) => {
                    if (!gql.accountability?.user && !gql.accountability?.role) return null;

                    const result = await fetchAccountabilityCollectionAccess(gql.accountability, {
                        schema: gql.schema,
                        knex: getDatabase(),
                    });

                    return result;
                }),
            },
        });
    }

    // 5.3 roles_me - 获取当前用户角色
    if ('directus_roles' in schema.read.collections) {
        schemaComposer.Query.addFields({
            roles_me: {
                type: ReadCollectionTypes['directus_roles']!.List,
                resolve: dedupeResolver(async (_, args, __, info) => {
                    if (!gql.accountability?.user && !gql.accountability?.role) return null;

                    const service = new RolesService({
                        accountability: gql.accountability,
                        schema: gql.schema,
                    });

                    const selections = replaceFragmentsInSelections(/* ... */);
                    const query = await getQuery(/* ... */);
                    query.limit = -1;

                    const roles = await service.readMany(gql.accountability.roles, query);
                    return roles;
                }),
            },
        });
    }

    // 6. Mutation 字段注册

    // 6.1 update_users_me - 更新当前用户信息
    if ('directus_users' in schema.update.collections && gql.accountability?.user) {
        schemaComposer.Mutation.addFields({
            update_users_me: {
                type: ReadCollectionTypes['directus_users']!,
                args: {
                    data: toInputObjectType(UpdateCollectionTypes['directus_users']!),
                },
                resolve: async (_, args, __, info) => {
                    if (!gql.accountability?.user) return null;

                    const service = new UsersService({
                        schema: gql.schema,
                        accountability: gql.accountability,
                    });

                    await service.updateOne(gql.accountability.user, args['data']);

                    // 读取更新后的结果
                    if ('directus_users' in ReadCollectionTypes) {
                        const selections = replaceFragmentsInSelections(/* ... */);
                        const query = await getQuery(/* ... */);
                        return await service.readOne(gql.accountability.user, query);
                    }

                    return true;
                },
            },
        });
    }

    // 6.2 import_file - 导入文件（FilesService 专用方法）
    if ('directus_files' in schema.create.collections) {
        schemaComposer.Mutation.addFields({
            import_file: {
                type: ReadCollectionTypes['directus_files'] ?? GraphQLBoolean,
                args: {
                    url: new GraphQLNonNull(GraphQLString),
                    data: toInputObjectType(CreateCollectionTypes['directus_files']!).setTypeName('create_directus_files_input'),
                },
                resolve: async (_, args, __, info) => {
                    const service = new FilesService({
                        accountability: gql.accountability,
                        schema: gql.schema,
                    });

                    const primaryKey = await service.importOne(args['url'], args['data']);

                    if ('directus_files' in ReadCollectionTypes) {
                        const selections = replaceFragmentsInSelections(/* ... */);
                        const query = await getQuery(/* ... */);
                        return await service.readOne(primaryKey, query);
                    }

                    return true;
                },
            },
        });
    }

    // 6.3 users_invite - 邀请用户（UsersService 专用方法）
    if ('directus_users' in schema.create.collections) {
        schemaComposer.Mutation.addFields({
            users_invite: {
                type: GraphQLBoolean,
                args: {
                    email: new GraphQLNonNull(GraphQLString),
                    role: new GraphQLNonNull(GraphQLString),
                    invite_url: GraphQLString,
                },
                resolve: async (_, args) => {
                    const service = new UsersService({
                        accountability: gql.accountability,
                        schema: gql.schema,
                    });

                    await service.inviteUser(args['email'], args['role'], args['invite_url'] || null);
                    return true;
                },
            },
        });
    }

    return schemaComposer;
}
```

### 2.5 类型生成机制

#### 2.5.1 基础类型生成 (`getTypes`)

**文件位置**：`api/src/services/graphql/schema/get-types.ts`

该函数为每个 collection 生成对应的 GraphQL ObjectType，并处理关系字段：

```typescript
export function getTypes(
    schemaComposer: SchemaComposer,
    scope: GQLScope,
    schema: Schema,
    inconsistentFields: InconsistentFields,
    action: 'read' | 'create' | 'update' | 'delete',
) {
    const CollectionTypes: Record<string, ObjectTypeComposer> = {};
    const VersionTypes: Record<string, ObjectTypeComposer> = {};

    for (const collection of Object.values(schema[action].collections)) {
        // 创建 ObjectTypeComposer
        CollectionTypes[collection.collection] = schemaComposer.createObjectTC({
            name: action === 'read' ? collection.collection : `${action}_${collection.collection}`,
            fields: Object.values(collection.fields).reduce((acc, field) => {
                // 类型映射：Directus 字段类型 → GraphQL 类型
                let type = getGraphQLType(field.type, field.special);

                // 非空约束处理（update 操作不强制非空）
                if (
                    field.nullable === false &&
                    !field.defaultValue &&
                    !GENERATE_SPECIAL.some((flag) => field.special.includes(flag)) &&
                    fieldIsInconsistent === false &&
                    action !== 'update'
                ) {
                    type = new GraphQLNonNull(type);
                }

                // 主键字段特殊处理
                if (collection.primary === field.field && fieldIsInconsistent === false) {
                    // permissions IDs need to be nullable
                    if (collection.collection === 'directus_permissions') {
                        type = GraphQLID;
                    } else if (!field.defaultValue && !field.special.includes('uuid') && action === 'create') {
                        type = new GraphQLNonNull(GraphQLID);
                    } else if (['create', 'update'].includes(action)) {
                        type = GraphQLID;
                    } else {
                        type = new GraphQLNonNull(GraphQLID);
                    }
                }

                acc[field.field] = {
                    type,
                    description: field.note,
                    resolve: (obj: Record<string, any>) => obj[field.field],
                };

                // 函数字段（_func 后缀）
                if (action === 'read') {
                    if (field.type === 'date') {
                        acc[`${field.field}_func`] = { type: DateFunctions, resolve: /* ... */ };
                    }
                    if (field.type === 'time') { /* ... */ }
                    if (field.type === 'dateTime' || field.type === 'timestamp') { /* ... */ }
                    if (field.type === 'json' || field.type === 'alias') { /* ... */ }
                }

                return acc;
            }, {}),
        });

        // 版本类型（仅 items scope）
        if (scope === 'items') {
            VersionTypes[collection.collection] = CollectionTypes[collection.collection]!.clone(
                `version_${collection.collection}`,
            );
        }
    }

    // 关系字段处理
    for (const relation of schema[action].relations) {
        if (relation.related_collection) {
            // M2O (Many-to-One) 关系
            CollectionTypes[relation.collection]?.addFields({
                [relation.field]: {
                    type: CollectionTypes[relation.related_collection]!,
                    resolve: (obj, _, __, info) => obj[info?.path?.key ?? relation.field],
                },
            });

            // O2M (One-to-Many) 反向关系
            if (relation.meta?.one_field) {
                CollectionTypes[relation.related_collection]?.addFields({
                    [relation.meta.one_field]: {
                        type: [CollectionTypes[relation.collection]!],
                        resolve: (obj, _, __, info) => obj[info?.path?.key ?? relation.meta!.one_field],
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
                            let path: (string | number)[] = [];
                            let currentPath = info.path;
                            while (currentPath.prev) {
                                path.push(currentPath.key);
                                currentPath = currentPath.prev;
                            }
                            path = path.reverse().slice(0, -1);

                            let parent = context['data']!;
                            for (const pathPart of path) {
                                parent = parent[pathPart];
                            }

                            const collection = parent[relation.meta!.one_collection_field!]!;
                            return CollectionTypes[collection]!.getType().name;
                        },
                    }),
                    resolve: (obj, _, __, info) => obj[info?.path?.key ?? relation.field],
                },
            });
        }
    }

    return { CollectionTypes, VersionTypes };
}
```

#### 2.5.2 可读类型增强 (`getReadableTypes`)

**文件位置**：`api/src/services/graphql/schema/read.ts`

在基础类型之上，添加查询能力：

1. **过滤器类型**：为每个字段创建 `_eq`, `_neq`, `_contains`, `_in` 等过滤操作符
2. **聚合查询**：为数值字段添加 `count`, `sum`, `avg`, `min`, `max` 等聚合函数
3. **排序/分页参数**：`sort`, `limit`, `offset`, `page`, `search`
4. **Resolver 注册**：

```typescript
ReadCollectionTypes[collection.collection]!.addResolver({
    name: collection.collection,
    type: collection.singleton
        ? ReadCollectionTypes[collection.collection]!
        : new GraphQLNonNull(
                new GraphQLList(new GraphQLNonNull(ReadCollectionTypes[collection.collection]!.getType())),
            ),
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

---

## 三、Query 解析与转换机制

### 3.1 Parse-Query 核心逻辑

**文件位置**：`api/src/services/graphql/schema/parse-query.ts`

这是 GraphQL 查询转换为 Directus Query 的核心模块，处理以下关键点：

```typescript
export async function getQuery(
    rawQuery: Query,
    schema: SchemaOverview,
    selections: readonly SelectionNode[],
    variableValues: GraphQLResolveInfo['variableValues'],
    accountability?: Accountability | null,
    collection?: string,
): Promise<Query> {
    // 1. 基础 sanitize（与 REST 共享）
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

            let current: string;
            let currentAlias: string | null = null;
            let childCollection: string | undefined = currentCollection;

            // 处理 InlineFragment（主要用于 A2O 关系）
            if (selection.kind === 'InlineFragment') {
                if (selection.typeCondition!.name.value.startsWith('__')) continue;

                const isM2A = getRelationInfo(
                    schema.relations, currentCollection, parentFieldName
                ).relationType === 'a2o';

                if (isM2A) {
                    // M2A 片段：格式化为 parent:collection
                    current = `${parent}:${selection.typeCondition!.name.value}`;
                    childCollection = selection.typeCondition!.name.value;
                } else {
                    // 非 M2A 片段：内联子字段
                    const children = await parseFields(
                        selection.selectionSet?.selections ?? [],
                        parent, currentCollection, parentFieldName
                    );
                    fields.push(...children);
                    continue;
                }
            } else {
                // 普通字段
                if (selection.name.value.startsWith('__')) continue;  // 过滤 __typename 等

                current = selection.name.value;
                if (selection.alias) {
                    currentAlias = selection.alias.value;
                }

                // 嵌套字段处理
                if (parent) {
                    current = `${parent}.${current}`;

                    if (currentAlias) {
                        currentAlias = `${parent}.${currentAlias}`;

                        // 关键：嵌套别名写入 deep 参数
                        if (selection.selectionSet) {
                            if (!query.deep) query.deep = {};

                            const path = parent.replaceAll(':', '__');

                            set(
                                query.deep,
                                path,
                                merge({}, get(query.deep, parent), {
                                    _alias: { [selection.alias!.value]: selection.name.value }
                                }),
                            );
                        }
                    }
                }

                // 解析子集合（用于递归）
                if (currentCollection && selection.selectionSet) {
                    childCollection = getRelatedCollection(
                        schema, currentCollection, selection.name.value
                    ) ?? currentCollection;
                }
            }

            // 递归处理子选择集
            if (selection.selectionSet) {
                let children: string[];

                if (current.endsWith('_func')) {
                    // 函数字段特殊处理：year(date) 格式
                    children = [];
                    const rootField = current.slice(0, -5);
                    for (const subSelection of selection.selectionSet.selections) {
                        if (subSelection.kind !== 'Field') continue;
                        if (subSelection.name!.value.startsWith('__')) continue;
                        children.push(`${subSelection.name!.value}(${rootField})`);
                    }
                } else {
                    children = await parseFields(
                        selection.selectionSet.selections,
                        currentAlias ?? current,
                        childCollection,
                        selection.kind === 'Field' ? selection.name.value : undefined,
                    );
                }

                fields.push(...children);
            } else {
                fields.push(current);
            }

            // 关键：字段参数写入 deep 参数
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

    // 6. 关键：M2A 过滤重写
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

        if (type === 'o2m' && relation) {
            // 递归处理 O2M 关系
            filter[key] = filterReplaceM2A(filter[key], relation.collection, schema, options);
        } else if (type === 'm2o' && relation) {
            // 递归处理 M2O 关系
            filter[key] = filterReplaceM2A(filter[key], relation.related_collection!, schema, options);
        } else if (
            type === 'a2o' &&
            relation &&
            any_collection &&
            relation.meta?.one_allowed_collections?.includes(any_collection)
        ) {
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

**M2A Deep 参数重写**：
```typescript
export function filterReplaceM2ADeep(
    deep_arg: NestedDeepQuery | null | undefined,
    collection: string,
    schema: SchemaOverview,
    options?: { aliasMap?: Query['alias'] },
) {
    const deep: any = deep_arg;

    for (const key in deep) {
        if (key.startsWith('_') === false) {
            const parts = key.split('__');
            let field = parts[0];
            const any_collection = parts[1];

            field = options?.aliasMap?.[field] || (deep as DeepQuery)._alias?.[field] || field;

            const relation = getRelation(schema.relations, collection, field);
            if (!relation) continue;

            const type = getRelationType({ relation, collection, field, useA2O: true });

            if (type === 'o2m') {
                deep[key] = filterReplaceM2ADeep(deep[key], relation.collection, schema);
            } else if (type === 'm2o') {
                deep[key] = filterReplaceM2ADeep(deep[key], relation.related_collection!, schema);
            } else if (type === 'a2o' && any_collection && relation.meta?.one_allowed_collections?.includes(any_collection)) {
                // 重写 deep 中的 M2A 字段
                deep[`${field}:${any_collection}`] = filterReplaceM2ADeep(deep[key], any_collection, schema);
                delete deep[key];
            }
        }

        // 递归处理 _filter 中的 M2A
        if (key === '_filter') {
            deep[key] = filterReplaceM2A(deep[key], collection, schema);
        }
    }

    return deep;
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

    // 6. 获取服务实例（与 REST 共享）
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
                    // update_batch：数组形式，每条包含主键
                    keys.push(...(await service.updateBatch(args['data'])));
                } else {
                    // update_items：ids + data 形式
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
    // ...

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

        try {
            // 调用 ItemsService 的 upsertSingleton
            await service.upsertSingleton(body);

            // 如果有返回字段需求，读取结果
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

## 五、GraphQL 与 REST 共享边界完整说明

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

### 5.2 共享边界详细说明

#### 5.2.1 完全共享的组件

| 组件 | 位置 | REST 使用方式 | GraphQL 使用方式 | 说明 |
|-----|------|--------------|-----------------|------|
| **ItemsService** | `services/items.ts` | 直接实例化 | 通过 `getService()` 获取 | 核心 CRUD 业务逻辑 |
| **系统服务** | `services/*.ts` | 直接实例化 | 通过 `getService()` 获取 | UsersService, FilesService 等 |
| **getService()** | `utils/get-service.ts` | 不直接使用 | 必用 | 服务路由 |
| **sanitizeQuery()** | `utils/sanitize-query.ts` | 直接调用 | `getQuery()` 内部调用 | Query 参数净化 |
| **validateQuery()** | `utils/validate-query.ts` | 直接调用 | `getQuery()` 内部调用 | Query 验证 |
| **权限模块** | `permissions/modules/*` | ItemsService 内部 | ItemsService 内部 | 权限检查 |
| **事件系统** | `emitter.ts` | ItemsService 内部 | ItemsService 内部 | Hooks 触发 |
| **PayloadService** | `services/payload.ts` | ItemsService 内部 | ItemsService 内部 | 关系处理 |
| **ActivityService** | `services/activity.ts` | ItemsService 内部 | ItemsService 内部 | 活动追踪 |
| **RevisionsService** | `services/revisions.ts` | ItemsService 内部 | ItemsService 内部 | 修订追踪 |

#### 5.2.2 GraphQL 独有的组件

| 组件 | 位置 | 说明 |
|-----|------|------|
| **GraphQLService** | `services/graphql/index.ts` | GraphQL 服务封装 |
| **Schema 生成** | `services/graphql/schema/*` | 动态 Schema 生成 |
| **Resolvers** | `services/graphql/resolvers/*` | Query/Mutation 解析器 |
| **Query 转换** | `services/graphql/schema/parse-query.ts` | GraphQL AST → Directus Query |
| **M2A 过滤重写** | `services/graphql/utils/filter-replace-m2a.ts` | M2A 关系过滤格式转换 |
| **System Resolvers** | `services/graphql/resolvers/system.ts` | 系统专用 resolvers |
| **Scope 机制** | N/A | `items`/`system` 双端点 |

#### 5.2.3 REST 独有的组件

| 组件 | 位置 | 说明 |
|-----|------|------|
| **Items Controller** | `controllers/items.ts` | Express 路由绑定 |
| **HTTP 中间件** | `middleware/*` | 认证、缓存、CORS 等 |
| **MetaService** | `services/meta.ts` | REST 响应的 meta 字段 |
| **respond 中间件** | `middleware/respond.ts` | 统一响应格式化 |

### 5.3 Query 对象的统一结构

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
通过 `sanitizeQuery(req.query)` 转换为 Query 对象。

**GraphQL 传递方式**：
```graphql
query {
    articles(filter: { status: { _eq: "published" } }, limit: 10) {
        id
        title
    }
}
```
通过 `getQuery()` 解析 GraphQL AST 转换为 Query 对象。

### 5.4 服务实例化参数的一致性

REST 和 GraphQL 传递给 ItemsService 的参数完全一致：

```typescript
// REST Controller (items.ts)
const service = new ItemsService(req.collection, {
    accountability: req.accountability,  // 用户身份、角色、IP、userAgent 等
    schema: req.schema,                    // SchemaOverview
    knex: req.knex,                        // 数据库连接（可选）
    nested: req.nested,                    // 嵌套关系标记（可选）
});

// GraphQLService (通过 getService)
const service = getService(collection, {
    accountability: this.accountability,   // 相同的 accountability 对象
    schema: this.schema,                    // 相同的 schema 对象
    knex: this.knex,                        // 相同的数据库连接
    // nested: 由 ItemsService 内部处理
});
```

**Accountability 对象结构**：
```typescript
interface Accountability {
    user: string | null;           // 用户 ID
    role: string | null;           // 角色 ID
    roles: string[];               // 所有角色 ID
    admin: boolean;                // 是否管理员
    app: boolean;                  // 是否 App 访问
    ip: string | null;             // IP 地址
    userAgent: string | null;      // User-Agent
    origin: string | null;         // 请求来源
}
```

### 5.5 操作流程对比

#### 5.5.1 读取操作流程对比

**REST 流程**：
```
1. HTTP 请求 → Express 路由
   GET /items/articles?fields=id,title,author.name&filter[status][_eq]=published

2. Controller 处理
   → sanitizeQuery(req.query) → Query 对象
   → new ItemsService(collection, options)
   → service.readByQuery(query)

3. ItemsService 处理
   → emitter.emitFilter('items.query', ...)  // 前置钩子
   → getAstFromQuery()                        // 构建 AST
   → processAst()                              // 权限注入
   → runAst()                                  // 执行查询
   → emitter.emitFilter('items.read', ...)    // 后置钩子
   → emitter.emitAction('items.read', ...)    // 事件触发

4. 响应格式化
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
   → new GraphQLService({ scope: 'items', accountability, schema })
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
   → getService(collection, options) → 获取 ItemsService
   → service.readByQuery(query, { stripNonRequested: false })

6. ItemsService 处理（与 REST 完全相同）
   → emitter.emitFilter('items.query', ...)
   → getAstFromQuery()
   → processAst()
   → runAst()
   → ...

7. GraphQL 响应
   → { data: { articles: [...] }, errors?: [...] }
```

#### 5.5.2 写入操作流程对比

**REST 创建流程**：
```
POST /items/articles
Body: { "title": "New Article", "author": 1 }

1. Controller 处理
   → new ItemsService('articles', options)
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
   → getService('articles', options) → 获取 ItemsService

2. 执行创建
   → service.createOne(args['data'])
   // ItemsService 内部流程与 REST 完全相同

3. 读取结果
   → service.readOne(primaryKey, query)
   → 返回: { id: 1, title: "New Article" }
```

### 5.6 共享机制的核心优势

| 优势 | 说明 |
|-----|------|
| **行为一致性** | REST 和 GraphQL 的权限检查、事件触发、数据验证完全一致 |
| **单点维护** | 业务逻辑只需在 ItemsService 中修改一次 |
| **安全保障** | 权限检查集中在 ItemsService 内部，不易遗漏 |
| **易于扩展** | 新功能只需在服务层实现，自动对 REST 和 GraphQL 生效 |
| **多态性** | 系统服务通过继承扩展功能，REST 和 GraphQL 自动受益 |
| **可测试性** | ItemsService 可以独立测试，无需通过 HTTP 层 |

### 5.7 架构设计原则

1. **单一职责**：
   - Controller/Resolver 只负责协议转换
   - ItemsService 负责业务逻辑
   - 数据层负责持久化

2. **依赖倒置**：
   - REST 和 GraphQL 都依赖于抽象的 ItemsService 接口
   - 具体实现可以替换（如 UsersService 扩展 ItemsService）

3. **共享上下文**：
   - Accountability、Schema、Knex 连接在整个调用链中保持一致
   - 这是实现权限检查、事件追踪的基础

4. **统一数据结构**：
   - Query 对象是 REST 和 GraphQL 的共同语言
   - 任何一方的扩展（如 deep 参数）都需要在 Query 结构中体现

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
| Query 净化 | `api/src/utils/sanitize-query.ts` |
| Query 验证 | `api/src/utils/validate-query.ts` |
| 响应中间件 | `api/src/middleware/respond.ts` |

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

## 七、总结

### 7.1 核心机制回顾

1. **GraphQL Schema 动态生成**：
   - 基于 `graphql-compose` 的 `SchemaComposer`
   - 按 `read/create/update/delete` 四种操作生成独立类型
   - 使用 `mnemonist/LRUMap` 缓存，容量可配置
   - 监听 `schemaChanged` 事件自动清除缓存
   - 通过 `Semaphore` 控制并发生成数

2. **Scope 过滤与 System Resolvers**：
   - `items` scope：用户自定义集合，通用 CRUD
   - `system` scope：系统集合，通过专用 resolvers 访问
   - `SYSTEM_DENY_LIST` 中的集合不通过通用模式访问
   - System Resolvers 提供服务器信息、元数据查询、用户管理等功能

3. **Query 解析与转换**：
   - `parseAliases()`：收集字段别名映射
   - `parseFields()`：递归解析字段，生成 `fields` 数组
   - **Deep 参数**：嵌套字段的别名和参数写入 `query.deep`
   - **M2A 过滤重写**：将 `field__collection` 转换为 `field:collection` 格式

4. **Mutation 写操作链路**：
   - 从字段名解析 `action` 和 `collection`
   - 使用 `getService()` 获取正确的服务实例
   - 调用 `createOne/createMany`, `updateOne/updateMany/updateBatch`, `deleteOne/deleteMany`
   - 单例集合使用 `upsertSingleton`
   - 执行后读取结果返回

### 7.2 共享边界总结

**完全共享**：
- ItemsService 及其子类（UsersService, FilesService 等）
- 权限检查模块（validateAccess, processPayload, processAst）
- 事件系统（emitter.emitFilter/emitAction）
- 活动/修订追踪（ActivityService, RevisionsService）
- Query 净化与验证（sanitizeQuery, validateQuery）
- 关系处理（PayloadService）

**GraphQL 独有**：
- Schema 动态生成与缓存
- Resolvers 层
- GraphQL AST → Directus Query 转换
- M2A 过滤重写
- Scope 机制
- System Resolvers

**REST 独有**：
- Express 路由与控制器
- HTTP 中间件栈
- MetaService（响应元数据）
- respond 中间件

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
   - Controller/Resolver：协议解析、参数验证
   - Service：业务逻辑、权限检查、事务管理
   - PayloadService：关系处理、类型转换
   - Data Layer：数据库操作

### 7.4 修正说明（相对于 V1 版本）

本文档（V2）修正和补充了以下内容：

1. **Schema 缓存实现修正**：
   - V1 错误描述为 `lru-cache` 库
   - 实际使用的是 `mnemonist` 库的 `LRUMap`
   - 补充了缓存容量配置（`GRAPHQL_SCHEMA_CACHE_CAPACITY`）、并发控制（`Semaphore`）、`schemaChanged` 事件监听

2. **Scope 过滤机制补充**：
   - 详细说明了 `items` 和 `system` 两个 scope 的区别
   - 补充了 `scopeFilter` 函数的实现逻辑
   - 说明了 `SYSTEM_DENY_LIST` 的作用

3. **System Resolver 注入补充**：
   - 详细说明了 `injectSystemResolvers()` 的功能
   - 补充了服务器信息查询（`server_info`, `server_ping`, `server_health`）
   - 补充了元数据查询（`collections`, `fields`, `relations`）
   - 补充了当前用户相关查询（`users_me`, `permissions_me`, `roles_me`）
   - 补充了系统专用 Mutation（`update_users_me`, `import_file`, `users_invite`）

4. **Mutation 写操作链路补充**：
   - 详细说明了 `resolveMutation()` 的完整流程
   - 补充了 Mutation 字段命名规范表格
   - 补充了 `GraphQLService.upsertSingleton()` 的实现
   - 详细说明了从字段名解析 action 和 collection 的逻辑

5. **Parse-Query 关键点补充**：
   - 详细说明了 `parseAliases()` 如何收集别名映射
   - 详细说明了 `parseFields()` 如何递归解析字段并处理嵌套关系
   - 补充了 **Deep 参数** 的生成机制：嵌套字段的别名和参数写入 `query.deep`
   - 详细说明了 **M2A 过滤重写** 机制：
     - `filterReplaceM2A()`：将 `field__collection` 转换为 `field:collection`
     - `filterReplaceM2ADeep()`：处理 deep 参数中的 M2A 关系
   - 补充了完整的代码示例和使用场景

6. **共享边界完整说明**：
   - 补充了完整的架构图，标明各层职责
   - 补充了详细的共享组件表格，标明各组件的位置和使用方式
   - 补充了 GraphQL 独有组件和 REST 独有组件的对比
   - 补充了完整的 `Query` 对象结构定义
   - 补充了 `Accountability` 对象结构
   - 补充了详细的读取操作流程对比（REST vs GraphQL）
   - 补充了详细的写入操作流程对比（REST vs GraphQL）
   - 补充了共享机制的核心优势表格
   - 补充了架构设计原则说明

---

## 八、附录

### A. 环境变量参考

| 环境变量 | 默认值 | 说明 |
|---------|-------|------|
| `GRAPHQL_SCHEMA_CACHE_CAPACITY` | 100 | Schema 缓存容量（LRUMap 大小） |
| `GRAPHQL_SCHEMA_GENERATION_MAX_CONCURRENT` | 5 | 并发生成 Schema 的最大并发数 |
| `GRAPHQL_INTROSPECTION` | true | 是否允许 GraphQL introspection |
| `QUERY_LIMIT_DEFAULT` | 100 | 默认查询限制 |
| `QUERY_LIMIT_MAX` | -1 | 最大查询限制（-1 表示无限制） |

### B. 关键类型定义

```typescript
// GQLScope - GraphQL 端点类型
type GQLScope = 'items' | 'system';

// AbstractServiceOptions - 服务实例化选项
interface AbstractServiceOptions {
    knex?: Knex;
    accountability?: Accountability | null;
    schema: SchemaOverview;
    nested?: string[];
}

// Query - 统一查询对象
interface Query {
    fields?: string[];
    filter?: Filter;
    sort?: string[];
    limit?: number;
    offset?: number;
    page?: number;
    search?: string;
    group?: string[];
    aggregate?: Aggregate;
    deep?: DeepQuery;
    alias?: Record<string, string>;
    version?: string;
    versionRaw?: boolean;
    export?: 'json' | 'csv';
}

// DeepQuery - 嵌套查询参数
interface DeepQuery {
    [field: string]: NestedDeepQuery | any;
    _alias?: Record<string, string>;
    _filter?: Filter;
    _sort?: string[];
    _limit?: number;
    _offset?: number;
    _page?: number;
    _search?: string;
}
```

### C. 系统集合服务继承关系

```
                    ┌─────────────────────────────┐
                    │      ItemsService (基类)     │
                    │  api/src/services/items.ts   │
                    └───────────────┬─────────────┘
                                    │
            ┌───────────┬───────────┼───────────┬───────────┐
            │           │           │           │           │
            ▼           ▼           ▼           ▼           ▼
    ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐
    │UsersService│ │FilesService│ │RolesService│ │...更多    │ │ 自定义集合  │
    │  密码哈希  │ │  文件处理  │ │  权限关联  │ │ 系统服务  │ │直接使用   │
    │  TFA处理   │ │  图片处理  │ │           │ │           │ │ItemsService│
    │  邀请用户  │ │  导入文件  │ │           │ │           │ │           │
    └───────────┘ └───────────┘ └───────────┘ └───────────┘ └───────────┘
```

