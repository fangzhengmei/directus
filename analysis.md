# Directus 管理后台字段类型与展示组件映射分析

## 1. 概述

Directus 通过多层映射机制实现从数据库字段类型到管理后台展示组件的灵活配置。本文档详细分析字段类型与展示组件的映射规则，以及扩展配置从后端到前端的传播路径。

## 2. 字段类型系统

### 2.1 类型层次结构

Directus 使用三层类型系统：

| 层级 | 说明 | 示例 |
|------|------|------|
| 数据库原生类型 | 各数据库（MySQL、PostgreSQL、MS SQL 等）的原始列类型 | `varchar`, `int4`, `nvarchar` |
| Directus 类型 (TYPES) | 标准化的字段类型，用于跨数据库兼容 | `string`, `integer`, `boolean`, `json`, `geometry.Point` |
| 本地类型 (LOCAL_TYPES) | 逻辑类型，用于描述字段的关系或特殊用途 | `m2o`, `o2m`, `m2m`, `file`, `translations` |

### 2.2 类型定义

**数据库原生类型 (KNEX_TYPES)** (`packages/constants/src/fields.ts:1-16`)：
- `bigInteger`, `boolean`, `date`, `dateTime`, `decimal`, `float`, `integer`, `json`, `string`, `text`, `time`, `timestamp`, `binary`, `uuid`

**Directus 类型 (TYPES)** (`packages/constants/src/fields.ts:18-57`)：
- 在 KNEX_TYPES 基础上增加：`alias`, `hash`, `csv`, `geometry`, `geometry.Point`, `geometry.LineString`, `geometry.Polygon`, `geometry.MultiPoint`, `geometry.MultiLineString`, `geometry.MultiPolygon`, `unknown`

**本地类型 (LOCAL_TYPES)** (`packages/constants/src/fields.ts:72-83`)：
- `standard` - 标准字段
- `file`, `files` - 文件/多文件
- `m2o`, `o2m`, `m2m`, `m2a` - 关系类型
- `presentation` - 展示型字段
- `translations` - 多语言翻译
- `group` - 字段组

**关系类型 (RELATIONAL_TYPES)** (`packages/constants/src/fields.ts:85`)：
- `file`, `files`, `m2o`, `o2m`, `m2m`, `m2a`, `translations`

## 3. 字段类型映射规则

### 3.1 数据库类型到 Directus 类型的映射

**后端映射逻辑** (`api/src/utils/get-local-type.ts:1-153`)：

映射通过 `localTypeMap` 对象实现，支持多种数据库：

**通用映射**：
- `boolean`, `tinyint`, `smallint`, `mediumint`, `int`, `integer`, `serial` → `integer`
- `bigint`, `bigserial` → `bigInteger`
- `clob`, `tinytext`, `mediumtext`, `longtext`, `text` → `text`
- `varchar`, `longvarchar`, `varchar2`, `nvarchar` → `string`
- `date` → `date`, `datetime` → `dateTime`, `timestamp` → `timestamp`
- `float`, `double`, `double precision`, `real` → `float`
- `decimal` → `decimal`, `numeric` → `integer`

**MySQL 特定**：
- `string` → `text`, `year` → `integer`, `blob`, `mediumblob` → `binary`
- 无符号整数类型映射到相应的 integer 类型

**MS SQL 特定**：
- `bit` → `boolean`, `smallmoney`, `money` → `float`
- `datetimeoffset` → `timestamp`, `datetime2`, `smalldatetime` → `dateTime`
- `nchar` → `text`, `uniqueidentifier` → `uuid`

**PostgreSQL / CockroachDB 特定**：
- `json`, `jsonb` → `json`, `uuid` → `uuid`
- `int2`, `serial4`, `int4` → `integer`, `serial8`, `int8` → `bigInteger`
- `bool` → `boolean`, `character varying`, `character` → `string`
- `timestamptz` → `timestamp`, `float4` → `float`, `float8` → `float`

**特殊处理** (`api/src/utils/get-local-type.ts:121-132`)：
```typescript
if (special) {
    if (special.includes('cast-json')) return 'json';
    if (special.includes('hash')) return 'hash';
    if (special.includes('cast-csv')) return 'csv';
    if (special.includes('uuid') || special.includes('file')) return 'uuid';
    if (special.includes('cast-timestamp')) return 'timestamp';
    if (special.includes('cast-datetime')) return 'dateTime';
}
```

### 3.2 前端本地类型判断

**前端本地类型判断逻辑** (`app/src/utils/get-local-type.ts:6-64`)：

`getLocalTypeForField(collection, field)` 函数根据字段信息和关系数据判断本地类型：

1. **无关系的字段**：
   - `alias` 类型且 special 包含 `group` → `group`
   - `alias` 类型 → `presentation`
   - 其他 → `standard`

2. **单个关系的字段**：
   - 关联到 `directus_files` → `file`
   - 当前字段在关系的多端 (many) → `m2o` (Many-to-One)
   - 其他 → `o2m` (One-to-Many)

3. **两个关系的字段**：
   - special 包含 `translations` → `translations`
   - special 包含 `m2a` → `m2a` (Many-to-Any)
   - 关联到 `directus_files` → `files` (多文件)
   - 其他 → `m2m` (Many-to-Many)

### 3.3 双关系字段的精确决策链

**双关系字段决策流程图** (`app/src/utils/get-local-type.ts:34-61`)：

```
字段有 2 个关系？
    │
    ├─ meta.special 包含 'translations'?
    │   └─ 是 → translations
    │
    ├─ meta.special 包含 'm2a'?
    │   └─ 是 → m2a
    │
    ├─ 当前字段是关系的多端 (many side)?
    │   └─ 是 → m2o
    │
    ├─ 关联到 directus_files?
    │   ├─ collection != directus_files 且 1 个关系关联 → files
    │   └─ 2 个关系都关联 → files (自关联多文件)
    │
    └─ 其他 → m2m
```

**决策逻辑代码分析**：

```typescript
if (relations.length === 2) {
    // 优先级 1: special 标记优先
    if ((fieldInfo.meta?.special || []).includes('translations')) {
        return 'translations';
    }

    if ((fieldInfo.meta?.special || []).includes('m2a')) {
        return 'm2a';
    }

    // 优先级 2: 当前字段在多端 (many side) → m2o
    // 这种情况常见于 M2M 关系中通过中间表访问多对一端
    const relationForCurrent = getRelation(relations, collection, field);
    if (relationForCurrent?.collection === collection && 
        relationForCurrent?.field === field) {
        return 'm2o';
    }

    // 优先级 3: 关联到 directus_files → files
    const directusFilesRelationsCount = relations.filter(
        (relation) => relation.related_collection === 'directus_files',
    ).length;

    const isRelationToDirectusFiles = collection !== 'directus_files' && 
                                     directusFilesRelationsCount === 1;
    const isSelfRelationToDirectusFiles = directusFilesRelationsCount === 2;

    if (isRelationToDirectusFiles || isSelfRelationToDirectusFiles) {
        return 'files';
    }
    
    // 优先级 4: 默认 m2m
    else {
        return 'm2m';
    }
}
```

**关键判定条件详解**：

| 条件 | 判定逻辑 | 示例场景 |
|------|---------|---------|
| `special.includes('translations')` | 字段被标记为多语言翻译字段 | `directus_languages` 表的多语言支持 |
| `special.includes('m2a')` | 字段被标记为多对任意 | 动态关联到多个不同集合 |
| `relation.collection === collection && relation.field === field` | 当前字段存储外键，在关系的多端 | 中间表的外键字段 |
| `related_collection === 'directus_files'` | 关系目标是文件管理表 | 多文件上传字段 |
| 以上都不满足 | 标准多对多关系 | 文章 ↔ 标签的多对多 |

### 3.4 数据库类型修正条件

**后端类型修正逻辑** (`api/src/utils/get-local-type.ts:134-150`)：

在基础映射之后，还有三个条件修正类型：

#### 3.4.1 PostgreSQL numeric 类型修正

```typescript
/** Handle Postgres numeric decimals */
if (dataType === 'numeric' && 
    column.numeric_precision !== null && 
    column.numeric_scale !== null) {
    return 'decimal';
}
```

**修正条件**：
- 数据库类型为 `numeric`
- `numeric_precision` 不为 `null`（有精度定义）
- `numeric_scale` 不为 `null`（有小数位数定义）

**修正前**：
- 基础映射：`numeric` → `integer`（`localTypeMap` 第 36 行）

**修正后**：
- 有 precision 和 scale → `decimal`

**应用场景**：
```sql
-- PostgreSQL
-- 以下会被修正为 decimal
ALTER TABLE products ADD COLUMN price NUMERIC(10, 2);
ALTER TABLE products ADD COLUMN discount NUMERIC(5, 4);

-- 以下保持为 integer（无 precision/scale）
ALTER TABLE products ADD COLUMN stock NUMERIC;
```

#### 3.4.2 MS SQL varchar(MAX) / nvarchar(MAX) 修正

```typescript
/** Handle MS SQL varchar(MAX) and nvarchar(MAX) types */
if (
    (column.data_type === 'nvarchar' || column.data_type === 'varchar') &&
    (column.max_length === -1 || column.max_length === null)
) {
    return 'text';
}
```

**修正条件**：
- 数据库类型为 `varchar` 或 `nvarchar`
- `max_length` 为 `-1` 或 `null`（表示 MAX）

**修正前**：
- 基础映射：`varchar` → `string`, `nvarchar` → `string`（`localTypeMap` 第 19、22 行）

**修正后**：
- `varchar(MAX)` / `nvarchar(MAX)` → `text`

**应用场景**：
```sql
-- MS SQL Server
-- 以下会被修正为 text
ALTER TABLE articles ADD COLUMN content NVARCHAR(MAX);
ALTER TABLE articles ADD COLUMN excerpt VARCHAR(MAX);

-- 以下保持为 string（有长度限制）
ALTER TABLE articles ADD COLUMN title NVARCHAR(200);
```

#### 3.4.3 CockroachDB 64 位整数修正

```typescript
/** Handle CockroachDB 64-bit integers (reported as 'integer' with precision 64) */
if ((dataType === 'integer' || dataType === 'int') && 
    column.numeric_precision === 64) {
    return 'bigInteger';
}
```

**修正条件**：
- 数据库类型为 `integer` 或 `int`
- `numeric_precision` 等于 `64`

**修正前**：
- 基础映射：`integer` → `integer`（`localTypeMap` 第 10 行）

**修正后**：
- 精度为 64 的整数 → `bigInteger`

**应用场景**：
```sql
-- CockroachDB
-- 以下会被修正为 bigInteger
CREATE TABLE orders (
    id INT8 PRIMARY KEY,  -- INT8 在 CockroachDB 中是 64 位整数
    amount INTEGER        -- 但如果 precision 为 64，也会被修正
);
```

**类型修正优先级**：

```
原始数据库类型
    ↓
localTypeMap 基础映射
    ↓
special 字段覆盖（cast-json, hash, cast-csv, uuid, cast-timestamp, cast-datetime, geometry）
    ↓
PostgreSQL numeric 精度修正
    ↓
MS SQL varchar(MAX) 修正
    ↓
CockroachDB 64-bit 整数修正
    ↓
最终 Directus 类型
```

## 4. 接口与展示组件系统

### 4.1 接口组件定义

接口组件在 `app/src/interfaces/*/index.ts` 中定义，使用 `defineInterface` 函数：

**示例：输入框接口** (`app/src/interfaces/input/index.ts:7-234`)：
```typescript
export default defineInterface({
    id: 'input',
    name: '$t:interfaces.input.input',
    description: '$t:interfaces.input.description',
    icon: 'text_fields',
    component: InterfaceInput,
    types: ['string', 'uuid', 'bigInteger', 'integer', 'float', 'decimal', 'text'],
    group: 'standard',
    options: ({ field }) => { /* 配置选项 */ },
    preview: PreviewSVG,
});
```

**示例：布尔开关接口** (`app/src/interfaces/boolean/index.ts:5-74`)：
```typescript
export default defineInterface({
    id: 'boolean',
    name: '$t:interfaces.boolean.toggle',
    types: ['boolean'],
    group: 'selection',
    recommendedDisplays: ['boolean'],
    options: [/* 配置选项 */],
});
```

**关键属性**：
- `id` - 唯一标识符
- `types` - 支持的 Directus 类型数组
- `localTypes` - 支持的本地类型数组（默认 `['standard']`）
- `component` - Vue 组件
- `options` - 配置选项（数组或函数）
- `recommendedDisplays` - 推荐的展示组件
- `group` - 分组（standard, selection, relational, presentation, groups）

### 4.2 展示组件定义

展示组件在 `app/src/displays/*/index.ts` 中定义，使用 `defineDisplay` 函数：

**示例：布尔展示** (`app/src/displays/boolean/index.ts:4-79`)：
```typescript
export default defineDisplay({
    id: 'boolean',
    name: '$t:displays.boolean.boolean',
    types: ['boolean'],
    icon: 'check_box',
    component: DisplayBoolean,
    options: [/* 配置选项 */],
});
```

### 4.3 默认接口与展示组件映射

**默认接口映射** (`app/src/utils/get-default-interface-for-type.ts:3-33`)：

| 字段类型 | 默认接口 |
|---------|---------|
| `string`, `uuid`, `bigInteger`, `integer`, `float`, `decimal`, `alias`, `unknown`, `binary` | `input` |
| `boolean` | `boolean` |
| `date`, `dateTime`, `time`, `timestamp` | `datetime` |
| `json` | `input-code` |
| `text` | `input-multiline` |
| `csv` | `tags` |
| `hash` | `input-hash` |
| 所有 `geometry*` 类型 | `map` |

**默认展示组件映射** (`app/src/utils/get-default-display-for-type.ts:3-33`)：

| 字段类型 | 默认展示组件 |
|---------|-------------|
| `string`, `text`, `uuid`, `bigInteger`, `hash` | `formatted-value` |
| `integer`, `float`, `decimal` | `formatted-value` |
| `boolean` | `boolean` |
| `date`, `dateTime`, `time`, `timestamp` | `datetime` |
| `alias`, `binary`, `json`, `unknown`, `geometry*` | `raw` |
| `csv` | `labels` |

### 4.4 字段配置中的接口/展示指定

字段的元数据存储在 `directus_fields` 表中，`FieldMeta` 类型定义 (`packages/types/src/fields.ts:36-59`)：

```typescript
export type FieldMeta = {
    id: number;
    collection: string;
    field: string;
    interface: string | null;      // 接口组件 ID
    display: string | null;        // 展示组件 ID
    options: Record<string, any> | null;           // 接口配置选项
    display_options: Record<string, any> | null;   // 展示配置选项
    special: string[] | null;      // 特殊标记（关系、自动生成等）
    // ... 其他属性
};
```

## 5. 字段信息从后端到前端的传播

### 5.1 后端字段服务

**FieldsService** (`api/src/services/fields.ts:68-1053`) 负责字段的 CRUD 操作。

**字段读取流程** (`api/src/services/fields.ts:126-281`)：

`readAll(collection?)` 方法：

1. **读取字段元数据**：
   - 从 `directus_fields` 表读取自定义字段
   - 添加系统字段（`systemFieldRows`）

2. **读取数据库列信息**：
   - 调用 `columnInfo()` 获取数据库 schema
   - 处理默认值转换

3. **组合字段对象**：
   - 调用 `getLocalType(column, field)` 确定 Directus 类型
   - 组合成 `Field` 对象：`{ collection, field, type, schema, meta }`

4. **处理别名字段**：
   - 筛选 `special` 包含别名类型的字段
   - 为这些字段创建虚拟 Field 对象

5. **权限过滤**：
   - 根据用户权限过滤可访问的字段

6. **数据库特定处理**：
   - 调用 `helpers.schema.processFieldType(field)` 进行数据库特定的类型调整

**返回的 Field 结构**：
```typescript
interface Field {
    collection: string;
    field: string;
    type: Type;                    // Directus 类型
    schema: Column | null;         // 数据库列信息
    meta: FieldMeta | null;        // 字段元数据（interface, display, options 等）
    name: string;                  // 显示名称
    children?: Field[] | null;     // 子字段（用于字段组）
}
```

### 5.2 后端 API 端点

**字段控制器** (`api/src/controllers/fields.ts:1-252`) 提供 REST API：

| 端点 | 方法 | 功能 |
|------|------|------|
| `/fields` | GET | 获取所有集合的所有字段 |
| `/fields/:collection` | GET | 获取指定集合的所有字段 |
| `/fields/:collection/:field` | GET | 获取指定字段详情 |
| `/fields/:collection` | POST | 创建新字段 |
| `/fields/:collection` | PATCH | 批量更新字段 |
| `/fields/:collection/:field` | PATCH | 更新指定字段 |
| `/fields/:collection/:field` | DELETE | 删除字段 |

### 5.3 前端字段存储

**Fields Store** (`app/src/stores/fields.ts:70-423`) 管理前端字段状态。

**字段加载流程** (`app/src/stores/fields.ts:101-107`)：

```typescript
async function hydrate(options?: HydrateOptions) {
    const fieldsResponse = await api.get<any>(`/fields`);
    const fieldsRaw: FieldRaw[] = fieldsResponse.data.data;
    fields.value = [...fieldsRaw.map(parseField), fakeFilesField];
    if (options?.skipTranslation !== true) translateFields();
}
```

**字段解析** (`app/src/stores/fields.ts:113-155`)：
- 格式化字段名称
- 处理多语言翻译
- 合并 i18n 翻译

## 6. 扩展配置从后端到前端的传播路径

### 6.1 后端扩展管理

**扩展管理器** (`api/src/extensions/manager.ts`) 负责扩展的加载和管理。

**扩展存储**：
- 扩展元数据存储在 `directus_extensions` 表
- 扩展文件存储在 `extensions/` 目录

**扩展 API** (`api/src/controllers/`)：
- `/extensions` - 获取扩展列表
- `/extensions/:id` - 管理单个扩展
- `/extensions/registry/install` - 从市场安装
- `/extensions/registry/uninstall/:id` - 卸载扩展

### 6.2 前端扩展加载

**扩展加载流程** (`app/src/main.ts:24-72`)：

应用启动时的扩展初始化：

```typescript
async function init() {
    const app = createApp(App);
    // ... 其他初始化
    
    await loadExtensions();      // 1. 加载扩展
    registerExtensions(app);      // 2. 注册扩展
    
    app.use(router);
    useSystem(app);
    app.mount('#app');
}
```

**扩展加载** (`app/src/extensions.ts:31-42`)：

```typescript
export async function loadExtensions(): Promise<void> {
    try {
        customExtensions = import.meta.env.DEV
            ? await import(/* @vite-ignore */ '@directus-extensions')
            : await import(/* @vite-ignore */ `${getRootPath()}extensions/sources/index.js`);
    } catch (err) {
        console.warn(`Couldn't load extensions`);
    }
}
```

**加载策略**：
- **开发环境**：从 Vite 虚拟模块 `@directus-extensions` 加载
- **生产环境**：从 `/extensions/sources/index.js` 动态导入

### 6.3 前端扩展注册

**扩展注册** (`app/src/extensions.ts:44-95`)：

```typescript
export function registerExtensions(app: App): void {
    // 1. 获取内置扩展
    const interfaces = getInternalInterfaces();
    const displays = getInternalDisplays();
    const layouts = getInternalLayouts();
    const modules = getInternalModules();
    const panels = getInternalPanels();
    const operations = getInternalOperations();
    const themes = [];

    // 2. 合并自定义扩展
    if (customExtensions !== null) {
        interfaces.push(...customExtensions.interfaces);
        displays.push(...customExtensions.displays);
        layouts.push(...customExtensions.layouts);
        modules.push(...customExtensions.modules);
        panels.push(...customExtensions.panels);
        operations.push(...customExtensions.operations);
        themes.push(...customExtensions.themes);
    }

    // 3. 注册到 Vue 应用
    registerInterfaces(interfaces, app);
    registerDisplays(displays, app);
    registerLayouts(layouts, app);
    registerPanels(panels, app);
    registerOperations(operations, app);
    registerThemes(themes);

    // 4. 监听语言变化，翻译扩展名称
    watch(i18n.global.locale, () => {
        extensions.interfaces.value = translate(interfaces);
        extensions.displays.value = translate(displays);
        // ...
    }, { immediate: true });
}
```

### 6.4 内置扩展自动发现

**接口自动发现** (`app/src/interfaces/index.ts:5-12`)：

```typescript
export function getInternalInterfaces(): InterfaceConfig[] {
    const interfaces = import.meta.glob<InterfaceConfig>(
        ['./*/index.ts', './_system/*/index.ts'], 
        { import: 'default', eager: true }
    );
    return sortBy(Object.values(interfaces), 'id');
}
```

使用 Vite 的 `import.meta.glob` 自动扫描 `app/src/interfaces/*/index.ts` 和 `app/src/interfaces/_system/*/index.ts` 文件。

**接口注册** (`app/src/interfaces/index.ts:14-22`)：

```typescript
export function registerInterfaces(interfaces: InterfaceConfig[], app: App): void {
    for (const inter of interfaces) {
        // 注册主组件：interface-{id}
        app.component(`interface-${inter.id}`, inter.component);
        
        // 注册选项组件（如果存在）
        if (typeof inter.options !== 'function' && 
            Array.isArray(inter.options) === false && 
            inter.options !== null) {
            app.component(`interface-options-${inter.id}`, inter.options);
        }
    }
}
```

**展示组件注册** (`app/src/displays/index.ts:11-19`)：

```typescript
export function registerDisplays(displays: DisplayConfig[], app: App): void {
    for (const display of displays) {
        app.component(`display-${display.id}`, display.component);
        // ... 选项组件注册
    }
}
```

### 6.5 前端扩展使用

**扩展 Store** (`app/src/stores/extensions.ts:36-123`)：

```typescript
export const useExtensionsStore = defineStore('extensions', () => {
    const extensions = ref<ApiOutput[]>([]);
    
    const refresh = async (forceRefresh = true) => {
        const response = await api.get('/extensions');
        extensions.value = response.data.data;
        // 检测扩展变化，提示用户刷新
    };
    
    return { extensions, refresh, /* 其他操作 */ };
});
```

**扩展查找** (`app/src/composables/use-extension.ts:7-17`)：

```typescript
export function useExtension<T extends AppExtensionType | HybridExtensionType>(
    type: T | Ref<T>,
    name: string | Ref<string | null>,
): Ref<AppExtensionConfigs[Plural<T>][number] | null> {
    const extensions = useExtensions();
    
    return computed(() => {
        if (unref(name) === null) return null;
        return (extensions[pluralize(unref(type))].value as any[])
            .find(({ id }) => id === unref(name)) ?? null;
    });
}
```

## 7. 接口与展示组件的选择与渲染

### 7.1 字段编辑时的接口选择

**字段详情 Store** (`app/src/modules/settings/routes/data-model/field-detail/store/index.ts:339-367`) 提供可用接口/展示的过滤：

```typescript
interfacesForType(): InterfaceConfig[] {
    const { interfaces } = useExtensions();
    
    return orderBy(
        interfaces.value.filter((inter: InterfaceConfig) => {
            if (inter.system === true) return false;
            const matchesType = inter.types.includes(this.field.type || 'alias');
            const matchesLocalType = (inter.localTypes || ['standard']).includes(this.localType);
            return matchesType && matchesLocalType;
        }),
        ['name'],
    );
},

displaysForType(): DisplayConfig[] {
    const { displays } = useExtensions();
    
    return orderBy(
        displays.value.filter((inter: DisplayConfig) => {
            const matchesType = inter.types.includes(this.field.type || 'alias');
            const matchesLocalType = (inter.localTypes || ['standard']).includes(this.localType);
            return matchesType && matchesLocalType;
        }),
        ['name'],
    );
},
```

**过滤逻辑**：
1. 排除系统接口（`inter.system === true`）
2. 匹配字段类型：`inter.types.includes(this.field.type)`
3. 匹配本地类型：`inter.localTypes.includes(this.localType)`

### 7.2 表单中的接口渲染

**表单字段接口组件** (`app/src/components/v-form/components/form-field-interface.vue:1-104`)：

```typescript
// 1. 获取接口配置
const inter = useExtension(
    'interface',
    computed(() => props.field?.meta?.interface ?? 'input'),
);

// 2. 确定组件名称
const componentName = computed(() => {
    return props.field?.meta?.interface
        ? `interface-${props.field.meta.interface}`
        : `interface-${getDefaultInterfaceForType(props.field.type!)}`;
});
```

**模板渲染**：
```vue
<template>
    <VErrorBoundary v-if="interfaceExists" :name="componentName">
        <component
            :is="componentName"
            v-bind="(field.meta && field.meta.options) || {}"
            :value="value"
            :type="field.type"
            :collection="field.collection"
            :field="field.field"
            :field-data="field"
            @input="$emit('update:modelValue', $event)"
        />
    </VErrorBoundary>
</template>
```

**组件选择优先级**：
1. 字段配置的 `meta.interface`
2. 默认接口 `getDefaultInterfaceForType(field.type)`

**传递给接口组件的属性**：
- `options` - 接口配置选项（从 `meta.options`）
- `value` - 字段值
- `type` - 字段类型
- `collection` - 集合名称
- `field` - 字段名
- `field-data` - 完整字段数据
- 其他：`disabled`, `readonly`, `width`, `primary-key` 等

### 7.3 列表中的展示渲染

**展示组件渲染** (`app/src/views/private/components/render-display.vue:1-49`)：

```typescript
const displayInfo = useExtension('display', display);
```

**模板渲染**：
```vue
<template>
    <ValueNull v-if="value === null || value === undefined" />
    <VTextOverflow v-else-if="displayInfo === null" class="display" :text="value" />
    <VErrorBoundary v-else :name="`display-${display}`">
        <component
            :is="`display-${display}`"
            v-bind="options"
            :interface="interface"
            :interface-options="interfaceOptions"
            :value="value"
            :type="type"
            :collection="collection"
            :field="field"
        />
    </VErrorBoundary>
</template>
```

**展示组件降级策略**：
1. 配置的展示组件
2. 如果展示组件不存在，使用 `VTextOverflow` 显示原始值
3. 如果值为 null，使用 `ValueNull` 组件

### 7.4 表格布局中的展示

**表格布局** (`app/src/layouts/tabular/tabular.vue` 及相关组件) 使用 `RenderDisplay` 组件渲染单元格内容，根据字段的 `meta.display` 和 `meta.display_options` 配置选择合适的展示组件。

### 7.5 Tabular 路径：字段级别展示组件选择与回退

#### 7.5.1 Tabular 路径概览

**Tabular 路径** 是字段级别的展示路径，用于数据表格的单元格渲染。

```
路径入口：Tabular 布局单元格渲染
    ↓
Tabular Header 配置阶段
    ├─ 从 FieldsStore 获取字段信息
    ├─ 确定 display = meta.display || getDefaultDisplayForType
    └─ 构建 header.field 配置
    ↓
RenderDisplay 组件渲染阶段
    ├─ 检查值状态 (null/undefined)
    ├─ 检查扩展是否存在 (useExtension)
    ├─ 渲染展示组件
    └─ 错误边界处理
```

#### 7.5.2 阶段一：Tabular Header 配置

**Header 构建逻辑** (`app/src/layouts/tabular/index.ts:247-284`)：

```typescript
const tableHeaders = computed<HeaderRaw[]>({
    get() {
        return activeFields.value.map((field) => {
            return {
                // ...
                field: {
                    // 优先级 1: meta.display 已配置?
                    //   是 → 使用配置值
                    //   否 → 调用 getDefaultDisplayForType(field.type)
                    display: field.meta?.display || getDefaultDisplayForType(field.type),
                    displayOptions: field.meta?.display_options,
                    interface: field.meta?.interface,
                    interfaceOptions: field.meta?.options,
                    type: field.type,
                    field: field.field,
                    collection: field.collection,
                },
                // ...
            } as HeaderRaw;
        });
    },
});
```

**阶段一决策优先级（display 确定）**：

```
字段 Field 对象
    │
    ├─ meta.display !== null && meta.display !== undefined?
    │   ├─ 是 → display = meta.display (用户配置)
    │   └─ 否 → display = getDefaultDisplayForType(field.type) (系统默认)
    │
    └─ display 确定完成 → 传递给 RenderDisplay
```

**默认展示组件映射表** (`app/src/utils/get-default-display-for-type.ts:3-33`)：

| Directus 类型 | 默认 display |
|--------------|-------------|
| `alias`, `binary`, `json`, `unknown`, `geometry*` | `'raw'` |
| `boolean` | `'boolean'` |
| `date`, `dateTime`, `time`, `timestamp` | `'datetime'` |
| `csv` | `'labels'` |
| `string`, `text`, `uuid`, `bigInteger`, `integer`, `float`, `decimal`, `hash` | `'formatted-value'` |

#### 7.5.3 阶段二：RenderDisplay 组件渲染

**RenderDisplay 组件** (`app/src/views/private/components/render-display.vue:1-49`)：

```typescript
// 输入参数
const props = defineProps<{
    display: string | null;       // 来自阶段一的结果
    options?: Record<string, unknown>;
    value?: any;
    type: string;
    collection: string;
    field: string;
}>();

// 步骤 1: 检查扩展是否存在
const displayInfo = useExtension('display', display);
```

**RenderDisplay 完整决策链（含所有回退）**：

```
RenderDisplay(props: { display, value, ... })
    │
    ├─ 优先级 1: 检查值状态
    │   ├─ value === null || value === undefined?
    │   │   └─ 是 → ValueNull 组件 (显示 "--")
    │   │
    │   └─ 值有效 → 继续
    │
    ├─ 优先级 2: 检查展示扩展是否存在
    │   ├─ displayInfo = useExtension('display', display)
    │   │
    │   └─ displayInfo === null (扩展未注册)?
    │       └─ 是 → VTextOverflow (显示原始值字符串)
    │
    ├─ 优先级 3: 渲染展示组件
    │   └─ VErrorBoundary 包裹
    │       │
    │       ├─ 渲染成功
    │       │   └─ 展示组件正常渲染 (display-{id})
    │       │
    │       └─ 渲染失败 (VErrorBoundary 捕获异常)
    │           └─ fallback → VTextOverflow (显示原始值字符串)
    │
    └─ 渲染完成
```

**Tabular 路径优先级与回退总结表**：

| 优先级 | 条件 | 结果 | 组件/行为 |
|-------|------|------|----------|
| **P1** | `value === null \|\| undefined` | ValueNull | `<ValueNull>` → "--" |
| **P2** | `displayInfo === null` (扩展不存在) | 原始值 | `<VTextOverflow :text="value">` |
| **P3** | 扩展存在且渲染成功 | 展示组件 | `<component :is="display-${display}">` |
| **P4 (fallback)** | 渲染过程中抛出异常 | 原始值 | `VErrorBoundary` → `<VTextOverflow>` |

**关键注意点**：
- `display = 'raw'` 在 Tabular 路径中**不会**被特殊处理，仍然会经过 `useExtension` 检查
- `raw` 展示组件实际存在（内置扩展），所以会正常渲染
- 只有当扩展**完全不存在**时才会触发 P2 回退

#### 7.5.4 Tabular 路径端到端示例

```
场景：Tabular 表格中一个类型为 string、配置了 display='formatted-value' 的字段
值为 "Hello World"

1. FieldsStore.hydrate() 从 API 加载字段
   → field.meta.display = 'formatted-value'

2. Tabular 构建 header
   → display = 'formatted-value' (meta.display 已配置)
   → displayInfo = useExtension('display', 'formatted-value')
   → displayInfo !== null (formatted-value 是内置展示组件)

3. RenderDisplay 渲染
   → value = "Hello World" (不是 null)
   → displayInfo !== null
   → 渲染 display-formatted-value 组件
   → 结果：格式化后的值 "Hello World"

异常场景 1：display = 'non-existent-display'
   → displayInfo === null
   → VTextOverflow: 显示原始值 "Hello World"

异常场景 2：display = 'formatted-value' 但组件内部报错
   → VErrorBoundary 捕获
   → fallback: VTextOverflow 显示原始值 "Hello World"

异常场景 3：value = null
   → ValueNull 组件: 显示 "--"
```

---

### 7.6 RenderTemplate 路径：集合级别展示模板选择与回退

#### 7.6.1 RenderTemplate 路径概览

**RenderTemplate 路径** 是集合级别的展示路径，用于关系字段预览、搜索结果、页面标题等场景。

```
路径入口：关系字段预览 / 页面标题 / 搜索结果
    ↓
useTemplateData 获取模板和数据
    ├─ 确定 template = props.template || collection.meta.display_template
    ├─ 解析模板字段
    └─ 获取数据
    ↓
RenderTemplate 组件渲染
    ├─ 解析模板 ({{field1}} - {{field2}})
    ├─ 为每个模板变量确定 display
    ├─ 检查特殊情况 (raw, null 等)
    └─ 渲染展示组件或回退
```

#### 7.6.2 阶段一：模板确定与数据获取

**useTemplateData** (`app/src/composables/use-template-data.ts:16-100`)：

```typescript
// 模板选择优先级
const template = computed(() => 
    options?.template?.value ??                  // P1: 组件传入的 template prop
    collection.value?.meta?.display_template ??  // P2: 集合配置的 display_template
    null                                          // P3: 无模板
);

// 从模板中提取字段
const templateFields = computed(() => {
    if (!template.value) return null;
    return getFieldsFromTemplate(template.value);  // 解析 "{{name}} - {{code}}" → ['name', 'code']
});

// 获取数据
async function fetchTemplateValues() {
    const item = await sdk.request<Item>(
        requestEndpoint(endpoint, {
            params: {
                fields: adjustFieldsForDisplays(templateFields.value, collection),
            },
        }),
    );
    itemData.value = item;
}
```

**模板获取优先级**：

```
需要展示一条记录的标题
    │
    ├─ 调用方传入 template prop?
    │   ├─ 是 → 使用传入的模板 (P1)
    │   └─ 否 → 继续
    │
    ├─ collection.meta.display_template 已配置?
    │   ├─ 是 → 使用集合配置 (P2)
    │   └─ 否 → 无模板 (P3)
    │
    └─ 模板确定 → 传递给 RenderTemplate
```

#### 7.6.3 阶段二：RenderTemplate 模板解析与展示组件确定

**RenderTemplate 组件** (`app/src/views/private/components/render-template.vue:67-139`)：

```typescript
const parts = computed(() =>
    props.template
        .split(regex)  // "{{name}} - {{code}}" → ["", "{{name}}", " - ", "{{code}}", ""]
        .filter((p) => p)
        .map((part) => {
            // 静态文本部分（非 {{...}}）
            if (part.startsWith('{{') === false) return [part];

            // 模板变量部分：{{fieldName}}
            const fieldKey = part.replace(/{{/g, '').replace(/}}/g, '').trim();
            
            // 步骤 1: 获取值（支持嵌套路径）
            let value = getNestedValues(props.item, fieldKey);
            
            // 步骤 2: 获取字段信息
            let field: Field | null = fieldsStore.getField(props.collection!, fieldKey);

            // ========== 模板变量决策链开始 ==========
            
            // 优先级 RT1: 字段信息不存在
            if (!field) return value;  // 直接返回原始值

            // 步骤 3: 确定展示组件 ID
            const component = field?.meta?.display ||          // RT2a: meta.display
                            getDefaultDisplayForType(field.type); // RT2b: 默认 display
            const options = field?.meta?.display_options;

            // 优先级 RT3: 特殊处理 'raw' 展示组件
            // 与 Tabular 不同，RenderTemplate 会跳过 'raw' 的组件渲染
            if (component === 'raw') return value;  // 直接返回原始值

            // 步骤 4: 检查展示扩展是否存在
            const displayInfo = useExtension(
                'display',
                computed(() => component ?? null),
            );

            // 优先级 RT4: 扩展不存在
            if (!displayInfo.value) return value;  // 直接返回原始值

            // 步骤 5: 构建展示组件配置（支持数组值的智能处理）
            if (['related-values', 'formatted-value', 'formatted-json-value', 'translations']
                 .includes(component)) {
                // 数组友好组件：整个数组作为 value
                return [{
                    component, options, value,
                    // ... 其他属性
                }];
            }
            
            // 其他组件：为数组中每个值创建独立配置
            return value.map((v) => ({
                component,
                options: field.meta?.display_options,
                value: v,
                // ... 其他属性
            }));
        })
        .map((p) => p ?? null),
);
```

#### 7.6.4 阶段三：RenderTemplate 模板渲染

**模板渲染逻辑** (`app/src/views/private/components/render-template.vue:142-171`)：

```vue
<template>
    <div class="render-template">
        <template v-for="(part, index) in parts" :key="index">
            <template v-for="(subPart, subIndex) in part" :key="subIndex">
                <VErrorBoundary>
                    <!-- 优先级 RT5: 值为 null -->
                    <ValueNull v-if="subPart === null || 
                                    (typeof subPart === 'object' && subPart.value === null)" />
                    
                    <!-- 优先级 RT6: 渲染展示组件 -->
                    <template v-else-if="subPart?.component">
                        <component
                            :is="`display-${subPart.component}`"
                            v-bind="subPart.options"
                            :value="subPart.value"
                            :type="subPart.type"
                            :collection="subPart.collection"
                            :field="subPart.field"
                        />
                    </template>
                    
                    <!-- 静态文本或原始值 -->
                    <span v-else-if="typeof subPart === 'string'">
                        {{ translate(subPart) }}
                    </span>
                    <span v-else>{{ subPart }}</span>
                    
                    <!-- 优先级 RT7: 渲染错误 fallback -->
                    <template #fallback>
                        <span>{{ subPart?.value || subPart }}</span>
                    </template>
                </VErrorBoundary>
            </template>
        </template>
    </div>
</template>
```

#### 7.6.5 RenderTemplate 路径完整决策链

**模板变量决策链（单个 `{{fieldName}}`）**：

```
模板变量 {{fieldName}}
    │
    ├─ RT1: 字段信息不存在 (field === null)?
    │   └─ 是 → 原始值 (直接显示 value)
    │
    ├─ 确定展示组件 ID
    │   ├─ RT2a: meta.display 已配置? → 使用配置值
    │   └─ RT2b: 否则 → getDefaultDisplayForType(field.type)
    │
    ├─ RT3: component === 'raw'?
    │   └─ 是 → 原始值 (跳过组件渲染，直接返回 value)
    │
    ├─ RT4: 展示扩展不存在 (displayInfo === null)?
    │   └─ 是 → 原始值 (直接显示 value)
    │
    └─ 进入渲染阶段
        │
        ├─ RT5: value === null?
        │   └─ 是 → ValueNull 组件
        │
        ├─ RT6: 渲染展示组件
        │   └─ 成功 → 展示组件正常渲染
        │
        └─ RT7: 渲染失败 (VErrorBoundary 捕获)?
            └─ 是 → fallback: 原始值 ({{ subPart?.value || subPart }})
```

**RenderTemplate 路径优先级与回退总结表**：

| 优先级 | 条件 | 结果 | 行为 |
|-------|------|------|------|
| **RT1** | `field === null` (字段不存在) | 原始值 | `return value` |
| **RT2** | `component = meta.display \|\| getDefaultDisplayForType` | 确定 display ID | 不渲染，仅确定 |
| **RT3** | `component === 'raw'` | 原始值 | `return value` (跳过组件) |
| **RT4** | `displayInfo === null` (扩展不存在) | 原始值 | `return value` |
| **RT5** | `subPart === null \|\| subPart.value === null` | ValueNull | `<ValueNull>` |
| **RT6** | 扩展存在且渲染成功 | 展示组件 | `<component :is="display-${component}">` |
| **RT7 (fallback)** | 渲染过程中抛出异常 | 原始值 | `{{ subPart?.value \|\| subPart }}` |

**RenderTemplate 与 Tabular 的关键差异**：

| 差异点 | Tabular (RenderDisplay) | RenderTemplate |
|--------|------------------------|---------------|
| **`component = 'raw'`** | 正常渲染 `display-raw` 组件 | 直接返回原始值，跳过组件 |
| **扩展不存在时** | VTextOverflow 组件 | 直接返回原始值数组 |
| **渲染错误 fallback** | VTextOverflow 组件 | `{{ subPart?.value }}` 模板插值 |
| **数组值处理** | 不处理，依赖展示组件 | 智能拆分：数组友好组件 vs 逐值 |
| **静态文本** | N/A（单字段） | 支持：`"Name: " + {{name}}` |

#### 7.6.6 RenderTemplate 路径端到端示例

```
场景：集合 articles，display_template = "{{title}} - {{status}}"
记录数据：{ title: "Hello", status: "published" }

1. useTemplateData 确定模板
   → template = collection.meta.display_template = "{{title}} - {{status}}"

2. RenderTemplate 解析模板
   → parts = ["{{title}}", " - ", "{{status}}"]

3. 处理 {{title}}
   → field = fieldsStore.getField('articles', 'title')
   → field.meta.display = null
   → component = getDefaultDisplayForType('string') = 'formatted-value'
   → component !== 'raw'
   → displayInfo = useExtension('display', 'formatted-value') !== null
   → 构建配置: [{ component: 'formatted-value', value: "Hello" }]

4. 处理 " - "
   → 静态文本，直接返回

5. 处理 {{status}}
   → 类似步骤，构建配置: [{ component: 'formatted-value', value: "published" }]

6. 渲染
   → 结果："Hello" + " - " + "published"
   → 显示："Hello - published"

特殊场景 1：display = 'raw'
   → RT3: 直接返回原始值，不渲染组件

特殊场景 2：display = 'non-existent'
   → RT4: displayInfo === null → 直接返回原始值

特殊场景 3：value = null
   → RT5: ValueNull 组件显示 "--"
```

---

### 7.7 两条路径对比与端到端数据流

#### 7.7.1 路径对比总表

| 维度 | Tabular 路径 | RenderTemplate 路径 |
|------|-------------|-------------------|
| **触发场景** | 数据表格单元格 | 关系预览、搜索结果、页面标题 |
| **配置级别** | 字段级别 (`field.meta.display`) | 集合级别 (`collection.meta.display_template`) |
| **入口组件** | `RenderDisplay` | `RenderTemplate` |
| **展示组件确定** | `display = meta.display \|\| getDefaultDisplayForType` | 每个模板变量独立确定 |
| **`'raw'` 处理** | 正常渲染组件 | 直接返回值，跳过组件 |
| **扩展不存在** | `VTextOverflow` | 直接返回原始值 |
| **渲染错误 fallback** | `VTextOverflow` | 模板插值 `{{ value }}` |
| **ValueNull** | 检查 `value === null` | 检查 `subPart === null` |

#### 7.7.2 端到端数据流：从 Stores 到渲染

**完整数据流（以 Tabular 路径为例）**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 1: 应用启动 - 数据加载
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

后端数据库
  ├─ directus_fields 表: 存储字段元数据 (display, display_options)
  ├─ directus_relations 表: 存储关系定义
  └─ directus_collections 表: 存储集合元数据 (display_template)
       ↓
API /fields 端点
       ↓
前端 FieldsStore.hydrate()
  ├─ GET /fields
  ├─ 解析响应 → Field[]
  └─ 存储到 Pinia: fields.value = [...fieldsRaw.map(parseField)]

API /relations 端点
       ↓
前端 RelationsStore.hydrate()
  ├─ GET /relations
  ├─ 解析响应 → Relation[]
  └─ 存储到 Pinia: relations.value = response.data.data

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 2: 展示组件 ID 确定
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tabular 布局初始化
  ├─ activeFields = 用户选择的字段列表
  │
  └─ 构建 tableHeaders
       ├─ 遍历 activeFields
       ├─ field = fieldsStore.getField(collection, fieldKey)
       ├─ display = field.meta?.display || getDefaultDisplayForType(field.type)
       └─ header.field = { display, displayOptions, type, ... }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 3: 渲染决策
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RenderDisplay 组件
  ├─ props: { display, value, type, collection, field }
  │
  ├─ P1: value === null || undefined?
  │   └─ 是 → <ValueNull /> → "--"
  │
  ├─ displayInfo = useExtension('display', display)
  │   └─ 在扩展注册表中查找:
  │      extensions.displays.value.find(({ id }) => id === display)
  │
  ├─ P2: displayInfo === null?
  │   └─ 是 → <VTextOverflow :text="value" /> → 原始值
  │
  └─ P3: 渲染展示组件
       ├─ <component :is="`display-${display}`" ... />
       │
       └─ 渲染异常?
           └─ VErrorBoundary fallback
               → <VTextOverflow :text="value" />

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 4: 最终输出
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

正常路径:
  display='formatted-value' → display-formatted-value 组件 → 格式化后的值

ValueNull 路径:
  value=null → <ValueNull /> → "--"

扩展不存在路径:
  display='unknown-display' → displayInfo=null → <VTextOverflow /> → 原始值

渲染错误路径:
  display='formatted-value' → 组件报错 → fallback → <VTextOverflow /> → 原始值
```

**完整数据流（以 RenderTemplate 路径为例）**：

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 1: 模板和数据获取
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

useTemplateData(collection, primaryKey)
  ├─ template = props.template || collection.meta.display_template || null
  │
  ├─ templateFields = getFieldsFromTemplate(template)
  │   → "{{title}} - {{author.name}}" → ['title', 'author.name']
  │
  ├─ fields = adjustFieldsForDisplays(templateFields, collection)
  │
  └─ 调用 API 获取数据
       GET /items/collection/123?fields=title,author.*
       ↓
       itemData = { title: "Hello", author: { name: "John" } }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 2: 模板解析
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RenderTemplate 组件
  ├─ props: { template, collection, item }
  │
  └─ parts = template.split(/({{.*?}})/g)
       → ["", "{{title}}", " - ", "{{author.name}}", ""]

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 3: 每个模板变量的决策
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

处理 "{{title}}":
  ├─ RT1: field = fieldsStore.getField(collection, 'title') → 存在
  ├─ RT2: component = meta.display || getDefaultDisplayForType('string') → 'formatted-value'
  ├─ RT3: component === 'raw'? → 否
  ├─ RT4: displayInfo = useExtension('display', 'formatted-value') → 存在
  └─ 构建配置 → 进入渲染阶段

处理 " - ":
  └─ 静态文本 → 直接显示

处理 "{{author.name}}":
  ├─ RT1: field = fieldsStore.getField(collection, 'author.name') → 可能不存在
  └─ RT1 触发 → 直接返回原始值 "John"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
阶段 4: 渲染
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

渲染 parts:
  ├─ "{{title}}" → display-formatted-value 组件 → "Hello"
  ├─ " - " → 静态文本 → " - "
  └─ "{{author.name}}" → 原始值 → "John"

最终输出: "Hello - John"
```

#### 7.7.3 关键组件职责

| 组件/Store | 职责 | 关键代码位置 |
|-----------|------|-------------|
| **FieldsStore** | 存储字段元数据，提供 `getField()` 查询 | `app/src/stores/fields.ts` |
| **RelationsStore** | 存储关系定义，提供 `getRelationsForField()` | `app/src/stores/relations.ts` |
| **useExtension** | 在扩展注册表中查找扩展配置 | `app/src/composables/use-extension.ts` |
| **RenderDisplay** | Tabular 路径的渲染决策入口 | `app/src/views/private/components/render-display.vue` |
| **RenderTemplate** | 模板解析 + 渲染决策 | `app/src/views/private/components/render-template.vue` |
| **ValueNull** | 显示 `--` 表示空值 | `app/src/views/private/components/value-null.vue` |
| **VTextOverflow** | 显示原始值，处理长文本溢出 | `app/src/components/v-text-overflow.vue` |
| **VErrorBoundary** | 捕获组件渲染错误，提供 fallback | `app/src/components/v-error-boundary.vue` |

---

### 7.8 边界场景分析：回退链路的潜在问题

#### 7.8.1 VErrorBoundary 工作原理

**VErrorBoundary 组件** (`app/src/components/v-error-boundary.vue:1-42`)：

```typescript
const error = ref<Error | null>(null);
const hasError = computed(() => !!error.value);

onErrorCaptured((err, vm, info) => {
    error.value = err;      // 捕获错误并标记 hasError = true
    console.warn(`[${source}-error] ${info}`);
    console.warn(err);
    if (props.stopPropagation) return false;
});
```

**模板渲染逻辑**：
```vue
<template>
    <!-- hasError = true → 渲染 fallback slot -->
    <template v-if="hasError">
        <template v-if="$slots.fallback">
            <slot name="fallback" v-bind="{ error }" />
        </template>
    </template>
    
    <!-- hasError = false → 正常渲染默认 slot -->
    <slot v-else></slot>
</template>
```

**VErrorBoundary 触发条件**：
- 子组件的 `setup()` 函数抛出异常
- 子组件的渲染函数抛出异常
- 子组件的生命周期钩子抛出异常
- 子组件的 watcher 回调抛出异常

**VErrorBoundary 不触发的场景**：
- 事件处理器中的错误（需要单独 try-catch）
- 异步组件的加载错误
- 服务端渲染错误
- 子组件自己捕获并处理的错误

#### 7.8.2 RenderTemplate 的 subPart 数据结构分析

**RenderTemplate 中的 parts 构建** (`app/src/views/private/components/render-template.vue:67-139`)：

`parts` 是一个二维数组，每个 `part` 对应模板中的一个片段（静态文本或模板变量）：

```
模板: "{{name}} - {{status}}"
      ├─ "{{name}}"     → 模板变量
      ├─ " - "          → 静态文本
      └─ "{{status}}"   → 模板变量

parts = [
    [ /* "{{name}}" 的处理结果 */ ],
    [ /* " - " 的处理结果 */ ],
    [ /* "{{status}}" 的处理结果 */ ]
]
```

**subPart 的 5 种可能类型**：

| subPart 类型 | 产生条件 | 结构示例 |
|-------------|---------|---------|
| **字符串** | 静态文本片段 | `" - "` |
| **原始值数组** | `!field` 或 `component='raw'` 或 `!displayInfo` | `["Hello"]` |
| **展示配置对象** | 数组友好组件（formatted-value 等） | `{ component: 'formatted-value', value: "Hello", ... }` |
| **展示配置对象数组** | 其他组件，逐值处理 | `[{ component: 'boolean', value: true, ... }]` |
| **null** | `.map((p) => p ?? null)` 的兜底 | `null` |

**完整数据结构示例**：

```typescript
// 模板: "{{name}} - {{active}}"
// item: { name: "Hello", active: false }
// name 字段: display='formatted-value', type='string'
// active 字段: display='boolean', type='boolean'

parts = [
    // "{{name}}" 的处理结果
    [
        {
            component: 'formatted-value',
            options: undefined,
            value: ["Hello"],           // 注意：value 是数组！
            interface: 'input',
            // ... 其他属性
        }
    ],
    
    // " - " 的处理结果
    [
        " - "                          // 静态文本，字符串类型
    ],
    
    // "{{active}}" 的处理结果
    [
        {
            component: 'boolean',
            options: undefined,
            value: false,              // 注意：单个值被展开
            interface: 'boolean',
            // ... 其他属性
        }
    ]
]
```

**关键细节：value 的包装与展开**：

```typescript
// 步骤 1: getNestedValues 总是返回数组
let value = getNestedValues(props.item, fieldKey);
// value = ["Hello"] 或 [false] 或 [null]

// 步骤 2: 数组友好组件 → 整个数组作为 value 传入
if (['related-values', 'formatted-value', ...].includes(component)) {
    return [{
        component,
        value,              // value 保持数组: ["Hello"]
        // ...
    }];
}

// 步骤 3: 其他组件 → 为数组中每个值创建独立配置
return value.map((v) => ({
    component,
    value: v,              // value 被展开: "Hello", false, null 等
    // ...
}));
```

#### 7.8.3 RenderTemplate VErrorBoundary fallback 的假值问题

**Fallback 代码** (`app/src/views/private/components/render-template.vue:164-166`)：

```vue
<template #fallback>
    <span>{{ subPart?.value || subPart }}</span>
</template>
```

**问题根源：`||` 运算符的假值语义**

JavaScript 的 `||` 运算符会将以下值视为"假值"：
- `false`
- `0`
- `""` (空字符串)
- `null`
- `undefined`
- `NaN`

**fallback 表达式分析**：

```javascript
subPart?.value || subPart

// 等价于：
subPart?.value ? subPart?.value : subPart

// 但由于 || 的假值语义：
subPart?.value === false    → 被视为假值 → 返回 subPart
subPart?.value === 0        → 被视为假值 → 返回 subPart
subPart?.value === ""       → 被视为假值 → 返回 subPart
```

#### 7.8.4 假值边界场景对照表

| 场景 | 原始 value | subPart 类型 | `subPart?.value` | `subPart?.value \|\| subPart` | 实际显示 | 预期显示 | 偏差？ |
|------|-----------|-------------|-----------------|------------------------------|---------|---------|--------|
| **字符串** | `"Hello"` | 对象 | `"Hello"` | `"Hello"` | `Hello` | `Hello` | ❌ 否 |
| **空字符串** | `""` | 对象 | `""` | `[object Object]` | `[object Object]` | `` | ✅ 是 |
| **数字** | `42` | 对象 | `42` | `42` | `42` | `42` | ❌ 否 |
| **数字 0** | `0` | 对象 | `0` | `[object Object]` | `[object Object]` | `0` | ✅ 是 |
| **布尔 true** | `true` | 对象 | `true` | `true` | `true` | `true` | ❌ 否 |
| **布尔 false** | `false` | 对象 | `false` | `[object Object]` | `[object Object]` | `false` | ✅ 是 |
| **null** | `null` | 对象 | `null` | `[object Object]` | `[object Object]` | `--` 或 `null` | ✅ 是 |
| **数组** | `[1, 2]` | 对象 | `[1, 2]` | `1,2` | `1,2` | `1,2` | 视情况 |
| **对象** | `{a: 1}` | 对象 | `{a: 1}` | `[object Object]` | `[object Object]` | `[object Object]` | 视情况 |

**详细场景分析**：

**场景 1：布尔字段 false**
```
字段: active (type=boolean, display=boolean)
值: false

正常渲染: display-boolean 组件 → 显示 "No" 或关闭图标
VErrorBoundary fallback 触发时:
  subPart?.value = false
  false || subPart → subPart (对象)
  {{ subPart }} → "[object Object]"
实际显示: "[object Object]"
预期显示: "false" 或 "--"
偏差: ✅ 存在
```

**场景 2：数字字段 0**
```
字段: count (type=integer, display=formatted-value)
值: 0

注意：formatted-value 是数组友好组件，value 保持数组
正常渲染: display-formatted-value 组件 → 显示 "0"
VErrorBoundary fallback 触发时:
  subPart?.value = [0]
  [0] || subPart → [0] (数组在布尔上下文中为真)
  {{ [0] }} → "0"
实际显示: "0"
预期显示: "0"
偏差: ❌ 不存在（幸运地）

但如果是非数组友好组件:
  subPart?.value = 0
  0 || subPart → subPart (对象)
  {{ subPart }} → "[object Object]"
实际显示: "[object Object]"
预期显示: "0"
偏差: ✅ 存在
```

**场景 3：空字符串**
```
字段: description (type=text, display=formatted-value)
值: ""

正常渲染: display-formatted-value 组件 → 显示 "" (空)
VErrorBoundary fallback 触发时:
  subPart?.value = "" (非数组友好组件) 或 [""] (数组友好)
  "" || subPart → subPart (对象)
  {{ subPart }} → "[object Object]"
实际显示: "[object Object]"
预期显示: "" 或 "--"
偏差: ✅ 存在
```

#### 7.8.5 根因分析

**问题 1：`||` 运算符的语义错误**

```javascript
// 当前代码
subPart?.value || subPart

// 问题：JavaScript 逻辑或运算符会将假值视为 false
// 应该使用空值合并运算符
subPart?.value ?? subPart

// 或者显式检查
subPart?.value !== undefined ? subPart?.value : subPart
```

**问题 2：对象的字符串化**

```vue
<!-- fallback 中的模板插值 -->
<span>{{ subPart?.value || subPart }}</span>

<!-- 当 subPart 是对象时 -->
{{ subPart }} → 调用 Object.prototype.toString() → "[object Object]"

<!-- 期望的行为 -->
{{ subPart?.value }} → 即使是 false/0/"" 也应该显示实际值
```

**问题 3：null 值的一致性**

```
RenderTemplate 中:
- ValueNull 检查: subPart === null || (typeof subPart === 'object' && subPart.value === null)
- 但 VErrorBoundary fallback 在 value === null 时也会触发
- 导致 null 值可能显示 "[object Object]" 而不是 "--"
```

#### 7.8.6 潜在修复方案

```vue
<!-- 方案 1：使用空值合并运算符 -->
<template #fallback>
    <span>{{ subPart?.value ?? subPart }}</span>
</template>

<!-- 方案 2：显式检查，同时处理 null -->
<template #fallback>
    <ValueNull v-if="subPart === null || (subPart?.value === null)" />
    <span v-else-if="subPart?.value !== undefined">{{ subPart?.value }}</span>
    <span v-else>{{ subPart }}</span>
</template>

<!-- 方案 3：参考 Tabular 的做法，使用 VTextOverflow -->
<template #fallback>
    <VTextOverflow :text="subPart?.value ?? subPart" />
</template>
```

---

### 7.9 Tabular 与 RenderTemplate 回退行为差异对照

#### 7.9.1 回退机制总览

```
Tabular 路径 (RenderDisplay):
    ├─ null 值 → ValueNull ("--")
    ├─ 扩展不存在 → VTextOverflow (原始值)
    └─ 渲染错误 → VErrorBoundary fallback → VTextOverflow (原始值)

RenderTemplate 路径:
    ├─ null 值 → ValueNull ("--")
    ├─ 扩展不存在 → 原始值数组（直接插值）
    ├─ component='raw' → 原始值数组（直接插值，跳过组件）
    └─ 渲染错误 → VErrorBoundary fallback → {{ subPart?.value || subPart }}
```

#### 7.9.2 详细差异对照

| 维度 | Tabular (RenderDisplay) | RenderTemplate |
|------|------------------------|---------------|
| **null 值处理** | 优先级最高，ValueNull 组件 | 优先级在渲染阶段检查 |
| **扩展不存在** | VTextOverflow 组件，统一处理 | 直接返回原始值数组，模板插值 |
| **渲染错误 fallback** | VTextOverflow 组件，`:text="value"` | 模板插值 `{{ subPart?.value \|\| subPart }}` |
| **component='raw'** | 正常渲染 display-raw 组件 | 特殊处理，直接返回值，跳过组件 |
| **假值 (false/0/\"\")** | VTextOverflow 可以正确显示 | `\|\|` 运算符导致显示偏差 |
| **对象值** | VTextOverflow 的 `:text` prop 接受对象 | 模板插值调用 `toString()` → "[object Object]" |
| **数组值** | VTextOverflow 显示 `Array.toString()` | 根据组件类型决定包装或展开 |

#### 7.9.3 假值回退行为对照表

| 值 | Tabular 正常 | Tabular fallback | RenderTemplate 正常 | RenderTemplate fallback |
|----|-------------|-----------------|--------------------|-----------------------|
| `"Hello"` | display-formatted-value → "Hello" | VTextOverflow → "Hello" | display-formatted-value → "Hello" | `"Hello"` |
| `""` | display-formatted-value → "" | VTextOverflow → "" | display-formatted-value → "" | `"[object Object]"` ⚠️ |
| `42` | display-formatted-value → "42" | VTextOverflow → "42" | display-formatted-value → "42" | `"42"` |
| `0` | display-formatted-value → "0" | VTextOverflow → "0" | display-formatted-value → "0" | `"[object Object]"` ⚠️ |
| `true` | display-boolean → "Yes" | VTextOverflow → "true" | display-boolean → "Yes" | `"true"` |
| `false` | display-boolean → "No" | VTextOverflow → "false" | display-boolean → "No" | `"[object Object]"` ⚠️ |
| `null` | ValueNull → "--" | ValueNull → "--" | ValueNull → "--" | `"[object Object]"` ⚠️ |

**⚠️ 表示存在显示偏差**

#### 7.9.4 回退行为的设计意图分析

**Tabular 路径的设计意图**：
- 表格场景需要一致性和可预测性
- VTextOverflow 提供统一的回退体验
- 支持长文本溢出处理（ellipsis + tooltip）
- 假值通过 `:text` prop 正确处理

```typescript
// VTextOverflow 接受任何类型的 text prop
interface Props {
    text?: string | number | boolean | Record<string, any> | Array<any>;
}

// 模板中直接插值
<template v-else>{{ text }}</template>
// Vue 会正确处理 false/0/""
```

**RenderTemplate 路径的设计意图**：
- 模板场景需要灵活性
- 支持静态文本与动态值的混合
- 数组友好组件的智能处理
- 但 fallback 实现存在缺陷

#### 7.9.5 建议的一致性改进

**RenderTemplate fallback 改进建议**：

```vue
<!-- 当前 (有问题) -->
<template #fallback>
    <span>{{ subPart?.value || subPart }}</span>
</template>

<!-- 改进方案 A：使用 VTextOverflow 保持一致性 -->
<template #fallback>
    <VTextOverflow :text="subPart?.value ?? subPart" />
</template>

<!-- 改进方案 B：更精细的控制 -->
<template #fallback>
    <ValueNull v-if="subPart === null || subPart?.value === null" />
    <VTextOverflow v-else :text="subPart?.value ?? subPart" />
</template>
```

**改进后的回退行为**：

| 值 | 改进前 | 改进后 |
|----|--------|--------|
| `""` | `"[object Object]"` | `""` |
| `0` | `"[object Object]"` | `"0"` |
| `false` | `"[object Object]"` | `"false"` |
| `null` | `"[object Object]"` | `"--"` |

---

## 8. 完整数据流总结

### 8.1 字段配置数据流

```
数据库 (directus_fields 表)
    ↓
后端 FieldsService.readAll()
    ├─ 读取 directus_fields 表（meta）
    ├─ 读取数据库 schema（schema）
    ├─ getLocalType() 确定 type
    └─ 组合成 Field 对象
    ↓
REST API /fields
    ↓
前端 FieldsStore.hydrate()
    ├─ 解析字段
    ├─ 应用翻译
    └─ 存储到 Pinia store
    ↓
字段使用（表单/列表）
    ├─ form-field-interface.vue → 接口组件
    └─ render-display.vue → 展示组件
```

### 8.2 扩展配置数据流

```
扩展文件系统 (extensions/ 目录)
    ↓
后端 ExtensionsManager
    ├─ 扫描扩展目录
    ├─ 构建扩展清单
    └─ 生成 extensions/sources/index.js
    ↓
前端 loadExtensions()
    ├─ DEV: @directus-extensions (Vite 虚拟模块)
    └─ PROD: /extensions/sources/index.js
    ↓
前端 registerExtensions()
    ├─ 合并内置和自定义扩展
    ├─ app.component() 注册 Vue 组件
    └─ 存储到响应式 extensions 对象
    ↓
使用时
    ├─ useExtension() 查找配置
    └─ 动态组件 <component :is="`interface-${id}`">
```

### 8.3 组件选择决策树

```
字段 Field 对象
    │
    ├─ meta.interface 已配置？
    │   ├─ 是 → 使用配置的接口 ID
    │   └─ 否 → getDefaultInterfaceForType(type)
    │
    ├─ meta.display 已配置？
    │   ├─ 是 → 使用配置的展示 ID
    │   └─ 否 → getDefaultDisplayForType(type)
    │
    └─ 验证组件是否存在？
        ├─ 接口：useExtension('interface', id) !== null
        └─ 展示：useExtension('display', id) !== null
```

## 9. 核心文件索引

| 文件路径 | 功能说明 |
|---------|---------|
| `packages/constants/src/fields.ts` | 字段类型常量定义（TYPES, LOCAL_TYPES, RELATIONAL_TYPES） |
| `packages/types/src/fields.ts` | 字段类型定义（Field, FieldMeta, FieldRaw） |
| `api/src/utils/get-local-type.ts` | 数据库类型到 Directus 类型的映射 |
| `api/src/services/fields.ts` | 字段服务（CRUD 操作） |
| `api/src/controllers/fields.ts` | 字段 REST API 控制器 |
| `app/src/stores/fields.ts` | 前端字段状态管理 |
| `app/src/utils/get-local-type.ts` | 前端本地类型判断 |
| `app/src/utils/get-default-interface-for-type.ts` | 默认接口映射 |
| `app/src/utils/get-default-display-for-type.ts` | 默认展示映射 |
| `app/src/extensions.ts` | 扩展加载与注册 |
| `app/src/stores/extensions.ts` | 扩展状态管理 |
| `app/src/composables/use-extension.ts` | 扩展查找 composable |
| `app/src/interfaces/index.ts` | 接口组件注册 |
| `app/src/displays/index.ts` | 展示组件注册 |
| `app/src/components/v-form/components/form-field-interface.vue` | 表单字段接口渲染 |
| `app/src/views/private/components/render-display.vue` | 展示组件渲染 |
| `app/src/modules/settings/routes/data-model/field-detail/store/index.ts` | 字段编辑时的接口/展示过滤 |

## 10. 关键类型定义

### Field 类型

```typescript
interface Field {
    collection: string;
    field: string;
    type: Type;           // Directus 类型：string, integer, boolean, json, m2o 等
    schema: Column | null;
    meta: FieldMeta | null;
    name: string;
}
```

### FieldMeta 类型

```typescript
interface FieldMeta {
    id: number;
    collection: string;
    field: string;
    interface: string | null;           // 接口组件 ID
    display: string | null;             // 展示组件 ID
    options: Record<string, any> | null;           // 接口配置
    display_options: Record<string, any> | null;   // 展示配置
    special: string[] | null;           // 特殊标记：m2o, o2m, file, translations 等
    hidden: boolean;
    readonly: boolean;
    required: boolean;
    sort: number | null;
    width: Width | null;                // half, full, fill
    note: string | null;
    translations: Translations[] | null;
    conditions: Condition[] | null;
    validation: Filter | null;
    searchable: boolean;
}
```

### InterfaceConfig 类型

```typescript
interface InterfaceConfig {
    id: string;
    name: string;
    description?: string;
    icon: string;
    component: Component;
    types: Type[];                      // 支持的 Directus 类型
    localTypes?: LocalType[];           // 支持的本地类型，默认 ['standard']
    group?: string;                     // standard, selection, relational, etc.
    options?: Field[] | (({ field, collections }) => Field[]) | null;
    recommendedDisplays?: string[];
    preview?: string;
    system?: boolean;
}
```

### DisplayConfig 类型

```typescript
interface DisplayConfig {
    id: string;
    name: string;
    description?: string;
    icon: string;
    component: Component;
    types: Type[];                      // 支持的 Directus 类型
    localTypes?: LocalType[];           // 支持的本地类型
    options?: Field[] | null;
}
```
