# Directus 认证体系深度分析

## 1. 概述

Directus 采用了基于 JWT (JSON Web Token) 的无状态认证机制，结合了灵活的会话管理系统。本文档将深入分析其认证体系的三个核心方面：

- 登录后会话建立机制
- 访问令牌的生成与刷新机制
- 操作者标识 (Accountability) 在多个服务层之间的传递

## 2. 登录后会话建立机制

### 2.1 登录流程

Directus 的登录流程由 `AuthenticationService` 服务主导，通过多种认证驱动程序支持不同的认证方式。

**核心登录流程：**

1. **请求入口**：用户通过 `/auth/login` 端点发起登录请求
2. **认证驱动选择**：根据配置的认证提供者选择相应的驱动程序
3. **用户验证**：
   - 本地认证：验证邮箱和密码
   - 外部认证：OAuth2、OpenID、LDAP、SAML 等
4. **权限检查**：验证用户状态和角色权限
5. **令牌生成**：创建访问令牌和刷新令牌
6. **会话记录**：在数据库中记录会话信息

### 2.2 认证驱动架构

Directus 采用了驱动程序模式来支持多种认证方式：

```
┌─────────────────┐
│  AuthDriver     │ ← 抽象基类
│  (抽象基类)      │
└────────┬────────┘
         │ 继承
    ┌────┴────┬────────┬────────┬────────┐
    ▼         ▼        ▼        ▼        ▼
┌────────┐ ┌────────┐ ┌──────┐ ┌──────┐ ┌──────┐
│Local   │ │OAuth2  │ │OpenID│ │LDAP  │ │SAML  │
│Auth    │ │Auth    │ │Auth  │ │Auth  │ │Auth  │
└────────┘ └────────┘ └──────┘ └──────┘ └──────┘
```

**关键代码分析**：

- **抽象基类** (`api/src/auth/auth.ts`)：定义了所有认证驱动必须实现的接口
  - `getUserID()`: 根据用户提供的凭据获取用户 ID
  - `verify()`: 验证用户密码
  - `login()`: 处理用户登录逻辑
  - `refresh()`: 处理会话刷新
  - `logout()`: 处理用户登出

- **本地认证驱动** (`api/src/auth/drivers/local.ts`)：
  - 使用 `argon2` 进行密码哈希验证
  - 支持双因素认证 (2FA)
  - 提供登录路由创建功能

### 2.3 会话建立的详细过程

当用户成功登录后，系统会执行以下步骤建立会话：

1. **创建认证服务实例**：
   ```typescript
   const authenticationService = new AuthenticationService({
       accountability: accountability,
       schema: req.schema,
   });
   ```
   *来源: `api/src/auth/drivers/local.ts:76-79`*

2. **生成令牌对**：
   - **访问令牌**：JWT 格式，包含用户身份和权限信息
   - **刷新令牌**：随机生成的 64 位字符串，用于获取新的访问令牌

3. **记录会话到数据库**：
   ```typescript
   await this.knex('directus_sessions').insert({
       token: refreshToken,
       user: user.id,
       expires: refreshTokenExpiration,
       ip: this.accountability?.ip,
       user_agent: this.accountability?.userAgent,
       origin: this.accountability?.origin,
   });
   ```
   *来源: `api/src/services/authentication.ts:281-288`*

4. **更新用户状态**：
   - 记录用户最后访问时间
   - 记录登录活动到审计日志

### 2.4 会话模式

Directus 支持三种会话模式，通过 `mode` 参数指定：

1. **JSON 模式** (`mode: 'json'`)：
   - 访问令牌和刷新令牌都在响应体中返回
   - 适用于 API 调用和移动应用

2. **Cookie 模式** (`mode: 'cookie'`)：
   - 刷新令牌存储在 HTTP-only cookie 中
   - 访问令牌在响应体中返回
   - 适用于 Web 应用，提高安全性

3. **会话模式** (`mode: 'session'`)：
   - 访问令牌存储在 cookie 中
   - 支持有状态会话管理
   - 适用于需要长时间保持登录状态的场景

## 3. 访问令牌的生成与刷新机制

### 3.1 令牌架构

Directus 使用双令牌机制：

- **访问令牌 (Access Token)**：JWT 格式，短期有效，用于 API 访问
- **刷新令牌 (Refresh Token)**：随机字符串，长期有效，用于获取新的访问令牌

### 3.2 访问令牌生成

**JWT 令牌结构**：

访问令牌是标准的 JWT，包含以下声明：

```typescript
interface DirectusTokenPayload {
    id: string;           // 用户 ID
    role: string;         // 角色 ID
    app_access: boolean;  // 是否有权限访问应用
    admin_access: boolean; // 是否有管理员权限
    session?: string;     // 会话模式下的刷新令牌引用
    share?: string;       // 共享链接 ID（如果适用）
    enforce_tfa?: boolean; // 是否强制启用双因素认证
}
```

**生成过程**：

1. **创建令牌 payload**：
   ```typescript
   const tokenPayload: DirectusTokenPayload = {
       id: user.id,
       role: user.role,
       app_access: globalAccess.app,
       admin_access: globalAccess.admin,
   };
   ```
   *来源: `api/src/services/authentication.ts:226-231`*

2. **添加自定义声明**：
   - 检查是否需要强制双因素认证
   - 触发 `auth.jwt` 过滤器钩子，允许扩展添加自定义声明

3. **签名生成令牌**：
   ```typescript
   const accessToken = jwt.sign(customClaims, getSecret(), {
       expiresIn: TTL,
       issuer: 'directus',
   });
   ```
   *来源: `api/src/services/authentication.ts:276-279`*

**令牌有效期**：
- 访问令牌：由 `ACCESS_TOKEN_TTL` 环境变量控制
- 刷新令牌：由 `REFRESH_TOKEN_TTL` 环境变量控制
- 会话模式：由 `SESSION_COOKIE_TTL` 环境变量控制

### 3.3 令牌刷新机制

Directus 实现了安全的令牌刷新机制，支持两种模式：

#### 3.3.1 无状态刷新模式

这是默认的刷新模式，流程如下：

1. **客户端请求**：通过 `/auth/refresh` 端点发送刷新令牌
2. **验证刷新令牌**：
   ```typescript
   const record = await this.knex
       .select(...)
       .from('directus_sessions AS s')
       .where('s.token', refreshToken)
       .andWhere('s.expires', '>=', new Date())
       .first();
   ```
   *来源: `api/src/services/authentication.ts:331-360`*

3. **生成新令牌对**：
   - 创建新的访问令牌
   - 创建新的刷新令牌
   - 更新数据库中的会话记录

4. **返回新令牌**：
   - 根据模式返回新的令牌对
   - 旧的刷新令牌立即失效

#### 3.3.2 有状态会话刷新模式

当使用 `session` 模式时，Directus 实现了更安全的刷新机制：

**核心特性**：
1. **令牌轮换**：每次刷新都生成新的令牌
2. **宽限期**：旧令牌在短时间内仍然有效，处理并发请求
3. **引用链**：通过 `next_token` 字段维护令牌链

**实现细节**：

```typescript
private async updateStatefulSession(
    sessionRecord: Record<string, any>,
    oldSessionToken: string,
    newSessionToken: string,
    sessionExpiration: Date,
): Promise<string> {
    if (sessionRecord['session_next_token']) {
        // 令牌已被刷新，返回下一个令牌
        await this.knex('directus_sessions')
            .update({ expires: sessionExpiration })
            .where({ token: newSessionToken });
        return newSessionToken;
    }

    // 设置宽限期
    const GRACE_PERIOD = getMilliseconds(env['SESSION_REFRESH_GRACE_PERIOD'], 10_000);

    // 更新旧令牌的过期时间并设置引用
    await this.knex('directus_sessions')
        .update(
            {
                next_token: newSessionToken,
                expires: new Date(Date.now() + GRACE_PERIOD),
            },
            ['next_token'],
        )
        .where({ token: oldSessionToken, next_token: null });

    // 创建新的会话记录
    await this.knex('directus_sessions').insert({
        token: newSessionToken,
        user: sessionRecord['user_id'],
        share: sessionRecord['share_id'],
        expires: sessionExpiration,
        ip: this.accountability?.ip,
        user_agent: this.accountability?.userAgent,
        origin: this.accountability?.origin,
    });

    return newSessionToken;
}
```
*来源: `api/src/services/authentication.ts:482-538`*

**安全优势**：
- 防止令牌复用攻击
- 检测令牌泄露（如果同一令牌被多次刷新）
- 保持用户体验（宽限期内并发请求不会失败）

### 3.4 令牌验证流程

当客户端携带访问令牌请求 API 时，系统会执行以下验证步骤：

1. **令牌提取**：从请求头、查询参数或 cookie 中提取令牌
2. **JWT 验证**：
   ```typescript
   export function verifyAccessJWT(token: string, secret: string) {
       const payload = verifyJWT(token, secret) as DirectusTokenPayload;
       
       if (payload.role === undefined || payload.app_access === undefined || payload.admin_access === undefined) {
           throw new InvalidTokenError();
       }
       
       return payload;
   }
   ```
   *来源: `api/src/utils/jwt.ts:25-33`*

3. **会话验证**（会话模式）：验证会话令牌是否有效
4. **权限解析**：从令牌 payload 中提取用户身份和权限信息

## 4. 操作者标识 (Accountability) 传递机制

### 4.1 什么是 Accountability

Accountability 是 Directus 中用于追踪操作执行者的核心对象，它包含了执行当前操作的用户的完整身份和上下文信息。

**Accountability 结构**：

```typescript
type Accountability = {
    role: string | null;       // 主角色 ID
    roles: string[];            // 所有角色 ID（包括继承的角色）
    user: string | null;        // 用户 ID
    admin: boolean;             // 是否有管理员权限
    app: boolean;               // 是否有权限访问应用
    share?: string;             // 共享链接 ID（如果适用）
    ip: string | null;          // 请求 IP 地址
    userAgent?: string;         // 用户代理字符串
    origin?: string;            // 请求来源
    session?: string;           // 会话令牌（会话模式）
};
```
*来源: `packages/types/src/accountability.ts:6-17`*

### 4.2 Accountability 的创建流程

Accountability 对象在请求处理的早期阶段创建，主要通过以下步骤：

1. **创建默认 Accountability**：
   ```typescript
   export function createDefaultAccountability(overrides?: Partial<Accountability>): Accountability {
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
   *来源: `api/src/permissions/utils/create-default-accountability.ts:3-13`*

2. **认证中间件处理**：
   ```typescript
   export const handler = async (req: Request, res: Response, next: NextFunction) => {
       // 创建默认 accountability，包含 IP 地址
       const defaultAccountability: Accountability = createDefaultAccountability({ 
           ip: getIPFromReq(req) 
       });

       // 添加用户代理和来源信息
       const userAgent = req.get('user-agent')?.substring(0, 1024);
       if (userAgent) defaultAccountability.userAgent = userAgent;

       const origin = req.get('origin');
       if (origin) defaultAccountability.origin = origin;

       // 允许扩展通过钩子自定义 accountability
       const customAccountability = await emitter.emitFilter(
           'authenticate',
           defaultAccountability,
           { req },
           { database, schema: null, accountability: null },
       );

       // 如果有自定义 accountability，使用自定义的
       if (customAccountability && isEqual(customAccountability, defaultAccountability) === false) {
           req.accountability = customAccountability;
           return next();
       }

       // 从令牌解析用户信息
       try {
           req.accountability = await getAccountabilityForToken(req.token, defaultAccountability);
       } catch (err) {
           // 处理令牌验证错误
           throw err;
       }

       return next();
   };
   ```
   *来源: `api/src/middleware/authenticate.ts:17-62`*

3. **从令牌解析用户信息**：
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
               // 验证 JWT 令牌
               const payload = verifyAccessJWT(token, getSecret());

               // 检查会话有效性（会话模式）
               if ('session' in payload) {
                   await verifySessionJWT(payload);
                   accountability.session = payload.session;
               }

               // 设置共享链接信息
               if (payload.share) accountability.share = payload.share;

               // 设置用户信息
               if (payload.id) accountability.user = payload.id;
               accountability.role = payload.role;

               // 获取角色树（包括继承的角色）
               accountability.roles = await fetchRolesTree(payload.role, { knex: database });

               // 获取全局权限
               const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });
               accountability.admin = admin;
               accountability.app = app;
           } else {
               // 处理静态令牌（用户 token 字段）
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

               // 设置用户信息
               accountability.user = user.id;
               accountability.role = user.role;
               accountability.roles = await fetchRolesTree(user.role, { knex: database });

               // 获取全局权限
               const { admin, app } = await fetchGlobalAccess(accountability, { knex: database });
               accountability.admin = admin;
               accountability.app = app;
           }
       }

       return accountability;
   }
   ```
   *来源: `api/src/utils/get-accountability-for-token.ts:12-69`*

### 4.3 Accountability 在服务层的传递

Accountability 对象通过服务构造函数传递到各个服务层，并在整个请求生命周期中保持一致。

**服务基类设计**：

所有 Directus 服务都遵循相同的模式，在构造函数中接收 `accountability` 参数：

```typescript
export class ItemsService<Item extends AnyItem = AnyItem, Collection extends string = string>
    implements AbstractService<Item>
{
    collection: Collection;
    knex: Knex;
    accountability: Accountability | null;  // Accountability 对象
    eventScope: string;
    schema: SchemaOverview;
    cache: Keyv<any> | null;
    nested: string[];

    constructor(collection: Collection, options: AbstractServiceOptions) {
        this.collection = collection;
        this.knex = options.knex || getDatabase();
        this.accountability = options.accountability || null;  // 注入 accountability
        this.eventScope = isSystemCollection(this.collection) ? this.collection.substring(9) : 'items';
        this.schema = options.schema;
        this.cache = getCache().cache;
        this.nested = options.nested ?? [];

        return this;
    }
    
    // ... 其他方法
}
```
*来源: `api/src/services/items.ts:42-63`*

**服务分支 (Forking)**：

Directus 服务支持创建分支，保持 accountability 的一致性：

```typescript
private fork(options?: Partial<AbstractServiceOptions>): ItemsService<AnyItem> {
    const Service = this.constructor;

    // ItemsService 期望 `collection` 和 `options` 作为参数，
    // 而其他服务只期望 `options`
    const isItemsService = Service.length === 2;

    const newOptions = {
        knex: this.knex,
        accountability: this.accountability,  // 保持 accountability 不变
        schema: this.schema,
        nested: this.nested,
        ...options,  // 允许覆盖特定选项
    };

    if (isItemsService) {
        return new ItemsService(this.collection, newOptions);
    }

    return new (Service as new (options: AbstractServiceOptions) => this)(newOptions);
}
```
*来源: `api/src/services/items.ts:68-88`*

### 4.4 Accountability 的使用场景

Accountability 对象在 Directus 中有多种重要用途：

#### 4.4.1 权限检查

服务层使用 accountability 中的用户和角色信息进行权限验证：

- **数据访问控制**：根据用户角色限制可访问的数据
- **字段级权限**：控制用户可以查看和修改哪些字段
- **操作权限**：控制用户可以执行的操作（创建、读取、更新、删除）

#### 4.4.2 审计日志

在认证服务中，accountability 用于记录用户活动：

```typescript
// 记录登录活动
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
*来源: `api/src/services/authentication.ts:293-301`*

#### 4.4.3 数据过滤

在处理数据库查询时，accountability 用于应用数据级别的权限过滤：

- 自动添加 `WHERE` 子句限制用户只能访问自己的数据
- 根据角色权限过滤可访问的记录

#### 4.4.4 事件钩子

当触发系统事件时，accountability 作为上下文传递给事件钩子：

```typescript
// 登录事件触发
emitter.emitAction(
    'auth.login',
    {
        payload: loginPayload,
        status,
        user: loginUser?.id,
        provider: providerName,
        error,
    },
    {
        database: this.knex,
        schema: this.schema,
        accountability: this.accountability,  // 传递 accountability
    },
);
```
*来源: `api/src/services/authentication.ts:76-91`*

### 4.5 Accountability 传递流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        HTTP 请求                                  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│              认证中间件 (authenticate.ts)                         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 1. 从请求中提取令牌                                          │  │
│  │ 2. 创建默认 accountability (包含 IP、User-Agent、Origin)    │  │
│  │ 3. 触发 'authenticate' 过滤器钩子 (允许自定义)               │  │
│  │ 4. 验证令牌并解析用户信息                                     │  │
│  │ 5. 填充 accountability 的用户、角色、权限信息                  │  │
│  │ 6. 将 accountability 附加到 req 对象                         │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                      控制器层 (Controllers)                        │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 从 req.accountability 获取 accountability 对象              │  │
│  │ 创建服务实例时传递 accountability                             │  │
│  │                                                              │  │
│  │ 示例:                                                        │  │
│  │ const service = new ItemsService(collection, {             │  │
│  │     accountability: req.accountability,                     │  │
│  │     schema: req.schema,                                      │  │
│  │ });                                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                       服务层 (Services)                            │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 构造函数接收 accountability 参数                              │  │
│  │ 保存为实例属性供所有方法使用                                   │  │
│  │                                                              │  │
│  │ 用途:                                                        │  │
│  │ • 权限检查 (validateAccess)                                  │  │
│  │ • 数据过滤 (processAst, processPayload)                     │  │
│  │ • 审计日志 (ActivityService)                                 │  │
│  │ • 事件钩子上下文                                              │  │
│  │ • 创建子服务时保持 accountability 一致性                      │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                    数据库层 (Database Operations)                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 通过服务层传递的 accountability 进行:                        │  │
│  │ • 应用权限过滤条件                                            │  │
│  │ • 记录操作执行者信息                                          │  │
│  │ • 检查数据访问权限                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## 5. 安全特性与最佳实践

### 5.1 安全特性

Directus 认证系统包含多项安全特性：

1. **密码安全**：
   - 使用 Argon2 算法进行密码哈希
   - 登录尝试速率限制
   - 登录失败后自动暂停用户

2. **令牌安全**：
   - JWT 使用 HS256 签名算法
   - 刷新令牌随机生成 (64 位)
   - 有状态会话模式的令牌轮换机制
   - 宽限期内的并发请求处理

3. **传输安全**：
   - 支持通过 HTTPS 传输
   - Cookie 模式使用 HTTP-only 和 Secure 标志
   - 防止 XSS 和 CSRF 攻击

4. **审计追踪**：
   - 记录登录、登出活动
   - 记录 IP 地址和用户代理
   - 追踪操作执行者

### 5.2 配置建议

1. **令牌有效期**：
   - 生产环境建议设置较短的访问令牌有效期
   - 刷新令牌有效期根据业务需求调整

2. **会话模式**：
   - 对于 Web 应用，建议使用 `cookie` 或 `session` 模式
   - 对于 API 访问，使用 `json` 模式

3. **安全配置**：
   - 确保 `SECRET` 环境变量设置为强随机值
   - 生产环境启用 HTTPS
   - 配置适当的 CORS 策略

## 6. 总结

Directus 的认证体系设计全面且灵活，通过以下核心机制确保安全可靠的用户认证和授权：

1. **多层认证架构**：
   - 抽象认证驱动支持多种认证方式
   - 本地认证与外部认证 (OAuth2、LDAP 等) 无缝集成
   - 双因素认证增强安全性

2. **双令牌机制**：
   - 短期访问令牌用于 API 访问
   - 长期刷新令牌用于获取新令牌
   - 有状态会话模式提供额外的安全保障

3. **完整的操作者追踪**：
   - Accountability 对象贯穿整个请求生命周期
   - 从中间件创建，通过控制器传递到服务层
   - 用于权限检查、审计日志和事件上下文

4. **安全特性**：
   - 密码哈希、速率限制、自动暂停
   - 令牌轮换、宽限期处理
   - 完整的审计追踪

这种设计既保证了系统的安全性，又提供了足够的灵活性来适应不同的应用场景和集成需求。

## 7. 参考代码位置

| 功能 | 文件路径 | 行号 |
|------|---------|------|
| 认证驱动抽象基类 | `api/src/auth/auth.ts` | 1-68 |
| 本地认证驱动 | `api/src/auth/drivers/local.ts` | 1-119 |
| 认证服务 | `api/src/services/authentication.ts` | 1-597 |
| 认证控制器 | `api/src/controllers/auth.ts` | 1-268 |
| 认证中间件 | `api/src/middleware/authenticate.ts` | 1-63 |
| JWT 工具 | `api/src/utils/jwt.ts` | 1-33 |
| Accountability 解析 | `api/src/utils/get-accountability-for-token.ts` | 1-69 |
| Accountability 类型 | `packages/types/src/accountability.ts` | 1-17 |
| 服务基类模式 | `api/src/services/items.ts` | 42-88 |
