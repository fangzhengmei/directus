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
