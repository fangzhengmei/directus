# Directus 角色权限系统深度分析

## 目录

1. [权限系统架构概览](#1-权限系统架构概览)
2. [数据读写限制机制](#2-数据读写限制机制)
3. [字段级权限与集合级权限的叠加顺序](#3-字段级权限与集合级权限的叠加顺序)
4. [GraphQL 与 REST 共用权限核心的机制](#4-graphql-与-rest-共用权限核心的机制)

---

## 1. 权限系统架构概览

### 1.1 核心组件

Directus 的权限系统采用模块化设计，位于 `api/src/permissions/` 目录，核心组件包括：

#### 1.1.1 权限获取层

| 模块 | 文件 | 职责 |
|------|------|------|
| 策略获取 | `lib/fetch-policies.ts` | 获取与用户关联的策略列表，支持角色继承和 IP 过滤 |
| 权限获取 | `lib/fetch-permissions.ts` | 从数据库获取原始权限并处理动态变量 |
| 原始权限 | `utils/fetch-raw-permissions.ts` | 缓存权限查询，支持按策略、操作、集合过滤 |

#### 1.1.2 权限验证模块

| 模块 | 文件 | 职责 |
|------|------|------|
| 访问验证 | `modules/validate-access/validate-access.ts` | 统一的权限验证入口 |
| 集合访问验证 | `modules/validate-access/lib/validate-collection-access.ts` | 验证用户对集合的访问权限 |
| 项目访问验证 | `modules/validate-access/lib/validate-item-access.ts` | 验证用户对具体数据项的访问权限 |
| 字段权限 | `modules/fetch-allowed-fields/fetch-allowed-fields.ts` | 获取用户可访问的字段列表 |
| 集合权限 | `modules/fetch-allowed-collections/fetch-allowed-collections.ts` | 获取用户可访问的集合列表 |
| 字段映射 | `modules/fetch-allowed-field-map/fetch-allowed-field-map.ts` | 获取所有集合的字段权限映射 |

#### 1.1.3 AST 处理模块

| 模块 | 文件 | 职责 |
|------|------|------|
| AST 处理 | `modules/process-ast/process-ast.ts` | GraphQL 查询 AST 的权限处理核心 |
| 路径验证 | `modules/process-ast/utils/validate-path/` | 验证字段路径的存在性和权限 |
| 案例注入 | `modules/process-ast/lib/inject-cases.ts` | 将权限规则注入查询 AST |

#### 1.1.4 权限合并模块

| 模块 | 文件 | 职责 |
|------|------|------|
| 权限合并 | `utils/merge-permissions.ts` | 合并多个权限规则 |
| 字段合并 | `utils/merge-fields.ts` | 合并字段权限规则 |

### 1.2 策略（Policy）系统

Directus 使用 **Policy（策略）** 作为权限的组织单位。每个 Policy 包含一组权限规则，用户可以通过以下方式关联策略：

1. **用户级策略**：直接附加到用户的策略
2. **角色级策略**：通过角色继承的策略
3. **公共策略**：无角色用户使用的公共策略

策略获取逻辑位于 `api/src/permissions/lib/fetch-policies.ts:44-55`，优先级排序为：

```
父角色策略 → 子角色策略 → 用户策略
```

### 1.3 权限数据结构

每个权限规则包含以下核心属性：

- **collection**: 关联的集合（表）
- **action**: 操作类型（`read`, `create`, `update`, `delete`）
- **fields**: 允许的字段列表（`*` 表示所有字段）
- **permissions**: 数据级过滤条件（Filter 对象）
- **validation**: 数据验证规则
- **presets**: 字段预设值

---

## 2. 数据读写限制机制

### 2.1 读取操作的权限控制

#### 2.1.1 流程概览

读取操作的权限控制发生在 **ItemsService.readByQuery** 方法中，位于 `api/src/services/items.ts:499-580`，核心流程：

```
1. 查询预处理（emitFilter）
2. 构建 AST（getAstFromQuery）
3. 权限注入（processAst）
4. 执行查询（runAst）
5. 结果返回
```

#### 2.1.2 processAst 深度解析

`processAst` 函数（`api/src/permissions/modules/process-ast/process-ast.ts`）是读取权限的核心：

**执行步骤：**

1. **提取字段映射**：从 AST 中提取所有路径对应的集合和字段
   ```typescript
   const fieldMap: FieldMap = fieldMapFromAst(options.ast, context.schema);
   const collections = collectionsInFieldMap(fieldMap);
   ```

2. **获取权限**：
   ```typescript
   const policies = await fetchPolicies(options.accountability, context);
   const permissions = await fetchPermissions(
     { action: options.action, policies, collections, accountability: options.accountability },
     context,
   );
   ```

3. **验证路径存在性**：检查请求的字段是否实际存在于 Schema 中
   ```typescript
   validatePathExistence(path, collection, fields, context.schema);
   ```

4. **验证路径权限**：
   - 对于写操作字段：使用当前操作的权限验证
   - 对于读操作字段：使用 `read` 权限验证

5. **注入权限案例**：
   ```typescript
   injectCases(options.ast, permissions);
   ```

#### 2.1.3 validatePathPermissions 字段验证

位于 `api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts`：

```typescript
// 1. 过滤当前集合的权限
const permissionsForCollection = permissions.filter((permission) => permission.collection === collection);

if (permissionsForCollection.length === 0) {
  throw createCollectionForbiddenError(path, collection);
}

// 2. 收集所有允许的字段
const allowedFields: Set<string> = new Set();

for (const { fields } of permissionsForCollection) {
  if (!fields) continue;
  for (const field of fields) {
    if (field === '*') return; // 通配符，所有字段都允许
    allowedFields.add(field);
  }
}

// 3. 检查请求的字段是否都在允许列表中
const forbiddenFields = requestedFields.filter((field) => allowedFields.has(field) === false);
```

### 2.2 写入操作的权限控制

#### 2.2.1 processPayload 载荷处理

写入操作（create/update）使用 `processPayload`（`api/src/permissions/modules/process-payload/process-payload.ts`）：

**核心逻辑：**

1. **检查集合权限**：验证用户对该集合是否有操作权限
   ```typescript
   if (permissions.length === 0) {
     throw createCollectionForbiddenError('', options.collection);
   }
   ```

2. **检查字段权限**：验证请求的字段是否都允许操作
   ```typescript
   const fieldsAllowed = uniq(permissions.map(({ fields }) => fields ?? []).flat());
   
   if (fieldsAllowed.includes('*') === false) {
     const fieldsUsed = Object.keys(options.payload);
     const notAllowed = difference(fieldsUsed, fieldsAllowed);
     
     if (notAllowed.length > 0) {
       throw createFieldsForbiddenError('', options.collection, notAllowed);
     }
   }
   ```

3. **应用预设值**：
   ```typescript
   const presets = (permissions ?? []).map((permission) => permission.presets);
   const payloadWithPresets = assign({}, ...presets, options.payload);
   ```

4. **验证数据**：合并字段级验证规则和权限级验证规则
   ```typescript
   const validationRules = [...fieldValidationRules, ...permissionValidationRules];
   ```

#### 2.2.2 validateAccess 访问验证

对于 update 和 delete 操作，还需要通过 `validateAccess` 验证具体数据项的权限：

**流程（`api/src/permissions/modules/validate-access/validate-access.ts:22-57`）：**

```typescript
export async function validateAccess(options: ValidateAccessOptions, context: Context) {
  // 1. 管理员绕过所有检查
  if (options.accountability.admin === true) {
    return;
  }

  // 2. 有主键时验证具体项目，否则验证集合访问
  if (options.primaryKeys) {
    const result = await validateItemAccess(options as Required<ValidateAccessOptions>, context);
    access = result.accessAllowed;
  } else {
    access = await validateCollectionAccess(options, context);
  }

  // 3. 权限不足时抛出异常
  if (!access) {
    throw new ForbiddenError({ reason: ... });
  }
}
```

**validateItemAccess 深度解析**（`api/src/permissions/modules/validate-access/lib/validate-item-access.ts`）：

该方法通过构建一个最小化的查询 AST，实际执行查询来验证权限：

```typescript
// 1. 构建测试 AST
const ast: AST = {
  type: 'root',
  name: options.collection,
  query: { limit: options.primaryKeys!.length },
  children: options.fields?.map((field) => ({ ... })) ?? [],
  cases: [],
};

// 2. 注入权限规则
await processAst({ ast, ...options }, context);

// 3. 过滤特定主键
ast.query.filter = { [primaryKeyField]: { _in: options.primaryKeys! } };

// 4. 实际执行查询，检查返回数量
const items = await fetchPermittedAstRootFields(ast, { ... });
const expectedCount = options.primaryKeys!.length;
const hasAccess = items && items.length === expectedCount;
```

### 2.3 injectCases 权限注入机制

`injectCases`（`api/src/permissions/modules/process-ast/lib/inject-cases.ts`）是将权限规则注入查询的关键：

**工作原理：**

1. **获取权限案例**：
   ```typescript
   const { cases, caseMap, allowedFields } = getCases(collection, permissions, requestedKeys);
   ```

2. **为每个字段附加 whenCase 条件**：
   ```typescript
   // 当没有全局通配符权限且字段不在允许列表中时
   if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
     child.whenCase = [...(globalWhenCase ?? []), ...(fieldWhenCase ?? [])];
   }
   ```

3. **递归处理关联字段**：
   - `m2o`（多对一）
   - `o2m`（一对多）
   - `a2o`（任对一）
   - `functionField`（函数字段）

**案例注入的效果：**

- 权限规则被转换为 SQL 查询的 `CASE WHEN` 条件
- 无权限访问的字段返回 `NULL`
- 数据级过滤条件被注入到 WHERE 子句

### 2.4 权限拒绝语义分析

Directus 权限系统中有两种不同的权限拒绝语义，对应不同的场景和行为：

#### 2.4.1 显式请求未授权字段：报错行为

**触发条件**：用户请求的字段不在任何权限规则的 `fields` 列表中，且没有 `*` 通配符权限。

**代码位置**：`api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts:4-44`

**核心逻辑**：

```typescript
// 1. 收集所有允许的字段
const allowedFields: Set<string> = new Set();

for (const { fields } of permissionsForCollection) {
  if (!fields) continue;
  for (const field of fields) {
    if (field === '*') return; // 通配符，所有字段都允许，直接返回
    allowedFields.add(field);
  }
}

// 2. 检查请求的字段是否都在允许列表中
const requestedFields = Array.from(fields);
const forbiddenFields = requestedFields.filter((field) => allowedFields.has(field) === false);

if (forbiddenFields.length > 0) {
  throw createFieldsForbiddenError(path, collection, forbiddenFields);
}
```

**错误信息**：

```
You don't have permission to access field "sensitive_field" in collection "articles" or it does not exist.
Queried in "root".
```

**写入操作的相同行为**：

`processPayload`（`api/src/permissions/modules/process-payload/process-payload.ts:29-142`）中也有相同的检查：

```typescript
if (fieldsAllowed.includes('*') === false) {
  const fieldsUsed = Object.keys(options.payload);
  const notAllowed = difference(fieldsUsed, fieldsAllowed);
  
  if (notAllowed.length > 0) {
    throw createFieldsForbiddenError('', options.collection, notAllowed);
  }
}
```

**REST 与 GraphQL 一致性**：

| 接口类型 | 行为 | 原因 |
|---------|------|------|
| **REST** | 抛出 `ForbiddenError` (403) | 直接通过 `processAst` 或 `processPayload` 验证 |
| **GraphQL** | 抛出 `ForbiddenError`（作为 GraphQL 错误返回） | Schema 层不会过滤这种情况，执行时统一验证 |

> **注意**：GraphQL 的 `reduceSchema` 会过滤无权限的集合和字段，但这里讨论的是「有集合权限但无特定字段权限」的情况，这种情况下字段仍会出现在 Schema 中（因为其他规则可能允许），但执行时会被 `validatePathPermissions` 拦截。

#### 2.4.2 条件不满足时的字段可见性变化：返回 NULL

**触发条件**：用户请求的字段在 `fields` 列表中，但数据级过滤条件（`permissions` 属性）不满足。

**代码位置**：
- `api/src/permissions/modules/process-ast/lib/inject-cases.ts:13-72`
- `api/src/database/run-ast/utils/apply-case-when.ts:21-59`

**核心逻辑**：

**第一步：识别需要条件保护的字段**（`injectCases`）

```typescript
// 从权限规则中提取案例和允许字段
const { cases, caseMap, allowedFields } = getCases(collection, permissions, requestedKeys);

// allowedFields = 来自无数据过滤条件的权限规则的字段
// 这些字段在任何情况下都可见

// 为每个字段附加 whenCase 条件
for (const child of children) {
  const fieldKey = getUnaliasedFieldKey(child);
  const globalWhenCase = caseMap['*'];
  const fieldWhenCase = caseMap[fieldKey];

  // 只有当字段不在「无条件允许」列表中时，才需要 CASE WHEN 保护
  if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
    child.whenCase = [...(globalWhenCase ?? []), ...(fieldWhenCase ?? [])];
  }
}
```

**第二步：生成 CASE WHEN SQL**（`applyCaseWhen`）

```typescript
export function applyCaseWhen({ columnCases, ... }: ApplyCaseWhenOptions, ...): Knex.Raw {
  // 应用权限过滤条件到查询的 WHERE 子句
  applyFilter(knex, schema, caseQuery, { _or: columnCases }, table, aliasMap, cases, permissions);

  // 生成 CASE WHEN 语句
  // 当条件满足时返回实际值，否则返回 NULL
  const rawCase = `(CASE WHEN ${sql} THEN ?? END)`;
  // 相当于：CASE WHEN <conditions> THEN <column> ELSE NULL END

  return knex.raw(rawCase, bindings);
}
```

**getCases 的工作原理**（`api/src/permissions/modules/process-ast/lib/get-cases.ts:6-57`）：

```typescript
export function getCases(collection: string, permissions: Permission[], requestedKeys: string[]) {
  const permissionsForCollection = permissions.filter((p) => p.collection === collection);
  const rules = dedupeAccess(permissionsForCollection);

  // rules = [{ rule: Filter, fields: Set<string> }, ...]
  // 按数据过滤条件分组，相同条件的字段合并

  const cases: Filter[] = [];      // 所有去重后的过滤条件
  const caseMap: Record<FieldKey, number[]> = {};  // 字段 → 案例索引映射

  for (const { rule, fields } of rules) {
    if (rule === null) continue;  // 无数据过滤条件的规则不生成案例

    cases.push(rule);
    const index = cases.length - 1;

    for (const field of fields) {
      caseMap[field] = [...(caseMap[field] ?? []), index];
    }
  }

  // allowedFields = 来自无数据过滤条件的权限规则的字段
  const allowedFields = new Set(
    permissionsForCollection
      .filter((permission) => hasItemPermissions(permission) === false)
      .map((permission) => permission.fields ?? [])
      .flat(),
  );

  return { cases, caseMap, allowedFields };
}
```

**hasItemPermissions 判断**（`api/src/permissions/modules/process-ast/utils/has-item-permissions.ts:3-5`）：

```typescript
export function hasItemPermissions(permission: Permission) {
  return permission.permissions !== null && Object.keys(permission.permissions).length > 0;
}
```

#### 2.4.3 两种拒绝语义的对比

| 维度 | 显式请求未授权字段 | 条件不满足 |
|------|------------------|-----------|
| **触发原因** | 字段不在 `fields` 列表中，无任何权限规则允许 | 字段在 `fields` 列表中，但 `permissions` 条件不满足 |
| **验证时机** | `validatePathPermissions`（查询执行前） | SQL 执行时（CASE WHEN） |
| **行为** | 抛出 `ForbiddenError` | 返回 `NULL` |
| **HTTP 状态码** | 403 Forbidden | 200 OK（数据中包含 NULL） |
| **是否可见** | 不可见（请求被拒绝） | 可见但值为 NULL |
| **是否可区分** | 可以明确知道「无权限」 | 无法区分「无权限」和「值本身就是 NULL」 |

#### 2.4.4 实际示例

**场景设定**：

`articles` 集合有字段：`id`, `title`, `status`, `author`, `secret_notes`

用户权限规则：

```json
{
  "collection": "articles",
  "action": "read",
  "fields": ["title", "status", "author"],
  "permissions": { "status": { "_eq": "published" } }
}
```

**示例 1：请求未授权字段**

```
GET /items/articles?fields=title,secret_notes
```

**结果**：

```json
{
  "errors": [
    {
      "message": "You don't have permission to access field \"secret_notes\" in collection \"articles\" or it does not exist.",
      "extensions": { "code": "FORBIDDEN" }
    }
  ]
}
```

**示例 2：请求授权字段但部分记录不满足条件**

数据库中的记录：

| id | title | status | author |
|----|-------|--------|--------|
| 1 | Article A | published | user1 |
| 2 | Article B | draft | user2 |

查询：

```
GET /items/articles?fields=title,status
```

**结果**：

```json
{
  "data": [
    {
      "id": 1,
      "title": "Article A",
      "status": "published"
    },
    {
      "id": 2,
      "title": null,
      "status": null
    }
  ]
}
```

> **说明**：
> - 记录 2（status=draft）不满足权限条件，但记录仍被返回
> - 记录 2 的 `title` 和 `status` 字段被设置为 `NULL`
> - 这种「静默降级」行为是为了保护隐私，防止通过错误信息推断数据存在性

---

## 3. 字段级权限与集合级权限的叠加顺序

### 3.1 权限层次结构

Directus 权限系统具有清晰的层次结构：

```
┌─────────────────────────────────────────────────────┐
│                    策略（Policy）                     │
│  优先级：父角色 → 子角色 → 用户                         │
└─────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────┐
│              权限规则（Permission）                    │
│  collection + action 组合，每个规则包含：               │
│  - 集合级：collection、permissions（数据过滤）          │
│  - 字段级：fields                                     │
└─────────────────────────────────────────────────────┘
                            ↓
┌─────────────────────────────────────────────────────┐
│              权限合并（Merge Permissions）             │
│  策略：and / or / intersection                       │
└─────────────────────────────────────────────────────┘
```

### 3.2 集合级权限与字段级权限的关系

**权限规则的结构：**

```typescript
interface Permission {
  collection: string;           // 集合级：适用的集合
  action: PermissionsAction;    // 集合级：适用的操作
  permissions?: Filter;         // 集合级：数据过滤条件
  validation?: Filter;          // 集合级：数据验证条件
  fields?: string[];            // 字段级：允许的字段列表
  presets?: Record<string, any>; // 字段级：字段预设值
}
```

**关键点：**

1. **权限规则以 collection + action 为单元**
   - 一条权限规则只能关联一个集合和一个操作
   - 同一集合+操作可以有多条权限规则（来自不同策略）

2. **字段级权限是权限规则的属性**
   - `fields` 数组定义该规则下允许的字段
   - `fields: null` 或空数组表示无字段权限
   - `fields: ['*']` 表示所有字段都允许

### 3.3 权限合并策略

#### 3.3.1 mergePermissions 函数分析

位于 `api/src/permissions/utils/merge-permissions.ts:12-51`：

```typescript
export function mergePermissions(
  strategy: 'and' | 'or' | 'intersection',
  ...permissions: Permission[][]
): Permission[] {
  let allPermissions;

  // intersection 策略：只保留所有列表共有的权限
  if (strategy === 'intersection') {
    const permissionKeys = permissions.map((permissions) => {
      return new Set(permissions.map((permission) => `${permission.collection}__${permission.action}`));
    });

    const intersectionKeys = permissionKeys.reduce((acc, val) => {
      return new Set([...acc].filter((x) => val.has(x)));
    }, permissionKeys[0]!);

    // 先对每个列表进行 'or' 合并
    const deduplicateSubpermissions = permissions.map((permissions) => {
      return mergePermissions('or', permissions);
    });

    // 过滤出共同的权限
    allPermissions = flatten(deduplicateSubpermissions).filter((permission) => {
      return intersectionKeys.has(`${permission.collection}__${permission.action}`);
    });

    strategy = 'and'; // 后续使用 'and' 合并
  } else {
    allPermissions = flatten(permissions);
  }

  // 按 collection__action 分组合并
  const mergedPermissions = allPermissions
    .reduce((acc, val) => {
      const key = `${val.collection}__${val.action}`;
      const current = acc.get(key);
      acc.set(key, current ? mergePermission(strategy, current, val) : val);
      return acc;
    }, new Map())
    .values();

  return Array.from(mergedPermissions);
}
```

#### 3.3.2 三种合并策略对比

| 策略 | 行为 | 使用场景 |
|------|------|----------|
| **or** | 权限取并集，字段取并集，条件取 OR | 同一策略内的多条规则 |
| **and** | 权限取交集，字段取交集，条件取 AND | 不同策略之间的权限组合 |
| **intersection** | 仅保留所有列表共有的权限，然后 AND | 需要严格限制时使用 |

### 3.4 mergePermission 单条权限合并

位于 `api/src/permissions/utils/merge-permissions.ts:53-126`：

#### 3.4.1 数据过滤条件合并（permissions 属性）

```typescript
let { permissions, validation, fields, presets } = currentPerm;

if (newPerm.permissions) {
  if (currentPerm.permissions && Object.keys(currentPerm.permissions)[0] === logicalKey) {
    // 已有相同逻辑运算符，追加到数组
    permissions = {
      [logicalKey]: [
        ...(currentPerm.permissions as LogicalFilterOR & LogicalFilterAND)[logicalKey],
        newPerm.permissions,
      ],
    };
  } else if (currentPerm.permissions) {
    // 创建新的逻辑运算符包裹
    if (strategy === 'or' && (isEqual(currentPerm.permissions, {}) || isEqual(newPerm.permissions, {}))) {
      permissions = {}; // OR 策略下，空条件（无限制）优先级最高
    } else {
      permissions = {
        [logicalKey]: [currentPerm.permissions, newPerm.permissions],
      };
    }
  } else {
    // 当前无条件，直接使用新条件
    permissions = {
      [logicalKey]: [newPerm.permissions],
    };
  }
}
```

#### 3.4.2 字段权限合并（fields 属性）

调用 `mergeFields` 函数（`api/src/permissions/utils/merge-fields.ts:3-27`）：

```typescript
export function mergeFields(fieldsA: string[] | null, fieldsB: string[] | null, strategy: 'and' | 'or') {
  if (fieldsA === null) fieldsA = [];
  if (fieldsB === null) fieldsB = [];

  let fields = [];

  if (strategy === 'and') {
    // AND 策略：取交集
    if (fieldsA.length === 0 || fieldsB.length === 0) return []; // 任一为空则结果为空
    if (fieldsA.includes('*')) return fieldsB; // A 为全权限，使用 B 的限制
    if (fieldsB.includes('*')) return fieldsA; // B 为全权限，使用 A 的限制
    fields = intersection(fieldsA, fieldsB);
  } else {
    // OR 策略：取并集
    if (fieldsA.length === 0) return fieldsB;
    if (fieldsB.length === 0) return fieldsA;
    if (fieldsA.includes('*') || fieldsB.includes('*')) return ['*']; // 任一为全权限则结果为全权限
    fields = union(fieldsA, fieldsB);
  }

  if (fields.includes('*')) return ['*'];
  return fields;
}
```

### 3.5 叠加顺序实例分析

#### 场景 1：同一集合，多策略权限（AND 合并）

**策略 A 权限：**
```json
{
  "collection": "articles",
  "action": "read",
  "fields": ["title", "content", "author"],
  "permissions": { "status": { "_eq": "published" } }
}
```

**策略 B 权限：**
```json
{
  "collection": "articles",
  "action": "read",
  "fields": ["title", "author", "created_at"],
  "permissions": { "author": { "_eq": "$CURRENT_USER" } }
}
```

**合并结果（AND 策略）：**
```json
{
  "collection": "articles",
  "action": "read",
  "fields": ["title", "author"],  // 字段交集
  "permissions": {
    "_and": [
      { "status": { "_eq": "published" } },
      { "author": { "_eq": "$CURRENT_USER" } }
    ]
  }
}
```

**说明：**
- **字段**：`intersection(['title','content','author'], ['title','author','created_at'])` → `['title','author']`
- **数据过滤**：两个条件都必须满足

#### 场景 2：同一策略，多条规则（OR 合并）

**规则 1：**
```json
{
  "collection": "articles",
  "action": "read",
  "fields": ["title", "status"],
  "permissions": { "category": { "_eq": "public" } }
}
```

**规则 2：**
```json
{
  "collection": "articles",
  "action": "read",
  "fields": ["title", "content", "author"],
  "permissions": { "author": { "_eq": "$CURRENT_USER" } }
}
```

**合并结果（OR 策略）：**
```json
{
  "collection": "articles",
  "action": "read",
  "fields": ["title", "status", "content", "author"],  // 字段并集
  "permissions": {
    "_or": [
      { "category": { "_eq": "public" } },
      { "author": { "_eq": "$CURRENT_USER" } }
    ]
  }
}
```

**说明：**
- **字段**：`union(['title','status'], ['title','content','author'])` → `['title','status','content','author']`
- **数据过滤**：任一条件满足即可

### 3.6 权限执行的实际顺序

在实际执行时，权限检查遵循以下顺序：

```
阶段 1：集合级访问检查
    ↓
检查是否有该 collection + action 的权限规则
    ↓
    ├─ 无规则 → 抛出 ForbiddenError
    └─ 有规则 → 继续

阶段 2：字段级权限检查
    ↓
收集所有规则的允许字段
    ↓
    ├─ 含 '*' → 所有字段允许
    └─ 无 '*' → 检查请求字段是否在允许列表中

阶段 3：数据级权限注入
    ↓
将 permissions 过滤条件注入查询 AST
    ↓
    ├─ 读取 → CASE WHEN + WHERE
    └─ 写入 → 验证数据是否满足条件
```

---

## 4. GraphQL 与 REST 共用权限核心的机制

### 5.1 架构概览

Directus 采用**统一服务层**设计，REST 和 GraphQL 只是不同的「入口」，最终都调用相同的服务层和权限核心：

```
┌─────────────────┐     ┌─────────────────┐
│   REST API      │     │   GraphQL API   │
│  (controllers)  │     │  (controllers)  │
└────────┬────────┘     └────────┬────────┘
         │                        │
         └──────────┬─────────────┘
                    ↓
         ┌──────────────────────────┐
         │     ItemsService         │
         │  (api/src/services/)     │
         │  ├─ createOne/Many       │
         │  ├─ readByQuery/One      │
         │  ├─ updateOne/Many       │
         │  └─ deleteOne/Many       │
         └──────────┬───────────────┘
                    ↓
         ┌──────────────────────────┐
         │   权限核心模块            │
         │  (api/src/permissions/)  │
         │  ├─ processAst           │
         │  ├─ validateAccess       │
         │  ├─ processPayload       │
         │  └─ fetchPermissions     │
         └──────────────────────────┘
```

### 4.2 REST API 的权限使用

#### 4.2.1 控制器层

REST 控制器（如 `api/src/controllers/items.ts`）非常薄，主要负责：

1. 路由参数解析
2. 服务实例化（传入 accountability）
3. 调用服务方法
4. 响应格式化

**示例：读取操作**

```typescript
const readHandler = asyncHandler(async (req, res, next) => {
  const service = new ItemsService(req.collection, {
    accountability: req.accountability,  // 关键：传入用户身份
    schema: req.schema,
  });

  const result = await service.readByQuery(req.sanitizedQuery);
  res.locals['payload'] = { data: result };
  return next();
});
```

#### 4.2.2 服务层中的权限调用

**ItemsService.readByQuery**（`api/src/services/items.ts:499-580`）：

```typescript
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
  // 1. 构建 AST
  let ast = await getAstFromQuery(
    {
      collection: this.collection,
      query: updatedQuery,
      accountability: this.accountability,
    },
    { schema: this.schema, knex: this.knex },
  );

  // 2. 权限处理核心调用
  ast = await processAst(
    { ast, action: 'read', accountability: this.accountability },
    { knex: this.knex, schema: this.schema },
  );

  // 3. 执行查询
  const records = await runAst(ast, this.schema, this.accountability, { ... });
  
  return records;
}
```

**ItemsService.updateOne**（`api/src/services/items.ts:647-790`）中的权限检查：

```typescript
async updateOne(key: PrimaryKey, data: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey> {
  // ... 预处理 ...
  
  if (this.accountability) {
    // 1. 验证访问权限
    await validateAccess(
      {
        accountability: this.accountability,
        action: 'update',
        collection: this.collection,
        primaryKeys: keys,
      },
      { schema: this.schema, knex: trx },
    );
  }

  // 2. 处理 payload（字段权限、验证、预设）
  const payloadWithPresets = this.accountability
    ? await processPayload(
        {
          accountability: this.accountability,
          action: 'update',
          collection: this.collection,
          payload: payloadAfterHooks,
          nested: this.nested,
        },
        { knex: trx, schema: this.schema },
      )
    : payloadAfterHooks;

  // ... 执行更新 ...
}
```

### 4.3 GraphQL API 的权限使用

#### 4.3.1 两层权限机制

GraphQL API 有两层权限保护：

**第一层：Schema 生成时的权限过滤**

位于 `api/src/services/graphql/schema/index.ts:57-129`：

```typescript
export async function generateSchema(gql: GraphQLService, type: 'schema' | 'sdl' = 'schema') {
  // 管理员使用完整 Schema
  if (!gql.accountability || gql.accountability.admin) {
    schema = {
      read: sanitizedSchema,
      create: sanitizedSchema,
      update: sanitizedSchema,
      delete: sanitizedSchema,
    };
  } else {
    // 非管理员：根据权限缩减 Schema
    schema = {
      read: reduceSchema(
        sanitizedSchema,
        await fetchAllowedFieldMap(
          { accountability: gql.accountability, action: 'read' },
          { schema: gql.schema, knex: gql.knex },
        ),
      ),
      // create/update/delete 同理...
    };
  }
}
```

**reduceSchema** 的作用（`api/src/utils/reduce-schema.ts:8-90`）：

```typescript
export function reduceSchema(schema: SchemaOverview, fieldMap: FieldMap): SchemaOverview {
  const reduced: SchemaOverview = {
    collections: {},
    relations: [],
  };

  // 1. 过滤集合：只保留有字段权限的集合
  for (const [collectionName, collection] of Object.entries(schema.collections)) {
    if (!fieldMap[collectionName]) continue;

    // 2. 过滤字段：只保留允许的字段
    for (const [fieldName, field] of Object.entries(collection.fields)) {
      if (!fieldMap[collectionName]?.includes('*') && 
          !fieldMap[collectionName]?.includes(fieldName)) {
        continue;
      }
      fields[fieldName] = field;
    }

    reduced.collections[collectionName] = { ...collection, fields };
  }

  // 3. 过滤关系：只保留两端集合都有权限的关系
  reduced.relations = schema.relations.filter((relation) => {
    let collectionsAllowed = true;
    let fieldsAllowed = true;
    // ... 检查关系两端的权限 ...
    return collectionsAllowed && fieldsAllowed;
  });

  return reduced;
}
```

**第二层：查询执行时的权限验证**

GraphQL 查询执行时，通过 Resolver 调用 ItemsService，复用 REST 相同的权限逻辑：

**GraphQLService**（`api/src/services/graphql/index.ts:100-144`）：

```typescript
// 读取操作
async read(collection: string, query: Query, id?: PrimaryKey): Promise<Partial<Item>> {
  const service = getService(collection, {
    knex: this.knex,
    accountability: this.accountability,  // 传入用户身份
    schema: this.schema,
  });

  if (id) return await service.readOne(id, query, { stripNonRequested: false });
  return await service.readByQuery(query, { stripNonRequested: false });
}

// 写入操作
async upsertSingleton(
  collection: string,
  body: Record<string, any> | Record<string, any>[],
  query: Query,
): Promise<Partial<Item> | boolean> {
  const service = getService(collection, {
    knex: this.knex,
    accountability: this.accountability,
    schema: this.schema,
  });

  await service.upsertSingleton(body);
  // ...
}
```

### 4.4 共用权限核心的实现细节

#### 4.4.1 统一的 Accountability 对象

权限核心使用统一的 `Accountability` 接口，无论请求来自 REST 还是 GraphQL：

```typescript
interface Accountability {
  user: string | null;           // 用户 ID
  role: string | null;           // 当前角色 ID
  roles: string[];               // 所有角色（含继承）
  admin: boolean;                // 是否管理员（绕过所有检查）
  app: boolean;                  // 是否来自 App
  ip: string;                    // IP 地址（用于策略过滤）
  share?: string;                // 共享令牌（可选）
}
```

#### 4.4.2 权限核心模块的独立性

权限核心模块不依赖任何 API 层代码，可以被任意入口调用：

```
permissions/
├── lib/                    # 权限获取层（纯业务）
│   ├── fetch-policies.ts
│   ├── fetch-permissions.ts
│   └── fetch-roles-tree.ts
├── modules/                # 权限处理模块（纯业务）
│   ├── fetch-allowed-fields/
│   ├── fetch-allowed-collections/
│   ├── process-ast/
│   ├── process-payload/
│   └── validate-access/
├── utils/                  # 工具函数（纯逻辑）
│   ├── merge-permissions.ts
│   ├── merge-fields.ts
│   └── ...
└── types.ts               # 类型定义
```

#### 4.4.3 AST 作为统一的数据表示

无论是 REST 的 Query 对象还是 GraphQL 的 Document，最终都被转换为内部的 **AST** 格式：

```typescript
interface AST {
  type: 'root' | 'field' | 'm2o' | 'o2m' | ...;
  name: string;           // 集合或字段名
  query: Query;           // 查询条件
  children: AST[];        // 子节点（关联或嵌套字段）
  cases: Case[];          // 权限案例（由 injectCases 注入）
  whenCase?: WhenCase[];  // 字段级权限条件
}
```

**转换流程：**

```
REST 查询参数
    ↓
getAstFromQuery()
    ↓
统一 AST 格式
    ↓
processAst()  ← 权限核心处理
    ↓
runAst() → SQL 查询
```

```
GraphQL 查询 Document
    ↓
GraphQL Resolver
    ↓
ItemsService.readByQuery()
    ↓
getAstFromQuery()
    ↓
统一 AST 格式
    ↓
processAst()  ← 权限核心处理
    ↓
runAst() → SQL 查询
```

### 5.5 权限调用流程图

#### 5.5.1 REST 读取流程

```
GET /items/articles?fields=title,author&filter[status][_eq]=published
    ↓
ItemsController.readHandler
    ↓
new ItemsService('articles', { accountability, schema })
    ↓
service.readByQuery(query)
    ↓
├─ getAstFromQuery() → 构建 AST
├─ processAst()      → 权限检查 + 注入
│   ├─ fetchPolicies()
│   ├─ fetchPermissions()
│   ├─ validatePathExistence()
│   ├─ validatePathPermissions()
│   └─ injectCases()
└─ runAst() → 执行 SQL
    ↓
返回结果（已过滤无权限字段）
```

#### 4.5.2 GraphQL 读取流程

```
POST /graphql
query { articles(filter: {status: {_eq: published}}) { title author } }
    ↓
GraphQLController
    ↓
new GraphQLService({ accountability, schema, scope: 'items' })
    ↓
service.execute(graphqlParams)
    ↓
├─ getSchema()
│   └─ generateSchema()
│       ├─ fetchAllowedFieldMap(action: 'read')
│       └─ reduceSchema() → 生成权限过滤后的 GraphQL Schema
└─ execute()
    ↓
GraphQL Resolver → service.read('articles', query)
    ↓
getService('articles') → ItemsService
    ↓
service.readByQuery(query)
    ↓
├─ getAstFromQuery()
├─ processAst()  ← 与 REST 完全相同
└─ runAst()
    ↓
返回结果
```

#### 4.5.3 REST 更新流程

```
PATCH /items/articles/123
{ "status": "archived" }
    ↓
ItemsController
    ↓
ItemsService.updateOne(123, data)
    ↓
├─ validateAccess()
│   ├─ fetchPolicies()
│   ├─ fetchPermissions()
│   └─ validateItemAccess() → 实际查询验证
├─ processPayload()
│   ├─ 检查字段权限
│   ├─ 应用预设值
│   └─ 验证数据
└─ 执行 UPDATE SQL
    ↓
返回更新结果
```

### 4.6 关键代码路径汇总

#### 4.6.1 读取权限（REST & GraphQL 共用）

| 层级 | 文件路径 | 关键函数 |
|------|----------|----------|
| 服务层 | `api/src/services/items.ts:499` | `readByQuery()` |
| AST 构建 | `api/src/database/get-ast-from-query/get-ast-from-query.ts:23` | `getAstFromQuery()` |
| 权限处理 | `api/src/permissions/modules/process-ast/process-ast.ts:19` | `processAst()` |
| 策略获取 | `api/src/permissions/lib/fetch-policies.ts:16` | `fetchPolicies()` |
| 权限获取 | `api/src/permissions/lib/fetch-permissions.ts:17` | `fetchPermissions()` |
| 路径验证 | `api/src/permissions/modules/process-ast/utils/validate-path/validate-path-permissions.ts:4` | `validatePathPermissions()` |
| 权限注入 | `api/src/permissions/modules/process-ast/lib/inject-cases.ts:13` | `injectCases()` |

#### 4.6.2 写入权限（REST & GraphQL 共用）

| 层级 | 文件路径 | 关键函数 |
|------|----------|----------|
| 服务层 | `api/src/services/items.ts:647` | `updateOne()` |
| 访问验证 | `api/src/permissions/modules/validate-access/validate-access.ts:22` | `validateAccess()` |
| 项目验证 | `api/src/permissions/modules/validate-access/lib/validate-item-access.ts:44` | `validateItemAccess()` |
| 载荷处理 | `api/src/permissions/modules/process-payload/process-payload.ts:29` | `processPayload()` |

#### 5.6.3 GraphQL Schema 权限

| 层级 | 文件路径 | 关键函数 |
|------|----------|----------|
| Schema 生成 | `api/src/services/graphql/schema/index.ts:57` | `generateSchema()` |
| 字段映射获取 | `api/src/permissions/modules/fetch-allowed-field-map/fetch-allowed-field-map.ts:14` | `fetchAllowedFieldMap()` |
| Schema 缩减 | `api/src/utils/reduce-schema.ts:8` | `reduceSchema()` |

---

## 附录：核心数据结构

### A.1 Permission 接口

```typescript
interface Permission {
  id: string;
  policy: string;
  collection: string;
  action: 'read' | 'create' | 'update' | 'delete';
  permissions?: Filter;      // 数据过滤条件
  validation?: Filter;       // 数据验证条件
  fields?: string[];         // 允许的字段，null 表示无权限，[] 同 null
  presets?: Record<string, any>;  // 字段预设值
  system?: boolean;
}
```

### A.2 Context 接口

```typescript
interface Context {
  schema: SchemaOverview;    // 数据库 Schema
  knex: Knex;                // 数据库连接
  accountability?: Accountability;  // 用户身份
}
```

### A.3 权限合并策略矩阵

| 条件 | 策略 A 字段 | 策略 B 字段 | AND 结果 | OR 结果 |
|------|------------|------------|---------|--------|
| 任一为空 | `['a']` | `[]` | `[]` | `['a']` |
| 无通配符 | `['a','b']` | `['b','c']` | `['b']` | `['a','b','c']` |
| A 有通配符 | `['*']` | `['a','b']` | `['a','b']` | `['*']` |
| B 有通配符 | `['a','b']` | `['*']` | `['a','b']` | `['*']` |
| 都有通配符 | `['*']` | `['*']` | `['*']` | `['*']` |
