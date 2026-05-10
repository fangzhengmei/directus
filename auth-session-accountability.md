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
│    → 通过邮箱查询 directus_users 表（数据库查询 #1）           │
│    → 校验用户状态 status === 'active'                         │
│    → 校验用户 provider 与请求 provider 匹配                    │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     2. 登录尝试速率限制                       │
├──────────────────────────────────────────────────────────────┤
│  读取 auth_login_attempts 配置（SettingsService，数据库查询 #2）│
│  超过限制则：                                                  │
│    → 将用户 status 更新为 'suspended'（数据库写入 #1）          │
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
│    （纯内存计算，无数据库访问）                                 │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     4. 双因素认证 (TFA)                       │
├──────────────────────────────────────────────────────────────┤
│  检查 user.tfa_secret 是否存在（来自第 1 步查询结果）           │
│  存在则要求请求包含 options.otp                               │
│  TFAService.verifyOTP(user.id, otp) 验证一次性密码            │
│  （可能涉及数据库查询，取决于 TFA 实现）                        │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     5. 权限计算                               │
├──────────────────────────────────────────────────────────────┤
│  fetchRolesTree(user.role)                                   │
│    → 递归获取角色继承链（数据库查询 #3，带缓存）                 │
│                                                              │
│  fetchGlobalAccess({ roles, user, ip })                      │
│    → 基于策略(Policies)计算 app_access / admin_access        │
│    → 数据库查询 #4-#6（带缓存）：                              │
│      • fetchGlobalAccessForRoles: 查 directus_access + policies │
│      • fetchGlobalAccessForUser: 查 directus_access + policies │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     6. 令牌生成                               │
├──────────────────────────────────────────────────────────────┤
│  生成 refreshToken: nanoid(64)（纯内存）                       │
│  构建 DirectusTokenPayload:                                  │
│    { id, role, app_access, admin_access, enforce_tfa? }      │
│                                                              │
│  emitter.emitFilter('auth.jwt', ...)                         │
│    → 允许扩展自定义 JWT Claims（可能涉及数据库，取决于扩展）     │
│                                                              │
│  jwt.sign(customClaims, getSecret(), {                       │
│    expiresIn: TTL,                                           │
│    issuer: 'directus'                                        │
│  }) （纯内存计算）                                             │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     7. 会话创建                               │
├──────────────────────────────────────────────────────────────┤
│  INSERT INTO directus_sessions（数据库写入 #2）                │
│    (token, user, expires, ip, user_agent, origin)            │
│                                                              │
│  清理：DELETE FROM directus_sessions WHERE expires < NOW()    │
│  （数据库写入 #3）                                             │
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
│  }) （数据库写入 #4）                                          │
│                                                              │
│  UPDATE directus_users SET last_access = NOW()               │
│  （数据库写入 #5）                                             │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                     9. 事件触发                               │
├──────────────────────────────────────────────────────────────┤
│  emitter.emitAction('auth.login', {                          │
│    payload, status: 'success',                               │
│    user: user.id, provider                                   │
│  }) （纯内存，事件分发）                                        │
│                                                              │
│  重置登录尝试计数器：loginAttemptsLimiter.set(user.id, 0, 0)  │
│  （内存操作，速率限制器）                                       │
└──────────────────────────────────────────────────────────────┘
```

**登录流程数据库访问统计**：

| 阶段 | 操作类型 | 说明 |
|-----|---------|------|
| 1. 认证提供者验证 | SELECT | 查询用户信息 |
| 2. 速率限制配置 | SELECT | 查询 settings |
| 2. 超过限制时 | UPDATE + INSERT | 禁用用户 + 记录审计 |
| 5. 权限计算 | SELECT × 3+ | 角色树 + 策略权限 |
| 7. 会话创建 | INSERT + DELETE | 创建会话 + 清理过期 |
| 8. 审计日志 | INSERT + UPDATE | 记录 Activity + 更新 last_access |

> **注意**：登录流程不依赖权限缓存（`withCache`），因为登录是低频操作且需要最新数据。

### 防暴力破解机制

`api/src/utils/stall.ts` 实现的登录延迟机制：

```typescript
// 无论登录成功或失败，都确保至少消耗 STALL_TIME
const STALL_TIME = env['LOGIN_STALL_TIME'] as number;  // 默认 500ms
const timeStart = performance.now();

// ... 登录逻辑 ...

await stall(STALL_TIME, timeStart);  // 补足剩余时间
```

**作用**：防止攻击者通过响应时间差异推断用户是否存在。

---

## 访问令牌颁发机制

### DirectusTokenPayload 结构

JWT 访问令牌的 payload 定义在 `api/src/types/auth.ts`：

```typescript
interface DirectusTokenPayload {
    id: string;                    // 用户 ID (directus_users.id)
    role: string;                  // 角色 ID (directus_roles.id)
    app_access: boolean;           // 是否可访问管理后台
    admin_access: boolean;         // 是否管理员权限（跳过所有权限检查）
    enforce_tfa?: boolean;         // 是否强制启用 2FA（角色策略要求）
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

允许扩展向 JWT 中注入自定义 Claims。

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
│                                                              │
│  控制器位置: api/src/controllers/auth.ts:70-101              │
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
│  （数据库查询 #1）                                             │
│                                                              │
│  用户状态检查：                                               │
│    - u.status !== 'active' → 删除会话（数据库写入 #1）         │
│    - u.status === 'suspended' → 抛 UserSuspendedError        │
│    - 其他非 active 状态 → 抛 InvalidCredentialsError          │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                 3. 权限重计算                                 │
├──────────────────────────────────────────────────────────────┤
│  fetchRolesTree(record.user_role)                            │
│    → 数据库查询 #2（带缓存）                                   │
│                                                              │
│  fetchGlobalAccess({ user, roles, ip })                      │
│    → 数据库查询 #3-#5（带缓存）                                │
│                                                              │
│  ⚠️ 关键设计：每次刷新都重新计算权限，确保权限变更立即生效      │
│                                                              │
│  外部认证提供者钩子：                                          │
│    provider.refresh(user)  → 允许 OAuth/LDAP 等校验外部状态   │
│    （可能涉及外部服务调用）                                     │
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
│  │ (数据库写入 #2-#3)  │        │ (数据库写入 #2)     │      │
│  └─────────────────────┘        └─────────────────────┘      │
│                                                              │
└──────────────────────────────┬───────────────────────────────┘
                               │
┌──────────────────────────────▼───────────────────────────────┐
│                 5. 生成新访问令牌                             │
├──────────────────────────────────────────────────────────────┤
│  构建新的 DirectusTokenPayload（纯内存）                        │
│  emitter.emitFilter('auth.jwt', ..., { type: 'refresh' })    │
│  jwt.sign(...) 签发新的 access_token（纯内存）                  │
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
│  （数据库写入 #3 或 #4）                                       │
│                                                              │
│  DELETE FROM directus_sessions                               │
│  WHERE user = record.user_id                                 │
│    AND share = record.share_id                               │
│    AND expires < NOW()                                       │
│  （数据库写入 #4 或 #5）                                       │
└──────────────────────────────────────────────────────────────┘
```

### 有状态会话的优雅过渡机制

`api/src/services/authentication.ts:482-538` 实现的 `updateStatefulSession()` 方法，解决**并发刷新问题**：

**问题场景**：
- 浏览器并发发起多个请求
- 请求 A 正在刷新令牌
- 请求 B 使用旧令牌可能失败

**解决方案 - 双令牌宽限期**：

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

**核心逻辑**（数据库操作）：

```typescript
// 1. 检查是否已有 next_token（防止重复刷新）
// SELECT next_token FROM directus_sessions WHERE token = oldSessionToken
// （数据库查询 #1）

// 2. 更新旧会话：缩短过期时间 + 指向新令牌
UPDATE directus_sessions
SET next_token = newSessionToken,
    expires = NOW() + GRACE_PERIOD  -- 默认 10s
WHERE token = oldSessionToken
  AND next_token IS NULL;
（数据库写入 #1）

// 3. 如果更新失败（说明已被并发刷新），查询现有 next_token
if (updatedSession.length === 0) {
    SELECT next_token FROM directus_sessions WHERE token = oldSessionToken
    // （数据库查询 #2）
    return next_token;
}

// 4. 创建新的会话记录
INSERT INTO directus_sessions (token, user, share, expires, ip, user_agent, origin)
VALUES (newSessionToken, ...)
（数据库写入 #2）
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
    // （数据库查询 #1）

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
            // （数据库写入 #1）
        }

        // 4. 删除会话记录
        await this.knex.delete().from('directus_sessions').where('token', refreshToken);
        // （数据库写入 #2）
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

**提取优先级**（互不覆盖，RFC 6750 合规）：

```
优先级 1: Query 参数
   ?access_token=eyJhbGciOiJIUzI1NiIs...

优先级 2: Authorization Header
   Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

优先级 3: Session Cookie (仅当前面都没有时)
   Cookie: directus_session_token=eyJhbGciOiJIUzI1NiIs...
```

**RFC 6750 合规性**：
- ❌ 不允许同时使用 query 参数 + Authorization Header
- ✅ 允许 Session Cookie 与其他方式共存（特殊例外）

**特殊例外的原因**（注释说明）：
- Data Studio 内 WYSIWYG 编辑器和扩展可能使用自定义令牌
- 外部应用在同域下运行时可能使用不同认证方式

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
│  （纯内存操作）                                                │
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
│       │    │  使用公共/访客权限                     │         │
│       │    │  （纯内存，无数据库访问）               │         │
│       │    └───────────────────────────────────────┘         │
│       │                                                       │
│  ┌────▼─────────────────────────────────────────────────┐    │
│  │  isDirectusJWT(req.token)?                           │    │
│  │  jwt.decode + 检查 iss === 'directus'                │    │
│  │  （纯内存操作）                                        │    │
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

**重要修正**：JWT 认证分支**不仅**做 JWT 验证，还会实时计算角色继承和全局权限。

`api/src/utils/get-accountability-for-token.ts:24-43` 的完整实现：

```typescript
if (isDirectusJWT(token)) {
    // Step 1: JWT 签名验证（纯内存，无数据库访问）
    const payload = verifyAccessJWT(token, getSecret());
    // → jwt.verify() with issuer='directus'
    // → 检查必需字段: role, app_access, admin_access

    // Step 2: 会话有效性验证（仅 session 模式，数据库查询 #1）
    if ('session' in payload) {
        await verifySessionJWT(payload);
        // SELECT 1 FROM directus_sessions
        // WHERE token = payload.session
        //   AND user = payload.id
        //   AND expires >= NOW()
        
        accountability.session = payload.session;
    }

    // Step 3: 填充基本字段（纯内存）
    if (payload.share) accountability.share = payload.share;
    if (payload.id) accountability.user = payload.id;
    accountability.role = payload.role;

    // Step 4: ⚠️ 实时计算角色继承链（数据库查询 #2，带缓存）
    accountability.roles = await fetchRolesTree(payload.role, { knex: database });
    // → 递归查询 directus_roles 表获取所有祖先角色

    // Step 5: ⚠️ 实时计算全局权限（数据库查询 #3-#5，带缓存）
    const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });
    // → fetchGlobalAccessForRoles: 查询 directus_access + directus_policies
    // → fetchGlobalAccessForUser: 查询 directus_access + directus_policies
    
    accountability.admin = admin;
    accountability.app = app;
}
```

**关键设计决策**（代码依据：`api/src/utils/get-accountability-for-token.ts:37-42`）：

```typescript
// 不使用 JWT 中的 app_access/admin_access，而是实时计算
accountability.role = payload.role;
accountability.roles = await fetchRolesTree(payload.role, { knex: database });

const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });

accountability.admin = admin;
accountability.app = app;
```

**为什么要实时计算？**
- JWT 中的 `app_access`/`admin_access` 可能已过期（用户权限被修改）
- 每次请求重新计算确保权限变更立即生效
- 虽然增加了数据库访问，但通过 `withCache` 缓存机制缓解性能影响

**性能优化**：
- `fetchRolesTree` 使用 `withCache('roles-tree', ...)` 缓存
- `fetchGlobalAccess` 使用 `withCache('global-access', ...)` 缓存
- 缓存 key 基于参数 hash，同一用户/角色的重复请求命中缓存

**JWT 分支数据库访问统计**：

| 条件 | 数据库查询次数 | 说明 |
|-----|--------------|------|
| 非 session 模式 + 缓存命中 | 0 | 纯内存操作 |
| 非 session 模式 + 缓存未命中 | 2+ | 角色树 + 权限策略 |
| session 模式 + 缓存命中 | 1 | 会话验证查询 |
| session 模式 + 缓存未命中 | 3+ | 会话验证 + 角色树 + 权限策略 |

**静态令牌 vs JWT 对比（修正版）**：

| 特性 | JWT 访问令牌 | 静态令牌 (directus_users.token) |
|-----|-------------|-------------------------------|
| 存储位置 | 客户端 | 数据库 directus_users 表 |
| 过期时间 | 有 (ACCESS_TOKEN_TTL) | 无（永久有效，除非手动重置） |
| 初始验证 | JWT 签名验证（内存） | 数据库查询用户 |
| 权限计算 | ⚠️ 实时计算（可能查库，带缓存） | ⚠️ 实时计算（可能查库，带缓存） |
| 性能 | 取决于缓存命中率 | 取决于缓存命中率 |
| 适用场景 | 用户登录会话 | 服务端 API 集成、自动化脚本 |
| 安全风险 | 泄露后短期有效 | 泄露后长期有效（需立即重置） |

> **重要修正**：之前的对比表错误地认为 JWT "性能高（仅 JWT 验证）"。实际上 JWT 和静态令牌**都会**实时计算权限，区别仅在于初始验证方式不同。两者都依赖相同的缓存机制。

### 分支 B：静态令牌认证

`api/src/utils/get-accountability-for-token.ts:44-65`：

```typescript
} else {
    // Step 1: 数据库查询验证令牌（数据库查询 #1）
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

    // Step 2: 填充基本字段（纯内存）
    accountability.user = user.id;
    accountability.role = user.role;

    // Step 3: ⚠️ 实时计算角色继承链（数据库查询 #2，带缓存）
    accountability.roles = await fetchRolesTree(user.role, { knex: database });

    // Step 4: ⚠️ 实时计算全局权限（数据库查询 #3-#5，带缓存）
    const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });

    accountability.admin = admin;
    accountability.app = app;
}
```

**静态令牌分支数据库访问统计**：

| 条件 | 数据库查询次数 | 说明 |
|-----|--------------|------|
| 缓存命中 | 1 | 用户令牌验证查询 |
| 缓存未命中 | 3+ | 用户令牌 + 角色树 + 权限策略 |

### 分支 C：匿名请求

当 `req.token` 为 `null` 或不存在时：

```typescript
// accountability 保持默认值（纯内存，无数据库访问）
accountability = {
    role: null,      // 无角色
    user: null,      // 无用户
    roles: [],       // 空角色链
    admin: false,    // 非管理员
    app: false,      // 不可访问管理后台
    ip: '...',       // 仍有 IP 信息
    userAgent: '...',// 仍有 User-Agent
    origin: '...',   // 仍有 Origin
}
```

**匿名请求的权限**：
- 依赖公共角色（Public Role）的权限配置
- 后续访问时会通过 `processAst`/`validateAccess` 检查权限
- 权限检查涉及数据库查询（但使用缓存）

### 三条认证分支性能对比总结

| 分支 | 数据库查询（缓存命中） | 数据库查询（缓存未命中） | 内存操作 |
|-----|---------------------|---------------------|---------|
| JWT（非 session） | 0 | 2+ | JWT 签名验证 |
| JWT（session） | 1 | 3+ | JWT 签名验证 |
| 静态令牌 | 1 | 3+ | 无 |
| 匿名请求 | 0 | 0 | 无 |

> **缓存机制说明**：所有权限相关查询（`fetchRolesTree`、`fetchGlobalAccess`、`fetchPolicies`、`fetchPermissions`）都通过 `withCache` 装饰器缓存。缓存 key 基于参数 hash，同一用户/角色的重复请求会命中缓存。

---

## 操作追责(Accountability)体系

### Accountability 类型定义

`packages/types/src/accountability.ts:6-17`：

```typescript
type Accountability = {
    role: string | null;      // 当前角色 ID
    roles: string[];          // 角色继承链（包含所有祖先角色）
    user: string | null;      // 用户 ID
    admin: boolean;           // 是否管理员（跳过所有权限检查）
    app: boolean;             // 是否可访问管理后台
    share?: string;           // 共享访问时的 share ID
    ip: string | null;        // 请求 IP 地址
    userAgent?: string;       // 浏览器 User-Agent (截断到 1024 字符)
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
│   role: 'role-