# Directus Schema Introspection 机制分析

## 概述

Directus 通过一套完整的 Schema Introspection 机制来读取不同数据库的结构信息。这套机制采用了**抽象接口 + 方言实现**的设计模式，在保持统一接口的同时，处理不同数据库之间的差异。

## 核心架构

### 1. 架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                        API Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐ │
│  │ SchemaService│  │ getSnapshot()│  │ CollectionsService│ │
│  └──────────────┘  └──────────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  @directus/schema Package                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              createInspector() 工厂函数                 │  │
│  │  (根据 Knex 客户端类型选择对应的 Inspector 实现)        │  │
│  └───────────────────────────────────────────────────────┘  │
│                              │                                │
│                              ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              SchemaInspector 接口 (统一契约)            │  │
│  │  - overview()       - tables()      - tableInfo()     │  │
│  │  - columns()        - columnInfo()  - foreignKeys()   │  │
│  │  - hasTable()       - hasColumn()   - primary()       │  │
│  └───────────────────────────────────────────────────────┘  │
│                              │                                │
│          ┌───────────────────┼───────────────────┐          │
│          ▼                   ▼                   ▼          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │PostgresSchema│    │MySQLSchema   │    │SqliteSchema  │ │
│  │Inspector     │    │Inspector     │    │Inspector     │ │
│  ├──────────────┤    ├──────────────┤    ├──────────────┤ │
│  │MSSQLSchema   │    │CockroachDBSch│    │OracleDBSchema│ │
│  │Inspector     │    │emaInspector  │    │Inspector     │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 2. 关键文件位置

| 组件 | 文件路径 | 说明 |
|------|---------|------|
| 核心接口 | `packages/schema/src/types/schema-inspector.ts` | SchemaInspector 接口定义 |
| 类型定义 | `packages/schema/src/types/*.ts` | Column, Table, ForeignKey 等类型 |
| 工厂函数 | `packages/schema/src/index.ts` | createInspector() |
| Postgres 实现 | `packages/schema/src/dialects/postgres.ts` | PostgreSQL 方言 |
| MySQL 实现 | `packages/schema/src/dialects/mysql.ts` | MySQL 方言 |
| SQLite 实现 | `packages/schema/src/dialects/sqlite.ts` | SQLite 方言 |
| MSSQL 实现 | `packages/schema/src/dialects/mssql.ts` | SQL Server 方言 |
| API 服务层 | `api/src/services/schema.ts` | SchemaService |
| 快照工具 | `api/src/utils/get-snapshot.ts` | getSnapshot() |

## SchemaInspector 接口定义

### 接口契约

**文件**: `packages/schema/src/types/schema-inspector.ts:7-36`

```typescript
export interface SchemaInspector {
    knex: Knex;

    // 获取完整的 schema 概览（最常用）
    overview: () => Promise<SchemaOverview>;

    // 表操作
    tables(): Promise<string[]>;
    tableInfo(): Promise<Table[]>;
    tableInfo(table: string): Promise<Table>;
    hasTable(table: string): Promise<boolean>;

    // 列操作
    columns(table?: string): Promise<{ table: string; column: string }[]>;
    columnInfo(): Promise<Column[]>;
    columnInfo(table?: string): Promise<Column[]>;
    columnInfo(table: string, column: string): Promise<Column>;
    hasColumn(table: string, column: string): Promise<boolean>;

    // 约束操作
    primary(table: string): Promise<string | null>;
    foreignKeys(table?: string): Promise<ForeignKey[]>;

    // 可选方法（部分数据库支持）
    withSchema?(schema: string): void;
}
```

### 统一类型定义

**Column 接口** (`packages/schema/src/types/column.ts`):

```typescript
export interface Column {
    name: string;
    table: string;
    data_type: string;
    default_value: string | number | boolean | null;
    max_length: number | null;
    numeric_precision: number | null;
    numeric_scale: number | null;

    // 布尔属性
    is_nullable: boolean;
    is_unique: boolean;
    is_indexed: boolean;
    is_primary_key: boolean;
    is_generated: boolean;
    has_auto_increment: boolean;

    // 外键信息
    foreign_key_table: string | null;
    foreign_key_column: string | null;

    // 数据库特定的可选属性
    comment?: string | null;           // SQLite/MSSQL 不支持
    schema?: string;                    // Postgres 独有
    foreign_key_schema?: string | null; // Postgres 独有
    generation_expression?: string | null;
}
```

**Table 接口** (`packages/schema/src/types/table.ts`):

```typescript
export interface Table {
    name: string;

    // 可选属性
    comment?: string | null;    // SQLite 不完整支持
    schema?: string;

    // MySQL 独有
    collation?: string;
    engine?: string;

    // Postgres 独有
    owner?: string;

    // SQLite 独有
    sql?: string;

    // MSSQL 独有
    catalog?: string;
}
```

## 工厂函数：createInspector

**文件**: `packages/schema/src/index.ts:16-46`

这是方言选择的核心逻辑，根据 Knex 客户端的构造函数名称来选择对应的 SchemaInspector 实现：

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

## 各数据库方言实现差异

### 1. PostgreSQL

**核心特点**:
- 使用 PostgreSQL 系统目录表 (`pg_class`, `pg_attribute`, `pg_constraint` 等)
- 支持 `withSchema()` 方法切换 schema
- 特殊处理 PostGIS 几何类型

**关键实现示例** (`packages/schema/src/dialects/postgres.ts:86-230`):

```typescript
async overview(): Promise<SchemaOverview> {
    // 使用 information_schema + 系统表组合查询
    const [columnsResult, primaryKeysResult] = await Promise.all([
        this.knex.raw(
            `
            SELECT c.table_name, c.column_name, ...
            FROM information_schema.columns c
            LEFT JOIN information_schema.tables t
              ON c.table_name = t.table_name
            WHERE t.table_type = 'BASE TABLE'
              AND c.table_schema IN (${bindings});
            `,
            this.explodedSchema,
        ),
        this.knex.raw(
            `
            SELECT relname as table_name, pg_attribute.attname as column_name
            FROM pg_index, pg_class, pg_attribute, pg_namespace
            WHERE indrelid = pg_class.oid
              AND nspname IN (${bindings})
              ...
              AND indisprimary
            `,
            this.explodedSchema,
        ),
    ]);

    // 特殊处理: 检查 PostGIS 扩展
    const hasPostGIS = (await this.knex.raw(
        `SELECT oid FROM pg_proc WHERE proname = 'postgis_version'`
    )).rows.length > 0;

    if (hasPostGIS) {
        // 读取 geometry_columns 和 geography_columns
    }
    // ...
}
```

**默认值解析** (`packages/schema/src/dialects/postgres.ts:39-48`):

```typescript
export function parseDefaultValue(value: string | null): string | null {
    if (value === null) return null;
    if (value.startsWith('nextval(')) return value; // 保留序列引用

    value = value.split('::')[0] ?? null; // 移除类型转换: 'text'::character varying => 'text'

    if (value?.trim().toLowerCase() === 'null') return null;
    return stripQuotes(value);
}
```

### 2. MySQL

**核心特点**:
- 使用 `INFORMATION_SCHEMA` 系统视图
- 特殊处理 `tinyint(1)` 为 boolean 类型
- 使用 `SHOW KEYS` 等 MySQL 特有语句

**关键实现示例** (`packages/schema/src/dialects/mysql.ts:83-137`):

```typescript
async overview(): Promise<SchemaOverview> {
    const columns = await this.knex.raw(
        `
        SELECT
            C.TABLE_NAME as table_name,
            C.COLUMN_NAME as column_name,
            C.COLUMN_DEFAULT as default_value,
            C.IS_NULLABLE as is_nullable,
            C.COLUMN_TYPE as data_type,
            C.COLUMN_KEY as column_key,
            C.CHARACTER_MAXIMUM_LENGTH as max_length,
            C.EXTRA as extra
        FROM INFORMATION_SCHEMA.COLUMNS AS C
        LEFT JOIN INFORMATION_SCHEMA.TABLES AS T 
            ON C.TABLE_NAME = T.TABLE_NAME AND C.TABLE_SCHEMA = T.TABLE_SCHEMA
        WHERE
            T.TABLE_TYPE = 'BASE TABLE' AND
            C.TABLE_SCHEMA = ?;
        `,
        [this.knex.client.database()],
    );

    // 特殊处理: tinyint(1) => boolean
    let dataType = column.data_type.replace(/\(.*?\)/, '');
    if (column.data_type.startsWith('tinyint(1)')) {
        dataType = 'boolean';
    }
}
```

**主键查询** (`packages/schema/src/dialects/mysql.ts:333-341`):

```typescript
async primary(table: string): Promise<string | null> {
    const results = await this.knex.raw(
        `SHOW KEYS FROM ?? WHERE Key_name = 'PRIMARY'`, 
        table
    );
    // MySQL 特有语法
}
```

### 3. SQLite

**核心特点**:
- 使用 SQLite 的 `PRAGMA` 命令而非标准 SQL
- 不支持 `INFORMATION_SCHEMA`
- 类型系统非常灵活（动态类型）

**关键实现示例** (`packages/schema/src/dialects/sqlite.ts:38-77`):

```typescript
async overview(): Promise<SchemaOverview> {
    // 1. 查找包含 AUTOINCREMENT 的表
    const tablesWithAutoIncrementPrimaryKeys = (
        await this.knex.select('name')
            .from('sqlite_master')
            .whereRaw(`sql LIKE "%AUTOINCREMENT%"`)
    ).map(({ name }) => name);

    const tables = await this.tables();

    for (const table of tables) {
        // 2. 使用 PRAGMA 获取表结构
        const columns = await this.knex.raw<RawColumn[]>(
            `PRAGMA table_xinfo(??)`, 
            table
        );
        // ...
    }
}
```

**列信息查询** (`packages/schema/src/dialects/sqlite.ts:159-240`):

```typescript
async columnInfo(table?: string, column?: string) {
    const getColumnsForTable = async (table: string): Promise<Column[]> => {
        // 使用多个 PRAGMA 命令组合信息
        const columns: RawColumn[] = await this.knex.raw(
            `PRAGMA table_xinfo(??)`, 
            table
        );
        
        const foreignKeys = await this.knex.raw(
            `PRAGMA foreign_key_list(??)`, 
            table
        );
        
        const indexList = await this.knex.raw(
            `PRAGMA index_list(??)`, 
            table
        );
        
        // 逐个获取索引的详细信息
        const indexInfoList = await Promise.all(
            indexList.map((index) =>
                this.knex.raw(`PRAGMA index_info(??)`, index.name),
            ),
        );
        // ...
    };
}
```

**类型提取工具** (`packages/schema/src/utils/extract-type.ts`):

```typescript
// SQLite 的类型是动态的，需要从 CREATE TABLE 语句中解析
export default function extractType(type: string): string {
    if (!type) return 'unknown';
    
    // 提取类型名称，忽略括号内的长度/精度
    const match = type.match(/^([a-zA-Z]+)/);
    return match ? match[1].toLowerCase() : 'unknown';
}
```

### 4. Microsoft SQL Server (MSSQL)

**核心特点**:
- 混合使用 `INFORMATION_SCHEMA` 和 `sys.*` 系统视图
- 支持 `withSchema()` 方法（默认 schema 为 `dbo`）
- 特殊处理 `nvarchar` 等 Unicode 类型的长度计算

**关键实现示例** (`packages/schema/src/dialects/mssql.ts:287-402`):

```typescript
async columnInfo(table?: string, column?: string) {
    const dbName = this.knex.client.database();
    const schemaIdResult = await this.knex.select('schema_id')
        .from('sys.schemas')
        .where({ name: this.schema })
        .first();

    // 使用临时表存储索引信息
    const dbResult = await this.knex.transaction(async (trx) => {
        await trx.raw(`IF OBJECT_ID('tempdb..##IndexInfo') IS NOT NULL DROP TABLE ##IndexInfo;`);

        // 将索引信息插入临时表
        await trx.raw(`
            SELECT [ic].[object_id], [ic].[column_id], ...
            INTO ##IndexInfo
            FROM [sys].[index_columns] ic
            JOIN [sys].[indexes] ix ON [ix].[object_id] = [ic].[object_id];
        `);

        // 复杂的查询组合
        const query = trx
            .with('FilteredIndexInfo', this.knex.raw(`...`))
            .select(trx.raw(`
                [o].[name] AS [table],
                [c].[name] AS [name],
                [t].[name] AS [data_type],
                ...
                OBJECT_NAME ([fk].[referenced_object_id]) AS [foreign_key_table],
                COL_NAME ([fk].[referenced_object_id], [fk].[referenced_column_id]) AS [foreign_key_column]
            `))
            .from(trx.raw(`??.[sys].[columns] [c]`, [dbName]))
            .joinRaw(`JOIN [sys].[types] [t] ON [c].[user_type_id] = [t].[user_type_id]`)
            // ... 更多 JOIN
            .where({ 's.schema_id': schemaId });
        // ...
    });
}
```

**Unicode 类型长度处理** (`packages/schema/src/dialects/mssql.ts:54-72`):

```typescript
function parseMaxLength(rawColumn: RawColumn) {
    const max_length = Number(rawColumn.max_length);
    // ...
    
    // n-* 类型每个字符占 2 字节
    // varchar(100)  => max_length = 100
    // nvarchar(100) => max_length = 200 (需要除以 2)
    if (['nvarchar', 'nchar', 'ntext'].includes(rawColumn.data_type)) {
        return max_length === -1 ? null : max_length / 2;
    }
    return max_length === -1 ? null : max_length;
}
```

## 各数据库差异汇总表

| 特性 | PostgreSQL | MySQL | SQLite | MSSQL |
|------|-----------|-------|--------|-------|
| **元数据来源** | 系统表 (`pg_*`) + `information_schema` | `INFORMATION_SCHEMA` | `PRAGMA` 命令 | `sys.*` + `INFORMATION_SCHEMA` |
| **Schema 支持** | ✅ 支持 (默认 `public`) | ✅ (数据库即 schema) | ❌ 不支持 | ✅ (默认 `dbo`) |
| **withSchema() 方法** | ✅ | ❌ | ❌ | ✅ |
| **几何类型** | ✅ PostGIS 扩展 | ❌ | ❌ | ✅ (内置) |
| **注释支持** | ✅ | ✅ | ❌ | 部分 |
| **自增列检测** | `nextval()` 序列 | `EXTRA = 'auto_increment'` | `AUTOINCREMENT` 关键字 | `is_identity` 属性 |
| **外键信息** | `pg_constraint` | `REFERENTIAL_CONSTRAINTS` | `PRAGMA foreign_key_list` | `sys.foreign_keys` |
| **默认值格式** | `'value'::type` | 原始值 | 原始值 | `((value))` 嵌套括号 |

## API 层集成

### SchemaService

**文件**: `api/src/services/schema.ts`

SchemaService 是 Directus API 暴露给外部的 schema 操作服务：

```typescript
export class SchemaService {
    knex: Knex;
    accountability: Accountability | null;

    // 获取当前数据库快照
    async snapshot(): Promise<Snapshot> {
        if (this.accountability?.admin !== true) throw new ForbiddenError();
        const currentSnapshot = await getSnapshot({ database: this.knex });
        return currentSnapshot;
    }

    // 应用 schema 差异
    async apply(payload: SnapshotDiffWithHash, options?: { force?: boolean }): Promise<void> {
        // 验证 + 应用
        await applyDiff(currentSnapshot, payload.diff, { database: this.knex });
    }

    // 比较两个 schema
    async diff(snapshot: Snapshot, options?: ...): Promise<SnapshotDiff | null> {
        const diff = getSnapshotDiff(currentSnapshot, snapshot);
        return diff;
    }
}
```

### getSnapshot 流程

**文件**: `api/src/utils/get-snapshot.ts`

```
getSnapshot()
    │
    ├──> getSchema()           # 使用 SchemaInspector 获取数据库原生结构
    │       │
    │       └──> createInspector(knex)
    │            └──> inspector.overview()
    │
    ├──> CollectionsService    # 读取 Directus 集合元数据
    ├──> FieldsService         # 读取 Directus 字段元数据
    └──> RelationsService      # 读取 Directus 关系元数据
```

### 调用链示例

```typescript
// 1. 创建 Inspector (工厂模式)
import { createInspector } from '@directus/schema';
const inspector = createInspector(knex);

// 2. 获取 schema 概览
const overview = await inspector.overview();
// 返回 SchemaOverview 类型:
// {
//   "users": {
//     "primary": "id",
//     "columns": {
//       "id": { data_type: "integer", default_value: "AUTO_INCREMENT", ... },
//       "email": { data_type: "varchar", max_length: 255, ... }
//     }
//   }
// }

// 3. 获取详细列信息
const columns = await inspector.columnInfo('users');

// 4. 获取外键信息
const foreignKeys = await inspector.foreignKeys('articles');
```

## 设计模式总结

### 1. 策略模式 (Strategy Pattern)
- **SchemaInspector** 接口定义了策略契约
- 每个数据库方言是一个具体策略实现
- **createInspector()** 工厂函数根据上下文选择策略

### 2. 适配器模式 (Adapter Pattern)
- 每个方言实现将数据库特有的 API 适配到统一的 SchemaInspector 接口
- 例如:
  - PostgreSQL 的 `pg_class` 查询 → `tables()` 方法
  - SQLite 的 `PRAGMA table_xinfo()` → `columnInfo()` 方法

### 3. 空对象/可选属性模式
- **Column** 和 **Table** 接口使用可选属性处理数据库差异
- 例如:
  - `schema?: string` - 只有 Postgres/MSSQL 有
  - `engine?: string` - 只有 MySQL 有
  - `comment?: string` - SQLite/MSSQL 不完整支持

### 4. 工厂模式
- **createInspector()** 根据 Knex 客户端类型动态创建对应的 Inspector
- 解耦了调用者和具体实现

## 关键差异处理技巧

### 1. 默认值解析
不同数据库返回的默认值格式差异很大:

| 数据库 | 默认值格式示例 | 解析策略 |
|--------|--------------|---------|
| PostgreSQL | `'example'::character varying` | 移除 `::type` 后缀，解引号 |
| PostgreSQL | `nextval('seq'::regclass)` | 保留序列引用 |
| MySQL | `'example'` | 直接解引号 |
| MSSQL | `((N'example'))` | 移除多层括号 |
| SQLite | `'example'` | 直接解引号 |

### 2. 自增列检测
| 数据库 | 检测方式 |
|--------|---------|
| PostgreSQL | `default_value` 以 `nextval(` 开头，或 `is_identity` |
| MySQL | `EXTRA = 'auto_increment'` |
| SQLite | 表的 `CREATE TABLE` SQL 包含 `AUTOINCREMENT` 关键字 |
| MSSQL | `COLUMNPROPERTY(..., 'IsIdentity') = 1` |

### 3. 布尔类型处理
| 数据库 | 存储方式 | 转换策略 |
|--------|---------|---------|
| PostgreSQL | `boolean` 原生类型 | 直接识别 |
| MySQL | `tinyint(1)` 约定 | 检测类型定义，转换为 `boolean` |
| SQLite | 无原生布尔类型 | 使用 `INTEGER` 存储，0/1 |
| MSSQL | `bit` 类型 | 直接识别 |

### 4. Unicode 长度计算
**MSSQL 特有问题**:
- `varchar(100)` → `max_length = 100` (单字节)
- `nvarchar(100)` → `max_length = 200` (双字节，需要除以 2)
- `nvarchar(MAX)` → `max_length = -1` (表示无限制)

## 扩展新数据库方言的指南

如果需要添加新的数据库支持，需要:

### 1. 创建方言类
```typescript
// packages/schema/src/dialects/newdb.ts
import type { Knex } from 'knex';
import type { SchemaInspector } from '../types/schema-inspector.js';

export default class NewDBSchemaInspector implements SchemaInspector {
    knex: Knex;

    constructor(knex: Knex) {
        this.knex = knex;
    }

    // 实现所有必需方法
    async overview(): Promise<SchemaOverview> { /* ... */ }
    async tables(): Promise<string[]> { /* ... */ }
    async tableInfo(): Promise<Table[]> { /* ... */ }
    async hasTable(table: string): Promise<boolean> { /* ... */ }
    async columns(table?: string): Promise<TableColumn[]> { /* ... */ }
    async columnInfo(table?: string, column?: string): Promise<any> { /* ... */ }
    async hasColumn(table: string, column: string): Promise<boolean> { /* ... */ }
    async primary(table: string): Promise<string | null> { /* ... */ }
    async foreignKeys(table?: string): Promise<ForeignKey[]> { /* ... */ }
}
```

### 2. 注册到工厂函数
```typescript
// packages/schema/src/index.ts
import NewDBSchemaInspector from './dialects/newdb.js';

export const createInspector = (knex: Knex): SchemaInspector => {
    switch (knex.client.constructor.name) {
        // ... 现有 case
        case 'Client_NewDB':  // Knex 客户端的构造函数名
            constructor = NewDBSchemaInspector;
            break;
        // ...
    }
};
```

### 3. 关键实现要点
1. **统一类型映射**: 将数据库特有类型映射到 Directus 通用类型
2. **默认值解析**: 处理数据库特有的默认值格式
3. **自增检测**: 正确识别自增列
4. **外键读取**: 正确解析外键关系
5. **性能优化**: 尽可能使用批量查询，避免 N+1 问题

## 性能考量

### 1. PostgreSQL 的批量查询
PostgreSQL 实现使用 `Promise.all()` 并行执行多个查询:

```typescript
const [columnsResult, primaryKeysResult] = await Promise.all([
    this.knex.raw(/* 列信息查询 */),
    this.knex.raw(/* 主键查询 */),
]);
```

### 2. SQLite 的 N+1 问题
SQLite 由于缺乏 JOIN 支持，可能存在 N+1 问题:

```typescript
// 获取所有表的列信息需要逐个表查询
const columnsPerTable = await Promise.all(
    tables.map(async (table) => await this.columns(table))
);
```

### 3. MSSQL 的临时表策略
MSSQL 使用临时表优化复杂的索引信息查询:

```typescript
// 先将索引信息存入临时表
await trx.raw(`SELECT ... INTO ##IndexInfo FROM [sys].[index_columns] ...`);

// 然后在主查询中引用
const query = trx.with('FilteredIndexInfo', this.knex.raw(`
    SELECT ... FROM ##IndexInfo WHERE ...
`));
```

## 总结

Directus 的 Schema Introspection 机制通过以下设计优雅地处理了多数据库差异:

1. **统一接口层**: `SchemaInspector` 定义了标准契约
2. **方言实现层**: 每个数据库有独立的实现类处理特有语法
3. **工厂选择层**: `createInspector()` 根据 Knex 客户端动态选择实现
4. **可选属性模式**: 类型定义使用可选属性处理数据库特性差异

这种设计使得:
- **调用者**无需关心底层数据库类型
- **扩展新数据库**只需实现 `SchemaInspector` 接口
- **维护性**各数据库的差异逻辑隔离在各自的实现类中
