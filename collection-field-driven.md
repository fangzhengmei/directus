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

### 2.2 FieldMeta 定义

**文件位置**: `packages/types/src/fields.ts:36-59`

```typescript
export type FieldMeta = {
    id: number;                    // 字段 ID
    collection: string;            // 所属集合
    field: string;                 // 字段名称
    group: string | null;          // 字段组
    hidden: boolean;               // 是否隐藏
    interface: string | null;      // 界面组件类型（如 'input', 'select'）
    display: string | null;        // 显示组件类型
    options: Record<string, any>;  // 界面配置选项
    display_options: Record<string, any>; // 显示配置选项
    readonly: boolean;             // 是否只读
    required: boolean;             // 是否必填
    sort: number | null;           // 排序序号
    special: string[] | null;      // 特殊类型标记（如 'file', 'm2o', 'o2m'）
    translations: Translations[];  // 多语言翻译
    width: Width | null;           // 表单宽度
    note: string | null;           // 字段说明
    conditions: Condition[];       // 条件逻辑
    validation: Filter | null;     // 验证规则
    validation_message: string;    // 验证失败消息
    searchable: boolean;           // 是否可搜索
    system?: true;                 // 是否系统字段
};
```

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

### 4.3 REST API 动态路由

REST API 通过 `ItemsService` 动态处理所有集合的 CRUD 操作：

**文件位置**: `api/src/services/items.ts`

`ItemsService` 使用 `SchemaOverview` 来：
- 验证字段是否存在
- 处理关系字段的特殊逻辑
- 应用权限过滤
- 处理文件上传、版本控制等特殊行为

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
