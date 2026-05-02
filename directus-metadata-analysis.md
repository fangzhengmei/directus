# Directus 元数据驱动架构分析报告

## 1. 概述

Directus 是一个基于元数据驱动的开源数据管理平台，管理员通过界面定义的数据集合（Collections和字段元数据，能够自动驱动 REST 和 GraphQL 接口的动态生成，并指导前端界面的渲染。本文档深入分析这一机制的实现原理。

---

## 2. 元数据存储结构

### 2.1 核心元数据表

Directus 的元数据主要存储在三个核心系统表中：

| 表名 | 用途 | 主要字段 |
|------|------|----------|
| `directus_collections` | 存储集合元数据 | collection, singleton, note, sort_field, accountability, group |
| `directus_fields` | 存储字段元数据 | id, collection, field, special, note, validation, searchable, interface, options, width, group, sort, readonly, hidden, display, display_options, translations, required, conditions |
| `directus_relations` | 存储关系元数据 | id, collection, field, related_collection, meta (包含 one_field, one_collection_field, one_allowed_collections, sort_field, junction_field 等) |

### 2.2 集合元数据结构

集合元数据在 `directus_collections` 表中存储，核心字段包括：

- **collection**: 集合名称（主键）
- **singleton**: 是否为单例模式
- **note**: 集合描述
- **sort_field**: 排序字段
- **accountability**: 责任追踪模式（'all', 'activity', null）
- **group**: 所属分组

### 2.3 字段元数据结构

字段元数据在 `directus_fields` 表中存储，核心字段包括：

- **collection**: 所属集合
- **field**: 字段名称
- **special**: 特殊类型标识（如 'alias', 'o2m', 'm2o', 'm2m', 'a2o', 'file', 'files', 'group', 'translations', 'presentation', 'no-data' 等）
- **interface**: 前端使用的界面组件类型
- **options**: 界面组件配置选项
- **display**: 显示组件类型
- **display_options**: 显示组件配置
- **validation**: 验证规则
- **readonly/hidden**: 界面控制属性
- **group**: 所属字段分组
- **sort**: 排序顺序
- **required**: 是否必填
- **conditions**: 条件显示规则

### 2.4 服务层访问

#### CollectionsService (`api/src/services/collections.ts`) 提供集合元数据的 CRUD 操作：

```typescript
// 创建集合时同时处理：
// 1. 创建实际数据库表
// 2. 插入 directus_collections 记录
// 3. 插入 directus_fields 记录
// 4. 处理字段关系
```

#### FieldsService (`api/src/services/fields.ts`) 提供字段元数据的 CRUD 操作：

```typescript
// 创建字段时同时处理：
// 1. 添加/修改数据库表列
// 2. 插入/更新 directus_fields 记录
// 3. 处理关系元数据
// 4. 处理权限、活动追踪
```

---

## 3. 运行时 Schema 生成机制

### 3.1 核心函数：getSchema()

**文件位置：`api/src/utils/get-schema.ts`

`getSchema()` 是运行时获取元数据 schema 的核心函数，返回 `SchemaOverview` 对象。

#### 3.1.1 缓存机制

```typescript
// 缓存策略
export async function getSchema(
    options?: {
        database?: Knex;
        bypassCache?: boolean;  // 绕过缓存
    },
    attempt = 0,
): Promise<SchemaOverview> {
    // 1. 检查是否绕过缓存或缓存未启用
    if (options?.bypassCache || env['CACHE_SCHEMA'] === false) {
        return await getDatabaseSchema(database, schemaInspector);
    }

    // 2. 检查内存缓存
    const cached = getMemorySchemaCache();
    if (cached) return cached;

    // 3. 多进程同步机制（使用锁和消息总线）
    const lock = useLock();
    const bus = useBus();
    
    // ... 分布式锁确保只有一个进程生成 schema
    // 其他进程等待并订阅完成消息
}
```

#### 3.1.2 Schema 构建流程

`getDatabaseSchema()` 函数实际构建 schema：

```typescript
async function getDatabaseSchema(database: Knex, schemaInspector: SchemaInspector): Promise<SchemaOverview> {
    const result: SchemaOverview = {
        collections: {},
        relations: [],
    };

    // 步骤 1: 获取数据库表结构概览
    const schemaOverview = await schemaInspector.overview();

    // 步骤 2: 从 directus_collections 获取集合元数据
    const collections = [
        ...(await database.select(...).from('directus_collections')),
        ...systemCollectionRows,  // 系统集合
    ];

    // 步骤 3: 构建集合信息
    for (const [collection, info] of Object.entries(schemaOverview)) {
        // 过滤排除的表、无主键表、含空格的表名
        if (toArray(env['DB_EXCLUDE_TABLES']).includes(collection)
            || !info.primary
            || collection.includes(' ')) {
            continue;
        }

        result.collections[collection] = {
            collection,
            primary: info.primary,
            singleton: toBoolean(collectionMeta?.singleton),
            note: collectionMeta?.note || null,
            sortField: collectionMeta?.sort_field || null,
            accountability: collectionMeta ? collectionMeta.accountability : 'all',
            fields: mapValues(schemaOverview[collection]?.columns, (column) => {
                return {
                    field: column.column_name,
                    defaultValue: getDefaultValue(column) ?? null,
                    nullable: column.is_nullable ?? true,
                    generated: column.is_generated ?? false,
                    type: getLocalType(column),  // 映射到 Directus 类型
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

    // 步骤 4: 从 directus_fields 增强字段元数据
    const fields = [
        ...(await database.select(...).from('directus_fields')),
        ...systemFieldRows,
    ];

    for (const field of fields) {
        if (!result.collections[field.collection]) continue;
        
        const special = field.special ? toArray(field.special) : [];
        
        // 合并数据库结构和元数据
        result.collections[field.collection]!.fields[field.field] = {
            ...existing,
            special: special,
            note: field.note,
            alias: existing?.alias ?? true,  // alias 字段
            validation: (validation as Filter) ?? null,
            searchable: toBoolean(field.searchable) ?? true,
        };
    }

    // 步骤 5: 获取关系元数据
    const relationsService = new RelationsService({ knex: database, schema: result });
    result.relations = await relationsService.readAll(undefined, undefined, true);

    return result;
}
```

### 3.2 SchemaOverview 结构

```typescript
interface SchemaOverview {
    collections: {
        [collection: string]: {
            collection: string;
            primary: string;           // 主键字段名
            singleton: boolean;         // 是否单例
            note: string | null;
            sortField: string | null;
            accountability: string | null;
            fields: {
                [field: string]: {
                    field: string;
                    defaultValue: any;
                    nullable: boolean;
                    generated: boolean;
                    type: Type;               // Directus 类型（string, integer, uuid, etc.
                    dbType: string | null;  // 数据库原生类型
                    precision: number | null;
                    scale: number | null;
                    special: string[];           // special 标识
                    note: string | null;
                    validation: Filter | null;
                    alias: boolean;
                    searchable: boolean;
                };
            };
        };
    };
    relations: Relation[];  // 关系定义
}
```

### 3.3 请求级别的 Schema 注入

**文件位置：`api/src/middleware/schema.ts`

```typescript
const schema: RequestHandler = asyncHandler(async (req, _res, next) => {
    req.schema = await getSchema();
    return next();
});
```

在 `app.ts` 中，schema 中间件在路由之前加载：

```typescript
app.use(schema);  // 每个请求都注入 schema
app.use('/items', itemsRouter);
app.use('/graphql', graphqlRouter);
```

---

## 4. REST 接口动态生成机制

### 4.1 路由设计

**文件位置：`api/src/app.ts`

REST 接口采用**通用路由模式**，而非为每个集合生成独立路由：

```typescript
// 注册固定路由
app.use('/items', itemsRouter);        // 核心数据操作路由
app.use('/collections', collectionsRouter);  // 集合元数据路由
app.use('/fields', fieldsRouter);      // 字段元数据路由
app.use('/relations', relationsRouter);  // 关系元数据路由
```

### 4.2 Items 路由实现

**文件位置：`api/src/controllers/items.ts`**

路由使用 URL 参数 `:collection` 动态指定集合：

```typescript
// 读取多个项目
router.get(
    '/:collection',
    asyncHandler(async (req, res) => {
        const service = new ItemsService(req.params.collection, {
            accountability: req.accountability,
            schema: req.schema,
        });

        const result = await service.readByQuery(req.sanitizedQuery);
        return res.json({ data: result });
    }),
);

// 读取单个项目
router.get('/:collection/:id', ...);

// 创建项目
router.post('/:collection', ...);

// 更新项目
router.patch('/:collection/:id', ...);
router.patch('/:collection', ...);

// 删除项目
router.delete('/:collection/:id', ...);
router.delete('/:collection', ...);
```

### 4.3 ItemsService 通用实现

**文件位置：`api/src/services/items.ts`

`ItemsService` 是核心的通用数据服务，通过构造参数 `collection` 动态处理不同集合：

```typescript
export class ItemsService<Item extends AnyItem = AnyItem, Collection extends string = string> {
    collection: Collection;
    knex: Knex;
    schema: SchemaOverview;

    constructor(collection: Collection, options: AbstractServiceOptions) {
        this.collection = collection;
        this.knex = options.knex || getDatabase();
        this.schema = options.schema;
        // ...
    }

    // 核心操作方法均基于 this.collection 动态执行

    async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
        // 1. 生成 AST（抽象语法树）
        let ast = await getAstFromQuery(
            {
                collection: this.collection,  // 动态集合
                query: updatedQuery,
                accountability: this.accountability,
            },
            { schema: this.schema, knex: this.knex },
        );

        // 2. 权限处理
        ast = await processAst(
            { ast, action: 'read', accountability: this.accountability },
            { knex: this.knex, schema: this.schema },
        );

        // 3. 执行查询
        const records = await runAst(ast, this.schema, this.accountability, {...});
        
        return filteredRecords as Item[];
    }

    async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
        const primaryKeyField = this.schema.collections[this.collection]!.primary;
        const fields = Object.keys(this.schema.collections[this.collection]!.fields);
        
        // 基于 schema 进行类型转换、关系处理、权限检查
        // ...
        
        // 动态插入到 this.collection 表
        const result = await trx
            .insert(payloadWithoutAliases)
            .into(this.collection)  // 动态表名
            .returning(primaryKeyField, returningOptions);
    }
}
```

### 4.4 查询处理流程

1. **getAstFromQuery** (`api/src/database/get-ast-from-query/`)
   - 解析 HTTP 查询参数（fields, filter, sort, limit, offset 等）
   - 基于 schema 验证字段有效性
   - 处理关系嵌套查询

2. **processAst** (`api/src/permissions/modules/process-ast/`)
   - 应用权限检查
   - 过滤无权访问的字段和记录

3. **runAst** (`api/src/database/run-ast/`)
   - 将 AST 转换为 Knex 查询
   - 执行数据库操作
   - 处理关系数据加载

---

## 5. GraphQL 接口动态生成机制

### 5.1 核心函数：generateSchema()

**文件位置：`api/src/services/graphql/schema/index.ts`

GraphQL schema 是**完全动态生成**的，基于当前的元数据 schema：

```typescript
export async function generateSchema(
    gql: GraphQLService,
    type: 'schema' | 'sdl' = 'schema',
): Promise<GraphQLSchema | string> {
    const key = `${gql.scope}_${type}_${gql.accountability?.role}_${gql.accountability?.user}`;

    // 1. 检查缓存
    const cachedSchema = cache.get(key);
    if (cachedSchema) return cachedSchema;

    // 2. 使用信号量控制并发
    return semaphore.runExclusive(async () => {
        const schemaComposer = new SchemaComposer<GraphQLParams['contextValue']>();

        // 3. 基于权限过滤 schema
        let schema: Schema;
        const sanitizedSchema = sanitizeGraphqlSchema(gql.schema);

        if (!gql.accountability || gql.accountability.admin) {
            // 管理员：完整 schema
            schema = {
                read: sanitizedSchema,
                create: sanitizedSchema,
                update: sanitizedSchema,
                delete: sanitizedSchema,
            };
        } else {
            // 普通用户：基于权限缩减 schema
            schema = {
                read: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap(
                    { accountability: gql.accountability, action: 'read' },
                    { schema: gql.schema, knex: gql.knex },
                )),
                create: reduceSchema(sanitizedSchema, await fetchAllowedFieldMap(
                    { accountability: gql.accountability, action: 'create' },
                    { schema: gql.schema, knex: gql.knex },
                )),
                // update, delete 同理
            };
        }

        // 4. 生成可读类型定义
        const { ReadCollectionTypes, VersionCollectionTypes } = await getReadableTypes(
            gql, schemaComposer, schema, inconsistentFields);

        const { CreateCollectionTypes, UpdateCollectionTypes, DeleteCollectionTypes } = getWritableTypes(
            gql, schemaComposer, schema, inconsistentFields, ReadCollectionTypes);

        // 5. 生成查询字段
        const readableCollections = Object.values(schema.read.collections)
            .filter((collection) => collection.collection in ReadCollectionTypes)
            .filter(scopeFilter);

        if (readableCollections.length > 0) {
            schemaComposer.Query.addFields(
                readableCollections.reduce(
                    (acc, collection) => {
                        const collectionName = gql.scope === 'items' 
                            ? collection.collection 
                            : collection.collection.substring(9);
                        
                        // 为每个集合添加查询字段
                        acc[collectionName] = ReadCollectionTypes[collection.collection]!
                            .getResolver(collection.collection);

                        if (gql.schema.collections[collection.collection]!.singleton === false) {
                            // 非单例集合添加 by_id 和 aggregated 查询
                            acc[`${collectionName}_by_id`] = ReadCollectionTypes[collection.collection]!
                                .getResolver(`${collection.collection}_by_id`);
                            acc[`${collectionName}_aggregated`] = ReadCollectionTypes[collection.collection]!
                                .getResolver(`${collection.collection}_aggregated`);
                        }

                        if (gql.scope === 'items') {
                            // 版本查询
                            acc[`${collectionName}_by_version`] = VersionCollectionTypes[collection.collection]!
                                .getResolver(`${collection.collection}_by_version`);
                        }

                        return acc;
                    },
                    {} as ObjectTypeComposerFieldConfigAsObjectDefinition<any, any>,
                ),
            );
        }

        // 6. 生成变更字段
        if (Object.keys(schema.create.collections).length > 0) {
            schemaComposer.Mutation.addFields(
                Object.values(schema.create.collections)
                    .filter((collection) => collection.collection in CreateCollectionTypes)
                    .filter(scopeFilter)
                    .filter((collection) => READ_ONLY.includes(collection.collection) === false)
                    .reduce(
                        (acc, collection) => {
                            const collectionName = gql.scope === 'items' 
                                ? collection.collection 
                                : collection.collection.substring(9);

                            acc[`create_${collectionName}_items`] = CreateCollectionTypes[collection.collection]!
                                .getResolver(`create_${collection.collection}_items`);
                            acc[`create_${collectionName}_item`] = CreateCollectionTypes[collection.collection]!
                                .getResolver(`create_${collection.collection}_item`);

                            return acc;
                        },
                        {} as ObjectTypeComposerFieldConfigAsObjectDefinition<any, any>,
                    ),
            );
        }

        // 7. 构建并缓存 schema
        const gqlSchema = schemaComposer.buildSchema();
        cache.set(key, gqlSchema);
        return gqlSchema;
    });
}
```

### 5.2 类型生成

**文件位置：`api/src/services/graphql/schema/read.ts` 和 `write.ts`**

#### 类型生成流程：

1. **字段类型映射**：
   ```typescript
   // Directus 类型 -> GraphQL 类型
   // string -> GraphQLString
   // integer -> GraphQLInt
   // bigInteger -> GraphQLFloat / GraphQLID
   // float, decimal -> GraphQLFloat
   // boolean -> GraphQLBoolean
   // date, dateTime, timestamp -> GraphQLDateTime (自定义标量)
   // json -> GraphQLJSON (自定义标量)
   // uuid -> GraphQLID
   ```

2. **关系字段处理**：
   - **m2o (Many-to-One)：嵌套对象类型
   - **o2m (One-to-Many)：列表类型，支持过滤、分页参数
   - **m2m (Many-to-Many)：列表类型，中间表处理
   - **a2o (Any-to-One)：联合类型

3. **解析器实现**：
   ```typescript
   // 查询解析器调用 ItemsService
   resolver = {
       resolve: async (source, args, context, info) => {
           const service = new ItemsService(collection, {
               accountability: context.accountability,
               schema: context.schema,
           });
           return service.readByQuery(transformGqlQueryToDirectusQuery(args));
       }
   }
   ```

### 5.3 Schema 缓存机制

**文件位置：`api/src/services/graphql/schema-cache.ts`

```typescript
import Keyv 缓存，TTL 控制：
- 按角色 + 用户 组合键
- 系统缓存清除时自动失效

// 集合/字段变更时触发缓存清除：
// 在 CollectionsService 和 FieldsService 中：
if (opts?.autoPurgeSystemCache !== false) {
    await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });
}
```

---

## 6. 前端界面元数据驱动渲染

### 6.1 数据获取与存储

#### 6.1.1 Stores 层

**文件位置：`app/src/stores/`

- **collections Store** (`app/src/stores/collections.ts`：存储集合元数据
- **fields Store** (`app/src/stores/fields.ts`：存储字段元数据
- **relations Store** (`app/src/stores/relations.ts`：存储关系元数据

```typescript
// 使用方式
const collectionsStore = useCollectionsStore();
const fieldsStore = useFieldsStore();

// 获取集合的字段
const fields = fieldsStore.getFieldsForCollection('articles');

// 获取主键字段
const primaryKey = fieldsStore.getPrimaryKeyFieldForCollection('articles');
```

#### 6.1.2 Schema 组合式函数

**文件位置：`app/src/composables/use-schema.ts`

```typescript
export function useSchemaOverview(): ComputedRef<SchemaOverview> {
    const relationsStore = useRelationsStore();
    const fieldsStore = useFieldsStore();
    const collectionsStore = useCollectionsStore();

    return computed(() => ({
        collections: Object.fromEntries(
            collectionsStore.collections.map((collection) => {
                const fields = fieldsStore.getFieldsForCollection(collection.collection)
                    .map((field) => [
                        field.field,
                        {
                            field: field.field,
                            defaultValue: field.schema?.default_value,
                            nullable: field.schema?.is_nullable ?? false,
                            type: field.type,
                            dbType: field.schema?.data_type ?? null,
                            alias: false,
                            searchable: field.meta?.searchable ?? true,
                            note: field.meta?.note ?? null,
                            precision: field.schema?.numeric_precision ?? null,
                            scale: field.schema?.numeric_scale ?? null,
                            special: field.meta?.special ?? [],
                            validation: field.meta?.validation ?? null,
                        } satisfies FieldOverview,
                    ]);

                const collectionInfo: CollectionOverview = {
                    collection: collection.collection,
                    fields: Object.fromEntries(fields),
                    accountability: collection.meta?.accountability ?? null,
                    note: collection.meta?.note ?? null,
                    primary: fieldsStore.getPrimaryKeyFieldForCollection(collection.collection)!.field,
                    singleton: collection.meta?.singleton ?? false,
                    sortField: collection.meta?.sort_field ?? null,
                };

                return [collection.collection, collectionInfo];
            }),
        ),
        relations: relationsStore.relations,
    }));
}
```

### 6.2 动态表单渲染

#### 6.2.1 VForm 组件

**文件位置：`app/src/components/v-form/v-form.vue`**

这是核心的动态表单组件：

```vue
<script setup lang="ts">
const props = withDefaults(
    defineProps<{
        collection?: string;    // 集合名
        fields?: Field[];         // 或直接传字段定义
        initialValues?: FieldValues | null;
        modelValue?: FieldValues | null;
        // ...
    }>(),
    { ... },
);

const fieldsStore = useFieldsStore();

// 根据 collection 或 fields 获取字段定义
const fieldDefinitions = computed<Field[]>(() => {
    if (props.collection) {
        return fieldsStore.getFieldsForCollection(props.collection);
    }
    if (props.fields) {
        return props.fields;
    }
    return [];
});

// 表单逻辑...
</script>

<template>
    <div ref="el" :class="['v-form', gridClass]">
        <template v-for="(fieldName, index) in fieldNames" :key="fieldName">
            <template v-if="fieldsMap[fieldName]">
                <!-- 分组字段：动态渲染 interface-${field.meta.interface 指定的组件 -->
                <component
                    :is="`interface-${fieldsMap[fieldName]!.meta?.interface || 'group-standard'}`"
                    v-if="fieldsMap[fieldName]!.meta?.special?.includes('group')"
                    :field="fieldsMap[fieldName]"
                    :fields="fieldsForGroup[index] || []"
                    :values="modelValue || {}"
                    :disabled="disabled || nonEditable"
                    v-bind="fieldsMap[fieldName]!.meta?.options || {}"
                    @apply="apply"
                />

                <!-- 普通字段：渲染 FormField 组件 -->
                <FormField
                    v-else-if="isFieldVisible(fieldsMap[fieldName])"
                    :field="fieldsMap[fieldName]!"
                    :model-value="(values || {})[fieldName]"
                    :disabled="isDisabled(fieldsMap[fieldName]!) || nonEditable"
                    @update:model-value="setValue(fieldName, $event)"
                />
            </template>
        </template>
    </div>
</template>
```

#### 6.2.2 FormField 组件

**文件位置：`app/src/components/v-form/components/form-field.vue`**

根据字段的 `interface` 属性动态选择界面组件：

```typescript
// 核心逻辑：根据 field.meta.interface 加载对应组件

// 例如：
// interface: 'input' -> 渲染 interface-input
// interface: 'select-dropdown' ->渲染 interface-select-dropdown
// interface: 'datetime' ->渲染 interface-datetime
// interface: 'file' ->渲染 interface-file

// 组件注册位置：app/src/interfaces/
```

### 6.3 配置页面实现

#### 6.3.1 集合配置页面

**文件位置：`app/src/modules/settings/routes/data-model/collections/collections.vue`

```vue
<template>
    <!-- 集合列表 -->
    <v-list>
        <CollectionItem
            v-for="collection in collections"
            :key="collection.collection"
            :collection="collection"
            @click="goToCollection(collection.collection)"
        />
    </v-list>

    <!-- 新建/编辑集合表单 -->
    <v-dialog v-model="showModal" persistent>
        <v-form
            :fields="collectionFormFields"
            v-model="collectionForm"
            @submit="saveCollection"
        />
    </v-dialog>
</template>
```

#### 6.3.2 字段配置页面

**文件位置：`app/src/modules/settings/routes/data-model/fields/fields.vue`

```vue
<template>
    <!-- 字段列表（可拖拽排序） -->
    <draggable
        v-model="fields"
        item-key="field"
        handle=".handle"
    >
        <template #item="{ element: field, index }">
            <fields-management
                :field="field"
                :collection="collection"
                @edit="editField(field.field)"
                @delete="deleteField(field.field)"
            />
        </template>
    </draggable>
</template>
```

#### 6.3.3 字段详情配置页面

**文件位置：`app/src/modules/settings/routes/data-model/field-detail/field-detail.vue`

这是最复杂的配置页面，动态渲染字段配置选项：

```vue
<template>
    <div class="field-detail">
        <!-- 基础配置区 -->
        <field-detail-simple
            :collection="collectionName"
            :field="editingField"
            @update:field="updateField($event)"
        />

        <!-- 高级配置区（根据字段类型动态显示） -->
        <field-detail-advanced
            v-if="showAdvanced"
            :collection="collectionName"
            :field="editingField"
            @update:field="updateField($event)"
        />

        <!-- 界面配置区（根据 interface 动态显示） -->
        <field-detail-interface
            :collection="collectionName"
            :field="editingField"
            @update:field="updateField($event)"
        />

        <!-- 显示配置区（根据 display 动态显示） -->
        <field-detail-display
            :collection="collectionName"
            :field="editingField"
            @update:field="updateField($event)"
        />

        <!-- 验证配置区 -->
        <field-detail-validation
            :collection="collectionName"
            :field="editingField"
            @update:field="updateField($event)"
        />
    </div>
</template>
```

### 6.4 Interface 组件系统

**文件位置：`app/src/interfaces/`

每个 interface 是独立的 Vue 组件，包含：

- **index.ts**：组件注册、配置
- **interface-xxx.vue**：编辑界面组件
- **display-xxx.vue**：（可选）显示组件
- **options.vue**：配置选项组件

#### 示例：`app/src/interfaces/input/

```typescript
// index.ts
import { defineInterface } from '@directus/extensions';
import InterfaceInput from './interface-input.vue';
import options from './options.js';

export default defineInterface({
    id: 'input',
    name: '$t:interfaces.input',
    icon: 'input',
    description: '$t:interfaces.input.description',
    component: InterfaceInput,
    options: options,
    types: ['string', 'text', 'uuid'],
    recommendedDisplays: ['formatted-value'],
});
```

---

## 7. 完整数据流图

### 7.1 元数据变更流程

```
管理员在前端修改集合/字段
        ↓
前端调用 API (POST/PATCH /collections 或 /fields)
        ↓
CollectionsService / FieldsService 执行：
  1. 修改数据库表结构 (DDL)
  2. 更新 directus_collections / directus_fields 记录
  3. 处理关系元数据
  4. 清除系统缓存 (systemCache)
        ↓
下次请求时：
  - getSchema() 重新生成 schema
  - generateSchema() 重新生成 GraphQL schema
  - 前端重新获取元数据
```

### 7.2 运行时数据访问流程

```
REST 请求: GET /items/articles?fields=id,title,author.*
        ↓
itemsRouter 路由处理
        ↓
schema 中间件注入 req.schema
        ↓
ItemsService('articles') 实例化
        ↓
readByQuery() 执行：
  1. getAstFromQuery() - 解析查询参数，验证字段
  2. processAst() - 权限检查
  3. runAst() - 执行数据库查询，加载关系
        ↓
返回 JSON 响应
```

### 7.3 GraphQL 执行流程

```
GraphQL 查询:
  query {
    articles(filter: { status: { _eq: "published" } }) {
        id
        title
        author { name }
    }
  }
        ↓
graphqlRouter 路由
        ↓
GraphQLService 生成/获取缓存的 schema
        ↓
graphql-compose 解析查询
        ↓
Resolver 执行：
  - 调用 ItemsService.readByQuery()
  - 处理嵌套关系字段
        ↓
返回 GraphQL 响应
```

---

## 8. 关键设计要点

### 8.1 缓存策略

| 缓存类型 | 存储位置 | 失效时机 | 说明 |
|----------|----------|----------|------|
| Schema 缓存 | 内存 (getMemorySchemaCache) | 集合/字段变更时 | 核心元数据 schema |
| GraphQL Schema 缓存 | Keyv (可配置 Redis) | 系统缓存清除时 | 按角色/用户缓存 |
| 数据缓存 | Keyv (可配置 Redis) | 数据变更时 | 可选，API 响应缓存 |

### 8.2 多进程同步

Directus 支持多实例部署，通过以下机制同步：

1. **锁机制** (`useLock`)：使用数据库或 Redis 分布式锁
2. **消息总线** (`useBus`)：使用 Redis Pub/Sub 或数据库通知
3. **Schema 同步**：一个进程生成 schema，其他进程等待并订阅

```typescript
// get-schema.ts 中的同步逻辑
const lock = useLock();
const bus = useBus();

const lockKey = 'schemaCache--preparing';
const messageKey = 'schemaCache--done';

const processId = await lock.increment(lockKey);

if (processId === 1) {
    // 第一个进程：生成 schema
    schema = await getDatabaseSchema(database, schemaInspector);
    setMemorySchemaCache(schema);
    await bus.publish(messageKey, { schema });
} else {
    // 其他进程：等待消息
    bus.subscribe(messageKey, (options) => {
        setMemorySchemaCache(options.schema);
        resolve(options.schema);
    });
}
```

### 8.3 类型系统映射

Directus 实现了三层类型系统：

1. **数据库层类型**：MySQL/PostgreSQL/SQLite 等原生类型
2. **Directus 层类型**：string, integer, bigInteger, float, decimal, boolean, date, dateTime, timestamp, json, csv, hash, uuid, geometry 等
3. **GraphQL 层类型**：GraphQLString, GraphQLInt, GraphQLFloat, GraphQLBoolean, GraphQLID, 自定义标量 (DateTime, JSON 等)

关键转换函数：

- **getLocalType** (`api/src/utils/get-local-type.ts`)：数据库类型 → Directus 类型
- **getReadableTypes/getWritableTypes** (`api/src/services/graphql/schema/`)：Directus 类型 → GraphQL 类型

### 8.4 权限集成

权限系统与元数据驱动深度集成：

1. **Schema 级别**：`fetchAllowedFieldMap` 获取用户可读/可写字段
2. **Schema 缩减**：`reduceSchema` 根据权限过滤 schema
3. **运行时检查**：`processAst` 和 `processPayload` 在查询/变更时检查

---

## 9. 核心代码文件索引

| 功能模块 | 文件路径 | 说明 |
|----------|----------|------|
| 集合服务 | `api/src/services/collections.ts` | 集合元数据 CRUD |
| 字段服务 | `api/src/services/fields.ts` | 字段元数据 CRUD |
| Schema 获取 | `api/src/utils/get-schema.ts` | 运行时 schema 生成 |
| Schema 中间件 | `api/src/middleware/schema.ts` | 请求级 schema 注入 |
| Items 服务 | `api/src/services/items.ts` | 通用数据操作服务 |
| Items 路由 | `api/src/controllers/items.ts` | REST 数据路由 |
| GraphQL Schema | `api/src/services/graphql/schema/index.ts` | GraphQL schema 生成 |
| GraphQL 路由 | `api/src/controllers/graphql.ts` | GraphQL 路由 |
| 应用入口 | `api/src/app.ts` | 路由注册和中间件 |
| 前端 Schema | `app/src/composables/use-schema.ts` | 前端 schema 组合式函数 |
| 动态表单 | `app/src/components/v-form/v-form.vue` | 核心动态表单组件 |
| 字段配置 | `app/src/modules/settings/routes/data-model/` | 数据模型配置页面 |

---

## 10. 总结

Directus 的元数据驱动架构是一个精心设计的分层系统：

1. **存储层**：使用 `directus_collections`、`directus_fields`、`directus_relations` 三张核心表存储元数据，与实际业务数据完全分离。

2. **运行时层**：
   - `getSchema()` 函数从数据库读取元数据并构建统一的 `SchemaOverview` 对象
   - 采用多级缓存和多进程同步机制保证性能
   - Schema 在请求前通过中间件注入，服务层依赖注入

3. **接口层**：
   - **REST**：采用通用路由模式，`:collection` URL 参数动态指定集合，`ItemsService` 基于 schema 处理不同集合的数据操作
   - **GraphQL**：完全动态生成 schema，为每个集合生成对应的类型、查询和变更，支持权限过滤

4. **前端层**：
   - 使用 Pinia stores 管理元数据状态
   - `v-form` 组件根据字段定义动态渲染表单
   - 根据 `field.meta.interface` 动态选择界面组件
   - 配置页面本身也是元数据驱动的

这种架构的核心优势：

- **高度灵活**：无需代码修改即可通过界面定义数据模型
- **即时生效**：元数据变更后缓存失效，下次请求自动使用新 schema
- **权限感知**：schema 生成时就考虑权限，不同角色看到不同接口
- **多数据库支持**：通过 Knex 和 schema inspector 抽象不同数据库差异
