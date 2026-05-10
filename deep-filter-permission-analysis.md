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

## 12. 总结

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

5. **性能优化**：
   - 跳过无关规则
   - 完全权限字段省略 CASE/WHEN
   - 空规则短路逻辑运算

该设计确保了：
- 权限检查在 SQL 级别执行，保证数据安全
- 关联查询的每一层都有独立的权限控制
- 深层过滤条件不会绕过权限限制
- 在保证安全的同时兼顾查询性能
