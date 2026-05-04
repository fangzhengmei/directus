# Directus ItemsService 与数据库适配架构分析

## 一、整体架构概览

Directus 通过精心设计的多层架构实现了 REST 和 GraphQL 请求的统一处理，以及多数据库的无缝适配。核心架构如下：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API Entry Layer                                   │
│  ┌─────────────────────┐          ┌─────────────────────┐                   │
│  │   REST Controller   │          │   GraphQL Service   │                   │
│  │  (controllers/*)    │          │  (services/graphql) │                   │
│  └──────────┬──────────┘          └──────────┬──────────┘                   │
└─────────────┼────────────────────────────────┼────────────────────────────────┘
              │                                │
              │  ┌───────────────────────────┐ │
              │  │    getService() 工具      │ │
              │  │  (utils/get-service.ts)   │ │
              │  └─────────────┬─────────────┘ │
              │                │                 │
              ▼                ▼                 │
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Unified Service Layer                                 │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                        ItemsService (核心)                              │   │
│  │  - createOne/createMany    │  - readByQuery/readOne/readMany         │   │
│  │  - updateOne/updateMany    │  - deleteOne/deleteMany                 │   │
│  │  - upsertOne/upsertMany    │  - readSingleton/upsertSingleton        │   │
│  └────────────────────────────┬──────────────────────────────────────────┘   │
│                               │                                                │
│              ┌────────────────┼────────────────┐                             │
│              ▼                ▼                ▼                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │  PayloadService │  │   MetaService   │  │  ActivityService │             │
│  │  (数据处理/转换) │  │  (元数据计算)   │  │  (操作审计)     │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
└───────────────────────┬────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Query Execution Layer                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                    AST 转换与执行管道                                   │   │
│  │                                                                         │   │
│  │  Query Object ──► getAstFromQuery() ──► AST ──► runAst()            │   │
│  │                         │                              │               │   │
│  │                         ▼                              ▼               │   │
│  │                  ┌─────────────┐              ┌─────────────┐       │   │
│  │                  │ parseFields │              │  getDBQuery │       │   │
│  │                  │ (字段解析)   │              │ (构建Knex)  │       │   │
│  │                  └─────────────┘              └─────────────┘       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
└───────────────────────┬────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Database Abstraction Layer                               │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                           Knex.js 核心                                  │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐  │   │
│  │  │ getDatabase │  │getDatabase- │  │      getHelpers()           │  │   │
│  │  │  (连接管理)  │  │ Client      │  │  (方言特定的辅助函数)        │  │   │
│  │  │             │  │ (类型检测)   │  │                             │  │   │
│  │  └─────────────┘  └─────────────┘  │  ┌───────────────────────┐  │  │   │
│  │                                      │  │ date: 日期函数        │  │  │   │
│  │                                      │  │ st: 空间几何函数      │  │  │   │
│  │                                      │  │ schema: 模式操作      │  │  │   │
│  │                                      │  │ sequence: 序列管理    │  │  │   │
│  │                                      │  │ number: 数值处理      │  │  │   │
│  │                                      │  └───────────────────────┘  │  │   │
│  │                                      └─────────────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │ Postgres │ │  MySQL   │ │  SQLite  │ │  MSSQL   │ │  Oracle  │       │
│  │CockroachDB│ │ MariaDB  │ │          │ │          │ │          │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 二、REST 与 GraphQL 统一到 ItemsService

### 2.1 统一入口设计

Directus 的核心设计理念是**将所有数据操作统一收敛到 ItemsService**，无论请求来自 REST 还是 GraphQL。

#### REST 控制器调用方式 (`api/src/controllers/items.ts`)

REST 控制器直接实例化 ItemsService：

```typescript
// 创建操作
router.post('/:collection', collectionExists, asyncHandler(async (req, res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    if (Array.isArray(req.body)) {
        const keys = await service.createMany(req.body);
        // ...
    } else {
        const key = await service.createOne(req.body);
        // ...
    }
}));

// 查询操作
const readHandler = asyncHandler(async (req, res, next) => {
    const service = new ItemsService(req.collection, {
        accountability: req.accountability,
        schema: req.schema,
    });

    if (req.singleton) {
        result = await service.readSingleton(req.sanitizedQuery);
    } else if (req.body.keys) {
        result = await service.readMany(req.body.keys, req.sanitizedQuery);
    } else {
        result = await service.readByQuery(req.sanitizedQuery);
    }
});
```

**特点**：
- 每个 REST 路由处理器都创建新的 ItemsService 实例
- 直接调用 `createOne/createMany/readByQuery/readOne` 等方法
- 通过 `req.sanitizedQuery` 传递查询参数（已通过中间件处理）

#### GraphQL 解析器调用方式 (`api/src/services/graphql/resolvers/`)

GraphQL 使用 `getService()` 工具函数获取服务实例：

**查询解析器** (`resolvers/query.ts`):
```typescript
export async function resolveQuery(gql: GraphQLService, info: GraphQLResolveInfo) {
    // 1. 解析 GraphQL 字段名和参数
    let collection = info.fieldName;
    if (gql.scope === 'system') collection = `directus_${collection}`;
    
    // 2. 解析参数为 Directus Query 对象
    const args = parseArgs(info.fieldNodes[0]!.arguments, info.variableValues);
    const query = await getQuery(args, gql.schema, selections, ...);
    
    // 3. 通过 GraphQLService.read() 间接调用 ItemsService
    const result = await gql.read(collection, query, args['id']);
    return result;
}
```

**变更解析器** (`resolvers/mutation.ts`):
```typescript
export async function resolveMutation(gql: GraphQLService, args, info) {
    // 解析动作和集合
    const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
    let collection = info.fieldName.substring(action.length + 1);
    
    // 获取 Query 对象
    const query = await getQuery(args, gql.schema, selections, ...);
    
    // 获取服务实例
    const service = getService(collection, {
        knex: gql.knex,
        accountability: gql.accountability,
        schema: gql.schema,
    });
    
    // 根据动作调用不同方法
    if (single) {
        if (action === 'create') {
            const key = await service.createOne(args['data']);
            return hasQuery ? await service.readOne(key, query) : true;
        }
        if (action === 'update') {
            const key = await service.updateOne(args['id'], args['data']);
            return hasQuery ? await service.readOne(key, query) : true;
        }
        if (action === 'delete') {
            await service.deleteOne(args['id']);
            return { id: args['id'] };
        }
    }
}
```

**GraphQLService 中的 read 方法** (`services/graphql/index.ts`):
```typescript
async read(collection: string, query: Query, id?: PrimaryKey) {
    const service = getService(collection, {
        knex: this.knex,
        accountability: this.accountability,
        schema: this.schema,
    });

    if (this.schema.collections[collection]!.singleton)
        return await service.readSingleton(query, { stripNonRequested: false });

    if (id) return await service.readOne(id, query, { stripNonRequested: false });

    return await service.readByQuery(query, { stripNonRequested: false });
}
```

### 2.2 getService() 服务工厂

`utils/get-service.ts` 是一个智能服务工厂，根据集合名称返回对应的服务：

```typescript
export function getService(collection: string, opts: AbstractServiceOptions): ItemsService {
    switch (collection) {
        // 系统集合使用专门的 Service（继承自 ItemsService）
        case 'directus_users':
            return new UsersService(opts);
        case 'directus_files':
            return new FilesService(opts);
        case 'directus_activity':
            return new ActivityService(opts);
        // ... 更多系统集合
        
        // 自定义集合使用通用 ItemsService
        default:
            if (collection.startsWith('directus_')) throw new ForbiddenError();
            return new ItemsService(collection, opts);
    }
}
```

**设计优势**：
1. **统一接口**：所有 Service 都继承或兼容 ItemsService 接口
2. **系统集合增强**：系统集合（如 `directus_users`）有特殊业务逻辑
3. **权限控制**：通过 switch-case 精确控制哪些集合可访问

### 2.3 统一调用模式对比

| 维度 | REST API | GraphQL |
|------|---------|---------|
| **服务获取** | `new ItemsService(collection, opts)` | `getService(collection, opts)` |
| **参数来源** | Express Request (query params, body) | GraphQL AST + variables |
| **Query 对象** | 由 `sanitizeQuery` 中间件生成 | 由 `getQuery()` 解析器生成 |
| **方法调用** | 直接调用 service 方法 | 解析后调用对应方法 |
| **权限上下文** | `req.accountability` | `gql.accountability` |

---

## 二、补充：系统集合在 REST 与 GraphQL 中的入口差异

### 2.4 系统集合的特殊处理

Directus 中的系统集合（以 `directus_` 为前缀）在 REST 和 GraphQL 中有不同的入口设计，但最终都通过 `getService()` 工厂函数统一到相同的服务调用链。

#### REST 中的系统集合处理

**独立控制器设计**：

系统集合在 REST 层有独立的控制器文件，每个系统集合都有专门的路由定义：

```
api/src/controllers/
├── access.ts           # directus_access
├── activity.ts         # directus_activity
├── comments.ts         # directus_comments
├── dashboards.ts       # directus_dashboards
├── deployment.ts       # directus_deployments
├── deployment-webhooks.ts  # directus_deployment_webhooks
├── fields.ts           # directus_fields
├── files.ts            # directus_files
├── flows.ts            # directus_flows
├── folders.ts          # directus_folders
├── notifications.ts    # directus_notifications
├── operations.ts       # directus_operations
├── panels.ts           # directus_panels
├── permissions.ts      # directus_permissions
├── policies.ts         # directus_policies
├── presets.ts          # directus_presets
├── relations.ts        # directus_relations
├── revisions.ts        # directus_revisions
├── roles.ts            # directus_roles
├── settings.ts         # directus_settings
├── shares.ts           # directus_shares
├── translations.ts     # directus_translations
├── users.ts            # directus_users
└── versions.ts         # directus_versions
```

**Users 控制器示例** (`controllers/users.ts`):

```typescript
const router = express.Router();

// 使用 useCollection 中间件设置集合上下文
router.use(useCollection('directus_users'));

// 独立的路由定义
router.post('/', asyncHandler(async (req, res, next) => {
    // 直接实例化专门的 UsersService，而非通用 ItemsService
    const service = new UsersService({
        accountability: req.accountability,
        schema: req.schema,
    });

    if (Array.isArray(req.body)) {
        const keys = await service.createMany(req.body);
        savedKeys.push(...keys);
    } else {
        const key = await service.createOne(req.body);
        savedKeys.push(key);
    }
    // ...
}));

// 系统集合特有的额外端点
router.post('/invite', asyncHandler(async (req, _res, next) => {
    const service = new UsersService({
        accountability: req.accountability,
        schema: req.schema,
    });
    // 邀请用户 - UsersService 特有的方法
    await service.inviteUser(req.body.email, req.body.role, req.body.invite_url || null);
    return next();
}));

router.post('/me/tfa/generate/', asyncHandler(async (req, res, next) => {
    const service = new TFAService({
        accountability: req.accountability,
        schema: req.schema,
    });
    // 双因素认证相关操作
    const { url, secret } = await service.generateTFA(req.accountability.user, requiresPassword);
    res.locals['payload'] = { data: { secret, otpauth_url: url } };
}));
```

**Files 控制器示例** (`controllers/files.ts`):

```typescript
const router = express.Router();
router.use(useCollection('directus_files'));

// 文件上传特有端点 - multipart/form-data 处理
router.post('/', asyncHandler(multipartHandler), asyncHandler(async (req, res, next) => {
    const service = new FilesService({
        accountability: req.accountability,
        schema: req.schema,
    });

    let keys: PrimaryKey | PrimaryKey[] = [];

    if (req.is('multipart/form-data')) {
        keys = res.locals['savedFiles'];
    } else {
        keys = await service.createOne(req.body);
    }
    // ...
}));

// 文件导入特有端点
router.post('/import', asyncHandler(async (req, res, next) => {
    const service = new FilesService({
        accountability: req.accountability,
        schema: req.schema,
    });
    const primaryKey = await service.importOne(req.body.url, req.body.data, req.body.options);
    // ...
}));
```

**Activity 控制器示例** (`controllers/activity.ts`):

```typescript
const router = express.Router();
router.use(useCollection('directus_activity'));

// Activity 是只读集合，只有查询端点
const readHandler = asyncHandler(async (req, res, next) => {
    const service = new ActivityService({
        accountability: req.accountability,
        schema: req.schema,
    });
    // ...
});

router.search('/', validateBatch('read'), readHandler, respond);
router.get('/', readHandler, respond);
router.get('/:pk', asyncHandler(async (req, res, next) => {
    const service = new ActivityService({
        accountability: req.accountability,
        schema: req.schema,
    });
    const record = await service.readOne(req.params['pk']!, req.sanitizedQuery);
    res.locals['payload'] = { data: record || null };
    return next();
}), respond);

// 没有 POST/PATCH/DELETE 端点 - 只读
```

**REST 系统集合控制器特点**：

| 特点 | 说明 |
|------|------|
| **独立路由文件** | 每个系统集合有自己的 `controllers/*.ts` 文件 |
| **useCollection 中间件** | 设置 `req.collection` 上下文 |
| **直接实例化专门 Service** | `new UsersService()` 而非 `new ItemsService()` |
| **额外业务端点** | 如 `/users/invite`、`/users/me/tfa/generate` |
| **只读限制** | 部分集合（如 activity）只暴露 GET 端点 |

#### GraphQL 中的系统集合处理

**Scope 隔离设计**：

GraphQL 通过 `scope` 概念将系统集合与普通集合完全隔离，有独立的访问端点：

```typescript
// services/graphql/schema/index.ts

// 系统集合黑名单 - 这些集合完全不暴露在 GraphQL 中
export const SYSTEM_DENY_LIST = [
    'directus_collections',
    'directus_fields',
    'directus_relations',
    'directus_migrations',
    'directus_sessions',
    'directus_extensions',
];

// 只读系统集合
export const READ_ONLY = ['directus_activity', 'directus_revisions'];

// Scope 过滤器 - 根据 scope 决定暴露哪些集合
const scopeFilter = (collection: SchemaOverview['collections'][string]) => {
    // items scope: 只暴露非系统集合
    if (gql.scope === 'items' && isSystemCollection(collection.collection)) 
        return false;

    // system scope: 只暴露系统集合（排除黑名单）
    if (gql.scope === 'system') {
        if (isSystemCollection(collection.collection) === false) 
            return false;
        if (SYSTEM_DENY_LIST.includes(collection.collection)) 
            return false;
    }

    return true;
};
```

**集合命名转换**：

```typescript
// 生成 Schema 时的集合名处理
const readableCollections = Object.values(schema.read.collections)
    .filter((collection) => collection.collection in ReadCollectionTypes)
    .filter(scopeFilter);

if (readableCollections.length > 0) {
    schemaComposer.Query.addFields(
        readableCollections.reduce(
            (acc, collection) => {
                // 集合名转换：
                // - items scope: 直接使用原名 (如 'articles')
                // - system scope: 去掉 'directus_' 前缀 (如 'users' 而非 'directus_users')
                const collectionName = gql.scope === 'items' 
                    ? collection.collection 
                    : collection.collection.substring(9);
                
                acc[collectionName] = ReadCollectionTypes[collection.collection]!.getResolver(collection.collection);

                // 非单例集合添加额外查询
                if (gql.schema.collections[collection.collection]!.singleton === false) {
                    // 添加 by_id 查询
                    acc[`${collectionName}_by_id`] = ReadCollectionTypes[collection.collection]!.getResolver(
                        `${collection.collection}_by_id`,
                    );
                    // 添加 aggregated 聚合查询
                    acc[`${collectionName}_aggregated`] = ReadCollectionTypes[collection.collection]!.getResolver(
                        `${collection.collection}_aggregated`,
                    );
                }

                return acc;
            },
            {} as ObjectTypeComposerFieldConfigAsObjectDefinition<any, any>,
        ),
    );
}
```

**Mutation 端点的只读限制**：

```typescript
// 只有不在 READ_ONLY 列表中的集合才生成 Mutation 端点
if (Object.keys(schema.create.collections).length > 0) {
    schemaComposer.Mutation.addFields(
        Object.values(schema.create.collections)
            .filter((collection) => collection.collection in CreateCollectionTypes && collection.singleton === false)
            .filter(scopeFilter)
            // 关键：过滤掉只读集合
            .filter((collection) => READ_ONLY.includes(collection.collection) === false)
            .reduce(
                (acc, collection) => {
                    const collectionName = gql.scope === 'items' 
                        ? collection.collection 
                        : collection.collection.substring(9);

                    acc[`create_${collectionName}_items`] = CreateCollectionTypes[collection.collection]!.getResolver(
                        `create_${collection.collection}_items`,
                    );
                    acc[`create_${collectionName}_item`] = CreateCollectionTypes[collection.collection]!.getResolver(
                        `create_${collection.collection}_item`,
                    );

                    return acc;
                },
                {} as ObjectTypeComposerFieldConfigAsObjectDefinition<any, any>,
            ),
    );
}
```

**查询解析器中的集合名还原**：

```typescript
// services/graphql/resolvers/query.ts
export async function resolveQuery(gql: GraphQLService, info: GraphQLResolveInfo) {
    let collection = info.fieldName;
    
    // 关键：system scope 下需要还原 'directus_' 前缀
    // GraphQL 查询字段名是 'users'，实际集合名是 'directus_users'
    if (gql.scope === 'system') 
        collection = `directus_${collection}`;
    
    // 后续通过 getService() 获取正确的服务
    // ...
}

// services/graphql/resolvers/mutation.ts
export async function resolveMutation(gql: GraphQLService, args, info) {
    const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
    let collection = info.fieldName.substring(action.length + 1);
    
    // 同样需要还原前缀
    if (gql.scope === 'system') 
        collection = `directus_${collection}`;
    
    // 通过 getService() 获取服务
    const service = getService(collection, {
        knex: gql.knex,
        accountability: gql.accountability,
        schema: gql.schema,
    });
    // ...
}
```

**GraphQL 系统集合特点**：

| 特点 | 说明 |
|------|------|
| **Scope 隔离** | 普通集合走 `/graphql`，系统集合走 `/graphql/system` |
| **命名转换** | 查询字段名去掉 `directus_` 前缀，解析时再还原 |
| **黑名单机制** | `SYSTEM_DENY_LIST` 中的集合完全不暴露 |
| **只读限制** | `READ_ONLY` 列表中的集合不生成 Mutation |
| **统一服务获取** | 通过 `getService()` 工厂函数获取服务 |

#### 最终统一到相同服务调用链

无论 REST 还是 GraphQL，最终都通过 `getService()` 工厂函数或直接实例化专门的 Service 子类，这些子类都继承自 `ItemsService`：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              入口层差异                                       │
├─────────────────────────────────────┬───────────────────────────────────────┤
│           REST API                  │              GraphQL                   │
├─────────────────────────────────────┼───────────────────────────────────────┤
│  controllers/users.ts               │  services/graphql/schema/index.ts     │
│  ├── router.use(useCollection(     │  ├── scopeFilter 过滤集合            │
│  │     'directus_users'))           │  ├── SYSTEM_DENY_LIST 黑名单         │
│  ├── POST /users                    │  └── READ_ONLY 只读限制              │
│  │    new UsersService()            │                                     │
│  ├── POST /users/invite             │  services/graphql/resolvers/         │
│  ├── POST /users/me/tfa             │  ├── query.ts                        │
│  └── ...                            │  │   collection = 'users'            │
│                                     │  │   if (scope === 'system')         │
│                                     │  │       collection = 'directus_users'│
│                                     │  └── mutation.ts                     │
│                                     │      getService(collection)          │
└─────────────────────────────────────┴───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         统一服务层                                             │
│                                                                               │
│   getService('directus_users', opts)                                        │
│       │                                                                       │
│       ▼                                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  switch (collection) {                                              │   │
│   │      case 'directus_users':                                         │   │
│   │          return new UsersService(opts);  // ← 继承 ItemsService   │   │
│   │      case 'directus_files':                                         │   │
│   │          return new FilesService(opts);  // ← 继承 ItemsService   │   │
│   │      case 'directus_activity':                                      │   │
│   │          return new ActivityService(opts); // ← 继承 ItemsService  │   │
│   │      // ...                                                         │   │
│   │      default:                                                        │   │
│   │          return new ItemsService(collection, opts);                │   │
│   │  }                                                                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                        │
│                                      ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     UsersService (继承 ItemsService)                │   │
│   │  ┌─────────────────────────────────────────────────────────────┐   │   │
│   │  │  覆盖的方法：                                                │   │   │
│   │  │  - createOne()     // 添加邮箱唯一性检查、密码策略检查      │   │   │
│   │  │  - createMany()    // 批量用户创建优化                      │   │   │
│   │  │  - updateMany()    // 角色变更、状态变更特殊处理            │   │   │
│   │  │  - deleteMany()    // 管理员剩余数量检查                      │   │   │
│   │  │                                                  │   │   │
│   │  │  新增的方法：                                                │   │   │
│   │  │  - inviteUser()        // 邀请用户                          │   │   │
│   │  │  - acceptInvite()      // 接受邀请                          │   │   │
│   │  │  - registerUser()      // 用户注册                          │   │   │
│   │  │  - requestPasswordReset() // 请求密码重置                   │   │   │
│   │  │  - resetPassword()     // 重置密码                          │   │   │
│   │  └─────────────────────────────────────────────────────────────┘   │   │
│   │                              │                                        │   │
│   │                              ▼                                        │   │
│   │  ┌─────────────────────────────────────────────────────────────┐   │   │
│   │  │              ItemsService (父类，核心实现)                   │   │   │
│   │  │  - readByQuery() / readOne() / readMany()                   │   │   │
│   │  │  - createOne() / createMany()                                │   │   │
│   │  │  - updateOne() / updateMany() / updateByQuery()             │   │   │
│   │  │  - deleteOne() / deleteMany() / deleteByQuery()             │   │   │
│   │  │  - ... 以及权限检查、Hooks、缓存清除等通用逻辑               │   │   │
│   │  └─────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

**REST 与 GraphQL 系统集合入口对比**：

| 维度 | REST API | GraphQL |
|------|---------|---------|
| **路由设计** | 每个系统集合独立控制器文件 | 通过 scope 动态生成 Schema |
| **集合命名** | 直接使用完整名 `directus_users` | 查询时去掉前缀，解析时还原 |
| **服务实例化** | 直接 `new UsersService()` | 通过 `getService()` 工厂 |
| **额外端点** | 支持 `/users/invite` 等自定义端点 | 无自定义端点，只能通过查询/变更 |
| **只读限制** | 控制器层面不暴露 POST/PATCH/DELETE | Schema 生成时不生成 Mutation |
| **黑名单** | 无（各控制器独立控制） | `SYSTEM_DENY_LIST` 完全排除 |
| **访问入口** | `/users`、`/files` 等独立路径 | `/graphql/system` 统一入口 |

---

## 三、ItemsService 核心实现分析

### 3.1 类结构与依赖

```typescript
export class ItemsService<Item extends AnyItem = AnyItem> implements AbstractService<Item> {
    collection: Collection;           // 当前操作的集合名
    knex: Knex;                       // 数据库连接实例
    accountability: Accountability | null;  // 用户权限上下文
    eventScope: string;               // 事件作用域（用于 hooks）
    schema: SchemaOverview;           // 数据库元数据
    cache: Keyv<any> | null;          // 缓存实例
    nested: string[];                 // 嵌套关系追踪

    constructor(collection: Collection, options: AbstractServiceOptions) {
        this.collection = collection;
        this.knex = options.knex || getDatabase();  // 使用传入的或全局连接
        this.accountability = options.accountability || null;
        this.eventScope = isSystemCollection(this.collection) 
            ? this.collection.substring(9)  // 去掉 'directus_' 前缀
            : 'items';
        this.schema = options.schema;
        this.cache = getCache().cache;
        this.nested = options.nested ?? [];
    }
}
```

### 3.2 CRUD 方法实现模式

#### 查询操作 (`readByQuery`)

```typescript
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
    // 1. 触发 query filter hook（允许修改查询）
    const updatedQuery = opts?.emitEvents !== false
        ? await emitter.emitFilter(
            this.eventScope === 'items'
                ? ['items.query', `${this.collection}.items.query`]
                : `${this.eventScope}.query`,
            query,
            { collection: this.collection },
            { database: this.knex, schema: this.schema, accountability: this.accountability },
        )
        : query;

    // 2. 将 Query 对象转换为 AST
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

    // 3. 应用权限处理（注入权限过滤条件）
    ast = await processAst(
        { ast, action: 'read', accountability: this.accountability },
        { knex: this.knex, schema: this.schema },
    );

    // 4. 执行 AST 获取数据
    const records = await runAst(ast, this.schema, this.accountability, {
        knex: this.knex,
        stripNonRequested: opts?.stripNonRequested !== undefined ? opts.stripNonRequested : true,
    });

    // 5. 触发 read filter hook（允许修改返回数据）
    const filteredRecords = opts?.emitEvents !== false
        ? await emitter.emitFilter(
            this.eventScope === 'items' 
                ? ['items.read', `${this.collection}.items.read`] 
                : `${this.eventScope}.read`,
            records,
            { query: updatedQuery, collection: this.collection },
            { database: this.knex, schema: this.schema, accountability: this.accountability },
        )
        : records;

    // 6. 触发 read action hook（审计、日志等）
    if (opts?.emitEvents !== false) {
        emitter.emitAction(
            this.eventScope === 'items' 
                ? ['items.read', `${this.collection}.items.read`] 
                : `${this.eventScope}.read`,
            { payload: filteredRecords, query: updatedQuery, collection: this.collection },
            { database: getDatabase(), schema: this.schema, accountability: this.accountability },
        );
    }

    return filteredRecords as Item[];
}
```

#### 创建操作 (`createOne`)

```typescript
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    const primaryKeyField = this.schema.collections[this.collection]!.primary;
    
    // 使用事务保证原子性
    const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
        // 1. 触发 create filter hook
        const payloadAfterHooks = opts.emitEvents !== false
            ? await emitter.emitFilter(
                this.eventScope === 'items'
                    ? ['items.create', `${this.collection}.items.create`]
                    : `${this.eventScope}.create`,
                payload,
                { collection: this.collection },
                { database: trx, schema: this.schema, accountability: this.accountability },
            )
            : payload;

        // 2. 处理权限预设值（如 created_by 等自动填充字段）
        const payloadWithPresets = this.accountability
            ? await processPayload(
                {
                    accountability: this.accountability,
                    action: 'create',
                    collection: this.collection,
                    payload: payloadAfterHooks,
                    nested: this.nested,
                },
                { knex: trx, schema: this.schema },
            )
            : payloadAfterHooks;

        // 3. 使用 PayloadService 处理复杂关系数据
        const payloadService = new PayloadService(this.collection, {
            accountability: this.accountability,
            knex: trx,
            schema: this.schema,
            nested: this.nested,
            overwriteDefaults: opts.overwriteDefaults,
        });

        // 处理多对一关系
        const { payload: payloadWithM2O, ... } = await payloadService.processM2O(payloadWithPresets, opts);
        
        // 处理任对一关系
        const { payload: payloadWithA2O, ... } = await payloadService.processA2O(payloadWithM2O, opts);
        
        // 类型转换
        const payloadWithTypeCasting = await payloadService.processValues('create', payloadWithoutAliases);

        // 4. 执行数据库插入
        try {
            const result = await trx
                .insert(payloadWithoutAliases)
                .into(this.collection)
                .returning(primaryKeyField, returningOptions)
                .then((result) => result[0]);
            // ...
        } catch (err: any) {
            const dbError = await translateDatabaseError(err, data);
            throw dbError;
        }

        // 5. 处理一对多关系（需要先有主键）
        const { ... } = await payloadService.processO2M(payloadWithPresets, primaryKey, opts);

        // 6. 创建活动记录（如果启用了 accountability）
        if (opts.skipTracking !== true && this.accountability && ...) {
            const activityService = new ActivityService({ knex: trx, schema: this.schema });
            const activity = await activityService.createOne({
                action: Action.CREATE,
                user: this.accountability!.user,
                collection: this.collection,
                // ...
            });

            // 创建修订记录
            if (this.schema.collections[this.collection]!.accountability === 'all') {
                const revisionsService = new RevisionsService({ knex: trx, schema: this.schema });
                // ...
            }
        }

        return primaryKey;
    });

    // 7. 事务外触发 action hook
    if (opts.emitEvents !== false) {
        emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
    }

    // 8. 清除缓存
    if (shouldClearCache(this.cache, opts, this.collection)) {
        await this.cache.clear();
    }

    return primaryKey;
}
```

### 3.3 方法调用层级关系

```
readByQuery (最通用)
  ├── readOne (单条查询)
  │     └── 构建 { filter: { id: { _eq: key } } } 后调用 readByQuery
  │
  ├── readMany (批量查询)
  │     └── 构建 { filter: { _and: [{ id: { _in: keys } }] } } 后调用 readByQuery
  │
  └── readSingleton (单例读取)
        └── 设置 limit: 1 后调用 readByQuery

createOne (单条创建)
  └── createMany (批量创建)
        └── 循环调用 createOne，在同一事务中

updateOne (单条更新)
  ├── updateMany (按主键批量更新)
  │     └── 核心更新逻辑
  │
  ├── updateBatch (批量不同更新)
  │     └── 循环调用 updateOne
  │
  └── updateByQuery (按查询更新)
        ├── getKeysByQuery (获取符合条件的主键)
        │     └── 调用 readByQuery 只查询主键
        │
        └── updateMany (按获取的主键更新)

deleteOne (单条删除)
  ├── deleteMany (批量删除)
  │     └── 核心删除逻辑
  │
  └── deleteByQuery (按查询删除)
        ├── getKeysByQuery
        └── deleteMany

upsertOne (单条 upsert)
  ├── 查询是否存在
  ├── 存在则 updateOne
  └── 不存在则 createOne

upsertSingleton (单例 upsert)
  └── 逻辑同上，但针对单例集合
```

---

## 四、数据库适配层分析

### 4.1 数据库连接管理 (`database/index.ts`)

#### 连接初始化

```typescript
export function getDatabase(): Knex {
    if (database) return database;  // 单例模式

    const env = useEnv();
    
    // 从环境变量提取配置
    const {
        client,
        version,
        searchPath,
        connectionString,
        pool: poolConfig = {},
        ...connectionConfig
    } = getConfigFromEnv('DB_', { omitPrefix: 'DB_EXCLUDE_TABLES' });

    // 构建 Knex 配置
    const knexConfig: Knex.Config = {
        client,
        version,
        searchPath,
        connection: connectionString || connectionConfig,
        log: { /* 日志处理 */ },
        pool: poolConfig,
    };

    // 数据库特定配置
    if (client === 'sqlite3') {
        knexConfig.useNullAsDefault = true;
        poolConfig.afterCreate = (conn: any, callback: any) => {
            // SQLite: 启用外键约束
            conn.run('PRAGMA foreign_keys = ON');
            callback(null, conn);
        };
    }

    if (client === 'cockroachdb') {
        poolConfig.afterCreate = (conn: any, callback: any) => {
            // CockroachDB: 设置序列和整数类型
            conn.query('SET serial_normalization = "sql_sequence"');
            conn.query('SET default_int_size = 4');
            callback(null, conn);
        };
    }

    if (client === 'oracledb') {
        poolConfig.afterCreate = (conn: any, callback: any) => {
            // Oracle: 设置日期格式
            conn.execute('ALTER SESSION SET NLS_TIMESTAMP_FORMAT = \'YYYY-MM-DD"T"HH24:MI:SS.FF3"Z"\'');
            conn.execute("ALTER SESSION SET NLS_DATE_FORMAT = 'YYYY-MM-DD'");
            callback(null, conn);
        };
    }

    if (client === 'mysql') {
        // MySQL: 使用 mysql2 驱动
        Object.assign(knexConfig, { client: 'mysql2' });
    }

    if (client === 'mssql') {
        // MSSQL: 禁用自动时区转换
        merge(knexConfig, { connection: { options: { useUTC: false } } });
    }

    database = knex.default(knexConfig);
    
    // 添加查询监控
    database
        .on('query', ({ __knexUid }) => {
            times.set(__knexUid, performance.now());
        })
        .on('query-response', (_response, queryInfo) => {
            const delta = performance.now() - times.get(queryInfo.__knexUid);
            metrics?.getDatabaseResponseMetric()?.observe(delta);
            logger.trace(`[${delta.toFixed(3)}ms] ${queryInfo.sql} [...]`);
        });

    return database;
}
```

#### 数据库类型检测

```typescript
export function getDatabaseClient(database?: Knex): DatabaseClient {
    database = database ?? getDatabase();

    // 根据 Knex 客户端构造函数名判断
    switch (database.client.constructor.name) {
        case 'Client_MySQL2':
            return 'mysql';
        case 'Client_PG':
            return 'postgres';
        case 'Client_CockroachDB':
            return 'cockroachdb';
        case 'Client_SQLite3':
            return 'sqlite';
        case 'Client_Oracledb':
        case 'Client_Oracle':
            return 'oracle';
        case 'Client_MSSQL':
            return 'mssql';
        case 'Client_Redshift':
            return 'redshift';
    }

    throw new Error(`Couldn't extract database client`);
}
```

### 4.2 方言辅助系统 (`database/helpers/`)

#### 辅助函数工厂

```typescript
// database/helpers/index.ts
export function getHelpers(database: Knex) {
    const client = getDatabaseClient(database);

    return {
        date: new dateHelpers[client](database),           // 日期函数
        st: new geometryHelpers[client](database),          // 空间几何函数
        schema: new schemaHelpers[client](database),        // 模式操作
        sequence: new sequenceHelpers[client](database),    // 序列管理
        number: new numberHelpers[client](database),        // 数值处理
        capabilities: new capabilitiesHelpers[client](database),  // 能力检测
    };
}
```

#### 方言实现目录结构

```
database/helpers/
├── index.ts              # 工厂函数
├── types.ts              # 类型定义
├── capabilities/         # 数据库能力检测
│   ├── index.ts
│   └── dialects/
│       ├── default.ts
│       ├── mysql.ts
│       ├── postgres.ts
│       ├── sqlite.ts
│       └── ...
├── date/                 # 日期函数
│   ├── index.ts
│   └── dialects/
│       ├── default.ts
│       ├── mysql.ts
│       ├── oracle.ts
│       ├── postgres.ts
│       ├── sqlite.ts
│       └── mssql.ts
├── fn/                   # 通用函数（JSON 等）
│   ├── index.ts
│   └── dialects/
├── geometry/             # 空间几何函数
│   ├── index.ts
│   └── dialects/
│       ├── mysql.ts      # MySQL: ST_* 函数
│       ├── postgres.ts   # PostGIS
│       ├── sqlite.ts     # SpatiaLite
│       └── ...
├── number/               # 数值处理（大数、精度）
│   ├── index.ts
│   └── dialects/
├── schema/               # 模式操作
│   ├── index.ts
│   ├── types.ts
│   └── dialects/
│       ├── default.ts
│       ├── cockroachdb.ts
│       ├── mssql.ts
│       ├── mysql.ts
│       ├── oracle.ts
│       ├── postgres.ts
│       └── sqlite.ts
└── sequence/             # 序列管理（自增重置等）
    ├── index.ts
    ├── types.ts
    └── dialects/
        ├── default.ts
        └── postgres.ts
```

#### 示例：日期函数方言差异

```typescript
// PostgreSQL 实现 (date/dialects/postgres.ts)
export class DateHelperPostgres extends DateHelper {
    dateField(field: string): Knex.Raw {
        // PostgreSQL: 日期转换
        return this.knex.raw(`??::date`, [field]);
    }

    dateFieldAsISO(field: string): Knex.Raw {
        // ISO 格式转换
        return this.knex.raw(`to_char(??::date, 'YYYY-MM-DD')`, [field]);
    }

    timestampFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`to_char(?? at time zone 'UTC', 'YYYY-MM-DD"T"HH24:MI:SS"Z"')`, [field]);
    }
}

// MySQL 实现 (date/dialects/mysql.ts)
export class DateHelperMySQL extends DateHelper {
    dateField(field: string): Knex.Raw {
        // MySQL: 日期转换
        return this.knex.raw(`date(??)`, [field]);
    }

    dateFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`date_format(??, '%Y-%m-%d')`, [field]);
    }

    timestampFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`date_format(convert_tz(??, @@session.time_zone, '+00:00'), '%Y-%m-%dT%H:%i:%sZ')`, [field]);
    }
}

// SQLite 实现 (date/dialects/sqlite.ts)
export class DateHelperSQLite extends DateHelper {
    dateField(field: string): Knex.Raw {
        // SQLite: 使用内置日期函数
        return this.knex.raw(`date(??)`, [field]);
    }

    dateFieldAsISO(field: string): Knex.Raw {
        return this.knex.raw(`strftime('%Y-%m-%d', ??)`, [field]);
    }

    timestampFieldAsISO(field: string): Knex.Raw {
        // SQLite: 处理 Unix 时间戳和 ISO 字符串
        return this.knex.raw(
            `case
                when typeof(??) = 'integer' then strftime('%Y-%m-%dT%H:%M:%SZ', datetime(??, 'unixepoch'))
                else strftime('%Y-%m-%dT%H:%M:%SZ', ??)
            end`,
            [field, field, field]
        );
    }
}
```

#### 示例：序列重置

```typescript
// PostgreSQL 特定实现 (sequence/dialects/postgres.ts)
export class SequenceHelperPostgres extends SequenceHelper {
    async resetAutoIncrementSequence(collection: string, pkField: string): Promise<void> {
        // 获取序列名
        const [row] = await this.knex
            .select(this.knex.raw('pg_get_serial_sequence(?, ?) as sequence'), [collection, pkField])
            .from(this.knex.raw('(VALUES (1)) as temp(val)'));

        if (!row.sequence) return;

        // 计算新的序列值（max(pk) + 1）
        const maxValueResult = await this.knex.max(pkField, { as: 'max' }).from(collection);
        const maxValue = maxValueResult[0]?.max;

        const newSequenceValue = typeof maxValue === 'number' ? maxValue + 1 : 1;

        // 重置序列
        await this.knex.raw('SELECT setval(?, ?, false)', [row.sequence, newSequenceValue]);
    }
}

// 默认实现（大多数数据库不需要或有其他方式）
export class SequenceHelperDefault extends SequenceHelper {
    async resetAutoIncrementSequence(): Promise<void> {
        // 默认不做任何操作
        return;
    }
}
```

#### 补充一：fn 函数辅助系统的深度分析

`fn` helper 是 Directus 中处理字段函数的核心方言适配系统，它直接参与 SELECT 语句的字段生成。

**fn helper 在查询管道中的调用位置**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           查询执行管道中的 fn helper                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ItemsService.readByQuery(query)                                            │
│         │                                                                    │
│         ▼                                                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  getAstFromQuery()  ──► 构建 AST                                       │ │
│  │         │                                                               │ │
│  │         ▼                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  FunctionFieldNode 处理                                           │   │ │
│  │  │  {                                                                 │   │ │
│  │  │    type: 'fn',                                                     │   │ │
│  │  │    fn: 'year' | 'json' | 'count'...,                              │   │ │
│  │  │    fieldKey: 'year_date_created',                                 │   │ │
│  │  │    path: ['date_created']                                          │   │ │
│  │  │  }                                                                 │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  runAst(ast, schema, ...)                                             │ │
│  │         │                                                               │ │
│  │         ▼                                                               │ │
│  │  parseCurrentLevel()  ──► 解析出 fieldNodes                           │ │
│  │         │                                                               │ │
│  │         ▼                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  getDBQuery()                                                    │   │ │
│  │  │    ├── flatQuery.select(fieldNodes.map((node) => preProcess(node)))│ │ │
│  │  │    │                                                              │   │ │
│  │  │    ▼                                                              │   │ │
│  │  │  ┌─────────────────────────────────────────────────────────────┐ │   │ │
│  │  │  │  getColumnPreprocessor()                                   │ │   │ │
│  │  │  │    └── getColumn(knex, table, column, alias, schema, ...)│ │   │ │
│  │  │  │           │                                                 │ │   │ │
│  │  │  │           ▼                                                 │ │   │ │
│  │  │  │  ┌───────────────────────────────────────────────────────┐ │ │   │ │
│  │  │  │  │  getFunctions(knex, schema)  →  FnHelper 实例       │ │ │   │ │
│  │  │  │  │                                                   │ │   │ │
│  │  │  │  │  // 核心：调用方言特定的函数实现                        │ │   │ │
│  │  │  │  │  result = fn[functionName](table, fieldName, options) │ │   │ │
│  │  │  │  │                                                   │ │   │ │
│  │  │  │  │  // 例如：                                       │ │   │ │
│  │  │  │  │  // PostgreSQL: fn.year() → EXTRACT(YEAR FROM ...)   │ │   │ │
│  │  │  │  │  // MySQL:      fn.year() → YEAR(...)                 │ │   │ │
│  │  │  │  │  // SQLite:     fn.year() → CAST(strftime(...) AS INT)│ │   │ │
│  │  │  │  └───────────────────────────────────────────────────────┘ │ │   │ │
│  │  │  └─────────────────────────────────────────────────────────────┘ │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**getColumn 函数的完整实现逻辑** (`run-ast/utils/get-column.ts`):

```typescript
export function getColumn(
    knex: Knex,
    table: string,
    column: string,
    alias: string | false = applyFunctionToColumnName(column),
    schema: SchemaOverview,
    options?: GetColumnOptions,
): Knex.Raw {
    // Step 1: 获取当前数据库的 fn helper 实例
    const fn = getFunctions(knex, schema);

    // Step 2: 检查是否是函数调用格式，如 "year(date_created)"
    if (column.includes('(') && column.includes(')')) {
        const functionName = column.split('(')[0] as FieldFunction;
        const columnName = column.match(REGEX_BETWEEN_PARENS)![1];

        // Step 3: 验证函数是否在 fn helper 中实现
        if (functionName in fn) {
            const collectionName = options?.originalCollectionName || table;
            
            // Step 4: 特殊处理 json 函数的路径解析
            let fieldName = columnName!;
            let jsonPath: string | undefined;
            
            if (functionName === 'json') {
                const result = parseJsonFunction(column);
                fieldName = result.field;
                jsonPath = result.path;
            }

            // Step 5: 验证字段类型是否支持该函数
            const type = schema?.collections[collectionName]?.fields?.[fieldName]?.type ?? 'unknown';
            const allowedFunctions = getFunctionsForType(type);
            
            if (allowedFunctions.includes(functionName) === false) {
                throw new InvalidQueryError({ 
                    reason: `Invalid function specified "${functionName}"` 
                });
            }

            // Step 6: 调用方言特定的函数实现
            const result = fn[functionName as keyof typeof fn](table, fieldName, {
                type,  // 字段类型，用于时区处理等
                relationalCountOptions: isFunctionColumnOptions(options)
                    ? {
                        query: options.query,
                        cases: options.cases,
                        permissions: options.permissions,
                    }
                    : undefined,
                originalCollectionName: options?.originalCollectionName,
                jsonPath,  // 仅用于 json() 函数
            }) as Knex.Raw;

            // Step 7: 添加 AS 别名
            if (alias) {
                return knex.raw(result + ' AS ??', [alias]);
            }

            return result;
        }
        // ...
    }
    // ...
}
```

**字段类型与允许的函数映射**：

```typescript
export function getFunctionsForType(type: string): string[] {
    switch (type) {
        case 'alias':
            return ['count'];
        case 'dateTime':
        case 'timestamp':
        case 'date':
            return ['year', 'month', 'week', 'day', 'weekday', 'hour', 'minute', 'second'];
        case 'json':
            return ['count', 'json'];
        default:
            return [];
    }
}
```

**fn helper 不同数据库实现的关键差异**：

| 数据库 | year() 实现 | 时区处理 | timestamp 格式 | JSON 路径语法 |
|--------|------------|---------|---------------|--------------|
| **PostgreSQL** | `EXTRACT(YEAR FROM ...)` | `AT TIME ZONE 'UTC'` (仅 timestamp 类型) | 驱动自动解析 | `->` 操作符链 |
| **MySQL** | `YEAR(...)` | `CONVERT_TZ(..., @@session.time_zone, '+00:00')` | 手动 `JSON.parse` | `JSON_EXTRACT` + 路径字符串 |
| **SQLite** | `CAST(strftime('%Y', ...) AS INTEGER)` | `strftime` 的 `'unixepoch'` 修饰符 | 手动 `JSON.parse` | `json_extract` + 路径字符串 |

**PostgreSQL 时区处理的条件逻辑**：

```typescript
const parseLocaltime = (columnType?: string) => {
    if (columnType === 'timestamp') {
        return ` AT TIME ZONE 'UTC'`;
    }
    return '';
};

// 使用时：
return this.knex.raw(
    `EXTRACT(YEAR FROM ??.??${parseLocaltime(options?.type)})`,
    [table, column]
);
```

这意味着：
- `timestamp` 类型字段：需要 `AT TIME ZONE 'UTC'` 转换
- `dateTime` 类型字段：不需要额外转换

**SQLite Unix 时间戳处理**：

```typescript
const parseLocaltime = (columnType?: string) => {
    if (columnType === 'timestamp') {
        return '';  // 已经是 Unix 时间
    }
    return `, 'localtime'`;
};

// year() 实现：
return this.knex.raw(
    `CAST(strftime('%Y', ??.?? / 1000, 'unixepoch'${parseLocaltime(options?.type)}) AS INTEGER)`,
    [table, column]
);
```

**关键注意点**：
1. SQLite 中 `timestamp` 类型存储为 **Unix 毫秒**，需要 `/ 1000` 转换
2. SQLite 中 `dateTime` 类型存储为 ISO 字符串，需要 `'localtime'` 修饰符

**JSON 路径解析的差异**：

```typescript
// PostgreSQL: 构建操作符链
// ".items[0].name" → "->'items'->0->'name'"
const { template, bindings } = buildPostgresJsonPath(options.jsonPath);

// MySQL: 转换为 JSONPath 语法
// ".items[0].name" → "$.items[0].name"
const jsonPath = convertToMySQLPath(options.jsonPath);

// SQLite: 直接添加 $ 前缀
// ".items[0].name" → "$.items[0].name"
const jsonPath = '$' + options.jsonPath;
```

---

#### 补充二：capabilities 能力检测系统的深度分析

`capabilities` helper 不直接生成 SQL，而是通过**条件分支**影响查询构建策略。

**两个 capabilities 方法的组合逻辑** (`get-db-query.ts`):

```typescript
// 在处理聚合查询和分组查询时
if (queryCopy.aggregate || queryCopy.group) {
    // ... 构建查询 ...

    // 关键：两个条件的组合判断
    if (
        helpers.capabilities.supportsDeduplicationOfParameters() &&
        !helpers.capabilities.supportsColumnPositionInGroupBy()
    ) {
        withPreprocessBindings(knex, dbQuery);
    }
}
```

**决策矩阵**：

| 数据库 | supportsDeduplication | supportsColumnPositionInGroupBy | 是否调用 withPreprocessBindings |
|--------|----------------------|---------------------------------|---------------------------------|
| PostgreSQL | ❌ | ✅ | ❌ |
| CockroachDB | ❌ | ✅ | ❌ |
| MySQL | ✅ | ❌ | ✅ |
| MSSQL | ✅ | ❌ | ✅ |
| Oracle | ✅ | ❌ | ✅ |
| SQLite | ✅ | ❌ | ✅ |

**withPreprocessBindings 的实际作用**：

```typescript
export function withPreprocessBindings(knex: Knex, dbQuery: Knex.QueryBuilder) {
    const schemaHelper = getHelpers(knex).schema;

    // 使用 Proxy 拦截 Knex 客户端的方法
    dbQuery.client = new Proxy(dbQuery.client, {
        get(target, prop, receiver) {
            // 拦截查询执行
            if (prop === 'query') {
                return (connection: Knex, queryParams: Knex.Sql) =>
                    Reflect.get(target, prop, receiver).bind(dbQuery.client)(
                        connection,
                        schemaHelper.prepQueryParams(queryParams),  // 预处理查询参数
                    );
            }

            // 拦截绑定处理
            if (prop === 'prepBindings') {
                return (bindings: Knex.Value[]) =>
                    schemaHelper.prepBindings(
                        Reflect.get(target, prop, receiver).bind(dbQuery.client)(bindings)
                    );
            }

            return Reflect.get(target, prop, receiver);
        },
    });
}
```

**prepQueryParams 的核心实现** (`schema/utils/prep-query-params.ts`):

```typescript
export function prepQueryParams(
    queryParams: (Partial<Sql> & Pick<Sql, 'sql'>) | string,
    options: PrepQueryParamsOptions,  // { format: (index) => string }
) {
    const query: Sql = { bindings: [], ...(isString(queryParams) ? { sql: queryParams } : queryParams) };

    // 构建绑定值去重映射
    // bindingIndices[value] = 首次出现的索引
    const bindingIndices = new Map<Knex.Value, number>();
    
    // 去重后的绑定值数组
    const bindings: Knex.Value[] = [];

    let matchIndex = 0;
    let nextBindingIndex = 0;

    // 替换 SQL 中的 ? 占位符
    const sql = query.sql.replace(/(\\*)(\?)/g, (_, escapes) => {
        if (escapes.length % 2) {
            // 转义的问号，保持不变
            return `${'\\'.repeat(escapes.length)}?`;
        }

        const binding = query.bindings[matchIndex]!;
        let bindingIndex: number;

        if (bindingIndices.has(binding)) {
            // 已存在的值，使用之前的索引
            bindingIndex = bindingIndices.get(binding)!;
        } else {
            // 新值，分配新索引
            bindingIndex = nextBindingIndex++;
            bindingIndices.set(binding, bindingIndex);
            bindings.push(binding);
        }

        matchIndex++;
        // 使用数据库特定的格式（@p1, :1, 等）
        return options.format(bindingIndex);
    });

    return { ...query, sql, bindings };
}
```

**不同数据库的占位符转换**：

| 数据库 | prepQueryParams 实现 | 占位符转换 |
|--------|---------------------|-----------|
| **MSSQL** | `prepQueryParams(queryParams, { format: (index) => `@p${index}` })` | `?` → `@p0`, `@p1`, `@p2`... |
| **Oracle** | `prepQueryParams(queryParams, { format: (index) => `:${index + 1}` })` | `?` → `:1`, `:2`, `:3`... |
| **MySQL** | 默认不转换 | 保持 `?` |
| **PostgreSQL** | 默认不转换 | 保持 `$1`, `$2`... |

**Oracle 的特殊处理**：

```typescript
override prepBindings(bindings: Knex.Value[]): any {
    // 将数组转换为对象 { 1: value1, 2: value2, ... }
    // 使用 "命名" 绑定语法而非位置绑定
    return Object.fromEntries(
        bindings.map((binding: any, index: number) => [index + 1, binding])
    );
}
```

**为什么 PostgreSQL 不支持参数去重**：

```typescript
// PostgreSQL 从参数首次引用的上下文推断类型
// 这可能导致问题：

// 假设查询：
// SELECT * FROM table 
// WHERE uuid_column = $1 AND string_column = $1

// PostgreSQL 会推断 $1 为 UUID 类型（从 uuid_column）
// 但 string_column 比较需要字符串类型
// 这会导致类型不匹配错误！

// 解决方案：使用独立的参数引用
// SELECT * FROM table 
// WHERE uuid_column = $1 AND string_column = $2
// bindings: [value, value]
```

---

#### 补充三：supportsColumnPositionInGroupBy 对分组查询的影响

这个能力检测在 `apply-query/index.ts` 中影响 GROUP BY 子句的构建：

```typescript
if (query.group) {
    const helpers = getHelpers(knex);
    const rawColumns = query.group.map((column) => 
        getColumn(knex, collection, column, false, schema)
    );
    let columns;

    if (options?.groupWhenCases) {
        // 关键：能力检测决定构建策略
        if (helpers.capabilities.supportsColumnPositionInGroupBy() && options.groupColumnPositions) {
            // PostgreSQL / CockroachDB: 使用列位置
            columns = query.group.map((column, index) =>
                options.groupColumnPositions![index] !== undefined 
                    ? knex.raw(options.groupColumnPositions![index])  // e.g., 1, 2, 3
                    : column,
            );
        } else {
            // MySQL / MSSQL / Oracle / SQLite: 重建完整列表达式
            columns = rawColumns.map((column, index) =>
                applyCaseWhen(
                    {
                        columnCases: options.groupWhenCases![index]!.map(
                            (caseIndex) => cases[caseIndex]!
                        ),
                        column,
                        aliasMap,
                        cases,
                        table: collection,
                        permissions,
                    },
                    { knex, schema },
                ),
            );
        }
        // ...
    } else {
        columns = rawColumns;
    }

    dbQuery.groupBy(columns);
}
```

**groupColumnPositions 的计算** (`get-db-query.ts`):

```typescript
// Map the group field to their respective select column positions (1 based, offset by the number of aggregate terms)
// 分组字段位置 = 索引 + 1 + 聚合字段数量
// 因为聚合字段在 SELECT 中排在前面
const groupColumnPositions = queryCopy.group?.map(
    (field) => fieldNodeMap[field]![1] + 1 + aggregateCount
) ?? [];
```

**两种策略的 SQL 差异**：

假设查询：
```
fields: ['status', 'count(id) as total']
group: ['status']
```

**PostgreSQL / CockroachDB 策略**：
```sql
-- 使用列位置
SELECT status, COUNT(id) AS total
FROM articles
GROUP BY 1  -- 1 表示第一个 SELECT 列
ORDER BY 1
```

**MySQL / MSSQL / Oracle / SQLite 策略**：
```sql
-- 使用完整列表达式
SELECT status, COUNT(id) AS total
FROM articles
GROUP BY status
ORDER BY status
```

**当存在权限 case/when 时的差异**：

假设用户有条件权限（根据某些条件才能读取字段）：

**PostgreSQL 策略**：
```sql
-- 使用列位置，不需要重复 case/when 逻辑
SELECT 
    CASE WHEN <condition> THEN status ELSE NULL END AS status,
    COUNT(id) AS total
FROM articles
GROUP BY 1  -- 简单、高效
```

**MySQL 策略**：
```sql
-- 需要完整的 case/when 表达式
SELECT 
    CASE WHEN <condition> THEN status ELSE NULL END AS status,
    COUNT(id) AS total
FROM articles
GROUP BY 
    CASE WHEN <condition> THEN status ELSE NULL END  -- 重复 case/when
```

---

#### 补充四：addInnerSortFieldsToGroupBy 的数据库差异

这是 `schema` helper 中的方法，用于处理**内部查询排序字段与 GROUP BY 的关系**：

```typescript
// 调用位置：get-db-query.ts
helpers.schema.addInnerSortFieldsToGroupBy(
    groupByFields,
    innerQuerySortRecords,
    (hasMultiRelationalSort || sortRecords?.some(({ column }) => column.includes('.'))) ?? false,
);
```

**各数据库实现对比**：

| 数据库 | hasRelationalSort=true | hasRelationalSort=false | 排序字段引用 |
|--------|------------------------|-------------------------|-------------|
| **PostgreSQL** | 添加别名 | 不添加 | `alias`（字符串） |
| **MySQL** | 添加别名 | 不添加 | `alias`（字符串） |
| **CockroachDB** | 添加别名 | 不添加 | `alias`（字符串） |
| **MSSQL** | **始终添加** | **始终添加** | `column`（Knex.Raw） |
| **Oracle** | **始终添加** | **始终添加** | `column`（Knex.Raw） |
| **SQLite** | 不添加 | 不添加 | - |

**PostgreSQL 实现**：
```typescript
override addInnerSortFieldsToGroupBy(
    groupByFields: (string | Knex.Raw)[],
    sortRecords: SortRecord[],
    hasRelationalSort: boolean,
) {
    if (hasRelationalSort) {
        // 只在有关系排序时添加
        // 使用别名（因为 PostgreSQL 支持 GROUP BY 别名）
        groupByFields.push(...sortRecords.map(({ alias }) => alias));
    }
}
```

**MSSQL 实现**：
```typescript
override addInnerSortFieldsToGroupBy(
    groupByFields: (string | Knex.Raw)[],
    sortRecords: SortRecord[],
    _hasRelationalSort: boolean,
) {
    // MSSQL 要求所有未聚合的 SELECT 列都在 GROUP BY 中
    // 且 MSSQL 不支持 GROUP BY 别名
    if (sortRecords.length > 0) {
        // 始终添加，使用原始列表达式
        groupByFields.push(...sortRecords.map(({ column }) => column));
    }
}
```

**SQLite 实现**：
```typescript
override addInnerSortFieldsToGroupBy() {
    // SQLite 不需要特殊处理
}
```

---

#### 补充五：数据库方言差异对最终 SQL 的影响汇总

**相同查询在不同数据库上的 SQL 差异**：

假设查询：
```
GET /items/articles?fields=year(date_created),json(metadata,color)&group=status
```

**PostgreSQL 生成的 SQL**：
```sql
SELECT
    EXTRACT(YEAR FROM "articles"."date_created" AT TIME ZONE 'UTC') AS "year_date_created",
    "articles"."metadata"::jsonb->'color' AS "json_metadata_color",
    "articles"."status"
FROM "articles"
GROUP BY 1, 2, 3  -- 使用列位置
```

**MySQL 生成的 SQL**：
```sql
SELECT
    YEAR(`articles`.`date_created`) AS `year_date_created`,
    JSON_UNQUOTE(JSON_EXTRACT(`articles`.`metadata`, '$.color')) AS `json_metadata_color`,
    `articles`.`status`
FROM `articles`
GROUP BY `articles`.`status`, `year_date_created`, `json_metadata_color`  -- 使用列名/别名
-- MySQL 支持 GROUP BY 别名
```

**SQLite 生成的 SQL**：
```sql
SELECT
    CAST(strftime('%Y', `articles`.`date_created` / 1000, 'unixepoch') AS INTEGER) AS `year_date_created`,
    json_extract(`articles`.`metadata`, '$.color') AS `json_metadata_color`,
    `articles`.`status`
FROM `articles`
GROUP BY `articles`.`status`, `year_date_created`, `json_metadata_color`
```

---

#### 补充六：关键能力检测的完整决策流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   查询构建时的能力检测决策流程                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  进入 getDBQuery()                                                          │
│         │                                                                    │
│         ▼                                                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  是否有 aggregate 或 group？                                           │ │
│  │         │                                                               │ │
│  │    Yes ────────────── No                                               │ │
│  │     │                  │                                               │ │
│  │     ▼                  ▼                                               │ │
│  │  ┌─────────────┐   普通查询路径                                        │ │
│  │  │ 聚合/分组查询│                                                      │ │
│  │  └──────┬──────┘                                                      │ │
│  │         │                                                               │ │
│  │         ▼                                                               │ │
│  │  计算 groupColumnPositions                                              │ │
│  │  (列位置 = 索引 + 1 + 聚合数量)                                        │ │
│  │         │                                                               │ │
│  │         ▼                                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  supportsDeduplicationOfParameters()  &&                         │   │ │
│  │  │  !supportsColumnPositionInGroupBy()                              │   │ │
│  │  │         │                                                         │   │ │
│  │  │    Yes ────────────── No                                          │   │ │
│  │  │     │                  │                                          │   │ │
│  │  │     ▼                  ▼                                          │   │ │
│  │  │  调用                  不调用                                      │   │ │
│  │  │  withPreprocessBindings                                           │   │ │
│  │  │         │                                                         │   │ │
│  │  │         ▼                                                         │   │ │
│  │  │  占位符转换：                                                     │   │ │
│  │  │  - MSSQL: ? → @p0, @p1...                                       │   │ │
│  │  │  - Oracle: ? → :1, :2...  + 绑定值对象化                         │   │ │
│  │  │  - 绑定值去重                                                     │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  进入 applyQuery()                                                          │
│         │                                                                    │
│         ▼                                                                    │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  是否有 query.group？                                                 │ │
│  │         │                                                               │ │
│  │    Yes ────────────── No                                               │ │
│  │     │                  │                                               │ │
│  │     ▼                  ▼                                               │ │
│  │  ┌─────────────────────────────────────────────────────────────────┐   │ │
│  │  │  supportsColumnPositionInGroupBy()                               │   │ │
│  │  │         │                                                         │   │ │
│  │  │    Yes ────────────── No                                          │   │ │
│  │  │     │                  │                                          │   │ │
│  │  │     ▼                  ▼                                          │   │ │
│  │  │  GROUP BY 1, 2, 3     GROUP BY column_name                       │   │ │
│  │  │  (列位置)              (完整列表达式)                              │   │ │
│  │  │                                                               │   │ │
│  │  │  如果有 groupWhenCases:   如果有 groupWhenCases:                 │   │ │
│  │  │    使用位置，无需重复       重建完整的 case/when 表达式           │   │ │
│  │  │    case/when              (可能很复杂)                           │   │ │
│  │  └─────────────────────────────────────────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│         │                                                                    │
│         ▼                                                                    │
│  是否需要内部查询 + CASE/WHEN 权限处理？                                    │
│         │                                                                    │
│    Yes ────────────── No                                                    │
│     │                  │                                                     │
│     ▼                  ▼                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  addInnerSortFieldsToGroupBy()                                        │ │
│  │         │                                                               │ │
│  │  ┌─────┴─────┬─────┴─────┬─────┴─────┐                               │ │
│  │  │  MSSQL    │  Oracle   │ PostgreSQL│                               │ │
│  │  │           │           │   MySQL   │                               │ │
│  │  │           │           │CockroachDB│                               │ │
│  │  ├───────────┼───────────┼───────────┤                               │ │
│  │  │ 始终添加  │ 始终添加  │ 有关系排序 │                               │ │
│  │  │ 排序字段  │ 排序字段  │ 时才添加  │                               │ │
│  │  │           │           │           │                               │ │
│  │  │ 使用完整  │ 使用完整  │ 使用别名  │                               │ │
│  │  │ 列表达式  │ 列表达式  │           │                               │ │
│  │  └───────────┴───────────┴───────────┘                               │ │
│  │                                                                          │ │
│  │  SQLite: 不添加排序字段到 GROUP BY                                      │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 查询执行管道 (`run-ast/`)

#### AST 到 SQL 的转换流程

```
┌────────────────────────────────────────────────────────────────────────┐
│                           runAst() 执行流程                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 解析当前层级字段                                                     │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  parseCurrentLevel(schema, collection, children, query)    │   │
│     │  ├── fieldNodes: 要查询的字段列表                            │   │
│     │  ├── primaryKeyField: 主键字段名                            │   │
│     │  └── nestedCollectionNodes: 关联关系节点                     │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  2. 获取权限（非管理员）                                                 │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  if (accountability && !accountability.admin) {             │   │
│     │      policies = fetchPolicies(accountability)                │   │
│     │      permissions = fetchPermissions({ action: 'read' })     │   │
│     │  }                                                            │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  3. 构建 Knex 查询                                                       │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  getDBQuery({                                                 │   │
│     │      table, fieldNodes, o2mNodes, query, cases, permissions │   │
│     │  }, { schema, knex })                                         │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  4. 执行查询获取原始数据                                                 │
│     const rawItems: Item | Item[] = await dbQuery;                    │
│                                                                          │
│  5. 数据类型转换                                                         │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  payloadService.processValues('read', rawItems, aliasMap)   │   │
│     │  ├── 日期格式转换                                             │   │
│     │  ├── JSON 解析                                                │   │
│     │  └── 特殊字段类型处理                                         │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  6. 递归处理嵌套关系                                                     │
│     ┌─────────────────────────────────────────────────────────────┐   │
│     │  for (const nestedNode of nestedNodes) {                     │   │
│     │      if (nestedNode.type === 'o2m') {                        │   │
│     │          // 批量处理一对多关系（支持分页）                    │   │
│     │          while (hasMore) {                                    │   │
│     │              nestedItems = runAst(node, ...)                 │   │
│     │              items = mergeWithParentItems(...)               │   │
│     │          }                                                     │   │
│     │      } else {                                                  │   │
│     │          // 处理多对一、任对一关系                            │   │
│     │          nestedItems = runAst(node, ...)                     │   │
│     │          items = mergeWithParentItems(...)                   │   │
│     │      }                                                         │   │
│     │  }                                                             │   │
│     └─────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  7. 移除临时字段（如关联查询所需的外键）                                 │
│     items = removeTemporaryFields(schema, items, originalAST)         │
│                                                                          │
└────────────────────────────────────────────────────────────────────────┘
```

#### getDBQuery 核心实现

```typescript
// database/run-ast/lib/get-db-qy.ts
export function getDBQuery(
    options: {
        table: string;
        fieldNodes: FieldNode[];
        o2mNodes: O2MNode[];
        query: Query;
        cases: Filter[];
        permissions: Permission[];
    },
    context: { schema: SchemaOverview; knex: Knex },
) {
    const { table, fieldNodes, o2mNodes, query, cases, permissions } = options;
    const { schema, knex } = context;

    // 创建基础查询
    let dbQuery = knex.select().from(table);

    // 应用各种查询子句
    // 1. 应用过滤器
    applyFilter(dbQuery, query.filter, { table, cases, permissions, schema, knex });
    
    // 2. 应用排序
    applySort(dbQuery, query.sort, { table, schema, knex });
    
    // 3. 应用分页
    applyPagination(dbQuery, { limit: query.limit, offset: query.offset, page: query.page });
    
    // 4. 应用聚合
    if (query.aggregate) {
        applyAggregate(dbQuery, query.aggregate, { table, schema, knex });
    }
    
    // 5. 应用分组
    if (query.group) {
        applyGroup(dbQuery, query.group, { table, schema, knex });
    }

    // 6. 处理搜索
    if (query.search) {
        applySearch(dbQuery, query.search, { table, schema, knex });
    }

    return dbQuery;
}
```

---

## 五、关键设计模式与架构优势

### 5.1 设计模式应用

| 设计模式 | 应用场景 | 实现位置 |
|---------|---------|---------|
| **Repository 模式** | ItemsService 作为数据访问仓库 | `services/items.ts` |
| **Service 模式** | 业务逻辑封装（ItemsService 及子类） | `services/*` |
| **Factory 模式** | getService() 根据集合类型创建对应服务 | `utils/get-service.ts` |
| **Adapter 模式** | Knex.js 适配不同数据库驱动 | `database/index.ts` |
| **Strategy 模式** | 方言辅助函数（每种数据库一个策略） | `database/helpers/*/dialects/` |
| **Template Method** | readByQuery 固定的 Hook + 执行流程 | `services/items.ts` |
| **Unit of Work** | Transaction 包装多个操作 | `utils/transaction.ts` |
| **Builder 模式** | AST 构建、Knex 查询构建 | `getAstFromQuery`, `getDBQuery` |

### 5.2 架构优势

#### 1. 统一数据访问层
- **单一真相源**：所有数据操作通过 ItemsService，保证业务逻辑一致性
- **易于维护**：修改数据逻辑只需修改一处
- **权限集中**：权限检查在 Service 层统一实施

#### 2. 灵活的数据库适配
- **Knex 抽象**：SQL 查询构建与具体数据库分离
- **方言策略**：数据库特定功能通过 Helper 类实现
- **连接初始化**：每个数据库有专门的初始化逻辑

#### 3. 强大的扩展能力
- **Hooks 系统**：`emitFilter` 和 `emitAction` 允许在操作前后注入逻辑
- **事件驱动**：基于事件的架构便于解耦
- **服务继承**：系统集合可通过继承 ItemsService 添加特殊逻辑

#### 4. 查询灵活性
- **Query 对象**：统一的查询描述格式，支持 REST 和 GraphQL
- **AST 中间表示**：Query → AST → SQL 的多级转换，便于优化和扩展
- **权限注入**：`processAst` 在执行前注入权限条件，数据安全有保障

### 5.3 数据流示例

#### REST 查询流程

```
GET /items/articles?fields=id,title,author(name)&filter[status]=published&limit=10

  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  Express Middleware Chain                                        │
│  ├── collectionExists()                                          │
│  └── sanitizeQuery()  ──► 生成 Query 对象                       │
│         {                                                         │
│           fields: ['id', 'title', { author: ['name'] }],        │
│           filter: { status: { _eq: 'published' } },             │
│           limit: 10                                               │
│         }                                                         │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  REST Controller                                                  │
│  const service = new ItemsService('articles', opts)             │
│  const result = service.readByQuery(sanitizedQuery)             │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  ItemsService.readByQuery()                                      │
│  ├── emitFilter('items.query', query)      [Hook]               │
│  ├── getAstFromQuery()  ──►  AST                               │
│  │         {                                                       │
│  │           type: 'root',                                         │
│  │           name: 'articles',                                     │
│  │           query: { ... },                                       │
│  │           children: [                                            │
│  │               { type: 'field', fieldKey: 'id' },               │
│  │               { type: 'field', fieldKey: 'title' },            │
│  │               {                                                  │
│  │                   type: 'm2o',                                  │
│  │                   fieldKey: 'author',                           │
│  │                   children: [{ type: 'field', fieldKey: 'name' }]│
│  │               }                                                  │
│  │           ]                                                      │
│  │         }                                                        │
│  ├── processAst(ast)  ──► 注入权限条件                          │
│  └── runAst(ast)  ──► 执行查询                                   │
│         ├── getDBQuery() ──► Knex Query Builder                 │
│         │       knex('articles')                                   │
│         │         .select('id', 'title', 'author_id')            │
│         │         .where('status', '=', 'published')              │
│         │         .limit(10)                                       │
│         ├── 执行获取 rawItems                                     │
│         ├── 递归处理 author 关联                                 │
│         │       runAst(author AST)                                │
│         │       ┌─────────────────────────────────────┐          │
│         │       │  knex('directus_users')              │          │
│         │       │    .select('name')                   │          │
│         │       │    .whereIn('id', [1, 2, 3, ...])  │          │
│         │       └─────────────────────────────────────┘          │
│         └── mergeWithParentItems()  ──► 合并关联数据            │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  返回结果                                                         │
│  {                                                                │
│    data: [                                                        │
│      { id: 1, title: 'Article 1', author: { name: 'User A' } },│
│      { id: 2, title: 'Article 2', author: { name: 'User B' } },│
│      ...                                                          │
│    ]                                                              │
│  }                                                                │
└─────────────────────────────────────────────────────────────────┘
```

#### GraphQL 等价查询流程

```graphql
query {
  articles(
    filter: { status: { _eq: "published" } }
    limit: 10
  ) {
    id
    title
    author {
      name
    }
  }
}

  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  GraphQLService.execute()                                        │
│  ├── validate()  ──► 验证查询和变量                            │
│  └── execute()  ──► 执行 GraphQL 解析                          │
└─────────────────────────────────────────────────────────────────┘
  │
  ▼
┌─────────────────────────────────────────────────────────────────┐
│  resolveQuery()                                                  │
│  ├── collection = 'articles'                                     │
│  ├── parseArgs() ──► { filter: ..., limit: 10 }                │
│  └── getQuery() ──► 生成 Query 对象                            │
│         {                                                         │
│           fields: ['id', 'title', { author: ['name'] }],        │
│           filter: { status: { _eq: 'published' } },             │
│           limit: 10                                               │
│         }                              ┌────────────────────────┐
│  └── gql.read('articles', query) ─────┤  ** 与 REST 相同路径  │
│                                         │  const service =       │
│                                         │    getService(...)     │
│                                         │  service.readByQuery() │
│                                         └────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、文件索引

| 功能模块 | 文件路径 | 说明 |
|---------|---------|------|
| **核心服务** | | |
| ItemsService | `api/src/services/items.ts` | 统一数据访问层核心 |
| PayloadService | `api/src/services/payload.ts` | 数据处理、关系处理 |
| ActivityService | `api/src/services/activity.ts` | 操作审计 |
| RevisionsService | `api/src/services/revisions.ts` | 数据版本控制 |
| UsersService | `api/src/services/users.ts` | 用户服务（继承 ItemsService） |
| FilesService | `api/src/services/files.ts` | 文件服务（继承 ItemsService） |
| | | |
| **REST 控制器** | | |
| Items Controller | `api/src/controllers/items.ts` | 集合 REST 端点 |
| | | |
| **GraphQL 层** | | |
| GraphQLService | `api/src/services/graphql/index.ts` | GraphQL 服务入口 |
| Query Resolver | `api/src/services/graphql/resolvers/query.ts` | 查询解析器 |
| Mutation Resolver | `api/src/services/graphql/resolvers/mutation.ts` | 变更解析器 |
| Schema Generator | `api/src/services/graphql/schema/` | 动态 Schema 生成 |
| | | |
| **服务工厂** | | |
| getService | `api/src/utils/get-service.ts` | 服务选择工厂 |
| | | |
| **数据库层** | | |
| 连接管理 | `api/src/database/index.ts` | Knex 连接、客户端检测 |
| Helpers 工厂 | `api/src/database/helpers/index.ts` | 方言辅助函数工厂 |
| 日期函数 | `api/src/database/helpers/date/` | 各数据库日期处理 |
| **fn 函数助手** | `api/src/database/helpers/fn/` | 字段函数（year, json, count 等）方言适配 |
| **capabilities 能力检测** | `api/src/database/helpers/capabilities/` | 数据库高级功能支持检测 |
| 几何函数 | `api/src/database/helpers/geometry/` | 空间数据处理 |
| 模式操作 | `api/src/database/helpers/schema/` | 数据库模式操作 |
| 序列管理 | `api/src/database/helpers/sequence/` | 自增序列管理 |
| 数值处理 | `api/src/database/helpers/number/` | 数值精度处理 |
| | | |
| **查询执行** | | |
| AST 构建 | `api/src/database/get-ast-from-query/` | Query 转 AST |
| AST 执行 | `api/src/database/run-ast/` | AST 转 SQL 并执行 |
| SQL 构建 | `api/src/database/run-ast/lib/get-db-query.ts` | Knex 查询构建器 |
| **列获取与函数处理** | `api/src/database/run-ast/utils/get-column.ts` | 调用 fn helper 生成字段 SQL |
| 过滤器 | `api/src/database/run-ast/lib/apply-query/filter/` | 过滤条件应用 |
| | | |
| **权限处理** | | |
| 权限处理 | `api/src/permissions/modules/process-ast/` | 权限条件注入 |
| 访问验证 | `api/src/permissions/modules/validate-access/` | 权限验证 |

---

## 七、总结

Directus 的架构设计体现了以下核心理念：

### 1. 接口统一
- REST 和 GraphQL 虽然入口不同，但最终都调用相同的 `ItemsService` 方法
- `Query` 对象作为中间表示，抹平了两种 API 的查询语法差异

### 2. 分层清晰
- **API 层**：处理 HTTP/GraphQL 协议
- **Service 层**：业务逻辑、权限、Hooks
- **Query 层**：AST 构建与执行
- **Database 层**：Knex 抽象 + 方言适配

### 3. 数据库无关性
- Knex.js 提供基础 SQL 抽象
- Helper 类处理数据库特定功能
- 连接初始化处理各数据库的特殊需求

### 4. 扩展友好
- 基于事件的 Hooks 系统
- 服务继承机制（系统集合可定制）
- AST 中间层便于查询优化和扩展

这种架构使得 Directus 能够同时支持 REST 和 GraphQL 两种 API，并能适配 8+ 种数据库，同时保持代码的可维护性和扩展性。
