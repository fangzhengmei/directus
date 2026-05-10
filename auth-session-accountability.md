# Directus 身份认证体系分析

## 目录

1. [整体架构概览](#整体架构概览)
2. [登录流程详解](#登录流程详解)
3. [访问令牌颁发机制](#访问令牌颁发机制)
4. [会话管理与令牌续签](#会话管理与令牌续签)
5. [请求认证中间件链与三条认证分支](#请求认证中间件链与三条认证分支)
6. [操作追责(Accountability)体系](#操作追责accountability体系)
7. [端到端时序说明](#端到端时序说明)
8. [关键配置项（校正后）](#关键配置项校正后)
9. [核心文件索引](#核心文件索引)

---

## 整体架构概览

Directus 的身份认证体系是一个多层次、可扩展的系统，核心设计理念：

```
┌─────────────────────────────────────────────────────────────────┐
│                        请求处理管道                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌─────────────────┐   │
│  │ extractToken │───▶│ authenticate │───▶│    业务路由      │   │
│  │  提取令牌     │    │  建立身份    │    │  (req.accounta- │   │
│  │              │    │              │    │   bility)       │   │
│  └──────────────┘    └──────┬───────┘    └────────┬────────┘   │
│                            │                       │            │
│                    ┌───────▼───────┐               │            │
│                    │ 认证分支路由  │               │            │
│                    │               │               │            │
│                    │ 1. JWT 令牌   │               │            │
│                    │ 2. 静态令牌   │               │            │
│                    │ 3. 匿名请求   │               │            │
│                    └───────────────┘               │            │
│                                                    │            │
│                                    ┌───────────────▼──────────┐ │
│                                    │    服务层 (ItemsService)  │ │
│                                    │                          │ │
│                                    │ ┌──────────────────────┐ │ │
│                                    │ │  processPayload      │ │ │
│                                    │ │  (权限校验 + 预设注入) │ │ │
│                                    │ └──────────────────────┘ │ │
│                                    │                          │ │
│                                    │ ┌──────────────────────┐ │ │
│                                    │ │  validateAccess      │ │ │
│                                    │ │  (访问控制检查)       │ │ │
│                                    │ └──────────────────────┘ │ │
│                                    │                          │ │
│                                    │ ┌──────────────────────┐ │ │
│                                    │ │  ActivityService     │ │ │
│                                    │ │  (审计日志落库)       │ │ │
│                                    │ └──────────────────────┘ │ │
│                                    └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### 认证提供者系统

Directus 通过可插拔的 `AuthDriver` 模式支持多种认证方式：

| 认证驱动 | 文件位置 | 说明 |
|---------|---------|------|
| Local | `api/src/auth/drivers/local.ts` | 邮箱/密码认证（argon2 哈希） |
| OAuth2 | `api/src/auth/drivers/oauth2.ts` | OAuth 2.0 协议 |
| OpenID | `api/src/auth/drivers/openid.ts` | OpenID Connect 协议 |
| LDAP | `api/src/auth/drivers/ldap.ts` | LDAP 目录服务 |
| SAML | `api/src/auth/drivers/saml.ts` | SAML 2.0 协议 |

抽象基类 `api/src/auth/auth.ts` 定义的生命周期方法：

```typescript
abstract class AuthDriver {
    abstract getUserID(payload): Promise<string>;   // 获取用户 ID
    abstract verify(user, password?): Promise<void>; // 验证凭据
    async login(user, payload): Promise<void>;       // 登录钩子
    async refresh(user): Promise<void>;              // 刷新钩子
    async logout(user): Promise<void>;               // 登出钩子
}
```

### 缓存装饰器机制

`api/src/permissions/utils/with-cache.ts` 定义了权限相关查询的缓存机制：

```typescript
export function withCache<F extends (...args: any) => any>(
    namespace: string,
    handler: F,
    prepareArg?: (...args: Parameters<F>) => Record<string, unknown>,
): (...args: Parameters<F>) => Promise<Awaited<ReturnType<F>>> {
    const cache = useCache();

    return async (...args) => {
        const hashArgs = prepareArg ? prepareArg(...args) : args;
        const key = namespace + '-' + getSimpleHash(JSON.stringify(hashArgs));
        const cached = await cache.get(key);

        if (cached !== undefined) {
            return cached as Awaited<ReturnType<F>>;
        }

        const res = await handler(...args);

        cache.set(key, res);

        return res;
    };
}
```

**缓存行为（代码可验证）：**
- 首次调用时执行 `handler` 并缓存结果
- 后续使用相同参数调用时从缓存返回
- 缓存键基于 `namespace` 和参数 hash 生成

**已确认使用 withCache 装饰的函数：**

| 函数 | 文件位置 | namespace |
|-----|---------|-----------|
| `fetchRolesTree` | `api/src/permissions/lib/fetch-roles-tree.ts` | `'roles-tree'` |
| `fetchGlobalAccess` | `api/src/permissions/modules/fetch-global-access/fetch-global-access.ts` | `'global-access'` |
| `fetchGlobalAccessForRoles` | 同上 | `'global-access-roles'` |
| `fetchGlobalAccessForUser` | 同上 | `'global-access-user'` |
| `fetchPolicies` | `api/src/permissions/lib/fetch-policies.ts` | `'policies'` |
| `fetchRawPermissions` | `api/src/permissions/utils/fetch-raw-permissions.ts` | `'raw-permissions'` |

---

## 登录流程详解

### 入口路由

登录请求通过 `POST /auth/login` 端点处理，路由定义在 `api/src/controllers/auth.ts:60-68`：

```typescript
// 默认本地认证
router.use('/login', createLocalAuthRouter(DEFAULT_AUTH_PROVIDER));

// 其他认证提供者
router.use(`/login/${authProvider.name}`, authRouter);
```

### 本地登录路由

`api/src/auth/drivers/local.ts:48-119` 定义了本地认证的具体实现：

**请求验证**：使用 Joi Schema 验证 `email` 和 `password` 字段

```typescript
const userLoginSchema = Joi.object({
    email: Joi.string().email().required(),
    password: Joi.string().required(),
    mode: Joi.string().valid('cookie', 'json', 'session'),
    otp: Joi.string(),
}).unknown();
```

**创建 Accountability**：从请求中提取上下文信息

```typescript
const accountability: Accountability = createDefaultAccountability({
    ip: getIPFromReq(req),
});
if (userAgent) accountability.userAgent = userAgent;
if (origin) accountability.origin = origin;
```

**三种认证返回模式**：

| 模式 (mode) | 说明 | refresh_token 存储 | access_token 返回 |
|------------|------|-------------------|------------------|
| `json` | 纯 API 模式 | 响应体 JSON 中 | 响应体 JSON 中 |
| `cookie` | Cookie 模式 | HttpOnly Cookie | 响应体 JSON 中 |
| `session` | 有状态会话模式 | 嵌入 JWT 的 session 字段 | HttpOnly Cookie (JWT) |

### AuthenticationService.login() 详解

核心登录逻辑在 `api/src/services/authentication.ts:55-320`，完整流程：

```
┌──────────────────────────────────────────────────────────────┐
│                     1. 认证提供者验证                         │
├──────────────────────────────────────────────────────────────┤
│  provider.getUserID(payload)                                 │
│    → 通过邮箱查询 directus_users 表                           │
│    → 校验用户状态 status === 'active'                         │
│    → 校验用户 provider 与请求 provider 匹配                    │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     2. 登录尝试速率限制                       │
├──────────────────────────────────────────────────────────────┤
│  读取 auth_login_attempts 配置（SettingsService）              │
│  超过限制则：                                                  │
│    → 将用户 status 更新为 'suspended'                          │
│    → 记录 Activity (UPDATE) 审计日志                          │
│    → 记录 Revision 数据变更历史                               │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     3. 密码验证                               │
├──────────────────────────────────────────────────────────────┤
│  provider.login(user, payload)                               │
│  LocalAuthDriver 实现：                                       │
│    argon2.verify(user.password, payload['password'])         │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     4. 双因素认证 (TFA)                       │
├──────────────────────────────────────────────────────────────┤
│  检查 user.tfa_secret 是否存在（来自第 1 步查询结果）           │
│  存在则要求请求包含 options.otp                               │
│  TFAService.verifyOTP(user.id, otp) 验证一次性密码            │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     5. 权限计算                               │
├──────────────────────────────────────────────────────────────┤
│  fetchRolesTree(user.role)                                   │
│    → 使用 withCache('roles-tree') 装饰                        │
│                                                              │
│  fetchGlobalAccess({ roles, user, ip })                      │
│    → 使用 withCache('global-access') 装饰                     │
│    → 内部调用 fetchGlobalAccessForRoles 和 fetchGlobalAccessForUser │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     6. 令牌生成                               │
├──────────────────────────────────────────────────────────────┤
│  生成 refreshToken: nanoid(64)                               │
│  构建 DirectusTokenPayload:                                  │
│    { id, role, app_access, admin_access, enforce_tfa? }      │
│                                                              │
│  emitter.emitFilter('auth.jwt', ...)                         │
│    → 允许扩展自定义 JWT Claims                                │
│                                                              │
│  jwt.sign(customClaims, getSecret(), {                       │
│    expiresIn: TTL,                                           │
│    issuer: 'directus'                                        │
│  })                                                          │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     7. 会话创建                               │
├──────────────────────────────────────────────────────────────┤
│  INSERT INTO directus_sessions                               │
│    (token, user, expires, ip, user_agent, origin)            │
│                                                              │
│  清理：DELETE FROM directus_sessions WHERE expires < NOW()    │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     8. 审计日志                               │
├──────────────────────────────────────────────────────────────┤
│  ActivityService.createOne({                                 │
│    action: Action.LOGIN,                                     │
│    user: user.id,                                            │
│    ip: accountability.ip,                                    │
│    user_agent: accountability.userAgent,                     │
│    origin: accountability.origin,                            │
│    collection: 'directus_users',                             │
│    item: user.id                                             │
│  })                                                          │
│                                                              │
│  UPDATE directus_users SET last_access = NOW()               │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     9. 事件触发                               │
├──────────────────────────────────────────────────────────────┤
│  emitter.emitAction('auth.login', {                          │
│    payload, status: 'success',                               │
│    user: user.id, provider                                   │
│  })                                                          │
│                                                              │
│  重置登录尝试计数器：loginAttemptsLimiter.set(user.id, 0, 0)  │
└──────────────────────────────────────────────────────────────┘
```

### 防暴力破解机制

`api/src/utils/stall.ts` 实现的登录延迟机制：

```typescript
const STALL_TIME = env['LOGIN_STALL_TIME'] as number;
const timeStart = performance.now();

// ... 登录逻辑 ...

await stall(STALL_TIME, timeStart);
```

**代码行为**：无论登录成功或失败，都确保至少消耗 `LOGIN_STALL_TIME`（默认 500ms）。

---

## 访问令牌颁发机制

### DirectusTokenPayload 结构

JWT 访问令牌的 payload 定义在 `api/src/types/auth.ts`：

```typescript
interface DirectusTokenPayload {
    id: string;                    // 用户 ID (directus_users.id)
    role: string;                  // 角色 ID (directus_roles.id)
    app_access: boolean;           // 是否可访问管理后台
    admin_access: boolean;         // 是否管理员权限
    enforce_tfa?: boolean;         // 是否强制启用 2FA
    session?: string;              // 有状态会话模式下的 refreshToken
    share?: string;                // 共享访问时的 share ID
}
```

### 令牌签名配置

`api/src/services/authentication.ts:276-279`：

```typescript
const accessToken = jwt.sign(customClaims, getSecret(), {
    expiresIn: TTL,
    issuer: 'directus',
});
```

### JWT 自定义扩展点

`api/src/services/authentication.ts:258-272` 提供的 `auth.jwt` filter 钩子：

```typescript
const customClaims = await emitter.emitFilter(
    'auth.jwt',
    tokenPayload,
    {
        status: 'pending',
        user: user?.id,
        provider: providerName,
        type: 'login',  // 或 'refresh'
    },
    { database, schema, accountability },
);
```

---

## 会话管理与令牌续签

### directus_sessions 表结构

会话信息持久化存储在数据库中，核心字段：

| 字段 | 类型 | 说明 |
|-----|------|------|
| token | string(64) | 刷新令牌（nanoid 生成的 64 位随机字符串） |
| user | string | 关联用户 ID (directus_users.id) |
| share | string | 关联共享 ID (directus_shares.id，可选) |
| expires | datetime | 过期时间 |
| next_token | string(64) | 下一刷新令牌（用于并发刷新的优雅过渡） |
| ip | string | 登录 IP 地址 |
| user_agent | text | 浏览器 User-Agent |
| origin | string | 请求来源 Origin |

### 令牌刷新流程

`POST /auth/refresh` 端点处理，核心逻辑在 `api/src/services/authentication.ts:322-480`：

```
┌──────────────────────────────────────────────────────────────┐
│                 1. 获取当前模式与刷新令牌                     │
├──────────────────────────────────────────────────────────────┤
│  三种模式获取 refresh_token 的方式：                          │
│                                                              │
│  mode='json':    req.body.refresh_token                      │
│  mode='cookie':  req.cookies[REFRESH_TOKEN_COOKIE_NAME]      │
│  mode='session': 从 JWT payload.session 中提取                │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                 2. 验证刷新令牌有效性                         │
├──────────────────────────────────────────────────────────────┤
│  SELECT s.*, u.*, d.*                                        │
│  FROM directus_sessions s                                    │
│  LEFT JOIN directus_users u ON s.user = u.id                 │
│  LEFT JOIN directus_shares d ON s.share = d.id               │
│  WHERE s.token = refreshToken                                │
│    AND s.expires >= NOW()                                    │
│    AND (d.date_end IS NULL OR d.date_end >= NOW())           │
│    AND (d.date_start IS NULL OR d.date_start <= NOW())       │
│                                                              │
│  用户状态检查：                                               │
│    - u.status !== 'active' → 删除会话                         │
│    - u.status === 'suspended' → 抛 UserSuspendedError        │
│    - 其他非 active 状态 → 抛 InvalidCredentialsError          │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                 3. 权限重计算                                 │
├──────────────────────────────────────────────────────────────┤
│  fetchRolesTree(record.user_role)                            │
│    → withCache('roles-tree')                                 │
│                                                              │
│  fetchGlobalAccess({ user, roles, ip })                      │
│    → withCache('global-access')                              │
│                                                              │
│  外部认证提供者钩子：                                          │
│    provider.refresh(user)                                    │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                 4. 分支：有状态 vs 无状态                     │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────┐        ┌─────────────────────┐      │
│  │   mode='session'    │        │  mode='json/cookie' │      │
│  │   有状态会话模式     │        │     无状态模式       │      │
│  ├─────────────────────┤        ├─────────────────────┤      │
│  │ updateStateful-     │        │ UPDATE directus_    │      │
│  │   Session()         │        │   sessions         │      │
│  │                     │        │ SET token=newToken,│      │
│  │ • 优雅过渡机制       │        │     expires=newExp │      │
│  │ • 宽限期 10s        │        │ WHERE token=oldTok │      │
│  └─────────────────────┘        └─────────────────────┘      │
│                                                              │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                 5. 生成新访问令牌                             │
├──────────────────────────────────────────────────────────────┤
│  构建新的 DirectusTokenPayload                                │
│  emitter.emitFilter('auth.jwt', ..., { type: 'refresh' })    │
│  jwt.sign(...) 签发新的 access_token                          │
│                                                              │
│  Share 模式特殊处理：                                         │
│    tokenPayload.share = record.share_id                      │
│    tokenPayload.role = null                                  │
│    delete tokenPayload.id                                    │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                 6. 清理与更新                                 │
├──────────────────────────────────────────────────────────────┤
│  UPDATE directus_users SET last_access = NOW()               │
│                                                              │
│  DELETE FROM directus_sessions                               │
│  WHERE user = record.user_id                                 │
│    AND share = record.share_id                               │
│    AND expires < NOW()                                       │
└──────────────────────────────────────────────────────────────┘
```

### 有状态会话的优雅过渡机制

`api/src/services/authentication.ts:482-538` 实现的 `updateStatefulSession()` 方法：

**问题场景（代码注释说明）：**
- 浏览器并发发起多个请求
- 请求 A 正在刷新令牌
- 请求 B 使用旧令牌可能失败

**代码实现的解决方案**：

```
时间线：
─────────────────────────────────────────────────────────────▶

T0 (刷新前)
┌─────────────────────────────┐
│ token: old_token            │
│ expires: T0 + 7d            │
│ next_token: null            │
└─────────────────────────────┘

T1 (刷新中)
┌─────────────────────────────┐      ┌─────────────────────────────┐
│ token: old_token            │◄─────│ token: new_token            │
│ expires: T1 + 10s (GRACE)   │ refs │ expires: T1 + 7d            │
│ next_token: new_token       │      │ next_token: null            │
└─────────────────────────────┘      └─────────────────────────────┘
         ↑                                    ↑
   宽限期内仍可使用                      新的主令牌

T2 (宽限期后，T1 + 10s)
                                    ┌─────────────────────────────┐
                                    │ token: new_token            │
                                    │ expires: T1 + 7d            │
                                    │ next_token: null            │
                                    └─────────────────────────────┘
old_token 已过期，仅 new_token 有效
```

**核心数据库操作逻辑（代码可验证）**：

```typescript
// 1. 检查是否已有 next_token（防止重复刷新）
// SELECT next_token FROM directus_sessions WHERE token = oldSessionToken

// 2. 更新旧会话：缩短过期时间 + 指向新令牌
UPDATE directus_sessions
SET next_token = newSessionToken,
    expires = NOW() + GRACE_PERIOD  -- 默认 10s
WHERE token = oldSessionToken
  AND next_token IS NULL;

// 3. 如果更新失败（说明已被并发刷新），查询现有 next_token
if (updatedSession.length === 0) {
    SELECT next_token FROM directus_sessions WHERE token = oldSessionToken
    return next_token;
}

// 4. 创建新的会话记录
INSERT INTO directus_sessions (token, user, share, expires, ip, user_agent, origin)
VALUES (newSessionToken, ...)
```

### 登出流程

`api/src/services/authentication.ts:540-570`：

```typescript
async logout(refreshToken: string): Promise<void> {
    // 1. 查询会话和用户信息
    const record = await knex
        .select('u.*, s.*')
        .from('directus_sessions as s')
        .innerJoin('directus_users as u', 's.user', 'u.id')
        .where('s.token', refreshToken)
        .first();

    if (record) {
        // 2. 调用认证提供者登出钩子
        const provider = getAuthProvider(user.provider);
        await provider.logout(clone(user));

        // 3. 记录登出审计日志
        if (this.accountability) {
            await this.activityService.createOne({
                action: Action.LOGOUT,
                user: user.id,
                ip: this.accountability.ip,
                user_agent: this.accountability.userAgent,
                origin: this.accountability.origin,
                collection: 'directus_users',
                item: user.id,
            });
        }

        // 4. 删除会话记录
        await this.knex.delete().from('directus_sessions').where('token', refreshToken);
    }
}
```

---

## 请求认证中间件链与三条认证分支

### 中间件注册顺序

`api/src/app.ts:245-307` 定义的完整中间件链：

```typescript
app.use(cookieParser());       // 1. 解析 Cookie (req.cookies)
app.use(extractToken);         // 2. 提取令牌到 req.token
// ... 速率限制中间件 ...
app.use(authenticate);         // 3. 认证并建立 req.accountability
app.use(schema);               // 4. 加载数据库 schema (req.schema)
app.use(sanitizeQuery);        // 5. 清理查询参数
// ... 业务路由 ...
```

### 1. extractToken 中间件

`api/src/middleware/extract-token.ts` 负责从多个来源提取令牌：

**提取优先级（代码可验证）**：

```
优先级 1: Query 参数
   ?access_token=eyJhbGciOiJIUzI1NiIs...

优先级 2: Authorization Header
   Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

优先级 3: Session Cookie (仅当前面都没有时)
   Cookie: directus_session_token=eyJhbGciOiJIUzI1NiIs...
```

**RFC 6750 合规性（代码注释说明）：**
- 不允许同时使用 query 参数 + Authorization Header
- 允许 Session Cookie 与其他方式共存（特殊例外）

### 2. authenticate 中间件

`api/src/middleware/authenticate.ts` 是认证核心：

```
┌──────────────────────────────────────────────────────────────┐
│              Step 1: 创建默认 Accountability                  │
├──────────────────────────────────────────────────────────────┤
│  defaultAccountability = createDefaultAccountability({       │
│      ip: getIPFromReq(req)                                   │
│  })                                                          │
│  defaultAccountability.userAgent = req.get('user-agent')     │
│  defaultAccountability.origin = req.get('origin')            │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│              Step 2: 自定义认证钩子（扩展点）                  │
├──────────────────────────────────────────────────────────────┤
│  customAccountability = emitter.emitFilter(                  │
│      'authenticate',                                         │
│      defaultAccountability,                                  │
│      { req },                                                │
│      { database, schema: null, accountability: null }        │
│  )                                                           │
│                                                              │
│  如果返回值与默认值不同 → 直接使用，跳过默认认证流程            │
│  if (customAccountability !== defaultAccountability) {       │
│      req.accountability = customAccountability;              │
│      return next();                                          │
│  }                                                           │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│              Step 3: 进入三条认证分支                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│       req.token 存在?                                        │
│            │                                                 │
│       ┌────┴────┐                                            │
│       │         │                                            │
│     Yes        No                                            │
│       │         │                                            │
│       │    ┌────▼──────────────────────────────────┐         │
│       │    │  分支 C: 匿名请求                      │         │
│       │    │  accountability = defaultAccounta-    │         │
│       │    │     bility (role=null, user=null)     │         │
│       │    └───────────────────────────────────────┘         │
│       │                                                       │
│  ┌────▼─────────────────────────────────────────────────┐    │
│  │  isDirectusJWT(req.token)?                           │    │
│  │  jwt.decode + 检查 iss === 'directus'                │    │
│  │      │                                               │    │
│  │  ┌───┴───┐                                           │    │
│  │  │       │                                           │    │
│  │ Yes      No                                          │    │
│  │  │       │                                           │    │
│  │  │  ┌────▼──────────────────────────────────────┐    │    │
│  │  │  │  分支 B: 静态令牌                          │    │    │
│  │  │  │  查询 directus_users.token                │    │    │
│  │  │  └───────────────────────────────────────────┘    │    │
│  │  │                                                    │    │
│  │  ┌────▼──────────────────────────────────────────┐    │    │
│  │  │  分支 A: JWT 访问令牌                         │    │    │
│  │  │  verifyAccessJWT() + verifySessionJWT()       │    │    │
│  │  │  + fetchRolesTree + fetchGlobalAccess         │    │    │
│  │  └───────────────────────────────────────────────┘    │    │
│  └───────────────────────────────────────────────────────┘    │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│              Step 4: 无效令牌处理                             │
├──────────────────────────────────────────────────────────────┤
│  catch (err) {                                               │
│      if (是无效令牌 && 来自 Session Cookie) {                 │
│          res.clearCookie(SESSION_COOKIE_NAME, ...);          │
│      }                                                        │
│      throw err;                                               │
│  }                                                            │
└──────────────────────────────────────────────────────────────┘
```

### 分支 A：JWT 访问令牌认证

`api/src/utils/get-accountability-for-token.ts:24-43` 的完整实现：

```typescript
if (isDirectusJWT(token)) {
    // Step 1: JWT 签名验证
    const payload = verifyAccessJWT(token, getSecret());
    // → jwt.verify() with issuer='directus'
    // → 检查必需字段: role, app_access, admin_access

    // Step 2: 会话有效性验证（仅 session 模式）
    if ('session' in payload) {
        await verifySessionJWT(payload);
        // SELECT 1 FROM directus_sessions
        // WHERE token = payload.session
        //   AND user = payload.id
        //   AND expires >= NOW()
        
        accountability.session = payload.session;
    }

    // Step 3: 填充基本字段
    if (payload.share) accountability.share = payload.share;
    if (payload.id) accountability.user = payload.id;
    accountability.role = payload.role;

    // Step 4: 获取角色继承链
    // 代码依据: 第 37 行调用 fetchRolesTree
    // 函数使用 withCache('roles-tree') 装饰
    accountability.roles = await fetchRolesTree(payload.role, { knex: database });

    // Step 5: 获取全局权限
    // 代码依据: 第 39-42 行调用 fetchGlobalAccess
    // 函数使用 withCache('global-access') 装饰
    const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });
    
    accountability.admin = admin;
    accountability.app = app;
}
```

**关键设计决策（代码依据：`api/src/utils/get-accountability-for-token.ts:36-42`）：**

```typescript
// 代码不直接使用 JWT payload 中的 app_access/admin_access
// 而是通过 fetchGlobalAccess 重新计算
accountability.role = payload.role;
accountability.roles = await fetchRolesTree(payload.role, { knex: database });

const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });

accountability.admin = admin;
accountability.app = app;
```

**JWT 分支数据库访问行为（代码可验证）：**

| 函数 | 缓存装饰 | 内部数据库操作 |
|-----|---------|---------------|
| `verifySessionJWT` | 无 | `SELECT 1 FROM directus_sessions` |
| `fetchRolesTree` | `withCache('roles-tree')` | 查询角色继承链 |
| `fetchGlobalAccess` | `withCache('global-access')` | 查询策略权限 |

### 分支 B：静态令牌认证

`api/src/utils/get-accountability-for-token.ts:44-65`：

```typescript
} else {
    // Step 1: 数据库查询验证令牌
    // 代码依据: 第 44-51 行
    const user = await database
        .select('directus_users.id', 'directus_users.role')
        .from('directus_users')
        .where({
            'directus_users.token': token,
            status: 'active',
        })
        .first();

    if (!user) {
        throw new InvalidCredentialsError();
    }

    // Step 2: 填充基本字段
    accountability.user = user.id;
    accountability.role = user.role;

    // Step 3: 获取角色继承链
    // 代码依据: 第 59 行
    accountability.roles = await fetchRolesTree(user.role, { knex: database });

    // Step 4: 获取全局权限
    // 代码依据: 第 61-64 行
    const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });

    accountability.admin = admin;
    accountability.app = app;
}
```

**静态令牌分支数据库访问行为（代码可验证）：**

| 函数 | 缓存装饰 | 内部数据库操作 |
|-----|---------|---------------|
| directus_users 查询 | 无 | `SELECT id, role FROM directus_users WHERE token = ?` |
| `fetchRolesTree` | `withCache('roles-tree')` | 查询角色继承链 |
| `fetchGlobalAccess` | `withCache('global-access')` | 查询策略权限 |

### 分支 C：匿名请求

当 `req.token` 为 `null` 或不存在时，`accountability` 保持默认值（代码依据：`get-accountability-for-token.ts:12-18`、`66-68`）：

```typescript
// accountability 保持默认值
accountability = {
    role: null,
    user: null,
    roles: [],
    admin: false,
    app: false,
    ip: '...',
    userAgent: '...',
    origin: '...',
}
```

**匿名请求的权限行为（代码可验证）：**
- `role = null` 表示使用公共角色
- 后续访问时通过 `fetchPolicies` 获取权限
- `fetchPolicies` 在 `roles.length === 0` 时查询公共角色的策略

### 三条认证分支的函数调用链

**代码可验证的执行流程对比：**

```
分支 A (JWT):
  verifyAccessJWT()           ← 无数据库访问
  verifySessionJWT()          ← 有数据库访问（仅 session 模式）
  fetchRolesTree()            ← withCache
  fetchGlobalAccess()         ← withCache

分支 B (静态令牌):
  SELECT directus_users       ← 有数据库访问
  fetchRolesTree()            ← withCache
  fetchGlobalAccess()         ← withCache

分支 C (匿名):
  无函数调用                   ← 保持默认 accountability
```

---

## 操作追责(Accountability)体系

### Accountability 类型定义

`packages/types/src/accountability.ts:6-17`：

```typescript
type Accountability = {
    role: string | null;      // 当前角色 ID
    roles: string[];          // 角色继承链（包含所有祖先角色）
    user: string | null;      // 用户 ID
    admin: boolean;           // 是否管理员
    app: boolean;             // 是否可访问管理后台
    share?: string;           // 共享访问时的 share ID
    ip: string | null;        // 请求 IP 地址
    userAgent?: string;       // 浏览器 User-Agent
    origin?: string;          // 请求来源 Origin
    session?: string;         // 会话令牌 (session 模式)
};
```

### Accountability 注入链路

```
HTTP 请求进入
    │
    ▼
┌─────────────────────────┐
│ express.Request         │
│ (原始请求对象)           │
└───────────┬─────────────┘
            │
    extractToken 中间件
            │
            ▼
┌─────────────────────────┐
│ req.token = xxx         │
│ (提取的令牌)             │
└───────────┬─────────────┘
            │
    authenticate 中间件
            │
            ▼
┌─────────────────────────┐
│ req.accountability = {  │
│   user: 'uuid-123',     │
│   role: 'role-456',     │
│   roles: [...],         │
│   admin: false,         │
│   app: true,            │
│   ip: '192.168.1.1',    │
│   userAgent: 'Moz...',  │
│   origin: 'https://...' │
│ }                       │
└───────────┬─────────────────────────────────────────────────┐
            │                                                 │
    控制器层                                          WebSocket 层
            │                                                 │
            ▼                                                 ▼
┌─────────────────────────────────────┐       ┌─────────────────────────────┐
│  const service = new ItemsService(  │       │ websocket/authenticate.ts   │
│      collection, {                  │       │ 类似流程建立 accountability  │
│          accountability: req.       │       └─────────────────────────────┘
│            accountability,          │
│          schema: req.schema         │
│      }                              │
│  );                                 │
└───────────┬─────────────────────────┘
            │
    服务层 (ItemsService)
            │
            ▼
┌─────────────────────────────────────────────────────────────────┐
│ this.accountability = {...}                                     │
│                                                                 │
│ 用于:                                                           │
│  • processPayload()  - 权限校验                                 │
│  • validateAccess()  - 访问控制检查                             │
│  • processAst()      - 读操作权限注入                           │
│  • ActivityService   - 审计日志落库                             │
│  • PayloadService    - 预设注入（解析动态变量）                  │
└─────────────────────────────────────────────────────────────────┘
```

### Accountability 在服务层的使用场景

#### 场景 1：读操作权限注入 (processAst)

`api/src/services/items.ts:518-533`（`readByQuery` 中的流程）：

```typescript
// 1. 构建 AST（抽象语法树）
let ast = await getAstFromQuery(
    {
        collection: this.collection,
        query: updatedQuery,
        accountability: this.accountability,
    },
    {
        schema: this.schema,
        knex: this.knex,
    },
);

// 2. 处理权限（关键步骤）
ast = await processAst(
    { ast, action: 'read', accountability: this.accountability },
    { knex: this.knex, schema: this.schema },
);

// 3. 执行查询
const records = await runAst(ast, this.schema, this.accountability, {
    knex: this.knex,
    stripNonRequested: opts?.stripNonRequested !== undefined ? opts.stripNonRequested : true,
});
```

`processAst` 详细流程（`api/src/permissions/modules/process-ast/process-ast.ts:19-67`）：

```typescript
export async function processAst(options: ProcessAstOptions, context: Context) {
    // Step 1: 从 AST 提取字段映射
    const fieldMap: FieldMap = fieldMapFromAst(options.ast, context.schema);
    const collections = collectionsInFieldMap(fieldMap);

    // Step 2: 管理员短路
    // 代码依据: 第 25-32 行
    if (!options.accountability || options.accountability.admin) {
        for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
            validatePathExistence(path, collection, fields, context.schema);
        }
        return options.ast;
    }

    // Step 3: 获取策略
    // 代码依据: 第 34 行
    // fetchPolicies 使用 withCache('policies') 装饰
    const policies = await fetchPolicies(options.accountability, context);

    // Step 4: 获取权限规则
    // 代码依据: 第 36-39 行
    // fetchPermissions 内部调用 fetchRawPermissions
    // fetchRawPermissions 使用 withCache('raw-permissions') 装饰
    const permissions = await fetchPermissions(
        { action: options.action, policies, collections, accountability: options.accountability },
        context,
    );

    // Step 5: 读操作还需要额外的 read 权限
    // 代码依据: 第 41-47 行
    const readPermissions =
        options.action === 'read'
            ? permissions
            : await fetchPermissions(
                    { action: 'read', policies, collections, accountability: options.accountability },
                    context,
                );

    // Step 6: 验证字段存在性
    for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
        validatePathExistence(path, collection, fields, context.schema);
    }

    // Step 7: 验证权限
    for (const [path, { collection, fields }] of fieldMap.other.entries()) {
        validatePathPermissions(path, permissions, collection, fields);
    }
    for (const [path, { collection, fields }] of fieldMap.read.entries()) {
        validatePathPermissions(path, readPermissions, collection, fields);
    }

    // Step 8: 注入权限 CASE 条件到 AST
    injectCases(options.ast, permissions);

    return options.ast;
}
```

**processAst 数据库访问行为（代码可验证）：**

| 函数 | 缓存装饰 | 条件 | 调用 |
|-----|---------|------|------|
| `fetchPolicies` | `withCache('policies')` | 非管理员 | 总是调用 |
| `fetchPermissions` | 间接使用 `fetchRawPermissions` 的 `withCache` | 非管理员 | 总是调用 |
| `fetchPermissions` (read) | 同上 | 非管理员 + 非 read 操作 | 有条件调用 |

#### 场景 2：写操作权限校验 (processPayload)

`api/src/permissions/modules/process-payload/process-payload.ts:29-131`：

```
createOne() 调用时:
┌──────────────────────────────────────────────────────────────┐
│  processPayload({                                             │
│      accountability: this.accountability,                    │
│      action: 'create',           // 或 'read'/'update'/'delete'│
│      collection: this.collection,                             │
│      payload: payloadAfterHooks,                              │
│      nested: this.nested                                       │
│  }, context)                                                  │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│  管理员短路: if (!accountability.admin) → 跳过所有校验         │
└──────────────────────────────┬───────────────────────────────┘
                              │
┌──────────────────────────────▼───────────────────────────────┐
│  1. 获取策略与权限                                             │
├──────────────────────────────────────────────────────────────┤
│  policies = fetchPolicies(accountability, context)           │
│  fetchPolicies 使用 withCache('policies') 装饰                │
│                                                              │
│  permissions = fetchPermissions({                            │
│      action, policies, collections: [collection],            │
│      accountability                                          │
│  }, context)                                                 │
│  fetchPermissions 内部使用 fetchRawPermissions                │
│  fetchRawPermissions 使用 withCache('raw-permissions') 装饰  │
│                                                              │
│  if (permissions.length === 0)                               │
│      → throw createCollectionForbiddenError()                │
└──────────────────────────────┬───────────────────────────────┘
                              │
┌──────────────────────────────▼───────────────────────────────┐
│  2. 字段级权限检查                                             │
├──────────────────────────────────────────────────────────────┤
│  fieldsAllowed = permissions.map(p => p.fields).flat()       │
│  fieldsUsed = Object.keys(payload)                           │
│  notAllowed = difference(fieldsUsed, fieldsAllowed)          │
│                                                              │
│  if (notAllowed.length > 0 && !fieldsAllowed.includes('*'))  │
│      → throw createFieldsForbiddenError()                    │
└──────────────────────────────┬───────────────────────────────┘
                              │
┌──────────────────────────────▼───────────────────────────────┐
│  3. 预设值注入                                                 │
├──────────────────────────────────────────────────────────────┤
│  presets = permissions.map(p => p.presets)                   │
│  payloadWithPresets = assign({}, ...presets, payload)        │
│                                                              │
│  预设可包含动态变量:                                           │
│    { user_created: '$CURRENT_USER' }                         │
│    { date_created: '$NOW' }                                  │
└──────────────────────────────┬───────────────────────────────┘
                              │
┌──────────────────────────────▼───────────────────────────────┐
│  4. 验证规则校验                                               │
├──────────────────────────────────────────────────────────────┤
│  validationRules = [                                         │
│      ...fieldValidationRules,      // 字段级验证              │
│      ...permissionValidationRules  // 权限级验证              │
│  ]                                                           │
│                                                              │
│  validatePayload({ _and: validationRules }, payloadWithPresets)│
└──────────────────────────────────────────────────────────────┘
```

#### 场景 3：访问控制检查 (validateAccess)

`api/src/permissions/modules/validate-access/validate-access.ts:22-57`：

```typescript
async function validateAccess(options, context) {
    // 管理员短路
    // 代码依据: 第 28-30 行
    if (options.accountability.admin === true) {
        return;
    }

    let access: boolean;

    if (options.primaryKeys) {
        // 有具体主键 → 实际读取数据验证
        const result = await validateItemAccess(options, context);
        access = result.accessAllowed;
    } else {
        // 无主键 → 仅检查 collection+action 权限
        access = await validateCollectionAccess(options, context);
    }

    if (!access) {
        throw new ForbiddenError({ reason: '...' });
    }
}
```

`validateCollectionAccess` 实现（`api/src/permissions/modules/validate-access/lib/validate-collection-access.ts:16-25`）：

```typescript
export async function validateCollectionAccess(options, context) {
    // 获取策略
    // withCache('policies')
    const policies = await fetchPolicies(options.accountability, context);

    // 获取权限
    // 内部使用 fetchRawPermissions (withCache)
    const permissions = await fetchPermissions(
        { action: options.action, policies, collections: [options.collection], accountability: options.accountability },
        context,
    );

    return permissions.length > 0;
}
```

`validateItemAccess` 实现（`api/src/permissions/modules/validate-access/lib/validate-item-access.ts:44-172`）：

```typescript
export async function validateItemAccess(options, context) {
    // Step 1: 处理权限
    // 代码依据: 第 77 行
    // 内部调用 fetchPolicies + fetchPermissions
    await processAst({ ast, ...options }, context);

    // Step 2: 如果需要返回允许字段，额外查询
    if (options.returnAllowedRootFields) {
        // fetchAllowedFields
        // fetchPolicies + fetchPermissions（withCache）
    }

    // Step 3: 实际读取数据验证
    // 代码依据: 第 127-132 行
    // 执行带权限条件的 SELECT 查询
    const items = await fetchPermittedAstRootFields(ast, {
        schema: context.schema,
        accountability: options.accountability,
        knex: context.knex,
        action: options.action,
    });

    const expectedCount = isSingleton && !hasPrimaryKeys ? 1 : options.primaryKeys!.length;
    const hasAccess = items && items.length === expectedCount;

    return { accessAllowed: hasAccess, ... };
}
```

**validateItemAccess 关键行为（代码依据：第 63-64 行注释）：**
> "When we're looking up access to specific items, we have to read them from the database to make sure you are allowed to access them."

#### 场景 4：审计日志落库 (ActivityService)

`api/src/services/items.ts:321-375`（createOne 中的审计部分）：

```typescript
// 条件检查
// 代码依据: 第 322-326 行
if (
    opts.skipTracking !== true &&
    this.accountability &&
    this.schema.collections[this.collection]!.accountability !== null
) {
    // 1. 创建 Activity 记录
    const activityService = new ActivityService({
        knex: trx,
        schema: this.schema,
    });

    const activity = await activityService.createOne({
        action: Action.CREATE,  // 或 UPDATE/DELETE
        user: this.accountability!.user,
        collection: this.collection,
        ip: this.accountability!.ip,
        user_agent: this.accountability!.userAgent,
        origin: this.accountability!.origin,
        item: primaryKey,
    });

    // 2. 如果配置为 'all'，创建 Revision 记录
    if (this.schema.collections[this.collection]!.accountability === 'all') {
        const revisionsService = new RevisionsService({
            knex: trx,
            schema: this.schema,
        });

        const revisionPayload = await payloadService.prepareDelta(omit(payloadWithPresets, relationalFields));

        const revision = await revisionsService.createOne({
            activity: activity,
            collection: this.collection,
            item: primaryKey,
            data: revisionPayload,
            delta: revisionPayload,
        });
    }
}
```

**Accountability 字段映射到审计记录**：

| Accountability 字段 | directus_activity 字段 |
|-------------------|----------------------|
| user | user |
| ip | ip |
| userAgent | user_agent |
| origin | origin |

**审计日志的数据库操作（代码可验证）：**
- 总是执行 INSERT 语句
- 不受 `withCache` 影响

#### 场景 5：动态变量解析

权限规则和预设值中可使用动态变量，基于 accountability 解析：

| 变量 | 解析来源 | 代码行为 |
|-----|---------|---------|
| `$CURRENT_USER` | accountability.user | 直接使用 |
| `$CURRENT_ROLE` | accountability.role | 直接使用 |
| `$CURRENT_ROLES` | accountability.roles | 直接使用 |
| `$CURRENT_USER.xxx` | 关联查询用户属性 | 需要数据库查询 |
| `$NOW` | 当前时间 | 直接使用 |

---

## 端到端时序说明

### 场景：创建一条记录 (POST /items/articles)

```
时间轴 ─────────────────────────────────────────────────────────────────▶

客户端                              Directus API                          数据库
   │                                     │                                │
   │ POST /items/articles                │                                │
   │ Authorization: Bearer <jwt>         │                                │
   │ { "title": "Hello", "body": "..." } │                                │
   ├────────────────────────────────────>│                                │
   │                                     │                                │
   │                                     │ 1. cookieParser()              │
   │                                     │    → req.cookies = {...}       │
   │                                     │                                │
   │                                     │ 2. extractToken()              │
   │                                     │    → req.token = <jwt>         │
   │                                     │                                │
   │                                     │ 3. authenticate()              │
   │                                     │    ┌────────────────────────┐  │
   │                                     │    │ createDefaultAccounta- │  │
   │                                     │    │   bility({ ip: ... })  │  │
   │                                     │    └───────────┬────────────┘  │
   │                                     │                │               │
   │                                     │    ┌───────────▼────────────┐  │
   │                                     │    │ emitter.emitFilter(    │  │
   │                                     │    │   'authenticate', ...) │  │
   │                                     │    └───────────┬────────────┘  │
   │                                     │                │               │
   │                                     │    ┌───────────▼────────────┐  │
   │                                     │    │ getAccountabilityFor-  │  │
   │                                     │    │   Token(req.token)     │  │
   │                                     │    │                        │  │
   │                                     │    │ isDirectusJWT()?       │  │
   │                                     │    │ Yes → 进入 JWT 分支    │  │
   │                                     │    │                        │  │
   │                                     │    │ verifyAccessJWT()      │  │
   │                                     │    │ ── JWT 签名验证 ──     │  │
   │                                     │    │                        │  │
   │                                     │    │ payload contains       │  │
   │                                     │    │ 'session'?            │  │
   │                                     │    │ ── Yes ──              │  │
   │                                     │    │                        │  │
   │                                     │    │ verifySessionJWT()     │──┼───────────────────────────────>│
   │                                     │    │ SELECT 1 FROM          │  │                                │
   │                                     │    │   directus_sessions    │  │                                │
   │                                     │    │ WHERE token=session    │  │                                │
   │                                     │    │   AND expires >= NOW() │  │                                │
   │                                     │    │                        │  │<───────────────────────────────│
   │                                     │    │                        │  │                                │
   │                                     │    │ fetchRolesTree(role)   │──┼───────────────────────────────>│
   │                                     │    │ withCache('roles-tree')│  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ fetchGlobalAccess()    │──┼───────────────────────────────>│
   │                                     │    │ withCache('global-     │  │                                │
   │                                     │    │   access')             │  │                                │
   │                                     │    └───────────┬────────────┘  │<───────────────────────────────│
   │                                     │                │               │                                │
   │                                     │    req.accountability = {      │                                │
   │                                     │      user: 'user-uuid',        │                                │
   │                                     │      role: 'role-uuid',        │                                │
   │                                     │      roles: ['role-uuid', ...],│                                │
   │                                     │      admin: false,             │                                │
   │                                     │      app: true,                │                                │
   │                                     │      ip: '192.168.1.1',        │                                │
   │                                     │      userAgent: 'Mozilla/...', │                                │
   │                                     │      origin: 'https://...'     │                                │
   │                                     │    }                           │                                │
   │                                     │                                │                                │
   │                                     │ 4. schema 中间件               │                                │
   │                                     │    → req.schema = {...}       │                                │
   │                                     │                                │                                │
   │                                     │ 5. 路由处理: POST /:collection │                                │
   │                                     │    ┌────────────────────────┐  │                                │
   │                                     │    │ ItemsService.createOne │  │                                │
   │                                     │    │ (payload, opts)        │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ Transaction begin      │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ emitter.emitFilter(    │  │                                │
   │                                     │    │   'items.create', ...) │  │                                │
   │                                     │    │ (payload 预处理钩子)    │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ processPayload({       │  │                                │
   │                                     │    │   accountability:      │  │                                │
   │                                     │    │     req.accountability,│  │                                │
   │                                     │    │   action: 'create',    │  │                                │
   │                                     │    │   collection: 'articles'│ │                                │
   │                                     │    │ })                      │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ 管理员? → 跳过校验      │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ fetchPolicies(acc)     │──┼───────────────────────────────>│
   │                                     │    │ withCache('policies')  │  │                                │
   │                                     │    │ fetchPermissions(...)  │  │                                │
   │                                     │    │ (withCache)            │  │<───────────────────────────────│
   │                                     │    │                        │  │                                │
   │                                     │    │ 字段权限检查            │  │                                │
   │                                     │    │ 预设值注入:             │  │                                │
   │                                     │    │   { user_created:      │  │                                │
   │                                     │    │      '$CURRENT_USER',  │  │                                │
   │                                     │    │     date_created:      │  │                                │
   │                                     │    │      '$NOW' }          │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ 验证规则校验            │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ PayloadService         │  │                                │
   │                                     │    │ .processM2O/A2O/O2M()  │  │                                │
   │                                     │    │ (处理关系字段)          │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ INSERT INTO articles   │──┼───────────────────────────────>│
   │                                     │    │   (title, body,        │  │                                │
   │                                     │    │    user_created,       │  │                                │
   │                                     │    │    date_created)       │  │                                │
   │                                     │    │ RETURNING id            │  │                                │
   │                                     │    │                        │  │<───────────────────────────────│
   │                                     │    │ primaryKey = 42         │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ 审计记录?              │  │                                │
   │                                     │    │ skipTracking?          │  │                                │
   │                                     │    │ accountability?        │  │                                │
   │                                     │    │ collection.            │  │                                │
   │                                     │    │   accountability?      │  │                                │
   │                                     │    │ ── Yes, Yes, Yes ──    │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ ActivityService.       │──┼───────────────────────────────>│
   │                                     │    │   createOne({          │  │                                │
   │                                     │    │     action: 'create',  │  │                                │
   │                                     │    │     user: acc.user,    │  │                                │
   │                                     │    │     collection: 'articles', │                               │
   │                                     │    │     ip: acc.ip,        │  │                                │
   │                                     │    │     user_agent: acc.   │  │                                │
   │                                     │    │       userAgent,       │  │                                │
   │                                     │    │     origin: acc.origin,│  │                                │
   │                                     │    │     item: 42           │  │                                │
   │                                     │    │   })                   │  │                                │
   │                                     │    │                        │  │<───────────────────────────────│
   │                                     │    │ activity.id = 100      │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ accountability ===     │  │                                │
   │                                     │    │   'all'?               │  │                                │
   │                                     │    │ ── Yes ──              │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ RevisionsService.      │──┼───────────────────────────────>│
   │                                     │    │   createOne({          │  │                                │
   │                                     │    │     activity: 100,     │  │                                │
   │                                     │    │     collection: 'articles', │                               │
   │                                     │    │     item: 42,          │  │                                │
   │                                     │    │     data: { title: ... │  │                                │
   │                                     │    │     delta: { title:...│  │                                │
   │                                     │    │   })                   │  │                                │
   │                                     │    │                        │  │<───────────────────────────────│
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ Transaction commit     │──┼───────────────────────────────>│
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │<───────────────────────────────│
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ emitter.emitAction(    │  │                                │
   │                                     │    │   'items.create', ...) │  │                                │
   │                                     │    └────────────────────────┘  │                                │
   │                                     │                                │                                │
   │                                     │ 6. readOne(42) 返回结果       │                                │
   │                                     │                                │                                │
   │                                     │    ┌────────────────────────┐  │                                │
   │                                     │    │ ItemsService.readOne   │  │                                │
   │                                     │    │   (42, query)          │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ getAstFromQuery()      │  │                                │
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ processAst({           │  │                                │
   │                                     │    │   ast, action: 'read', │  │                                │
   │                                     │    │   accountability: ...  │  │                                │
   │                                     │    │ })                      │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ 管理员? → 跳过          │  │                                │
   │                                     │    │                        │  │                                │
   │                                     │    │ fetchPolicies()        │──┼───────────────────────────────>│
   │                                     │    │ fetchPermissions()     │  │                                │
   │                                     │    │ injectCases()          │  │<───────────────────────────────│
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ runAst()               │──┼───────────────────────────────>│
   │                                     │    │ SELECT * FROM articles │  │                                │
   │                                     │    │ WHERE id = 42          │  │                                │
   │                                     │    │   AND (权限条件)        │  │                                │
   │                                     │    │                        │  │<───────────────────────────────│
   │                                     │    └───────────┬────────────┘  │                                │
   │                                     │                │               │                                │
   │                                     │    ┌───────────▼────────────┐  │                                │
   │                                     │    │ emitter.emitAction(    │  │                                │
   │                                     │    │   'items.read', ...)   │  │                                │
   │                                     │    └────────────────────────┘  │                                │
   │<────────────────────────────────────┤                                │
   │ HTTP 200 OK                         │                                │
   │ {                                   │                                │
   │   "data": {                         │                                │
   │     "id": 42,                       │                                │
   │     "title": "Hello",               │                                │
   │     "user_created": "user-uuid",    │                                │
   │     "date_created": "2025-05-10..." │                                │
   │   }                                 │                                │
   │ }                                   │                                │
```

### 数据库表关联关系（审计部分）

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   directus_activity     │         │   directus_revisions    │
├─────────────────────────┤         ├─────────────────────────┤
│ id (PK)                 │◄────────│ activity (FK)           │
│ action                  │         │ id (PK)                 │
│ user (FK → users)       │         │ collection              │
│ collection              │         │ item                    │
│ ip                      │         │ data (完整快照 JSON)    │
│ user_agent              │         │ delta (变更 JSON)       │
│ origin                  │         │ parent (FK → self)      │
│ item                    │         └─────────────────────────┘
│ timestamp               │
└─────────────────────────┘
```

**审计记录触发条件**（`api/src/services/items.ts:322-326`）：

```typescript
if (
    opts.skipTracking !== true &&          // 未显式跳过
    this.accountability &&                 // 有 accountability
    this.schema.collections[this.collection]!.accountability !== null
                                           // 集合配置了审计
) {
    // 创建 activity 记录
}
```

**集合审计配置值**：

| 配置值 | 说明 |
|-------|------|
| `null` | 不记录审计 |
| `'activity'` | 仅记录 activity |
| `'all'` | 记录 activity + revisions |

---

## 关键配置项（校正后）

基于 `packages/env/src/constants/defaults.ts` 的正确默认值：

| 环境变量 | 说明 | 默认值 |
|---------|------|--------|
| `SECRET` | JWT 签名密钥（至少 32 字节） | (无，必须配置) |
| `ACCESS_TOKEN_TTL` | 访问令牌有效期 | `'15m'` |
| `REFRESH_TOKEN_TTL` | 刷新令牌有效期 | `'7d'` |
| `REFRESH_TOKEN_COOKIE_NAME` | 刷新令牌 Cookie 名称 | `'directus_refresh_token'` |
| `REFRESH_TOKEN_COOKIE_SECURE` | 刷新令牌 Cookie Secure 属性 | `false` |
| `REFRESH_TOKEN_COOKIE_SAME_SITE` | 刷新令牌 Cookie SameSite 属性 | `'lax'` |
| `SESSION_COOKIE_TTL` | 会话 Cookie 有效期 | `'1d'` (⚠️ 之前误写为 7d) |
| `SESSION_COOKIE_NAME` | 会话 Cookie 名称 | `'directus_session_token'` (⚠️ 之前误写为 directus_session) |
| `SESSION_COOKIE_SECURE` | 会话 Cookie Secure 属性 | `false` |
| `SESSION_COOKIE_SAME_SITE` | 会话 Cookie SameSite 属性 | `'lax'` |
| `SESSION_REFRESH_GRACE_PERIOD` | 会话刷新宽限期 | `'10s'` |
| `LOGIN_STALL_TIME` | 登录延迟（防暴力破解） | `500` (毫秒) |
| `REGISTER_STALL_TIME` | 注册延迟 | `750` (毫秒) |
| `AUTH_PROVIDERS` | 启用的认证提供者列表 | `''` (空，仅默认本地认证) |
| `AUTH_DISABLE_DEFAULT` | 禁用默认本地认证 | `false` |
| `IP_TRUST_PROXY` | 信任代理层级（获取真实 IP） | `true` |
| `IP_CUSTOM_HEADER` | 自定义 IP 头 | `false` |

### 配置更正说明

| 错误项 | 之前错误值 | 正确值（来自 defaults.ts） |
|-------|-----------|--------------------------|
| SESSION_COOKIE_TTL | `'7d'` | `'1d'` |
| SESSION_COOKIE_NAME | `'directus_session'` | `'directus_session_token'` |
| SESSION_COOKIE_SAME_SITE | `'strict'` | `'lax'` |

---

## 核心文件索引

| 功能 | 文件路径 |
|-----|---------|
| 认证服务 (登录/刷新/登出) | `api/src/services/authentication.ts` |
| 认证中间件 | `api/src/middleware/authenticate.ts` |
| 令牌提取中间件 | `api/src/middleware/extract-token.ts` |
| 本地认证驱动 | `api/src/auth/drivers/local.ts` |
| 认证驱动抽象基类 | `api/src/auth/auth.ts` |
| Accountability 类型 | `packages/types/src/accountability.ts` |
| 从令牌获取 Accountability | `api/src/utils/get-accountability-for-token.ts` |
| JWT 验证 | `api/src/utils/jwt.ts` |
| 会话 JWT 验证 | `api/src/utils/verify-session-jwt.ts` |
| 默认 Accountability 创建 | `api/src/permissions/utils/create-default-accountability.ts` |
| 权限校验 (processPayload) | `api/src/permissions/modules/process-payload/process-payload.ts` |
| 访问控制 (validateAccess) | `api/src/permissions/modules/validate-access/validate-access.ts` |
| 读操作权限注入 (processAst) | `api/src/permissions/modules/process-ast/process-ast.ts` |
| 缓存装饰器 | `api/src/permissions/utils/with-cache.ts` |
| 获取角色树 | `api/src/permissions/lib/fetch-roles-tree.ts` |
| 获取全局权限 | `api/src/permissions/modules/fetch-global-access/fetch-global-access.ts` |
| 获取策略 | `api/src/permissions/lib/fetch-policies.ts` |
| 获取权限规则 | `api/src/permissions/utils/fetch-raw-permissions.ts` |
| 认证控制器路由 | `api/src/controllers/auth.ts` |
| 数据操作服务 | `api/src/services/items.ts` |
| 活动日志服务 | `api/src/services/activity.ts` |
| 数据修订服务 | `api/src/services/revisions.ts` |
| 默认环境变量值 | `packages/env/src/constants/defaults.ts` |
| 环境变量定义 | `packages/env/src/constants/directus-variables.ts` |
| 应用中间件注册 | `api/src/app.ts` |
| 登录延迟工具 | `api/src/utils/stall.ts` |
