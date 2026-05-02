# Directus 元数据驱动接口生成深度分析

## 1. 概述

本文档深入分析 Directus 中元数据驱动接口生成的两条核心链路：

1. **关系字段权限裁剪链路**：关系字段（m2o, o2m, m2m, a2o）如何在查询和变更操作中参与权限检查和数据过滤

2. **配置改动回写与 Schema 更新链路**：前端配置页面的改动如何回写到数据库，并触发各级缓存失效和 Schema 重新生成

---

## 2. 关系字段权限裁剪机制

### 2.1 关系类型概览

Directus 支持四种核心关系类型：

| 关系类型 | 标识 | 存储位置 | 典型场景 |
|----------|------|----------|----------|
| Many-to-One (M2O) | `m2o` | 多端表的外键字段 | 文章 → 分类 |
| One-to-Many (O2M) | `o2m` | 直接关系元数据 (one_field) | 分类 → 多篇文章 |
| Many-to-Many (M2M) | `m2m` | 中间连接表 | 文章 ↔ 标签 |
| Any-to-One (A2O) | `a2o` | 多端表 + 关系元数据 | 评论 → 可关联任意内容 |

### 2.2 权限检查的两个层次

权限检查发生在三个关键层次：

#### 层次一：Schema 级别过滤（GraphQL 专用）

**文件位置：`api/src/utils/reduce-schema.ts`

在生成 GraphQL Schema 时，根据权限对 Schema 进行裁剪：

```typescript
export function reduceSchema(schema: SchemaOverview, fieldMap: FieldMap): SchemaOverview {
    const reduced: SchemaOverview = {
        collections: {},
        relations: [],
    };

    // 1. 过滤无权访问的集合
    for (const [collectionName, collection] of Object.entries(schema.collections)) {
        if (!fieldMap[collectionName]) {
            continue;  // 集合完全无权访问
        }

        // 2. 过滤无权访问的字段
        const fields: SchemaOverview['collections'][string]['fields'] = {};

        for (const [fieldName, field] of Object.entries(schema.collections[collectionName]!.fields)) {
            if (!fieldMap[collectionName]?.includes('*') && 
                !fieldMap[collectionName]?.includes(fieldName)) {
                continue;  // 字段无权访问
            }

            // 3. 特殊处理 O2M 关系字段
            const o2mRelation = schema.relations.find(
                (relation) => 
                    relation.related_collection === collectionName && 
                    relation.meta?.one_field === fieldName,
            );

            // O2M 字段指向的集合也需要有权限
            if (o2mRelation && !fieldMap[collectionName]) {
                continue;
            }

            fields[fieldName] = field;
        }

        reduced.collections[collectionName] = {
            ...collection,
            fields,
        };
    }

    // 4. 过滤关系（关键！关系的两端集合都需要有权限）
    reduced.relations = schema.relations.filter((relation) => {
        let collectionsAllowed = true;
        let fieldsAllowed = true;

        // 检查关系两端的集合是否都有权限
        if (Object.keys(fieldMap).includes(relation.collection) === false) {
            collectionsAllowed = false;
        }

        if (
            relation.related_collection &&
            (Object.keys(fieldMap).includes(relation.related_collection) === false ||
                fieldMap[relation.related_collection]?.length === 0)
        ) {
            collectionsAllowed = false;
        }

        // 检查 A2O 关系的 allowed_collections
        if (
            relation.meta?.one_allowed_collections &&
            relation.meta.one_allowed_collections.every(
                (collection) => Object.keys(fieldMap).includes(collection)
            ) === false
        ) {
            collectionsAllowed = false;
        }

        // 检查关系字段本身是否有权限
        if (
            !fieldMap[relation.collection] ||
            (fieldMap[relation.collection]?.includes('*') === false &&
                fieldMap[relation.collection]?.includes(relation.field) === false)
        ) {
            fieldsAllowed = false;
        }

        // 检查反向关系字段（O2M 的 one_field）
        if (
            relation.related_collection &&
            relation.meta?.one_field &&
            (!fieldMap[relation.related_collection] ||
                (fieldMap[relation.related_collection]?.includes('*') === false &&
                    fieldMap[relation.related_collection]?.includes(relation.meta?.one_field) === false))
        ) {
            fieldsAllowed = false;
        }

        return collectionsAllowed && fieldsAllowed;
    });

    return reduced;
}
```

**关键要点**：
- 关系的**两端集合**都需要有权限
- 关系字段本身（如 `category_id`）需要有权限
- 反向关系字段（如 O2M 的 `articles`）需要有权限
- A2O 关系的 `one_allowed_collections` 中的每个集合都需要有权限

#### 层次二：AST 构建时的权限检查

**文件位置：`api/src/database/get-ast-from-query/lib/parse-fields.ts`**

在构建查询 AST 时，对关系字段进行权限预检查：

```typescript
export async function parseFields(
    options: ParseFieldsOptions,
    context: ParseFieldsContext,
): Promise<[...]> {
    // ...
    
    const policies =
        options.accountability && options.accountability.admin === false
            ? await fetchPolicies(options.accountability, context)
            : null;

    // ...

    for (const [fieldKey, nestedFields] of Object.entries(relationalStructure)) {
        // ...

        // 获取关系信息
        const relatedCollection = getRelatedCollection(context.schema, options.parentCollection, fieldName);
        const relation = getRelation(context.schema.relations, options.parentCollection, fieldName);

        // ...

        if (relationType === 'a2o') {
            // A2O 特殊处理：过滤掉无权访问的集合
            let allowedCollections = relation.meta!.one_allowed_collections!;

            if (options.accountability && options.accountability.admin === false && policies) {
                const permissions = await fetchPermissions(
                    {
                        action: 'read',
                        collections: allowedCollections,
                        policies: policies,
                        accountability: options.accountability,
                    },
                    context,
                );

                // 关键：只保留有权限的集合
                allowedCollections = allowedCollections.filter((collection) =>
                    permissions.some((permission) => permission.collection === collection),
                );
            }

            // 为每个有权限的集合递归解析字段
            for (const relatedCollection of allowedCollections) {
                child.children[relatedCollection] = await parseFields(
                    {
                        parentCollection: relatedCollection,
                        // ...
                    },
                    { ...context, parentRelation: relation },
                );
            }
        } else if (relatedCollection) {
            // 其他关系类型：检查目标集合权限
            if (options.accountability && options.accountability.admin === false && policies) {
                const permissions = await fetchPermissions(
                    {
                        action: 'read',
                        collections: [relatedCollection],
                        policies: policies,
                        accountability: options.accountability,
                    },
                    context,
                );

                // 关键：如果目标集合无权访问，跳过此关系
                if (permissions.length === 0) {
                    continue;
                }
            }

            // 递归解析嵌套字段
            child = {
                type: relationType,
                name: relatedCollection,
                // ...
                children: await parseFields(
                    {
                        parentCollection: relatedCollection,
                        fields: nestedFields as string[],
                        // ...
                    },
                    { ...context, parentRelation: relation },
                ),
                // ...
            };
        }

        // ...
    }

    // ...
}
```

**关键要点**：
- A2O 关系：`one_allowed_collections` 中只保留有权限的集合
- 其他关系：目标集合无权限则**完全跳过**该关系字段
- 递归检查：嵌套关系字段同样会递归执行相同的权限检查逻辑

#### 层次三：AST 处理时的深度权限注入

**文件位置：`api/src/permissions/modules/process-ast/process-ast.ts`**

这是最核心的权限处理阶段，执行两项关键任务：

```typescript
export async function processAst(options: ProcessAstOptions, context: Context) {
    // 1. 从 AST 提取字段映射（包括嵌套关系）
    const fieldMap: FieldMap = fieldMapFromAst(options.ast, context.schema);
    const collections = collectionsInFieldMap(fieldMap);

    // 管理员直接放行（只验证字段存在性）
    if (!options.accountability || options.accountability.admin) {
        for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
            validatePathExistence(path, collection, fields, context.schema);
        }
        return options.ast;
    }

    // 2. 获取权限规则
    const policies = await fetchPolicies(options.accountability, context);
    const permissions = await fetchPermissions(
        { action: options.action, policies, collections, accountability: options.accountability },
        context,
    );

    // 读取操作还需要读取权限（用于关系字段的读取）
    const readPermissions =
        options.action === 'read'
            ? permissions
            : await fetchPermissions(
                    { action: 'read', policies, collections, accountability: options.accountability },
                    context,
                );

    // 3. 验证字段存在性
    for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
        validatePathExistence(path, collection, fields, context.schema);
    }

    // 4. 验证权限（关键！包括嵌套关系路径）
    // other 表示需要当前 action 权限的字段
    for (const [path, { collection, fields }] of fieldMap.other.entries()) {
        validatePathPermissions(path, permissions, collection, fields);
    }

    // read 表示只需要读取权限的字段（如嵌套关系中的字段）
    for (const [path, { collection, fields }] of fieldMap.read.entries()) {
        validatePathPermissions(path, readPermissions, collection, fields);
    }

    // 5. 注入权限条件到 AST（行级权限）
    injectCases(options.ast, permissions);

    return options.ast;
}
```

### 2.3 字段映射的构建

**文件位置：`api/src/permissions/modules/process-ast/lib/field-map-from-ast.ts`**

```typescript
export function fieldMapFromAst(ast: AST, schema: SchemaOverview): FieldMap {
    const fieldMap: FieldMap = { read: new Map(), other: new Map() };

    // 提取 AST 子节点中的字段
    extractFieldsFromChildren(ast.name, ast.children, fieldMap, schema);
    
    // 提取查询条件中的字段
    extractFieldsFromQuery(ast.name, ast.query, fieldMap, schema);

    return fieldMap;
}
```

**文件位置：`api/src/permissions/modules/process-ast/lib/extract-fields-from-children.ts`

```typescript
// 核心逻辑：递归处理嵌套关系节点
function extractFieldsFromChildren(
    currentCollection: string,
    children: (NestedCollectionNode | FieldNode | FunctionFieldNode)[],
    fieldMap: FieldMap,
    schema: SchemaOverview,
    path: string[] = [],
) {
    for (const child of children) {
        const fieldKey = getUnaliasedFieldKey(child);
        const currentPath = [...path, fieldKey].join('.');

        if (child.type === 'field' || child.type === 'functionField') {
            // 普通字段或函数字段
            addToFieldMap(fieldMap.other, currentPath, currentCollection, fieldKey);
        } else if (child.type === 'm2o') {
            // M2O 关系：外键字段需要当前 action 权限
            addToFieldMap(fieldMap.other, currentPath, currentCollection, child.fieldKey);
            
            // 递归处理嵌套字段（只需要读取权限）
            if (child.children && child.children.length > 0) {
                extractFieldsFromChildren(
                    child.relation.related_collection!,
                    child.children,
                    fieldMap,
                    schema,
                    [...path, fieldKey],
                );
            }
        } else if (child.type === 'o2m') {
            // O2M 关系：反向字段需要当前 action 权限
            addToFieldMap(fieldMap.other, currentPath, currentCollection, fieldKey);
            
            // 递归处理嵌套字段
            if (child.children && child.children.length > 0) {
                extractFieldsFromChildren(
                    child.relation.collection,
                    child.children,
                    fieldMap,
                    schema,
                    [...path, fieldKey],
                );
            }
        } else if (child.type === 'a2o') {
            // A2O 关系：多态字段需要当前 action 权限
            addToFieldMap(fieldMap.other, currentPath, currentCollection, fieldKey);
            
            // 递归处理每个可能的集合
            for (const [collection, children of Object.entries(child.children || {})) {
                extractFieldsFromChildren(
                    collection,
                    children,
                    fieldMap,
                    schema,
                    [...path, `${fieldKey}:${collection}`],
                );
            }
        }
    }
}
```

**关键区分 read vs other**：
- **other**：需要当前操作权限（如 update/delete/create）的字段
- **read**：只需要读取权限的字段（如嵌套关系中的字段）

### 2.4 路径权限验证

**文件位置：`api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts`**

```typescript
export function validatePathPermissions(
    path: string,
    permissions: Permission[],
    collection: string,
    fields: Set<string>,
) {
    const permissionsForCollection = permissions.filter(
        (permission) => permission.collection === collection
    );

    // 集合级别权限检查
    if (permissionsForCollection.length === 0) {
        throw createCollectionForbiddenError(path, collection);
    }

    // 字段级别权限检查
    const allowedFields: Set<string> = new Set();

    for (const { fields } of permissionsForCollection) {
        if (!fields) {
            continue;
        }

        for (const field of fields) {
            if (field === '*') {
                // 通配符：所有字段都允许
                return;
            }
            allowedFields.add(field);
        }
    }

    // 检查请求的字段是否都在允许范围内
    const requestedFields = Array.from(fields);
    const forbiddenFields = allowedFields.has('*')
        ? []
        : requestedFields.filter((field) => allowedFields.has(field) === false);

    if (forbiddenFields.length > 0) {
        throw createFieldsForbiddenError(path, collection, forbiddenFields);
    }
}
```

**路径示例**：
- `articles` → 根集合
- `articles.category` → 一级关系
- `articles.category.created_by` → 嵌套关系
- `articles.tags:directus_tags` → A2O 多态关系

### 2.5 权限条件注入（行级权限）

**文件位置：`api/src/permissions/modules/process-ast/lib/inject-cases.ts`**

```typescript
export function injectCases(ast: AST, permissions: Permission[]) {
    ast.cases = processChildren(ast.name, ast.children, permissions);
}

function processChildren(
    collection: string,
    children: (NestedCollectionNode | FieldNode | FunctionFieldNode)[],
    permissions: Permission[],
) {
    const requestedKeys = uniq(children.map(getUnaliasedFieldKey));
    
    // 获取权限规则中的条件（preset/validate 字段）
    const { cases, caseMap, allowedFields } = getCases(collection, permissions, requestedKeys);

    for (const child of children) {
        const fieldKey = getUnaliasedFieldKey(child);

        const globalWhenCase = caseMap['*'];
        const fieldWhenCase = caseMap[fieldKey];

        // 只有在没有完全访问权限时才注入条件
        if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
            // 将权限条件注入到字段节点
            child.whenCase = [...(globalWhenCase ?? []), ...(fieldWhenCase ?? [])];
        }

        // 递归处理关系节点
        if (child.type === 'm2o') {
            child.cases = processChildren(
                child.relation.related_collection!, 
                child.children, 
                permissions
            );
        }

        if (child.type === 'o2m') {
            child.cases = processChildren(
                child.relation.collection, 
                child.children, 
                permissions
            );
        }

        if (child.type === 'a2o') {
            for (const collection of child.names) {
                child.cases[collection] = processChildren(
                    collection, 
                    child.children[collection] ?? [], 
                    permissions
                );
            }
        }

        if (child.type === 'functionField') {
            const { cases } = getCases(child.relatedCollection, permissions, []);
            child.cases = cases;
        }
    }

    return cases;
}
```

**关键要点**：
- `whenCase`：字段级别的访问条件（哪些行可以访问该字段）
- `cases`：集合级别的访问条件（哪些行可以被返回）
- **递归注入**：关系字段的子节点也会注入对应集合的权限条件

### 2.6 关系权限裁剪完整流程图

```
查询: GET /items/articles?fields=*,category.*,tags.*

                    ↓

1. parseFields 构建 AST
   ├─ 检测到 m2o 字段 category → 检查 categories 集合权限
   │   └─ 无权限 → 跳过该关系字段
   │
   └─ 检测到 m2m 字段 tags → 检查 tags 集合权限
       └─ 无权限 → 跳过该关系字段

                    ↓

2. fieldMapFromAst 提取字段映射
   ├─ read: {
   │   "": { collection: "articles", fields: ["id", "title", "category", "tags"] },
   │   "category": { collection: "categories", fields: ["*"] },
   │   "tags": { collection: "tags", fields: ["*"] }
   │ }
   └─ other: {}

                    ↓

3. validatePathPermissions 验证权限
   ├─ 验证 "" (articles) → 检查 articles 权限
   ├─ 验证 "category" → 检查 categories 权限
   └─ 验证 "tags" → 检查 tags 权限

                    ↓

4. injectCases 注入行级权限
   ├─ articles 注入 articles 集合的权限条件
   ├─ category 子节点注入 categories 集合的权限条件
   └─ tags 子节点注入 tags 集合的权限条件

                    ↓

5. runAst 执行查询
   └─ whenCase/cases 在 SQL 查询中生效，过滤无权访问的行
```

---

## 3. 前端配置改动回写与 Schema 更新链路

### 3.1 前端配置页面架构

前端配置页面位于：
- **集合配置**：`app/src/modules/settings/routes/data-model/collections/`
- **字段配置**：`app/src/modules/settings/routes/data-model/fields/`
- **字段详情**：`app/src/modules/settings/routes/data-model/field-detail/`

### 3.2 前端 API 调用模式

前端通过 API 层与后端交互：

#### 集合操作 API：

| 操作 | HTTP 方法 | 端点 | 控制器 |
|------|-----------|------|--------|
| 创建集合 | POST | `/collections` | `collections.ts:12-42` |
| 读取集合 | GET/SEARCH | `/collections` | `collections.ts:44-70` |
| 读取单个集合 | GET | `/collections/:collection` | `collections.ts:72-86` |
| 批量更新集合 | PATCH | `/collections` | `collections.ts:88-112` |
| 更新单个集合 | PATCH | `/collections/:collection` | `collections.ts:114-138` |
| 删除集合 | DELETE | `/collections/:collection` | `collections.ts:140-153` |

#### 字段操作 API：

| 操作 | HTTP 方法 | 端点 | 控制器 |
|------|-----------|------|--------|
| 读取所有字段 | GET | `/fields` | `fields.ts:19-33` |
| 读取集合字段 | GET | `/fields/:collection` | `fields.ts:35-50` |
| 读取单个字段 | GET | `/fields/:collection/:field` | `fields.ts:52-67` |
| 创建字段 | POST | `/fields/:collection` | `fields.ts:86-122` |
| 批量更新字段 | PATCH | `/fields/:collection` | `fields.ts:124-171` |
| 更新单个字段 | PATCH | `/fields/:collection/:field` | `fields.ts:187-231` |
| 删除字段 | DELETE | `/fields/:collection/:field` | `fields.ts:233-250` |

### 3.3 集合创建完整流程

**文件位置：`api/src/services/collections.ts:createOne`**

```typescript
async createOne(payload: RawCollection, opts?: FieldMutationOptions): Promise<string> {
    // ... 前置验证（名称格式、是否已存在等

    try {
        // 1. 在事务中执行
        await transaction(this.knex, async (trx) => {
            
            // 2. 创建数据库表（如果指定了 schema）
            if (payload.schema) {
                // 确保有主键字段
                if (!payload.fields || payload.fields.length === 0) {
                    payload.fields = [injectedPrimaryKeyField];
                } else if (!payload.fields.some(...)) {
                    payload.fields = [injectedPrimaryKeyField, ...payload.fields];
                }

                // 创建表
                await trx.schema.createTable(payload.collection, (table) => {
                    for (const field of payload.fields!) {
                        if (field.type && ALIAS_TYPES.includes(field.type) === false) {
                            fieldsService.addColumnToTable(table, payload.collection, field, {...});
                        }
                    }
                });

                // 3. 插入 directus_fields 记录
                const fieldItemsService = new ItemsService('directus_fields', {...});
                await fieldItemsService.createMany(sortedFieldPayloads, {...});
            }

            // 4. 插入 directus_collections 记录
            if (payload.meta) {
                const collectionsItemsService = new ItemsService('directus_collections', {...});
                await collectionsItemsService.createOne({
                    ...payload.meta,
                    collection: payload.collection,
                }, {...});
            }

            return payload.collection;
        });

        return payload.collection;
    } finally {
        // 5. 清除缓存（关键！）
        if (shouldClearCache(this.cache, opts)) {
            await this.cache.clear();
        }

        if (opts?.autoPurgeSystemCache !== false) {
            await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });
        }

        // 6. 发射事件（使用更新后的 Schema）
        if (opts?.emitEvents !== false && nestedActionEvents.length > 0) {
            const updatedSchema = await getSchema();

            for (const nestedActionEvent of nestedActionEvents) {
                nestedActionEvent.context.schema = updatedSchema;
                emitter.emitAction(
                    nestedActionEvent.event, 
                    nestedActionEvent.meta, 
                    nestedActionEvent.context
                );
            }
        }
    }
}
```

### 3.4 字段更新完整流程

**文件位置：`api/src/services/fields.ts:updateField`**

```typescript
async updateField(collection: string, field: RawField, opts?: FieldMutationOptions): Promise<string> {
    // ...

    try {
        // 1. 执行 filter hook（允许扩展修改字段
        const hookAdjustedField = opts?.emitEvents !== false
            ? await emitter.emitFilter(`fields.update`, field, {...})
            : field;

        // 2. 检查是否需要修改数据库列
        if (hookAdjustedField.schema) {
            const existingColumn = await this.columnInfo(collection, hookAdjustedField.field);

            // 比较现有列和新 schema
            if (!isEqual(columnToCompare, hookAdjustedField.schema)) {
                try {
                    await transaction(this.knex, async (trx) => {
                        await trx.schema.alterTable(collection, (table) => {
                            this.addColumnToTable(table, collection, field, {
                                existing: existingColumn,
                                // ...
                            });
                        });
                    });
                } catch (err: any) {
                    throw await translateDatabaseError(err, field);
                }
            }
        }

        // 3. 更新 directus_fields 记录
        if (hookAdjustedField.meta && !isSystemField(collection, hookAdjustedField.field)) {
            const record = field.meta
                ? await this.knex.select('id').from('directus_fields')
                    .where({ collection, field: field.field }).first()
                : null;

            if (record) {
                // 更新现有记录
                await this.itemsService.updateOne(record.id, {
                    ...hookAdjustedField.meta,
                    collection: collection,
                    field: hookAdjustedField.field,
                }, { emitEvents: false });
            } else {
                // 创建新记录
                await this.itemsService.createOne({
                    ...hookAdjustedField.meta,
                    collection: collection,
                    field: hookAdjustedField.field,
                }, { emitEvents: false });
            }
        }

        return field.field;
    } finally {
        // 4. 清除缓存
        if (shouldClearCache(this.cache, opts)) {
            await this.cache.clear();
        }

        if (opts?.autoPurgeSystemCache !== false) {
            await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });
        }

        // 5. 发射事件
        if (opts?.emitEvents !== false && nestedActionEvents.length > 0) {
            const updatedSchema = await getSchema({ database: this.knex });

            for (const nestedActionEvent of nestedActionEvents) {
                nestedActionEvent.context.schema = updatedSchema;
                emitter.emitAction(...);
            }
        }
    }
}
```

### 3.5 缓存清除机制详解

**文件位置：`api/src/cache.ts`**

#### clearSystemCache 函数：

```typescript
export async function clearSystemCache(opts?: {
    forced?: boolean | undefined;
    autoPurgeCache?: false | undefined;
}): Promise<void> {
    const { systemCache, localSchemaCache, lockCache } = getCache();

    // 1. 清除系统缓存（带锁保护，防止并发清除）
    if (opts?.forced || !(await lockCache.get('system-cache-lock'))) {
        await lockCache.set('system-cache-lock', true, 10000);
        await systemCache.clear();
        await lockCache.delete('system-cache-lock');
    }

    // 2. 清除本地 Schema 缓存
    await localSchemaCache.clear();
    memorySchemaCache = null;

    // 3. 清除权限缓存（因为权限依赖 Schema
    await clearPermissionCache();

    // 4. 发布 Schema 变更消息（多实例同步）
    messenger.publish<CacheMessage>('schemaChanged', { 
        autoPurgeCache: opts?.autoPurgeCache 
    });
}
```

#### 多实例同步机制：

```typescript
// 在模块加载时订阅消息
if (redisConfigAvailable() && !messengerSubscribed) {
    messengerSubscribed = true;

    messenger.subscribe<CacheMessage>('schemaChanged', async (opts) => {
        // 1. 清除 API 数据缓存
        if (env['CACHE_STORE'] === 'memory' && 
            env['CACHE_AUTO_PURGE'] && 
            cache && 
            opts?.['autoPurgeCache'] !== false) {
            await cache.clear();
        }

        // 2. 清除 Schema 缓存
        await localSchemaCache?.clear();
        memorySchemaCache = null;
    });
}
```

**关键要点**：
- **锁保护**：使用 `system-cache-lock` 防止多个进程同时清除缓存
- **多级缓存**：系统缓存、Schema 缓存、权限缓存都需要清除
- **多实例同步**：通过 Redis Pub/Sub（或数据库消息总线）通知其他实例

### 3.6 Schema 重新生成机制

**文件位置：`api/src/utils/get-schema.ts`**

```typescript
export async function getSchema(
    options?: { database?: Knex; bypassCache?: boolean },
    attempt = 0,
): Promise<SchemaOverview> {
    // 1. 检查是否绕过缓存（如 snapshot 操作
    if (options?.bypassCache || env['CACHE_SCHEMA'] === false) {
        const database = options?.database || getDatabase();
        const schemaInspector = createInspector(database);
        return await getDatabaseSchema(database, schemaInspector);
    }

    // 2. 检查内存缓存
    const cached = getMemorySchemaCache();
    if (cached) return cached;

    // 3. 多进程同步
    const lock = useLock();
    const bus = useBus();

    const lockKey = 'schemaCache--preparing';
    const messageKey = 'schemaCache--done';
    const processId = await lock.increment(lockKey);

    if (processId === 1) {
        // 第一个进程：生成 Schema
        let schema: SchemaOverview | null = null;

        try {
            const database = options?.database || getDatabase();
            const schemaInspector = createInspector(database);

            schema = await getDatabaseSchema(database, schemaInspector);
            setMemorySchemaCache(schema);
            return schema;
        } finally {
            // 通知其他进程
            await bus.publish(messageKey, { schema });
            await lock.delete(lockKey);
        }
    } else {
        // 其他进程：等待消息
        logger.trace('Schema cache is prepared in another process, waiting for result.');

        const timeout: Promise<any> = new Promise((_, reject) =>
            setTimeout(reject, env['CACHE_SCHEMA_SYNC_TIMEOUT'] as number),
        );

        const subscription = new Promise<SchemaOverview>((resolve, reject) => {
            bus.subscribe(messageKey, busListener).catch(reject);

            function busListener(options: { schema: SchemaOverview | null }) {
                // ... 处理消息
            }
        });

        return Promise.race([timeout, subscription])
            .catch(() => getSchema(options, attempt + 1));
    }
}
```

### 3.7 GraphQL Schema 缓存失效

**文件位置：`api/src/services/graphql/schema/index.ts`**

```typescript
// generateSchema 函数中的缓存机制

export async function generateSchema(
    gql: GraphQLService,
    type: 'schema' | 'sdl' = 'schema',
): Promise<GraphQLSchema | string> {
    // 缓存键包含：scope + type + role + user
    const key = `${gql.scope}_${type}_${gql.accountability?.role}_${gql.accountability?.user}`;

    // 检查缓存
    const cachedSchema = cache.get(key);
    if (cachedSchema) return cachedSchema;

    // ... 生成新 Schema

    // 缓存新 Schema
    const gqlSchema = schemaComposer.buildSchema();
    cache.set(key, gqlSchema);
    return gqlSchema;
}
```

**缓存失效时机**：
- 当 `clearSystemCache()` 被调用时
- 间接通过 `systemCache.clear()` 清除
- **注意**：GraphQL 缓存使用 `systemCache` 的 Keyv 实例

### 3.8 完整的配置改动回写链路

```
前端：管理员修改集合/字段配置
                    ↓
前端 API 调用：
  POST/PATCH /collections 或 /fields
                    ↓
后端控制器：
  collections.ts / fields.ts
                    ↓
服务层：
  CollectionsService.createOne/updateOne/deleteOne
  FieldsService.createField/updateField/deleteField
                    ↓
事务内操作：
  1. 执行 DDL（CREATE/ALTER/DROP TABLE）
  2. 操作 directus_collections 表
  3. 操作 directus_fields 表
  4. 操作 directus_relations 表（关系变更时）
                    ↓
finally 块：
  1. clearSystemCache()
     ├─ 清除 systemCache
     ├─ 清除 localSchemaCache
     ├─ 设置 memorySchemaCache = null
     ├─ 清除权限缓存
     └─ 发布 schemaChanged 消息
                    ↓
多实例同步（Redis 环境）：
  其他实例收到 schemaChanged 消息
  ├─ 清除 API 数据缓存
  └─ 清除 Schema 缓存
                    ↓
下次请求：
  1. schema 中间件调用 getSchema()
  2. 检测到缓存为空
  3. 重新从数据库构建 Schema
  4. 缓存新 Schema
                    ↓
GraphQL 请求：
  1. generateSchema() 检测缓存失效
  2. 基于新 Schema 重新生成 GraphQL 类型
  3. 缓存新 GraphQL Schema
```

### 3.9 触发缓存清除的操作

以下操作会触发 `clearSystemCache()`：

| 操作 | 服务 | 文件位置 |
|------|------|----------|
| 创建集合 | `CollectionsService.createOne` | `collections.ts:247-249` |
| 批量创建集合 | `CollectionsService.createMany` | `collections.ts:298-300` |
| 更新集合 | `CollectionsService.updateOne` | `collections.ts:493-495` |
| 批量更新集合 | `CollectionsService.updateBatch` | `collections.ts:552-554` |
| 更新多个集合 | `CollectionsService.updateMany` | `collections.ts:602-604` |
| 删除集合 | `CollectionsService.deleteOne` | `collections.ts:786-788` |
| 批量删除集合 | `CollectionsService.deleteMany` | `collections.ts:834-836` |
| 创建字段 | `FieldsService.createField` | `fields.ts:488-490` |
| 更新字段 | `FieldsService.updateField` | `fields.ts:643-645` |
| 批量更新字段 | `FieldsService.updateFields` | `fields.ts:682-684` |
| 删除字段 | `FieldsService.deleteField` | `fields.ts:890-892` |

---

## 4. 关键代码索引

### 4.1 关系权限裁剪

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| Schema 权限过滤 | `api/src/utils/reduce-schema.ts` | GraphQL Schema 裁剪 |
| AST 字段解析 | `api/src/database/get-ast-from-query/lib/parse-fields.ts` | 关系字段权限预检查 |
| AST 权限处理 | `api/src/permissions/modules/process-ast/process-ast.ts` | 核心权限处理 |
| 字段映射提取 | `api/src/permissions/modules/process-ast/lib/field-map-from-ast.ts` | 从 AST 提取访问路径 |
| 路径权限验证 | `api/src/permissions/modules/process-ast/utils/validate-path/` | 路径级别权限验证 |
| 条件注入 | `api/src/permissions/modules/process-ast/lib/inject-cases.ts` | 行级权限条件注入 |

### 4.2 配置回写与 Schema 更新

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 集合服务 | `api/src/services/collections.ts` | 集合 CRUD + 缓存清除 |
| 字段服务 | `api/src/services/fields.ts` | 字段 CRUD + 缓存清除 |
| 缓存管理 | `api/src/cache.ts` | 多级缓存 + 多实例同步 |
| Schema 生成 | `api/src/utils/get-schema.ts` | Schema 构建 + 缓存 + 多进程同步 |
| GraphQL Schema | `api/src/services/graphql/schema/index.ts` | GraphQL Schema 动态生成 |
| 集合控制器 | `api/src/controllers/collections.ts` | REST API 端点 |
| 字段控制器 | `api/src/controllers/fields.ts` | REST API 端点 |

---

## 5. 总结

### 5.1 关系权限裁剪的核心要点

1. **三级权限检查**：
   - Schema 级：GraphQL Schema 生成时过滤无权访问的关系
   - AST 构建级：解析字段时预检查目标集合权限
   - AST 处理级：深度验证所有访问路径的权限

2. **关系类型特殊处理**：
   - M2O：外键字段 + 目标集合都需要权限
   - O2M：反向字段 + 目标集合都需要权限
   - M2M：中间表 + 两端集合都需要权限
   - A2O：多态字段 + 每个 allowed_collection 都需要权限

3. **行级权限注入**：
   - `whenCase`：字段级条件
   - `cases`：集合级条件
   - 递归注入到所有嵌套关系

### 5.2 配置回写与 Schema 更新的核心要点

1. **事务一致性**：
   - 数据库 DDL 操作与元数据变更在同一事务中
   - 缓存清除在 finally 块中确保执行

2. **多级缓存失效**：
   - `systemCache`：系统级缓存
   - `localSchemaCache`：Schema 缓存
   - `memorySchemaCache`：内存 Schema 缓存
   - 权限缓存：依赖 Schema

3. **多实例同步**：
   - 使用消息总线发布 `schemaChanged` 消息
   - Redis 环境下自动同步
   - 内存缓存环境下依赖 TTL 失效

4. **事件驱动**：
   - 配置变更后发射事件
   - 事件上下文包含更新后的 Schema
   - 扩展可订阅事件进行自定义逻辑

### 5.3 设计亮点

1. **权限与 Schema 深度集成**：权限检查不仅在接口层，而是深入到 AST 构建和执行的每个阶段

2. **递归权限检查**：关系字段的嵌套访问会递归执行相同的权限检查逻辑

3. **悲观缓存策略**：多级缓存 + 锁保护 + 多实例同步，确保高性能和一致性

4. **事件驱动架构**：配置变更通过事件通知系统，扩展点丰富
