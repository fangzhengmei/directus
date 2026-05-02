# Directus 数据读写架构深度分析报告（R3 - 修正与补充版）

## 1. 事实错误修正

### 1.1 GraphQL 实际路由入口修正

#### 之前的分析（不准确）

> "GraphQL - 单一入口 `/graphql`"

#### 实际情况

GraphQL 有两个路由入口，通过路由前缀区分业务数据和系统数据：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Express 路由挂载层次                           │
│                                                                   │
│  app.ts 中的挂载:                                                  │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  app.use('/graphql', graphqlRouter);  // 前缀: /graphql    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  graphql.ts 中的子路由:                                           │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  router.use('/system', parseGraphQL, ...);  // 系统数据    │ │
│  │  router.use('/', parseGraphQL, ...);         // 业务数据    │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                                                                   │
│  实际完整路由:                                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  /graphql/system  → 系统数据 (users, roles, permissions 等)│ │
│  │  /graphql         → 业务数据 (自定义集合)                   │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### 代码验证

```typescript
// app.ts:323 - 路由挂载
app.use('/graphql', graphqlRouter);

// graphql.ts:7-50 - 子路由定义
const router = Router();

// 系统数据路由: /graphql/system
router.use(
	'/system',
	parseGraphQL,
	asyncHandler(async (req, res, next) => {
		const service = new GraphQLService({
			accountability: req.accountability,
			schema: req.schema,
			scope: 'system',  // 关键: scope = 'system'
		});
		// ...
	}),
	respond,
);

// 业务数据路由: /graphql
router.use(
	'/',
	parseGraphQL,
	asyncHandler(async (req, res, next) => {
		const service = new GraphQLService({
			accountability: req.accountability,
			schema: req.schema,
			scope: 'items',  // 关键: scope = 'items'
		});
		// ...
	}),
	respond,
);
```

#### scope 参数的影响

`scope` 参数会影响 `GraphQLService` 的行为：

```typescript
// graphql/index.ts 中的 scope 使用
export class GraphQLService {
	// ...
	
	async getSchema(): Promise<GraphQLSchema> {
		if (this.scope === 'system') {
			// 系统数据: 使用 directus_* 集合
			// 如: directus_users, directus_roles, directus_permissions 等
			const systemCollections = Object.keys(this.schema.collections).filter((collection) =>
				collection.startsWith('directus_'),
			);
			
			// 构建系统数据的 GraphQL Schema
			return generateGraphQLSchema(systemCollections, this.schema, {
				scope: this.scope,
				excludeFields: this.systemFieldsToExclude,
			});
		} else {
			// 业务数据: 排除 directus_* 集合
			const collections = Object.keys(this.schema.collections).filter(
				(collection) => !collection.startsWith('directus_'),
			);
			
			// 构建业务数据的 GraphQL Schema
			return generateGraphQLSchema(collections, this.schema, {
				scope: this.scope,
				excludeFields: [],
			});
		}
	}
	
	// scope 也影响解析器中的集合名称处理
	// 如: system 模式下，集合名会自动添加 directus_ 前缀
}
```

#### 对比 REST API 的系统数据路由

REST API 对系统数据使用独立的控制器和路由：

```typescript
// app.ts 中的 REST 路由挂载
app.use('/auth', authRouter);           // 认证
app.use('/activity', activityRouter);   // 活动日志
app.use('/access', accessRouter);       // 访问令牌
app.use('/assets', assetsRouter);       // 资源文件
app.use('/collections', collectionsRouter);  // 集合管理
app.use('/fields', fieldsRouter);       // 字段管理
app.use('/files', filesRouter);         // 文件
app.use('/flows', flowsRouter);         // 工作流
app.use('/folders', foldersRouter);     // 文件夹
app.use('/items', itemsRouter);         // 业务数据 (通用)
app.use('/notifications', notificationsRouter);  // 通知
app.use('/permissions', permissionsRouter);        // 权限
app.use('/policy', policyRouter);       // 策略
app.use('/presets', presetsRouter);     // 预设
app.use('/relations', relationsRouter);  // 关系
app.use('/revisions', revisionsRouter);  // 修订
app.use('/roles', rolesRouter);          // 角色
app.use('/settings', settingsRouter);    // 设置
app.use('/shares', sharesRouter);        // 分享
app.use('/translations', translationsRouter);  // 翻译
app.use('/users', usersRouter);          // 用户
app.use('/utils', utilsRouter);          // 工具
app.use('/webhooks', webhooksRouter);    // Webhook
```

**关键发现：**

| 维度 | REST API | GraphQL API |
|------|----------|-------------|
| **业务数据路由** | `/items/:collection` | `/graphql` |
| **系统数据路由** | 多个独立路由 (`/users`, `/roles`, `/permissions` 等) | `/graphql/system` |
| **路由设计理念** | 资源导向，每个系统资源独立路由 | 统一入口，通过 `scope` 参数区分 |
| **Schema 构建** | 无统一 Schema | 根据 `scope` 动态构建 |

### 1.2 REST 响应结构差异修正

#### 之前的分析（不准确）

> "REST: 固定格式 `{ data: ..., meta: ... }`"

#### 实际情况

REST API 的响应结构因操作类型而异，**`meta` 字段是可选的**，且只在特定条件下出现：

```typescript
// controllers/items.ts 中的响应设置

// 情况 1: 列表查询 (GET /:collection)
const readHandler = asyncHandler(async (req, res, next) => {
	// ...
	const result = await service.readByQuery(req.sanitizedQuery);
	
	// 关键: meta 只在 query.meta 存在时才返回
	const meta = await metaService.getMetaForQuery(req.collection, req.sanitizedQuery);
	
	res.locals['payload'] = {
		meta: meta,      // 可能是 undefined!
		data: result,
	};
});

// MetaService 的实现
// services/meta.ts:22-39
async getMetaForQuery(collection: string, query: any): Promise<Record<string, any> | undefined> {
	// 关键: 如果没有 meta 参数，直接返回 undefined
	if (!query || !query.meta) return;
	
	// 只有当 query.meta 存在时才计算 meta
	const results = await Promise.all(
		query.meta.map((metaVal: string) => {
			if (metaVal === 'total_count') return this.totalCount(collection);
			if (metaVal === 'filter_count') return this.filterCount(collection, query);
			return undefined;
		}),
	);
	
	return results.reduce((metaObject: Record<string, any>, value, index) => {
		return {
			...metaObject,
			[query.meta![index]]: value,
		};
	}, {});
}
```

#### 不同操作类型的响应结构

| 操作类型 | HTTP 方法 | 路由 | 响应结构 | 说明 |
|----------|-----------|------|----------|------|
| **列表查询** | GET | `/:collection` | `{ meta?, data: Item[] }` | `meta` 可选，需 `?meta=total_count` |
| **单个查询** | GET | `/:collection/:pk` | `{ data: Item \| null }` | **无 meta** |
| **创建** | POST | `/:collection` | `{ data: Item \| null }` | **无 meta** |
| **批量创建** | POST | `/:collection` (body 为数组) | `{ data: Item[] \| null }` | **无 meta** |
| **更新单个** | PATCH | `/:collection/:pk` | `{ data: Item \| null }` | **无 meta** |
| **更新多个** | PATCH | `/:collection` | `{ data: Item[] \| null }` | **无 meta** |
| **删除单个** | DELETE | `/:collection/:pk` | 204 No Content | **无响应体** |
| **删除多个** | DELETE | `/:collection` | 204 No Content | **无响应体** |

#### 代码验证

```typescript
// 单个查询 (GET /:collection/:pk)
// controllers/items.ts:103-115
router.get('/:collection/:pk', collectionExists, asyncHandler(async (req, res, next) => {
	const service = new ItemsService(req.collection, {
		accountability: req.accountability,
		schema: req.schema,
	});
	
	const result = await service.readOne(req.params['pk']!, req.sanitizedQuery);
	
	// 关键: 只有 data，没有 meta
	res.locals['payload'] = {
		data: result || null,
	};
	
	return next();
}), respond);

// 创建操作 (POST /:collection)
// controllers/items.ts:39-48
if (Array.isArray(req.body)) {
	const result = await service.readMany(savedKeys, req.sanitizedQuery);
	res.locals['payload'] = { data: result || null };  // 无 meta
} else {
	const result = await service.readOne(savedKeys[0]!, req.sanitizedQuery);
	res.locals['payload'] = { data: result || null };  // 无 meta
}

// 更新操作 (PATCH /:collection/:pk)
// controllers/items.ts:178-194
const result = await service.readOne(updatedPrimaryKey, req.sanitizedQuery);
res.locals['payload'] = { data: result || null };  // 无 meta
```

#### meta 字段的实际使用

客户端需要显式请求 `meta` 字段：

```
# 请求示例

# 1. 无 meta 的列表查询
GET /articles?fields=id,title&limit=10

# 响应:
{
  "data": [
    { "id": 1, "title": "Article 1" },
    { "id": 2, "title": "Article 2" }
  ]
}
# 注意: 没有 meta 字段!

# 2. 有 meta 的列表查询
GET /articles?fields=id,title&limit=10&meta=total_count&meta=filter_count

# 响应:
{
  "meta": {
    "total_count": 100,      // 集合中总记录数
    "filter_count": 25        // 符合过滤条件的记录数
  },
  "data": [
    { "id": 1, "title": "Article 1" },
    { "id": 2, "title": "Article 2" }
  ]
}
```

#### 与 GraphQL 响应结构的对比

| 维度 | REST API | GraphQL API |
|------|----------|-------------|
| **响应包装** | 始终 `{ data: ... }` (meta 可选) | `{ data: ..., errors?: [] }` |
| **meta 信息** | 可选，需显式请求 `?meta=...` | 无内置 meta，需通过查询获取 |
| **错误处理** | HTTP 状态码 + `errors` 数组 | HTTP 200 + `errors` 数组 (部分成功) |
| **空响应** | 204 No Content (删除操作) | `{ data: null }` 或 `{ errors: [...] }` |

#### respond 中间件的实际行为

```typescript
// middleware/respond.ts:120-126
if (Buffer.isBuffer(res.locals['payload'])) {
	return res.end(res.locals['payload']);
} else if (res.locals['payload']) {
	return res.json(res.locals['payload']);
} else {
	return res.status(204).end();
}
```

**关键发现：**

1. `respond` 中间件不添加任何额外包装，直接返回 `res.locals['payload']`
2. 如果 `payload` 是 `undefined`，返回 204 No Content
3. 控制器完全决定响应结构，`respond` 只是负责发送

---

## 2. 服务层到数据库适配层的边界约束和关键分岔条件

### 2.1 整体分岔点架构

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         服务层入口 (ItemsService)                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  第一级分岔: 操作类型                                                      │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐   │   │
│  │  │   读取操作   │  │   创建操作   │  │   更新操作   │  │   删除操作   │   │   │
│  │  │readByQuery  │  │ createOne   │  │ updateMany  │  │ deleteMany  │   │   │
│  │  │  readOne    │  │ createMany  │  │  updateOne  │  │ deleteOne   │   │   │
│  │  │  readMany   │  │             │  │  upsertOne  │  │             │   │   │
│  │  │readSingleton│  │             │  │ upsertMany  │  │             │   │   │
│  │  │             │  │             │  │upsertSinglet│  │             │   │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘   │   │
│  └─────────┼────────────────┼────────────────┼────────────────┼───────────┘   │
└────────────┼────────────────┼────────────────┼────────────────┼────────────────┘
             │                │                │                │
             ▼                ▼                ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         第二级分岔: 集合类型                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  分岔条件:                                                                 │   │
│  │  1. isSystemCollection(collection) → 系统集合 vs 普通集合                │   │
│  │  2. schema.collections[collection].singleton → 单例集合 vs 普通集合      │   │
│  │  3. schema.collections[collection].accountability → 问责制追踪设置        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
             │
             ▼ (读取操作继续下钻)
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         第三级分岔: 查询特征                                      │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  分岔条件 (读取操作特有):                                                 │   │
│  │  1. query.aggregate 或 query.group → 聚合/分组查询 vs 普通查询           │   │
│  │  2. hasMultiRelationalSort 或 hasMultiRelationalFilter → 是否需要内部查询│   │
│  │  3. hasCaseWhen → 是否有权限条件 (影响 DISTINCT vs GROUP BY)            │   │
│  │  4. accountability.admin → 管理员用户 vs 非管理员用户                     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         第四级分岔: 数据库类型                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  分岔条件:                                                                 │   │
│  │  getDatabaseClient(knex) → PostgreSQL / MySQL / SQLite / MSSQL / Oracle  │   │
│  │                                                                           │   │
│  │  影响:                                                                     │   │
│  │  - SQL 语法差异 (如 LIMIT vs TOP vs ROWNUM)                              │   │
│  │  -  returning 子句支持                                                    │   │
│  │  - DISTINCT 数据类型支持                                                  │   │
│  │  - GROUP BY 列位置支持                                                    │   │
│  │  - 自增序列重置方式                                                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 第一级分岔：操作类型

#### 2.2.1 读取操作 vs 写入操作的核心差异

| 维度 | 读取操作 | 写入操作 |
|------|----------|----------|
| **事务包装** | 无 | 有 (transaction) |
| **权限校验函数** | `processAst` | `validateAccess` + `processPayload` |
| **数据处理层** | `PayloadService.processValues('read', ...)` | `PayloadService.processM2O/O2M/A2O` + `processValues('create'|'update', ...)` |
| **问责制追踪** | 无 | 有 (Activity + Revisions) |
| **缓存处理** | 可能命中缓存 | 一定清理缓存 |
| **事件钩子** | `items.query` + `items.read` | `items.create/update/delete` (filter + action) |

#### 2.2.2 读取操作内部的分岔

```typescript
// ItemsService 中的读取操作分岔

// 1. readOne / readMany / readByQuery / readSingleton
async readOne(key: PrimaryKey, query: Query, opts?: QueryOptions): Promise<Partial<Item>> {
	// readOne 本质是 readByQuery 的特例
	const readQuery = cloneDeep(query);
	readQuery.filter = {
		_and: [readQuery.filter, { [primaryKeyField]: { _eq: key } }],
	};
	const result = await this.readByQuery(readQuery, opts);
	return result[0] ?? null;
}

async readMany(keys: PrimaryKey[], query: Query, opts?: QueryOptions): Promise<Partial<Item>[]> {
	// readMany 本质是 readByQuery 的特例
	const readQuery = cloneDeep(query);
	readQuery.filter = {
		_and: [readQuery.filter, { [primaryKeyField]: { _in: keys } }],
	};
	return this.readByQuery(readQuery, opts);
}

async readSingleton(query: Query, opts?: QueryOptions): Promise<Partial<Item>> {
	// 单例集合的特殊处理
	query = clone(query);
	query.limit = 1;  // 单例集合只有一条记录
	
	const result = await this.readByQuery(query, opts);
	
	// 如果没有记录，可能需要返回默认值
	if (!result || result.length === 0) {
		// 检查是否有预设的默认值
		const defaultRecord = this.getDefaultSingletonRecord();
		if (defaultRecord) {
			return defaultRecord;
		}
		throw new ForbiddenError();
	}
	
	return result[0];
}
```

#### 2.2.3 写入操作内部的分岔

```typescript
// ItemsService 中的写入操作分岔

// 创建操作
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
	// 单条创建: 有完整的关系处理流程
	// 包括: M2O → A2O → 主表插入 → O2M
}

async createMany(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 批量创建: 在事务中循环调用 createOne
	return transaction(this.knex, async (trx) => {
		const service = this.fork({ knex: trx });
		const keys: PrimaryKey[] = [];
		
		for (const item of data) {
			const key = await service.createOne(item, { ...opts, bypassLimits: true });
			keys.push(key);
		}
		
		return keys;
	});
}

// 更新操作
async updateOne(key: PrimaryKey, data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
	// 单条更新: 委托给 updateMany
	return this.updateMany([key], data, opts);
}

async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 多条更新: 所有记录使用相同的 data
}

async updateBatch(data: Partial<Item>[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 批量更新: 每条记录可以有不同的 data
	// 需要根据主键分别更新
	return transaction(this.knex, async (trx) => {
		const service = this.fork({ knex: trx });
		const keys: PrimaryKey[] = [];
		
		for (const item of data) {
			const key = item[primaryKeyField]!;
			await service.updateOne(key, omit(item, primaryKeyField), { ...opts, bypassLimits: true });
			keys.push(key);
		}
		
		return keys;
	});
}

async updateByQuery(query: Query, data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 按条件更新: 先查询主键，再更新
	const keys = await this.getKeysByQuery(query);
	return this.updateMany(keys, data, opts);
}

// Upsert 操作
async upsertOne(payload: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
	// 检查主键是否存在
	const key = payload[primaryKeyField];
	
	if (key !== undefined && key !== null) {
		// 检查是否存在
		const existing = await this.readOne(key, { fields: [primaryKeyField] }).catch(() => null);
		
		if (existing) {
			// 存在则更新
			return this.updateOne(key, omit(payload, primaryKeyField), opts);
		}
	}
	
	// 不存在则创建
	return this.createOne(payload, opts);
}

async upsertSingleton(data: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey> {
	// 单例集合的 upsert: 特殊处理
	const primaryKeyField = this.schema.collections[this.collection]!.primary;
	
	try {
		// 尝试读取
		const existing = await this.readSingleton({ fields: [primaryKeyField] });
		
		// 存在则更新
		if (existing && existing[primaryKeyField]) {
			return this.updateOne(existing[primaryKeyField] as PrimaryKey, data, opts);
		}
	} catch {
		// 不存在则创建
	}
	
	return this.createOne(data, opts);
}

// 删除操作
async deleteOne(key: PrimaryKey, opts: MutationOptions = {}): Promise<PrimaryKey> {
	// 单条删除: 委托给 deleteMany
	return this.deleteMany([key], opts);
}

async deleteMany(keys: PrimaryKey[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 多条删除: 统一处理
}

async deleteByQuery(query: Query, opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 按条件删除: 先查询主键，再删除
	const keys = await this.getKeysByQuery(query);
	return this.deleteMany(keys, opts);
}
```

### 2.3 第二级分岔：集合类型

#### 2.3.1 系统集合 vs 普通集合

```typescript
// ItemsService 构造函数中的分岔
constructor(collection: Collection, options: AbstractServiceOptions) {
	this.collection = collection;
	this.knex = options.knex || getDatabase();
	this.accountability = options.accountability || null;
	
	// 关键分岔: 系统集合 vs 普通集合
	this.eventScope = isSystemCollection(this.collection) 
		? this.collection.substring(9)  // 去掉 directus_ 前缀
		: 'items';                       // 普通集合使用 'items'
	
	this.schema = options.schema;
	this.cache = getCache().cache;
	this.nested = options.nested ?? [];
}

// eventScope 的影响
// 事件钩子名称会根据 eventScope 变化

// 系统集合 (如 directus_users)
// filter 钩子: ['users.create', 'directus_users.items.create']
// action 钩子: ['users.create', 'directus_users.items.create']

// 普通集合 (如 articles)
// filter 钩子: ['items.create', 'articles.items.create']
// action 钩子: ['items.create', 'articles.items.create']
```

#### 2.3.2 单例集合 vs 普通集合

```typescript
// 单例集合的特殊处理

// 1. 读取时: readSingleton
async readSingleton(query: Query, opts?: QueryOptions): Promise<Partial<Item>> {
	query = clone(query);
	query.limit = 1;  // 强制 limit=1
	
	const result = await this.readByQuery(query, opts);
	
	// 单例集合可能没有记录，需要特殊处理
	if (!result || result.length === 0) {
		// 检查是否有权限创建
		if (this.accountability) {
			await validateAccess(
				{
					accountability: this.accountability,
					action: 'create',
					collection: this.collection,
				},
				{ schema: this.schema, knex: this.knex },
			);
		}
		
		// 返回空对象或默认值
		return {};
	}
	
	return result[0];
}

// 2. 更新时: upsertSingleton
async upsertSingleton(data: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey> {
	const primaryKeyField = this.schema.collections[this.collection]!.primary;
	
	try {
		// 尝试读取现有记录
		const existing = await this.readSingleton({ fields: [primaryKeyField] });
		
		if (existing && existing[primaryKeyField]) {
			// 存在则更新
			return this.updateOne(existing[primaryKeyField] as PrimaryKey, data, opts);
		}
	} catch {
		// 不存在则继续创建
	}
	
	// 创建新记录
	return this.createOne(data, opts);
}

// 3. GraphQL 解析器中的单例检测
// graphql/resolvers/mutation.ts
const singleton =
	collection.endsWith('_batch') === false &&
	collection.endsWith('_items') === false &&
	collection.endsWith('_item') === false &&
	collection in gql.schema.collections;

if (singleton && action === 'update') {
	// 单例集合使用 upsertSingleton
	return await gql.upsertSingleton(collection, args['data'], query);
}
```

#### 2.3.3 问责制设置的影响

```typescript
// 写入操作中的问责制分岔

// createOne/updateMany/deleteMany 中都有这段逻辑
if (
	opts.skipTracking !== true &&
	this.accountability &&
	this.schema.collections[this.collection]!.accountability !== null
) {
	// 创建活动记录
	const activityService = new ActivityService({
		knex: trx,
		schema: this.schema,
	});
	
	const activity = await activityService.createOne({
		action: Action.CREATE, // 或 Action.UPDATE / Action.DELETE
		user: this.accountability!.user,
		collection: this.collection,
		ip: this.accountability!.ip,
		user_agent: this.accountability!.userAgent,
		origin: this.accountability!.origin,
		item: primaryKey,
	});
	
	// 额外的修订记录 (只有当 accountability === 'all' 时)
	if (this.schema.collections[this.collection]!.accountability === 'all') {
		const revisionsService = new RevisionsService({
			knex: trx,
			schema: this.schema,
		});
		
		await revisionsService.createOne({
			activity: activity,
			collection: this.collection,
			item: primaryKey,
			data: revisionPayload,
			delta: revisionPayload,
		});
	}
}

// accountability 的三个可能值:
// null - 不追踪
// 'activity' - 只追踪活动记录
// 'all' - 追踪活动记录 + 修订记录
```

### 2.4 第三级分岔：查询特征（读取操作特有）

#### 2.4.1 聚合/分组查询 vs 普通查询

```typescript
// getDBQuery 中的第一级分岔
// get-db-query.ts:47-89

// 分支条件
if (queryCopy.aggregate || queryCopy.group) {
	// 聚合/分组查询 - 简化流程
	const flatQuery = knex.from(table);
	
	// ... 简化的字段处理
	// 没有多关系排序/过滤的复杂处理
	// 没有内部查询的需求
	
	const dbQuery = applyQuery(knex, table, flatQuery, queryCopy, schema, cases, permissions, {
		aliasMap,
		groupWhenCases,
		groupColumnPositions,
	}).query;
	
	flatQuery.select(fieldNodes.map((node) => preProcess(node)));
	
	return dbQuery;
} else {
	// 普通查询 - 完整流程
	// 包括: 多关系排序/过滤检测、内部查询处理、权限条件处理等
}
```

#### 2.4.2 内部查询分岔

```typescript
// getDBQuery 中的第二级分岔
// get-db-query.ts:91-142

// 检测是否需要内部查询
const { hasMultiRelationalFilter } = applyQuery(knex, table, dbQuery, queryCopy, schema, cases, permissions, {
	aliasMap,
	isInnerQuery: true,
	hasMultiRelationalSort,
});

// 关键分岔条件
const needsInnerQuery = hasMultiRelationalSort || hasMultiRelationalFilter;

if (needsInnerQuery) {
	// 分支 A: 需要内部查询
	// 原因: 多关系排序或多关系过滤会导致重复行
	// 策略: 先获取唯一主键，再关联查询完整数据
	
	// 内部查询只选择主键
	dbQuery.select(`${table}.${primaryKey}`);
	
	// DISTINCT 或 GROUP BY 的选择 (下一个分岔点)
	if (!hasCaseWhen) {
		dbQuery.distinct();  // 无权限条件时用 DISTINCT
	}
	// 有权限条件时用 GROUP BY (见 2.4.3)
	
	// ... 后续需要外部查询包装
} else {
	// 分支 B: 不需要内部查询
	// 直接查询所有字段
	dbQuery.select(fieldNodes.map((node) => preProcess(node)));
	
	// 添加 O2M 权限标志 (如果有)
	dbQuery.select(
		o2mNodes
			.filter((node) => node.whenCase && node.whenCase.length > 0)
			.map((node) => applyCaseWhen(...)),
	);
}
```

#### 2.4.3 权限条件分岔 (hasCaseWhen)

```typescript
// getDBQuery 中的第三级分岔
// 这个分岔影响两个地方:
// 1. 内部查询中的去重方式 (DISTINCT vs GROUP BY)
// 2. 外部查询中的字段选择方式

// 分岔条件的定义
const hasCaseWhen =
	o2mNodes.some((node) => node.whenCase && node.whenCase.length > 0) ||
	fieldNodes.some((node) => node.whenCase && node.whenCase.length > 0);

// 影响 1: 内部查询的去重方式
if (needsInnerQuery) {
	dbQuery.select(`${table}.${primaryKey}`);
	
	// 关键分岔
	if (!hasCaseWhen) {
		// 无权限条件: 使用 DISTINCT
		dbQuery.distinct();
	} else {
		// 有权限条件: 不能用 DISTINCT
		// 原因: 某些数据库不支持某些数据类型的 DISTINCT
		// 策略: 使用 GROUP BY + COUNT 技巧
	}
}

// 影响 2: 内部查询的字段选择 (有权限条件时)
if (hasCaseWhen) {
	// 使用特殊的内部查询列预处理器
	const innerPreprocess = getInnerQueryColumnPreProcessor(
		knex, schema, table, cases, permissions, aliasMap, innerCaseWhenAliasPrefix
	);
	
	// 选择:
	// 1. 带权限条件的字段 → COUNT(CASE WHEN condition THEN 1 END) as flag
	// 2. O2M 字段的权限标志 → 同样的 COUNT 技巧
	dbQuery.select(fieldNodes.map(innerPreprocess).filter((x) => x !== null));
	dbQuery.select(o2mNodes.map(innerPreprocess).filter((x) => x !== null));
	
	// 使用 GROUP BY 代替 DISTINCT
	const groupByFields = [knex.raw('??.??', [table, primaryKey])];
	
	// 某些数据库需要将排序列也加入 GROUP BY
	helpers.schema.addInnerSortFieldsToGroupBy(
		groupByFields,
		innerQuerySortRecords,
		(hasMultiRelationalSort || sortRecords?.some(({ column }) => column.includes('.'))) ?? false,
	);
	
	dbQuery.groupBy(groupByFields);
}

// 影响 3: 外部查询的字段选择
const wrapperQuery = knex
	.from(table)
	.innerJoin(knex.raw('??', dbQuery.as('inner')), `${table}.${primaryKey}`, `inner.${primaryKey}`);

if (!hasCaseWhen) {
	// 无权限条件: 直接选择预处理后的字段
	wrapperQuery.select(fieldNodes.map((node) => preProcess(node)));
} else {
	// 有权限条件: 根据内部查询的标志决定是否显示字段
	
	// 区分普通字段和带权限的字段
	const plainColumns = fieldNodes.filter((fieldNode) => !fieldNode.whenCase || fieldNode.whenCase.length === 0);
	const whenCaseColumns = fieldNodes.filter((fieldNode) => fieldNode.whenCase && fieldNode.whenCase.length > 0);
	
	// 普通字段: 直接选择
	wrapperQuery.select(plainColumns.map((node) => preProcess(node)));
	
	// 带权限的字段: 使用 CASE WHEN 检查标志
	wrapperQuery.select(
		whenCaseColumns.map((fieldNode) => {
			const alias = getNodeAlias(fieldNode);
			const innerAlias = `${innerCaseWhenAliasPrefix}_${alias}`;
			const column = preProcess({ ...fieldNode, whenCase: [] }, { noAlias: true });
			
			// 关键: 根据内部查询的标志决定是否返回值
			// CASE WHEN inner.flag > 0 THEN column ELSE NULL END as alias
			return knex.raw(`CASE WHEN ??.?? > 0 THEN ?? END as ??`, ['inner', innerAlias, column, alias]);
		}),
	);
	
	// O2M 字段标志传递
	wrapperQuery.select(
		o2mNodes
			.filter((node) => node.whenCase && node.whenCase.length > 0)
			.map((node) => {
				const alias = node.fieldKey;
				const innerAlias = `${innerCaseWhenAliasPrefix}_${alias}`;
				return knex.raw(`CASE WHEN ??.?? > 0 THEN 1 END as ??`, ['inner', innerAlias, alias]);
			}),
	);
}
```

#### 2.4.4 多关系排序的特殊处理

```typescript
// 多关系排序时的窗口函数技巧
if (hasMultiRelationalSort) {
	// 使用窗口函数为每个分区的行编号
	dbQuery.rowNumber(
		knex.ref('directus_row_number').toQuery(),
		knex.raw(`partition by ?? order by ${orderByString}`, [`${table}.${primaryKey}`, ...orderByFields]),
	);
	
	// 外部查询中只取每个分区的第一行
	wrapperQuery.where('inner.directus_row_number', '=', 1);
}

// 为什么需要这个?
// 场景: 按关联表的字段排序，如:
// GET /articles?sort=-comments.date_created
// 或 GraphQL: articles(sort: ["-comments.date_created"]) { ... }

// 问题: 多关系 JOIN 会导致重复行
// article 1 有 3 个 comments → JOIN 后产生 3 行
// 但我们只想返回 1 条 article 记录

// 解决方案:
// 1. 按 article.id 分区 (partition by)
// 2. 每个分区内按排序条件编号 (row number)
// 3. 只取 row_number = 1 的记录
// 这样确保每个 article 只返回一行，且是排序最靠前的关联记录
```

### 2.5 第四级分岔：数据库类型

#### 2.5.1 数据库类型检测

```typescript
// database/index.ts:253-275
export function getDatabaseClient(database?: Knex): DatabaseClient {
	database = database ?? getDatabase();
	
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

#### 2.5.2 数据库特定处理

```typescript
// 1. returning 子句支持
// createOne 中的处理
let returningOptions = undefined;

// MSSQL 特殊处理: 支持触发器修改
if (getDatabaseClient(trx) === 'mssql') {
	returningOptions = { includeTriggerModifications: true };
}

const result = await trx
	.insert(payloadWithoutAliases)
	.into(this.collection)
	.returning(primaryKeyField, returningOptions)
	.then((result) => result[0]);

// 2. 自增序列重置
// createOne 中的处理
if (autoIncrementSequenceNeedsToBeReset) {
	await getHelpers(trx).sequence.resetAutoIncrementSequence(
		this.collection,
		primaryKeyField,
	);
}

// helpers 中的数据库特定实现:
// database/helpers/postgres/sequence.ts
// database/helpers/mysql/sequence.ts
// database/helpers/sqlite/sequence.ts
// 等等

// 3. GROUP BY 列位置支持
// getDBQuery 中的处理
if (helpers.capabilities.supportsColumnPositionInGroupBy() && options.groupColumnPositions) {
	// 支持列位置的数据库: 使用位置
	columns = query.group.map((column, index) =>
		options.groupColumnPositions![index] !== undefined ? knex.raw(options.groupColumnPositions![index]) : column,
	);
} else {
	// 不支持的数据库: 使用完整的 CASE WHEN 表达式
	columns = rawColumns.map((column, index) =>
		applyCaseWhen({ columnCases: options.groupWhenCases![index]!, column, ... }, { knex, schema }),
	);
}

// 4. 参数去重支持
if (helpers.capabilities.supportsDeduplicationOfParameters() &&
	!helpers.capabilities.supportsColumnPositionInGroupBy()) {
	withPreprocessBindings(knex, dbQuery);
}

// 5. 内部分组排序字段处理
helpers.schema.addInnerSortFieldsToGroupBy(
	groupByFields,
	innerQuerySortRecords,
	(hasMultiRelationalSort || sortRecords?.some(({ column }) => column.includes('.'))) ?? false,
);

// 不同数据库的 schema helpers:
// database/helpers/postgres/schema.ts
// database/helpers/mysql/schema.ts
// database/helpers/sqlite/schema.ts
// database/helpers/mssql/schema.ts
// database/helpers/oracle/schema.ts
// database/helpers/cockroachdb/schema.ts
```

#### 2.5.3 数据库能力检测

```typescript
// helpers 中的 capabilities
interface DatabaseCapabilities {
	supportsDeduplicationOfParameters: () => boolean;
	supportsColumnPositionInGroupBy: () => boolean;
	// ... 其他能力
}

// PostgreSQL 实现示例
class PostgresHelpers {
	get capabilities() {
		return {
			supportsDeduplicationOfParameters: () => true,
			supportsColumnPositionInGroupBy: () => true,
		};
	}
}

// MySQL 实现示例
class MySQLHelpers {
	get capabilities() {
		return {
			supportsDeduplicationOfParameters: () => false,
			supportsColumnPositionInGroupBy: () => {
				// MySQL 8.0+ 支持，但需要检测版本
				return false; // 简化处理
			},
		};
	}
}
```

---

## 3. 读场景与写场景的并排调用链对照

### 3.1 调用链总览

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              客户端请求                                           │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐       │
│  │      REST API 请求              │  │      GraphQL API 请求           │       │
│  │  GET /articles?fields=id,title  │  │  query { articles { id title } } │       │
│  │  POST /articles (body)          │  │  mutation { create_articles_item  │       │
│  │  PATCH /articles/1 (body)       │  │    (data: { title: "New" }) {   │       │
│  │  DELETE /articles/1             │  │      id title                    │       │
│  │                                 │  │    } }                           │       │
│  └──────────────┬──────────────────┘  └──────────────┬──────────────────┘       │
└─────────────────┼─────────────────────────────────────┼──────────────────────────┘
                  │                                     │
                  ▼                                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              接口层 (差异处理)                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  REST:                                                                   │   │
│  │  1. 路由匹配 (/:collection, /:collection/:pk)                            │   │
│  │  2. 中间件处理 (collectionExists, validateBatch, sanitizeQuery)          │   │
│  │  3. 参数提取 (req.params, req.query, req.body)                           │   │
│  │  4. 直接实例化 ItemsService                                              │   │
│  │                                                                           │   │
│  │  GraphQL:                                                                │   │
│  │  1. 路由匹配 (/graphql 或 /graphql/system)                               │   │
│  │  2. parseGraphQL 中间件解析查询文档                                      │   │
│  │  3. GraphQL 引擎执行查询                                                 │   │
│  │  4. 解析器转换: GraphQL 查询 → Directus Query                            │   │
│  │  5. getService 获取服务实例                                              │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           服务层 (统一处理)                                        │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐       │
│  │         读取操作路径             │  │         写入操作路径             │       │
│  │  readByQuery / readOne          │  │  createOne / updateMany          │       │
│  │  readMany / readSingleton       │  │  deleteMany / upsertOne          │       │
│  │                                 │  │  upsertMany / upsertSingleton    │       │
│  │  ┌───────────────────────────┐  │  │  ┌───────────────────────────┐  │       │
│  │  │ 1. emitFilter (items.query)│  │  │  │ 1. emitFilter (items.create)│  │       │
│  │  │ 2. getAstFromQuery        │  │  │  │ 2. validateAccess           │  │       │
│  │  │ 3. processAst (权限)      │  │  │  │ 3. processPayload           │  │       │
│  │  │ 4. runAst                 │  │  │  │ 4. transaction (事务包装)    │  │       │
│  │  │ 5. emitFilter (items.read)│  │  │  │    - PayloadService 关系处理│  │       │
│  │  │ 6. emitAction (items.read)│  │  │  │    - Knex 执行              │  │       │
│  │  └───────────────────────────┘  │  │  │    - 问责制追踪              │  │       │
│  │                                 │  │  │ 5. cache.clear()             │  │       │
│  │  无事务                          │  │  │ 6. emitAction                │  │       │
│  │  可能命中缓存                    │  │  │  一定清理缓存                 │  │       │
│  │  无问责制追踪                    │  │  │  有问责制追踪                 │  │       │
│  └─────────────────────────────────┘  │  └─────────────────────────────────┘  │
└──────────────────────────────────────────┼──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          数据库层 (执行落点)                                      │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐       │
│  │         读取操作执行             │  │         写入操作执行             │       │
│  │  ┌───────────────────────────┐  │  │  ┌───────────────────────────┐  │       │
│  │  │ getDBQuery 分岔:         │  │  │  │ Knex 直接调用:           │  │       │
│  │  │ - 聚合查询?               │  │  │  │ - insert                   │  │       │
│  │  │ - 多关系排序/过滤?         │  │  │  │ - update                   │  │       │
│  │  │ - 有权限条件?             │  │  │  │ - delete                   │  │       │
│  │  │                           │  │  │  │                           │  │       │
│  │  │ 分岔结果:                 │  │  │  │ 主键获取策略:             │  │       │
│  │  │ - 普通查询: 直接执行      │  │  │  │ - returning (PG, MSSQL)   │  │       │
│  │  │ - 需内部查询: 2 阶段      │  │  │  │ - max(pk) (MySQL, SQLite) │  │       │
│  │  │ - 有权限: CASE WHEN      │  │  │  │                           │  │       │
│  │  └───────────────────────────┘  │  │  │ 数据库特定处理:           │  │       │
│  │                                 │  │  │  │ - MSSQL returningOptions│  │       │
│  │  PayloadService.processValues  │  │  │  │ - 自增序列重置           │  │       │
│  │  ('read', ...)                  │  │  │  │ - 外键约束处理           │  │       │
│  │                                 │  │  │  └───────────────────────────┘  │       │
│  │  嵌套关系处理:                  │  │  │                                 │       │
│  │  - 批量查询 (O2M)               │  │  │  事务边界:                  │  │       │
│  │  - 单次查询 (M2O/A2O)           │  │  │  - 自动提交/回滚           │  │       │
│  │  - 权限标志检查                 │  │  │  - 错误翻译                 │  │       │
│  └─────────────────────────────────┘  │  └─────────────────────────────────┘  │
└──────────────────────────────────────────┴──────────────────────────────────────┘
```

### 3.2 详细调用链对照表

#### 3.2.1 读取操作调用链

| 阶段 | 步骤 | REST API | GraphQL API | 关键代码位置 |
|------|------|-----------|-------------|--------------|
| **1. 入口** | 路由匹配 | `GET /:collection` 或 `GET /:collection/:pk` | `POST /graphql` 或 `POST /graphql/system` | `app.ts:323`, `graphql.ts:7-50` |
| | 中间件 | `collectionExists`, `sanitizeQuery` | `parseGraphQL` (解析查询文档) | `items.ts:23-27`, `middleware/graphql.ts` |
| | 参数提取 | `req.params.pk`, `req.sanitizedQuery` | 解析器中的 `args`, `info.fieldNodes` | `items.ts:81-89`, `graphql/schema/parse-query.ts` |
| **2. 服务层** | 服务实例化 | `new ItemsService(collection, { accountability, schema })` | `getService(collection, { knex, accountability, schema })` | `items.ts:59-64`, `graphql/resolvers/query.ts:59` |
| | 方法调用 | `service.readByQuery(query)` 或 `service.readOne(pk, query)` | `gql.read(collection, query, id?)` | `items.ts:499`, `graphql/index.ts:156` |
| **3. 读取流程** | Filter 钩子 1 | `emitter.emitFilter('items.query', query, ...)` | 相同 | `items.ts:507-527` |
| | AST 构造 | `getAstFromQuery({ collection, query, accountability })` | 相同 | `items.ts:540`, `get-ast-from-query.ts:23` |
| | 权限处理 | `processAst({ ast, action: 'read', accountability })` | 相同 | `items.ts:549`, `process-ast/process-ast.ts:19` |
| | 执行查询 | `runAst(ast, schema, accountability, { knex, stripNonRequested })` | 相同 | `items.ts:567`, `run-ast/run-ast.ts:20` |
| | 空结果检查 | `if (records === null) throw new ForbiddenError()` | 相同 | `items.ts:576` |
| | Filter 钩子 2 | `emitter.emitFilter('items.read', records, ...)` | 相同 | `items.ts:583-603` |
| | Action 钩子 | `emitter.emitAction('items.read', { payload, query, collection }, ...)` | 相同 | `items.ts:615-637` |
| **4. 响应** | 响应设置 | `res.locals['payload'] = { meta?, data: result }` | `res.locals['payload'] = { data?, errors? }` | `items.ts:86-89`, `graphql.ts:40-50` |
| | 响应发送 | `respond` 中间件 `res.json(payload)` | 相同 | `middleware/respond.ts:122-123` |

#### 3.2.2 写入操作调用链（以创建为例）

| 阶段 | 步骤 | REST API | GraphQL API | 关键代码位置 |
|------|------|-----------|-------------|--------------|
| **1. 入口** | 路由匹配 | `POST /:collection` | `POST /graphql` | `items.ts:13-48`, `graphql.ts:7-50` |
| | 中间件 | `collectionExists` | `parseGraphQL` | `items.ts:14`, `middleware/graphql.ts` |
| | 参数提取 | `req.body` (数组或对象) | 解析器中的 `args['data']` | `items.ts:38-48`, `graphql/resolvers/mutation.ts:48-70` |
| **2. 服务层** | 服务实例化 | `new ItemsService(collection, { accountability, schema })` | `getService(collection, { knex, accountability, schema })` | `items.ts:59-64`, `graphql/resolvers/mutation.ts:78` |
| | 方法调用 | `service.createOne(data)` 或 `service.createMany(data[])` | `service.createOne(args['data'])` 或 `createMany` | `items.ts:127`, `graphql/resolvers/mutation.ts:85-92` |
| **3. 创建流程** | 突变追踪 | `mutationTracker.trackMutations(count)` | 相同 | `items.ts:128-132` |
| | Filter 钩子 | `emitter.emitFilter('items.create', payload, ...)` | 相同 | `items.ts:154-170` |
| | 权限处理 | `processPayload({ accountability, action: 'create', collection, payload })` | 相同 | `items.ts:172-186` |
| | 事务开始 | `transaction(this.knex, async (trx) => { ... })` | 相同 | `items.ts:151` |
| | M2O 处理 | `payloadService.processM2O(payload, opts)` | 相同 | `items.ts:201` |
| | A2O 处理 | `payloadService.processA2O(payload, opts)` | 相同 | `items.ts:208` |
| | 类型转换 | `payloadService.processValues('create', payload)` | 相同 | `items.ts:215` |
| | 数据库插入 | `trx.insert(payload).into(collection).returning(primaryKey)` | 相同 | `items.ts:223-231` |
| | O2M 处理 | `payloadService.processO2M(payload, primaryKey, opts)` | 相同 | `items.ts:250` |
| | 完整性验证 | `validateUserCountIntegrity({ flags, knex: trx })` | 相同 | `items.ts:275-281` |
| | 问责制追踪 | `activityService.createOne({ action: Action.CREATE, ... })` | 相同 | `items.ts:288-340` |
| | 修订记录 | `revisionsService.createOne({ activity, collection, item, data, delta })` | 相同 (当 accountability === 'all') | `items.ts:306-338` |
| | 事务结束 | 自动提交或回滚 | 相同 | `items.ts:151` 的 transaction 包装 |
| | 缓存清理 | `cache.clear()` | 相同 | `items.ts:354-356` |
| | Action 钩子 | `emitter.emitAction('items.create', { payload, key, collection }, ...)` | 相同 | `items.ts:359-383` |
| **4. 响应** | 读取返回 | `service.readOne(savedKey, query)` 或 `readMany` | 相同 | `items.ts:41-46`, `graphql/resolvers/mutation.ts:87-90` |
| | 响应设置 | `res.locals['payload'] = { data: result \|\| null }` | 解析器返回结果 → `res.locals['payload']` | `items.ts:43-46`, `graphql.ts:42` |
| | 响应发送 | `respond` 中间件 | 相同 | `middleware/respond.ts:122-123` |

### 3.3 关键差异点详解

#### 3.3.1 服务实例化方式

```typescript
// REST API: 直接实例化
// controllers/items.ts
const service = new ItemsService(req.collection, {
	accountability: req.accountability,
	schema: req.schema,
	// knex 是可选的，默认使用 getDatabase()
});

// GraphQL API: 通过 getService 获取
// graphql/resolvers/query.ts
const service = getService(collection, {
	knex: gql.knex,           // 明确传入
	accountability: gql.accountability,
	schema: gql.schema,
});

// getService 的实现
// graphql/utils/get-service.ts
export function getService(collection: string, context: {
	knex: Knex;
	accountability: Accountability | null;
	schema: SchemaOverview;
}): ItemsService {
	return new ItemsService(collection, {
		knex: context.knex,
		accountability: context.accountability,
		schema: context.schema,
	});
}

// 关键差异:
// 1. REST: 不传入 knex，使用默认的 getDatabase()
// 2. GraphQL: 明确传入 gql.knex（来自 GraphQLService 的 this.knex）
// 
// 影响:
// - 在事务中运行时，GraphQL 需要确保所有服务使用同一个 knex 实例
// - REST 依赖服务层自己的 fork 机制来处理事务
```

#### 3.3.2 查询构造方式

```typescript
// REST API: 直接使用 req.sanitizedQuery
// controllers/items.ts
const result = await service.readByQuery(req.sanitizedQuery);

// req.sanitizedQuery 的来源:
// 1. 从 req.query 提取
// 2. 通过 sanitizeQuery 中间件清理和验证
// 结果已经是 Directus Query 对象

// GraphQL API: 需要转换
// graphql/schema/parse-query.ts
export async function getQuery(
	rawQuery: Query,
	schema: SchemaOverview,
	selections: readonly SelectionNode[],
	variableValues: GraphQLResolveInfo['variableValues'],
	accountability?: Accountability | null,
	collection?: string,
): Promise<Query> {
	// 1. 基础清理
	const query: Query = await sanitizeQuery(rawQuery, schema, accountability);
	
	// 2. 解析别名
	query.alias = parseAliases(selections);
	
	// 3. 解析字段选择（递归处理嵌套）
	query.fields = await parseFields(selections, undefined, collection);
	
	// 4. 处理函数替换
	if (query.filter) query.filter = replaceFuncs(query.filter);
	query.deep = replaceFuncs(query.deep as any) as any;
	
	// 5. 处理 M2A 关系
	if (collection) {
		if (query.filter) {
			query.filter = filterReplaceM2A(query.filter, collection, schema, { aliasMap: query.alias });
		}
		query.deep = filterReplaceM2ADeep(query.deep, collection, schema, { aliasMap: query.alias });
	}
	
	// 6. 验证
	validateQuery(query);
	
	return query;
}

// 关键差异:
// 1. REST: 查询已经是 Directus 格式，无需转换
// 2. GraphQL: 需要将 GraphQL AST 转换为 Directus Query
// 
// 转换内容包括:
// - 字段选择: GraphQL 选择集 → query.fields
// - 嵌套查询: 嵌套字段 → query.deep
// - 别名: GraphQL 别名 → query.alias
// - 函数: GraphQL 函数调用 → Directus 函数格式
// - M2A 关系: 内联片段 → 集合限定符
```

#### 3.3.3 响应构造方式

```typescript
// REST API: 控制器明确设置
// controllers/items.ts

// 列表查询
const meta = await metaService.getMetaForQuery(req.collection, req.sanitizedQuery);
res.locals['payload'] = {
	meta: meta,      // 可能是 undefined
	data: result,
};

// 单个查询/创建/更新
res.locals['payload'] = {
	data: result || null,  // 没有 meta
};

// GraphQL API: 解析器返回，GraphQL 引擎包装
// graphql/index.ts
async execute({ document, variables, operationName, contextValue }: GraphQLParams) {
	// ... 执行查询 ...
	
	const result: ExecutionResult = await execute({
		schema,
		document,
		contextValue,
		variableValues: variables,
		operationName,
	});
	
	// 构建响应
	const formattedResult: FormattedExecutionResult = {};
	
	if (result['data']) formattedResult.data = result['data'];
	
	if (result['errors']) {
		formattedResult.errors = result['errors'].map((error) => 
			processError(this.accountability, error)
		);
	}
	
	if (result['extensions']) formattedResult.extensions = result['extensions'];
	
	res.locals['payload'] = formattedResult;
}

// 关键差异:
// 1. REST: 控制器完全控制响应结构
//    - 有 data 字段
//    - 有可选的 meta 字段
//    - 没有 errors 字段（错误通过中间件处理）
// 
// 2. GraphQL: 遵循 GraphQL 规范
//    - 有 data 字段（可能是 null）
//    - 有可选的 errors 字段（支持部分成功）
//    - 有可选的 extensions 字段
//    - 没有 meta 字段（需要通过查询获取）
```

#### 3.3.4 错误处理方式

```typescript
// REST API: HTTP 状态码 + errors 数组
// 错误通过 Express 错误处理中间件处理

// 示例: 控制器中抛出错误
throw new ForbiddenError({
	reason: `You don't have permission to perform "${action}" for collection "${collection}".`,
});

// 错误处理中间件会:
// 1. 设置合适的 HTTP 状态码 (403)
// 2. 构造响应: { errors: [{ message, extensions: { code, ... } }] }
// 3. 不包含 data 字段

// GraphQL API: HTTP 200 + errors 数组
// 错误在 GraphQL 引擎层面处理

// 示例: 解析器中抛出错误
throw new ForbiddenError({ ... });

// GraphQL 引擎会:
// 1. 捕获错误并添加到 errors 数组
// 2. 保持 HTTP 状态码为 200
// 3. 构造响应: { data: null 或 部分数据, errors: [...] }
// 4. 支持部分成功（部分 data + 部分 errors）

// 关键差异:
// | 维度 | REST API | GraphQL API |
// |------|----------|-------------|
// | HTTP 状态码 | 语义化 (400, 403, 404, 500) | 通常 200 |
// | 部分成功 | 不支持 | 支持 |
// | data 字段 | 成功时有，失败时无 | 始终有（可能是 null） |
// | errors 字段 | 失败时有，成功时无 | 有错误时有，无错误时无 |
// | 缓存影响 | 错误状态码不缓存 | HTTP 200 可能被缓存（需要额外处理） |
```

### 3.4 统一与差异总结

#### 3.4.1 统一的部分（核心流程）

```
┌─────────────────────────────────────────────────────────────────┐
│                    统一的核心流程 (REST + GraphQL)                │
│                                                                   │
│  服务层:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │  • 权限校验函数完全相同                                        │ │
│  │    - validateAccess (写入操作)                                │ │
│  │    - processAst (读取操作)                                    │ │
│  │    - processPayload (创建/更新)                               │ │
│  │                                                                 │ │
│  │  • 事件钩子完全相同                                            │ │
│  │    - filter 钩子: 'items.query', 'items.create/update/delete'│ │
│  │    - action 钩子: 'items.read/create/update/delete'           │ │
│  │                                                                 │ │
│  │  • 数据库交互完全相同                                          │ │
│  │    - 读取: getAstFromQuery → processAst → runAst             │ │
│  │    - 写入: transaction → PayloadService → Knex               │ │
│  │                                                                 │ │
│  │  • 缓存策略完全相同                                            │ │
│  │    - 读取: 可能命中缓存                                        │ │
│  │    - 写入: 一定清理缓存                                        │ │
│  │                                                                 │ │
│  │  • 问责制追踪完全相同                                          │ │
│  │    - Activity 记录                                             │ │
│  │    - Revisions 记录 (当 accountability === 'all')             │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.4.2 差异的部分（协议适配）

```
┌─────────────────────────────────────────────────────────────────┐
│                    差异的部分 (协议适配层)                        │
│                                                                   │
│  接口层:                                                          │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                                                                 │ │
│  │  路由设计:                                                     │ │
│  │  • REST: 资源导向，每个资源独立路由                          │ │
│  │  • GraphQL: 单一入口，通过查询语言区分                        │ │
│  │                                                                 │ │
│  │  参数解析:                                                     │ │
│  │  • REST: 直接从 req.params/query/body 提取                    │ │
│  │  • GraphQL: 需要解析 GraphQL AST                              │ │
│  │                                                                 │ │
│  │  查询构造:                                                     │ │
│  │  • REST: 直接使用 Directus Query 对象                         │ │
│  │  • GraphQL: 需要将 GraphQL 查询转换为 Directus Query         │ │
│  │                                                                 │ │
│  │  响应构造:                                                     │ │
│  │  • REST: 控制器明确设置 { meta?, data }                       │ │
│  │  • GraphQL: 引擎包装 { data?, errors?, extensions? }        │ │
│  │                                                                 │ │
│  │  错误处理:                                                     │ │
│  │  • REST: HTTP 状态码 + errors 数组                           │ │
│  │  • GraphQL: HTTP 200 + errors 数组 (支持部分成功)            │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 设计取舍的深层分析

### 4.1 为什么选择统一服务层？

#### 优势分析

```
┌─────────────────────────────────────────────────────────────────┐
│                    统一服务层的设计优势                           │
│                                                                   │
│  1. 避免重复代码                                                  │
│     ├── 权限校验逻辑只需实现一次                                  │
│     ├── 业务逻辑只需实现一次                                      │
│     ├── 数据库交互只需实现一次                                    │
│     └── 事件钩子只需实现一次                                      │
│                                                                   │
│  2. 确保行为一致性                                                │
│     ├── REST 和 GraphQL 返回相同数据结构                         │
│     ├── 权限校验规则一致                                          │
│     ├── 验证逻辑一致                                              │
│     └── 缓存策略一致                                              │
│                                                                   │
│  3. 便于维护和扩展                                                │
│     ├── 新增功能只需修改服务层                                    │
│     ├── 新增接口协议只需添加接口层                                │
│     ├── 测试只需针对服务层                                        │
│     └── Bug 修复只需一处                                          │
│                                                                   │
│  4. 安全性                                                        │
│     ├── 无法绕过服务层直接操作数据库                              │
│     ├── 权限校验在服务层强制执行                                  │
│     └── 所有数据操作都经过同一套安全检查                          │
└─────────────────────────────────────────────────────────────────┘
```

#### 代价分析

```
┌─────────────────────────────────────────────────────────────────┐
│                    统一服务层的设计代价                           │
│                                                                   │
│  1. 接口层需要额外的转换层                                        │
│     ├── GraphQL 需要 AST 解析和转换                              │
│     └── 转换过程可能引入 Bug                                      │
│                                                                   │
│  2. 服务层需要考虑所有接口的需求                                  │
│     ├── 服务层 API 设计需要足够通用                              │
│     └── 可能导致接口不够直观                                      │
│                                                                   │
│  3. 性能开销                                                      │
│     ├── 统一的权限校验可能对某些场景过度                          │
│     └── 事件钩子系统有额外开销                                    │
│                                                                   │
│  4. 学习曲线                                                      │
│     ├── 需要理解完整的架构层次                                    │
│     └── 新开发者需要时间理解各层职责                              │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 为什么选择 AST 作为中间表示？

#### 优势分析

```
┌─────────────────────────────────────────────────────────────────┐
│                    AST 作为中间表示的设计优势                     │
│                                                                   │
│  1. 解耦查询格式与数据库执行                                      │
│     ├── REST 查询 → AST                                          │
│     ├── GraphQL 查询 → AST                                       │
│     └── AST → 数据库查询                                         │
│     新增查询格式只需新增一个转换器，无需修改数据库执行层          │
│                                                                   │
│  2. 便于权限注入                                                  │
│     ├── 权限条件直接注入 AST 的 cases 数组                       │
│     ├── 不依赖原始查询格式                                        │
│     └── 灵活的字段级权限控制                                      │
│                                                                   │
│  3. 支持复杂的查询优化                                            │
│     ├── 多关系排序/过滤的内部查询优化                            │
│     ├── case/when 权限条件的优化                                 │
│     └── 批量查询优化                                              │
│                                                                   │
│  4. 统一的查询表示                                                │
│     ├── 无论哪种接口，最终都转换为相同的 AST                     │
│     └── 便于调试和日志                                            │
└─────────────────────────────────────────────────────────────────┘
```

#### 代价分析

```
┌─────────────────────────────────────────────────────────────────┐
│                    AST 作为中间表示的设计代价                     │
│                                                                   │
│  1. 额外的转换层                                                  │
│     ├── 查询 → AST 的转换开销                                    │
│     ├── AST → SQL 的转换开销                                     │
│     └── 增加了理解复杂度                                          │
│                                                                   │
│  2. 调试困难                                                      │
│     ├── 需要理解 AST 结构                                         │
│     ├── 需要追踪整个转换链路                                      │
│     └── Bug 可能隐藏在转换过程中                                  │
│                                                                   │
│  3. 性能开销                                                      │
│     ├── AST 的构建和遍历有额外开销                                │
│     └── 深层嵌套查询可能影响性能                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 为什么 REST 和 GraphQL 采用不同的路由策略？

#### REST 的多路由策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    REST 多路由策略的设计考量                      │
│                                                                   │
│  1. 符合 RESTful 设计原则                                         │
│     ├── 每个资源有独立的 URL                                      │
│     ├── HTTP 方法表达操作意图                                     │
│     └── 便于理解和调试                                            │
│                                                                   │
│  2. 天然支持 HTTP 缓存                                            │
│     ├── GET 请求可以被浏览器缓存                                  │
│     ├── 不同资源的缓存独立                                        │
│     └── ETag、Last-Modified 等机制天然适用                       │
│                                                                   │
│  3. 便于权限控制                                                  │
│     ├── 可以基于 URL 路径设置权限                                 │
│     └── 中间件可以在路由层面拦截                                  │
│                                                                   │
│  4. 与第三方系统集成                                              │
│     ├── 大多数系统理解 REST 风格                                  │
│     └── 无需额外的查询语言学习                                    │
└─────────────────────────────────────────────────────────────────┘
```

#### GraphQL 的单一路由策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    GraphQL 单一路由策略的设计考量                 │
│                                                                   │
│  1. 统一入口，简化管理                                            │
│     ├── 所有查询通过一个端点                                      │
│     ├── 中间件只需挂载一次                                        │
│     └── 路由配置简单                                              │
│                                                                   │
│  2. 灵活的查询能力                                                │
│     ├── 一个请求获取多个资源                                      │
│     ├── 精确控制返回字段                                          │
│     └── 嵌套关系的自然表达                                        │
│                                                                   │
│  3. 类型系统支持                                                  │
│     ├── 强类型的 Schema 定义                                      │
│     ├── 自动文档生成                                              │
│     └── 类型安全的查询                                            │
│                                                                   │
│  4. 前端友好                                                      │
│     ├── 前端可以精确获取所需数据                                  │
│     ├── 避免过度获取或获取不足                                    │
│     └── 减少请求次数                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 为什么 meta 字段在 REST 中是可选的？

#### 设计考量

```
┌─────────────────────────────────────────────────────────────────┐
│                    meta 字段可选的设计考量                        │
│                                                                   │
│  1. 性能优化                                                      │
│     ├── total_count 需要额外的 COUNT 查询                        │
│     ├── filter_count 也需要额外查询                              │
│     └── 不请求 meta 可以避免这些开销                              │
│                                                                   │
│  2. 灵活性                                                        │
│     ├── 客户端可以按需请求                                        │
│     ├── 支持多种 meta 类型 (total_count, filter_count)          │
│     └── 可以组合请求多个 meta                                    │
│                                                                   │
│  3. 与 GraphQL 保持语义一致                                      │
│     ├── GraphQL 中没有内置的 meta 概念                          │
│     └── 客户端需要通过额外查询获取计数                            │
│                                                                   │
│  4. 向后兼容                                                      │
│     ├── 早期版本可能没有 meta                                    │
│     └── 可选字段避免破坏现有客户端                                │
└─────────────────────────────────────────────────────────────────┘
```

#### 请求示例

```
# 不请求 meta (性能最优)
GET /articles?fields=id,title&limit=10

# 响应:
{
  "data": [
    { "id": 1, "title": "Article 1" },
    { "id": 2, "title": "Article 2" }
  ]
}

# 请求单个 meta
GET /articles?fields=id,title&limit=10&meta=total_count

# 响应:
{
  "meta": {
    "total_count": 100
  },
  "data": [...]
}

# 请求多个 meta
GET /articles?fields=id,title&limit=10&meta=total_count&meta=filter_count

# 响应:
{
  "meta": {
    "total_count": 100,
    "filter_count": 25
  },
  "data": [...]
}
```

---

## 5. 修正总结

### 5.1 已修正的事实错误

| 错误描述 | 修正内容 |
|----------|----------|
| GraphQL "单一入口 `/graphql`" | 实际有两个入口：`/graphql` (业务数据) 和 `/graphql/system` (系统数据)，通过 `scope` 参数区分 |
| REST "固定 `{ data, meta }`" 响应 | `meta` 是可选的，只有当查询参数包含 `?meta=...` 时才返回；单个查询、创建、更新操作没有 `meta` |
| REST 系统数据路由 | 不是通过 `items` 集合，而是有独立的控制器和路由：`/users`、`/roles`、`/permissions` 等 |

### 5.2 新增的深度内容

1. **服务层到数据库适配层的边界约束和关键分岔条件**
   - 操作类型分岔（读取 vs 写入）
   - 集合类型分岔（系统集合 vs 普通集合、单例集合 vs 普通集合）
   - 查询特征分岔（聚合/分组查询 vs 普通查询、多关系排序/过滤 vs 普通排序/过滤、有权限条件 vs 无权限条件）
   - 数据库类型分岔（PostgreSQL、MySQL、SQLite、MSSQL、Oracle）

2. **读场景和写场景的并排调用链对照**
   - 详细的调用链对照表
   - 关键差异点的代码示例
   - 统一与差异的总结

### 5.3 设计取舍的深层分析

- 统一服务层的优势与代价
- AST 作为中间表示的优势与代价
- REST 多路由 vs GraphQL 单一路由的设计考量
- meta 字段可选的设计考量

---

## 附录

### 附录 A: 关键分岔点速查表

| 分岔点 | 条件 | 分支 A | 分支 B | 代码位置 |
|--------|------|--------|--------|----------|
| **操作类型** | 方法调用 | 读取操作 (`read*`) | 写入操作 (`create*`/`update*`/`delete*`) | `items.ts` 各方法 |
| **集合类型** | `isSystemCollection()` | 系统集合 (`eventScope = collection`) | 普通集合 (`eventScope = 'items'`) | `items.ts:57` |
| **单例集合** | `schema.collections[collection].singleton` | 单例集合 | 普通集合 | `items.ts:1190` |
| **聚合查询** | `query.aggregate \|\| query.group` | 聚合/分组查询 | 普通查询 | `get-db-query.ts:47` |
| **内部查询** | `hasMultiRelationalSort \|\| hasMultiRelationalFilter` | 需要内部查询 | 不需要内部查询 | `get-db-query.ts:112` |
| **权限条件** | `hasCaseWhen` | 有权限条件 (GROUP BY) | 无权限条件 (DISTINCT) | `get-db-query.ts:41-43` |
| **管理员** | `accountability.admin` | 跳过权限检查 | 执行权限检查 | `process-ast.ts`, `validate-access.ts` |
| **数据库类型** | `getDatabaseClient()` | PostgreSQL/MySQL/SQLite/MSSQL/Oracle | - | `database/index.ts:253` |

### 附录 B: 完整路由对照表

| 资源类型 | REST 路由 | GraphQL 路由 | 说明 |
|----------|-----------|--------------|------|
| 业务数据列表 | `GET /items/articles` | `POST /graphql` (query: articles) | |
| 业务数据单个 | `GET /items/articles/1` | `POST /graphql` (query: articles_by_id) | |
| 业务数据创建 | `POST /items/articles` | `POST /graphql` (mutation: create_articles_item) | |
| 业务数据更新 | `PATCH /items/articles/1` | `POST /graphql` (mutation: update_articles_item) | |
| 业务数据删除 | `DELETE /items/articles/1` | `POST /graphql` (mutation: delete_articles_item) | |
| 用户列表 | `GET /users` | `POST /graphql/system` (query: users) | 系统数据 |
| 角色列表 | `GET /roles` | `POST /graphql/system` (query: roles) | 系统数据 |
| 权限列表 | `GET /permissions` | `POST /graphql/system` (query: permissions) | 系统数据 |

### 附录 C: 响应结构对照表

| 操作类型 | REST 响应结构 | GraphQL 响应结构 |
|----------|---------------|------------------|
| **列表查询 (有 meta)** | `{ meta: { total_count?, filter_count? }, data: Item[] }` | `{ data: { articles: Item[] }, errors?: Error[] }` |
| **列表查询 (无 meta)** | `{ data: Item[] }` | 同上 |
| **单个查询** | `{ data: Item \| null }` | `{ data: { articles_by_id: Item \| null }, errors?: Error[] }` |
| **创建操作** | `{ data: Item \| null }` | `{ data: { create_articles_item: Item \| boolean }, errors?: Error[] }` |
| **更新操作** | `{ data: Item \| null }` | `{ data: { update_articles_item: Item \| boolean }, errors?: Error[] }` |
| **删除操作** | `204 No Content` (无响应体) | `{ data: { delete_articles_item: { id: ID } }, errors?: Error[] }` |