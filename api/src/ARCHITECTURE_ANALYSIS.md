# Directus 数据读写架构分析报告

## 1. 整体架构概述

Directus 采用了分层架构设计，通过统一的服务层（Service Layer）为 REST 和 GraphQL 两种接口提供数据读写支持。这种设计确保了：

- **接口无关的业务逻辑**：核心业务逻辑完全封装在服务层，不依赖于任何特定的接口协议
- **一致的权限校验**：所有数据操作都经过相同的权限校验逻辑
- **统一的数据库交互**：通过抽象的 AST（抽象语法树）和 Knex.js 实现与多种数据库的兼容

### 架构层次结构

```
┌─────────────────────────────────────────────────────────────────┐
│                         接口层 (Interface Layer)                  │
│  ┌─────────────────────┐  ┌─────────────────────────────────┐   │
│  │    REST API         │  │         GraphQL API             │   │
│  │  (controllers/)     │  │      (controllers/graphql.ts)   │   │
│  └─────────────────────┘  └─────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      服务层 (Service Layer)                       │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    ItemsService (核心服务)                    │ │
│  │  - createOne/createMany                                       │ │
│  │  - readOne/readMany/readByQuery/readSingleton                │ │
│  │  - updateOne/updateMany/updateBatch/updateByQuery            │ │
│  │  - deleteOne/deleteMany/deleteByQuery                        │ │
│  │  - upsertOne/upsertMany/upsertSingleton                      │ │
│  └─────────────────────────────────────────────────────────────┘ │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    权限校验模块                                 │ │
│  │  - validateAccess (更新/删除操作)                              │ │
│  │  - processAst (读取操作)                                        │ │
│  │  - processPayload (创建/更新操作的数据处理)                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    数据库层 (Database Layer)                      │
│  ┌─────────────────────┐  ┌─────────────────────────────────┐   │
│  │   读取操作          │  │         写入操作                   │   │
│  │  - getAstFromQuery  │  │  - Knex.js insert/update/delete │   │
│  │  - processAst       │  │  - 事务管理                        │   │
│  │  - runAst           │  │                                 │   │
│  └─────────────────────┘  └─────────────────────────────────┘   │
│                              ↓                                    │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    Knex.js 数据库适配器                        │ │
│  │  - PostgreSQL / MySQL / MariaDB / SQLite / MS SQL Server    │ │
│  │  - OracleDB / CockroachDB                                     │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. REST API 请求处理流程

### 2.1 整体流程

REST API 的请求处理流程从 Express 路由开始，经过中间件处理后，调用相应的控制器方法，最终通过服务层完成数据操作。

```
HTTP 请求 → Express 路由 → 中间件链 → 控制器 → 服务层 → 数据库
                                      ↓
                              响应中间件 → HTTP 响应
```

### 2.2 详细流程分析

#### 2.2.1 路由定义

REST API 的路由定义在各个控制器文件中，以 `items.ts` 为例：

```typescript
// api/src/controllers/items.ts:13-59
const router = express.Router();

router.post(
	'/:collection',
	collectionExists,
	asyncHandler(async (req, res, next) => {
		// 创建操作处理
		const service = new ItemsService(req.collection, {
			accountability: req.accountability,
			schema: req.schema,
		});
		
		const savedKeys: PrimaryKey[] = [];
		if (Array.isArray(req.body)) {
			const keys = await service.createMany(req.body);
			savedKeys.push(...keys);
		} else {
			const key = await service.createOne(req.body);
			savedKeys.push(key);
		}
		
		// 读取创建的数据并返回
		if (Array.isArray(req.body)) {
			const result = await service.readMany(savedKeys, req.sanitizedQuery);
			res.locals['payload'] = { data: result || null };
		} else {
			const result = await service.readOne(savedKeys[0]!, req.sanitizedQuery);
			res.locals['payload'] = { data: result || null };
		}
		
		return next();
	}),
	respond,
);
```

#### 2.2.2 中间件处理

REST API 使用了多个中间件来处理请求的各个阶段：

| 中间件 | 功能 | 位置 |
|--------|------|------|
| `collectionExists` | 验证集合是否存在 | `api/src/middleware/collection-exists.ts` |
| `validateBatch` | 验证批量操作的参数 | `api/src/middleware/validate-batch.ts` |
| `sanitizeQuery` | 清理和验证查询参数 | `api/src/utils/sanitize-query.ts` |
| `respond` | 统一处理响应格式 | `api/src/middleware/respond.ts` |

#### 2.2.3 控制器到服务层的调用

控制器的主要职责是：

1. **解析 HTTP 请求**：从 URL 参数、查询字符串、请求体中提取数据
2. **参数验证**：通过中间件和直接检查确保参数有效性
3. **服务实例化**：创建 `ItemsService` 实例，传入必要的上下文信息
4. **调用服务方法**：根据 HTTP 方法调用相应的服务方法
5. **准备响应**：将服务返回的结果准备好，通过 `res.locals['payload']` 传递给响应中间件

### 2.3 REST API 的特殊处理

REST API 有一些与 HTTP 协议相关的特殊处理：

#### 2.3.1 HTTP 方法到服务方法的映射

| HTTP 方法 | 服务方法 | 说明 |
|-----------|----------|------|
| POST /:collection | createOne/createMany | 创建单个或多个项目 |
| GET /:collection | readByQuery | 按查询条件读取项目列表 |
| GET /:collection/:pk | readOne | 按主键读取单个项目 |
| PATCH /:collection | updateMany/updateByQuery/updateBatch | 更新多个项目 |
| PATCH /:collection/:pk | updateOne | 更新单个项目 |
| DELETE /:collection | deleteMany/deleteByQuery | 删除多个项目 |
| DELETE /:collection/:pk | deleteOne | 删除单个项目 |

#### 2.3.2 URL 参数处理

- `:collection`：从 URL 中提取集合名称，用于确定操作的数据表
- `:pk`：从 URL 中提取主键值，用于单个项目的操作

#### 2.3.3 查询参数处理

REST API 使用 `req.sanitizedQuery` 对象来处理查询参数，这些参数通过 `sanitizeQuery` 中间件进行清理和验证，包括：

- `fields`：指定返回的字段
- `filter`：过滤条件
- `sort`：排序规则
- `limit`/`offset`：分页参数
- `search`：搜索条件
- `aggregate`：聚合函数
- `group`：分组条件

---

## 3. GraphQL API 请求处理流程

### 3.1 整体流程

GraphQL API 的请求处理流程与 REST API 有所不同，它使用 GraphQL 引擎来解析和执行查询，然后通过解析器（Resolvers）调用服务层。

```
HTTP 请求 → Express 路由 → parseGraphQL 中间件 → GraphQL 引擎
                                                          ↓
                                                  解析器 (Resolvers)
                                                          ↓
                                              服务层 (Service Layer)
                                                          ↓
                                                    数据库
                                                          ↓
                                              respond 中间件 → HTTP 响应
```

### 3.2 详细流程分析

#### 3.2.1 路由定义

GraphQL API 的路由定义非常简洁，只有两个主要路由：

```typescript
// api/src/controllers/graphql.ts:7-50
const router = Router();

router.use(
	'/system',
	parseGraphQL,
	asyncHandler(async (req, res, next) => {
		const service = new GraphQLService({
			accountability: req.accountability,
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

router.use(
	'/',
	parseGraphQL,
	asyncHandler(async (req, res, next) => {
		const service = new GraphQLService({
			accountability: req.accountability,
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
```

#### 3.2.2 GraphQL 服务执行

`GraphQLService` 是 GraphQL API 的核心服务，负责执行 GraphQL 查询：

```typescript
// api/src/services/graphql/index.ts:31-99
export class GraphQLService {
	accountability: Accountability | null;
	knex: Knex;
	schema: SchemaOverview;
	scope: GQLScope;
	
	constructor(options: AbstractServiceOptions & { scope: GQLScope }) {
		this.accountability = options?.accountability || null;
		this.knex = options?.knex || getDatabase();
		this.schema = options.schema;
		this.scope = options.scope;
	}
	
	async execute({
		document,
		variables,
		operationName,
		contextValue,
	}: GraphQLParams): Promise<FormattedExecutionResult> {
		const schema = await this.getSchema();
		
		// 验证查询
		const validationErrors = validate(schema, document, validationRules).map((validationError) =>
			addPathToValidationError(validationError),
		);
		
		if (validationErrors.length > 0) {
			throw new GraphQLValidationError({ errors: validationErrors });
		}
		
		// 执行查询
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
			formattedResult.errors = result['errors'].map((error) => processError(this.accountability, error));
		}
		if (result['extensions']) formattedResult.extensions = result['extensions'];
		
		return formattedResult;
	}
}
```

#### 3.2.3 解析器（Resolvers）

解析器是 GraphQL 查询与服务层之间的桥梁，负责将 GraphQL 查询转换为 Directus 的查询格式，并调用相应的服务方法。

##### 查询解析器（resolveQuery）

```typescript
// api/src/services/graphql/resolvers/query.ts:15-82
export async function resolveQuery(gql: GraphQLService, info: GraphQLResolveInfo): Promise<Partial<Item> | null> {
	let collection = info.fieldName;
	if (gql.scope === 'system') collection = `directus_${collection}`;
	const selections = replaceFragmentsInSelections(info.fieldNodes[0]?.selectionSet?.selections, info.fragments);
	
	if (!selections) return null;
	const args: Record<string, any> = parseArgs(info.fieldNodes[0]!.arguments || [], info.variableValues);
	
	let query: Query;
	
	// 处理特殊的查询模式
	const isAggregate = collection.endsWith('_aggregated') && collection in gql.schema.collections === false;
	
	if (isAggregate) {
		collection = collection.slice(0, -11);
		query = await getAggregateQuery(args, selections, gql.schema, gql.accountability, collection);
	} else {
		if (collection.endsWith('_by_id') && collection in gql.schema.collections === false) {
			collection = collection.slice(0, -6);
		}
		
		query = await getQuery(args, gql.schema, selections, info.variableValues, gql.accountability, collection);
		
		if (collection.endsWith('_by_version') && collection in gql.schema.collections === false) {
			collection = collection.slice(0, -11);
			query.versionRaw = true;
		}
	}
	
	// 调用 GraphQLService 的 read 方法
	const result = await gql.read(collection, query, args['id']);
	
	// 处理聚合查询的特殊格式
	if (args['id']) return result;
	
	if (query.group) {
		const aggregateKeys = Object.keys(query.aggregate ?? {});
		result['map']((payload: Item) => {
			payload['group'] = omit(payload, aggregateKeys);
		});
	}
	
	return result;
}
```

##### 突变解析器（resolveMutation）

```typescript
// api/src/services/graphql/resolvers/mutation.ts:9-92
export async function resolveMutation(
	gql: GraphQLService,
	args: Record<string, any>,
	info: GraphQLResolveInfo,
): Promise<Partial<Item> | boolean | undefined> {
	// 解析操作类型和集合名称
	const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
	let collection = info.fieldName.substring(action.length + 1);
	if (gql.scope === 'system') collection = `directus_${collection}`;
	
	const selections = replaceFragmentsInSelections(info.fieldNodes[0]?.selectionSet?.selections, info.fragments);
	const query = await getQuery(args, gql.schema, selections || [], info.variableValues, gql.accountability, collection);
	
	// 处理特殊的集合类型
	const singleton =
		collection.endsWith('_batch') === false &&
		collection.endsWith('_items') === false &&
		collection.endsWith('_item') === false &&
		collection in gql.schema.collections;
	
	const single = collection.endsWith('_items') === false && collection.endsWith('_batch') === false;
	const batchUpdate = action === 'update' && collection.endsWith('_batch');
	
	// 清理集合名称后缀
	if (collection.endsWith('_batch')) collection = collection.slice(0, -6);
	if (collection.endsWith('_items')) collection = collection.slice(0, -6);
	if (collection.endsWith('_item')) collection = collection.slice(0, -5);
	
	// 单例集合的更新处理
	if (singleton && action === 'update') {
		return await gql.upsertSingleton(collection, args['data'], query);
	}
	
	// 获取服务实例
	const service = getService(collection, {
		knex: gql.knex,
		accountability: gql.accountability,
		schema: gql.schema,
	});
	
	const hasQuery = (query.fields || []).length > 0;
	
	// 根据操作类型调用相应的服务方法
	try {
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
			
			return undefined;
		} else {
			if (action === 'create') {
				const keys = await service.createMany(args['data']);
				return hasQuery ? await service.readMany(keys, query) : true;
			}
			
			if (action === 'update') {
				const keys: PrimaryKey[] = [];
				
				if (batchUpdate) {
					keys.push(...(await service.updateBatch(args['data'])));
				} else {
					keys.push(...(await service.updateMany(args['ids'], args['data'])));
				}
				
				return hasQuery ? await service.readMany(keys, query) : true;
			}
			
			if (action === 'delete') {
				const keys = await service.deleteMany(args['ids']);
				return { ids: keys };
			}
			
			return undefined;
		}
	} catch (err: any) {
		return formatError(err);
	}
}
```

### 3.3 GraphQL API 的特殊处理

GraphQL API 有一些与 GraphQL 协议相关的特殊处理：

#### 3.3.1 查询模式

GraphQL API 支持多种查询模式，通过字段名称的后缀来区分：

| 字段模式 | 说明 | 服务方法 |
|----------|------|----------|
| `collection` | 普通列表查询 | `readByQuery` |
| `collection_by_id` | 按 ID 查询单个项目 | `readOne` |
| `collection_aggregated` | 聚合查询 | 特殊聚合处理 |
| `collection_by_version` | 版本查询 | 带版本参数的读取 |

#### 3.3.2 突变模式

GraphQL API 的突变操作也通过字段名称来区分：

| 字段模式 | 说明 | 服务方法 |
|----------|------|----------|
| `create_collection_item` | 创建单个项目 | `createOne` |
| `create_collection_items` | 创建多个项目 | `createMany` |
| `update_collection_item` | 更新单个项目 | `updateOne` |
| `update_collection_items` | 更新多个项目 | `updateMany` |
| `update_collection_batch` | 批量更新 | `updateBatch` |
| `delete_collection_item` | 删除单个项目 | `deleteOne` |
| `delete_collection_items` | 删除多个项目 | `deleteMany` |

#### 3.3.3 单例集合处理

对于标记为单例（singleton）的集合，GraphQL API 有特殊的处理逻辑：

- 读取操作：调用 `readSingleton` 方法
- 更新操作：调用 `upsertSingleton` 方法（自动处理创建或更新）

#### 3.3.4 GraphQL 到 Directus 查询的转换

GraphQL API 的一个重要职责是将 GraphQL 的查询格式转换为 Directus 的查询格式，这主要通过以下函数完成：

- `parseArgs`：解析 GraphQL 字段参数
- `getQuery`：将 GraphQL 选择集转换为 Directus 查询对象
- `getAggregateQuery`：处理聚合查询的特殊转换
- `replaceFragmentsInSelections`：处理 GraphQL 片段

---

## 4. 统一服务层（ItemsService）分析

### 4.1 服务层的核心作用

`ItemsService` 是 Directus 数据读写的核心服务，被 REST 和 GraphQL 两种接口共同使用。它的主要职责包括：

1. **封装业务逻辑**：所有与数据操作相关的业务逻辑都集中在这里
2. **统一权限校验**：确保所有数据操作都经过相同的权限检查
3. **管理数据一致性**：通过事务机制确保数据操作的原子性
4. **处理事件钩子**：在数据操作的各个阶段触发相应的事件
5. **管理缓存**：在数据变更时清理相关的缓存
6. **处理数据转换**：通过 `PayloadService` 处理数据的类型转换和验证

### 4.2 服务层的主要方法

#### 4.2.1 读取操作

##### readByQuery - 按查询条件读取

```typescript
// api/src/services/items.ts:499-580
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
	// 1. 触发查询前的 filter 钩子
	const updatedQuery =
		opts?.emitEvents !== false
			? await emitter.emitFilter(
					this.eventScope === 'items'
						? ['items.query', `${this.collection}.items.query`]
						: `${this.eventScope}.query`,
					query,
					{
						collection: this.collection,
					},
					{
						database: this.knex,
						schema: this.schema,
						accountability: this.accountability,
					},
				)
			: query;
	
	// 2. 将查询转换为 AST
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
	
	// 3. 处理权限校验（核心步骤）
	ast = await processAst(
		{ ast, action: 'read', accountability: this.accountability },
		{ knex: this.knex, schema: this.schema },
	);
	
	// 4. 执行查询
	const records = await runAst(ast, this.schema, this.accountability, {
		knex: this.knex,
		stripNonRequested: opts?.stripNonRequested !== undefined ? opts.stripNonRequested : true,
	});
	
	// 5. 检查是否有权限访问
	if (records === null) {
		throw new ForbiddenError();
	}
	
	// 6. 触发读取后的 filter 钩子
	const filteredRecords =
		opts?.emitEvents !== false
			? await emitter.emitFilter(
					this.eventScope === 'items' ? ['items.read', `${this.collection}.items.read`] : `${this.eventScope}.read`,
					records,
					{
						query: updatedQuery,
						collection: this.collection,
					},
					{
						database: this.knex,
						schema: this.schema,
						accountability: this.accountability,
					},
				)
			: records;
	
	// 7. 触发读取后的 action 钩子
	if (opts?.emitEvents !== false) {
		emitter.emitAction(
			this.eventScope === 'items' ? ['items.read', `${this.collection}.items.read`] : `${this.eventScope}.read`,
			{
				payload: filteredRecords,
				query: updatedQuery,
				collection: this.collection,
			},
			{
				database: this.knex || getDatabase(),
				schema: this.schema,
				accountability: this.accountability,
			},
		);
	}
	
	return filteredRecords as Item[];
}
```

##### 其他读取方法

| 方法 | 说明 | 底层实现 |
|------|------|----------|
| `readOne(key, query)` | 按主键读取单个项目 | 调用 `readByQuery`，添加主键过滤条件 |
| `readMany(keys, query)` | 按主键列表读取多个项目 | 调用 `readByQuery`，添加主键 IN 过滤条件 |
| `readSingleton(query)` | 读取单例集合 | 调用 `readByQuery`，设置 limit=1，处理默认值 |

#### 4.2.2 创建操作

##### createOne - 创建单个项目

```typescript
// api/src/services/items.ts:127-426
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
	// 1. 初始化突变追踪器
	if (!opts.mutationTracker) opts.mutationTracker = this.createMutationTracker();
	
	if (!opts.bypassLimits) {
		opts.mutationTracker.trackMutations(1);
	}
	
	// 2. 在事务中执行创建操作
	const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
		// 3. 触发创建前的 filter 钩子
		const payloadAfterHooks =
			opts.emitEvents !== false
				? await emitter.emitFilter(
						this.eventScope === 'items'
							? ['items.create', `${this.collection}.items.create`]
							: `${this.eventScope}.create`,
						payload,
						{
							collection: this.collection,
						},
						{
							database: trx,
							schema: this.schema,
							accountability: this.accountability,
						},
					)
				: payload;
		
		// 4. 处理数据权限和预设值（核心权限校验步骤）
		const payloadWithPresets = this.accountability
			? await processPayload(
					{
						accountability: this.accountability,
						action: 'create',
						collection: this.collection,
						payload: payloadAfterHooks,
						nested: this.nested,
					},
					{
						knex: trx,
						schema: this.schema,
					},
				)
			: payloadAfterHooks;
		
		// 5. 使用 PayloadService 处理数据
		const payloadService = new PayloadService(this.collection, {
			accountability: this.accountability,
			knex: trx,
			schema: this.schema,
			nested: this.nested,
			overwriteDefaults: opts.overwriteDefaults,
		});
		
		// 处理多对一关系
		const {
			payload: payloadWithM2O,
			revisions: revisionsM2O,
			nestedActionEvents: nestedActionEventsM2O,
			userIntegrityCheckFlags: userIntegrityCheckFlagsM2O,
		} = await payloadService.processM2O(payloadWithPresets, opts);
		
		// 处理任意对一关系
		const {
			payload: payloadWithA2O,
			revisions: revisionsA2O,
			nestedActionEvents: nestedActionEventsA2O,
			userIntegrityCheckFlags: userIntegrityCheckFlagsA2O,
		} = await payloadService.processA2O(payloadWithM2O, opts);
		
		// 处理数据类型转换
		const payloadWithoutAliases = pick(payloadWithA2O, without(fields, ...aliases));
		const payloadWithTypeCasting = await payloadService.processValues('create', payloadWithoutAliases);
		
		// 6. 执行数据库插入
		try {
			let returningOptions = undefined;
			
			// 支持 MSSQL 表的触发器
			if (getDatabaseClient(trx) === 'mssql') {
				returningOptions = { includeTriggerModifications: true };
			}
			
			const result = await trx
				.insert(payloadWithoutAliases)
				.into(this.collection)
				.returning(primaryKeyField, returningOptions)
				.then((result) => result[0]);
			
			// 处理返回的主键
			const returnedKey = typeof result === 'object' ? result[primaryKeyField] : result;
			
			if (pkField!.type === 'uuid') {
				primaryKey = getHelpers(trx).schema.formatUUID(primaryKey ?? returnedKey);
			} else {
				primaryKey = primaryKey ?? returnedKey;
			}
		} catch (err: any) {
			const dbError = await translateDatabaseError(err, data);
			throw dbError;
		}
		
		// 7. 处理一对多关系
		const {
			revisions: revisionsO2M,
			nestedActionEvents: nestedActionEventsO2M,
			userIntegrityCheckFlags: userIntegrityCheckFlagsO2M,
		} = await payloadService.processO2M(payloadWithPresets, primaryKey, opts);
		
		// 8. 验证用户完整性（如果需要）
		const userIntegrityCheckFlags =
			(opts.userIntegrityCheckFlags ?? UserIntegrityCheckFlag.None) |
			userIntegrityCheckFlagsM2O |
			userIntegrityCheckFlagsA2O |
			userIntegrityCheckFlagsO2M;
		
		if (userIntegrityCheckFlags) {
			if (opts.onRequireUserIntegrityCheck) {
				opts.onRequireUserIntegrityCheck(userIntegrityCheckFlags);
			} else {
				await validateUserCountIntegrity({ flags: userIntegrityCheckFlags, knex: trx });
			}
		}
		
		// 9. 创建活动记录和修订记录（如果启用了问责制）
		if (
			opts.skipTracking !== true &&
			this.accountability &&
			this.schema.collections[this.collection]!.accountability !== null
		) {
			const { ActivityService } = await import('./activity.js');
			const { RevisionsService } = await import('./revisions.js');
			
			const activityService = new ActivityService({
				knex: trx,
				schema: this.schema,
			});
			
			const activity = await activityService.createOne({
				action: Action.CREATE,
				user: this.accountability!.user,
				collection: this.collection,
				ip: this.accountability!.ip,
				user_agent: this.accountability!.userAgent,
				origin: this.accountability!.origin,
				item: primaryKey,
			});
			
			// 如果启用了修订追踪，创建修订记录
			if (this.schema.collections[this.collection]!.accountability === 'all') {
				// ... 创建修订记录的逻辑
			}
		}
		
		return primaryKey;
	});
	
	// 10. 触发创建后的 action 钩子
	if (opts.emitEvents !== false) {
		const actionEvent = {
			event:
				this.eventScope === 'items'
					? ['items.create', `${this.collection}.items.create`]
					: `${this.eventScope}.create`,
			meta: {
				payload: actionHookPayload,
				key: primaryKey,
				collection: this.collection,
			},
			context: {
				database: getDatabase(),
				schema: this.schema,
				accountability: this.accountability,
			},
		};
		
		if (opts.bypassEmitAction) {
			opts.bypassEmitAction(actionEvent);
		} else {
			emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
		}
		
		// 触发嵌套关系的 action 事件
		for (const nestedActionEvent of nestedActionEvents) {
			if (opts.bypassEmitAction) {
				opts.bypassEmitAction(nestedActionEvent);
			} else {
				emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
			}
		}
	}
	
	// 11. 清理缓存
	if (shouldClearCache(this.cache, opts, this.collection)) {
		await this.cache.clear();
	}
	
	return primaryKey;
}
```

##### 其他创建方法

| 方法 | 说明 | 底层实现 |
|------|------|----------|
| `createMany(data, opts)` | 创建多个项目 | 在事务中循环调用 `createOne` |

#### 4.2.3 更新操作

##### updateMany - 更新多个项目

```typescript
// api/src/services/items.ts:709-974
async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 1. 初始化突变追踪器
	if (!opts.mutationTracker) opts.mutationTracker = this.createMutationTracker();
	
	if (!opts.bypassLimits) {
		opts.mutationTracker.trackMutations(keys.length);
	}
	
	// 2. 验证主键
	const primaryKeyField = this.schema.collections[this.collection]!.primary;
	validateKeys(this.schema, this.collection, primaryKeyField, keys);
	
	// 3. 触发更新前的 filter 钩子
	const payloadAfterHooks =
		opts.emitEvents !== false
			? await emitter.emitFilter(
					this.eventScope === 'items'
						? ['items.update', `${this.collection}.items.update`]
						: `${this.eventScope}.update`,
					payload,
					{
						keys,
						collection: this.collection,
					},
					{
						database: this.knex,
						schema: this.schema,
						accountability: this.accountability,
					},
				)
			: payload;
	
	// 4. 排序主键以确保顺序
	keys.sort();
	
	// 5. 权限校验（核心步骤）
	if (this.accountability) {
		await validateAccess(
			{
				accountability: this.accountability,
				action: 'update',
				collection: this.collection,
				primaryKeys: keys,
				fields: Object.keys(payloadAfterHooks),
			},
			{
				schema: this.schema,
				knex: this.knex,
			},
		);
	}
	
	// 6. 处理数据权限和预设值
	const payloadWithPresets = this.accountability
		? await processPayload(
				{
					accountability: this.accountability,
					action: 'update',
					collection: this.collection,
					payload: payloadAfterHooks,
					nested: this.nested,
				},
				{
					knex: this.knex,
					schema: this.schema,
				},
			)
		: payloadAfterHooks;
	
	// 7. 在事务中执行更新
	await transaction(this.knex, async (trx) => {
		const payloadService = new PayloadService(this.collection, {
			accountability: this.accountability,
			knex: trx,
			schema: this.schema,
			nested: this.nested,
			overwriteDefaults: opts.overwriteDefaults,
		});
		
		// 处理多对一关系
		const {
			payload: payloadWithM2O,
			revisions: revisionsM2O,
			nestedActionEvents: nestedActionEventsM2O,
			userIntegrityCheckFlags: userIntegrityCheckFlagsM2O,
		} = await payloadService.processM2O(payloadWithPresets, opts);
		
		// 处理任意对一关系
		const {
			payload: payloadWithA2O,
			revisions: revisionsA2O,
			nestedActionEvents: nestedActionEventsA2O,
			userIntegrityCheckFlags: userIntegrityCheckFlagsA2O,
		} = await payloadService.processA2O(payloadWithM2O, opts);
		
		// 处理数据类型转换
		const payloadWithoutAliasAndPK = pick(payloadWithA2O, without(fields, primaryKeyField, ...aliases));
		const payloadWithTypeCasting = await payloadService.processValues('update', payloadWithoutAliasAndPK);
		
		// 8. 执行数据库更新
		if (Object.keys(payloadWithTypeCasting).length > 0) {
			try {
				await trx(this.collection).update(payloadWithTypeCasting).whereIn(primaryKeyField, keys);
			} catch (err: any) {
				throw await translateDatabaseError(err, data);
			}
		}
		
		// 9. 处理一对多关系
		for (const key of keys) {
			const {
				revisions,
				nestedActionEvents: nestedActionEventsO2M,
				userIntegrityCheckFlags: userIntegrityCheckFlagsO2M,
			} = await payloadService.processO2M(payloadWithA2O, key, opts);
			
			// ... 处理修订记录和嵌套事件
		}
		
		// 10. 验证用户完整性
		if (userIntegrityCheckFlags) {
			if (opts?.onRequireUserIntegrityCheck) {
				opts.onRequireUserIntegrityCheck(userIntegrityCheckFlags);
			} else {
				await validateUserCountIntegrity({ flags: userIntegrityCheckFlags, knex: trx });
			}
		}
		
		// 11. 创建活动记录和修订记录
		if (
			opts.skipTracking !== true &&
			this.accountability &&
			this.schema.collections[this.collection]!.accountability !== null
		) {
			// ... 创建活动记录和修订记录的逻辑
		}
	});
	
	// 12. 清理缓存
	if (shouldClearCache(this.cache, opts, this.collection)) {
		await this.cache.clear();
	}
	
	// 13. 触发更新后的 action 钩子
	if (opts.emitEvents !== false) {
		// ... 触发 action 事件的逻辑
	}
	
	return keys;
}
```

##### 其他更新方法

| 方法 | 说明 | 底层实现 |
|------|------|----------|
| `updateOne(key, data, opts)` | 更新单个项目 | 调用 `updateMany([key], data, opts)` |
| `updateBatch(data, opts)` | 批量更新多个项目（每个项目可以有不同的数据） | 在事务中循环调用 `updateOne` |
| `updateByQuery(query, data, opts)` | 按查询条件更新项目 | 先通过查询获取主键，再调用 `updateMany` |
| `upsertOne(payload, opts)` | 更新或创建单个项目 | 检查是否存在，存在则 `updateOne`，否则 `createOne` |
| `upsertMany(payloads, opts)` | 更新或创建多个项目 | 在事务中循环调用 `upsertOne` |
| `upsertSingleton(data, opts)` | 更新或创建单例集合项目 | 检查是否存在，存在则 `updateOne`，否则 `createOne` |

#### 4.2.4 删除操作

##### deleteMany - 删除多个项目

```typescript
// api/src/services/items.ts:1070-1185
async deleteMany(keys: PrimaryKey[], opts: MutationOptions = {}): Promise<PrimaryKey[]> {
	// 1. 初始化突变追踪器
	if (!opts.mutationTracker) opts.mutationTracker = this.createMutationTracker();
	
	if (!opts.bypassLimits) {
		opts.mutationTracker.trackMutations(keys.length);
	}
	
	// 2. 验证主键
	const primaryKeyField = this.schema.collections[this.collection]!.primary;
	validateKeys(this.schema, this.collection, primaryKeyField, keys);
	
	// 3. 触发删除前的 filter 钩子
	const keysAfterHooks =
		opts.emitEvents !== false
			? await emitter.emitFilter(
					this.eventScope === 'items'
						? ['items.delete', `${this.collection}.items.delete`]
						: `${this.eventScope}.delete`,
					keys,
					{
						collection: this.collection,
					},
					{
						database: this.knex,
						schema: this.schema,
						accountability: this.accountability,
					},
				)
			: keys;
	
	// 4. 权限校验（核心步骤）
	if (this.accountability) {
		await validateAccess(
			{
				accountability: this.accountability,
				action: 'delete',
				collection: this.collection,
				primaryKeys: keysAfterHooks,
			},
			{
				knex: this.knex,
				schema: this.schema,
			},
		);
	}
	
	// 5. 在事务中执行删除
	await transaction(this.knex, async (trx) => {
		// 6. 执行数据库删除
		await trx(this.collection).whereIn(primaryKeyField, keysAfterHooks).delete();
		
		// 7. 验证用户完整性
		if (opts.userIntegrityCheckFlags) {
			if (opts.onRequireUserIntegrityCheck) {
				opts.onRequireUserIntegrityCheck(opts.userIntegrityCheckFlags);
			} else {
				await validateUserCountIntegrity({ flags: opts.userIntegrityCheckFlags, knex: trx });
			}
		}
		
		// 8. 创建活动记录
		if (
			opts.skipTracking !== true &&
			this.accountability &&
			this.schema.collections[this.collection]!.accountability !== null
		) {
			const { ActivityService } = await import('./activity.js');
			
			const activityService = new ActivityService({
				knex: trx,
				schema: this.schema,
			});
			
			await activityService.createMany(
				keysAfterHooks.map((key) => ({
					action: Action.DELETE,
					user: this.accountability!.user,
					collection: this.collection,
					ip: this.accountability!.ip,
					user_agent: this.accountability!.userAgent,
					origin: this.accountability!.origin,
					item: key,
				})),
				{ bypassLimits: true },
			);
		}
	});
	
	// 9. 清理缓存
	if (shouldClearCache(this.cache, opts, this.collection)) {
		await this.cache.clear();
	}
	
	// 10. 触发删除后的 action 钩子
	if (opts.emitEvents !== false) {
		const actionEvent = {
			event:
				this.eventScope === 'items'
					? ['items.delete', `${this.collection}.items.delete`]
					: `${this.eventScope}.delete`,
			meta: {
				payload: keysAfterHooks,
				keys: keysAfterHooks,
				collection: this.collection,
			},
			context: {
				database: getDatabase(),
				schema: this.schema,
				accountability: this.accountability,
			},
		};
		
		if (opts.bypassEmitAction) {
			opts.bypassEmitAction(actionEvent);
		} else {
			emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
		}
	}
	
	return keysAfterHooks;
}
```

##### 其他删除方法

| 方法 | 说明 | 底层实现 |
|------|------|----------|
| `deleteOne(key, opts)` | 删除单个项目 | 调用 `deleteMany([key], opts)` |
| `deleteByQuery(query, opts)` | 按查询条件删除项目 | 先通过查询获取主键，再调用 `deleteMany` |

### 4.3 服务层的关键机制

#### 4.3.1 事务管理

服务层使用 `transaction` 函数来确保数据操作的原子性：

```typescript
// 在事务中执行操作
const result = await transaction(this.knex, async (trx) => {
	// 在事务中创建服务实例的分支
	const service = this.fork({ knex: trx });
	
	// 执行多个操作
	const key1 = await service.createOne(data1);
	const key2 = await service.createOne(data2);
	
	return { key1, key2 };
});
```

`transaction` 函数确保：
- 所有操作要么全部成功，要么全部失败
- 操作失败时自动回滚
- 操作成功时自动提交

#### 4.3.2 事件钩子系统

服务层集成了完整的事件钩子系统，允许扩展在数据操作的各个阶段介入：

##### Filter 钩子

Filter 钩子允许在操作执行前修改数据或查询：

- **读取操作**：
  - `items.query` / `{collection}.items.query`：在查询执行前修改查询条件
  - `items.read` / `{collection}.items.read`：在数据返回前修改结果

- **写入操作**：
  - `items.create` / `{collection}.items.create`：在创建前修改数据
  - `items.update` / `{collection}.items.update`：在更新前修改数据
  - `items.delete` / `{collection}.items.delete`：在删除前修改主键列表

##### Action 钩子

Action 钩子在操作成功执行后触发，用于执行副作用操作：

- `items.create` / `{collection}.items.create`：创建成功后触发
- `items.read` / `{collection}.items.read`：读取成功后触发
- `items.update` / `{collection}.items.update`：更新成功后触发
- `items.delete` / `{collection}.items.delete`：删除成功后触发

#### 4.3.3 缓存管理

服务层在数据变更时会清理相关的缓存：

```typescript
// 检查是否需要清理缓存
if (shouldClearCache(this.cache, opts, this.collection)) {
	await this.cache.clear();
}
```

缓存清理策略：
- 创建、更新、删除操作后自动清理缓存
- 可以通过 `opts.autoPurgeCache` 选项控制是否清理
- 可以通过 `opts.bypassEmitAction` 等选项进行更精细的控制

#### 4.3.4 问责制追踪（Accountability）

服务层支持对数据操作进行追踪，记录操作的用户、时间、IP 等信息：

```typescript
// 创建活动记录
const activity = await activityService.createOne({
	action: Action.CREATE, // 或 UPDATE/DELETE
	user: this.accountability!.user,
	collection: this.collection,
	ip: this.accountability!.ip,
	user_agent: this.accountability!.userAgent,
	origin: this.accountability!.origin,
	item: primaryKey,
});

// 创建修订记录（如果启用）
if (this.schema.collections[this.collection]!.accountability === 'all') {
	const revision = await revisionsService.createOne({
		activity: activity,
		collection: this.collection,
		item: primaryKey,
		data: revisionPayload,
		delta: revisionPayload,
	});
}
```

---

## 5. 权限校验机制

### 5.1 权限校验的核心函数

Directus 使用三个核心函数来进行权限校验：

| 函数 | 适用操作 | 作用 |
|------|----------|------|
| `validateAccess` | 更新、删除 | 验证用户是否有权限对指定的项目执行操作 |
| `processAst` | 读取 | 处理读取操作的权限，修改 AST 以应用权限限制 |
| `processPayload` | 创建、更新 | 处理创建/更新操作的数据，应用权限预设值和验证 |

### 5.2 validateAccess 函数分析

`validateAccess` 函数用于验证用户是否有权限执行更新或删除操作：

```typescript
// api/src/permissions/modules/validate-access/validate-access.ts:22-57
export async function validateAccess(options: ValidateAccessOptions, context: Context) {
	// 1. 检查集合是否存在
	if (!options.skipCollectionExistsCheck && options.collection in context.schema.collections === false) {
		throw createCollectionForbiddenError('', options.collection);
	}
	
	// 2. 管理员用户跳过权限检查
	if (options.accountability.admin === true) {
		return;
	}
	
	let access: boolean;
	
	// 3. 根据是否提供主键选择不同的验证方式
	if (options.primaryKeys) {
		// 提供了主键，需要实际读取项目来验证权限
		const result = await validateItemAccess(options as Required<ValidateAccessOptions>, context);
		access = result.accessAllowed;
	} else {
		// 没有提供主键，只需要检查集合级别的权限
		access = await validateCollectionAccess(options, context);
	}
	
	// 4. 如果没有权限，抛出错误
	if (!access) {
		if (options.fields?.length ?? 0 > 0) {
			throw new ForbiddenError({
				reason: `You don't have permissions to perform "${options.action}" for the field(s) ${options
					.fields!.map((field) => `"${field}"`)
					.join(', ')} in collection "${options.collection}" or it does not exist.`,
			});
		}
		
		throw new ForbiddenError({
			reason: `You don't have permission to perform "${options.action}" for collection "${options.collection}" or it does not exist.`,
		});
	}
}
```

### 5.3 processAst 函数分析

`processAst` 函数用于处理读取操作的权限，它会修改 AST 以应用权限限制：

```typescript
// api/src/permissions/modules/process-ast/process-ast.ts:19-67
export async function processAst(options: ProcessAstOptions, context: Context) {
	// 1. 从 AST 中提取字段映射
	const fieldMap: FieldMap = fieldMapFromAst(options.ast, context.schema);
	const collections = collectionsInFieldMap(fieldMap);
	
	// 2. 管理员用户或无 accountability 时只验证字段存在性
	if (!options.accountability || options.accountability.admin) {
		for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
			validatePathExistence(path, collection, fields, context.schema);
		}
		
		return options.ast;
	}
	
	// 3. 获取用户的策略和权限
	const policies = await fetchPolicies(options.accountability, context);
	
	const permissions = await fetchPermissions(
		{ action: options.action, policies, collections, accountability: options.accountability },
		context,
	);
	
	// 4. 如果不是读取操作，还需要获取读取权限
	const readPermissions =
		options.action === 'read'
			? permissions
			: await fetchPermissions(
					{ action: 'read', policies, collections, accountability: options.accountability },
					context,
				);
	
	// 5. 验证字段存在性
	for (const [path, { collection, fields }] of [...fieldMap.read.entries(), ...fieldMap.other.entries()]) {
		validatePathExistence(path, collection, fields, context.schema);
	}
	
	// 6. 验证非读取字段的权限
	for (const [path, { collection, fields }] of fieldMap.other.entries()) {
		validatePathPermissions(path, permissions, collection, fields);
	}
	
	// 7. 验证只读字段的权限
	for (const [path, { collection, fields }] of fieldMap.read.entries()) {
		validatePathPermissions(path, readPermissions, collection, fields);
	}
	
	// 8. 注入权限条件到 AST 中
	injectCases(options.ast, permissions);
	
	return options.ast;
}
```

`processAst` 函数的关键作用：

1. **字段存在性验证**：确保查询的字段在集合中存在
2. **权限验证**：确保用户有权限访问请求的字段
3. **条件注入**：将权限条件（如 `{ user: { _eq: $CURRENT_USER } }`）注入到 AST 中，这样在执行查询时会自动应用这些条件

### 5.4 权限校验的触发点

权限校验在服务层的不同方法中被触发：

| 操作类型 | 权限校验函数 | 触发位置 |
|----------|--------------|----------|
| 读取 | `processAst` | `readByQuery` 方法中，在 `getAstFromQuery` 之后，`runAst` 之前 |
| 创建 | `processPayload` | `createOne` 方法中，在 filter 钩子之后，数据库插入之前 |
| 更新 | `validateAccess` + `processPayload` | `updateMany` 方法中，在 filter 钩子之后，数据库更新之前 |
| 删除 | `validateAccess` | `deleteMany` 方法中，在 filter 钩子之后，数据库删除之前 |

### 5.5 管理员用户的特殊处理

对于管理员用户（`accountability.admin === true`），Directus 会跳过大部分权限检查：

1. **`validateAccess`**：直接返回，不进行任何权限检查
2. **`processAst`**：只验证字段存在性，不验证权限，也不注入条件
3. **`processPayload`**：同样会跳过权限相关的检查

这种设计确保了管理员用户可以访问和修改所有数据。

---

## 6. 数据库交互层

### 6.1 读取操作的数据库交互

读取操作通过抽象语法树（AST）来实现与数据库的交互，主要涉及三个函数：

1. **`getAstFromQuery`**：将 Directus 查询对象转换为抽象语法树（AST）
2. **`processAst`**：处理权限，修改 AST 以应用权限限制
3. **`runAst`**：执行 AST，生成实际的数据库查询并执行

#### 6.1.1 runAst 函数分析

`runAst` 函数是执行读取操作的核心，它将 AST 转换为实际的数据库查询：

```typescript
// api/src/database/run-ast/run-ast.ts:20-193
export async function runAst(
	originalAST: AST | NestedCollectionNode,
	schema: SchemaOverview,
	accountability: Accountability | null,
	options?: RunASTOptions,
): Promise<null | Item | Item[]> {
	const ast = cloneDeep(originalAST);
	const knex = options?.knex || getDatabase();
	
	// 处理任意对一关系的特殊情况
	if (ast.type === 'a2o') {
		const results: { [collection: string]: null | Item | Item[] } = {};
		
		for (const collection of ast.names) {
			results[collection] = await run(
				collection,
				ast.children[collection]!,
				ast.query[collection]!,
				ast.cases[collection] ?? [],
				accountability,
			);
		}
		
		return results;
	} else {
		return await run(ast.name, ast.children, options?.query || ast.query, ast.cases, accountability);
	}
	
	async function run(
		collection: string,
		children: (NestedCollectionNode | FieldNode | FunctionFieldNode)[],
		query: Query,
		cases: Filter[],
		accountability: Accountability | null,
	) {
		const env = useEnv();
		
		// 1. 解析当前层级的字段和关系
		const { fieldNodes, primaryKeyField, nestedCollectionNodes } = await parseCurrentLevel(
			schema,
			collection,
			children,
			query,
		);
		
		const o2mNodes = nestedCollectionNodes.filter((node): node is O2MNode => node.type === 'o2m');
		
		// 2. 获取权限（如果不是管理员）
		let permissions: Permission[] = [];
		
		if (accountability && !accountability.admin) {
			const policies = await fetchPolicies(accountability, { schema, knex });
			permissions = await fetchPermissions({ action: 'read', accountability, policies }, { schema, knex });
		}
		
		// 3. 构建数据库查询
		const dbQuery = getDBQuery(
			{
				table: collection,
				fieldNodes,
				o2mNodes,
				query,
				cases,
				permissions,
			},
			{ schema, knex },
		);
		
		// 4. 执行数据库查询
		const rawItems: Item | Item[] = await dbQuery;
		
		if (!rawItems) return null;
		
		// 5. 处理数据转换
		const payloadService = new PayloadService(collection, { accountability, knex, schema });
		
		// 构建别名映射
		const aliasMap: Record<string, string> = { ...query.alias };
		
		for (const child of children) {
			if (child.type === 'functionField' && child.name.startsWith('json(')) {
				const alias = applyFunctionToColumnName(child.fieldKey);
				aliasMap[alias] = child.name;
			}
		}
		
		// 处理数据类型转换
		let items: null | Item | Item[] = await payloadService.processValues(
			'read',
			rawItems,
			aliasMap,
			query.aggregate ?? {},
		);
		
		if (!items || (Array.isArray(items) && items.length === 0)) return items;
		
		// 6. 处理嵌套关系
		const nestedNodes = applyParentFilters(schema, nestedCollectionNodes, items);
		
		for (const nestedNode of nestedNodes) {
			let nestedItems: Item[] | null = [];
			
			if (nestedNode.type === 'o2m') {
				// 处理一对多关系，支持批量查询
				let hasMore = true;
				let batchCount = 0;
				
				// 处理权限条件标记
				const hasWhenCase = nestedNode.whenCase && nestedNode.whenCase.length > 0;
				let fieldAllowed: boolean | boolean[] = true;
				
				if (hasWhenCase) {
					// 提取权限标记并从结果中移除
					if (Array.isArray(items)) {
						fieldAllowed = [];
						for (const item of items) {
							fieldAllowed.push(!!item[nestedNode.fieldKey]);
							delete item[nestedNode.fieldKey];
						}
					} else {
						fieldAllowed = !!items[nestedNode.fieldKey];
						delete items[nestedNode.fieldKey];
					}
				}
				
				// 批量获取嵌套数据
				while (hasMore) {
					const node = merge({}, nestedNode, {
						query: {
							limit: env['RELATIONAL_BATCH_SIZE'],
							offset: batchCount * (env['RELATIONAL_BATCH_SIZE'] as number),
							page: null,
						},
					});
					
					// 递归调用 runAst 获取嵌套数据
					nestedItems = (await runAst(node, schema, accountability, { knex, nested: true })) as Item[] | null;
					
					if (nestedItems) {
						// 将嵌套数据合并到父数据中
						items = mergeWithParentItems(schema, nestedItems, items!, nestedNode, fieldAllowed)!;
					}
					
					// 检查是否还有更多数据
					if (!nestedItems || nestedItems.length < (env['RELATIONAL_BATCH_SIZE'] as number)) {
						hasMore = false;
					}
					
					batchCount++;
				}
			} else {
				// 处理其他类型的嵌套关系（多对一、任意对一）
				const node = merge({}, nestedNode, {
					query: { limit: -1 },
				});
				
				nestedItems = (await runAst(node, schema, accountability, { knex, nested: true })) as Item[] | null;
				
				if (nestedItems) {
					// 将嵌套数据合并到父数据中
					items = mergeWithParentItems(schema, nestedItems, items!, nestedNode, true)!;
				}
			}
		}
		
		// 7. 移除临时字段
		if (options?.nested !== true && options?.stripNonRequested !== false) {
			items = removeTemporaryFields(schema, items, originalAST, primaryKeyField);
		}
		
		return items;
	}
}
```

### 6.2 写入操作的数据库交互

写入操作（创建、更新、删除）直接使用 Knex.js 的查询构建器来执行数据库操作：

#### 6.2.1 创建操作

```typescript
// 在 createOne 方法中
const result = await trx
	.insert(payloadWithoutAliases)
	.into(this.collection)
	.returning(primaryKeyField, returningOptions)
	.then((result) => result[0]);
```

#### 6.2.2 更新操作

```typescript
// 在 updateMany 方法中
await trx(this.collection)
	.update(payloadWithTypeCasting)
	.whereIn(primaryKeyField, keys);
```

#### 6.2.3 删除操作

```typescript
// 在 deleteMany 方法中
await trx(this.collection)
	.whereIn(primaryKeyField, keysAfterHooks)
	.delete();
```

### 6.3 Knex.js 数据库适配器

Directus 使用 Knex.js 作为数据库查询构建器，支持多种数据库：

- PostgreSQL
- MySQL / MariaDB
- SQLite
- MS SQL Server
- OracleDB
- CockroachDB

数据库连接通过 `getDatabase` 函数获取：

```typescript
import getDatabase from '../database/index.js';

const knex = getDatabase();
```

`getDatabase` 函数根据配置返回相应的 Knex 实例，配置信息来自环境变量或配置文件。

### 6.4 数据转换与验证

服务层使用 `PayloadService` 来处理数据的转换和验证：

#### 6.4.1 PayloadService 的主要功能

| 方法 | 功能 |
|------|------|
| `processM2O` | 处理多对一关系 |
| `processA2O` | 处理任意对一关系 |
| `processO2M` | 处理一对多关系 |
| `processValues` | 处理数据类型转换 |
| `prepareDelta` | 准备修订记录的差异数据 |

#### 6.4.2 数据类型转换

`processValues` 方法负责处理数据的类型转换，确保数据符合数据库的要求：

```typescript
// 读取操作的数据转换
let items: null | Item | Item[] = await payloadService.processValues(
	'read',
	rawItems,
	aliasMap,
	query.aggregate ?? {},
);

// 写入操作的数据转换
const payloadWithTypeCasting = await payloadService.processValues('create', payloadWithoutAliases);
```

类型转换包括：
- 日期/时间格式转换
- JSON 数据的序列化/反序列化
- UUID 格式标准化
- 数值类型转换
- 布尔值处理

---

## 7. REST 和 GraphQL 接口的对比

### 7.1 架构对比

| 特性 | REST API | GraphQL API |
|------|----------|-------------|
| **路由定义** | 每个端点单独定义路由 | 只有两个主要路由（`/` 和 `/system`） |
| **参数解析** | 从 URL、查询字符串、请求体中提取 | 从 GraphQL 查询文档中解析 |
| **查询格式** | 直接使用 Directus 查询对象 | 需要将 GraphQL 查询转换为 Directus 查询 |
| **服务调用** | 控制器直接调用 `ItemsService` | 解析器通过 `getService` 获取服务实例 |
| **响应格式** | 固定格式（`{ data: ..., meta: ... }`） | 灵活格式，根据查询而定 |

### 7.2 处理流程对比

#### REST API 流程

```
1. 客户端发送 HTTP 请求（GET/POST/PATCH/DELETE）
2. Express 路由匹配到相应的控制器方法
3. 中间件处理（collectionExists, validateBatch 等）
4. 控制器创建 ItemsService 实例
5. 控制器调用相应的服务方法（createOne/readByQuery 等）
6. 服务层执行权限校验、数据操作、事件钩子
7. 控制器准备响应数据到 res.locals['payload']
8. respond 中间件格式化响应并返回
```

#### GraphQL API 流程

```
1. 客户端发送 HTTP 请求（POST 或 GET）
2. Express 路由匹配到 GraphQL 控制器
3. parseGraphQL 中间件解析 GraphQL 查询文档
4. 控制器创建 GraphQLService 实例
5. GraphQLService.execute 方法执行查询
6. GraphQL 引擎解析查询并调用相应的解析器
7. 解析器（resolveQuery/resolveMutation）将 GraphQL 查询转换为 Directus 查询
8. 解析器通过 getService 获取服务实例
9. 解析器调用相应的服务方法
10. 服务层执行权限校验、数据操作、事件钩子
11. 解析器处理返回结果，转换为 GraphQL 格式
12. 控制器准备响应数据到 res.locals['payload']
13. respond 中间件格式化响应并返回
```

### 7.3 特殊处理环节对比

#### REST API 的特殊处理

1. **HTTP 方法映射**：
   - POST → 创建
   - GET → 读取
   - PATCH → 更新
   - DELETE → 删除

2. **URL 参数处理**：
   - `:collection` 从 URL 路径中提取集合名称
   - `:pk` 从 URL 路径中提取主键

3. **查询参数处理**：
   - 使用 `req.sanitizedQuery` 处理查询参数
   - 中间件 `sanitizeQuery` 负责清理和验证

4. **响应格式**：
   - 固定格式：`{ data: ..., meta: ... }`
   - `meta` 包含分页、计数等元信息

#### GraphQL API 的特殊处理

1. **查询模式**：
   - 普通查询：`collection`
   - 按 ID 查询：`collection_by_id`
   - 聚合查询：`collection_aggregated`
   - 版本查询：`collection_by_version`

2. **突变模式**：
   - 创建单个：`create_collection_item`
   - 创建多个：`create_collection_items`
   - 更新单个：`update_collection_item`
   - 更新多个：`update_collection_items`
   - 批量更新：`update_collection_batch`
   - 删除单个：`delete_collection_item`
   - 删除多个：`delete_collection_items`

3. **查询转换**：
   - `parseArgs`：解析 GraphQL 字段参数
   - `getQuery`：将 GraphQL 选择集转换为 Directus 查询
   - `getAggregateQuery`：处理聚合查询
   - `replaceFragmentsInSelections`：处理片段

4. **单例集合处理**：
   - 读取：调用 `readSingleton`
   - 更新：调用 `upsertSingleton`

### 7.4 共同之处

尽管 REST 和 GraphQL 接口有很多不同，但它们在核心层面是完全一致的：

1. **统一的服务层**：
   - 两种接口都使用相同的 `ItemsService`
   - 都调用相同的服务方法（`createOne`、`readByQuery`、`updateMany`、`deleteMany` 等）

2. **统一的权限校验**：
   - 都使用 `validateAccess`、`processAst`、`processPayload` 进行权限校验
   - 权限校验逻辑完全相同

3. **统一的数据库交互**：
   - 都通过相同的 AST 处理流程（读取操作）
   - 都使用相同的 Knex.js 查询构建器（写入操作）

4. **统一的上下文传递**：
   - 都传递 `accountability`（用户身份信息）
   - 都传递 `schema`（数据库模式信息）
   - 都传递 `knex`（数据库连接实例）

5. **统一的事件钩子**：
   - 都触发相同的 filter 钩子和 action 钩子
   - 扩展可以通过钩子介入任何数据操作

---

## 8. 总结与关键设计要点

### 8.1 核心架构设计

Directus 的数据读写架构采用了清晰的分层设计，核心设计要点包括：

1. **接口与业务逻辑分离**：
   - REST 和 GraphQL 接口只负责协议相关的处理
   - 核心业务逻辑完全封装在服务层（`ItemsService`）

2. **统一的服务层**：
   - `ItemsService` 是所有数据操作的唯一入口
   - 提供了完整的 CRUD 操作方法
   - 集成了权限校验、事务管理、事件钩子等机制

3. **基于 AST 的读取操作**：
   - 使用抽象语法树（AST）作为中间表示
   - `getAstFromQuery` 将查询转换为 AST
   - `processAst` 在 AST 层面应用权限限制
   - `runAst` 执行 AST 生成实际的数据库查询

4. **统一的权限校验**：
   - 三种核心权限校验函数：`validateAccess`、`processAst`、`processPayload`
   - 在服务层的固定位置触发
   - 管理员用户自动跳过权限检查

5. **灵活的事件钩子系统**：
   - Filter 钩子：在操作执行前修改数据或查询
   - Action 钩子：在操作执行后触发副作用
   - 支持集合级和全局级别的钩子

### 8.2 两种接口的差异与统一

| 维度 | REST API | GraphQL API | 统一处理 |
|------|----------|-------------|----------|
| **路由定义** | 每个端点单独定义 | 单一入口 | - |
| **参数解析** | URL/查询字符串/请求体 | GraphQL 文档 | - |
| **查询转换** | 无需转换 | 需要转换为 Directus 查询 | - |
| **服务调用** | 直接调用 `ItemsService` | 通过 `getService` 获取实例 | ✅ 统一服务层 |
| **权限校验** | `validateAccess`/`processAst`/`processPayload` | 相同的函数 | ✅ 统一权限 |
| **数据库交互** | AST 读取 / Knex 写入 | 相同的流程 | ✅ 统一数据库 |
| **事件钩子** | filter/action 钩子 | 相同的钩子 | ✅ 统一钩子 |

### 8.3 关键代码位置

#### 接口层
- **REST 控制器**：`api/src/controllers/items.ts`
- **GraphQL 控制器**：`api/src/controllers/graphql.ts`
- **GraphQL 解析器**：`api/src/services/graphql/resolvers/`

#### 服务层
- **核心服务**：`api/src/services/items.ts`
- **GraphQL 服务**：`api/src/services/graphql/index.ts`

#### 权限校验
- **validateAccess**：`api/src/permissions/modules/validate-access/validate-access.ts`
- **processAst**：`api/src/permissions/modules/process-ast/process-ast.ts`

#### 数据库交互
- **runAst**：`api/src/database/run-ast/run-ast.ts`
- **getAstFromQuery**：`api/src/database/get-ast-from-query/get-ast-from-query.ts`

### 8.4 设计优势

1. **可维护性**：
   - 业务逻辑集中在服务层，易于维护和修改
   - 接口层只负责协议处理，逻辑清晰

2. **一致性**：
   - 两种接口的行为完全一致
   - 权限校验、数据验证等机制统一

3. **可扩展性**：
   - 通过事件钩子系统支持扩展
   - 服务层可以被其他接口或内部组件复用

4. **灵活性**：
   - 支持多种数据库（通过 Knex.js）
   - 支持多种接口协议（REST、GraphQL）

5. **安全性**：
   - 权限校验在服务层强制执行
   - 无法绕过服务层直接操作数据库

### 8.5 数据读写完整流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              客户端请求                                           │
│  ┌─────────────────────┐                    ┌─────────────────────────────────┐ │
│  │    REST 请求        │                    │        GraphQL 请求              │ │
│  │  GET/POST/PATCH/    │                    │    POST /graphql 或 GET         │ │
│  │  DELETE             │                    │                                 │ │
│  └──────────┬──────────┘                    └──────────────┬──────────────────┘ │
│             │                                               │                     │
└─────────────┼───────────────────────────────────────────────┼─────────────────────┘
              │                                               │
              ▼                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              接口层 (Interface Layer)                            │
│  ┌─────────────────────┐                    ┌─────────────────────────────────┐ │
│  │  REST 控制器        │                    │     GraphQL 控制器               │ │
│  │  - 路由匹配         │                    │  - parseGraphQL 中间件          │ │
│  │  - 中间件处理       │                    │  - GraphQLService.execute        │ │
│  │  - 参数提取         │                    │  - 解析器 (Resolvers)           │ │
│  └──────────┬──────────┘                    │    - resolveQuery               │ │
│             │                                │    - resolveMutation            │ │
│             │                                │    - 查询格式转换                │ │
│             │                                └──────────────┬──────────────────┘ │
│             │                                               │                     │
│             │              共同点：                          │                     │
│             │  - 创建服务实例                                │                     │
│             │  - 传递 accountability 和 schema              │                     │
│             │  - 调用服务层方法                             │                     │
│             └───────────────────────────┬───────────────────┘                     │
│                                         │                                           │
└─────────────────────────────────────────┼───────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           服务层 (Service Layer) - ItemsService                   │
│                                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         操作类型分发                                       │   │
│  │                                                                           │   │
│  │   读取操作              创建操作              更新操作              删除操作    │   │
│  │   ┌───────┐            ┌───────┐            ┌───────┐            ┌───────┐ │   │
│  │   │readBy │            │create │            │update │            │delete │ │   │
│  │   │Query  │            │One/   │            │Many/  │            │Many/  │ │   │
│  │   │readOne│            │Many   │            │One    │            │One    │ │   │
│  │   └───┬───┘            └───┬───┘            └───┬───┘            └───┬───┘ │   │
│  └───────┼────────────────────┼────────────────────┼────────────────────┼───────┘   │
│          │                    │                    │                    │           │
│          ▼                    ▼                    ▼                    ▼           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         1. Filter 钩子 (可选)                                │   │
│  │  - items.query / items.create / items.update / items.delete                 │   │
│  │  - 允许在操作执行前修改数据或查询                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         2. 权限校验 (核心)                                   │   │
│  │                                                                               │   │
│  │   读取操作: processAst                                                        │   │
│  │   - 从 AST 提取字段映射                                                       │   │
│  │   - 验证字段存在性                                                            │   │
│  │   - 验证字段权限                                                              │   │
│  │   - 注入权限条件到 AST                                                        │   │
│  │                                                                               │   │
│  │   写入操作: validateAccess + processPayload                                  │   │
│  │   - validateAccess: 验证集合/项目级权限                                      │   │
│  │   - processPayload: 处理数据权限和预设值                                     │   │
│  │                                                                               │   │
│  │   管理员用户 (accountability.admin === true): 跳过权限检查                   │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         3. 事务管理                                           │   │
│  │  - 使用 transaction() 函数包装所有写入操作                                   │   │
│  │  - 确保操作的原子性                                                           │   │
│  │  - 失败时自动回滚                                                             │   │
│  │  - 成功时自动提交                                                             │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         4. 数据处理 (PayloadService)                         │   │
│  │  - processM2O: 处理多对一关系                                                │   │
│  │  - processA2O: 处理任意对一关系                                              │   │
│  │  - processO2M: 处理一对多关系                                                │   │
│  │  - processValues: 处理数据类型转换                                           │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         5. 数据库交互                                         │   │
│  │                                                                               │   │
│  │   读取操作:                                                                   │   │
│  │   - getAstFromQuery: 将查询转换为 AST                                        │   │
│  │   - processAst: 应用权限限制 (已在步骤 2 执行)                               │   │
│  │   - runAst: 执行 AST，生成 Knex 查询                                        │   │
│  │   - 支持嵌套关系的批量查询                                                    │   │
│  │                                                                               │   │
│  │   写入操作:                                                                   │   │
│  │   - Knex.insert(): 插入数据                                                  │   │
│  │   - Knex.update(): 更新数据                                                  │   │
│  │   - Knex.delete(): 删除数据                                                  │   │
│  │   - 支持多种数据库 (PostgreSQL, MySQL, SQLite 等)                            │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         6. 问责制追踪 (可选)                                  │   │
│  │  - 当 accountability 存在且集合启用问责制时执行                               │   │
│  │  - 创建活动记录 (ActivityService.createOne)                                  │   │
│  │  - 创建修订记录 (RevisionsService.createOne) - 当 accountability === 'all'  │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         7. 缓存管理 (可选)                                     │   │
│  │  - 写入操作后清理缓存                                                         │   │
│  │  - shouldClearCache() 决定是否需要清理                                        │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                          │                                           │
│                                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────────────────────┐   │
│  │                         8. Action 钩子 (可选)                                 │   │
│  │  - items.create / items.read / items.update / items.delete                   │   │
│  │  - 操作成功执行后触发                                                         │   │
│  │  - 用于执行副作用操作                                                         │   │
│  └─────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                   │
└───────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              接口层响应处理                                         │
│  ┌─────────────────────┐                    ┌─────────────────────────────────┐ │
│  │  REST 响应          │                    │     GraphQL 响应                 │ │
│  │  - respond 中间件   │                    │  - 解析器处理返回结果            │ │
│  │  - 固定格式:         │                    │  - 转换为 GraphQL 格式           │ │
│  │    { data, meta }   │                    │  - respond 中间件统一处理        │ │
│  └──────────┬──────────┘                    └──────────────┬──────────────────┘ │
│             │                                               │                     │
└─────────────┼───────────────────────────────────────────────┼─────────────────────┘
              │                                               │
              ▼                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              客户端响应                                           │
│  ┌─────────────────────┐                    ┌─────────────────────────────────┐ │
│  │    REST 响应        │                    │        GraphQL 响应              │ │
│  │  {                  │                    │    {                            │ │
│  │    data: [...],     │                    │      data: { ... },             │ │
│  │    meta: {          │                    │      errors: [...] // 可选      │ │
│  │      total: 100     │                    │    }                            │ │
│  │    }                │                    │                                 │ │
│  │  }                  │                    │                                 │ │
│  └─────────────────────┘                    └─────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────────┘
```

### 8.6 关键技术点总结

1. **服务层的核心地位**：
   - `ItemsService` 是所有数据操作的唯一入口
   - 接口层只负责协议转换，不包含业务逻辑
   - 这种设计确保了行为的一致性和代码的可维护性

2. **权限校验的统一性**：
   - 三种核心权限函数在服务层的固定位置触发
   - 无论是 REST 还是 GraphQL，都无法绕过权限检查
   - 管理员用户的特殊处理集中在权限函数内部

3. **AST 的抽象作用**：
   - 读取操作使用 AST 作为中间表示
   - 权限校验在 AST 层面进行，不依赖具体的查询格式
   - `runAst` 负责将 AST 转换为具体的数据库查询

4. **事务的完整性**：
   - 所有写入操作都包装在事务中
   - 包括关系处理、活动记录、修订记录等
   - 确保数据的一致性和完整性

5. **事件钩子的灵活性**：
   - Filter 钩子允许在操作前修改数据
   - Action 钩子允许在操作后执行副作用
   - 支持集合级和全局级别的钩子

这种架构设计使得 Directus 能够同时支持 REST 和 GraphQL 两种接口，同时保持核心业务逻辑的一致性和可维护性。无论是通过哪种接口访问数据，都会经过相同的权限校验、数据处理和数据库交互流程。