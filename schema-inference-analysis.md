# Directus Schema 推断机制深度分析

## 1. 概述

Directus 是一个强大的实时 API 和仪表板系统，用于管理 SQL 数据库内容。其核心能力之一是能够从现有数据库中自动推断表结构、字段类型和关系，从而无需手动配置即可使用 Directus 管理数据库。

本报告深入分析 Directus 的 schema 推断机制，包括：
- 运行时如何从数据库中提取 schema 信息
- 如何适配不同的数据库方言
- 多数据库差异在哪一层被抹平
- 推断结果如何反映到业务层的元数据中

## 2. 架构层次

Directus 的 schema 推断机制采用了多层架构设计，从底层数据库到上层业务层，每一层都有明确的职责：

```
┌─────────────────────────────────────────────────────────────┐
│                      业务层 (Business Layer)                  │
│  - Field 类型 (包含 schema、meta、type)                       │
│  - Relation 类型 (包含 schema、meta)                          │
│  - Collections、Fields、Relations 服务                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   标准化层 (Normalization Layer)              │
│  - sanitize-schema.ts: 清理和标准化 schema 信息               │
│  - build-collection-and-field-relations.ts: 构建关系映射      │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Schema Inspector 层 (@directus/schema)       │
│  - SchemaInspector 接口: 统一的 schema 访问接口               │
│  - 方言实现: MySQL、PostgreSQL、SQLite、MSSQL、Oracle、CockroachDB │
│  - 类型定义: Column、Table、ForeignKey、SchemaOverview         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      数据库层 (Database Layer)                │
│  - Knex.js: 数据库连接和查询构建器                              │
│  - 各种 SQL 数据库: MySQL、PostgreSQL、SQLite、MSSQL、Oracle、CockroachDB │
└─────────────────────────────────────────────────────────────┘
```

## 3. 核心组件分析

### 3.1 @directus/schema 包

`@directus/schema` 包是 Directus schema 推断的核心，提供了统一的接口来访问不同数据库的 schema 信息。

#### 3.1.1 SchemaInspector 接口

`SchemaInspector` 接口定义了统一的 schema 访问方法：

```typescript
export interface SchemaInspector {
  knex: Knex;

  overview: () => Promise<SchemaOverview>;

  tables(): Promise<string[]>;
  tableInfo(): Promise<Table[]>;
  tableInfo(table: string): Promise<Table>;
  hasTable(table: string): Promise<boolean>;

  columns(table?: string): Promise<{ table: string; column: string }[]>;
  columnInfo(): Promise<Column[]>;
  columnInfo(table?: string): Promise<Column[]>;
  columnInfo(table: string, column: string): Promise<Column>;
  hasColumn(table: string, column: string): Promise<boolean>;
  primary(table: string): Promise<string | null>;

  foreignKeys(table?: string): Promise<ForeignKey[]>;

  // 可选方法，不是所有数据库都支持
  withSchema?(schema: string): void;
}
```
[packages/schema/src/types/schema-inspector.ts:7-32](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/types/schema-inspector.ts)

#### 3.1.2 核心类型定义

**Column 类型** - 表示数据库列的标准化信息：

```typescript
export interface Column {
  name: string;
  table: string;
  data_type: string;
  default_value: string | number | boolean | null;
  max_length: number | null;
  numeric_precision: number | null;
  numeric_scale: number | null;

  is_nullable: boolean;
  is_unique: boolean;
  is_indexed: boolean;
  is_primary_key: boolean;
  is_generated: boolean;
  generation_expression?: string | null;
  has_auto_increment: boolean;
  foreign_key_table: string | null;
  foreign_key_column: string | null;

  // 可选属性，不是所有数据库都支持
  comment?: string | null;
  
  // PostgreSQL 特有
  schema?: string;
  foreign_key_schema?: string | null;
}
```
[packages/schema/src/types/column.ts:1-26](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/types/column.ts)

**ForeignKey 类型** - 表示数据库外键关系：

```typescript
export type ForeignKey = {
  table: string;
  column: string;
  foreign_key_table: string;
  foreign_key_column: string;
  foreign_key_schema?: string;
  constraint_name: null | string;
  on_update: null | 'NO ACTION' | 'RESTRICT' | 'CASCADE' | 'SET NULL' | 'SET DEFAULT';
  on_delete: null | 'NO ACTION' | 'RESTRICT' | 'CASCADE' | 'SET NULL' | 'SET DEFAULT';
};
```
[packages/schema/src/types/foreign-key.ts:1-10](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/types/foreign-key.ts)

#### 3.1.3 SchemaInspector 工厂函数

`createInspector` 函数根据 Knex 客户端的类型动态创建对应的方言实现：

```typescript
export const createInspector = (knex: Knex): SchemaInspector => {
  let constructor: SchemaInspectorConstructor;

  switch (knex.client.constructor.name) {
    case 'Client_MySQL':
    case 'Client_MySQL2':
      constructor = MySQLSchemaInspector;
      break;
    case 'Client_PG':
      constructor = PostgresSchemaInspector;
      break;
    case 'Client_CockroachDB':
      constructor = CockroachDBSchemaInspector;
      break;
    case 'Client_SQLite3':
      constructor = SqliteSchemaInspector;
      break;
    case 'Client_Oracledb':
    case 'Client_Oracle':
      constructor = OracleDBSchemaInspector;
      break;
    case 'Client_MSSQL':
      constructor = MSSQLSchemaInspector;
      break;

    default:
      throw Error('Unsupported driver used: ' + knex.client.constructor.name);
  }

  return new constructor(knex);
};
```
[packages/schema/src/index.ts:16-46](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/index.ts)

### 3.2 方言实现

每种数据库都有自己的 `SchemaInspector` 实现，以处理不同数据库之间的差异。

#### 3.2.1 MySQL 方言实现

MySQL 方言使用 `information_schema` 系统表来获取 schema 信息：

**表信息查询**：
```typescript
async tables(): Promise<string[]> {
  const records = await this.knex
    .select<{ TABLE_NAME: string }[]>('TABLE_NAME')
    .from('INFORMATION_SCHEMA.TABLES')
    .where({
      TABLE_TYPE: 'BASE TABLE',
      TABLE_SCHEMA: this.knex.client.database(),
    });

  return records.map(({ TABLE_NAME }) => TABLE_NAME);
}
```
[packages/schema/src/dialects/mysql.ts:145-155](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/mysql.ts)

**列信息查询** - 使用复杂的 JOIN 来获取列的完整信息：

```typescript
async columnInfo(table?: string, column?: string) {
  const query = this.knex
    .select(
      'c.TABLE_NAME',
      'c.COLUMN_NAME',
      'c.COLUMN_DEFAULT',
      'c.COLUMN_TYPE',
      'c.CHARACTER_MAXIMUM_LENGTH',
      'c.IS_NULLABLE',
      'c.COLUMN_KEY',
      'c.EXTRA',
      'c.COLLATION_NAME',
      'c.COLUMN_COMMENT',
      'c.NUMERIC_PRECISION',
      'c.NUMERIC_SCALE',
      'c.GENERATION_EXPRESSION',
      'fk.REFERENCED_TABLE_NAME',
      'fk.REFERENCED_COLUMN_NAME',
      'fk.CONSTRAINT_NAME',
      'rc.UPDATE_RULE',
      'rc.DELETE_RULE',
      'rc.MATCH_OPTION',
      'stats.INDEX_NAME',
    )
    .from('INFORMATION_SCHEMA.COLUMNS as c')
    .leftJoin('INFORMATION_SCHEMA.KEY_COLUMN_USAGE as fk', function () {
      this.on('c.TABLE_NAME', '=', 'fk.TABLE_NAME')
        .andOn('fk.COLUMN_NAME', '=', 'c.COLUMN_NAME')
        .andOn('fk.CONSTRAINT_SCHEMA', '=', 'c.TABLE_SCHEMA');
    })
    .leftJoin('INFORMATION_SCHEMA.REFERENTIAL_CONSTRAINTS as rc', function () {
      this.on('rc.TABLE_NAME', '=', 'fk.TABLE_NAME')
        .andOn('rc.CONSTRAINT_NAME', '=', 'fk.CONSTRAINT_NAME')
        .andOn('rc.CONSTRAINT_SCHEMA', '=', 'fk.CONSTRAINT_SCHEMA');
    })
    .leftJoin('INFORMATION_SCHEMA.STATISTICS as stats', function () {
      this.on('stats.TABLE_SCHEMA', '=', 'c.TABLE_SCHEMA')
        .andOn('stats.TABLE_NAME', '=', 'c.TABLE_NAME')
        .andOn('stats.COLUMN_NAME', '=', 'c.COLUMN_NAME')
        .andOnVal('stats.NON_UNIQUE', 1)
        .andOnVal('stats.SEQ_IN_INDEX', 1);
    })
    .where({
      'c.TABLE_SCHEMA': this.knex.client.database(),
    });

  // ... 过滤和转换逻辑
}
```
[packages/schema/src/dialects/mysql.ts:243-311](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/mysql.ts)

**MySQL 特殊处理** - `tinyint(1)` 被视为 boolean 类型：

```typescript
export function rawColumnToColumn(rawColumn: RawColumn): Column {
  let dataType = rawColumn.COLUMN_TYPE.replace(/\(.*?\)/, '');

  if (rawColumn.COLUMN_TYPE.startsWith('tinyint(1)')) {
    dataType = 'boolean';
  }

  return {
    name: rawColumn.COLUMN_NAME,
    table: rawColumn.TABLE_NAME,
    data_type: dataType,
    default_value: parseDefaultValue(rawColumn.COLUMN_DEFAULT),
    generation_expression: rawColumn.GENERATION_EXPRESSION || null,
    max_length: rawColumn.CHARACTER_MAXIMUM_LENGTH,
    numeric_precision: rawColumn.NUMERIC_PRECISION,
    numeric_scale: rawColumn.NUMERIC_SCALE,
    is_generated: !!rawColumn.EXTRA?.endsWith('GENERATED'),
    is_nullable: rawColumn.IS_NULLABLE === 'YES',
    is_unique: rawColumn.COLUMN_KEY === 'UNI',
    is_indexed: !!rawColumn.INDEX_NAME && rawColumn.INDEX_NAME.length > 0,
    is_primary_key: rawColumn.CONSTRAINT_NAME === 'PRIMARY' || rawColumn.COLUMN_KEY === 'PRI',
    has_auto_increment: rawColumn.EXTRA === 'auto_increment',
    foreign_key_column: rawColumn.REFERENCED_COLUMN_NAME,
    foreign_key_table: rawColumn.REFERENCED_TABLE_NAME,
    comment: rawColumn.COLUMN_COMMENT,
  };
}
```
[packages/schema/src/dialects/mysql.ts:39-65](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/mysql.ts)

#### 3.2.2 PostgreSQL 方言实现

PostgreSQL 方言使用系统目录表（如 `pg_class`、`pg_attribute`、`pg_constraint` 等）来获取 schema 信息。

**PostgreSQL 特殊处理**：

1. **Schema 支持** - PostgreSQL 支持多个 schema，默认使用 `public`：

```typescript
constructor(knex: Knex) {
  this.knex = knex;
  const config = knex.client.config;

  if (!config.searchPath) {
    this.schema = 'public';
    this.explodedSchema = [this.schema];
  } else if (typeof config.searchPath === 'string') {
    this.schema = config.searchPath;
    this.explodedSchema = [config.searchPath];
  } else {
    this.schema = config.searchPath[0];
    this.explodedSchema = config.searchPath;
  }
}
```
[packages/schema/src/dialects/postgres.ts:55-69](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts)

2. **PostGIS 支持** - 检测并处理 PostGIS 几何类型：

```typescript
// 检查 PostGIS 是否存在
const hasPostGIS =
  (await this.knex.raw(`SELECT oid FROM pg_proc WHERE proname = 'postgis_version'`)).rows.length > 0;

if (hasPostGIS) {
  // 查询几何类型列
  const result = await this.knex.raw<{ rows: RawGeometryColumn[] }>(
    `WITH geometries as (
      select * from geometry_columns
      union
      select * from geography_columns
    )
    SELECT f_table_name as table_name
      , f_geometry_column as column_name
      , type as data_type
    FROM geometries g
    JOIN information_schema.tables t
      ON g.f_table_name = t.table_name
      AND t.table_type = 'BASE TABLE'
    WHERE f_table_schema in (${bindings})
    `,
    this.explodedSchema,
  );

  geometryColumns = result.rows;
}
```
[packages/schema/src/dialects/postgres.ts:159-182](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts)

3. **版本兼容** - 处理不同 PostgreSQL 版本的差异（如生成列支持）：

```typescript
const versionResponse = await this.knex.raw(`SHOW server_version`);
const majorVersion = versionResponse.rows?.[0]?.server_version?.split('.')?.[0] ?? 10;

let generationSelect = `
  NULL AS generation_expression,
  pg_get_expr(ad.adbin, ad.adrelid) AS default_value,
  FALSE AS is_generated,
`;

if (Number(majorVersion) >= 12) {
  generationSelect = `
  CASE WHEN att.attgenerated = 's' THEN pg_get_expr(ad.adbin, ad.adrelid) ELSE null END AS generation_expression,
  CASE WHEN att.attgenerated = '' THEN pg_get_expr(ad.adbin, ad.adrelid) ELSE null END AS default_value,
  att.attgenerated = 's' AS is_generated,
  `;
}
```
[packages/schema/src/dialects/postgres.ts:363-379](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts)

4. **默认值解析** - 处理 PostgreSQL 特有的默认值格式：

```typescript
export function parseDefaultValue(value: string | null): string | null {
  if (value === null) return null;
  if (value.startsWith('nextval(')) return value;  // 序列值保持原样

  value = value.split('::')[0] ?? null;  // 移除类型转换

  if (value?.trim().toLowerCase() === 'null') return null;

  return stripQuotes(value);
}
```
[packages/schema/src/dialects/postgres.ts:39-48](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts)

### 3.3 业务层类型定义

在 `@directus/types` 包中，定义了更高层次的业务层类型，这些类型将数据库 schema 信息与 Directus 的元数据结合起来。

#### 3.3.1 Field 类型

`Field` 类型表示 Directus 中的字段，包含三个关键部分：
- `schema`: 来自数据库的原始列信息（`Column` 类型）
- `meta`: Directus 的字段元数据（如界面、显示方式、验证规则等）
- `type`: Directus 的字段类型（与数据库类型映射）

```typescript
export interface FieldRaw {
  collection: string;
  field: string;
  type: Type;  // Directus 字段类型
  schema: Column | null;  // 数据库列信息
  meta: FieldMeta | null;  // Directus 元数据
}

export interface Field extends FieldRaw {
  name: string;
  children?: Field[] | null;  // 用于嵌套字段
}

export type FieldMeta = {
  id: number;
  collection: string;
  field: string;
  group: string | null;
  hidden: boolean;
  interface: string | null;
  display: string | null;
  options: Record<string, any> | null;
  display_options: Record<string, any> | null;
  readonly: boolean;
  required: boolean;
  sort: number | null;
  special: string[] | null;
  translations: Translations[] | null;
  width: Width | null;
  note: string | null;
  conditions: Condition[] | null;
  validation: Filter | null;
  validation_message: string | null;
  searchable: boolean;
  system?: true;
  clear_hidden_value_on_save?: boolean;
};
```
[packages/types/src/fields.ts:36-72](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/types/src/fields.ts)

#### 3.3.2 Relation 类型

`Relation` 类型表示 Directus 中的关系，同样包含 schema 和 meta 两部分：

```typescript
export type Relation = {
  collection: string;
  field: string;
  related_collection: string | null;
  schema: ForeignKey | null;  // 数据库外键信息
  meta: RelationMeta | null;  // Directus 关系元数据
};

export type RelationMeta = {
  id: number;

  many_collection: string;
  many_field: string;

  one_collection: string | null;
  one_field: string | null;
  one_collection_field: string | null;
  one_allowed_collections: string[] | null;
  one_deselect_action: 'nullify' | 'delete';

  junction_field: string | null;
  sort_field: string | null;

  system?: boolean;
};
```
[packages/types/src/relations.ts:21-19](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/types/src/relations.ts)

## 4. 运行时机制详解

### 4.1 SchemaInspector 的创建流程

当 Directus 需要访问数据库 schema 时，会按照以下流程创建 `SchemaInspector`：

1. **初始化 Knex 连接** - 首先创建 Knex 数据库连接
2. **调用 createInspector** - 传入 Knex 实例
3. **检测数据库类型** - 通过 `knex.client.constructor.name` 判断数据库类型
4. **创建对应方言实例** - 实例化对应的 `SchemaInspector` 实现
5. **返回统一接口** - 返回实现了 `SchemaInspector` 接口的实例

### 4.2 表结构推断过程

表结构的推断主要通过 `SchemaInspector` 的以下方法：

1. **`tables()`** - 获取所有表名
   - MySQL: 查询 `INFORMATION_SCHEMA.TABLES`
   - PostgreSQL: 查询 `pg_class` 系统表
   - 过滤掉视图，只返回基础表（`BASE TABLE`）

2. **`tableInfo()`** - 获取表的详细信息
   - 返回表名、schema、注释等信息
   - 不同数据库有不同的特有属性：
     - MySQL: `collation`、`engine`
     - PostgreSQL: `owner`
     - SQLite: `sql`
     - MSSQL: `catalog`

### 4.3 字段类型推断过程

字段类型推断是 schema 推断中最复杂的部分，涉及：

1. **原始列信息获取**
   - 查询数据库系统表获取列的原始信息
   - 包括：列名、数据类型、默认值、是否可空、字符最大长度、数值精度等

2. **类型标准化**
   - 将数据库特定的类型转换为统一的表示
   - 例如 MySQL 的 `tinyint(1)` → `boolean`
   - 例如 PostgreSQL 的 `character varying` → `varchar`

3. **约束检测**
   - 检测主键、外键、唯一约束、索引等
   - 主键检测：
     - MySQL: `COLUMN_KEY = 'PRI'`
     - PostgreSQL: `pg_constraint.contype = 'p'`
   - 外键检测：
     - MySQL: 通过 `KEY_COLUMN_USAGE` 和 `REFERENTIAL_CONSTRAINTS`
     - PostgreSQL: 通过 `pg_constraint` 表

4. **特殊属性检测**
   - 自增列：
     - MySQL: `EXTRA = 'auto_increment'`
     - PostgreSQL: 检测序列依赖（`pg_get_serial_sequence`）
   - 生成列：
     - MySQL: `EXTRA` 以 `GENERATED` 结尾
     - PostgreSQL: 12+ 版本支持，通过 `attgenerated` 字段检测

### 4.4 关系推断过程

关系推断主要通过 `foreignKeys()` 方法：

1. **外键信息查询**
   - 获取所有外键约束
   - 包括：约束名、更新规则、删除规则等

2. **关系类型识别**
   - 基于外键的位置和元数据（如果有）识别关系类型：
     - 多对一（M2O）：外键在当前表
     - 一对多（O2M）：外键在关联表，需要反向查找
     - 多对多（M2M）：需要中间表
     - 多对任意（M2A）：通过 `one_collection_field` 动态指定

3. **关系元数据补充**
   - 如果存在 Directus 的 `directus_relations` 表，会从该表获取关系元数据
   - 元数据包括：关系名称、显示方式、排序字段等

## 5. 多数据库差异分析

### 5.1 主要差异点

不同数据库之间存在以下主要差异：

| 差异类别 | MySQL | PostgreSQL | SQLite | MSSQL | Oracle |
|---------|-------|------------|--------|-------|--------|
| **系统表** | `information_schema.*` | `pg_class`, `pg_attribute`, `pg_constraint` 等 | `sqlite_master`, `pragma` | `sys.*`, `INFORMATION_SCHEMA` | `ALL_TABLES`, `ALL_TAB_COLUMNS` 等 |
| **Schema 支持** | 数据库级 | 多 Schema | 不支持 | 多 Schema | 多 Schema |
| **自增实现** | `AUTO_INCREMENT` | `SERIAL` / `IDENTITY` | `AUTOINCREMENT` | `IDENTITY` | `SEQUENCE` + `TRIGGER` |
| **布尔类型** | `tinyint(1)` | `boolean` | `INTEGER` | `bit` | `NUMBER(1)` |
| **注释支持** | 支持 | 支持 | 不支持 | 支持 | 支持 |
| **外键规则** | 标准 | 标准（字符码表示） | 部分支持 | 标准 | 标准 |

### 5.2 差异抹平的层次

Directus 在多个层次上抹平了不同数据库之间的差异：

#### 5.2.1 SchemaInspector 层（第一层）

这是最底层的差异抹平，通过方言实现将不同数据库的查询结果转换为统一的 `Column`、`Table`、`ForeignKey` 类型。

**示例：默认值解析**

MySQL 默认值解析：
```typescript
export function parseDefaultValue(value: string | null): string | null {
  if (value === null || value.trim().toLowerCase() === 'null') return null;
  return stripQuotes(value);
}
```
[packages/schema/src/dialects/mysql.ts:67-71](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/mysql.ts)

PostgreSQL 默认值解析：
```typescript
export function parseDefaultValue(value: string | null): string | null {
  if (value === null) return null;
  if (value.startsWith('nextval(')) return value;
  value = value.split('::')[0] ?? null;
  if (value?.trim().toLowerCase() === 'null') return null;
  return stripQuotes(value);
}
```
[packages/schema/src/dialects/postgres.ts:39-48](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts)

#### 5.2.2 标准化层（第二层）

在 `sanitize-schema.ts` 中，通过 `pick` 函数只保留与数据库无关的关键属性，进一步抹平差异：

```typescript
export function sanitizeColumn(column: Column) {
  return pick(column, [
    'name',
    'table',
    'data_type',
    'default_value',
    'max_length',
    'numeric_precision',
    'numeric_scale',
    'is_nullable',
    'is_unique',
    'is_indexed',
    'is_primary_key',
    'is_generated',
    'generation_expression',
    'has_auto_increment',
    'foreign_key_table',
    'foreign_key_column',
    // 注意：这里没有包含数据库特有的属性如 schema、comment 等
  ]);
}
```
[api/src/utils/sanitize-schema.ts:59-78](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/sanitize-schema.ts)

#### 5.2.3 业务层（第三层）

在业务层，通过 `Field` 和 `Relation` 类型，将数据库 schema 信息与 Directus 元数据结合，完全屏蔽了底层数据库的差异：

```typescript
export interface Field {
  collection: string;
  field: string;
  type: Type;  // Directus 统一的字段类型
  schema: Column | null;  // 标准化的数据库列信息
  meta: FieldMeta | null;  // Directus 元数据
  name: string;
  children?: Field[] | null;
}
```

### 5.3 数据库特有的处理

#### 5.3.1 PostgreSQL 特有处理

1. **多 Schema 支持** - 通过 `withSchema()` 方法切换 schema
2. **PostGIS 集成** - 自动检测并处理几何类型
3. **序列检测** - 通过 `pg_get_serial_sequence` 检测自增列
4. **版本兼容** - 处理 12+ 版本的生成列支持

#### 5.3.2 MySQL 特有处理

1. **`tinyint(1)` 布尔处理** - 将 `tinyint(1)` 视为布尔类型
2. **存储引擎和字符集** - 保留 `engine` 和 `collation` 信息
3. **`AUTO_INCREMENT` 检测** - 通过 `EXTRA` 字段检测自增列

#### 5.3.3 SQLite 特有处理

1. **外键支持** - SQLite 对外部键的支持有限，需要特殊处理
2. **自增实现** - SQLite 使用 `AUTOINCREMENT` 关键字
3. **注释不支持** - SQLite 不支持列和表的注释

## 6. 业务层映射机制

### 6.1 从 Column 到 Field 的映射

`Field` 类型是 `Column` 类型的超集，它包含：

1. **直接映射的属性**：
   - `field` ↔ `Column.name`
   - `schema` ↔ 完整的 `Column` 对象
   - `collection` ↔ `Column.table`

2. **需要推断或配置的属性**：
   - `type`: Directus 字段类型，基于 `Column.data_type` 推断
   - `meta`: Directus 元数据，来自 `directus_fields` 表或默认值

### 6.2 从 ForeignKey 到 Relation 的映射

`Relation` 类型同样是 `ForeignKey` 的超集：

1. **直接映射的属性**：
   - `collection` ↔ `ForeignKey.table`
   - `field` ↔ `ForeignKey.column`
   - `related_collection` ↔ `ForeignKey.foreign_key_table`
   - `schema` ↔ 完整的 `ForeignKey` 对象

2. **需要元数据补充的属性**：
   - `meta`: 来自 `directus_relations` 表，包含关系的详细配置

### 6.3 关系构建机制

`buildCollectionAndFieldRelations` 函数用于构建集合和字段之间的关系映射：

```typescript
export function buildCollectionAndFieldRelations(relations: Relation[]) {
  // Map<Collection, Set<RelatedCollection>>
  const collectionRelationTree = new Map<string, Set<string>>();
  // Map<Field:Collection, RelatedCollection>
  const fieldToCollectionList = new Map<string, string>();

  for (const relation of relations) {
    let relatedCollections = [];

    if (relation.related_collection) {
      relatedCollections.push(relation.related_collection);
    } else if (relation.meta?.one_collection_field && relation.meta?.one_allowed_collections) {
      // 处理多对任意（M2A）关系
      relatedCollections = relation.meta?.one_allowed_collections;
    } else {
      continue;
    }

    for (const relatedCollection of relatedCollections) {
      // 构建字段到集合的映射
      let fieldToCollectionListKey = relation.collection + ':' + relation.field;
      const collectionList = collectionRelationTree.get(relatedCollection) ?? new Set<string>();

      collectionList.add(relation.collection);

      // 处理一对多（O2M）关系的反向映射
      if (relation.meta?.one_field) {
        const relatedfieldToCollectionListKey = relatedCollection + ':' + relation.meta.one_field;
        const realatedCollectionList = collectionRelationTree.get(relation.collection) ?? new Set<string>();

        realatedCollectionList.add(relatedCollection);

        fieldToCollectionList.set(relatedfieldToCollectionListKey, relation.collection);
        collectionRelationTree.set(relation.collection, realatedCollectionList);
      }

      // 处理多对任意（M2A）关系的特殊键格式
      if (relation.meta?.one_allowed_collections) {
        fieldToCollectionListKey += ':' + relatedCollection;
      }

      fieldToCollectionList.set(fieldToCollectionListKey, relatedCollection);
      collectionRelationTree.set(relatedCollection, collectionList);
    }
  }

  return { collectionRelationTree, fieldToCollectionList };
}
```
[api/src/services/fields/build-collection-and-field-relations.ts:19-65](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/fields/build-collection-and-field-relations.ts)

这个函数构建了两个重要的映射：
1. `collectionRelationTree`: 集合到相关集合的映射
2. `fieldToCollectionList`: 字段（格式为 `collection:field`）到相关集合的映射

这些映射用于：
- 检测循环依赖
- 处理嵌套查询
- 计算关系深度
- 验证查询中的关系引用

## 7. 核心数据流

### 7.1 完整的 Schema 推断流程

当 Directus 启动或需要刷新 schema 时，会执行以下流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                        1. 初始化数据库连接                         │
│  - 创建 Knex 实例                                                 │
│  - 验证连接可用性                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        2. 创建 SchemaInspector                    │
│  - 调用 createInspector(knex)                                     │
│  - 根据数据库类型选择对应的方言实现                                  │
│  - 返回 SchemaInspector 接口实例                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        3. 提取数据库 Schema                        │
│  - inspector.tables() → 获取所有表名                               │
│  - inspector.tableInfo() → 获取表详细信息                           │
│  - inspector.columnInfo() → 获取列详细信息                          │
│  - inspector.foreignKeys() → 获取外键关系                          │
│  - 结果: Column[]、Table[]、ForeignKey[]                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        4. 查询 Directus 元数据                     │
│  - 从 directus_collections 表查询集合元数据                          │
│  - 从 directus_fields 表查询字段元数据                               │
│  - 从 directus_relations 表查询关系元数据                            │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        5. 合并 Schema 和元数据                      │
│  - 将 Column 与 FieldMeta 合并为 Field                              │
│  - 将 ForeignKey 与 RelationMeta 合并为 Relation                    │
│  - 为没有元数据的字段/关系生成默认值                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        6. 标准化和验证                              │
│  - 调用 sanitizeField、sanitizeRelation 等函数                      │
│  - 移除数据库特有的属性                                             │
│  - 验证关系的完整性                                                │
│  - 构建关系映射（collectionRelationTree、fieldToCollectionList）    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        7. 缓存和使用                                │
│  - 将最终的 schema 缓存到内存中                                     │
│  - 供 API 层、权限检查、查询构建等使用                                │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 运行时 Schema 使用

一旦 schema 被推断和缓存，它会在多个地方被使用：

1. **API 层**：
   - 验证请求中的字段和关系
   - 构建正确的 SQL 查询
   - 处理权限检查
   - 格式化响应数据

2. **权限系统**：
   - 基于字段和集合的权限检查
   - 关系权限的递归验证

3. **查询构建**：
   - 自动处理 JOIN 和关系
   - 正确处理不同数据库的 SQL 语法
   - 应用字段级别的转换和验证

4. **App 仪表板**：
   - 动态生成界面
   - 显示字段的元数据（标签、帮助文本等）
   - 构建关系选择器

## 8. 关键设计模式和架构决策

### 8.1 适配器模式

`SchemaInspector` 接口及其各种方言实现是**适配器模式**的典型应用：
- **目标接口**：`SchemaInspector` 定义了统一的访问接口
- **适配器**：每个数据库的方言实现（`MySQLSchemaInspector`、`PostgresSchemaInspector` 等）
- **适配者**：不同数据库的系统表和 SQL 语法

这种模式使得上层代码可以完全不关心底层数据库的差异，只需要与统一的接口交互。

### 8.2 工厂模式

`createInspector` 函数是**工厂模式**的应用：
- 根据输入（Knex 实例）动态决定创建哪个具体实现
- 封装了对象创建的逻辑
- 提供了统一的创建入口

### 8.3 分层架构

Directus 采用了清晰的**分层架构**：
1. **数据库层**：Knex 和原始数据库
2. **Schema 层**：`@directus/schema` 包，提供统一的 schema 访问
3. **标准化层**：`sanitize-*` 函数，进一步标准化数据
4. **业务层**：`Field`、`Relation` 类型，结合元数据
5. **应用层**：API、App 等使用最终的 schema

每一层都只依赖于下一层，而不依赖于更底层的实现细节，这使得替换或修改某一层变得更加容易。

### 8.4 元数据驱动设计

Directus 的一个核心设计理念是**元数据驱动**：
- 数据库 schema 是基础元数据
- Directus 的元数据（`directus_*` 表）扩展了基础元数据
- 所有的界面、API 行为都由元数据决定
- 用户可以通过修改元数据来改变系统行为，而无需编写代码

## 9. 总结

Directus 的 schema 推断机制是一个设计精良的多层架构系统，它：

1. **通过方言适配层抹平数据库差异**：
   - 每种数据库都有对应的 `SchemaInspector` 实现
   - 将不同数据库的系统表查询、SQL 语法差异封装在方言层
   - 输出统一的 `Column`、`Table`、`ForeignKey` 类型

2. **在多层架构中逐步标准化**：
   - 第一层：`SchemaInspector` 方言层，输出统一类型
   - 第二层：`sanitize-*` 函数，移除数据库特有属性
   - 第三层：`Field`、`Relation` 业务类型，结合元数据

3. **采用元数据驱动设计**：
   - 数据库 schema 是基础元数据
   - Directus 元数据表扩展了 schema 信息
   - 系统行为由元数据决定，而非硬编码

4. **使用经典设计模式**：
   - 适配器模式：统一不同数据库的接口
   - 工厂模式：动态创建合适的方言实现
   - 分层架构：清晰的层次结构和依赖关系

这种设计使得 Directus 能够：
- 支持多种 SQL 数据库（MySQL、PostgreSQL、SQLite、MSSQL、Oracle、CockroachDB）
- 从现有数据库自动推断 schema，实现即插即用
- 提供统一的 API 和界面，无论底层使用哪种数据库
- 允许用户通过元数据自定义系统行为
- 易于扩展以支持新的数据库类型

## 10. 参考代码位置

- **SchemaInspector 接口定义**：[packages/schema/src/types/schema-inspector.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/types/schema-inspector.ts)
- **Column 类型定义**：[packages/schema/src/types/column.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/types/column.ts)
- **ForeignKey 类型定义**：[packages/schema/src/types/foreign-key.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/types/foreign-key.ts)
- **createInspector 工厂函数**：[packages/schema/src/index.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/index.ts)
- **MySQL 方言实现**：[packages/schema/src/dialects/mysql.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/mysql.ts)
- **PostgreSQL 方言实现**：[packages/schema/src/dialects/postgres.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/schema/src/dialects/postgres.ts)
- **Field 类型定义**：[packages/types/src/fields.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/types/src/fields.ts)
- **Relation 类型定义**：[packages/types/src/relations.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/packages/types/src/relations.ts)
- **sanitize-schema 工具函数**：[api/src/utils/sanitize-schema.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/utils/sanitize-schema.ts)
- **关系构建函数**：[api/src/services/fields/build-collection-and-field-relations.ts](g:/fangzheng/solo-dogfeeding/code/17734-directus/api/src/services/fields/build-collection-and-field-relations.ts)
