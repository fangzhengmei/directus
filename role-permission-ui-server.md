# Directus 角色权限系统链路分析

本文档详细分析 Directus 中管理员配置角色、字段权限和过滤规则后，服务端如何解析并在每次请求中强制执行的完整链路。

---

## 一、整体架构概览

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                    Admin UI (Vue 3)                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────────────────┐  │
│  │ 角色管理页面  │  │ 策略管理页面  │  │         权限规则配置组件                  │  │
│  │ (item.vue)  │  │              │  │ (字段选择、过滤规则构建器、预设值设置)    │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼ HTTP API (REST/GraphQL)
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                API Controllers                                         │
│  ┌─────────────────────────────────────┐  ┌──────────────────────────────────────┐  │
│  │ permissions.ts (权限 CRUD)           │  │ roles.ts (角色 CRUD)                 │  │
│  │ - POST /permissions                  │  │ - POST /roles                        │  │
│  │ - GET /permissions                   │  │ - GET /roles/:id                     │  │
│  │ - PATCH /permissions                 │  │ - PATCH /roles/:id                   │  │
│  │ - DELETE /permissions                │  │ - DELETE /roles/:id                  │  │
│  └─────────────────────────────────────┘  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 Services Layer                                         │
│  ┌───────────────────────────────┐  ┌──────────────────────────────────────────────┐ │
│  │ PermissionsService            │  │ RolesService                                │ │
│  │ (api/src/services/permissions.ts)│  │ (api/src/services/roles.ts)               │ │
│  │ - 操作 directus_permissions 表 │  │ - 操作 directus_roles 表                   │ │
│  │ - 权限变更时清除系统缓存        │  │ - 角色删除时级联清理权限/预设              │ │
│  └───────────────────────────────┘  └──────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                              数据库层 (PostgreSQL/MySQL/SQLite 等)                   │
│                                                                                        │
│  ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────────────┐  │
│  │ directus_roles   │      │ directus_policies│      │ directus_permissions     │  │
│  │ (角色表)          │◄────►│ (策略表)          │◄────►│ (权限规则表)             │  │
│  │ - id (PK)        │      │ - id (PK)        │      │ - id (PK)                │  │
│  │ - name           │      │ - name           │      │ - policy (FK)            │  │
│  │ - icon           │      │ - admin_access   │      │ - collection              │  │
│  │ - description    │      │ - app_access     │      │ - action                  │  │
│  │ - parent (FK)    │      │ - ip_access      │      │ - permissions (JSON)     │  │
│  └──────────────────┘      │ - enforce_tfa    │      │ - validation (JSON)      │  │
│         ▲                   └──────────────────┘      │ - presets (JSON)         │  │
│         │                                              │ - fields (CSV)            │  │
│         │                    ┌──────────────────┐      └──────────────────────────┘  │
│         └────────────────────│ directus_access  │◄─────────────────────────────────┘  │
│                              │ (关联表)          │                                     │
│                              │ - role (FK)      │                                     │
│                              │ - user (FK)      │                                     │
│                              │ - policy (FK)    │                                     │
│                              └──────────────────┘                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、权限配置链路（UI → Server → Database）

### 2.1 前端 UI 配置层

**关键文件位置：**
- `app/src/modules/settings/routes/roles/item.vue` - 角色编辑主页面
- `app/src/modules/settings/routes/roles/collection.vue` - 角色列表页面

#### 角色配置流程

```typescript
// item.vue 中使用 useItem composable 管理角色数据
const { edits, hasEdits, item, save, remove } = useItem<Role>(
  ref('directus_roles'),  // 操作的集合
  primaryKey,              // 角色主键
  null,
  { deep: { users: { _limit: 0 } } }
);

// 保存角色时的流程
async function saveAndQuit() {
  await save();                              // 1. 保存角色基本信息
  await userStore.hydrate();                // 2. 刷新当前用户权限缓存
  router.push({ name: 'settings-roles-collection' });
}
```

#### 权限规则配置

权限规则配置通过表单组件操作 `directus_permissions` 集合，包含以下核心字段：

| 字段 | 类型 | 说明 |
|------|------|------|
| `policy` | UUID | 关联的策略 ID |
| `collection` | string | 目标集合名称 |
| `action` | string | 操作类型：create/read/update/delete/share |
| `fields` | string[] | 允许访问的字段列表（null 表示全部） |
| `permissions` | Filter | 数据过滤规则（JSON 格式的过滤条件） |
| `validation` | Filter | 数据验证规则 |
| `presets` | object | 字段预设值 |

### 2.2 API 控制器层

**关键文件：** `api/src/controllers/permissions.ts`

#### 权限 CRUD 端点

```typescript
// 创建权限
router.post(
  '/',
  asyncHandler(async (req, res, next) => {
    const service = new PermissionsService({
      accountability: req.accountability,
      schema: req.schema,
    });

    if (Array.isArray(req.body)) {
      await service.createMany(req.body);   // 批量创建
    } else {
      await service.createOne(req.body);     // 单个创建
    }
  }),
  respond,
);

// 读取权限
router.get('/', validateBatch('read'), readHandler, respond);

// 更新权限
router.patch('/', validateBatch('update'), ...);

// 删除权限
router.delete('/', validateBatch('delete'), ...);
```

### 2.3 服务层

**关键文件：** `api/src/services/permissions.ts`

#### PermissionsService 核心实现

```typescript
export class PermissionsService extends ItemsService {
  constructor(options: AbstractServiceOptions) {
    super('directus_permissions', options);  // 操作 directus_permissions 表
  }

  // 权限变更时清除缓存
  private async clearCaches(opts?: MutationOptions) {
    await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });

    if (this.cache && opts?.autoPurgeCache !== false) {
      await this.cache.clear();
    }
  }

  override async createOne(data: Partial<Item>, opts?: MutationOptions) {
    const res = await super.createOne(data, opts);
    await this.clearCaches(opts);  // 创建后清除缓存
    return res;
  }

  override async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts?: MutationOptions) {
    const res = await super.updateMany(keys, data, opts);
    await this.clearCaches(opts);  // 更新后清除缓存
    return res;
  }

  override async deleteMany(keys: PrimaryKey[], opts?: MutationOptions) {
    const res = await super.deleteMany(keys, opts);
    await this.clearCaches(opts);  // 删除后清除缓存
    return res;
  }
}
```

#### 角色删除级联处理

**关键文件：** `api/src/services/roles.ts`

```typescript
override async deleteMany(keys: PrimaryKey[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
  await transaction(this.knex, async (trx) => {
    const accessService = new AccessService(options);
    const presetsService = new PresetsService(options);
    const usersService = new UsersService(options);

    // 1. 删除该角色关联的权限策略
    await accessService.deleteByQuery(
      { filter: { role: { _in: keys } } },
      { ...opts, bypassLimits: true },
    );

    // 2. 删除该角色关联的预设
    await presetsService.deleteByQuery(
      { filter: { role: { _in: keys } } },
      { ...opts, bypassLimits: true },
    );

    // 3. 停用该角色下的所有用户
    await usersService.updateByQuery(
      { filter: { role: { _in: keys } } },
      { status: 'suspended', role: null },
      { ...opts, bypassLimits: true },
    );

    // 4. 清除子角色的 parent 引用
    await rolesService.updateByQuery(
      { filter: { parent: { _in: keys } } },
      { parent: null },
    );

    // 5. 最后删除角色本身
    await rolesItemsService.deleteMany(keys, opts);
  });

  await this.clearCaches();  // 清除权限缓存
  return keys;
}
```

### 2.4 数据库表结构详解

#### 核心表关系（v11 引入策略系统）

**关键迁移文件：** `api/src/database/migrations/20240806A-permissions-policies.ts`

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         权限系统数据模型 (v11)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   directus_roles (角色表)                                                    │
│   ┌─────────────────────────────────────────────────────────────┐          │
│   │ id          │ UUID │ 主键                                    │          │
│   │ name        │ string │ 角色名称                              │          │
│   │ icon        │ string │ 图标                                  │          │
│   │ description │ text │ 描述                                    │          │
│   │ parent      │ UUID │ 父角色 ID (支持角色继承)                │          │
│   └─────────────────────────────────────────────────────────────┘          │
│                                      │                                       │
│                                      ▼                                       │
│   directus_access (关联表 - 连接角色/用户与策略)                           │
│   ┌─────────────────────────────────────────────────────────────┐          │
│   │ id      │ UUID │ 主键                                        │          │
│   │ role    │ UUID │ 角色 ID (可为空)                            │          │
│   │ user    │ UUID │ 用户 ID (可为空，支持用户级策略)            │          │
│   │ policy  │ UUID │ 策略 ID (必填)                              │          │
│   │ sort    │ int  │ 排序顺序                                    │          │
│   └─────────────────────────────────────────────────────────────┘          │
│                                      │                                       │
│                                      ▼                                       │
│   directus_policies (策略表)                                                 │
│   ┌─────────────────────────────────────────────────────────────┐          │
│   │ id            │ UUID │ 主键                                  │          │
│   │ name          │ string │ 策略名称                            │          │
│   │ icon          │ string │ 图标                                │          │
│   │ description   │ text │ 描述                                  │          │
│   │ admin_access  │ boolean │ 是否管理员访问权限                 │          │
│   │ app_access    │ boolean │ 是否允许访问管理后台               │          │
│   │ ip_access     │ text │ IP 白名单                             │          │
│   │ enforce_tfa   │ boolean │ 是否强制双重认证                   │          │
│   └─────────────────────────────────────────────────────────────┘          │
│                                      │                                       │
│                                      ▼                                       │
│   directus_permissions (权限规则表)                                          │
│   ┌─────────────────────────────────────────────────────────────┐          │
│   │ id          │ UUID      │ 主键                               │          │
│   │ policy      │ UUID      │ 关联策略 ID                        │          │
│   │ collection  │ string    │ 目标集合名                         │          │
│   │ action      │ string    │ 操作类型: create/read/update/     │          │
│   │             │           │           delete/share              │          │
│   │ permissions │ JSON      │ 数据过滤规则 (Filter 类型)         │          │
│   │ validation  │ JSON      │ 数据验证规则                       │          │
│   │ presets     │ JSON      │ 字段预设值                         │          │
│   │ fields      │ string[]  │ 允许的字段列表                     │          │
│   │ system      │ boolean   │ 是否系统权限                       │          │
│   └─────────────────────────────────────────────────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 权限规则数据结构

**关键类型定义：** `packages/types/src/permissions.ts`

```typescript
export type Permission = {
  id?: number;
  policy: string | null;           // 关联的策略
  collection: string;               // 目标集合
  action: PermissionsAction;        // 'create' | 'read' | 'update' | 'delete' | 'share'
  permissions: Filter | null;       // 数据过滤规则（最关键的部分）
  validation: Filter | null;        // 数据验证规则
  presets: Record<string, any> | null;  // 字段预设值
  fields: string[] | null;          // 允许访问的字段
  system?: true;                     // 是否系统内置权限
};

// 过滤规则示例 (Filter 类型)
// {
//   "_and": [
//     { "status": { "_eq": "published" } },
//     { "owner": { "_eq": "$CURRENT_USER" } }
//   ]
// }
```

---

## 三、权限执行链路（请求到达 → 权限验证 → 数据访问）

### 3.1 请求处理流程图

```
                    HTTP 请求到达
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    1. 认证中间件 (authenticate.ts)                        │
│  - 解析 JWT Token / Session Cookie                                        │
│  - 调用 getAccountabilityForToken() 获取用户身份信息                      │
│  - 设置 req.accountability 对象                                           │
└──────────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    2. Accountability 构建                                  │
│  accountability = {                                                        │
│    user: "uuid",           // 当前用户 ID                                 │
│    role: "uuid",           // 主角色 ID                                   │
│    roles: ["uuid1", ...],  // 角色树（包含继承的角色）                    │
│    admin: false,           // 是否管理员                                   │
│    app: true,              // 是否允许访问 App                            │
│    ip: "192.168.1.1",      // 请求 IP                                     │
│    share: "uuid",          // 共享链接 ID（如果是通过分享访问）          │
│  }                                                                         │
└──────────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    3. 路由分发到具体 Controller                             │
│  - GET    /items/:collection     → 读取数据                                │
│  - POST   /items/:collection     → 创建数据                                │
│  - PATCH  /items/:collection/:id → 更新数据                                │
│  - DELETE /items/:collection/:id → 删除数据                                │
└──────────────────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                    4. ItemsService 处理                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 读取操作 (readByQuery):                                               │ │
│  │   1. getAstFromQuery() → 构建查询 AST                                 │ │
│  │   2. processAst() → 注入权限过滤规则（核心步骤）                       │ │
│  │   3. runAst() → 执行带权限的 SQL 查询                                 │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 写入操作 (updateMany/deleteMany):                                     │ │
│  │   1. validateAccess() → 验证权限（核心步骤）                          │ │
│  │   2. processPayload() → 处理预设值和验证规则                          │ │
│  │   3. 执行 SQL 操作                                                    │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

### 3.2 认证与 Accountability 构建

**关键文件：** `api/src/middleware/authenticate.ts`

```typescript
export const handler = async (req: Request, res: Response, next: NextFunction) => {
  const defaultAccountability: Accountability = createDefaultAccountability({ 
    ip: getIPFromReq(req) 
  });

  // 1. 触发自定义认证钩子
  const customAccountability = await emitter.emitFilter(
    'authenticate',
    defaultAccountability,
    { req },
    { database, schema: null, accountability: null },
  );

  if (customAccountability && isEqual(customAccountability, defaultAccountability) === false) {
    req.accountability = customAccountability;
    return next();
  }

  // 2. 从 Token 解析用户身份
  try {
    req.accountability = await getAccountabilityForToken(req.token, defaultAccountability);
  } catch (err) {
    // 处理无效 Token
    throw err;
  }

  return next();
};
```

**关键文件：** `api/src/utils/get-accountability-for-token.ts`

```typescript
export async function getAccountabilityForToken(
  token?: string | null,
  accountability?: Accountability,
): Promise<Accountability> {
  if (!accountability) {
    accountability = createDefaultAccountability();
  }

  const database = getDatabase();

  if (token) {
    if (isDirectusJWT(token)) {
      // 处理 JWT Token
      const payload = verifyAccessJWT(token, getSecret());

      if (payload.id) accountability.user = payload.id;
      accountability.role = payload.role;
      
      // 获取角色树（包含继承关系）
      accountability.roles = await fetchRolesTree(payload.role, { knex: database });

      // 获取全局访问权限（admin_access, app_access）
      const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });
      accountability.admin = admin;
      accountability.app = app;
    } else {
      // 处理静态 Token（用户表中的 token 字段）
      const user = await database
        .select('directus_users.id', 'directus_users.role')
        .from('directus_users')
        .where({ 'directus_users.token': token, status: 'active' })
        .first();

      if (!user) throw new InvalidCredentialsError();

      accountability.user = user.id;
      accountability.role = user.role;
      accountability.roles = await fetchRolesTree(user.role, { knex: database });
      
      const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });
      accountability.admin = admin;
      accountability.app = app;
    }
  }

  return accountability;
}
```

### 3.3 读取操作权限处理

**关键文件：** `api/src/services/items.ts:readByQuery`

```typescript
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
  // 1. 构建查询 AST
  let ast = await getAstFromQuery(
    {
      collection: this.collection,
      query: updatedQuery,
      accountability: this.accountability,
    },
    { schema: this.schema, knex: this.knex },
  );

  // 2. 处理权限 - 核心步骤！
  ast = await processAst(
    { ast, action: 'read', accountability: this.accountability },
    { knex: this.knex, schema: this.schema },
  );

  // 3. 执行带权限的查询
  const records = await runAst(ast, this.schema, this.accountability, {
    knex: this.knex,
    stripNonRequested: opts?.stripNonRequested !== undefined ? opts.stripNonRequested : true,
  });

  return filteredRecords as Item[];
}
```

### 3.4 processAst - 权限注入核心

**关键文件：** `api/src/permissions/modules/process-ast/process-ast.ts`

```typescript
export async function processAst(options: ProcessAstOptions, context: Context) {
  const { ast, action, accountability } = options;

  // 1. 从 AST 提取字段映射
  const fieldMap: FieldMap = fieldMapFromAst(ast, context.schema);
  const collections = collectionsInFieldMap(fieldMap);

  // 2. 管理员或无 accountability 时跳过权限检查
  if (!accountability || accountability.admin) {
    // 仅验证字段存在性
    for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
      validatePathExistence(path, collection, fields, context.schema);
    }
    return ast;
  }

  // 3. 获取用户关联的策略
  const policies = await fetchPolicies(accountability, context);

  // 4. 获取具体权限规则
  const permissions = await fetchPermissions(
    { action, policies, collections, accountability },
    context,
  );

  // 5. 获取读取权限（用于验证关联字段）
  const readPermissions = action === 'read'
    ? permissions
    : await fetchPermissions(
        { action: 'read', policies, collections, accountability },
        context,
      );

  // 6. 验证字段存在性
  for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
    validatePathExistence(path, collection, fields, context.schema);
  }

  // 7. 验证字段权限
  for (const [path, { collection, fields }] of fieldMap.other.entries()) {
    validatePathPermissions(path, permissions, collection, fields);
  }
  for (const [path, { collection, fields }] of fieldMap.read.entries()) {
    validatePathPermissions(path, readPermissions, collection, fields);
  }

  // 8. 注入过滤规则到 AST
  injectCases(ast, permissions);

  return ast;
}
```

### 3.5 injectCases - 过滤规则注入

**关键文件：** `api/src/permissions/modules/process-ast/lib/inject-cases.ts`

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
  
  // 1. 构建过滤规则的 case/when 结构
  const { cases, caseMap, allowedFields } = getCases(collection, permissions, requestedKeys);

  for (const child of children) {
    const fieldKey = getUnaliasedFieldKey(child);
    const globalWhenCase = caseMap['*'];
    const fieldWhenCase = caseMap[fieldKey];

    // 2. 将过滤规则绑定到字段
    if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
      // 该字段需要满足过滤条件才能访问
      child.whenCase = [...(globalWhenCase ?? []), ...(fieldWhenCase ?? [])];
    }

    // 3. 递归处理关联字段
    if (child.type === 'm2o') {
      child.cases = processChildren(child.relation.related_collection!, child.children, permissions);
    }
    if (child.type === 'o2m') {
      child.cases = processChildren(child.relation.collection, child.children, permissions);
    }
    // ... 处理 a2o, functionField 等
  }

  return cases;
}
```

### 3.6 getCases - 构建过滤规则

**关键文件：** `api/src/permissions/modules/process-ast/lib/get-cases.ts`

```typescript
export function getCases(collection: string, permissions: Permission[], requestedKeys: string[]) {
  const permissionsForCollection = permissions.filter((p) => p.collection === collection);
  
  // 去重访问规则
  const rules = dedupeAccess(permissionsForCollection);
  
  const cases: Filter[] = [];              // 所有过滤规则
  const caseMap: Record<FieldKey, number[]> = {};  // 字段到规则索引的映射
  let index = 0;

  for (const { rule, fields } of rules) {
    // 检查当前规则是否覆盖请求的字段
    if (requestedKeys.length > 0 && 
        fields.has('*') === false && 
        Array.from(fields).every((field) => requestedKeys.includes(field) === false)
    ) {
      continue;  // 不相关的规则跳过
    }

    if (rule === null) continue;

    cases.push(rule);  // 添加过滤规则

    // 记录哪些字段应用了这些规则
    for (const field of fields) {
      caseMap[field] = [...(caseMap[field] ?? []), index];
    }

    index++;
  }

  // 无条件访问的字段（permissions 为 null）
  const allowedFields = new Set(
    permissionsForCollection
      .filter((p) => hasItemPermissions(p) === false)  // 没有过滤规则
      .map((p) => p.fields ?? [])
      .flat(),
  );

  return { cases, caseMap, allowedFields };
}
```

### 3.7 写入操作权限验证

**关键文件：** `api/src/services/items.ts:updateMany`

```typescript
async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey[]> {
  // 1. 触发更新前钩子
  const payloadAfterHooks = await emitter.emitFilter(
    this.eventScope === 'items'
      ? ['items.update', `${this.collection}.items.update`]
      : `${this.eventScope}.update`,
    payload,
    { keys, collection: this.collection },
    { database: this.knex, schema: this.schema, accountability: this.accountability },
  );

  // 2. 验证权限 - 核心步骤！
  if (this.accountability) {
    await validateAccess(
      {
        accountability: this.accountability,
        action: 'update',
        collection: this.collection,
        primaryKeys: keys,
        fields: Object.keys(payloadAfterHooks),
      },
      { schema: this.schema, knex: this.knex },
    );
  }

  // 3. 处理预设值和验证规则
  const payloadWithPresets = this.accountability
    ? await processPayload(
        {
          accountability: this.accountability,
          action: 'update',
          collection: this.collection,
          payload: payloadAfterHooks,
          nested: this.nested,
        },
        { knex: this.knex, schema: this.schema },
      )
    : payloadAfterHooks;

  // 4. 执行数据库更新
  await transaction(this.knex, async (trx) => {
    // ... 执行更新操作
  });

  return keys;
}
```

### 3.8 validateAccess - 访问验证

**关键文件：** `api/src/permissions/modules/validate-access/validate-access.ts`

```typescript
export async function validateAccess(options: ValidateAccessOptions, context: Context) {
  // 1. 检查集合是否存在
  if (!options.skipCollectionExistsCheck && 
      options.collection in context.schema.collections === false) {
    throw createCollectionForbiddenError('', options.collection);
  }

  // 2. 管理员直接放行
  if (options.accountability.admin === true) {
    return;
  }

  let access: boolean;

  // 3. 根据是否有主键选择不同的验证方式
  if (options.primaryKeys) {
    // 有具体主键时，需要实际查询数据库验证
    const result = await validateItemAccess(
      options as Required<ValidateAccessOptions>, 
      context
    );
    access = result.accessAllowed;
  } else {
    // 无主键时，仅验证集合级别的权限
    access = await validateCollectionAccess(options, context);
  }

  // 4. 权限不足时抛出异常
  if (!access) {
    if (options.fields?.length ?? 0 > 0) {
      throw new ForbiddenError({
        reason: `You don't have permissions to perform "${options.action}" for the field(s) ${
          options.fields!.map((field) => `"${field}"`).join(', ')
        } in collection "${options.collection}" or it does not exist.`,
      });
    }

    throw new ForbiddenError({
      reason: `You don't have permission to perform "${options.action}" for collection "${options.collection}" or it does not exist.`,
    });
  }
}
```

### 3.9 validateItemAccess - 项级权限验证

**关键文件：** `api/src/permissions/modules/validate-access/lib/validate-item-access.ts`

```typescript
export async function validateItemAccess(
  options: ValidateItemAccessOptions,
  context: Context,
): Promise<ValidateItemAccessResult> {
  const collectionInfo = context.schema.collections[options.collection];
  const primaryKeyField = collectionInfo?.primary;

  // 1. 构建验证用的 AST
  const ast: AST = {
    type: 'root',
    name: options.collection,
    query: { limit: options.primaryKeys!.length },
    children: options.fields?.map((field) => ({
      type: 'field',
      name: field,
      fieldKey: field,
      whenCase: [],
      alias: false,
    })) ?? [],
    cases: [],
  };

  // 2. 通过 processAst 注入权限规则
  await processAst({ ast, ...options }, context);

  // 3. 注入主键过滤条件
  if (options.primaryKeys && options.primaryKeys.length > 0) {
    ast.query.filter = {
      [primaryKeyField]: { _in: options.primaryKeys! },
    };
  }

  // 4. 执行查询验证
  const items = await fetchPermittedAstRootFields(ast, {
    schema: context.schema,
    accountability: options.accountability,
    knex: context.knex,
    action: options.action,
  });

  // 5. 检查返回数量是否匹配
  const expectedCount = options.primaryKeys!.length;
  const hasAccess = items && items.length === expectedCount;

  if (!hasAccess) {
    return { accessAllowed: false };
  }

  // 6. 验证指定字段是否都可访问
  let accessAllowed = true;
  if (options.fields) {
    accessAllowed = items.every((item: any) => 
      options.fields!.every((field) => toBoolean(item[field]))
    );
  }

  return { accessAllowed };
}
```

---

## 四、权限缓存机制

### 4.1 缓存配置

**关键文件：** `api/src/permissions/cache.ts`

```typescript
const localOnly = redisConfigAvailable() === false;
const env = useEnv();
const ttl = getMilliseconds(env['CACHE_SYSTEM_TTL']);

// 根据是否有 Redis 选择缓存策略
const config: CacheConfig = localOnly
  ? {
      type: 'local',
      maxKeys: 500,
    }
  : {
      type: 'multi',
      redis: {
        namespace: (env['REDIS_PERMISSIONS_NAMESPACE'] as string) ?? 'permissions',
        redis: useRedis(),
        ...(ttl !== undefined ? { ttl } : {}),
      },
      local: {
        maxKeys: 100,
      },
    };

export const useCache = defineCache(config);

export function clearCache() {
  const cache = useCache();
  return cache.clear();
}
```

### 4.2 缓存清除时机

权限缓存会在以下情况被清除：

1. **权限规则变更时** (`PermissionsService`)
   ```typescript
   // api/src/services/permissions.ts
   private async clearCaches(opts?: MutationOptions) {
     await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });
     
     if (this.cache && opts?.autoPurgeCache !== false) {
       await this.cache.clear();
     }
   }
   ```

2. **角色变更时** (`RolesService`)
   ```typescript
   // api/src/services/roles.ts
   override async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts: MutationOptions = {}) {
     if ('parent' in data) {
       // 父角色变更时清除缓存
       await this.clearCaches();
     }
     // ...
   }
   ```

---

## 五、动态变量解析

权限规则中支持使用动态变量，如 `$CURRENT_USER`、`$CURRENT_ROLE` 等。

### 5.1 动态变量处理流程

**关键文件：** `api/src/permissions/lib/fetch-permissions.ts`

```typescript
export async function fetchPermissions(options: FetchPermissionsOptions, context: Context) {
  // 1. 获取原始权限（包含动态变量占位符）
  const permissions = await fetchRawPermissions(
    { ...options, bypassMinimalAppPermissions: options.bypassDynamicVariableProcessing ?? false },
    context,
  );

  // 2. 处理动态变量
  if (options.accountability && !options.bypassDynamicVariableProcessing) {
    // 提取需要的动态变量上下文
    const dynamicVariableContext = extractRequiredDynamicVariableContextForPermissions(permissions);

    // 获取动态变量的实际值
    const permissionsContext = await fetchDynamicVariableData(
      {
        accountability: options.accountability,
        policies: options.policies,
        dynamicVariableContext,
      },
      context,
    );

    // 替换动态变量为实际值
    const processedPermissions = processPermissions({
      permissions,
      accountability: options.accountability,
      permissionsContext,
    });

    return processedPermissions;
  }

  return permissions;
}
```

### 5.2 常用动态变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `$CURRENT_USER` | 当前用户 ID | `{ "owner": { "_eq": "$CURRENT_USER" } }` |
| `$CURRENT_ROLE` | 当前角色 ID | `{ "role": { "_eq": "$CURRENT_ROLE" } }` |
| `$CURRENT_TIMESTAMP` | 当前时间戳 | `{ "created_at": { "_lte": "$CURRENT_TIMESTAMP" } }` |
| `$NOW` | 当前日期时间 | 同上 |
| `$FOLLOW` | 关联字段引用 | 用于嵌套过滤 |

---

## 六、关键代码位置速查表

| 功能 | 文件路径 |
|------|----------|
| 角色管理页面 | `app/src/modules/settings/routes/roles/item.vue` |
| 权限 API 控制器 | `api/src/controllers/permissions.ts` |
| 角色 API 控制器 | `api/src/controllers/roles.ts` |
| 权限服务 | `api/src/services/permissions.ts` |
| 角色服务 | `api/src/services/roles.ts` |
| 认证中间件 | `api/src/middleware/authenticate.ts` |
| Accountability 构建 | `api/src/utils/get-accountability-for-token.ts` |
| AST 权限处理 | `api/src/permissions/modules/process-ast/process-ast.ts` |
| 访问验证 | `api/src/permissions/modules/validate-access/validate-access.ts` |
| 项级验证 | `api/src/permissions/modules/validate-access/lib/validate-item-access.ts` |
| 过滤规则注入 | `api/src/permissions/modules/process-ast/lib/inject-cases.ts` |
| 权限获取 | `api/src/permissions/lib/fetch-permissions.ts` |
| 权限缓存 | `api/src/permissions/cache.ts` |
| 权限类型定义 | `packages/types/src/permissions.ts` |
| 策略迁移 | `api/src/database/migrations/20240806A-permissions-policies.ts` |

---

## 七、执行流程图总结

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        一次 API 请求的权限检查完整流程                            │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  1. 请求到达                                                                   │
│     │                                                                         │
│     ▼                                                                         │
│  2. authenticate.ts (认证中间件)                                              │
│     ├── 解析 JWT Token / Session Cookie                                       │
│     └── 调用 getAccountabilityForToken()                                      │
│         │                                                                     │
│         ├── fetchRolesTree() → 获取角色继承树                                │
│         └── fetchGlobalAccess() → 获取 admin/app 权限                       │
│     │                                                                         │
│     ▼                                                                         │
│  3. accountability 对象构建完成                                               │
│     { user, role, roles, admin, app, ip, ... }                              │
│     │                                                                         │
│     ▼                                                                         │
│  4. 路由分发到 ItemsService                                                   │
│     │                                                                         │
│     ├── 读取操作 (GET)                                                        │
│     │   ├── getAstFromQuery() → 构建查询 AST                                │
│     │   ├── processAst() → 权限注入 [核心]                                  │
│     │   │   ├── fetchPolicies() → 获取关联策略                              │
│     │   │   ├── fetchPermissions() → 获取权限规则                            │
│     │   │   ├── validatePathPermissions() → 验证字段权限                     │
│     │   │   └── injectCases() → 注入过滤规则到 AST                          │
│     │   └── runAst() → 执行带权限的 SQL 查询                                │
│     │                                                                         │
│     └── 写入操作 (POST/PATCH/DELETE)                                          │
│         ├── validateAccess() → 权限验证 [核心]                               │
│         │   └── validateItemAccess()                                         │
│         │       ├── processAst() → 注入权限规则                             │
│         │       └── fetchPermittedAstRootFields() → 实际查询验证            │
│         ├── processPayload() → 处理预设值和验证规则                          │
│         └── 执行 SQL 写入                                                     │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

---

## 八、关键设计要点

### 8.1 权限模型演变

Directus v11 引入了**策略 (Policy)** 概念，将权限模型从：
```
角色 → 权限
```
演变为：
```
角色/用户 → 策略 → 权限
```

这种设计的优势：
- 支持多角色继承
- 支持用户级策略（不依赖角色）
- 权限规则复用性更好
- 更灵活的权限组合

### 8.2 权限检查的双重保障

1. **读取操作**：通过 `processAst` 将过滤规则**注入到 SQL 查询**中，在数据库层面过滤数据
2. **写入操作**：通过 `validateAccess` **实际查询验证**用户是否有权操作目标数据

### 8.3 字段级权限实现

字段级权限通过以下机制实现：
1. **白名单机制**：`fields` 字段明确列出允许访问的字段
2. **条件字段访问**：通过 `whenCase` 实现"满足过滤条件时才能访问该字段"
3. **SQL 层面处理**：使用 `CASE WHEN` 语句，不满足条件时返回 `NULL`

### 8.4 缓存策略

- 权限信息缓存提高性能
- 权限变更时自动清除缓存
- 支持 Redis 分布式缓存或本地内存缓存

---

*本文档基于 Directus v11 代码分析，版本差异可能导致实现细节有所不同。*
