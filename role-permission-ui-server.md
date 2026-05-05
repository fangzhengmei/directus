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
                                          ▼ HTTP API (REST/GraphQL) / WebSocket
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                API Controllers                                         │
│  ┌─────────────────────────────────────┐  ┌──────────────────────────────────────┐  │
│  │ permissions.ts (权限 CRUD)           │  │ roles.ts (角色 CRUD)                 │  │
│  │ graphql.ts (GraphQL 端点)            │  │ access.ts (策略关联 CRUD)            │  │
│  └─────────────────────────────────────┘  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                                 Services Layer                                         │
│  ┌───────────────────────────────┐  ┌──────────────────────────────────────────────┐ │
│  │ PermissionsService            │  │ RolesService                                │ │
│  │ (api/src/services/permissions.ts)│  │ (api/src/services/roles.ts)               │ │
│  │ AccessService                 │  │ GraphQLService                              │ │
│  │ (api/src/services/access.ts)    │  │ (api/src/services/graphql/index.ts)       │ │
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

### 2.4 角色与策略绑定（AccessService）

**关键文件：** `api/src/services/access.ts`

角色与策略的绑定关系存储在 `directus_access` 表中，由 `AccessService` 管理：

```typescript
export class AccessService extends ItemsService {
  constructor(options: AbstractServiceOptions) {
    super('directus_access', options);  // 操作 directus_access 表
  }

  private async clearCaches(opts?: MutationOptions) {
    await clearSystemCache({ autoPurgeCache: opts?.autoPurgeCache });

    if (this.cache && opts?.autoPurgeCache !== false) {
      await this.cache.clear();
    }
  }

  override async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    // 创建新的策略关联会影响 admin/app/api 用户数量
    opts.userIntegrityCheckFlags =
      (opts.userIntegrityCheckFlags ?? UserIntegrityCheckFlag.None) | UserIntegrityCheckFlag.UserLimits;

    opts.onRequireUserIntegrityCheck?.(opts.userIntegrityCheckFlags);

    const result = await super.createOne(data, opts);

    // 策略关联变更，清除缓存
    await this.clearCaches();

    return result;
  }

  override async updateMany(
    keys: PrimaryKey[],
    data: Partial<Item>,
    opts: MutationOptions = {},
  ): Promise<PrimaryKey[]> {
    // 更新策略关联可能影响用户数量
    opts.userIntegrityCheckFlags = UserIntegrityCheckFlag.All;
    opts.onRequireUserIntegrityCheck?.(opts.userIntegrityCheckFlags);

    const result = await super.updateMany(keys, data, { ...opts, userIntegrityCheckFlags: UserIntegrityCheckFlag.All });

    await this.clearCaches();  // 清除缓存

    return result;
  }

  override async deleteMany(keys: PrimaryKey[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
    // 删除策略关联可能影响用户数量
    opts.userIntegrityCheckFlags = UserIntegrityCheckFlag.All;
    opts.onRequireUserIntegrityCheck?.(opts.userIntegrityCheckFlags);

    const result = await super.deleteMany(keys, opts);

    await this.clearCaches();  // 清除缓存

    return result;
  }
}
```

#### directus_access 表结构

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | UUID | 主键 |
| `role` | UUID | 角色 ID（可为空，支持用户级策略） |
| `user` | UUID | 用户 ID（可为空，支持角色级策略） |
| `policy` | UUID | 策略 ID（必填） |
| `sort` | int | 排序顺序 |

### 2.5 数据库表结构详解

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

## 三、fetchPolicies 装载与权限合并链路

### 3.1 整体流程概览

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                    fetchPolicies 装载与权限合并完整流程                          │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  1. 配置阶段（UI → Database）                                                  │
│     ┌──────────┐     ┌─────────────────┐     ┌─────────────────┐           │
│     │ 角色管理  │────►│ directus_access │────►│ directus_policy │           │
│     │ 策略管理  │     │  (关联绑定)      │     │  (权限策略)     │           │
│     └──────────┘     └─────────────────┘     └─────────────────┘           │
│                                                                                │
│  2. 运行阶段（请求处理）                                                        │
│                                                                                │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │  authenticate.ts (认证中间件)                                         │  │
│     │  └── getAccountabilityForToken()                                     │  │
│     │      └── fetchRolesTree() → 构建角色继承树                           │  │
│     │      └── fetchGlobalAccess() → 检查 admin_access                    │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                      │                                        │
│                                      ▼                                        │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │  processAst() / validateAccess() (权限检查核心)                     │  │
│     │  └── fetchPolicies() → 从 directus_access 加载策略                 │  │
│     │      │                                                               │  │
│     │      ├── 1. 构建过滤条件                                             │  │
│     │      │    - 有角色：role IN (roles 树)                             │  │
│     │      │    - 有用户：OR user = current_user                         │  │
│     │      │    - 无角色无用户：Public 角色 (role IS NULL)               │  │
│     │      │                                                               │  │
│     │      ├── 2. 查询 directus_access                                     │  │
│     │      │    SELECT policy.id, policy.ip_access, role                  │  │
│     │      │    FROM directus_access                                       │  │
│     │      │    LEFT JOIN directus_policies ON policy = policy.id        │  │
│     │      │    WHERE [过滤条件]                                           │  │
│     │      │                                                               │  │
│     │      ├── 3. IP 过滤 (filterPoliciesByIp)                            │  │
│     │      │    - 检查 policy.ip_access 是否包含当前请求 IP               │  │
│     │      │                                                               │  │
│     │      ├── 4. 优先级排序                                               │  │
│     │      │    - 父角色策略 < 子角色策略 < 用户策略                      │  │
│     │      │    - 基于 roles 数组的索引顺序                               │  │
│     │      │                                                               │  │
│     │      └── 5. 缓存 (withCache)                                        │  │
│     │           - Key: policies-{hash(roles, user, ip)}                 │  │
│     │           - 自动清除：权限变更时                                     │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                      │                                        │
│                                      ▼                                        │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │  fetchPermissions() (获取具体权限规则)                              │  │
│     │  ├── 从 directus_permissions 查询                                   │  │
│     │  │   WHERE policy IN (policies 列表)                               │  │
│     │  │   AND collection IN (目标集合)                                   │  │
│     │  │   AND action = (目标操作)                                        │  │
│     │  │                                                                  │  │
│     │  ├── 动态变量处理                                                   │  │
│     │  │   - $CURRENT_USER, $CURRENT_ROLE 等                           │  │
│     │  │   - 替换为实际值                                                 │  │
│     │  │                                                                  │  │
│     │  └── 权限合并 (mergePermissions)                                   │  │
│     │      - 多个策略的权限规则如何合并                                   │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 WebSocket 认证详细流程

**关键文件：** `api/src/websocket/authenticate.ts`

WebSocket 支持三种认证方式，最终都通过 `getAccountabilityForToken()` 构建 `accountability` 对象：

```typescript
export async function authenticateConnection(
	message: BasicAuthMessage & Record<string, any>,
	accountabilityOverrides?: Partial<Accountability>,
): Promise<AuthenticationState> {
	let access_token: string | undefined, refresh_token: string | undefined;

	try {
		// 方式 1：用户名密码登录
		if ('email' in message && 'password' in message) {
			const authenticationService = new AuthenticationService({ schema: await getSchema() });
			const { accessToken, refreshToken } = await authenticationService.login(DEFAULT_AUTH_PROVIDER, message);
			access_token = accessToken;
			refresh_token = refreshToken;
		}

		// 方式 2：Refresh Token 刷新
		if ('refresh_token' in message) {
			const authenticationService = new AuthenticationService({ schema: await getSchema() });
			const { accessToken, refreshToken } = await authenticationService.refresh(message.refresh_token);
			access_token = accessToken;
			refresh_token = refreshToken;
		}

		// 方式 3：直接使用 Access Token
		if ('access_token' in message) {
			access_token = message.access_token;
		}

		if (!access_token) throw new Error();

		// 构建默认 accountability（包含 IP 地址）
		const defaultAccountability = createDefaultAccountability(accountabilityOverrides);

		const authenticationState = {
			accountability: defaultAccountability,
			expires_at: getExpiresAtForToken(access_token),
			refresh_token,
		} as AuthenticationState;

		// 触发自定义认证钩子
		const customAccountability = await emitter.emitFilter(
			'websocket.authenticate',
			defaultAccountability,
			{ message },
			{ database: getDatabase(), schema: null, accountability: null },
		);

		// 使用自定义 accountability 或从 Token 构建
		if (customAccountability && isEqual(customAccountability, defaultAccountability) === false) {
			authenticationState.accountability = customAccountability;
		} else {
			// 与 HTTP 认证相同的流程：解析 JWT → 构建角色树 → 检查 admin/app 权限
			authenticationState.accountability = await getAccountabilityForToken(
				access_token, 
				defaultAccountability
			);
		}

		return authenticationState;
	} catch {
		throw new WebSocketError('auth', 'AUTH_FAILED', 'Authentication failed.', message['uid']);
	}
}
```

### 6.3 verifyPermissions 核心实现

**关键文件：** `api/src/websocket/collab/verify-permissions.ts`

这是 WebSocket 协作权限校验的核心函数，与 REST API 的权限检查相比有以下特点：
- 独立的缓存机制（permissionCache）
- 动态变量解析支持
- 返回允许的字段列表而非抛出异常

```typescript
export async function verifyPermissions(
	accountability: Accountability | null,
	collection: string,
	item: PrimaryKey | null,
	action: 'create' | 'read' | 'update' | 'delete' = 'read',
	options: { knex: Knex; schema: SchemaOverview },
): Promise<string[] | null> {
	// 1. 无 accountability：返回空列表（无权限）
	if (!accountability) return [];

	const { schema, knex } = options;

	// 2. 集合不存在：返回空列表
	if (!schema.collections[collection]) return [];

	// 3. 管理员：返回全部字段
	if (accountability.admin) return ['*'];

	// 4. 尝试从缓存获取
	const cached = permissionCache.get(accountability, collection, String(item), action);
	if (cached !== undefined) return cached;

	// 5. 记录失效计数，防止竞态条件
	const startInvalidationCount = permissionCache.getInvalidationCount();

	let itemData: any = null;

	try {
		const adminService = getService(collection, { schema, knex });

		// 6. 加载用户策略（与 REST API 相同的 fetchPolicies）
		const policies = await fetchPolicies(accountability, { knex, schema });

		// 7. 获取权限规则（跳过动态变量处理，后面单独处理）
		const rawPermissions = await fetchPermissions(
			{ 
				action, 
				collections: [collection], 
				policies, 
				accountability, 
				bypassDynamicVariableProcessing: true 
			},
			{ knex, schema },
		);

		// 8. 检查是否有项级过滤规则（需要查询实际数据）
		const hasItemRules = rawPermissions.some(
			(p) => p.permissions && Object.keys(p.permissions).length > 0
		);

		if (hasItemRules) {
			// 9. 解析权限中使用的动态变量
			const dynamicVariableContext = extractRequiredDynamicVariableContextForPermissions(rawPermissions);

			const permissionsContext = await fetchDynamicVariableData(
				{ accountability, policies, dynamicVariableContext },
				{ knex, schema },
			);

			// 10. 处理权限规则（替换动态变量值）
			const processedPermissions = processPermissions({
				permissions: rawPermissions,
				accountability,
				permissionsContext,
			});

			// 11. 确定需要查询的字段（用于评估过滤条件）
			const fieldsToFetch = processedPermissions
				.map((perm) => (perm.permissions ? filterToFields(perm.permissions, collection, schema) : []))
				.flat();

			// 12. 查询当前数据项（用于评估条件权限）
			if (item && action !== 'create') {
				try {
					itemData = await adminService.readOne(item, {
						fields: fieldsToFetch,
					});
				} catch {
					// 数据项不存在
					permissionCache.set(accountability, collection, String(item), action, null, []);
					return null;
				}
			} else if (schema.collections[collection]?.singleton && action !== 'create') {
				// 处理单例集合
				itemData = await adminService.readSingleton({ fields: fieldsToFetch });
			}
		}

		// 13. 获取允许的字段列表
		let allowedFields: string[] = [];

		if ((item || schema.collections[collection]?.singleton) && hasItemRules) {
			// 有项级规则：使用 validateItemAccess（实际查询验证）
			const primaryKeys: (string | number)[] = item ? [item] : [];

			const validationContext = {
				collection,
				accountability,
				action,
				primaryKeys,
				returnAllowedRootFields: true,
			};

			allowedFields = (await validateItemAccess(validationContext, { knex, schema })).allowedRootFields || [];
		} else {
			// 无项级规则：使用 fetchAllowedFields（仅集合级别验证）
			allowedFields = await fetchAllowedFields({ accountability, action, collection }, { knex, schema });
		}

		// 14. 缓存结果（仅当期间无失效发生）
		if (permissionCache.getInvalidationCount() === startInvalidationCount) {
			// 计算 TTL 和依赖关系
			const { ttlMs, dependencies } = calculateCacheMetadata(
				collection,
				itemData,
				rawPermissions,
				schema,
				accountability,
			);

			permissionCache.set(
				accountability, 
				collection, 
				String(item), 
				action, 
				allowedFields, 
				dependencies, 
				ttlMs
			);
		}

		return allowedFields;
	} catch (err) {
		useLogger().error(
			err,
			`[Collab] verifyPermissions failed for user "${accountability.user}", collection "${collection}", and item "${item}"`,
		);
		return [];
	}
}
```

### 6.4 协作权限缓存机制

**关键文件：** `api/src/websocket/collab/permissions-cache.ts`

WebSocket 协作使用独立的权限缓存系统，与 REST API 的 `withCache` 不同：

```typescript
export class PermissionCache {
	private cache: LRUMapWithDelete<CacheKey, string[] | null>;  // LRU 缓存
	private tags = new Map<Tag, Set<CacheKey>>();   // 标签到缓存键的映射
	private keyTags = new Map<CacheKey, Set<Tag>>(); // 缓存键到标签的映射
	private timers = new Map<CacheKey, NodeJS.Timeout>(); // TTL 定时器
	private bus = useBus();  // 消息总线（用于多实例同步）
	private invalidationCount = 0;  // 失效计数（用于竞态条件检测）

	constructor(maxSize: number) {
		this.cache = new LRUMapWithDelete(maxSize);

		// 订阅系统事件，处理缓存失效
		this.bus.subscribe('websocket.event', (event: any) => {
			this.handleInvalidation(event);
		});
	}

	/**
	 * 处理缓存失效
	 */
	private handleInvalidation(event: any) {
		const { collection, keys, key } = event;
		const items = keys || (key ? [key] : []);
		const affectedKeys = new Set<CacheKey>();

		// 系统级失效：角色、权限、策略、结构变更 → 清空全部缓存
		if (
			[
				'directus_roles',
				'directus_permissions',
				'directus_policies',
				'directus_access',
				'directus_fields',
				'directus_relations',
				'directus_collections',
			].includes(collection)
		) {
			this.clear();
			return;
		}

		// 跳过已知高流量集合
		if (IRRELEVANT_COLLECTIONS.includes(collection)) {
			return;
		}

		this.invalidationCount++;

		// 集合级失效
		if (items.length === 0 && this.tags.has(`collection:${collection}`)) {
			for (const k of this.tags.get(`collection:${collection}`)!) affectedKeys.add(k);
		}

		// 项级失效
		for (const id of items) {
			const tag = `item:${collection}:${id}`;
			if (this.tags.has(tag)) {
				for (const k of this.tags.get(tag)!) affectedKeys.add(k);
			}
		}

		// 依赖失效（关联集合变化）
		const depTags = [`dependency:${collection}`];
		if (items.length > 0) {
			for (const id of items) {
				depTags.push(`dependency:${collection}:${id}`);
			}
		} else {
			depTags.push(`collection-dependency:${collection}`);
		}

		for (const tag of depTags) {
			if (this.tags.has(tag)) {
				for (const k of this.tags.get(tag)!) affectedKeys.add(k);
			}
		}

		// 执行失效
		for (const k of affectedKeys) {
			this.invalidateKey(k);
		}
	}

	/**
	 * 缓存 key 格式：user:collection:item:action
	 */
	private getCacheKey(
		accountability: Accountability,
		collection: string,
		item: string | null,
		action: string,
	): CacheKey {
		return `${accountability.user || 'public'}:${collection}:${item || 'singleton'}:${action}`;
	}
}

// 全局单例，默认容量 2000
export const permissionCache = new PermissionCache(
	Number(env['WEBSOCKETS_COLLAB_PERMISSIONS_CACHE_CAPACITY'] ?? 2000)
);
```

### 6.5 房间加入权限检查

**关键文件：** `api/src/websocket/collab/collab.ts` 的 `onJoin` 方法

```typescript
async onJoin(client: WebSocketClient, message: JoinMessage) {
	// 1. 不支持共享链接的协作编辑
	if (client.accountability?.share) {
		throw new ForbiddenError({
			reason: 'Collaborative editing is not supported for shares',
		});
	}

	const schema = await getSchema();
	const db = getDatabase();

	try {
		// 2. 验证对目标数据项的读取权限
		const { accessAllowed } = await validateItemAccess(
			{
				accountability: client.accountability!,
				action: 'read',
				collection: message.collection,
				// 单例集合不需要主键
				primaryKeys: schema.collections[message.collection]?.singleton ? [] : [message.item!],
			},
			{ knex: db, schema },
		);

		if (!accessAllowed) throw new ForbiddenError();

		// 3. 如果有版本号，验证对版本的访问权限
		if (message.version) {
			const { accessAllowed: versionAccessAllowed } = await validateItemAccess(
				{
					accountability: client.accountability!,
					action: 'read',
					collection: 'directus_versions',
					primaryKeys: [message.version],
				},
				{ knex: db, schema },
			);

			if (!versionAccessAllowed) throw new ForbiddenError();
		}
	} catch {
		throw new ForbiddenError({
			reason: `No permission to access item or it does not exist`,
		});
	}

	// 4. 验证初始变更的权限
	if (message.initialChanges) {
		await validateChanges(
			message.initialChanges, 
			message.collection, 
			message.item, 
			{
				knex: db,
				schema,
				accountability: client.accountability,
			}
		);
	}

	// 5. 创建房间并加入
	const room = await this.roomManager.createRoom(
		message.collection,
		message.item,
		message.version ?? null,
		message.initialChanges,
	);

	await room.join(client, message.color);
}
```

---

## 七、三种请求方式权限校验对比

### 7.1 整体对比表

| 维度 | REST API | GraphQL | WebSocket 协作 |
|------|----------|---------|----------------|
| **认证入口** | `authenticate.ts` 中间件 | `authenticate.ts` 中间件 | `authenticateConnection()` 函数 |
| **认证方式** | JWT / Session / Static Token | 与 REST 相同 | email/password / refresh_token / access_token |
| **Accountability 构建** | `getAccountabilityForToken()` | 与 REST 相同 | 与 REST 相同 |
| **权限校验核心** | `processAst()` / `validateAccess()` | 与 REST 相同 | `verifyPermissions()` |
| **策略加载** | `fetchPolicies()` | 与 REST 相同 | `fetchPolicies()`（复用） |
| **权限缓存** | `withCache()` 包装 | 与 REST 相同 | 独立 `PermissionCache` (LRU + 标签) |
| **缓存失效** | 系统级 `clearSystemCache()` | 与 REST 相同 | 消息总线驱动 + 标签精确失效 |
| **返回值** | 抛出 `ForbiddenError` | 与 REST 相同 | 返回允许的字段列表 / null |

### 7.2 代码复用关系

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                          权限系统核心模块（所有入口复用）                        │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  fetchPolicies()              ← 从 directus_access 加载策略列表    │    │
│  │  fetchPermissions()           ← 从 directus_permissions 加载规则   │    │
│  │  fetchAllowedFields()        ← 获取允许的字段列表                   │    │
│  │  validateItemAccess()        ← 验证项级访问权限                     │    │
│  │  processAst()                ← 注入过滤规则到 SQL AST               │    │
│  │  processPermissions()        ← 处理动态变量替换                      │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
                                      ▲
                                      │ 复用
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────┐          ┌───────────────┐          ┌───────────────────────┐
│  REST API     │          │  GraphQL      │          │  WebSocket 协作       │
├───────────────┤          ├───────────────┤          ├───────────────────────┤
│               │          │               │          │                       │
│  入口:        │          │  入口:        │          │  入口:                │
│  ItemsService │          │ GraphQLService│          │ CollabHandler         │
│               │          │               │          │                       │
│  权限检查:    │          │  权限检查:    │          │  权限检查:            │
│  - read:      │          │  - read:      │          │  verifyPermissions()  │
│    processAst │          │    read() →   │          │  (独立实现，复用底层) │
│  - write:     │          │    ItemsService│          │                       │
│    validateA- │          │  - write:     │          │  缓存:                │
│    ccess      │          │    Resolver → │          │  PermissionCache      │
│               │          │    ItemsService│          │  (独立 LRU + 标签)   │
│  缓存:        │          │               │          │                       │
│  withCache()  │          │  缓存:        │          │  通信:                │
│  (系统级)     │          │  与 REST 相同 │          │  消息总线驱动失效     │
│               │          │               │          │                       │
└───────────────┘          └───────────────┘          └───────────────────────┘
```

### 7.3 关键差异点详解

#### 差异 1：缓存机制

| 特性 | REST/GraphQL | WebSocket 协作 |
|------|--------------|----------------|
| **缓存实现** | `withCache()` 装饰器 | `PermissionCache` 类 |
| **缓存结构** | 简单 KV 存储 | LRU Map + 标签索引 |
| **缓存 Key** | `namespace-hash(params)` | `user:collection:item:action` |
| **失效粒度** | 系统级清空（粗粒度） | 标签精确失效（细粒度） |
| **多实例同步** | 依赖 Redis 缓存 | 消息总线 (`websocket.event`) |
| **竞态保护** | 无 | `invalidationCount` 计数器 |

#### 差异 2：权限检查返回值

**REST/GraphQL：抛出异常**
```typescript
// validateAccess.ts
if (!access) {
    throw new ForbiddenError({
        reason: `You don't have permission to perform "${action}"...`,
    });
}
```

**WebSocket 协作：返回字段列表**
```typescript
// verifyPermissions.ts
// 返回值含义：
// - ['*']: 全部字段权限
// - ['title', 'status']: 特定字段权限
// - []: 无权限
// - null: 数据项不存在
return allowedFields;
```

#### 差异 3：动态变量处理时机

**REST/GraphQL：在 fetchPermissions 中处理**
```typescript
// fetch-permissions.ts
const processedPermissions = processPermissions({
    permissions: rawPermissions,
    accountability,
    permissionsContext,
});
```

**WebSocket 协作：在 verifyPermissions 中单独处理**
```typescript
// verify-permissions.ts
// 1. 先获取原始权限（跳过动态变量处理）
const rawPermissions = await fetchPermissions(
    { ..., bypassDynamicVariableProcessing: true },
    ...
);

// 2. 检查是否需要项级数据
const hasItemRules = rawPermissions.some(
    (p) => p.permissions && Object.keys(p.permissions).length > 0
);

// 3. 有项级规则时才查询数据并处理动态变量
if (hasItemRules) {
    const dynamicVariableContext = extractRequiredDynamicVariableContextForPermissions(rawPermissions);
    const permissionsContext = await fetchDynamicVariableData(...);
    const processedPermissions = processPermissions({...});
    // ... 查询数据项 ...
}
```

---

## 八、总结

### 8.1 权限系统核心流程总览

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                         Directus 权限系统完整执行流程                            │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  【配置阶段】                                                                   │
│                                                                                │
│  Admin UI → REST API → AccessService/PermissionsService → Database          │
│     │              │                │                          │              │
│     ▼              ▼                ▼                          ▼              │
│  角色/策略    directus_access   清除系统缓存          directus_roles         │
│  权限配置     directus_permissions              directus_policies           │
│                                          directus_access             │
│                                          directus_permissions              │
│                                                                                │
│  ============================================================================ │
│                                                                                │
│  【运行阶段】                                                                   │
│                                                                                │
│  请求到达（HTTP / WebSocket）                                                  │
│         │                                                                      │
│         ▼                                                                      │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  1. 认证（三种入口，同一构建逻辑）                                    │    │
│  │                                                                       │    │
│  │  REST:     authenticate 中间件 → getAccountabilityForToken()       │    │
│  │  GraphQL:  authenticate 中间件 → getAccountabilityForToken()       │    │
│  │  WebSocket: authenticateConnection() → getAccountabilityForToken() │    │
│  │                                                                       │    │
│  │  产出: accountability = { user, role, roles, admin, app, ip }      │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│         │                                                                      │
│         ▼                                                                      │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  2. 策略加载（所有入口复用 fetchPolicies）                           │    │
│  │                                                                       │    │
│  │  fetchPolicies(accountability, context)                             │    │
│  │    │                                                                  │    │
│  │    ├── 1. 构建过滤条件                                               │    │
│  │    │    - 有角色: role IN (roles 树)                               │    │
│  │    │    - 有用户: OR user = current_user                            │    │
│  │    │    - 无角色: role IS NULL (Public)                             │    │
│  │    │                                                                  │    │
│  │    ├── 2. 查询 directus_access + directus_policies                 │    │
│  │    │                                                                  │    │
│  │    ├── 3. IP 过滤 (filterPoliciesByIp)                               │    │
│  │    │                                                                  │    │
│  │    ├── 4. 优先级排序                                                 │    │
│  │    │    父角色 < 子角色 < 用户策略                                   │    │
│  │    │                                                                  │    │
│  │    └── 5. 缓存 (withCache 或 PermissionCache)                       │    │
│  │                                                                       │    │
│  │  产出: string[] - 策略 ID 列表                                       │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│         │                                                                      │
│         ▼                                                                      │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  3. 权限规则获取与处理                                               │    │
│  │                                                                       │    │
│  │  fetchPermissions({ action, policies, collections, accountability })│    │
│  │    │                                                                  │    │
│  │    ├── 1. 查询 directus_permissions                                 │    │
│  │    │    WHERE policy IN (policies)                                  │    │
│  │    │      AND collection IN (collections)                           │    │
│  │    │      AND action = action                                        │    │
│  │    │                                                                  │    │
│  │    ├── 2. 动态变量替换                                               │    │
│  │    │    $CURRENT_USER → accountability.user                         │    │
│  │    │    $CURRENT_ROLE → accountability.role                         │    │
│  │    │    $NOW → 当前时间戳                                            │    │
│  │    │                                                                  │    │
│  │    └── 3. OR 合并多个策略的权限                                      │    │
│  │                                                                       │    │
│  │  产出: Permission[] - 权限规则列表                                   │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│         │                                                                      │
│         ▼                                                                      │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  4. 权限执行（三种入口差异）                                         │    │
│  │                                                                       │    │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐│    │
│  │  │   REST/GraphQL  │  │   REST/GraphQL  │  │   WebSocket 协作    ││    │
│  │  │   (读取操作)     │  │   (写入操作)     │  │                     ││    │
│  │  ├─────────────────┤  ├─────────────────┤  ├─────────────────────┤│    │
│  │  │                 │  │                 │  │                     ││    │
│  │  │ processAst()    │  │ validateAccess()│  │ verifyPermissions() ││    │
│  │  │                 │  │                 │  │                     ││    │
│  │  │ 注入过滤规则到   │  │ 实际查询数据库  │  │ 返回允许的字段列表  ││    │
│  │  │ SQL AST         │  │ 验证权限        │  │                     ││    │
│  │  │                 │  │                 │  │ 用于:               ││    │
│  │  │ 适用:            │  │ 适用:            │  │ - 房间加入         ││    │
│  │  │ - readByQuery    │  │ - create/update │  │ - 字段更新         ││    │
│  │  │ - readOne        │  │ - delete        │  │ - 聚焦/取消聚焦    ││    │
│  │  │ - readSingleton  │  │                 │  │ - 丢弃变更         ││    │
│  │  │                 │  │                 │  │                     ││    │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────┘│    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 关键代码位置速查表

| 功能模块 | 文件路径 | 关键函数/类 |
|----------|----------|-------------|
| **策略加载** | `api/src/permissions/lib/fetch-policies.ts` | `fetchPolicies`, `_fetchPolicies` |
| **权限加载** | `api/src/permissions/lib/fetch-permissions.ts` | `fetchPermissions` |
| **读取权限注入** | `api/src/permissions/modules/process-ast/process-ast.ts` | `processAst`, `injectCases` |
| **写入权限验证** | `api/src/permissions/modules/validate-access/validate-access.ts` | `validateAccess`, `validateItemAccess` |
| **权限合并** | `api/src/permissions/utils/merge-permissions.ts` | `mergePermissions` |
| **IP 过滤** | `api/src/permissions/utils/filter-policies-by-ip.ts` | `filterPoliciesByIp` |
| **动态变量处理** | `api/src/permissions/utils/process-permissions.ts` | `processPermissions` |
| **REST 缓存** | `api/src/permissions/utils/with-cache.ts` | `withCache` |
| **Accountability 构建** | `api/src/utils/get-accountability-for-token.ts` | `getAccountabilityForToken` |
| **角色树加载** | `api/src/utils/fetch-roles-tree.ts` | `fetchRolesTree` |
| **GraphQL 服务** | `api/src/services/graphql/index.ts` | `GraphQLService` |
| **GraphQL 控制器** | `api/src/controllers/graphql.ts` | 路由定义 |
| **WebSocket 认证** | `api/src/websocket/authenticate.ts` | `authenticateConnection` |
| **协作权限校验** | `api/src/websocket/collab/verify-permissions.ts` | `verifyPermissions` |
| **协作权限缓存** | `api/src/websocket/collab/permissions-cache.ts` | `PermissionCache` |
| **协作处理器** | `api/src/websocket/collab/collab.ts` | `CollabHandler` |
| **协作房间** | `api/src/websocket/collab/room.ts` | `Room`, `RoomManager` |
| **Access 服务** | `api/src/services/access.ts` | `AccessService` |
| **权限服务** | `api/src/services/permissions.ts` | `PermissionsService` |
| **角色服务** | `api/src/services/roles.ts` | `RolesService` |

### 8.3 核心设计思想

1. **三层权限模型（v11 引入）**
   - 角色 → 策略 → 权限规则
   - 通过 `directus_access` 关联表实现灵活绑定
   - 支持角色继承、用户级策略覆盖

2. **OR 权限合并策略**
   - 多个策略的权限规则使用 OR 逻辑合并
   - 任意一个策略允许的字段/条件，用户就有权访问
   - 优先级：用户策略 > 子角色 > 父角色

3. **动态变量支持**
   - `$CURRENT_USER`, `$CURRENT_ROLE`, `$NOW` 等
   - 在权限检查时动态替换为实际值
   - 支持基于数据状态的条件权限

4. **缓存分层设计**
   - REST/GraphQL：系统级缓存，简单高效
   - WebSocket 协作：独立 LRU 缓存 + 标签精确失效
   - 多实例通过消息总线同步失效事件

5. **代码复用最大化**
   - `fetchPolicies`, `fetchPermissions`, `validateItemAccess` 等核心函数所有入口复用
   - 三种请求方式的差异仅在最上层封装
   - 降低维护成本，确保行为一致性

### 3.2 fetchPolicies 核心实现

**关键文件：** `api/src/permissions/lib/fetch-policies.ts`

```typescript
export interface AccessRow {
  policy: { id: string; ip_access: string[] | null };
  role: string | null;
}

// 使用缓存包装器
export const fetchPolicies = withCache(
  'policies', 
  _fetchPolicies, 
  ({ roles, user, ip }) => ({ roles, user, ip })  // 缓存 key 的参数
);

/**
 * Fetch the policies associated with the current user accountability
 */
export async function _fetchPolicies(
  { roles, user, ip }: Pick<Accountability, 'user' | 'roles' | 'ip'>,
  context: Context,
): Promise<string[]> {
  const { AccessService } = await import('../../services/access.js');
  const accessService = new AccessService(context);

  let roleFilter: Filter;

  // 1. 构建角色过滤条件
  if (roles.length === 0) {
    // 无角色用户使用 Public 角色权限
    roleFilter = { _and: [{ role: { _null: true } }, { user: { _null: true } }] };
  } else {
    // 有角色用户：过滤角色树中的所有角色
    roleFilter = { role: { _in: roles } };
  }

  // 2. 如果有用户，还需包含用户级策略
  const filter = user ? { _or: [{ user: { _eq: user } }, roleFilter] } : roleFilter;

  // 3. 查询 directus_access 表，关联获取策略信息
  const accessRows = (await accessService.readByQuery({
    filter,
    fields: ['policy.id', 'policy.ip_access', 'role'],  // 同时获取策略的 IP 限制
    limit: -1,
  })) as AccessRow[];

  // 4. IP 地址过滤
  const filteredAccessRows = filterPoliciesByIp(accessRows, ip);

  /*
   * 5. 按优先级排序 (从低到高):
   * - 父角色策略 (roles 数组中索引较小的)
   * - 子角色策略 (roles 数组中索引较大的)
   * - 用户策略 (role 为 null 但 user 有值)
   */
  filteredAccessRows.sort((a, b) => {
    if (!a.role && !b.role) return 0;    // 都是用户策略：顺序不变
    if (!a.role) return 1;                 // a 是用户策略：排后面（高优先级）
    if (!b.role) return -1;                // b 是用户策略：b 排后面

    // 基于 roles 数组的索引顺序（父角色在前，子角色在后）
    return roles.indexOf(a.role) - roles.indexOf(b.role);
  });

  // 6. 提取策略 ID 列表
  const ids = filteredAccessRows.map(({ policy }) => policy.id);

  return ids;
}
```

### 3.3 IP 过滤机制

**关键文件：** `api/src/permissions/utils/filter-policies-by-ip.ts`

```typescript
export function filterPoliciesByIp(
  policies: AccessRow[], 
  ip: string | null | undefined
) {
  return policies.filter(({ policy }) => {
    // 1. 没有配置 IP 白名单的策略：始终保留
    if (!policy.ip_access || policy.ip_access.length === 0) {
      return true;
    }

    // 2. 配置了 IP 白名单但无法获取客户端 IP：安全起见，拒绝访问
    if (!ip) {
      return false;
    }

    // 3. 检查 IP 是否在允许列表中（支持 CIDR 格式）
    return ipInNetworks(ip, policy.ip_access);
  });
}
```

### 3.4 缓存机制

**关键文件：** `api/src/permissions/utils/with-cache.ts`

```typescript
export function withCache<F extends (...args: any) => any>(
  namespace: string,
  handler: F,
  prepareArg?: (...args: Parameters<F>) => Record<string, unknown>,
): (...args: Parameters<F>) => Promise<Awaited<ReturnType<F>>> {
  const cache = useCache();

  return async (...args) => {
    // 1. 准备缓存 key 参数
    const hashArgs = prepareArg ? prepareArg(...args) : args;
    
    // 2. 生成缓存 key：namespace + hash(参数)
    const key = namespace + '-' + getSimpleHash(JSON.stringify(hashArgs));
    
    // 3. 尝试从缓存获取
    const cached = await cache.get(key);

    if (cached !== undefined) {
      return cached as Awaited<ReturnType<F>>;
    }

    // 4. 缓存未命中，执行实际查询
    const res = await handler(...args);

    // 5. 存入缓存
    cache.set(key, res);

    return res;
  };
}
```

### 3.5 缓存清除触发点

策略缓存会在以下情况被清除：

| 触发点 | 服务/文件 | 说明 |
|--------|----------|------|
| 策略关联创建 | `AccessService.createOne` | 新绑定角色-策略 |
| 策略关联更新 | `AccessService.updateMany` | 修改策略绑定 |
| 策略关联删除 | `AccessService.deleteMany` | 解除策略绑定 |
| 角色父级变更 | `RolesService.updateMany` | 角色继承关系变化 |
| 角色删除 | `RolesService.deleteMany` | 级联清理策略关联 |
| 权限规则变更 | `PermissionsService.*` | 权限规则增删改 |

### 3.6 权限合并策略

当多个策略都有权限规则时，Directus 使用 OR 逻辑合并：

**关键文件：** `api/src/permissions/utils/merge-permissions.ts`

```typescript
// 权限合并规则（OR 逻辑）：
// 1. 多个策略中任意一个允许的字段，用户就有权访问
// 2. 多个策略的过滤条件使用 OR 合并
// 3. 优先级：用户策略 > 子角色策略 > 父角色策略

// 示例场景：
// 策略 A（父角色）：
//   - 允许读取 articles 集合的 title, status 字段
//   - 过滤条件：status = 'published'
//
// 策略 B（子角色）：
//   - 允许读取 articles 集合的 content, author 字段
//   - 过滤条件：author = $CURRENT_USER
//
// 合并后权限：
//   - 允许字段：title, status, content, author（OR 合并）
//   - 过滤条件：(status = 'published') OR (author = $CURRENT_USER)
```

---

## 四、权限执行链路（请求到达 → 权限验证 → 数据访问）

### 4.1 请求处理流程图

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
│  - POST   /graphql              → GraphQL 查询                             │
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

### 4.2 认证与 Accountability 构建

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

### 4.3 读取操作权限处理

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

### 4.4 processAst - 权限注入核心

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

### 4.5 injectCases - 过滤规则注入

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

### 4.6 getCases - 构建过滤规则

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

### 4.7 写入操作权限验证

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

### 4.8 validateAccess - 访问验证

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

### 4.9 validateItemAccess - 项级权限验证

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

## 五、GraphQL 请求权限校验链路

### 5.1 整体流程概览

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                        GraphQL 请求权限校验完整流程                                │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  1. HTTP 请求到达                                                              │
│     POST /graphql 或 POST /graphql/system                                    │
│     │                                                                         │
│     ▼                                                                         │
│  2. 认证中间件 (authenticate.ts)                                              │
│     - 与 REST API 完全相同的认证流程                                          │
│     - 构建 req.accountability 对象                                            │
│     │                                                                         │
│     ▼                                                                         │
│  3. GraphQL 路由 (graphql.ts)                                                 │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │  router.use(                                                         │  │
│     │    '/',                                                              │  │
│     │    parseGraphQL,        // 解析 GraphQL 查询文档                   │  │
│     │    asyncHandler(async (req, res, next) => {                        │  │
│     │      const service = new GraphQLService({                           │  │
│     │        accountability: req.accountability,  // 传递 accountability  │  │
│     │        schema: req.schema,                                          │  │
│     │        scope: 'items',              // 或 'system'                 │  │
│     │      });                                                             │  │
│     │                                                                      │  │
│     │      res.locals['payload'] = await service.execute(                │  │
│     │        res.locals['graphqlParams']  // 解析后的查询参数            │  │
│     │      );                                                              │  │
│     │    }),                                                               │  │
│     │    respond,                                                          │  │
│     │  );                                                                  │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│     │                                                                         │
│     ▼                                                                         │
│  4. GraphQLService.execute()                                                 │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │  async execute({ document, variables, operationName, contextValue }) │  │
│     │  {                                                                   │  │
│     │    const schema = await this.getSchema();  // 生成 GraphQL Schema   │  │
│     │                                                                      │  │
│     │    // 1. 验证 GraphQL 查询文档                                       │  │
│     │    const validationErrors = validate(schema, document, rules);    │  │
│     │                                                                      │  │
│     │    // 2. 执行查询                                                    │  │
│     │    result = await execute({                                         │  │
│     │      schema,                                                        │  │
│     │      document,                                                      │  │
│     │      contextValue,                                                  │  │
│     │      variableValues: variables,                                    │  │
│     │      operationName,                                                 │  │
│     │    });                                                               │  │
│     │  }                                                                   │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│     │                                                                         │
│     ▼                                                                         │
│  5. Resolver 执行 (resolveQuery / resolveMutation)                          │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │  // 查询解析器                                                        │  │
│     │  async function resolveQuery(gql: GraphQLService, info) {          │  │
│     │    // 1. 解析 GraphQL 参数为 Directus Query                          │  │
│     │    const query = await getQuery(args, gql.schema, selections, ...); │  │
│     │                                                                      │  │
│     │    // 2. 调用 GraphQLService.read()                                  │  │
│     │    const result = await gql.read(collection, query, args['id']);   │  │
│     │  }                                                                   │  │
│     │                                                                      │  │
│     │  // 变更解析器                                                        │  │
│     │  async function resolveMutation(gql, args, info) {                  │  │
│     │    const action = info.fieldName.split('_')[0]; // create/update/delete│
│     │    const service = getService(collection, {                         │  │
│     │      knex: gql.knex,                                                 │  │
│     │      accountability: gql.accountability,  // 传递 accountability    │  │
│     │      schema: gql.schema,                                             │  │
│     │    });                                                               │  │
│     │                                                                      │  │
│     │    // 直接调用 ItemsService 方法                                      │  │
│     │    if (action === 'create') await service.createOne(args['data']);  │  │
│     │    if (action === 'update') await service.updateOne(args['id'], ...);│  │
│     │    if (action === 'delete') await service.deleteOne(args['id']);    │  │
│     │  }                                                                   │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│     │                                                                         │
│     ▼                                                                         │
│  6. 复用 REST API 权限校验                                                   │
│     - GraphQLService.read() → ItemsService.readByQuery/readOne             │
│     - Resolver 直接调用 ItemsService.createOne/updateOne/deleteOne          │
│     - 所有权限检查逻辑与 REST API 完全一致                                   │
│       ├── processAst() → 注入权限过滤规则                                   │
│       ├── validateAccess() → 验证访问权限                                   │
│       └── fetchPolicies() → 加载策略列表                                   │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 GraphQL 控制器

**关键文件：** `api/src/controllers/graphql.ts`

```typescript
import { Router } from 'express';
import { parseGraphQL } from '../middleware/graphql.js';
import { respond } from '../middleware/respond.js';
import { GraphQLService } from '../services/graphql/index.js';
import asyncHandler from '../utils/async-handler.js';

const router = Router();

// 系统集合 GraphQL 端点
router.use(
  '/system',
  parseGraphQL,
  asyncHandler(async (req, res, next) => {
    const service = new GraphQLService({
      accountability: req.accountability,  // 从认证中间件传递
      schema: req.schema,
      scope: 'system',
    });

    res.locals['payload'] = await service.execute(res.locals['graphqlParams']);

    if (res.locals['payload']?.errors?.length > 0) {
      res.locals['cache'] = false;
    }

    return next();
  }),
  respond,
);

// 用户集合 GraphQL 端点
router.use(
  '/',
  parseGraphQL,
  asyncHandler(async (req, res, next) => {
    const service = new GraphQLService({
      accountability: req.accountability,  // 传递 accountability
      schema: req.schema,
      scope: 'items',
    });

    res.locals['payload'] = await service.execute(res.locals['graphqlParams']);

    if (res.locals['payload']?.errors?.length > 0) {
      res.locals['cache'] = false;
    }

    return next();
  }),
  respond,
);

export default router;
```

### 5.3 GraphQLService 核心实现

**关键文件：** `api/src/services/graphql/index.ts`

```typescript
export class GraphQLService {
  accountability: Accountability | null;  // 保存 accountability
  knex: Knex;
  schema: SchemaOverview;
  scope: GQLScope;

  constructor(options: AbstractServiceOptions & { scope: GQLScope }) {
    this.accountability = options?.accountability || null;  // 存储
    this.knex = options?.knex || getDatabase();
    this.schema = options.schema;
    this.scope = options.scope;
  }

  /**
   * Execute a GraphQL structure
   */
  async execute({
    document,
    variables,
    operationName,
    contextValue,
  }: GraphQLParams): Promise<FormattedExecutionResult> {
    const schema = await this.getSchema();

    // 验证 GraphQL 查询
    const validationErrors = validate(schema, document, validationRules).map((validationError) =>
      addPathToValidationError(validationError),
    );

    if (validationErrors.length > 0) {
      throw new GraphQLValidationError({ errors: validationErrors });
    }

    // 执行 GraphQL 查询
    let result: ExecutionResult;
    try {
      result = await execute({
        schema,
        document,
        contextValue,
        variableValues: variables,
        operationName,
      });
    } catch (err: any) {
      throw new GraphQLExecutionError({ errors: [err.message] });
    }

    // 格式化结果
    const formattedResult: FormattedExecutionResult = {};
    if (result['data']) formattedResult.data = result['data'];
    if (result['errors']) {
      formattedResult.errors = result['errors'].map((error) => 
        processError(this.accountability, error)
      );
    }

    return formattedResult;
  }

  /**
   * 读取操作 - 委托给 ItemsService
   */
  async read(collection: string, query: Query, id?: PrimaryKey): Promise<Partial<Item>> {
    const service = getService(collection, {
      knex: this.knex,
      accountability: this.accountability,  // 传递 accountability
      schema: this.schema,
    });

    if (this.schema.collections[collection]!.singleton)
      return await service.readSingleton(query, { stripNonRequested: false });

    if (id) return await service.readOne(id, query, { stripNonRequested: false });

    return await service.readByQuery(query, { stripNonRequested: false });
  }

  /**
   * 单例更新操作
   */
  async upsertSingleton(
    collection: string,
    body: Record<string, any> | Record<string, any>[],
    query: Query,
  ): Promise<Partial<Item> | boolean> {
    const service = getService(collection, {
      knex: this.knex,
      accountability: this.accountability,  // 传递 accountability
      schema: this.schema,
    });

    try {
      await service.upsertSingleton(body);

      if ((query.fields || []).length > 0) {
        const result = await service.readSingleton(query);
        return result;
      }

      return true;
    } catch (err: any) {
      throw formatError(err);
    }
  }
}
```

### 5.4 查询解析器

**关键文件：** `api/src/services/graphql/resolvers/query.ts`

```typescript
export async function resolveQuery(gql: GraphQLService, info: GraphQLResolveInfo): Promise<Partial<Item> | null> {
  let collection = info.fieldName;
  if (gql.scope === 'system') collection = `directus_${collection}`;
  
  const selections = replaceFragmentsInSelections(
    info.fieldNodes[0]?.selectionSet?.selections, 
    info.fragments
  );

  if (!selections) return null;
  
  // 解析 GraphQL 参数
  const args: Record<string, any> = parseArgs(
    info.fieldNodes[0]!.arguments || [], 
    info.variableValues
  );

  let query: Query;
  const isAggregate = collection.endsWith('_aggregated') && 
    collection in gql.schema.collections === false;

  if (isAggregate) {
    // 聚合查询
    collection = collection.slice(0, -11);
    query = await getAggregateQuery(args, selections, gql.schema, gql.accountability, collection);
  } else {
    // 普通查询
    if (collection.endsWith('_by_id') && collection in gql.schema.collections === false) {
      collection = collection.slice(0, -6);
    }

    // 构建 Directus Query 对象
    query = await getQuery(args, gql.schema, selections, info.variableValues, gql.accountability, collection);

    if (collection.endsWith('_by_version') && collection in gql.schema.collections === false) {
      collection = collection.slice(0, -11);
      query.versionRaw = true;
    }
  }

  // 调用 GraphQLService.read()，最终委托给 ItemsService
  const result = await gql.read(collection, query, args['id']);

  if (args['id']) return result;

  // 处理分组查询
  if (query.group) {
    const aggregateKeys = Object.keys(query.aggregate ?? {});
    result['map']((payload: Item) => {
      payload['group'] = omit(payload, aggregateKeys);
    });
  }

  return result;
}
```

### 5.5 变更解析器

**关键文件：** `api/src/services/graphql/resolvers/mutation.ts`

```typescript
export async function resolveMutation(
  gql: GraphQLService,
  args: Record<string, any>,
  info: GraphQLResolveInfo,
): Promise<Partial<Item> | boolean | undefined> {
  // 从字段名解析操作类型和集合
  const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
  let collection = info.fieldName.substring(action.length + 1);
  if (gql.scope === 'system') collection = `directus_${collection}`;

  const selections = replaceFragmentsInSelections(
    info.fieldNodes[0]?.selectionSet?.selections, 
    info.fragments
  );
  
  // 构建查询（用于读取返回数据）
  const query = await getQuery(
    args, gql.schema, selections || [], info.variableValues, gql.accountability, collection
  );

  // 判断操作类型
  const singleton =
    collection.endsWith('_batch') === false &&
    collection.endsWith('_items') === false &&
    collection.endsWith('_item') === false &&
    collection in gql.schema.collections;

  const single = collection.endsWith('_items') === false && collection.endsWith('_batch') === false;
  const batchUpdate = action === 'update' && collection.endsWith('_batch');

  // 清理集合名称
  if (collection.endsWith('_batch')) collection = collection.slice(0, -6);
  if (collection.endsWith('_items')) collection = collection.slice(0, -6);
  if (collection.endsWith('_item')) collection = collection.slice(0, -5);

  // 单例更新
  if (singleton && action === 'update') {
    return await gql.upsertSingleton(collection, args['data'], query);
  }

  // 获取 ItemsService（会传递 accountability）
  const service = getService(collection, {
    knex: gql.knex,
    accountability: gql.accountability,  // 关键：传递 accountability
    schema: gql.schema,
  });

  const hasQuery = (query.fields || []).length > 0;

  try {
    if (single) {
      // 单条操作
      if (action === 'create') {
        const key = await service.createOne(args['data']);  // 触发权限检查
        return hasQuery ? await service.readOne(key, query) : true;
      }

      if (action === 'update') {
        const key = await service.updateOne(args['id'], args['data']);  // 触发权限检查
        return hasQuery ? await service.readOne(key, query) : true;
      }

      if (action === 'delete') {
        await service.deleteOne(args['id']);  // 触发权限检查
        return { id: args['id'] };
      }

      return undefined;
    } else {
      // 批量操作
      if (action === 'create') {
        const keys = await service.createMany(args['data']);  // 触发权限检查
        return hasQuery ? await service.readMany(keys, query) : true;
      }

      if (action === 'update') {
        const keys: PrimaryKey[] = [];

        if (batchUpdate) {
          keys.push(...(await service.updateBatch(args['data'])));  // 触发权限检查
        } else {
          keys.push(...(await service.updateMany(args['ids'], args['data'])));  // 触发权限检查
        }

        return hasQuery ? await service.readMany(keys, query) : true;
      }

      if (action === 'delete') {
        const keys = await service.deleteMany(args['ids']);  // 触发权限检查
        return { ids: keys };
      }

      return undefined;
    }
  } catch (err: any) {
    return formatError(err);
  }
}
```

### 5.6 GraphQL 与 REST API 权限校验对比

| 阶段 | REST API | GraphQL |
|------|----------|---------|
| 认证 | authenticate.ts 中间件 | authenticate.ts 中间件（完全相同） |
| Accountability | req.accountability | GraphQLService.accountability |
| 读取权限 | ItemsService.readByQuery 调用 processAst | 相同：通过 GraphQLService.read → ItemsService |
| 写入权限 | ItemsService.update/delete 调用 validateAccess | 相同：Resolver 直接调用 ItemsService 方法 |
| 策略加载 | fetchPolicies | 相同：由 ItemsService 内部调用 |

**核心结论：** GraphQL 请求的权限校验与 REST API **完全复用相同的代码路径**，只是入口不同。

---

## 六、WebSocket 协作请求权限校验链路

### 6.1 整体流程概览

```
┌────────────────────────────────────────────────────────────────────────────────┐
│                    WebSocket 协作请求权限校验完整流程                            │
├────────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  【第一阶段：WebSocket 连接建立与认证】                                        │
│                                                                                │
│  1. WebSocket 连接请求到达                                                     │
│     GET /websocket/collab (Upgrade: websocket)                              │
│     │                                                                         │
│     ▼                                                                         │
│  2. WebSocket 认证 (websocket/authenticate.ts)                              │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │  async function authenticateConnection(message, accountabilityOverrides)│  │
│     │  {                                                                   │  │
│     │    // 支持多种认证方式                                               │  │
│     │    if ('email' in message && 'password' in message) {              │  │
│     │      // 1. 用户名密码登录                                           │  │
│     │      const { accessToken, refreshToken } =                         │  │
│     │        await authenticationService.login(DEFAULT_AUTH_PROVIDER, message);│  │
│     │      access_token = accessToken;                                    │  │
│     │    }                                                                │  │
│     │                                                                      │  │
│     │    if ('refresh_token' in message) {                               │  │
│     │      // 2. Refresh Token 刷新                                       │  │
│     │      const { accessToken, refreshToken } =                         │  │
│     │        await authenticationService.refresh(message.refresh_token);  │  │
│     │      access_token = accessToken;                                    │  │
│     │    }                                                                │  │
│     │                                                                      │  │
│     │    if ('access_token' in message) {                                │  │
│     │      // 3. 直接使用 Access Token                                    │  │
│     │      access_token = message.access_token;                           │  │
│     │    }                                                                │  │
│     │                                                                      │  │
│     │    // 4. 构建 accountability（与 HTTP 认证相同）                    │  │
│     │    authenticationState.accountability =                             │  │
│     │      await getAccountabilityForToken(access_token, defaultAccountability);│  │
│     │  }                                                                   │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│     │                                                                         │
│     ▼                                                                         │
│  3. 认证成功，client.accountability 已设置                                   │
│                                                                                │
│  ============================================================================ │
│                                                                                │
│  【第二阶段：协作房间加入与权限校验】                                          │
│                                                                                │
│  4. 客户端发送 join 消息                                                       │
│     { type: 'collab