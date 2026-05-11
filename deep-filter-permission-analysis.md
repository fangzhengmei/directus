# Directus 深层过滤条件在关联查询中的权限传播机制分析

## 1. 概述

Directus 的权限系统采用了基于 **AST（抽象语法树）** 的权限注入机制，结合 **Case/When** 结构在 SQL 级别实现细粒度的行级权限控制。本文档详细分析深层过滤条件（Deep Filtering）在关联查询中的权限限制传播机制。

## 2. 核心架构

### 2.1 整体流程

```
用户查询请求
    ↓
getAstFromQuery() - 解析查询为 AST
    ↓
processAst() - 权限处理核心
    ├─ fieldMapFromAst() - 提取访问路径
    ├─ fetchPolicies() - 获取策略
    ├─ fetchPermissions() - 获取权限规则
    ├─ validatePathExistence() - 验证字段存在
    ├─ validatePathPermissions() - 验证权限
    └─ injectCases() - 注入权限条件 ← 关键步骤
    ↓
getDBQuery() - 生成 SQL
    └─ applyCaseWhen() - 应用 CASE/WHEN 权限检查
```

### 2.2 关键文件位置

| 功能模块 | 文件路径 | 行号 |
|---------|---------|------|
| AST 权限处理入口 | `api/src/permissions/modules/process-ast/process-ast.ts` | 19-67 |
| 权限注入逻辑 | `api/src/permissions/modules/process-ast/lib/inject-cases.ts` | 13-73 |
| 获取权限规则 | `api/src/permissions/modules/process-ast/lib/get-cases.ts` | 6-57 |
| 深层过滤遍历 | `packages/utils/shared/deep-map-filter.ts` | 8-193 |
| SQL 过滤应用 | `api/src/database/run-ast/lib/apply-query/filter/index.ts` | 17-227 |
| CASE/WHEN 应用 | `api/src/database/run-ast/utils/apply-case-when.ts` | 21-59 |
| 权限路径验证 | `api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts` | 4-43 |
| 深层查询处理 | `api/src/database/get-ast-from-query/utils/get-deep-query.ts` | 16-21 |

## 3. 权限传播的核心机制

### 3.1 AST 结构定义

Directus 使用精心设计的 AST 来承载查询结构和权限信息，定义在 `api/src/types/ast.ts`:

```typescript
// 根节点
type AST = {
    type: 'root';
    name: string;                    // 集合名称
    children: [...];                 // 子节点（字段/关联）
    query: Query;                    // 查询参数
    cases: Filter[];                 // 权限规则数组 ← 权限传播载体
};

// 多对一 (M2O) 关联节点
type M2ONode = {
    type: 'm2o';
    name: string;
    children: [...];
    query: Query;
    relation: Relation;
    whenCase: number[];              // 索引数组，指向 cases 中的规则
    cases: Filter[];                 // 子集合的权限规则
};

// 一对多 (O2M) 关联节点
type O2MNode = {
    type: 'o2m';
    name: string;
    children: [...];
    query: Query;
    relation: Relation;
    whenCase: number[];
    cases: Filter[];
};

// 字段节点
type FieldNode = {
    type: 'field';
    name: string;
    fieldKey: string;
    whenCase: number[];              // 哪些规则满足时可访问此字段
};
```

**关键设计点：**
- `cases` 存储所有权限规则（Filter 对象数组）
- `whenCase` 存储需要满足的规则索引
- 每个关联节点有自己独立的 `cases` 数组

### 3.2 权限注入流程

#### 阶段 1: 提取访问路径 (fieldMapFromAst)

`fieldMapFromAst()` 遍历 AST，提取所有访问路径：

```
FieldMap 结构:
{
    read: Map<path, { collection, fields }>   // 只读访问路径
    other: Map<path, { collection, fields }>  // 其他操作访问路径
}
```

例如查询 `products` 包含关联 `category`:
```
路径: ''         → { collection: 'products', fields: ['id', 'name'] }
路径: 'category' → { collection: 'categories', fields: ['id', 'name'] }
```

#### 阶段 2: 权限验证

`validatePathPermissions()` 检查每个路径的权限：

```typescript
// 伪代码逻辑
for (const [path, { collection, fields }] of fieldMap) {
    const permissionsForCollection = permissions.filter(
        p => p.collection === collection
    );
    
    if (permissionsForCollection.length === 0) {
        throw ForbiddenError();
    }
    
    // 检查字段权限
    const allowedFields = extractAllowedFields(permissionsForCollection);
    const forbiddenFields = fields.filter(f => !allowedFields.has(f));
    
    if (forbiddenFields.length > 0) {
        throw FieldsForbiddenError();
    }
}
```

#### 阶段 3: 权限规则注入 (injectCases)

这是权限传播的核心，定义在 `inject-cases.ts:13-73`：

```typescript
function injectCases(ast: AST, permissions: Permission[]) {
    ast.cases = processChildren(ast.name, ast.children, permissions);
}

function processChildren(
    collection: string,
    children: (NestedCollectionNode | FieldNode | FunctionFieldNode)[],
    permissions: Permission[],
) {
    // 1. 获取当前集合的权限规则
    const { cases, caseMap, allowedFields } = getCases(
        collection, 
        permissions, 
        requestedKeys
    );
    
    // 2. 遍历每个子节点
    for (const child of children) {
        const fieldKey = getUnaliasedFieldKey(child);
        
        // 3. 确定该字段需要哪些规则
        const globalWhenCase = caseMap['*'];      // 全局规则
        const fieldWhenCase = caseMap[fieldKey];  // 字段特定规则
        
        // 4. 设置 whenCase（需要满足的规则索引）
        if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
            child.whenCase = [
                ...(globalWhenCase ?? []), 
                ...(fieldWhenCase ?? [])
            ];
        }
        
        // 5. 递归处理关联节点 ← 权限传播关键
        if (child.type === 'm2o') {
            child.cases = processChildren(
                child.relation.related_collection!,
                child.children,
                permissions  // 同一套权限规则，内部根据 collection 过滤
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
    }
    
    return cases;
}
```

### 3.3 getCases 逻辑详解

`get-cases.ts:6-57` 负责从权限配置中提取规则：

```typescript
function getCases(collection: string, permissions: Permission[], requestedKeys: string[]) {
    // 1. 过滤当前集合的权限
    const permissionsForCollection = permissions.filter(
        p => p.collection === collection
    );
    
    // 2. 去重规则
    const rules = dedupeAccess(permissionsForCollection);
    
    const cases: Filter[] = [];           // 规则数组
    const caseMap: Record<FieldKey, number[]> = {};  // 字段 → 规则索引映射
    let index = 0;
    
    for (const { rule, fields } of rules) {
        // 3. 跳过不相关的规则（规则字段与请求字段无交集）
        if (requestedKeys.length > 0 &&
            !fields.has('*') &&
            Array.from(fields).every(f => !requestedKeys.includes(f))
        ) {
            continue;
        }
        
        if (rule === null) continue;
        
        // 4. 添加规则到 cases 数组
        cases.push(rule);
        
        // 5. 记录每个字段对应哪些规则
        for (const field of fields) {
            caseMap[field] = [...(caseMap[field] ?? []), index];
        }
        
        index++;
    }
    
    // 6. 识别无条件允许的字段（rule 为 {} 或 null）
    const allowedFields = new Set(
        permissionsForCollection
            .filter(p => hasItemPermissions(p) === false)
            .map(p => p.fields ?? [])
            .flat()
    );
    
    return { cases, caseMap, allowedFields };
}
```

## 4. 深层过滤 (Deep Filtering) 机制

### 4.1 deepMapFilter 工作原理

`packages/utils/shared/deep-map-filter.ts:8-193` 提供了递归遍历过滤器的能力：

```typescript
function deepMapFilter(
    filter: Filter,
    callback: (entry, context) => [key, value] | undefined,
    context: { schema, collection, path? }
) {
    const collection = context.schema.collections[context.collection]!;
    const path = context.path ?? [];
    
    const result = Object.fromEntries(
        Object.entries(filter).map(([key, value]) => {
            // 1. 处理逻辑运算符
            if (key === '_or' || key === '_and') {
                value = value.map(subFilter => 
                    deepMapFilter(subFilter, callback, context)
                );
                return callback([key, value], { ...context, leaf: false });
            }
            
            // 2. 检测是否为关联字段
            const relationInfo = getRelationInfo(
                schema.relations, 
                collection.collection, 
                key
            );
            
            if (!relationInfo) return [key, value];
            
            // 3. 根据关联类型递归处理
            if (relationInfo.relation && !isPrimitive(value)) {
                switch (relationInfo.relationType) {
                    case 'm2o':
                        value = deepMapFilter(value, callback, {
                            schema,
                            collection: relationInfo.relation.related_collection!,
                            path: [...path, key]
                        });
                        leaf = false;
                        break;
                        
                    case 'o2m':
                        // 处理 _some/_none 量词
                        const quantityInfo = extractQuantity(value);
                        // 递归处理
                        value = deepMapFilter(quantityInfo.object, callback, {
                            schema,
                            collection: relationInfo.relation.collection!,
                            path: [...path, key]
                        });
                        leaf = false;
                        break;
                        
                    case 'a2o':
                        // 需要指定目标集合 key:targetCollection
                        value = deepMapFilter(value, callback, {
                            schema,
                            collection: targetCollection,
                            path: [...path, `${key}:${targetCollection}`]
                        });
                        leaf = false;
                        break;
                }
            }
            
            return callback([key, value], {
                collection,
                field,
                ...relationInfo,
                leaf,
                path
            });
        })
    );
    
    return result;
}
```

### 4.2 _some 和 _none 量词

对于 O2M/O2A 关系，支持特殊量词：

```javascript
// 筛选有至少一个已发布评论的文章
{
    "comments": {
        "_some": {
            "status": { "_eq": "published" }
        }
    }
}

// 筛选没有任何已删除评论的文章
{
    "comments": {
        "_none": {
            "status": { "_eq": "deleted" }
        }
    }
}
```

**实现位置：** `deep-map-filter.ts:102-132` 和 `filter/index.ts:134-173`

```typescript
// filter/index.ts 中的处理
if (childKey === '_none' || childKey === '_some') {
    const subQueryBuilder = (filter, cases) => (subQueryKnex) => {
        subQueryKnex
            .select({ [field]: column })
            .from(collection)
            .whereNotNull(column);
        
        // 递归应用查询（包含权限）
        applyQuery(knex, relation!.collection, subQueryKnex, 
            { filter }, schema, cases, permissions
        );
    };
    
    // 获取子查询的权限规则
    const { cases: subCases } = getCases(
        relation!.collection, permissions, []
    );
    
    if (childKey === '_none') {
        dbQuery[logical].whereNotIn(pkField, subQueryBuilder(filter, subCases));
    } else if (childKey === '_some') {
        dbQuery[logical].whereIn(pkField, subQueryBuilder(filter, subCases));
    }
}
```

**关键点：** 子查询会通过 `applyQuery` 递归应用权限规则，确保深层过滤也受权限限制。

## 5. SQL 生成阶段的权限应用

### 5.1 applyCaseWhen 机制

`apply-case-when.ts:21-59` 将权限规则转换为 SQL CASE/WHEN：

```typescript
function applyCaseWhen(options, context) {
    const { column, columnCases, table, cases, permissions } = options;
    const { knex, schema } = context;
    
    // 构建 WHERE 条件来检测权限
    const caseQuery = knex.queryBuilder();
    
    applyFilter(knex, schema, caseQuery, 
        { _or: columnCases },  // 将索引对应的规则组合
        table, aliasMap, cases, permissions
    );
    
    // 生成 SQL
    const sql = sqlParts.join(' ');
    
    // CASE WHEN <权限条件> THEN <实际列值> END
    let rawCase = `(CASE WHEN ${sql} THEN ?? END)`;
    
    return knex.raw(rawCase, bindings);
}
```

### 5.2 实际 SQL 示例

假设有权限规则：
```json
{
    "collection": "products",
    "action": "read",
    "fields": ["*"],
    "permissions": { "status": { "_eq": "published" } }
}
```

**生成的 SQL 结构：**

```sql
SELECT
    id,
    CASE WHEN `products`.`status` = 'published' THEN `products`.`name` END AS `name`,
    CASE WHEN `products`.`status` = 'published' THEN `products`.`description` END AS `description`
FROM `products`
WHERE `products`.`status` = 'published'
```

**关联查询示例（M2O）：**

```sql
SELECT
    `products`.`id`,
    CASE WHEN `products`.`status` = 'published' 
         THEN `products`.`name` 
    END AS `name`,
    -- 关联字段的权限检查
    CASE WHEN `products`.`status` = 'published'
         THEN `category`.`id`
    END AS `category__id`,
    CASE WHEN `products`.`status` = 'published' 
         AND `category`.`is_active` = 1  -- 关联表权限
         THEN `category`.`name`
    END AS `category__name`
FROM `products`
LEFT JOIN `categories` AS `category` 
    ON `products`.`category_id` = `category`.`id`
WHERE `products`.`status` = 'published'
```

## 6. 权限传播路径详解

### 6.1 M2O 关联的权限传播

**场景：** 查询产品及其关联分类

```javascript
// 查询
{
    "collection": "products",
    "fields": ["id", "name", "category.id", "category.name"],
    "filter": { "status": { "_eq": "active" } }
}

// 权限配置
[
    {
        "collection": "products",
        "action": "read",
        "fields": ["*"],
        "permissions": { "status": { "_eq": "published" } }
    },
    {
        "collection": "categories",
        "action": "read",
        "fields": ["id", "name"],
        "permissions": { "is_active": { "_eq": true } }
    }
]
```

**AST 处理过程：**

```
AST 初始结构:
root (products)
├── field: id
├── field: name
└── m2o: category
    ├── field: id
    └── field: name

injectCases 处理后:
root (products)
├── cases: [{ status: { _eq: "published" } }]
├── field: id (whenCase: [0])
├── field: name (whenCase: [0])
└── m2o: category
    ├── whenCase: [0]           // 父级权限索引
    ├── cases: [{ is_active: { _eq: true } }]  // 子级权限规则
    ├── field: id (whenCase: [0])
    └── field: name (whenCase: [0])
```

**权限传播链：**
1. `products` 级别：`cases = [rule_p]`
2. `category` 关联：`whenCase = [0]`（需要满足 products 的规则 0）
3. `category` 自身：`cases = [rule_c]`
4. `category.name`：`whenCase = [0]`（需要满足 category 的规则 0）

### 6.2 O2M 关联的权限传播

**场景：** 查询文章及其评论

```javascript
// 查询
{
    "collection": "articles",
    "fields": ["id", "title", "comments.id", "comments.content"]
}

// 权限配置
[
    {
        "collection": "articles",
        "action": "read",
        "fields": ["*"],
        "permissions": { "status": { "_eq": "published" } }
    },
    {
        "collection": "comments",
        "action": "read",
        "fields": ["*"],
        "permissions": { "is_approved": { "_eq": true } }
    }
]
```

**权限传播特点：**
- O2M 关联在 `get-db-query.ts` 中会生成独立的子查询
- 每个子查询都会应用自己的 `cases` 权限规则
- `whenCase` 控制父级是否有权访问该关联

### 6.3 A2O (Any-to-One) 关联的权限传播

A2O 关联可以指向多个集合，权限传播需要为每个目标集合单独处理：

```typescript
// inject-cases.ts:60-64
if (child.type === 'a2o') {
    for (const collection of child.names) {
        child.cases[collection] = processChildren(
            collection, 
            child.children[collection] ?? [],
            permissions
        );
    }
}
```

**AST 结构：**
```
a2o: related
├── names: ['documents', 'images']
├── cases: {
│   'documents': [{ status: { _eq: 'public' } }],
│   'images': [{ visibility: { _eq': 'public' } }]
│ }
├── children: {
│   'documents': [...],
│   'images': [...]
│ }
└── whenCase: [0]  // 父级权限
```

## 7. 深层过滤中的权限传播

### 7.1 关联字段过滤

```javascript
// 筛选特定分类的产品
{
    "filter": {
        "category": {
            "name": { "_eq": "Electronics" }
        }
    }
}
```

**权限传播：**
1. 主查询 `products` 应用权限规则
2. JOIN `categories` 时，需要有 `categories` 的读取权限
3. 过滤条件会同时受到：
   - 用户对 `products` 的权限（通过 WHERE 子句）
   - 用户对 `categories` 的权限（通过 JOIN 条件）

### 7.2 _some/_none 量词的权限

```javascript
// 筛选有已批准评论的文章
{
    "filter": {
        "comments": {
            "_some": {
                "content": { "_contains": "important" },
                "is_approved": { "_eq": true }
            }
        }
    }
}
```

**执行流程（filter/index.ts:145-172）：**

```
1. 识别 _some 量词
2. 构建子查询：
   SELECT comment.article_id 
   FROM comments
   WHERE content LIKE '%important%'
     AND is_approved = true
     -- 权限规则自动注入
     AND (comments.status = 'published')
3. 主查询 WHERE id IN (子查询结果)
```

**关键点：** 子查询会通过 `applyQuery` 递归调用，自动注入子集合的权限规则。

## 8. 边界情况与特殊处理

### 8.1 完全权限（无限制）

当权限规则为 `{}` 或 `null` 时：

```typescript
// get-cases.ts:44-50
const allowedFields = new Set(
    permissionsForCollection
        .filter(p => hasItemPermissions(p) === false)  // rule 为空
        .map(p => p.fields ?? [])
        .flat()
);

// inject-cases.ts:47-50
if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
    child.whenCase = [...];  // 只有不完全权限时才设置
}
// 完全权限的字段 whenCase = []，不添加 CASE/WHEN
```

**SQL 优化：** 完全权限的字段直接选择，不包裹 CASE/WHEN。

### 8.2 多规则合并

```typescript
// 用户有多个权限规则
const permissions = [
    { 
        collection: 'products', 
        permissions: { status: { _eq: 'published' } },
        fields: ['name', 'description']
    },
    { 
        collection: 'products',
        permissions: { category: { _eq: 'sale' } },
        fields: ['price', 'discount']
    }
];

// getCases 处理后
cases = [
    { status: { _eq: 'published' } },
    { category: { _eq: 'sale' } }
];
caseMap = {
    'name': [0],
    'description': [0],
    'price': [1],
    'discount': [1]
};
```

**字段访问条件：**
- `name`: 满足规则 0 即可
- `price`: 满足规则 1 即可
- 任一规则满足即可访问对应字段

### 8.3 管理员和公开访问

```typescript
// process-ast.ts:25-32
if (!options.accountability || options.accountability.admin) {
    // 只验证字段存在性，不注入权限
    for (const [path, { collection, fields }] of fieldMap) {
        validatePathExistence(path, collection, fields, context.schema);
    }
    return options.ast;  // 直接返回，不调用 injectCases
}
```

**管理员特权：**
- `accountability.admin = true` 时跳过权限注入
- `accountability = null`（公开访问）也只做存在性验证

## 9. 性能优化考虑

### 9.1 规则跳过优化

```typescript
// get-cases.ts:24-30
if (requestedKeys.length > 0 &&
    fields.has('*') === false &&
    Array.from(fields).every(field => !requestedKeys.includes(field))
) {
    continue;  // 跳过与查询无关的规则
}
```

### 9.2 完全权限优化

```typescript
// inject-cases.ts:47-50
if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
    child.whenCase = [...];
}
// 完全权限的字段不添加 CASE/WHEN 开销
```

### 9.3 空规则短路

```typescript
// filter/index.ts:42-48
if (key === '_or' && value.some(subFilter => Object.keys(subFilter).length === 0)) {
    // 空过滤器 = 完全权限，短路跳过其他条件
    if (value !== cases || value.length === 1) {
        continue;
    }
}
```

## 10. 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        1. 查询解析阶段                          │
├─────────────────────────────────────────────────────────────────┤
│  getAstFromQuery()                                              │
│  ├── 解析 fields / deep / filter                                │
│  └── 生成初始 AST（无权限信息）                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        2. 权限处理阶段                          │
├─────────────────────────────────────────────────────────────────┤
│  processAst()                                                   │
│  ├── fieldMapFromAst() → 提取所有访问路径                        │
│  ├── fetchPolicies() → 获取用户策略                              │
│  ├── fetchPermissions() → 获取权限规则                           │
│  ├── validatePathExistence() → 字段存在验证                      │
│  ├── validatePathPermissions() → 权限验证                        │
│  └── injectCases()                                              │
│       ├── 递归遍历 AST 节点                                      │
│       ├── getCases() → 按集合提取权限规则                         │
│       ├── 设置 whenCase（需要满足的规则索引）                      │
│       └── 递归处理关联节点（M2O/O2M/A2O） ← 权限传播              │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        3. SQL 生成阶段                          │
├─────────────────────────────────────────────────────────────────┤
│  getDBQuery()                                                   │
│  ├── 处理顶级过滤（包含权限规则）                                 │
│  ├── 构建 JOIN（关联查询）                                        │
│  ├── applyCaseWhen() → 字段级权限检查                            │
│  │   └── CASE WHEN <权限条件> THEN <列值> END                   │
│  └── O2M 子查询独立处理                                          │
│       └── 递归 applyQuery（注入子集合权限）                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        4. 深层过滤阶段                          │
├─────────────────────────────────────────────────────────────────┤
│  deepMapFilter() 遍历过滤条件                                    │
│  ├── 处理 _or/_and 逻辑运算                                      │
│  ├── 识别关联字段（M2O/O2M/A2O）                                 │
│  ├── 提取 _some/_none 量词                                      │
│  └── 递归处理子过滤条件                                          │
│                                                                 │
│  _some/_none 子查询：                                            │
│  ├── 构建子查询                                                  │
│  ├── 递归 applyQuery（包含权限）                                 │
│  └── WHERE IN / NOT IN 连接                                     │
└─────────────────────────────────────────────────────────────────┘
```

## 11. 关键代码位置索引

| 功能 | 文件 | 关键函数/行号 |
|-----|------|-------------|
| AST 权限处理入口 | `process-ast.ts` | `processAst()` L19 |
| 权限注入核心 | `inject-cases.ts` | `injectCases()` L13, `processChildren()` L17 |
| 权限规则提取 | `get-cases.ts` | `getCases()` L6 |
| 权限路径验证 | `validate-path-permissions.ts` | `validatePathPermissions()` L4 |
| 深层过滤遍历 | `deep-map-filter.ts` | `deepMapFilter()` L8 |
| SQL 过滤应用 | `filter/index.ts` | `applyFilter()` L17, `_some/_none` L145 |
| CASE/WHEN 应用 | `apply-case-when.ts` | `applyCaseWhen()` L21 |
| 关联字段权限 | `inject-cases.ts` | M2O L52, O2M L56, A2O L60 |
| 完全权限检测 | `get-cases.ts` | `allowedFields` L45-50 |

## 12. 权限错误 vs 结果过滤：触发条件详解

Directus 的权限系统采用了**双重策略**来处理权限限制：

| 处理方式 | 触发时机 | 表现形式 |
|---------|---------|---------|
| **直接报错 (ForbiddenError)** | 查询解析阶段（SQL 执行前） | HTTP 403 Forbidden，立即终止 |
| **字段置空 (CASE WHEN NULL)** | SQL 执行阶段 | 无权限字段返回 `null`，其他字段正常 |
| **结果过滤 (WHERE 条件)** | SQL 执行阶段 | 不符合权限条件的行不返回 |

### 12.1 直接报权限错误的场景

这类错误发生在 `processAst()` 阶段，即在生成 SQL 之前就被拦截。

#### 场景 1：目标集合无任何权限

**触发条件：**
- 查询路径指向的集合在当前用户权限中完全不存在
- 包括：`directus_permissions` 表中没有该 `collection + action` 的任何记录

**代码位置：** `validate-path-permissions.ts:10-14`

```typescript
const permissionsForCollection = permissions.filter(
    p => p.collection === collection
);

if (permissionsForCollection.length === 0) {
    throw createCollectionForbiddenError(path, collection);
}
```

**示例：**
```javascript
// 用户权限：只有 products 的读取权限
// 执行查询：查询 categories 集合
GET /items/categories
// 结果：403 Forbidden
// "You don't have permission to access collection 'categories'..."
```

**在关联查询中的表现：**
```javascript
// 查询 products，同时查询其关联的 category
{
    "fields": ["id", "name", "category.name"]
}

// 如果用户没有 categories 集合的读取权限
// 结果：403 Forbidden
// 错误信息会包含路径："Queried in 'category'"
```

#### 场景 2：访问未授权字段

**触发条件：**
- 有权访问集合，但请求的字段不在权限的 `fields` 配置中
- 权限的 `fields` 不是 `*`，且不包含请求的字段

**代码位置：** `validate-path-permissions.ts:16-42`

```typescript
const allowedFields: Set<string> = new Set();

for (const { fields } of permissionsForCollection) {
    if (!fields) continue;
    for (const field of fields) {
        if (field === '*') return;  // 有 * 就完全放行
        allowedFields.add(field);
    }
}

const forbiddenFields = requestedFields.filter(
    field => allowedFields.has(field) === false
);

if (forbiddenFields.length > 0) {
    throw createFieldsForbiddenError(path, collection, forbiddenFields);
}
```

**示例：**
```javascript
// 用户权限配置
{
    "collection": "products",
    "action": "read",
    "fields": ["id", "name", "status"],  // 没有 price 字段
    "permissions": {}
}

// 执行查询
GET /items/products?fields=id,name,price
// 结果：403 Forbidden
// "You don't have permission to access field 'price'..."
```

**在关联查询中的表现：**
```javascript
// 用户对 categories 只有 id 字段权限
// 但查询中请求了 category.name
{
    "fields": ["id", "category.id", "category.name"]
}
// 结果：403 Forbidden
// "Queried in 'category'"
```

#### 场景 3：字段/集合不存在

**触发条件：**
- 请求的集合在 schema 中不存在
- 请求的字段在集合 schema 中不存在

**代码位置：** `validate-path-existence.ts:4-17`

```typescript
export function validatePathExistence(path, collection, fields, schema) {
    const collectionInfo = schema.collections[collection];
    
    if (collectionInfo === undefined) {
        throw createCollectionForbiddenError(path, collection);
    }
    
    const nonExistentFields = requestedFields.filter(
        field => collectionInfo.fields[field] === undefined
    );
    
    if (nonExistentFields.length > 0) {
        throw createFieldsForbiddenError(path, collection, nonExistentFields);
    }
}
```

**注意：** 此检查即使对 `admin` 用户和 `accountability = null`（公开访问）也会执行。

#### 场景 4：写入/更新操作的权限验证

**触发条件：**
- `create`、`update`、`delete` 操作时验证失败
- 通过 `validateAccess()` 函数执行

**代码位置：** `validate-access.ts:22-57`

```typescript
export async function validateAccess(options, context) {
    // 跳过管理员
    if (options.accountability.admin === true) {
        return;
    }
    
    let access: boolean;
    
    if (options.primaryKeys) {
        // 有具体主键时，实际查询数据库验证
        const result = await validateItemAccess(options, context);
        access = result.accessAllowed;
    } else {
        // 无主键时，检查集合级权限
        access = await validateCollectionAccess(options, context);
    }
    
    if (!access) {
        throw new ForbiddenError({ ... });
    }
}
```

**关键区别：** 写入操作会实际查询数据库验证行级权限，而非仅在 AST 层面检查。

### 12.2 字段置空的场景

这类处理发生在 SQL 执行阶段，通过 `CASE WHEN ... THEN ... END` 实现。

#### 触发条件

- 有权访问集合（通过了 `validatePathPermissions`）
- 有权访问字段（`fields` 配置包含该字段）
- 但**行级权限条件** `permissions` 不为空（不是 `{}` 或 `null`）
- 当前记录不满足行级权限条件

**核心机制：**

```typescript
// inject-cases.ts:47-50
// 只有不完全权限时才设置 whenCase
if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
    child.whenCase = [...(globalWhenCase ?? []), ...(fieldWhenCase ?? [])];
}

// get-column-pre-processor.ts:73-92
if (hasWhenCase) {
    const columnCases: Filter[] = [];
    
    for (const index of fieldNode.whenCase) {
        columnCases.push(cases[index]!);
    }
    
    // 包裹成 CASE WHEN
    column = applyCaseWhen({
        column,
        columnCases,
        ...
    });
}

// apply-case-when.ts:51
let rawCase = `(CASE WHEN ${sql} THEN ?? END)`;
// 不满足条件时返回 NULL（没有 ELSE 子句）
```

**生成的 SQL：**
```sql
SELECT
    id,
    CASE WHEN `products`.`status` = 'published' 
         THEN `products`.`name` 
    END AS `name`,
    CASE WHEN `products`.`status` = 'published'
         THEN `products`.`price`
    END AS `price`
FROM `products`
```

**查询结果对比：**

| id | status | name (实际值) | name (查询结果) |
|----|--------|---------------|-----------------|
| 1 | published | "Product A" | "Product A" |
| 2 | draft | "Product B" | NULL |
| 3 | archived | "Product C" | NULL |

#### 在关联查询中的表现

**M2O 关联的字段置空：**

```sql
-- 主表权限 + 关联表权限的组合检查
SELECT
    `products`.`id`,
    CASE WHEN `products`.`status` = 'published'
         THEN `category`.`name`
    END AS `category__name`
FROM `products`
LEFT JOIN `categories` AS `category`
    ON `products`.`category_id` = `category`.`id`
```

**O2M 关联的特殊处理：**

O2M 关联通过 `whenCase` 标志位控制：

```typescript
// run-ast.ts:126-145
const hasWhenCase = nestedNode.whenCase && nestedNode.whenCase.length > 0;
let fieldAllowed: boolean | boolean[] = true;

if (hasWhenCase) {
    if (Array.isArray(items)) {
        fieldAllowed = [];
        for (const item of items) {
            // 从查询结果中提取标志位
            fieldAllowed.push(!!item[nestedNode.fieldKey]);
            delete item[nestedNode.fieldKey];
        }
    }
}

// 后续 mergeWithParentItems 时使用 fieldAllowed
```

**生成的 SQL 中包含标志位：**
```sql
SELECT
    `articles`.`id`,
    `articles`.`title`,
    -- 权限标志位
    CASE WHEN `articles`.`status` = 'published' THEN 1 END AS `comments`
FROM `articles`
```

**结果表现：**
```json
{
    "data": [
        {
            "id": 1,
            "title": "Published Article",
            "comments": [
                { "id": 1, "content": "Comment 1" },
                { "id": 2, "content": "Comment 2" }
            ]
        },
        {
            "id": 2,
            "title": "Draft Article",
            "comments": null  // 权限标志位为 false
        }
    ]
}
```

### 12.3 结果被过滤的场景

这类处理通过在 `WHERE` 子句中添加权限条件实现。

#### 触发条件

- 查询有 `filter` 参数，或者有权限规则
- 权限规则的 `permissions` 不为空

**核心机制：** `joinFilterWithCases` 将用户过滤条件与权限条件合并

```typescript
// join-filter-with-cases.ts:3-12
export function joinFilterWithCases(filter, cases) {
    if (cases.length > 0 && !filter) {
        return { _or: cases };  // 只有权限条件
    } else if (filter && cases.length === 0) {
        return filter;  // 只有用户过滤条件
    } else if (filter && cases.length > 0) {
        return { _and: [filter, { _or: cases }] };  // 两者都有，AND 合并
    }
    return null;
}
```

**权限规则的语义：**

```typescript
// permissions = {} 或 null → 完全权限，无行级限制
// permissions = { status: { _eq: 'published' } } → 行级限制
```

**SQL 生成：**
```sql
-- 用户过滤 + 权限条件
WHERE (
    `products`.`category` = 'electronics'  -- 用户过滤
    AND (
        `products`.`status` = 'published'  -- 权限条件（OR 组合）
        OR `products`.`created_by` = 123
    )
)
```

#### 深层过滤中的结果过滤

**关联字段过滤（M2O）：**

```javascript
{
    "filter": {
        "category": {
            "name": { "_eq": "Electronics" }
        }
    }
}
```

**权限传播：**
1. 主查询应用权限条件
2. JOIN 关联表时，关联表的权限条件也会被检查
3. 用户过滤条件与权限条件通过 AND 合并

**_some/_none 量词的结果过滤：**

```javascript
{
    "filter": {
        "comments": {
            "_some": {
                "status": { "_eq": "approved" }
            }
        }
    }
}
```

**SQL 生成（包含权限）：**
```sql
WHERE `articles`.`id` IN (
    SELECT `comments`.`article_id`
    FROM `comments`
    WHERE `comments`.`status` = 'approved'
      -- 自动注入的权限条件
      AND `comments`.`is_deleted` = 0
)
```

### 12.4 三种场景的决策流程图

```
用户查询请求
    │
    ▼
┌─────────────────────────────────┐
│  processAst() 阶段              │
│  (SQL 执行前)                   │
├─────────────────────────────────┤
│  检查访问路径权限                │
│                                 │
│  ┌─ 集合无权限? ── YES ─────────┼─→ ForbiddenError (403)
│  │                              │
│  ├─ 字段不在 fields 中? ── YES ─┼─→ ForbiddenError (403)
│  │                              │
│  └─ 字段/集合不存在? ── YES ────┼─→ ForbiddenError (403)
│                                 │
└─────────────────────────────────┘
    │
    ▼ 权限验证通过，开始生成 SQL
┌─────────────────────────────────┐
│  SQL 执行阶段                   │
│  (数据库层面)                   │
├─────────────────────────────────┤
│                                 │
│  WHERE 条件过滤                 │
│  ┌───────────────────────────┐  │
│  │ 用户过滤 AND 权限条件      │  │
│  │ 不满足条件的行 ──→ 不返回  │  │
│  └───────────────────────────┘  │
│                                 │
│  SELECT 字段处理                │
│  ┌───────────────────────────┐  │
│  │ CASE WHEN 权限条件        │  │
│  │ THEN 列值 ELSE NULL       │  │
│  │ 不满足条件的字段 ──→ NULL │  │
│  └───────────────────────────┘  │
│                                 │
│  O2M 关联处理                   │
│  ┌───────────────────────────┐  │
│  │ 权限标志位决定是否查询关联 │  │
│  │ 标志位为 0 ──→ 关联为 null│  │
│  └───────────────────────────┘  │
│                                 │
└─────────────────────────────────┘
```

## 13. 完整追踪示例：同时包含 M2O 与 O2M 的查询

### 13.1 原示例的问题：validatePathPermissions 阶段报错

#### 数据模型和查询保持不变

**数据模型：**
```
articles
├── id (PK)
├── title
├── content
├── status
├── author_id (FK → users.id)  ← M2O
└── created_at

users
├── id (PK)
├── name
├── email
└── role

comments  ← O2M from articles
├── id (PK)
├── article_id (FK → articles.id)
├── content
└── is_approved
```

**用户查询：**
```javascript
GET /items/articles
{
    "fields": [
        "id",
        "title",
        "author.id",
        "author.name",
        "comments.id",
        "comments.content"
    ],
    "filter": {
        "created_at": { "_gte": "2024-01-01" }
    },
    "deep": {
        "comments": {
            "_limit": 5,
            "_sort": ["-id"]
        }
    }
}
```

#### 原权限配置（有问题的版本）

```javascript
// 权限 1: articles - 只能看已发布的文章
{
    "collection": "articles",
    "action": "read",
    "fields": ["id", "title", "status", "author"],  // ⚠️ 缺少 comments 字段
    "permissions": { "status": { "_eq": "published" } }
}

// 权限 2: users - 只能看作者信息（不是管理员）
{
    "collection": "users",
    "action": "read",
    "fields": ["id", "name"],
    "permissions": { "role": { "_neq": "admin" } }
}

// 权限 3: comments - 只能看已批准的评论
{
    "collection": "comments",
    "action": "read",
    "fields": ["*"],
    "permissions": { "is_approved": { "_eq": true } }
}
```

#### 问题分析：为什么会报错

**关键背景：FieldMap 的 read vs other 分区**

根据源码 `field-map-from-ast.ts:7-13`，FieldMap 提取分为两步：

```typescript
export function fieldMapFromAst(ast: AST, schema: SchemaOverview): FieldMap {
    const fieldMap: FieldMap = { read: new Map(), other: new Map() };

    extractFieldsFromChildren(ast.name, ast.children, fieldMap, schema);  // 第一步
    extractFieldsFromQuery(ast.name, ast.query, fieldMap, schema);         // 第二步

    return fieldMap;
}
```

**步骤 1：extractFieldsFromChildren 写入 `other` map**

`extract-fields-from-children.ts:16` 明确写入 `other` map：

```typescript
const info = getInfoForPath(fieldMap, 'other', path, collection);
//                                           ^^^^^ 写入 other map

for (const child of children) {
    info.fields.add(getUnaliasedFieldKey(child));
    // 递归处理关联节点...
}
```

**步骤 2：extractFieldsFromQuery 写入 `read` 或 `other` map**

`extract-fields-from-query.ts:17-22` 根据路径类型区分：

```typescript
const { paths: otherPaths, readOnlyPaths } = extractPathsFromQuery(query);

const groupedPaths = {
    other: otherPaths,      // aggregate、group → other map
    read: readOnlyPaths,    // filter、sort → read map
};
```

根据 `extract-paths-from-query.ts:26-84`：
- **`readOnlyPaths`**：来自 `filter`、`sort` 条件中的字段
- **`otherPaths`**：来自 `aggregate`、`group` 条件中的字段

---

**现在回到示例，逐步追踪 FieldMap 提取：**

**查询内容：**
```javascript
{
    "fields": ["id", "title", "author.id", "author.name", "comments.id", "comments.content"],
    "filter": {
        "created_at": { "_gte": "2024-01-01" }
    },
    "deep": {
        "comments": {
            "_limit": 5,
            "_sort": ["-id"]
        }
    }
}
```

**初始 AST：**
```
AST {
    name: 'articles',
    query: { filter: { created_at: { _gte: '2024-01-01' } } },
    children: [
        { type: 'field', fieldKey: 'id' },
        { type: 'field', fieldKey: 'title' },
        {
            type: 'm2o', fieldKey: 'author',
            relation: { related_collection: 'users' },
            children: [
                { type: 'field', fieldKey: 'id' },
                { type: 'field', fieldKey: 'name' }
            ],
            query: {}
        },
        {
            type: 'o2m', fieldKey: 'comments',
            relation: { collection: 'comments' },
            children: [
                { type: 'field', fieldKey: 'id' },
                { type: 'field', fieldKey: 'content' }
            ],
            query: { limit: 5, sort: ['-id'] }
        }
    ]
}
```

---

**FieldMap 提取逐步追踪：**

**第一步：extractFieldsFromChildren（写入 `other` map）**

调用：`extractFieldsFromChildren('articles', ast.children, fieldMap, schema, [])`

1. **处理根路径 `''`（articles）**
   ```
   getInfoForPath(fieldMap, 'other', [], 'articles')
   → fieldMap.other.set('', { collection: 'articles', fields: Set{} })
   
   遍历 children，添加 fieldKey：
   - child 'id' → fields.add('id')
   - child 'title' → fields.add('title')
   - child 'author' → fields.add('author')
   - child 'comments' → fields.add('comments')  // ⚠️ 这里！
   ```

2. **递归处理 M2O 关联 `author`（path = ['author']）**
   ```
   调用：extractFieldsFromChildren('users', author.children, fieldMap, schema, ['author'])
   
   getInfoForPath(fieldMap, 'other', ['author'], 'users')
   → fieldMap.other.set('author', { collection: 'users', fields: Set{} })
   
   遍历 children，添加 fieldKey：
   - child 'id' → fields.add('id')
   - child 'name' → fields.add('name')
   
   调用 extractFieldsFromQuery('users', {}, ...) → query 为空，无操作
   ```

3. **递归处理 O2M 关联 `comments`（path = ['comments']）**
   ```
   调用：extractFieldsFromChildren('comments', comments.children, fieldMap, schema, ['comments'])
   
   getInfoForPath(fieldMap, 'other', ['comments'], 'comments')
   → fieldMap.other.set('comments', { collection: 'comments', fields: Set{} })
   
   遍历 children，添加 fieldKey：
   - child 'id' → fields.add('id')
   - child 'content' → fields.add('content')
   
   调用 extractFieldsFromQuery('comments', { limit: 5, sort: ['-id'] }, ...)
   ```

**第二步：extractFieldsFromQuery（写入 `read` 或 `other` map）**

调用：`extractFieldsFromQuery('articles', { filter: { created_at: ... } }, fieldMap, schema, [])`

1. **根查询的 filter → `read` map**
   ```
   extractPathsFromQuery({ filter: { created_at: ... } })
   → readOnlyPaths = [['created_at']]
   → otherPaths = []
   
   处理路径 ['created_at']：
   getInfoForPath(fieldMap, 'read', [], 'articles')
   → fieldMap.read.set('', { collection: 'articles', fields: Set{} })
   info.fields.add('created_at')
   ```

2. **O2M 关联的 sort → `read` map**
   在第一步的 O2M 递归中已调用：
   ```
   extractFieldsFromQuery('comments', { limit: 5, sort: ['-id'] }, ..., ['comments'])
   
   extractPathsFromQuery({ limit: 5, sort: ['-id'] })
   → readOnlyPaths = [['id']]
   → otherPaths = []
   
   处理路径 ['id']：
   getInfoForPath(fieldMap, 'read', ['comments'], 'comments')
   → fieldMap.read.set('comments', { collection: 'comments', fields: Set{} })
   info.fields.add('id')
   ```

---

**最终 FieldMap 结构：**

```javascript
fieldMap = {
    // other map：来自 extractFieldsFromChildren（AST children 的字段）
    other: Map{
        '': {
            collection: 'articles',
            fields: Set{'id', 'title', 'author', 'comments'}  // ⚠️ 包含 comments
        },
        'author': {
            collection: 'users',
            fields: Set{'id', 'name'}
        },
        'comments': {
            collection: 'comments',
            fields: Set{'id', 'content'}
        }
    },
    // read map：来自 extractFieldsFromQuery（filter/sort 中的字段）
    read: Map{
        '': {
            collection: 'articles',
            fields: Set{'created_at'}
        },
        'comments': {
            collection: 'comments',
            fields: Set{'id'}
        }
    }
}
```

---

**校验顺序：先校验 `other`，再校验 `read`**

根据 `process-ast.ts:54-62`：

```typescript
// 第一步：校验 fieldMap.other（使用 permissions）
for (const [path, { collection, fields }] of fieldMap.other.entries()) {
    validatePathPermissions(path, permissions, collection, fields);
}

// 第二步：校验 fieldMap.read（使用 readPermissions）
for (const [path, { collection, fields }] of fieldMap.read.entries()) {
    validatePathPermissions(path, readPermissions, collection, fields);
}
```

**对于 action === 'read' 的特殊情况**（process-ast.ts:41-47）：
```typescript
const readPermissions =
    options.action === 'read'
        ? permissions  // 同一个权限集合
        : await fetchPermissions({ action: 'read', ... }, context);
```

---

**报错触发详细过程：**

**第一步：校验 `fieldMap.other`**

原权限配置：
```javascript
{
    "collection": "articles",
    "fields": ["id", "title", "status", "author"],  // ⚠️ 缺少 comments
    "permissions": { "status": { "_eq": "published" } }
}
```

校验路径 `''` (articles)：
```typescript
// fieldMap.other.get('')
const requestedFields = ['id', 'title', 'author', 'comments'];

// 构建 allowedFields
const allowedFields = new Set(['id', 'title', 'status', 'author']);

// 检查
const forbiddenFields = requestedFields.filter(
    field => allowedFields.has(field) === false
);

// forbiddenFields = ['comments']

if (forbiddenFields.length > 0) {
    throw createFieldsForbiddenError('', 'articles', ['comments']);
}
```

**报错触发：**
```
HTTP 403 Forbidden
"You don't have permission to access the following fields in 'articles': comments"
```

**注意：**
- 报错发生在校验 `fieldMap.other` 的第一步
- `fieldMap.read` 还没开始校验
- 对于 `action === 'read'`，即使校验 `read` map 也会用相同的权限集合，结果相同

**根本原因：**
- 即使 `comments` 是关联字段（O2M），它仍然作为父集合的字段被检查
- Directus 的权限模型中：访问 `articles.comments` 需要同时满足：
  1. `articles` 集合的 `fields` 包含 `comments`（或为 `*`）
  2. `comments` 集合有读取权限

### 13.2 修正后的权限配置

**方法：在 articles 的权限中添加 `comments` 字段**

```javascript
// 权限 1: articles - 只能看已发布的文章
{
    "collection": "articles",
    "action": "read",
    "fields": ["id", "title", "status", "author", "comments"],  // ✅ 添加了 comments
    "permissions": { "status": { "_eq": "published" } }
}

// 权限 2: users - 只能看作者信息（不是管理员）
{
    "collection": "users",
    "action": "read",
    "fields": ["id", "name"],
    "permissions": { "role": { "_neq": "admin" } }
}

// 权限 3: comments - 只能看已批准的评论
{
    "collection": "comments",
    "action": "read",
    "fields": ["*"],
    "permissions": { "is_approved": { "_eq": true } }
}
```

**或者更简单的方式：使用 `*`**
```javascript
{
    "collection": "articles",
    "action": "read",
    "fields": ["*"],  // 所有字段
    "permissions": { "status": { "_eq": "published" } }
}
```

---

### 13.3 完整追踪：修正后的权限配置

#### 阶段一：查询解析为初始 AST

**调用：** `getAstFromQuery()`

**初始 AST 结构（无权限信息）：**

```
AST {
    type: 'root',
    name: 'articles',
    query: {
        filter: { created_at: { _gte: '2024-01-01' } }
    },
    children: [
        { type: 'field', name: 'id', fieldKey: 'id' },
        { type: 'field', name: 'title', fieldKey: 'title' },
        {
            type: 'm2o',
            name: 'author',
            fieldKey: 'author',
            relation: {
                field: 'author_id',
                related_collection: 'users'
            },
            children: [
                { type: 'field', name: 'id', fieldKey: 'id' },
                { type: 'field', name: 'name', fieldKey: 'name' }
            ],
            query: {}
        },
        {
            type: 'o2m',
            name: 'comments',
            fieldKey: 'comments',
            relation: {
                field: 'article_id',
                collection: 'comments'
            },
            children: [
                { type: 'field', name: 'id', fieldKey: 'id' },
                { type: 'field', name: 'content', fieldKey: 'content' }
            ],
            query: {
                limit: 5,
                sort: ['-id']
            }
        }
    ],
    cases: [],
    whenCase: undefined
}
```

**FieldMap 提取：**

```javascript
fieldMap = {
    read: Map{
        '': { collection: 'articles', fields: Set{'id', 'title', 'author', 'comments'} },
        'author': { collection: 'users', fields: Set{'id', 'name'} },
        'comments': { collection: 'comments', fields: Set{'id', 'content'} }
    },
    other: Map{}
}
```

#### 阶段二：权限验证与注入

**调用：** `processAst()`

##### 步骤 1：权限获取

```javascript
permissions = [
    {
        collection: 'articles',
        permissions: { status: { _eq: 'published' } },
        fields: ['id', 'title', 'status', 'author', 'comments']
    },
    {
        collection: 'users',
        permissions: { role: { _neq: 'admin' } },
        fields: ['id', 'name']
    },
    {
        collection: 'comments',
        permissions: { is_approved: { _eq: true } },
        fields: ['*']
    }
]
```

##### 步骤 2：路径权限验证（validatePathPermissions）

**路径 '' (articles)：**
- 权限存在 ✓
- 请求字段：`id`, `title`, `author`, `comments`
- 权限字段：`id`, `title`, `status`, `author`, `comments`
- 完全匹配 ✓（现在包含 comments 了）

**路径 'author' (users)：**
- 权限存在 ✓
- 请求字段：`id`, `name`
- 权限字段：`id`, `name`
- 完全匹配 ✓

**路径 'comments' (comments)：**
- 权限存在 ✓
- 请求字段：`id`, `content`
- 权限字段：`*`
- 完全权限 ✓

**验证通过，继续执行 injectCases**

##### 步骤 3：injectCases 递归注入

**第一层：articles（根节点）**

```typescript
// getCases('articles', permissions, ['id', 'title', 'author', 'comments'])
// 从 articles 的权限规则中提取
cases = [
    { status: { _eq: 'published' } }
]
caseMap = {
    'id': [0],
    'title': [0],
    'status': [0],
    'author': [0],
    'comments': [0]  // ✅ 现在包含了
}
allowedFields = new Set()  // permissions 不是空对象
```

**设置根节点 cases：**
```
ast.cases = [{ status: { _eq: 'published' } }]
```

**处理子节点：**

| 子节点 | whenCase | 说明 |
|-------|----------|------|
| field: id | [0] | 需要满足规则 0 |
| field: title | [0] | 需要满足规则 0 |
| m2o: author | [0] | 需要满足规则 0，递归处理 |
| o2m: comments | [0] | 需要满足规则 0，递归处理 |

**第二层：M2O 关联 - author (users)**

```typescript
// getCases('users', permissions, ['id', 'name'])
cases = [
    { role: { _neq: 'admin' } }
]
caseMap = {
    'id': [0],
    'name': [0]
}
allowedFields = new Set()
```

**设置 M2O 节点：**
```
m2oNode = {
    type: 'm2o',
    name: 'author',
    whenCase: [0],  // 父级权限索引
    cases: [{ role: { _neq: 'admin' } }],  // 自身权限规则
    children: [
        { type: 'field', name: 'id', whenCase: [0] },
        { type: 'field', name: 'name', whenCase: [0] }
    ]
}
```

**第三层：O2M 关联 - comments (comments)**

```typescript
// getCases('comments', permissions, ['id', 'content'])
cases = [
    { is_approved: { _eq: true } }
]
caseMap = {
    '*': [0]  // fields: ['*']
}
allowedFields = new Set()  // permissions 不是空对象
```

**设置 O2M 节点：**
```
o2mNode = {
    type: 'o2m',
    name: 'comments',
    whenCase: [0],  // 父级权限索引
    cases: [{ is_approved: { _eq: true } }],  // 自身权限规则
    children: [
        { type: 'field', name: 'id', whenCase: [0] },
        { type: 'field', name: 'content', whenCase: [0] }
    ]
}
```

##### 最终注入权限后的 AST

```
AST {
    type: 'root',
    name: 'articles',
    query: { filter: { created_at: { _gte: '2024-01-01' } } },
    cases: [
        { status: { _eq: 'published' } }  // 根节点权限规则
    ],
    children: [
        {
            type: 'field',
            name: 'id',
            fieldKey: 'id',
            whenCase: [0]  // 需满足 cases[0]
        },
        {
            type: 'field',
            name: 'title',
            fieldKey: 'title',
            whenCase: [0]
        },
        {
            type: 'm2o',
            name: 'author',
            fieldKey: 'author',
            relation: { field: 'author_id', related_collection: 'users' },
            whenCase: [0],  // 父级权限
            cases: [
                { role: { _neq: 'admin' } }  // M2O 自身权限
            ],
            children: [
                { type: 'field', name: 'id', whenCase: [0] },
                { type: 'field', name: 'name', whenCase: [0] }
            ]
        },
        {
            type: 'o2m',
            name: 'comments',
            fieldKey: 'comments',
            relation: { field: 'article_id', collection: 'comments' },
            whenCase: [0],  // 父级权限
            cases: [
                { is_approved: { _eq: true } }  // O2M 自身权限
            ],
            children: [
                { type: 'field', name: 'id', whenCase: [0] },
                { type: 'field', name: 'content', whenCase: [0] }
            ],
            query: { limit: 5, sort: ['-id'] }
        }
    ]
}
```

#### 阶段三：SQL 生成

##### 主查询 SQL（articles）

**调用：** `getDBQuery()` + `runAst()`

**过滤条件合并：**
```typescript
// joinFilterWithCases(
//     { created_at: { _gte: '2024-01-01' } },  // 用户过滤
//     [{ status: { _eq: 'published' } }]         // 权限条件
// )

result = {
    _and: [
        { created_at: { _gte: '2024-01-01' } },
        { _or: [{ status: { _eq: 'published' } }] }
    ]
}
```

**生成的主查询 SQL：**

```sql
SELECT
    `articles`.`id`,
    -- CASE WHEN 包裹有权限限制的字段
    CASE WHEN `articles`.`status` = 'published'
         THEN `articles`.`title`
    END AS `title`,
    -- 权限标志位（用于 O2M 关联）
    CASE WHEN `articles`.`status` = 'published' THEN 1 END AS `comments`,
    -- M2O 外键（用于后续查询）
    `articles`.`author_id`
FROM `articles`
WHERE
    -- 用户过滤 + 权限条件
    `articles`.`created_at` >= '2024-01-01'
    AND `articles`.`status` = 'published'
```

**关键点：**
- `title` 字段被 `CASE WHEN` 包裹：不满足权限条件时返回 NULL
- `comments` 作为权限标志位：决定是否查询 O2M 关联
- `author_id` 直接选择：用于后续 M2O 关联查询

##### M2O 关联的独立查询

M2O 字段通过单独的 SELECT 查询（不是 JOIN）：

```sql
-- 查询 author 关联
SELECT
    `users`.`id`,
    CASE WHEN `users`.`role` != 'admin'
         THEN `users`.`name`
    END AS `name`
FROM `users`
WHERE
    `users`.`id` IN (101, 102, 103)  -- 从主查询结果提取的 author_id
    AND `users`.`role` != 'admin'  -- 权限条件注入 WHERE
```

**关键点：**
- 权限条件同时出现在 `WHERE`（过滤不满足条件的行）和 `CASE WHEN`（字段置空）
- 这是双重保护机制

##### O2M 关联的独立查询

```typescript
// runAst.ts:121-167
// O2M 关联通过递归 runAst 处理
nestedItems = await runAst(o2mNode, schema, accountability, { knex, nested: true });
```

**生成的 O2M 查询 SQL：**

```sql
SELECT
    `comments`.`id`,
    CASE WHEN `comments`.`is_approved` = 1
         THEN `comments`.`content`
    END AS `content`,
    `comments`.`article_id`  -- 用于关联回父表
FROM `comments`
WHERE
    `comments`.`article_id` IN (1, 2, 3)  -- 父级主键
    AND `comments`.`is_approved` = 1  -- 权限条件
ORDER BY `comments`.`id` DESC
LIMIT 5
```

#### 阶段四：结果合并

**假设数据库中的数据：**

```
articles:
┌────┬───────────┬────────────┬─────────────────────────┐
│ id │ status    │ title      │ author_id               │
├────┼───────────┼────────────┼─────────────────────────┤
│ 1  │ published │ Article A  │ 101 (role: 'editor')    │
│ 2  │ draft     │ Article B  │ 102 (role: 'admin')     │
│ 3  │ published │ Article C  │ 102 (role: 'admin')     │
└────┴───────────┴────────────┴─────────────────────────┘

comments:
┌────┬────────────┬─────────────┬─────────────┐
│ id │ article_id │ content     │ is_approved │
├────┼────────────┼─────────────┼─────────────┤
│ 1  │ 1          │ Comment 1   │ true        │
│ 2  │ 1          │ Comment 2   │ false       │
│ 3  │ 2          │ Comment 3   │ true        │
│ 4  │ 3          │ Comment 4   │ true        │
└────┴────────────┴─────────────┴─────────────┘
```

##### 主查询结果（WHERE 过滤后）

```sql
WHERE created_at >= '2024-01-01' AND status = 'published'
```

结果只包含 `id=1` 和 `id=3`（`id=2` 的 `status='draft'` 被过滤）

**主查询返回行：**

| id | title | comments (flag) | author_id |
|----|-------|-----------------|-----------|
| 1 | Article A | 1 | 101 |
| 3 | Article C | 1 | 102 |

##### id=1 (published, author=101 role='editor')

**主查询行数据：**
```javascript
{
    id: 1,
    title: 'Article A',  // CASE WHEN 条件满足
    comments: 1,         // 权限标志位 = true
    author_id: 101
}
```

**M2O 关联查询（users）：**
```sql
WHERE id = 101 AND role != 'admin'
```

查询返回：
```javascript
{ id: 101, name: 'User 101' }
```

**O2M 关联查询（comments）：**
```sql
WHERE article_id = 1 AND is_approved = true
ORDER BY id DESC
LIMIT 5
```

查询返回（只包含 is_approved=true 的评论）：
```javascript
[
    { id: 1, content: 'Comment 1', article_id: 1 }
]
```

##### id=3 (published, author=102 role='admin')

**主查询行数据：**
```javascript
{
    id: 3,
    title: 'Article C',  // CASE WHEN 条件满足
    comments: 1,         // 权限标志位 = true
    author_id: 102
}
```

**M2O 关联查询（users）：**
```sql
WHERE id = 102 AND role != 'admin'
```

**查询结果为空！** 因为 `role='admin'` 不满足权限条件

```javascript
[]  // 空结果
```

**O2M 关联查询（comments）：**
```sql
WHERE article_id = 3 AND is_approved = true
ORDER BY id DESC
LIMIT 5
```

查询返回：
```javascript
[
    { id: 4, content: 'Comment 4', article_id: 3 }
]
```

#### 阶段五：最终返回结果

**合并处理后的结果：**

```json
{
    "data": [
        {
            "id": 1,
            "title": "Article A",
            "author": {
                "id": 101,
                "name": "User 101"
            },
            "comments": [
                {
                    "id": 1,
                    "content": "Comment 1"
                }
            ]
        },
        {
            "id": 3,
            "title": "Article C",
            "author": null,  // author.role='admin'，WHERE 过滤掉了
            "comments": [
                {
                    "id": 4,
                    "content": "Comment 4"
                }
            ]
        }
    ]
}
```

**各字段的权限处理总结：**

| 数据 | 权限处理方式 | 结果 |
|-----|-------------|------|
| Article 2 (draft) | WHERE 过滤 | 不返回 |
| Article 1.title | CASE WHEN 满足 | 'Article A' |
| Article 1.author (role='editor') | WHERE 满足 | 返回对象 |
| Article 3.author (role='admin') | WHERE 不满足 | null |
| Comment 2 (is_approved=false) | WHERE 过滤 | 不返回 |

### 13.4 权限传播路径总结图

```
用户请求
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 1. 初始 AST（无权限）                                       │
├────────────────────────────────────────────────────────────┤
│ articles                                                   │
│ ├── id                                                     │
│ ├── title                                                  │
│ ├── author (M2O → users)                                   │
│ │   ├── id                                                 │
│ │   └── name                                               │
│ └── comments (O2M → comments)                              │
│     ├── id                                                 │
│     └── content                                            │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 2. validatePathPermissions 验证                            │
├────────────────────────────────────────────────────────────┤
│ 路径 '' (articles):                                        │
│   请求字段: {id, title, author, comments}                  │
│   权限字段: {id, title, status, author, comments}          │
│   ✓ 完全匹配（原示例缺少 comments，会 403）                  │
│                                                            │
│ 路径 'author' (users):                                     │
│   请求字段: {id, name}                                     │
│   权限字段: {id, name}                                     │
│   ✓ 完全匹配                                                │
│                                                            │
│ 路径 'comments' (comments):                                │
│   请求字段: {id, content}                                  │
│   权限字段: {*}                                            │
│   ✓ 完全权限                                                │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 3. 注入权限后的 AST                                         │
├────────────────────────────────────────────────────────────┤
│ articles                                                   │
│ ├── cases: [{ status: 'published' }]                       │
│ ├── id          (whenCase: [0])                            │
│ ├── title       (whenCase: [0])                            │
│ ├── author (M2O)                                           │
│ │   ├── whenCase: [0]        ← 父级权限                    │
│ │   ├── cases: [{ role: '!= admin' }] ← 自身权限          │
│ │   ├── id      (whenCase: [0])                            │
│ │   └── name    (whenCase: [0])                            │
│ └── comments (O2M)                                         │
│     ├── whenCase: [0]        ← 父级权限                    │
│     ├── cases: [{ is_approved: true }] ← 自身权限          │
│     ├── id      (whenCase: [0])                            │
│     └── content (whenCase: [0])                            │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 4. SQL 生成                                                 │
├────────────────────────────────────────────────────────────┤
│ 主查询 articles:                                            │
│ ├── WHERE: created_at >= ? AND status = 'published'        │
│ ├── SELECT: id, CASE title, CASE comments_flag             │
│ └── 不使用 JOIN，M2O/O2M 均为独立查询                        │
│                                                            │
│ 子查询 users (M2O):                                         │
│ ├── WHERE: id IN (?) AND role != 'admin'                   │
│ └── SELECT: id, CASE name                                  │
│                                                            │
│ 子查询 comments (O2M):                                      │
│ ├── WHERE: article_id IN (?) AND is_approved = true        │
│ ├── ORDER BY: id DESC                                      │
│ └── LIMIT: 5                                               │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│ 5. 结果过滤                                                 │
├────────────────────────────────────────────────────────────┤
│ id=1: ✓ (published)                                        │
│ ├── title: 'Article A'     (CASE WHEN 满足)                │
│ ├── author: {id:101, name:'...'} (role!='admin')           │
│ └── comments: [Comment 1]   (is_approved=true)             │
│                                                            │
│ id=2: ✗ (draft → WHERE 过滤)                               │
│                                                            │
│ id=3: ✓ (published)                                        │
│ ├── title: 'Article C'     (CASE WHEN 满足)                │
│ ├── author: null            (role='admin' → WHERE 过滤)    │
│ └── comments: [Comment 4]   (is_approved=true)             │
└────────────────────────────────────────────────────────────┘
```

### 13.5 关键教训

**关联字段的权限检查是双重的：**

1. **父集合层面**：访问 `articles.comments` 需要 `articles` 的 `fields` 包含 `comments`
2. **子集合层面**：需要 `comments` 集合有读取权限

**常见配置错误：**
```javascript
// ❌ 错误：父集合 fields 不包含关联字段
{
    "collection": "articles",
    "fields": ["id", "title"],  // 缺少 comments
    "permissions": { ... }
}

// ✅ 正确：父集合 fields 包含关联字段
{
    "collection": "articles",
    "fields": ["id", "title", "comments"],  // 包含 comments
    "permissions": { ... }
}

// ✅ 或者使用 *
{
    "collection": "articles",
    "fields": ["*"],  // 所有字段
    "permissions": { ... }
}
```

## 14. 总结

Directus 的权限传播机制具有以下特点：

1. **基于 AST 的递归注入**：通过 `injectCases()` 递归遍历 AST 每个节点，为每个集合独立注入权限规则。

2. **双层权限控制**：
   - `cases` 数组存储所有权限规则
   - `whenCase` 索引数组指定每个字段需要满足哪些规则

3. **关联类型差异化处理**：
   - **M2O**：通过 LEFT JOIN + CASE/WHEN 实现
   - **O2M**：通过独立子查询 + 递归 `applyQuery` 实现
   - **A2O**：为每个目标集合独立维护权限规则

4. **深层过滤权限保证**：
   - `_some/_none` 量词的子查询会递归应用权限
   - `deepMapFilter` 支持遍历任意深度的关联过滤
   - 所有 JOIN 和子查询都会经过权限检查

5. **三级权限处理策略**：
   - **403 Forbidden**：集合/字段完全无权限（SQL 执行前）
   - **字段置空 NULL**：行级权限条件不满足（SQL CASE/WHEN）
   - **结果过滤**：WHERE 条件排除不满足权限的行

6. **性能优化**：
   - 跳过无关规则
   - 完全权限字段省略 CASE/WHEN
   - 空规则短路逻辑运算

该设计确保了：
- 权限检查在 SQL 级别执行，保证数据安全
- 关联查询的每一层都有独立的权限控制
- 深层过滤条件不会绕过权限限制
- 在保证安全的同时兼顾查询性能
