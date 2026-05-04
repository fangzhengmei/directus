# Collection & Field 元数据驱动机制分析报告

## 1. 概述

Directus 采用**元数据驱动架构**，通过 Collection 和 Field 的配置同时驱动：
- **API 结构生成**：动态生成 REST API 端点和 GraphQL Schema
- **管理后台渲染**：动态渲染表单、列表、导航等 UI 组件

核心数据流转路径：

```
数据库层 (directus_collections / directus_fields)
       ↓
元数据聚合层 (getSchema → SchemaOverview)
       ↓
    ┌──┴──┐
    ↓     ↓
 API层   App层
(GraphQL) (Vue Components)
```

---

## 2. 核心元数据结构

### 2.1 CollectionMeta 定义

**文件位置**: `packages/types/src/collection.ts:11-33`

```typescript
export type CollectionMeta = {
    collection: string;           // 集合名称（主键）
    note: string | null;          // 描述说明
    hidden: boolean;              // 是否在管理后台隐藏
    singleton: boolean;           // 是否为单例模式
    icon: string | null;          // 图标
    color: string | null;         // 颜色标识
    translations: Translations[]; // 多语言翻译
    display_template: string;     // 显示模板
    preview_url: string | null;   // 预览 URL
    versioning: boolean;          // 是否启用版本控制
    sort_field: string | null;    // 排序字段
    archive_field: string | null; // 归档字段
    archive_value: string;        // 归档值
    unarchive_value: string;      // 取消归档值
    archive_app_filter: boolean;  // 归档筛选器
    item_duplication_fields: string[]; // 复制字段
    accountability: 'all' | 'activity' | null; // 审计追踪
    system: boolean | null;       // 是否系统集合
    sort: number | null;          // 排序序号
    group: string | null;         // 所属分组
    collapse: 'open' | 'closed' | 'locked'; // 折叠状态
};
```

### 2.2 FieldMeta 定义与可空性分析

**文件位置**: `packages/types/src/fields.ts:36-59`

根据类型定义，以下是 `FieldMeta` 每个字段的可空性分析：

```typescript
export type FieldMeta = {
    // ===== 非空字段 (No null) =====
    id: number;                         // 字段 ID (非空)
    collection: string;                 // 所属集合 (非空)
    field: string;                      // 字段名称 (非空)
    hidden: boolean;                    // 是否隐藏 (非空)
    readonly: boolean;                  // 是否只读 (非空)
    required: boolean;                  // 是否必填 (非空)
    searchable: boolean;                // 是否可搜索 (非空)

    // ===== 可空字段 (Allow null) =====
    group: string | null;               // 字段组
    interface: string | null;           // 界面组件类型（如 'input', 'select'）
    display: string | null;             // 显示组件类型
    options: Record<string, any> | null; // 界面配置选项
    display_options: Record<string, any> | null; // 显示配置选项
    sort: number | null;                // 排序序号
    special: string[] | null;           // 特殊类型标记（如 'file', 'm2o', 'o2m'）
    translations: Translations[] | null; // 多语言翻译
    width: Width | null;                // 表单宽度
    note: string | null;                // 字段说明
    conditions: Condition[] | null;     // 条件逻辑
    validation: Filter | null;          // 验证规则
    validation_message: string | null;  // 验证失败消息

    // ===== 可选字段 (Optional, 可能 undefined) =====
    system?: true;                      // 是否系统字段
    clear_hidden_value_on_save?: boolean; // 隐藏时是否清除值
};
```

#### FieldMeta 可空性总结表

| 字段 | 类型 | 非空? | 来源/用途 |
|------|------|-------|-----------|
| `id` | `number` | ✅ 非空 | 数据库主键 |
| `collection` | `string` | ✅ 非空 | 所属集合标识 |
| `field` | `string` | ✅ 非空 | 字段名称 |
| `hidden` | `boolean` | ✅ 非空 | 是否隐藏 |
| `readonly` | `boolean` | ✅ 非空 | 是否只读 |
| `required` | `boolean` | ✅ 非空 | UI 层面的必填标记 |
| `searchable` | `boolean` | ✅ 非空 | 是否可搜索 |
| `group` | `string \| null` | ❌ 可空 | 字段组引用 |
| `interface` | `string \| null` | ❌ 可空 | 表单输入组件类型 |
| `display` | `string \| null` | ❌ 可空 | 列表显示组件类型 |
| `options` | `Record<string, any> \| null` | ❌ 可空 | Interface 配置选项 |
| `display_options` | `Record<string, any> \| null` | ❌ 可空 | Display 配置选项 |
| `sort` | `number \| null` | ❌ 可空 | 排序序号 |
| `special` | `string[] \| null` | ❌ 可空 | 特殊类型标记（关系、文件、自动生成等）|
| `translations` | `Translations[] \| null` | ❌ 可空 | 多语言翻译 |
| `width` | `Width \| null` | ❌ 可空 | 表单宽度（half, full, fill）|
| `note` | `string \| null` | ❌ 可空 | 字段帮助说明 |
| `conditions` | `Condition[] \| null` | ❌ 可空 | 条件逻辑（条件隐藏、只读等）|
| `validation` | `Filter \| null` | ❌ 可空 | 自定义验证规则 |
| `validation_message` | `string \| null` | ❌ 可空 | 验证失败提示消息 |
| `system` | `true \| undefined` | ❌ 可选 | 是否为系统字段 |
| `clear_hidden_value_on_save` | `boolean \| undefined` | ❌ 可选 | 字段隐藏时是否清除值 |

#### 关键发现：可空性与默认值的关系

注意：`FieldMeta` 中的可空字段如果为 `null`，在不同场景下有不同的默认行为：

1. **`interface: string | null`**
   - 为 `null` 时：使用 `getDefaultInterfaceForType(field.type)` 选择默认 Interface
   - 默认映射见 `app/src/utils/get-default-interface-for-type.ts`

2. **`display: string | null`**
   - 为 `null` 时：使用 `getDefaultDisplayForType(field.type)` 选择默认 Display

3. **`options: Record<string, any> | null`**
   - 为 `null` 时：传递空对象 `{}` 给 Interface 组件

4. **`required: boolean`（非空但常被误解）**
   - 注意：这个字段**只影响 UI 显示**（显示星号）
   - **不影响 API 验证**！API 验证使用的是 `FieldOverview.nullable` + `FieldOverview.defaultValue`

### 2.3 SchemaOverview - 核心共享结构

**文件位置**: `packages/types/src/schema.ts:37-40`

```typescript
export type SchemaOverview = {
    collections: CollectionsOverview;  // 集合概览
    relations: Relation[];              // 关系定义
};

export type CollectionOverview = {
    collection: string;
    primary: string;                    // 主键字段
    singleton: boolean;
    sortField: string | null;
    note: string | null;
    accountability: 'all' | 'activity' | null;
    fields: { [name: string]: FieldOverview };
};

export type FieldOverview = {
    field: string;
    defaultValue: any;
    nullable: boolean;
    generated: boolean;
    type: Type;                         // 逻辑类型（如 'string', 'integer', 'json'）
    dbType: string | null;              // 数据库类型
    special: string[];                  // 特殊标记
    alias: boolean;
    searchable: boolean;
    validation: Filter | null;
};
```

---

## 3. 数据存储层

元数据存储在系统表中：

| 系统表 | 存储内容 | 关键字段 |
|--------|----------|----------|
| `directus_collections` | Collection 元数据 | `collection`, `singleton`, `icon`, `hidden`, `versioning` |
| `directus_fields` | Field 元数据 | `collection`, `field`, `interface`, `display`, `options`, `special` |
| `directus_relations` | 关系定义 | `collection`, `field`, `related_collection`, `one_field` |

---

## 4. API 端驱动路径

### 4.1 元数据加载入口

**文件位置**: `api/src/utils/get-schema.ts`

```typescript
export async function getSchema(): Promise<SchemaOverview> {
    // 1. 检查缓存
    const cached = getMemorySchemaCache();
    if (cached) return cached;

    // 2. 从数据库构建 SchemaOverview
    return await getDatabaseSchema(database, schemaInspector);
}
```

**构建流程** (`getDatabaseSchema` 函数):

1. **获取数据库结构**：通过 `schemaInspector.overview()` 获取表和列信息
2. **合并 Collection 元数据**：从 `directus_collections` 表读取配置
3. **合并 Field 元数据**：从 `directus_fields` 表读取配置
4. **构建关系定义**：通过 `RelationsService.readAll()` 获取关系

### 4.2 GraphQL Schema 动态生成

**文件位置**: `api/src/services/graphql/schema/index.ts`

核心函数 `generateSchema()` 流程：

```
输入: SchemaOverview
       ↓
步骤1: 权限过滤
       ↓  (fetchAllowedFieldMap, reduceSchema)
步骤2: 构建可读类型
       ↓  (getReadableTypes)
   - 遍历 schema.read.collections
   - 为每个 Collection 创建 ObjectTypeComposer
   - 为每个 Field 创建 GraphQL 字段
   - 根据 FieldOverview.type 映射 GraphQL 类型
       ↓
步骤3: 构建可写类型
       ↓  (getWritableTypes)
   - 创建输入类型（Create/Update）
   - 处理关系字段的嵌套输入
       ↓
步骤4: 挂载查询和变更
       ↓
输出: GraphQLSchema
```

**类型映射示例** (`api/src/services/graphql/schema/read.ts:442-501`):

```typescript
for (const field of Object.values(collection.fields)) {
    const graphqlType = getGraphQLType(field.type, field.special);
    
    // 根据字段类型选择过滤器操作符
    switch (graphqlType) {
        case GraphQLBoolean:
            filterOperatorType = BooleanFilterOperators;
            break;
        case GraphQLInt:
        case GraphQLFloat:
            filterOperatorType = NumberFilterOperators;
            break;
        case GraphQLString:
            filterOperatorType = StringFilterOperators;
            break;
        // ... 更多类型
    }
    
    acc[field.field] = filterOperatorType;
}
```

### 4.3 REST API 动态路由：从路由入口到服务层的完整链路

#### 4.3.1 路由注册入口

**文件位置**: `api/src/app.ts:342`

```typescript
app.use('/items', itemsRouter);
```

这行代码将 `itemsRouter` 注册到 Express 应用，使得所有 `/items/*` 路径的请求都由 items 控制器处理。

#### 4.3.2 Schema 中间件：请求前的元数据加载

在路由处理之前，`schema` 中间件会为每个请求加载 `SchemaOverview`：

**文件位置**: `api/src/middleware/schema.ts:1-8`

```typescript
import type { RequestHandler } from 'express';
import asyncHandler from '../utils/async-handler.js';
import { getSchema } from '../utils/get-schema.js';

const schema: RequestHandler = asyncHandler(async (req, _res, next) => {
    req.schema = await getSchema();
    return next();
});

export default schema;
```

**关键机制**：
- `getSchema()` 会从缓存或数据库获取 `SchemaOverview`
- 结果挂载到 `req.schema`，供后续所有中间件和控制器使用
- 缓存由 `CACHE_SCHEMA` 环境变量控制

**注册位置**: `api/src/app.ts:309`

```typescript
app.use(schema);
```

#### 4.3.3 Items 控制器：路由定义与 Schema 传递

**文件位置**: `api/src/controllers/items.ts`

控制器定义了所有 REST 端点，并在创建 `ItemsService` 时传入 `req.schema`：

**路由定义汇总**:

| HTTP 方法 | 路径 | 处理函数 | 功能 |
|-----------|------|----------|------|
| POST | `/items/:collection` | `createOne` / `createMany` | 创建单条或批量记录 |
| GET | `/items/:collection` | `readHandler` | 查询列表（支持 `singleton`） |
| GET | `/items/:collection/:pk` | `readOne` | 按主键查询单条 |
| PATCH | `/items/:collection` | `updateBatch` / `updateMany` / `updateByQuery` | 批量更新 |
| PATCH | `/items/:collection/:pk` | `updateOne` | 按主键更新单条 |
| DELETE | `/items/:collection` | `deleteMany` / `deleteByQuery` | 批量删除 |
| DELETE | `/items/:collection/:pk` | `deleteOne` | 按主键删除单条 |

**关键代码示例 - 创建操作** (`api/src/controllers/items.ts:15-59`):

```typescript
router.post(
    '/:collection',
    collectionExists,
    asyncHandler(async (req, res, next) => {
        // 1. 检查是否为系统集合（系统集合不允许直接操作）
        if (isSystemCollection(req.params['collection']!)) throw new ForbiddenError();

        // 2. 单例集合不支持 POST 创建
        if (req.singleton) {
            throw new RouteNotFoundError({ path: req.path });
        }

        // 3. 创建 ItemsService，传入 req.schema！
        const service = new ItemsService(req.collection, {
            accountability: req.accountability,
            schema: req.schema,  // 关键：SchemaOverview 被传递到服务层
        });

        // 4. 执行创建
        if (Array.isArray(req.body)) {
            const keys = await service.createMany(req.body);
            savedKeys.push(...keys);
        } else {
            const key = await service.createOne(req.body);
            savedKeys.push(key);
        }

        // 5. 读取并返回创建的数据
        if (Array.isArray(req.body)) {
            const result = await service.readMany(savedKeys, req.sanitizedQuery);
            res.locals['payload'] = { data: result || null };
        } else {
            const result = await service.readOne(savedKeys[0]!, req.sanitizedQuery);
            res.locals['payload'] = { data: result || null };
        }

        return next();
    }),
    respond,
);
```

#### 4.3.4 ItemsService：Schema 在业务层的实际使用

**文件位置**: `api/src/services/items.ts`

`ItemsService` 在构造函数中接收 `SchemaOverview`，并在所有 CRUD 操作中使用：

**构造函数** (`api/src/services/items.ts:53-63`):

```typescript
constructor(collection: Collection, options: AbstractServiceOptions) {
    this.collection = collection;
    this.knex = options.knex || getDatabase();
    this.accountability = options.accountability || null;
    this.eventScope = isSystemCollection(this.collection) ? this.collection.substring(9) : 'items';
    this.schema = options.schema;  // 保存 SchemaOverview
    this.cache = getCache().cache;
    this.nested = options.nested ?? [];

    return this;
}
```

**Schema 的实际使用场景**:

1. **获取主键字段** (`api/src/services/items.ts:108`):
   ```typescript
   async getKeysByQuery(query: Query): Promise<PrimaryKey[]> {
       const primaryKeyField = this.schema.collections[this.collection]!.primary;
       // ...
   }
   ```

2. **创建操作中的字段处理** (`api/src/services/items.ts:134-139`):
   ```typescript
   async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
       const primaryKeyField = this.schema.collections[this.collection]!.primary;
       const fields = Object.keys(this.schema.collections[this.collection]!.fields);  // 获取所有字段名
       
       const aliases = Object.values(this.schema.collections[this.collection]!.fields)
           .filter((field) => field.alias === true)
           .map((field) => field.field);
       // ...
   }
   ```

3. **自增主键检测** (`api/src/services/items.ts:234-244`):
   ```typescript
   const pkField = this.schema.collections[this.collection]!.fields[primaryKeyField];
   
   if (
       primaryKey &&
       pkField &&
       !opts.bypassAutoIncrementSequenceReset &&
       ['integer', 'bigInteger'].includes(pkField.type) &&
       pkField.defaultValue === 'AUTO_INCREMENT'
   ) {
       autoIncrementSequenceNeedsToBeReset = true;
   }
   ```

#### 4.3.5 PayloadService：字段值的类型转换与特殊处理

**文件位置**: `api/src/services/payload.ts`

`PayloadService` 负责处理字段值的转换，依赖 `FieldOverview.special` 属性：

**核心转换器定义** (`api/src/services/payload.ts:74-195`):

```typescript
public transformers: Transformers = {
    async hash({ action, value }) {
        // 创建/更新时对密码等字段进行哈希
        if (action === 'create' || action === 'update') {
            return await generateHash(String(value));
        }
        return value;
    },
    async uuid({ action, value }) {
        // 创建时自动生成 UUID（如果未提供）
        if (action === 'create' && !value) {
            return randomUUID();
        }
        return value;
    },
    async 'user-created'({ action, value, accountability, overwriteDefaults }) {
        // 创建时自动设置当前用户
        if (action === 'create') return (overwriteDefaults ? overwriteDefaults._user : accountability?.user) ?? null;
        return value;
    },
    async 'date-created'({ action, value, helpers, overwriteDefaults }) {
        // 创建时自动设置当前时间
        if (action === 'create')
            return new Date(
                overwriteDefaults ? overwriteDefaults._date : helpers.date.writeTimestamp(new Date().toISOString()),
            );
        return value;
    },
    // ... 更多转换器：user-updated, role-created, date-updated, cast-json, cast-boolean 等
};
```

**处理入口** (`api/src/services/payload.ts:213-282`):

```typescript
async processValues(
    action: PayloadAction,
    payload: Partial<Item> | Partial<Item>[],
    aliasMap: Record<string, string> = {},
    aggregate: Aggregate = {},
): Promise<Partial<Item> | Partial<Item>[]> {
    const fieldEntries = Object.entries(this.schema.collections[this.collection]!.fields);
    
    // 筛选有 special 标记的字段
    let specialFields: [string, FieldOverview][] = [];
    for (const [name, field] of fieldEntries) {
        if (field.special && field.special.length > 0) {
            specialFields.push([name, field]);
        }
    }
    
    // 对每个记录应用转换器
    for (const record of processedPayload) {
        for (const [name, field] of specialFields) {
            const newValue = await this.processField(field, record, action, this.accountability);
            if (newValue !== undefined) record[name] = newValue;
        }
    }
    // ...
}
```

---

## 4.4 可空字段与默认值的语义边界

### 4.4.1 三个相关属性的定义与来源

在 Directus 中，有三个属性共同控制字段的"必填"语义，但它们来自不同层级：

| 属性 | 类型 | 来源 | 语义 |
|------|------|------|------|
| `FieldOverview.nullable` | `boolean` | 数据库列定义 `schema.is_nullable` | 数据库层面是否允许 NULL |
| `FieldOverview.defaultValue` | `any` | 数据库列默认值 `schema.default_value` | 数据库层面的默认值 |
| `FieldMeta.required` | `boolean` | `directus_fields.required` | Directus 层面的"必填"标记（主要用于 UI）|
| `FieldOverview.generated` | `boolean` | 数据库列定义 | 是否为数据库生成的列（如计算列）|

**SchemaOverview 构建时的赋值逻辑** (`api/src/utils/get-schema.ts:153-177`):

```typescript
result.collections[collection] = {
    collection,
    primary: info.primary,
    singleton: toBoolean(collectionMeta?.singleton),
    // ...
    fields: mapValues(schemaOverview[collection]?.columns, (column) => {
        return {
            field: column.column_name,
            defaultValue: getDefaultValue(column) ?? null,  // 来自数据库 schema
            nullable: column.is_nullable ?? true,          // 来自数据库 schema
            generated: column.is_generated ?? false,       // 来自数据库 schema
            type: getLocalType(column),
            dbType: column.data_type,
            // ...
        };
    }),
};
```

### 4.4.2 默认值的处理逻辑

**文件位置**: `api/src/utils/get-default-value.ts`

`getDefaultValue()` 函数负责将数据库原始默认值转换为正确的 JavaScript 类型：

```typescript
export default function getDefaultValue(
    column: SchemaOverview[string]['columns'][string] | Column,
    field?: { special?: FieldMeta['special'] },
): string | boolean | number | Record<string, any> | any[] | null {
    const type = getLocalType(column, field);

    const defaultValue = column.default_value ?? null;
    if (defaultValue === null) return null;
    if (defaultValue === '0000-00-00 00:00:00') return null;  // MySQL 的"零日期"视为 null

    switch (type) {
        case 'bigInteger':
        case 'integer':
        case 'decimal':
        case 'float':
            return Number.isNaN(Number(defaultValue)) === false ? Number(defaultValue) : defaultValue;
        case 'boolean':
            return castToBoolean(defaultValue);  // 处理 '0'/'1', 'false'/'true' 等
        case 'json':
            return castToObject(defaultValue);  // 解析 JSON 字符串
        default:
            return defaultValue;
    }
}
```

**重要发现**：
- 默认值 `'AUTO_INCREMENT'` 是一个特殊字符串，表示该字段由数据库自增生成
- MySQL 的 `'0000-00-00 00:00:00'` 被视为 `null`

### 4.4.3 字段可空性的判断逻辑

**文件位置**: `api/src/permissions/modules/process-payload/lib/is-field-nullable.ts`

这是 API 层面判断字段是否允许为 `null` 的核心函数：

```typescript
import { GENERATE_SPECIAL } from '@directus/constants';
import type { FieldOverview } from '@directus/types';

export function isFieldNullable(field: FieldOverview) {
    // 1. 数据库层面允许 NULL → 可空
    if (field.nullable) return true;
    
    // 2. 是数据库生成的列（如计算列）→ 可空（不由用户提供值）
    if (field.generated) return true;

    // 3. 有 "自动生成" 的 special 标记 → 可空（系统会自动填充）
    const hasGenerateSpecial = GENERATE_SPECIAL.some((name) => field.special.includes(name));

    return hasGenerateSpecial;
}
```

**GENERATE_SPECIAL 包含的标记**（来自 `@directus/constants`）:
- `uuid` - 自动生成 UUID
- `user-created` - 自动设置创建者
- `user-updated` - 自动设置更新者
- `role-created` - 自动设置创建角色
- `role-updated` - 自动设置更新角色
- `date-created` - 自动设置创建时间
- `date-updated` - 自动设置更新时间

### 4.4.4 创建时的必填验证逻辑

**文件位置**: `api/src/permissions/modules/process-payload/process-payload.ts`

这是实际执行验证的核心代码，清晰地展示了语义边界：

```typescript
for (const field of fields) {
    if (!isFieldNullable(field)) {
        // 关键判断：创建时 + 无默认值 → 必须提交该字段
        const isSubmissionRequired = options.action === 'create' && field.defaultValue === null;

        if (isSubmissionRequired) {
            fieldValidationRules.push({
                [field.field]: {
                    _submitted: true,  // 验证：字段必须存在于 payload 中
                },
            });
        }

        // 无论创建还是更新，值都不能为 null
        fieldValidationRules.push({
            [field.field]: {
                _nnull: true,  // 验证：值不能为 null
            },
        });
    }

    // 额外：应用字段自定义的 validation 规则
    if (field.validation) {
        // ...
        fieldValidationRules.push(validationFilter);
    }
}
```

### 4.4.5 语义边界总结表

| 场景 | `nullable=true` | `nullable=false` + `defaultValue=AUTO_INCREMENT` | `nullable=false` + `defaultValue='foo'` | `nullable=false` + `defaultValue=null` |
|------|-----------------|---------------------------------------------------|-------------------------------------------|------------------------------------------|
| 创建时不提交字段 | ✅ 允许，存 NULL | ✅ 允许，数据库自增 | ✅ 允许，使用默认值 `'foo'` | ❌ 报错：`_submitted` 验证失败 |
| 创建时提交 `null` | ✅ 允许 | ❌ 报错：`_nnull` 验证失败 | ❌ 报错：`_nnull` 验证失败 | ❌ 报错：`_nnull` 验证失败 |
| 创建时提交有效值 | ✅ 允许 | ✅ 允许（会覆盖自增值）| ✅ 允许（会覆盖默认值）| ✅ 允许 |
| 更新时设为 `null` | ✅ 允许 | ❌ 报错：`_nnull` 验证失败 | ❌ 报错：`_nnull` 验证失败 | ❌ 报错：`_nnull` 验证失败 |

### 4.4.6 FieldMeta.required 的作用

注意 `FieldMeta.required` 与上述验证逻辑**没有直接关系**！

**`FieldMeta.required` 的实际用途**：

1. **UI 层面**：表单中显示红色星号 `*`
2. **OpenAPI/Spec 生成**：在动态生成的 OpenAPI schema 中标记为 `required`

**代码位置**: `api/src/services/specifications.ts:414-427`

```typescript
// 在 generateComponents 中
for (const field of fieldsInCollection) {
    const fieldSchema = this.generateField(schema, collection.collection, field, tags);
    schemaComponent.properties![field.field] = fieldSchema;

    // Check if field is required
    if (field.nullable === false && field.defaultValue === null && field.generated === false) {
        requiredFields.push(field.field);
    }
}
```

**注意**：OpenAPI 的 `required` 判断使用的是 `FieldOverview` 的属性，而非 `FieldMeta.required`。

---

## 4.5 动态 OpenAPI 规范生成

**文件位置**: `api/src/services/specifications.ts`

`SpecificationService` 使用 `SchemaOverview` 动态生成 OpenAPI 3.0 规范：

**路径生成** (`api/src/services/specifications.ts:173-347`):
- 遍历 `schema.collections` 为每个集合生成 `/items/{collection}` 路径
- 根据权限过滤可见的路径和方法

**组件 Schema 生成** (`api/src/services/specifications.ts:349-434`):
```typescript
for (const field of fieldsInCollection) {
    const fieldSchema = this.generateField(schema, collection.collection, field, tags);
    schemaComponent.properties![field.field] = fieldSchema;

    // 使用 FieldOverview 判断必填性
    if (field.nullable === false && field.defaultValue === null && field.generated === false) {
        requiredFields.push(field.field);
    }
}
```

**字段类型映射** (`api/src/services/specifications.ts:552-635`):
```typescript
private fieldTypes: Record<Type, Partial<SchemaObject>> = {
    alias: { type: 'string' },
    bigInteger: { type: 'integer', format: 'int64' },
    boolean: { type: 'boolean' },
    date: { type: 'string', format: 'date' },
    dateTime: { type: 'string', format: 'date-time' },
    float: { type: 'number', format: 'float' },
    integer: { type: 'integer' },
    uuid: { type: 'string', format: 'uuid' },
    // ... 更多类型映射
};
```

---

## 5. App 端驱动路径

### 5.1 数据获取与 Store 管理

App 端通过 API 端点获取元数据，并存储在 Pinia stores 中：

| Store | API 端点 | 职责 |
|-------|----------|------|
| `useCollectionsStore` | `GET /collections` | 集合元数据管理 |
| `useFieldsStore` | `GET /fields` | 字段元数据管理 |
| `useRelationsStore` | `GET /relations` | 关系定义管理 |

**Collections Store 核心逻辑** (`app/src/stores/collections.ts:89-95`):

```typescript
async function hydrate() {
    const response = await api.get<any>(`/collections`);
    const rawCollections: CollectionRaw[] = response.data.data;
    collections.value = rawCollections.map(prepareCollectionForApp);
}
```

**Fields Store 核心逻辑** (`app/src/stores/fields.ts:101-107`):

```typescript
async function hydrate(options?: HydrateOptions) {
    const fieldsResponse = await api.get<any>(`/fields`);
    const fieldsRaw: FieldRaw[] = fieldsResponse.data.data;
    fields.value = [...fieldsRaw.map(parseField), fakeFilesField];
    if (options?.skipTranslation !== true) translateFields();
}
```

### 5.2 SchemaOverview 转换

App 端也提供 `SchemaOverview` 的计算属性，用于统一的类型系统：

**文件位置**: `app/src/composables/use-schema.ts`

```typescript
export function useSchemaOverview(): ComputedRef<SchemaOverview> {
    return computed(() => ({
        collections: Object.fromEntries(
            collectionsStore.collections.map((collection) => {
                const fields = fieldsStore.getFieldsForCollection(collection.collection)
                    .map((field) => [
                        field.field,
                        {
                            field: field.field,
                            defaultValue: field.schema?.default_value,
                            type: field.type,
                            special: field.meta?.special ?? [],
                            // ... 映射其他属性
                        } satisfies FieldOverview,
                    ]);

                return [collection.collection, {
                    collection: collection.collection,
                    fields: Object.fromEntries(fields),
                    singleton: collection.meta?.singleton ?? false,
                    // ... 其他属性
                } satisfies CollectionOverview];
            }),
        ),
        relations: relationsStore.relations,
    }));
}
```

### 5.3 UI 渲染驱动

FieldMeta 中的关键字段直接驱动 UI 渲染：

#### 5.3.1 Interface 字段驱动表单组件

```typescript
// FieldMeta.interface 决定使用哪个输入组件
// 例如: 'input', 'select', 'textarea', 'checkbox', 'datetime' 等

// 实际使用示例 (伪代码)
const component = resolveInterface(field.meta.interface);
// 渲染: <component :is="component" :options="field.meta.options" />
```

#### 5.3.2 Display 字段驱动列表显示

```typescript
// FieldMeta.display 决定列表中如何显示该字段
// 例如: 'formatted-value', 'related-values', 'thumbnail', 'user' 等

// 实际使用示例 (伪代码)
const displayComponent = resolveDisplay(field.meta.display);
// 渲染: <display-component :value="item[field.field]" :options="field.meta.display_options" />
```

#### 5.3.3 其他影响 UI 的字段

| 字段 | 影响 |
|------|------|
| `hidden` | 是否在表单和列表中隐藏 |
| `readonly` | 表单中是否只读 |
| `required` | 表单中是否必填，显示星号 |
| `width` | 表单中的宽度（half, full, fill）|
| `sort` | 字段在表单和列表中的顺序 |
| `note` | 字段下方的帮助文本 |
| `translations` | 多语言显示 |
| `conditions` | 条件显示/只读逻辑 |
| `options` | Interface 的配置选项 |

#### 5.3.4 CollectionMeta 驱动导航和布局

| 字段 | 影响 |
|------|------|
| `icon` | 侧边栏导航图标 |
| `color` | 集合标识颜色 |
| `hidden` | 是否在导航中显示 |
| `singleton` | 决定是列表页还是详情页 |
| `translations` | 多语言名称 |
| `group` | 导航分组 |
| `sort` | 导航排序 |

---

## 6. 跨模块联动架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           数据库层 (Database)                                  │
│  ┌─────────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │
│  │ directus_collections│  │ directus_fields  │  │ directus_relations   │   │
│  │ (CollectionMeta)    │  │ (FieldMeta)      │  │ (Relation)           │   │
│  └──────────┬──────────┘  └────────┬─────────┘  └──────────┬───────────┘   │
└─────────────┼──────────────────────┼────────────────────────┼────────────────┘
              │                      │                        │
              └──────────────────────┼────────────────────────┘
                                     │
                    ┌────────────────┴────────────────┐
                    │         元数据聚合层              │
                    │  api/src/utils/get-schema.ts    │
                    │      → SchemaOverview            │
                    └────────────────┬────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
         ▼                           ▼                           ▼
┌────────────────┐        ┌────────────────────┐        ┌────────────────┐
│   API 层       │        │    共享类型定义      │        │   App 层       │
│  (GraphQL)     │◄──────►│  @directus/types   │◄──────►│  (Vue 3)       │
└────────────────┘        └────────────────────┘        └────────────────┘
         │                                                   │
         ▼                                                   ▼
┌─────────────────────────┐                      ┌─────────────────────────┐
│  GraphQL Schema 生成     │                      │  Pinia Stores           │
│  - getReadableTypes()   │                      │  - useCollectionsStore  │
│  - getWritableTypes()   │                      │  - useFieldsStore       │
│  - generateSchema()     │                      │  - useRelationsStore    │
└─────────────────────────┘                      └─────────────────────────┘
         │                                                   │
         ▼                                                   ▼
┌─────────────────────────┐                      ┌─────────────────────────┐
│  动态 API 端点           │                      │  动态 UI 渲染           │
│  - ItemsService         │                      │  - Interface 组件        │
│  - REST /items/:collection│                     │  - Display 组件          │
│  - GraphQL Queries      │                      │  - 表单/列表/导航        │
└─────────────────────────┘                      └─────────────────────────┘
```

---

## 7. 关键代码位置索引

### 7.1 类型定义层

| 功能 | 文件路径 |
|------|----------|
| Collection 类型 | `packages/types/src/collection.ts` |
| Field 类型 | `packages/types/src/fields.ts` |
| SchemaOverview | `packages/types/src/schema.ts` |
| Relation 类型 | `packages/types/src/relations.ts` |

### 7.2 API 层

| 功能 | 文件路径 |
|------|----------|
| SchemaOverview 构建 | `api/src/utils/get-schema.ts` |
| GraphQL Schema 生成 | `api/src/services/graphql/schema/index.ts` |
| GraphQL 可读类型 | `api/src/services/graphql/schema/read.ts` |
| GraphQL 可写类型 | `api/src/services/graphql/schema/write.ts` |
| ItemsService | `api/src/services/items.ts` |
| CollectionsService | `api/src/services/collections.ts` |
| FieldsService | `api/src/services/fields.ts` |
| Schema 中间件 | `api/src/middleware/schema.ts` |

### 7.3 App 层

| 功能 | 文件路径 |
|------|----------|
| Collections Store | `app/src/stores/collections.ts` |
| Fields Store | `app/src/stores/fields.ts` |
| Relations Store | `app/src/stores/relations.ts` |
| SchemaOverview Composable | `app/src/composables/use-schema.ts` |
| Interfaces 目录 | `app/src/interfaces/` |
| Displays 目录 | `app/src/displays/` |
| Layouts 目录 | `app/src/layouts/` |

---

## 8. 核心设计要点

### 8.1 单一数据源原则

Collection 和 Field 元数据只存储一次，但被多个模块消费：
- **存储**: `directus_collections` 和 `directus_fields` 表
- **API 消费**: `getSchema()` → `SchemaOverview` → GraphQL/REST
- **App 消费**: `/collections` `/fields` API → Pinia Stores → UI

### 8.2 字段职责分离

Field 配置中有明确的职责分离：

| 配置类别 | 字段示例 | 消费方 |
|----------|----------|--------|
| 数据库层 | `schema.is_nullable`, `schema.default_value` | Knex/Schema Inspector |
| API 层 | `type`, `special`, `validation` | GraphQL 类型生成, 权限验证 |
| UI 层 | `interface`, `display`, `options`, `width`, `hidden` | App 渲染 |
| 共享层 | `required`, `readonly`, `translations` | API 验证 + UI 显示 |

### 8.3 缓存策略

- **API 端**: `getSchema()` 使用内存缓存，通过 `CACHE_SCHEMA` 环境变量控制
- **App 端**: Stores 作为内存缓存，通过 `hydrate()` 方法刷新

### 8.4 权限感知

SchemaOverview 在生成时会考虑权限：
- `fetchAllowedFieldMap()` 检查字段级权限
- `reduceSchema()` 过滤无权限访问的集合和字段

---

## 9. 数据流示例

### 9.1 创建新 Collection 的完整流程

```
1. 用户在 App 中创建 Collection 配置
   ↓ App/src/modules/settings/routes/data-model
   
2. App 调用 POST /collections API
   ↓ api/src/controllers/collections.ts
   
3. CollectionsService.createOne() 执行:
   a. 创建数据库表 (Knex schema.createTable)
   b. 插入 directus_collections 记录
   c. 插入 directus_fields 记录（包括主键字段）
   ↓ api/src/services/collections.ts:63-260
   
4. 清除系统缓存，触发 schema 重新生成
   ↓ clearSystemCache()
   
5. 下次请求时:
   - API: getSchema() 重新构建 SchemaOverview
   - App: hydrate() 重新获取元数据
   
6. 结果:
   - 新的 REST 端点 /items/:new-collection 可用
   - 新的 GraphQL 类型和查询可用
   - App 侧边栏显示新集合
   - App 可以渲染该集合的表单和列表
```

### 9.2 GraphQL 查询的元数据驱动示例

假设有一个 Field 配置：
```typescript
{
    field: 'title',
    type: 'string',
    meta: {
        interface: 'input',
        options: { placeholder: 'Enter title' },
        required: true
    },
    schema: { is_nullable: false }
}
```

**API 端影响**:
- GraphQL 类型中 `title` 字段为 `String!`（非空）
- 过滤器支持 `_eq`, `_contains`, `_starts_with` 等字符串操作符
- 创建/更新时验证非空

**App 端影响**:
- 表单中渲染 `<v-input>` 组件
- 显示必填星号
- 显示占位符 "Enter title"
- 列表中显示为纯文本

---

## 10. 总结

Directus 的元数据驱动架构核心在于：

1. **统一的元数据模型**：`CollectionMeta` 和 `FieldMeta` 定义了所有配置
2. **共享的数据结构**：`SchemaOverview` 作为 API 和 App 之间的桥梁
3. **分层消费**：不同层级消费元数据的不同部分
4. **动态生成**：API 结构和 UI 组件完全由配置决定

这种架构使得 Directus 能够：
- ✅ 无需代码变更即可添加新的数据模型
- ✅ API 和管理后台自动保持同步
- ✅ 高度可扩展的 Interface/Display 插件系统
- ✅ 统一的权限和验证机制
