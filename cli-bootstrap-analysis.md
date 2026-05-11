# Directus CLI 启动与初始化流程分析报告

## 1. 概述

本报告分析 Directus 命令行工具（CLI）的环境配置加载机制，以及项目初始化流程各步骤的衔接关系。

---

## 2. CLI 入口结构

### 2.1 入口文件链

Directus CLI 的启动流程涉及以下入口文件：

| 层级 | 文件路径 | 职责 |
|------|----------|------|
| 顶层入口 | `directus/cli.js` | 检查更新，转发到 API 包的 CLI 模块 |
| CLI 入口 | `api/src/cli/run.ts` | 调用 `createCli()` 并解析命令行参数 |
| CLI 定义 | `api/src/cli/index.ts` | 定义所有可用命令及其处理器 |

### 2.2 启动流程

```
directus/cli.js
  ├── 版本检查 (updateCheck)
  └── 动态导入 @directus/api/cli/run.js
       └── createCli()
            ├── 加载扩展 (loadExtensions)
            ├── 事件钩子 (cli.before)
            ├── 注册命令 (start, init, bootstrap 等)
            ├── 事件钩子 (cli.after)
            └── 解析命令行参数 (program.parseAsync)
```

关键代码位置：
- `directus/cli.js:1-9`
- `api/src/cli/run.ts:1-9`
- `api/src/cli/index.ts:20-124`

---

## 3. 环境配置加载机制

Directus 的环境配置加载由 `@directus/env` 包统一管理，采用懒加载单例模式。

### 3.1 核心接口

**`useEnv()`** 函数 (`packages/env/src/lib/use-env.ts:8-16`)：
- 首次调用时通过 `createEnv()` 创建配置对象
- 后续调用直接返回缓存的 `_cache.env`
- 全局单例，整个应用共享同一配置实例

### 3.2 配置加载优先级

配置加载顺序（后面覆盖前面）：

```
1. 默认值 (DEFAULTS)
   ↓
2. 进程环境变量 (process.env)
   ↓
3. 配置文件 (.env, .config.js, .config.json, .config.yml 等)
```

关键代码：`packages/env/src/lib/create-env.ts:14-49`

```typescript
const baseConfiguration = readConfigurationFromProcess();
const fileConfiguration = readConfigurationFromFile(getConfigPath());
const rawConfiguration = { ...baseConfiguration, ...fileConfiguration };
```

### 3.3 配置文件支持格式

`readConfigurationFromFile()` 支持以下格式（`packages/env/src/lib/read-configuration-from-file.ts:13-33`）：

| 扩展名 | 处理方式 | 读取函数 |
|--------|----------|----------|
| `.js`, `.mjs`, `.cjs` | JavaScript 模块 | `readConfigurationFromJavaScript` |
| `.json` | JSON 文件 | `readConfigurationFromJson` |
| `.yml`, `.yaml` | YAML 文件 | `readConfigurationFromYaml` |
| 其他（如 `.env`） | 点环境文件 | `readConfigurationFromDotEnv` |

### 3.4 文件型配置变量

支持通过 `_FILE` 后缀从文件读取敏感值（`packages/env/src/lib/create-env.ts:27-43`）：

```bash
# 示例：从文件读取数据库密码
DB_PASSWORD_FILE=/path/to/secret.txt
```

处理流程：
1. 检测到 `_FILE` 后缀且是 Directus 变量
2. 读取指定文件内容
3. 移除 `_FILE` 后缀，将文件内容作为变量值
4. 支持类型转换前缀（如 `json:`）

### 3.5 类型转换系统

所有配置值通过 `cast()` 函数进行类型转换，支持：
- 基本类型：string, number, boolean, json, array
- 自动类型推断
- 强制类型前缀（如 `json:`、`array:`）

---

## 4. 服务启动流程 (`directus start`)

### 4.1 整体流程

```
startServer() [api/src/server.ts:166-212]
  ├── createServer() [api/src/server.ts:37-164]
  │    ├── createApp() [api/src/app.ts:88-396]
  │    │    ├── 环境配置验证
  │    │    ├── 数据库连接验证
  │    │    ├── Directus 安装状态检查
  │    │    ├── 迁移状态检查
  │    │    ├── SECRET/PUBLIC_URL 警告
  │    │    ├── 数据库扩展验证
  │    │    ├── 存储配置验证
  │    │    ├── 认证提供者注册
  │    │    ├── 部署驱动注册
  │    │    ├── 扩展管理器初始化
  │    │    ├── 流程管理器初始化
  │    │    ├── Express 应用配置
  │    │    ├── 中间件注册
  │    │    ├── 路由注册
  │    │    ├── 定时任务初始化
  │    │    └── 事件钩子
  │    ├── HTTP 服务器创建
  │    ├── WebSocket 控制器（可选）
  │    └── Terminus 健康检查配置
  └── 监听端口 / Unix Socket
```

### 4.2 应用初始化 (`createApp`) 详解

`api/src/app.ts:88-396` 中的 `createApp()` 是核心初始化函数，执行以下验证和配置：

#### 4.2.1 前置验证

| 验证项 | 代码位置 | 失败处理 |
|--------|----------|----------|
| 数据库连接 | `api/src/app.ts:93` | 抛出错误，退出进程 |
| Directus 是否已安装 | `api/src/app.ts:95-98` | 未安装则退出 |
| 迁移是否完整 | `api/src/app.ts:100-102` | 不完整则警告 |
| SECRET 配置 | `api/src/app.ts:104-114` | 缺失或过短则警告 |
| PUBLIC_URL 格式 | `api/src/app.ts:116-118` | 非绝对 URL 则警告 |
| 数据库扩展 | `api/src/app.ts:120` | 警告不阻止启动 |
| 存储配置 | `api/src/app.ts:121` | - |

#### 4.2.2 管理器初始化

```typescript
const extensionManager = getExtensionManager();
const flowManager = getFlowManager();
await extensionManager.initialize();
await flowManager.initialize();
```

#### 4.2.3 中间件注册顺序

1. 压力限制器 (`PRESSURE_LIMITER_ENABLED`)
2. Helmet CSP 安全头
3. Cross-Origin-Opener-Policy
4. HSTS
5. 请求日志
6. CORS (`CORS_ENABLED`)
7. JSON 解析 + Raw Body 捕获
8. Cookie 解析
9. Token 提取
10. 全局速率限制 (`RATE_LIMITER_GLOBAL_ENABLED`)
11. IP 速率限制 (`RATE_LIMITER_ENABLED`)
12. 认证中间件
13. Schema 中间件
14. 查询参数清理
15. 请求计数器
16. 缓存中间件

#### 4.2.4 路由注册

核心路由（`api/src/app.ts:321-380`）：
- `/auth` - 认证
- `/graphql` - GraphQL API
- `/activity`, `/access`, `/assets`, `/collections`, `/comments`, `/dashboards`
- `/deployments`, `/extensions`, `/fields`, `/files`, `/flows`, `/folders`
- `/items`, `/mcp` (可选), `/ai` (可选), `/metrics` (可选)
- `/notifications`, `/operations`, `/panels`, `/permissions`, `/policies`
- `/presets`, `/translations`, `/relations`, `/revisions`, `/roles`
- `/schema`, `/server`, `/settings`, `/shares`, `/users`, `/utils`, `/versions`
- 扩展端点路由

#### 4.2.5 定时任务

在应用初始化最后启动以下定时任务：
- 保留策略任务 (`retentionSchedule`)
- 遥测任务 (`telemetrySchedule`)
- TUS 清理任务 (`tusSchedule`)
- 指标任务 (`metricsSchedule`)
- 项目任务 (`projectSchedule`)

### 4.3 事件钩子

`createApp()` 中触发的事件钩子（通过 `emitter.emitInit`）：

| 钩子名称 | 触发时机 |
|----------|----------|
| `app.before` | 创建 Express app 之前 |
| `middlewares.before` | 注册中间件之前 |
| `middlewares.after` | 注册中间件之后 |
| `routes.before` | 注册路由之前 |
| `routes.custom.before` | 注册扩展端点之前 |
| `routes.custom.after` | 注册扩展端点之后 |
| `routes.after` | 注册路由之后 |
| `app.after` | 应用完全初始化之后 |

---

## 5. 项目初始化流程 (`directus init`)

### 5.1 整体流程

```
init() [api/src/cli/commands/init/index.ts:18-125]
  ├── 1. 交互式选择数据库驱动
  ├── 2. 安装数据库驱动包 (npm install)
  ├── 3. 交互式输入数据库连接信息
  ├── 4. 验证连接并初始化数据库
  │    ├── runSeed() - 创建系统表
  │    └── runMigrations() - 执行迁移
  ├── 5. 生成 .env 配置文件
  ├── 6. 创建首个管理员用户
  └── 7. 输出使用说明
```

### 5.2 步骤详解

#### 步骤 1-2：数据库驱动选择与安装

```typescript
// 交互式选择数据库类型
const { client } = await inquirer.prompt([...]);
const dbClient = getDriverForClient(client);

// 使用 npm 安装驱动
await execa('npm', ['install', dbClient, '--production']);
```

代码位置：`api/src/cli/commands/init/index.ts:21-34`

#### 步骤 3：数据库连接配置

根据数据库类型，通过 `inquirer` 交互式提问收集连接信息：

| 数据库类型 | 问题配置 |
|------------|----------|
| SQLite | 文件名 |
| MySQL | host, port, database, user, password |
| PostgreSQL | host, port, database, user, password, ssl |
| CockroachDB | 同 PostgreSQL |
| OracleDB | host, port, database, user, password |
| MSSQL | host, port, database, user, password, encrypt |

问题定义：`api/src/cli/commands/init/questions.ts:69-76`

#### 步骤 4：数据库初始化

```typescript
const db = createDBConnection(dbClient, credentials);
await runSeed(db);           // 创建 Directus 系统表
await runMigrations(db, 'latest', false);  // 执行所有迁移
```

**`runSeed()`** (`api/src/database/seeds/run.ts:33-115`)：
- 检查是否已安装（`directus_collections` 表是否存在）
- 读取 `database/seeds/` 目录下的 YAML 种子文件
- 按定义创建所有系统表

**`runMigrations()`** (`api/src/database/migrations/run.ts:16-148`)：
- 读取内置迁移 + 自定义迁移（`MIGRATIONS_PATH`）
- 按版本号排序
- 执行所有未完成的 `up()` 迁移
- 记录到 `directus_migrations` 表

#### 步骤 5：生成环境配置文件

```typescript
await createEnv(dbClient, credentials, rootPath);
```

**`createEnv()`** (`api/src/cli/utils/create-env/index.ts:21-54`) 执行：
1. 生成 32 字节随机 `SECRET` (nanoid)
2. 将数据库凭据转换为 `DB_*` 环境变量格式
3. 使用 Liquid 模板引擎渲染 `env-stub.liquid` 模板
4. 写入 `.env` 文件并设置权限为 `0o640`

模板变量包含：
- `security` 部分：`SECRET`
- `database` 部分：`DB_CLIENT`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USER`, `DB_PASSWORD` 等

#### 步骤 6：创建管理员账户

```typescript
// 交互式获取邮箱和密码
const firstUser = await inquirer.prompt([...]);
firstUser.password = await generateHash(firstUser.password);

// 创建角色、策略、访问权限、用户
const role = randomUUID();
const policy = randomUUID();

await db('directus_roles').insert({ ...defaultAdminRole, id: role });
await db('directus_policies').insert({ ...defaultAdminPolicy, id: policy });
await db('directus_access').insert({ id: randomUUID(), role, policy });
await db('directus_users').insert({ ...defaultAdminUser, id: randomUUID(), ... });
```

代码位置：`api/src/cli/commands/init/index.ts:74-114`

---

## 6. 数据库引导流程 (`directus bootstrap`)

### 6.1 与 `init` 的区别

| 特性 | `directus init` | `directus bootstrap` |
|------|-----------------|----------------------|
| 适用场景 | 全新项目首次初始化 | 容器化/自动化部署 |
| 交互性 | 交互式 | 非交互式 |
| 依赖 .env | 生成新的 | 需要预先配置好 |
| 数据库驱动 | 自动安装 | 需预先安装 |
| 管理员创建 | 交互式创建 | 可选跳过 |

### 6.2 流程

```
bootstrap() [api/src/cli/commands/bootstrap/index.ts:16-66]
  ├── 1. 等待数据库可用 (最多 5 次重试，间隔 5 秒)
  ├── 2. 检查是否已安装
  │    ├── 未安装：完整安装流程
  │    │    ├── installDatabase() - 创建系统表
  │    │    ├── runMigrations() - 执行迁移
  │    │    ├── getSchema() - 加载 Schema
  │    │    ├── createAdmin() - 创建管理员 (可跳过)
  │    │    └── 设置 PROJECT_NAME / PROJECT_OWNER
  │    └── 已安装：仅执行迁移
  └── 3. 销毁数据库连接，退出
```

### 6.3 数据库等待机制

`waitForDatabase()` (`api/src/cli/commands/bootstrap/index.ts:68-84`)：
- 最多尝试 5 次
- 每次间隔 5 秒
- 通过执行 `SELECT 1` (或 Oracle 的 `select 1 from DUAL`) 验证连接
- 失败则调用 `validateDatabaseConnection()` 抛出明确错误

### 6.4 环境变量驱动的配置

`bootstrap` 支持通过环境变量自动化配置：

| 环境变量 | 作用 |
|----------|------|
| `PROJECT_NAME` | 设置项目名称（写入 `directus_settings`） |
| `PROJECT_OWNER` | 设置项目所有者邮箱 |
| `ADMIN_EMAIL`, `ADMIN_PASSWORD` | 自动创建管理员账户（在 `createAdmin()` 中使用） |

---

## 7. 数据库连接与验证机制

### 7.1 数据库连接创建

`getDatabase()` (`api/src/database/index.ts:35-206`) 执行：

1. **环境变量验证**（`validateEnv`）：
   - 必需：`DB_CLIENT`
   - 根据数据库类型追加必需变量（如 SQLite 需要 `DB_FILENAME`）

2. **Knex 配置构建**：
   - 从 `DB_*` 前缀环境变量构建配置
   - 各数据库特定优化：
     - SQLite：启用外键支持 (`PRAGMA foreign_keys = ON`)
     - CockroachDB：设置 `serial_normalization` 和 `default_int_size`
     - OracleDB：设置日期时间格式
     - MySQL：自动转换为 `mysql2` 客户端
     - MSSQL：禁用 UTC 自动转换

3. **连接池与查询监控**：
   - 配置查询事件监听（`query`, `query-response`, `query-error`）
   - 记录查询执行时间到指标系统
   - 输出 Trace 级别日志

### 7.2 关键验证函数

| 函数 | 用途 | 代码位置 |
|------|------|----------|
| `hasDatabaseConnection()` | 轻量级连接检查 | `api/src/database/index.ts:220-234` |
| `validateDatabaseConnection()` | 严格连接验证，失败则退出 | `api/src/database/index.ts:236-251` |
| `isInstalled()` | 检查 `directus_collections` 表是否存在 | `api/src/database/index.ts:277-284` |
| `validateMigrations()` | 检查所有迁移是否已执行 | `api/src/database/index.ts:286-318` |
| `validateDatabaseExtensions()` | 检查 PostGIS/Spatialite 等空间扩展 | `api/src/database/index.ts:323-343` |

---

## 8. 流程衔接关系图

### 8.1 首次完整部署流程

```
用户执行: directus init
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 数据库驱动安装 (npm install)                              │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 数据库连接配置 (交互式)                                    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 数据库初始化                                              │
│    - runSeed() → 创建 directus_* 系统表                      │
│    - runMigrations() → 执行所有数据迁移                      │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 生成 .env 配置文件                                        │
│    - SECRET (随机生成)                                       │
│    - DB_* (数据库连接信息)                                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. 创建管理员账户                                            │
│    - directus_roles (默认管理员角色)                         │
│    - directus_policies (完全访问策略)                        │
│    - directus_access (角色-策略关联)                         │
│    - directus_users (管理员用户)                             │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
完成，可以运行: directus start
```

### 8.2 容器化部署流程

```
Kubernetes/Docker 启动
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 预先注入环境变量                                           │
│    - DB_CLIENT, DB_HOST, DB_PORT, DB_DATABASE...            │
│    - SECRET, ADMIN_EMAIL, ADMIN_PASSWORD                    │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 执行: directus bootstrap                                  │
│    - waitForDatabase() → 等待数据库就绪                      │
│    - isInstalled() → 检查是否首次启动                        │
│    ├─ 首次: installDatabase + migrations + createAdmin       │
│    └─ 非首次: 仅 migrations                                  │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 执行: directus start                                      │
│    - createApp() → 完整应用初始化                            │
│    - createServer() → HTTP/WebSocket 服务器                  │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
服务就绪
```

### 8.3 日常启动流程

```
用户执行: directus start
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 环境加载                                                  │
│    - useEnv() → process.env + .env 文件 → 类型转换           │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 数据库层初始化                                            │
│    - getDatabase() → Knex 连接池 + 验证必需变量              │
│    - validateDatabaseConnection() → 连接测试                 │
│    - isInstalled() → 必须已安装                              │
│    - validateMigrations() → 警告未完成迁移                   │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 应用层初始化                                              │
│    - registerAuthProviders()                                │
│    - extensionManager.initialize()                          │
│    - flowManager.initialize()                               │
│    - Express 中间件 + 路由注册                               │
│    - 定时任务启动                                            │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 服务器启动                                                │
│    - HTTP 服务器监听 PORT/HOST 或 UNIX_SOCKET_PATH           │
│    - WebSocket 控制器 (可选)                                 │
│    - Terminus 健康检查 + 优雅关闭                            │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. 就绪信号                                                  │
│    - logger.info("Server started at ...")                   │
│    - process.send('ready') (PM2/Cluster 模式)                │
│    - emitter.emitAction('server.start')                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 9. 关键文件索引

| 功能模块 | 主文件路径 |
|----------|------------|
| CLI 顶层入口 | `directus/cli.js` |
| CLI 命令定义 | `api/src/cli/index.ts` |
| 项目初始化命令 | `api/src/cli/commands/init/index.ts` |
| 数据库引导命令 | `api/src/cli/commands/bootstrap/index.ts` |
| 初始化问题配置 | `api/src/cli/commands/init/questions.ts` |
| .env 文件生成 | `api/src/cli/utils/create-env/index.ts` |
| 环境配置加载 | `packages/env/src/lib/create-env.ts` |
| 环境单例访问 | `packages/env/src/lib/use-env.ts` |
| 应用初始化 | `api/src/app.ts` |
| 服务器启动 | `api/src/server.ts` |
| 数据库连接管理 | `api/src/database/index.ts` |
| 数据库表种子 | `api/src/database/seeds/run.ts` |
| 数据库迁移 | `api/src/database/migrations/run.ts` |
| 环境变量验证 | `api/src/utils/validate-env.ts` |

---

## 10. 设计要点总结

1. **懒加载单例模式**：`useEnv()` 确保环境配置只加载一次，全局共享

2. **配置优先级**：进程环境变量 > 配置文件 > 默认值，支持 `_FILE` 后缀从文件读取

3. **多格式配置支持**：`.env`, `.js`, `.json`, `.yml` 均可作为配置文件

4. **分离的初始化路径**：
   - `init`：交互式，面向开发者
   - `bootstrap`：非交互式，面向自动化部署

5. **数据库状态机**：通过 `directus_collections` 表存在性判断安装状态，通过 `directus_migrations` 表追踪迁移进度

6. **事件驱动架构**：`emitter.emitInit` 提供多个扩展点，允许插件介入初始化流程

7. **优雅关闭**：Terminus 集成确保在 `SIGINT/SIGTERM` 时正确关闭数据库连接、WebSocket 和定时任务
