# Directus 身份认证体系分析

## 目录

1. [整体架构概览](#整体架构概览)
2. [登录流程](#登录流程)
3. [访问令牌颁发机制](#访问令牌颁发机制)
4. [会话管理与令牌续签](#会话管理与令牌续签)
5. [请求认证中间件链](#请求认证中间件链)
6. [操作追责(Accountability)体系](#操作追责accountability体系)
7. [关键配置项](#关键配置项)

---

## 整体架构概览

Directus 的身份认证体系由以下核心组件构成：

```
请求进入
    ↓
┌─────────────────────────────┐
│  extractToken 中间件        │  ← 从请求中提取令牌
│  (api/src/middleware/      │
│   extract-token.ts)        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│  authenticate 中间件        │  ← 验证令牌并建立 accountability
│  (api/src/middleware/      │
│   authenticate.ts)         │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│  业务路由 / 服务层           │  ← accountability 注入请求上下文
│  (req.accountability)      │
└─────────────────────────────┘
```

### 认证提供者系统

Directus 支持多种认证方式，通过可插拔的 AuthDriver 模式实现：

| 认证驱动 | 文件位置 | 说明 |
|---------|---------|------|
| Local | `api/src/auth/drivers/local.ts` | 邮箱/密码认证 |
| OAuth2 | `api/src/auth/drivers/oauth2.ts` | OAuth 2.0 协议 |
| OpenID | `api/src/auth/drivers/openid.ts` | OpenID Connect 协议 |
| LDAP | `api/src/auth/drivers/ldap.ts` | LDAP 目录服务 |
| SAML | `api/src/auth/drivers/saml.ts` | SAML 2.0 协议 |

抽象基类定义在 `api/src/auth/auth.ts`，提供以下生命周期方法：

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

## 登录流程

### 入口路由

登录请求通过 `POST /auth/login` 端点处理，路由定义在 `api/src/controllers/auth.ts:60-64`：

```typescript
// 默认本地认证
router.use('/login', createLocalAuthRouter(DEFAULT_AUTH_PROVIDER));

// 其他认证提供者
router.use(`/login/${authProvider.name}`, authRouter);
```

### 本地登录路由

`api/src/auth/drivers/local.ts:48-119` 定义了本地认证的具体实现：

1. **请求验证**：使用 Joi Schema 验证 `email` 和 `password` 字段
2. **创建 Accountability**：从请求中提取 IP、User-Agent、Origin 等上下文信息
3. **调用 AuthenticationService**：执行实际的登录逻辑
4. **返回模式支持**：

   | 模式 (mode) | 说明 | refresh_token 存储 | access_token 返回 |
   |------------|------|-------------------|------------------|
   | `json` | 纯 API 模式 | 响应体中 | 响应体中 |
   | `cookie` | Cookie 模式 | HttpOnly Cookie | 响应体中 |
   | `session` | 有状态会话 | 不返回 | HttpOnly Cookie (作为 JWT) |

### AuthenticationService.login() 详解

核心登录逻辑在 `api/src/services/authentication.ts:55-320`，流程如下：

```
1. 认证提供者验证
   ├─ provider.getUserID(payload)  → 通过邮箱查找用户
   └─ 校验用户状态 (active) 和 provider 匹配

2. 登录尝试速率限制
   ├─ 读取 auth_login_attempts 配置
   └─ 超过限制则将用户状态设为 suspended

3. 密码验证
   └─ provider.login(user, payload)  → LocalAuthDriver 使用 argon2 验证

4. 双因素认证 (TFA)
   ├─ 检查 user.tfa_secret 是否存在
   └─ TFAService.verifyOTP() 验证一次性密码

5. 权限计算
   ├─ fetchRolesTree() → 获取角色继承链
   └─ fetchGlobalAccess() → 计算 app_access / admin_access

6. 令牌生成
   ├─ 生成 refreshToken (nanoid 64位随机字符串)
   ├─ 构建 JWT Payload (DirectusTokenPayload)
   └─ jwt.sign() 使用 SECRET 签名

7. 会话创建
   └─ 插入 directus_sessions 表记录

8. 审计日志
   ├─ ActivityService 记录 LOGIN 动作
   └─ 更新 user.last_access 时间戳

9. 事件触发
   └─ emitter.emitAction('auth.login', ...)
```

---

## 访问令牌颁发机制

### DirectusTokenPayload 结构

JWT 访问令牌的 payload 定义在 `api/src/types/auth.ts`：

```typescript
interface DirectusTokenPayload {
    id: string;                    // 用户 ID
    role: string;                  // 角色 ID
    app_access: boolean;           // 是否可访问管理后台
    admin_access: boolean;         // 是否管理员权限
    enforce_tfa?: boolean;         // 是否强制启用 2FA
    session?: string;              // 有状态会话模式下的刷新令牌
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

### 令牌有效期配置

| 配置项 | 环境变量 | 默认值 | 应用场景 |
|-------|---------|--------|---------|
| 访问令牌 TTL | `ACCESS_TOKEN_TTL` | 15m | 普通 API 访问 |
| 会话 Cookie TTL | `SESSION_COOKIE_TTL` | 7d | 有状态会话模式 |
| 刷新令牌 TTL | `REFRESH_TOKEN_TTL` | 7d | 令牌刷新 |

---

## 会话管理与令牌续签

### directus_sessions 表结构

会话信息存储在数据库中，核心字段：

| 字段 | 类型 | 说明 |
|-----|------|------|
| token | string(64) | 刷新令牌（会话标识） |
| user | string | 关联用户 ID |
| share | string | 关联共享 ID（可选） |
| expires | datetime | 过期时间 |
| next_token | string(64) | 下一刷新令牌（用于优雅过渡） |
| ip | string | 登录 IP |
| user_agent | text | 浏览器 User-Agent |
| origin | string | 请求来源 |

### 令牌刷新流程

`POST /auth/refresh` 端点处理，核心逻辑在 `api/src/services/authentication.ts:322-480`：

```
1. 验证刷新令牌
   ├─ 查询 directus_sessions 表
   ├─ 检查 expires >= now
   └─ 校验用户状态 (active)

2. 权限重计算
   ├─ fetchRolesTree()
   └─ fetchGlobalAccess()

3. 生成新令牌
   ├─ 有状态会话 (session mode):
   │   ├─ updateStatefulSession() 处理优雅过渡
   │   └─ 旧令牌保留 SESSION_REFRESH_GRACE_PERIOD (默认10秒)
   │
   └─ 无状态模式:
       └─ 直接更新当前会话记录的 token 和 expires

4. 清理过期会话
   └─ 删除当前用户的所有过期会话
```

### 有状态会话的优雅过渡

`api/src/services/authentication.ts:482-538` 实现的 `updateStatefulSession()` 方法：

**问题场景**：并发请求时，一个请求正在刷新令牌，另一个请求使用旧令牌可能失败。

**解决方案**：

```
刷新前: ┌─────────────────────┐
        │  token: old_token   │
        │  expires: future    │
        │  next_token: null   │
        └─────────────────────┘

刷新中: ┌─────────────────────┐    ┌─────────────────────┐
        │  token: old_token   │◄───│  token: new_token   │
        │  expires: +10s      │    │  expires: +7d       │
        │  next_token: new_   │    │  next_token: null   │
        └─────────────────────┘    └─────────────────────┘

旧令牌宽限期 (SESSION_REFRESH_GRACE_PERIOD, 默认 10s) 内仍有效
```

### 登出流程

`api/src/services/authentication.ts:540-570`：

1. 查找会话记录
2. 调用 `provider.logout()` 钩子
3. 记录 LOGOUT 活动日志
4. 删除 directus_sessions 记录
5. 清除相关 Cookie

---

## 请求认证中间件链

### 中间件注册顺序

`api/src/app.ts:245-307` 定义的中间件链：

```typescript
app.use(cookieParser());       // 1. 解析 Cookie
app.use(extractToken);         // 2. 提取令牌
// ... 速率限制中间件 ...
app.use(authenticate);         // 3. 认证并建立 accountability
app.use(schema);               // 4. 加载数据库 schema
app.use(sanitizeQuery);        // 5. 清理查询参数
// ... 路由 ...
```

### 1. extractToken 中间件

`api/src/middleware/extract-token.ts` 负责从多个来源提取令牌：

**提取优先级**：

1. **Query 参数**: `?access_token=xxx`
2. **Authorization Header**: `Authorization: Bearer xxx`
3. **Session Cookie**: `directus_session` (或配置的名称)

**RFC 6750 合规性**：
- 不允许同时使用 query 参数和 Authorization Header 传递令牌
- Session Cookie 作为例外，允许与其他方式共存（支持 Data Studio 内的自定义令牌）

### 2. authenticate 中间件

`api/src/middleware/authenticate.ts` 是认证的核心：

```
1. 创建默认 Accountability
   ├─ IP 地址 (getIPFromReq)
   ├─ User-Agent (截断到 1024 字符)
   └─ Origin

2. 自定义认证钩子
   └─ emitter.emitFilter('authenticate', ...)
      允许扩展完全接管认证逻辑

3. 令牌验证
   └─ getAccountabilityForToken(req.token, defaultAccountability)

4. 无效令牌处理
   └─ 若是 Session Cookie 中的无效令牌，自动清除 Cookie
```

### 3. getAccountabilityForToken 工具函数

`api/src/utils/get-accountability-for-token.ts:12-69`：

```typescript
async function getAccountabilityForToken(
    token?: string | null,
    accountability?: Accountability,
): Promise<Accountability>
```

**两种令牌类型的处理**：

#### A. JWT 访问令牌 (isDirectusJWT)

```
1. verifyAccessJWT() 验证 JWT 签名和必需字段
2. 若包含 session 字段 → verifySessionJWT() 检查会话有效性
3. 填充 accountability:
   ├─ user = payload.id
   ├─ role = payload.role
   ├─ roles = fetchRolesTree(role)  ← 角色继承链
   ├─ admin/app = fetchGlobalAccess() ← 重新计算权限
   └─ session/share = 从 payload 复制
```

#### B. 静态令牌 (directus_users.token)

```
1. 查询 directus_users 表，条件:
   ├─ token = 传入令牌
   └─ status = 'active'

2. 同 JWT 方式填充 accountability
```

**关键设计决策**：每次请求都重新计算 `roles` 继承链和 `admin/app` 权限，确保权限变更立即生效，而不是依赖 JWT 中的缓存值。

---

## 操作追责(Accountability)体系

### Accountability 类型定义

`packages/types/src/accountability.ts:6-17`：

```typescript
type Accountability = {
    role: string | null;      // 当前角色
    roles: string[];          // 角色继承链（包含所有祖先）
    user: string | null;      // 用户 ID
    admin: boolean;           // 是否有管理员权限
    app: boolean;             // 是否可访问管理后台
    share?: string;           // 共享访问时的 share ID
    ip: string | null;        // 请求 IP 地址
    userAgent?: string;       // 浏览器 User-Agent
    origin?: string;          // 请求来源
    session?: string;         // 会话令牌
};
```

### 默认 Accountability 创建

`api/src/permissions/utils/create-default-accountability.ts`：

```typescript
function createDefaultAccountability(overrides?: Partial<Accountability>): Accountability {
    return {
        role: null,
        user: null,
        roles: [],
        admin: false,
        app: false,
        ip: null,
        ...overrides,
    };
}
```

**未认证请求**的 accountability 即为默认值（公共/访客权限）。

### Accountability 在服务层的注入

所有服务都继承自 `AbstractServiceOptions`，在构造时接收 accountability：

`api/src/services/items.ts:53-56`：

```typescript
constructor(collection: Collection, options: AbstractServiceOptions) {
    this.collection = collection;
    this.knex = options.knex || getDatabase();
    this.accountability = options.accountability || null;
    // ...
}
```

### Accountability 的使用场景

#### 1. 权限检查

权限系统基于 `accountability.role` 和 `accountability.roles` 进行访问控制。

#### 2. 活动日志 (Activity)

登录/登出等操作记录活动日志时使用 accountability 中的上下文信息：

`api/src/services/authentication.ts:292-302`：

```typescript
await this.activityService.createOne({
    action: Action.LOGIN,
    user: user.id,
    ip: this.accountability.ip,
    user_agent: this.accountability.userAgent,
    origin: this.accountability.origin,
    collection: 'directus_users',
    item: user.id,
});
```

#### 3. 数据修订 (Revisions)

记录数据变更时， accountability 信息用于追踪谁在什么环境下做了修改。

#### 4. Payload 处理

创建/更新数据时，自动注入创建者/修改者信息：

`api/src/services/items.ts:172-176`：

```typescript
const payloadWithPresets = this.accountability
    ? await processPayload({
          accountability: this.accountability,
          action: 'create',
          collection: this.collection,
          // ...
      })
    : payload;
```

### 自定义 Accountability 扩展点

`api/src/middleware/authenticate.ts:30-46` 提供的 `authenticate` filter 钩子：

```typescript
const customAccountability = await emitter.emitFilter(
    'authenticate',
    defaultAccountability,
    { req },
    { database, schema: null, accountability: null },
);

if (customAccountability !== defaultAccountability) {
    req.accountability = customAccountability;
    return next();  // 跳过默认认证逻辑
}
```

允许扩展完全自定义认证和 accountability 建立逻辑。

---

## 关键配置项

| 环境变量 | 说明 | 默认值 |
|---------|------|--------|
| `SECRET` | JWT 签名密钥 | (必须配置) |
| `ACCESS_TOKEN_TTL` | 访问令牌有效期 | `15m` |
| `REFRESH_TOKEN_TTL` | 刷新令牌有效期 | `7d` |
| `SESSION_COOKIE_TTL` | 会话 Cookie 有效期 | `7d` |
| `SESSION_COOKIE_NAME` | 会话 Cookie 名称 | `directus_session` |
| `REFRESH_TOKEN_COOKIE_NAME` | 刷新令牌 Cookie 名称 | `directus_refresh_token` |
| `SESSION_REFRESH_GRACE_PERIOD` | 会话刷新宽限期 | `10s` |
| `LOGIN_STALL_TIME` | 登录失败延迟 (防暴力破解) | `500` (ms) |
| `AUTH_LOGIN_ATTEMPTS` | 允许的登录失败次数 | `null` (不限制) |
| `AUTH_DISABLE_DEFAULT` | 禁用默认本地认证 | `false` |
| `IP_TRUST_PROXY` | 信任的代理层级 (用于获取真实 IP) | |

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
| 认证控制器路由 | `api/src/controllers/auth.ts` |
| 应用中间件注册 | `api/src/app.ts` |
