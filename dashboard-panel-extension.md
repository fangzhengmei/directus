# Directus 仪表盘面板扩展机制分析

## 概述

本文档详细分析 Directus 中用户配置自定义仪表盘面板或模块后，前端扩展、数据查询和权限校验的协作机制，以及面板的数据源和权限如何绑定保存并重新渲染。

---

## 1. 前端扩展机制

### 1.1 扩展加载流程

**入口文件**：`app/src/extensions.ts`

扩展加载分为两个阶段：

1. **加载扩展** (`loadExtensions`)：
   - 开发环境：从 `@directus-extensions` 动态导入
   - 生产环境：从 `extensions/sources/index.js` 加载
   - 支持的扩展类型：interfaces, displays, layouts, modules, panels, operations, themes

2. **注册扩展** (`registerExtensions`)：
   - 合并内部扩展和自定义扩展
   - 调用各类型的注册函数（如 `registerPanels`）
   - 监听语言变化，自动翻译扩展名称

**核心代码**：
```typescript
// app/src/extensions.ts:31-42
export async function loadExtensions(): Promise<void> {
    try {
        customExtensions = import.meta.env.DEV
            ? await import(/* @vite-ignore */ '@directus-extensions')
            : await import(/* @vite-ignore */ `${getRootPath()}extensions/sources/index.js`);
    } catch (err: any) {
        console.warn(`Couldn't load extensions`);
        console.warn(err);
    }
}

// app/src/extensions.ts:44-95
export function registerExtensions(app: App): void {
    const interfaces = getInternalInterfaces();
    const displays = getInternalDisplays();
    const layouts = getInternalLayouts();
    const modules = getInternalModules();
    const panels = getInternalPanels();
    const operations = getInternalOperations();
    const themes = [];

    if (customExtensions !== null) {
        interfaces.push(...customExtensions.interfaces);
        displays.push(...customExtensions.displays);
        layouts.push(...customExtensions.layouts);
        modules.push(...customExtensions.modules);
        panels.push(...customExtensions.panels);
        operations.push(...customExtensions.operations);
        themes.push(...customExtensions.themes);
    }

    registerPanels(panels, app);
    // ... 注册其他扩展类型
}
```

### 1.2 面板注册机制

**入口文件**：`app/src/panels/index.ts`

面板注册流程：

1. **获取内部面板** (`getInternalPanels`)：
   - 使用 `import.meta.glob` 动态导入所有面板
   - 按 id 排序

2. **注册面板** (`registerPanels`)：
   - 将面板组件注册为 Vue 组件，命名为 `panel-{id}`
   - 如果面板有自定义选项组件，注册为 `panel-options-{id}`

**核心代码**：
```typescript
// app/src/panels/index.ts:5-19
export function getInternalPanels(): PanelConfig[] {
    const panels = import.meta.glob<PanelConfig>('./*/index.ts', { import: 'default', eager: true });
    return sortBy(Object.values(panels), 'id');
}

export function registerPanels(panels: PanelConfig[], app: App): void {
    for (const panel of panels) {
        app.component(`panel-${panel.id}`, panel.component);

        if (typeof panel.options !== 'function' && Array.isArray(panel.options) === false && panel.options !== null) {
            app.component(`panel-options-${panel.id}`, panel.options);
        }
    }
}
```

### 1.3 面板定义结构

**示例**：`app/src/panels/metric/index.ts`

一个完整的面板定义包含以下属性：

| 属性 | 类型 | 说明 |
|------|------|------|
| `id` | string | 面板唯一标识 |
| `name` | string | 面板名称（支持 i18n） |
| `description` | string | 面板描述 |
| `icon` | string | 面板图标 |
| `preview` | string | 预览 SVG 或图片 |
| `component` | Component | 面板渲染组件 |
| `query` | Function | 生成数据查询的函数 |
| `options` | Function \| Array | 配置选项定义 |
| `minWidth` | number | 最小宽度 |
| `minHeight` | number | 最小高度 |
| `variable` | boolean | 是否作为变量面板 |
| `skipUndefinedKeys` | boolean | 选项处理时是否跳过 undefined 键 |

**核心代码**：
```typescript
// app/src/panels/metric/index.ts:9-565
export default definePanel({
    id: 'metric',
    name: '$t:panels.metric.name',
    description: '$t:panels.metric.description',
    icon: 'functions',
    preview: PreviewSVG,
    component: PanelMetric,
    query(options) {
        if (!options || !options.function) return;
        const collectionsStore = useCollectionsStore();
        const collectionInfo = collectionsStore.getCollection(options.collection);

        if (!collectionInfo) return;
        if (collectionInfo?.meta?.singleton) return;

        const isRawValue = ['first', 'last'].includes(options.function);
        const sort = options.sortField && `${options.function === 'last' ? '-' : ''}${options.sortField}`;

        const aggregate = isRawValue
            ? undefined
            : {
                    [options.function]: [options.field || '*'],
                };

        const panelQuery: PanelQuery = {
            collection: options.collection,
            query: {
                sort,
                limit: 1,
                fields: [options.field],
            },
        };

        if (options.filter && Object.keys(options.filter).length > 0) {
            panelQuery.query.filter = options.filter;
        }

        if (aggregate) {
            panelQuery.query.aggregate = aggregate;
            delete panelQuery.query.fields;
        }

        return panelQuery;
    },
    options: ({ options }) => {
        // 返回配置选项数组
        return [
            {
                field: 'collection',
                type: 'string',
                name: '$t:collection',
                meta: {
                    interface: 'system-collection',
                    // ...
                },
            },
            // ... 更多选项
        ];
    },
    minWidth: 6,
    minHeight: 2,
});
```

---

## 2. 数据查询机制

### 2.1 查询准备流程

**核心文件**：`app/src/stores/insights.ts`

数据查询的核心流程在 `useInsightsStore` 中实现：

1. **准备查询** (`prepareQuery`)：
   - 根据面板类型获取面板定义
   - 调用面板的 `query` 函数生成查询配置
   - 应用变量替换（处理 `{{ variable }}` 语法）

2. **加载面板数据** (`loadPanelData`)：
   - 为所有面板准备查询
   - 检查是否有关联变量未设置
   - 将查询转换为 GraphQL 格式
   - 执行查询并处理结果

**核心代码**：
```typescript
// app/src/stores/insights.ts:318-323
function prepareQuery(panel: Pick<Panel, 'options' | 'type'>) {
    const panelType = unref(panelTypes).find((panelType) => panelType.id === panel.type);
    return (
        panelType?.query?.(applyOptionsData(panel.options ?? {}, unref(variables), panelType.skipUndefinedKeys)) ?? null
    );
}

// app/src/stores/insights.ts:205-316
async function loadPanelData(
    panels: Pick<Panel, 'id' | 'options' | 'type'> | Pick<Panel, 'id' | 'options' | 'type'>[],
) {
    panels = toArray(panels);
    const queries = new Map();

    for (const panel of panels) {
        const req = prepareQuery(panel);
        if (!req) continue;

        if (hasEmptyRelation(panel)) {
            data.value[panel.id] = {};
            continue;
        }

        toArray(req).forEach(({ collection, query }, index) => {
            const key = getSimpleHash(panel.id + collection + JSON.stringify(query));
            queries.set(key, { panel: panel.id, collection, query, key, index, length: toArray(req).length });
        });
    }

    loading.value = uniq([...loading.value, ...Array.from(queries.values()).map(({ panel }) => panel)]);

    // 转换为 GraphQL 查询
    const gqlString = queryToGqlString(
        Array.from(queries.values())
            .filter(({ collection }) => isSystemCollection(collection) === false)
            .map(({ key, ...rest }) => ({ key: `query_${key}`, ...rest })),
    );

    const systemGqlString = queryToGqlString(
        Array.from(queries.values())
            .filter(({ collection }) => isSystemCollection(collection))
            .map(({ key, ...rest }) => ({
                key: `query_${key}`,
                ...rest,
            })),
    );

    try {
        const requests: Promise<AxiosResponse<any, any>>[] = [];
        if (gqlString) requests.push(api.post(`/graphql`, { query: gqlString }));
        if (systemGqlString) requests.push(api.post(`/graphql/system`, { query: systemGqlString }));

        const responses = await Promise.all(requests);
        const results: { [panel: string]: Item | Item[] } = {};

        for (const { data } of responses) {
            const result = mapKeys(data.data, (_, key) => key.substring('query_'.length));

            for (const [key, data] of Object.entries(result)) {
                const { panel, length } = queries.get(key);
                if (length === 1) results[panel] = data;
                else if (!results[panel]) results[panel] = [data];
                else results[panel]?.push(data);
            }
        }

        data.value = assign({}, data.value, results);
    } catch (error: any) {
        // 错误处理
    } finally {
        loading.value = pull(unref(loading), ...Array.from(queries.values()).map(({ panel }) => panel));
    }
}
```

### 2.2 变量替换机制

**工具函数**：`@directus/utils` 中的 `applyOptionsData`

面板选项支持使用 `{{ variableName }}` 语法引用变量。变量来源包括：

1. **变量面板**（如 `variable` 和 `relational-variable` 类型）：
   - 当面板定义 `variable: true` 时，其值会被存入 `variables` store
   - 默认值从面板选项的 `defaultValue` 获取

2. **变量使用**：
   - 其他面板的选项中可以使用 `{{ variableField }}` 引用变量
   - `applyOptionsData` 函数会替换这些变量占位符

**核心代码**：
```typescript
// app/src/stores/insights.ts:125-136
const variableDefaults: Record<string, any> = {};

panels.value.forEach((panel) => {
    const panelType = unref(panelTypes).find((panelType) => panelType.id === panel.type);

    if (panelType?.variable === true && panel.options?.field) {
        variableDefaults[panel.options.field] = panel.options?.defaultValue;
    }
});

variables.value = assign({}, variableDefaults, variables.value);

// app/src/stores/insights.ts:463-499
function setVariable(field: string, value: unknown) {
    const newVariables = assign({}, variables.value, { [field]: value });

    // 查找所有使用此变量的面板
    const regex = new RegExp(`{{\\s*?${escapeStringRegexp(field)}\\s*?}}`);

    const needReload = unref(panelsWithEdits).filter((panel) => {
        if (panel.id in unref(data) === false) return false;

        const optionsString = JSON.stringify(panel.options ?? {});
        const containsVariable = regex.test(optionsString);
        if (!containsVariable) return false;

        const panelType = unref(panelTypes).find((panelType) => panelType.id === panel.type);
        if (!panelType) return false;

        // 检查查询是否变化
        const oldQuery = panelType.query?.(
            applyOptionsData(panel.options ?? {}, unref(variables), panelType.skipUndefinedKeys),
        );

        const newQuery = panelType.query?.(
            applyOptionsData(panel.options ?? {}, unref(newVariables), panelType.skipUndefinedKeys),
        );

        return JSON.stringify(oldQuery) !== JSON.stringify(newQuery);
    });

    variables.value = newVariables;

    if (needReload.length > 0) {
        loadPanelData(needReload);
    }
}
```

### 2.3 GraphQL 查询执行

**后端服务**：`api/src/services/graphql/index.ts`

GraphQL 查询的执行流程：

1. **解析查询**：使用 `parseGraphQL` 中间件解析 GraphQL 查询
2. **创建服务**：创建 `GraphQLService` 实例，传入 `accountability` 对象
3. **执行查询**：调用 `execute` 方法执行查询
4. **权限校验**：在执行过程中通过 `accountability` 进行权限检查

**核心代码**：
```typescript
// api/src/controllers/graphql.ts:30-49
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
        return next();
    }),
    respond,
);

// api/src/services/graphql/index.ts:31-116
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

    async execute({ document, variables, operationName, contextValue }: GraphQLParams): Promise<FormattedExecutionResult> {
        const schema = await this.getSchema();

        const validationErrors = validate(schema, document, validationRules).map((validationError) =>
            addPathToValidationError(validationError),
        );

        if (validationErrors.length > 0) {
            throw new GraphQLValidationError({ errors: validationErrors });
        }

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

        // 处理结果和错误
        return formattedResult;
    }

    async read(collection: string, query: Query, id?: PrimaryKey): Promise<Partial<Item>> {
        const service = getService(collection, {
            knex: this.knex,
            accountability: this.accountability,
            schema: this.schema,
        });

        // 调用具体的服务方法读取数据
        // 服务内部会进行权限校验
    }
}
```

---

## 3. 权限校验机制

### 3.1 权限架构概述

**关键发现**：权限不是存储在面板配置中的，而是在查询时**动态应用**的。

Directus 的权限体系分为以下几个层次：

| 层级 | 说明 | 存储位置 |
|------|------|----------|
| 模块级权限 | 控制是否能访问某个模块 | `directus_permissions` 表 |
| 集合级权限 | 控制对某个集合的 CRUD 操作 | `directus_permissions` 表 |
| 项目级权限 | 控制对特定记录的访问 | 动态计算 |
| 字段级权限 | 控制可访问的字段 | 动态计算 |

**核心机制**：
- 面板配置（`directus_panels.options`）存储的是**数据源配置**（collection、filter、aggregation 等），不是权限
- 权限在**查询执行时**通过 `accountability` 对象动态应用
- 权限配置存储在 `directus_permissions` 表中，与面板配置分离

### 3.2 前端权限检查

**核心 Store**：`app/src/stores/permissions.ts`

前端权限检查主要通过 `usePermissionsStore` 实现：

1. **权限加载** (`hydrate`)：
   - 从 `/permissions/me` 接口获取当前用户权限
   - 解析动态变量字段（如 `$CURRENT_USER.xxx`）
   - 解析预设值

2. **权限检查**：
   - `getPermission(collection, action)`：获取具体权限配置
   - `hasPermission(collection, action)`：检查是否有权限
   - 管理员用户始终返回 `true`

**核心代码**：
```typescript
// app/src/stores/permissions.ts:15-76
export const usePermissionsStore = defineStore({
    id: 'permissionsStore',
    state: () => ({
        permissions: {} as CollectionAccess,
    }),
    actions: {
        async hydrate() {
            const userStore = useUserStore();
            const response = await api.get('/permissions/me');

            const fields = getNestedDynamicVariableFields(response.data.data);
            if (fields.length > 0) {
                await userStore.hydrateAdditionalFields(fields);
            }

            this.permissions = mapValues(
                response.data.data as CollectionAccess,
                (collectionPermission: CollectionPermission) => {
                    Object.values(collectionPermission).forEach((actionPermission) => {
                        if (actionPermission.presets) {
                            actionPermission.presets = parsePreset(actionPermission.presets);
                        }
                    });
                    return collectionPermission;
                },
            ) as CollectionAccess;
        },
        getPermission(collection: string, action: PermissionsAction) {
            return this.permissions[collection]?.[action] ?? null;
        },
        hasPermission(collection: string, action: PermissionsAction) {
            const userStore = useUserStore();
            if (userStore.isAdmin) return true;
            return (this.getPermission(collection, action)?.access ?? 'none') !== 'none';
        },
    },
});
```

### 3.3 后端权限校验核心模块

**关键模块**：`api/src/permissions/modules/`

后端权限校验通过三个核心模块实现：

| 模块 | 函数 | 作用 | 触发时机 |
|------|------|------|----------|
| **processAst** | `processAst()` | 在查询 AST 中注入权限过滤条件 | **读操作** (read) |
| **validateAccess** | `validateAccess()` | 验证用户对特定记录的访问权限 | **更新/删除**操作 |
| **processPayload** | `processPayload()` | 在创建/更新时应用权限预设 | **创建/更新**操作 |

#### 3.3.1 processAst - 读操作权限注入

**作用**：在查询执行前，将权限条件注入到查询 AST 中

**流程**：
1. 解析用户的权限配置（`directus_permissions` 表）
2. 提取权限中的 `permissions` 过滤条件
3. 将过滤条件合并到用户的查询中
4. 确保用户只能访问有权限的数据

**在 ItemsService 中的调用**：
```typescript
// api/src/services/items.ts:499-539
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
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

    // 关键：在执行查询前注入权限条件
    ast = await processAst(
        { ast, action: 'read', accountability: this.accountability },
        { knex: this.knex, schema: this.schema },
    );

    const records = await runAst(ast, this.schema, this.accountability, {
        knex: this.knex,
        stripNonRequested: opts?.stripNonRequested !== undefined ? opts.stripNonRequested : true,
    });

    // ...
}
```

**processAst 核心逻辑**（简化）：
```typescript
// api/src/permissions/modules/process-ast/process-ast.ts
export async function processAst(
    input: { ast: QueryAST; action: PermissionsAction; accountability: Accountability | null },
    context: { knex: Knex; schema: SchemaOverview },
): Promise<QueryAST> {
    if (!input.accountability) return input.ast;

    // 1. 检查用户是否有该集合的权限
    const collectionAccess = fetchAccountabilityCollectionAccess(...);
    
    if (collectionAccess.access === 'none') {
        throw new ForbiddenError();
    }

    // 2. 如果是 'partial' 权限，需要注入权限过滤条件
    if (collectionAccess.access === 'partial') {
        // 从权限配置中提取过滤条件
        const permissionFilters = collectionAccess.permissions;
        
        // 将权限过滤条件合并到查询 AST 中
        input.ast.query.filter = {
            _and: [
                input.ast.query.filter, // 用户查询的 filter
                permissionFilters,       // 权限配置的 filter
            ].filter(Boolean),
        };
    }

    // 3. 字段级权限检查
    // 检查请求的字段是否在允许的字段列表中
    const allowedFields = fetchAllowedFields(...);
    const requestedFields = extractFieldsFromQuery(input.ast);
    
    for (const field of requestedFields) {
        if (!allowedFields.includes(field)) {
            throw new ForbiddenError(`You don't have permission to access the "${field}" field`);
        }
    }

    return input.ast;
}
```

#### 3.3.2 validateAccess - 写操作权限验证

**作用**：在更新/删除操作前，验证用户对特定记录的访问权限

**流程**：
1. 检查用户是否有该集合的 `update` 或 `delete` 权限
2. 如果是 'partial' 权限，需要验证目标记录是否符合权限条件
3. 确保用户只能修改有权限的记录

**在 ItemsService 中的调用**：
```typescript
// api/src/services/items.ts:748-766
async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey[]> {
    // ...

    keys.sort();

    if (this.accountability) {
        // 关键：在更新前验证访问权限
        await validateAccess(
            {
                accountability: this.accountability,
                action: 'update',
                collection: this.collection,
                primaryKeys: keys,
            },
            {
                knex: this.knex,
                schema: this.schema,
            },
        );
    }

    // ... 执行更新
}

// api/src/services/items.ts:1094-1110
async deleteMany(keys: PrimaryKey[], opts?: MutationOptions): Promise<PrimaryKey[]> {
    // ...

    keysAfterHooks = uniq(keysAfterHooks);

    if (this.accountability) {
        // 关键：在删除前验证访问权限
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

    // ... 执行删除
}
```

**validateAccess 核心逻辑**（简化）：
```typescript
// api/src/permissions/modules/validate-access/validate-access.ts
export async function validateAccess(
    input: {
        accountability: Accountability;
        action: 'create' | 'read' | 'update' | 'delete' | 'share';
        collection: string;
        primaryKeys?: PrimaryKey[];
    },
    context: { knex: Knex; schema: SchemaOverview },
): Promise<void> {
    const { accountability, action, collection, primaryKeys } = input;

    // 1. 检查集合级权限
    const collectionAccess = fetchAccountabilityCollectionAccess(...);

    if (collectionAccess.access === 'none') {
        throw new ForbiddenError(`You don't have permission to ${action} ${collection}`);
    }

    if (collectionAccess.access === 'full') {
        // 完全权限，无需进一步检查
        return;
    }

    // 2. partial 权限：需要验证具体记录
    if (primaryKeys && primaryKeys.length > 0) {
        // 从权限配置中获取过滤条件
        const permissionFilters = collectionAccess.permissions;

        // 构建查询：检查这些主键的记录是否符合权限条件
        const query = context.knex(collection)
            .whereIn('id', primaryKeys)
            .andWhere(permissionFilters); // 应用权限过滤条件

        const validRecords = await query.select('id');
        const validIds = validRecords.map(r => r.id);

        // 检查是否所有请求的记录都有效
        const invalidIds = difference(primaryKeys, validIds);
        
        if (invalidIds.length > 0) {
            throw new ForbiddenError(
                `You don't have permission to ${action} the following records: ${invalidIds.join(', ')}`
            );
        }
    }
}
```

#### 3.3.3 processPayload - 创建/更新时的权限预设

**作用**：在创建/更新操作时，自动应用权限配置中的 `presets`

**流程**：
1. 检查用户是否有该集合的 `create` 或 `update` 权限
2. 应用权限配置中的 `presets`（预设值）
3. 限制可修改的字段（字段级权限）

**在 ItemsService 中的调用**：
```typescript
// api/src/services/items.ts:172-186
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    // ...

    // 在事务中执行
    const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
        // ... filter hooks ...

        // 关键：应用权限预设
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

        // ... 执行创建
    });

    // ...
}

// api/src/services/items.ts:768-782
async updateMany(keys: PrimaryKey[], data: Partial<Item>, opts?: MutationOptions): Promise<PrimaryKey[]> {
    // ... validateAccess 之后 ...

    // 关键：应用权限预设
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

    // ... 执行更新
}
```

**processPayload 核心逻辑**（简化）：
```typescript
// api/src/permissions/modules/process-payload/process-payload.ts
export async function processPayload(
    input: {
        accountability: Accountability;
        action: 'create' | 'update';
        collection: string;
        payload: AnyItem;
        nested?: string[];
    },
    context: { knex: Knex; schema: SchemaOverview },
): Promise<AnyItem> {
    const { accountability, action, collection, payload } = input;

    // 1. 检查集合级权限
    const collectionAccess = fetchAccountabilityCollectionAccess(...);

    if (collectionAccess.access === 'none') {
        throw new ForbiddenError();
    }

    // 2. 字段级权限：检查 payload 中的字段是否允许修改
    const allowedFields = fetchAllowedFields(
        accountability,
        action,
        collection,
        context.schema,
    );

    const payloadFields = Object.keys(payload);
    const forbiddenFields = difference(payloadFields, allowedFields);

    if (forbiddenFields.length > 0) {
        throw new ForbiddenError(
            `You don't have permission to modify the following fields: ${forbiddenFields.join(', ')}`
        );
    }

    // 3. 应用权限预设 (presets)
    const presets = collectionAccess.presets;
    let processedPayload = { ...payload };

    if (presets && Object.keys(presets).length > 0) {
        // presets 中的值会强制覆盖或添加到 payload 中
        // 例如：presets = { status: 'draft', created_by: $CURRENT_USER }
        processedPayload = { ...processedPayload, ...presets };
    }

    // 4. 对于 'partial' 权限，可能需要额外的验证
    if (collectionAccess.access === 'partial' && action === 'create') {
        // 验证创建的 payload 是否符合权限条件
        // 例如：权限要求 category = 'news'，则 payload 中 category 必须是 'news'
        const validation = validatePayloadAgainstPermissions(
            processedPayload,
            collectionAccess.permissions,
        );

        if (!validation.valid) {
            throw new ForbiddenError(validation.reason);
        }
    }

    return processedPayload;
}
```

### 3.4 服务层权限应用

**核心服务**：`ItemsService`

`PanelsService` 和 `DashboardsService` 都继承自 `ItemsService`，自动继承所有权限校验逻辑：

```typescript
// api/src/services/panels.ts:4-7
export class PanelsService extends ItemsService {
    constructor(options: AbstractServiceOptions) {
        super('directus_panels', options);
    }
}

// api/src/services/dashboards.ts:4-7
export class DashboardsService extends ItemsService {
    constructor(options: AbstractServiceOptions) {
        super('directus_dashboards', options);
    }
}
```

**控制器中的使用模式**：
```typescript
// api/src/controllers/panels.ts:16-53
router.post(
    '/',
    asyncHandler(async (req, res, next) => {
        const service = new PanelsService({
            accountability: req.accountability,  // 关键：传递 accountability
            schema: req.schema,
        });

        // 调用服务方法时会自动应用权限校验
        if (Array.isArray(req.body)) {
            const keys = await service.createMany(req.body);  // 内部调用 processPayload
            savedKeys.push(...keys);
        } else {
            const key = await service.createOne(req.body);  // 内部调用 processPayload
            savedKeys.push(key);
        }
        // ...
    }),
    respond,
);
```

### 3.5 模块访问权限

**模块定义**：`app/src/modules/insights/index.ts`

模块级别通过 `preRegisterCheck` 进行权限检查，控制用户是否能访问仪表盘模块：

```typescript
// app/src/modules/insights/index.ts:42-49
preRegisterCheck(user, permissions) {
    const admin = user.admin_access;

    if (admin) return true;

    // 检查用户是否有 directus_dashboards 集合的 read 权限
    const access = permissions['directus_dashboards']?.['read']?.access;
    return access === 'partial' || access === 'full';
},
```

### 3.6 权限关系总结

#### 3.6.1 权限层级关系

```
┌─────────────────────────────────────────────────────────────────┐
│                         权限层级架构                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              模块级权限 (Module Level)                    │   │
│  │  ─────────────────────────────────────────────────────  │   │
│  │  控制：是否能访问仪表盘模块                                │   │
│  │  位置：modules/*/index.ts (preRegisterCheck)            │   │
│  │  检查：permissions['directus_dashboards']?.read         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           集合级权限 (Collection Level)                   │   │
│  │  ─────────────────────────────────────────────────────  │   │
│  │  控制：对集合的 CRUD 操作权限                              │   │
│  │  位置：directus_permissions 表                            │   │
│  │  影响：                                                    │   │
│  │    • directus_dashboards - 仪表盘配置的增删改查          │   │
│  │    • directus_panels - 面板配置的增删改查                │   │
│  │    • 业务集合 (如 articles, users) - 面板数据源的权限    │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           项目级权限 (Item Level)                         │   │
│  │  ─────────────────────────────────────────────────────  │   │
│  │  控制：对特定记录的访问权限                                │   │
│  │  位置：directus_permissions.permissions (过滤条件)        │   │
│  │  应用：                                                    │   │
│  │    • 读操作：processAst 注入 WHERE 条件                  │   │
│  │    • 写操作：validateAccess 验证记录是否符合条件         │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              ↓                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │           字段级权限 (Field Level)                        │   │
│  │  ─────────────────────────────────────────────────────  │   │
│  │  控制：可访问/修改的字段                                  │   │
│  │  位置：directus_permissions.fields                        │   │
│  │  应用：                                                    │   │
│  │    • 读操作：检查请求的字段是否在允许列表中              │   │
│  │    • 写操作：processPayload 过滤不允许的字段             │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 3.6.2 权限与面板的关系

| 权限类型 | 存储位置 | 应用时机 | 与面板的关系 |
|----------|----------|----------|--------------|
| **仪表盘配置权限** | `directus_permissions` (collection: `directus_dashboards`) | 仪表盘 CRUD 时 | 控制能否创建/修改/删除仪表盘 |
| **面板配置权限** | `directus_permissions` (collection: `directus_panels`) | 面板 CRUD 时 | 控制能否创建/修改/删除面板 |
| **数据源权限** | `directus_permissions` (collection: 业务集合，如 `articles`) | **面板数据查询时** | **动态应用**，控制面板能访问哪些数据 |
| **模块访问权限** | 模块定义中的 `preRegisterCheck` | 模块加载时 | 控制能否访问仪表盘模块 |

**关键点**：
- 面板配置（`directus_panels.options`）存储的是**数据源配置**（查询哪个集合、用什么过滤条件等）
- 权限配置（`directus_permissions`）与面板配置**分离存储**
- 数据查询时，权限条件会**动态注入**到查询中，与面板配置的过滤条件合并

---

## 4. 配置保存与重新渲染

### 4.1 面板数据模型

**数据库表**：`directus_panels`

面板配置存储在 `directus_panels` 表中，字段结构如下：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | uuid | 面板唯一标识 |
| `name` | string | 面板名称 |
| `icon` | string | 面板图标 |
| `color` | string | 面板颜色 |
| `note` | string | 面板说明 |
| `type` | string | 面板类型（如 'metric', 'list' 等） |
| `show_header` | boolean | 是否显示头部 |
| `position_x` | integer | X 轴位置 |
| `position_y` | integer | Y 轴位置 |
| `width` | integer | 宽度 |
| `height` | integer | 高度 |
| `options` | json | 面板特定配置选项（**数据源配置，不含权限**） |
| `dashboard` | uuid | 所属仪表盘 ID |
| `date_created` | timestamp | 创建时间 |
| `user_created` | uuid | 创建用户 |

**关键字段说明 - `options`**:

`options` 字段存储的是面板的**数据源配置**，不包含权限信息。例如 `metric` 类型面板的 `options` 可能包含：

```json
{
  "collection": "articles",
  "field": "views",
  "function": "sum",
  "filter": {
    "status": {
      "_eq": "published"
    }
  }
}
```

**字段定义**：
```yaml
# packages/system-data/src/fields/panels.yaml
table: directus_panels

fields:
  - field: id
    special:
      - uuid
  - field: name
  - field: icon
  - field: color
  - field: note
  - field: type
  - field: show_header
    special:
      - cast-boolean
  - field: position_x
  - field: position_y
  - field: width
  - field: height
  - field: options
    special:
      - cast-json
  - field: date_created
    special:
      - date-created
      - cast-timestamp
  - field: user_created
    special:
      - user-created
  - field: dashboard
```

### 4.2 暂存编辑机制

**核心 Store**：`app/src/stores/insights.ts`

为了提供更好的用户体验，Directus 使用暂存编辑机制：

1. **编辑状态** (`edits`)：
   - `create`：待创建的面板列表
   - `update`：待更新的面板列表
   - `delete`：待删除的面板 ID 列表

2. **计算属性** (`panelsWithEdits`)：
   - 合并原始面板数据和暂存编辑
   - 过滤已标记删除的面板
   - 应用更新的面板数据

**核心代码**：
```typescript
// app/src/stores/insights.ts:44-48
const edits = reactive<{ create: CreatePanel[]; update: Partial<Panel>[]; delete: string[] }>({
    create: [],
    update: [],
    delete: [],
});

// app/src/stores/insights.ts:61-76
const panelsWithEdits = computed(() => {
    return [
        ...unref(panels)
            .filter((panel) => edits.delete.includes(panel.id) === false)
            .map((panel) => {
                const updates = edits.update.find((updated) => updated.id === panel.id);

                if (updates) {
                    return assign({}, panel, updates);
                }

                return panel;
            }),
        ...edits.create,
    ];
});
```

### 4.3 编辑操作

**创建面板** (`stagePanelCreate`)：
```typescript
// app/src/stores/insights.ts:325-328
function stagePanelCreate(panel: CreatePanel) {
    edits.create.push(panel);
    loadPanelData(panel);  // 立即加载数据用于预览
}
```

**更新面板** (`stagePanelUpdate`)：
```typescript
// app/src/stores/insights.ts:330-387
function stagePanelUpdate({ id, edits: panelEdits }: { id: string; edits: Partial<Panel> }) {
    panelEdits = omitBy(panelEdits, isUndefined);

    const isNew = id.startsWith('_');
    const arr = isNew ? edits.create : edits.update;

    // 检查查询是否变化，决定是否重新加载数据
    let oldQuery;
    if ('options' in panelEdits) {
        const panel = unref(panelsWithEdits).find((panel) => panel.id === id);
        if (panel) {
            const panelType = unref(panelTypes).find((panelType) => panelType.id === panel.type)!;
            oldQuery = panelType.query?.(
                applyOptionsData(panel.options ?? {}, unref(variables), panelType.skipUndefinedKeys),
            );
        }
    }

    // 应用更新
    if (arr.map(({ id }) => id).includes(id)) {
        const updatedArr = arr.map((currentEdit) => {
            if (currentEdit.id === id) {
                return assign({}, currentEdit, panelEdits);
            }
            return currentEdit;
        });
        if (isNew) edits.create = updatedArr as CreatePanel[];
        else edits.update = updatedArr;
    } else {
        arr.push({ id, ...panelEdits });
    }

    // 如果查询变化，重新加载数据
    if ('options' in panelEdits) {
        const panel = unref(panelsWithEdits).find((panel) => panel.id === id)!;
        const panelType = unref(panelTypes).find((panelType) => panelType.id === panel.type)!;

        const newQuery = panelType.query?.(
            applyOptionsData(panelEdits.options ?? {}, unref(variables), panelType.skipUndefinedKeys),
        );

        if (JSON.stringify(oldQuery) !== JSON.stringify(newQuery)) loadPanelData(panel);
    }
}
```

**删除面板** (`stagePanelDelete`)：
```typescript
// app/src/stores/insights.ts:410-419
function stagePanelDelete(panelKey: string) {
    if (edits.create.some((created) => created.id === panelKey)) {
        edits.create = edits.create.filter((created) => created.id !== panelKey);
        return;
    }

    edits.update = edits.update.filter((updated) => updated.id !== panelKey);
    edits.delete.push(panelKey);
    data.value = omit(data.value, panelKey);
}
```

### 4.4 保存变更

**保存流程** (`saveChanges`)：

1. **收集请求**：
   - 创建操作：POST `/panels`
   - 更新操作：PATCH `/panels`
   - 删除操作：DELETE `/panels`

2. **执行请求**：并行执行所有请求

3. **刷新数据**：
   - 重新从服务器加载面板数据
   - 清除缓存的临时面板数据
   - 加载新创建面板的数据

4. **清除暂存**：清空 `edits` 对象

**核心代码**：
```typescript
// app/src/stores/insights.ts:421-461
async function saveChanges() {
    saving.value = true;

    try {
        const requests: Promise<AxiosResponse<any, any>>[] = [];

        if (edits.create.length > 0) {
            // 移除临时 ID
            requests.push(
                api.post(
                    `/panels`,
                    edits.create.map((create) => omit(create, 'id')),
                ),
            );
        }

        if (edits.update.length > 0) {
            requests.push(api.patch(`/panels`, edits.update));
        }

        if (edits.delete.length > 0) {
            requests.push(api.delete(`/panels`, { data: edits.delete }));
        }

        await Promise.all(requests);
        await hydrate();

        // 清除临时面板的缓存数据
        data.value = omit(data.value, ...edits.create.map(({ id }) => id));

        // 加载新创建面板的数据
        const panelsToLoad = unref(panelsWithEdits).filter(({ id }) => id in unref(data) === false);
        loadPanelData(panelsToLoad);

        clearEdits();
    } catch (error) {
        unexpectedError(error);
    } finally {
        saving.value = false;
    }
}
```

### 4.5 面板渲染流程

**视图文件**：`app/src/modules/insights/routes/dashboard.vue`

面板渲染流程：

1. **准备数据**：
   - 从 `insightsStore` 获取面板数据
   - 计算面板坐标和边界
   - 应用变量替换到面板选项

2. **动态组件渲染**：
   - 使用 `<component :is="panel-${type}">` 动态渲染面板
   - 传递面板选项、数据和其他 props
   - 错误边界处理

3. **状态管理**：
   - 加载状态显示
   - 错误状态处理
   - 无数据状态处理

**核心代码**：
```typescript
// app/src/modules/insights/routes/dashboard.vue:57-132
const tiles = computed<AppTile[]>(() => {
    const panels = insightsStore.getPanelsForDashboard(props.primaryKey);

    const panelsWithCoordinates = panels.map((panel) => ({
        ...panel,
        coordinates: [
            [panel.position_x!, panel.position_y!],
            [panel.position_x! + panel.width!, panel.position_y!],
            [panel.position_x! + panel.width!, panel.position_y! + panel.height!],
            [panel.position_x!, panel.position_y! + panel.height!],
        ] as [number, number][],
    }));

    const tiles: AppTile[] = panelsWithCoordinates
        .map((panel) => {
            // ... 计算边界相交等

            const panelType = unref(panelsInfo).find((panelType) => panelType.id === panel.type);

            const tile: AppTile = {
                id: panel.id,
                x: panel.position_x,
                y: panel.position_y,
                width: panel.width,
                height: panel.height,
                name: panel.name,
                icon: panel.icon ?? panelType?.icon,
                color: panel.color,
                note: panel.note,
                showHeader: panel.show_header === true,
                minWidth: panelType?.minWidth,
                minHeight: panelType?.minHeight,
                draggable: true,
                borderRadius: [!topLeftIntersects, !topRightIntersects, !bottomRightIntersects, !bottomLeftIntersects],
                data: {
                    options: applyOptionsData(panel.options ?? {}, unref(variables), panelType?.skipUndefinedKeys),
                    type: panel.type,
                },
            };

            return tile;
        })
        .filter((t) => t);

    return tiles;
});
```

**模板渲染**：
```vue
<!-- app/src/modules/insights/routes/dashboard.vue:282-324 -->
<template #default="{ tile, gridSize }">
    <VProgressCircular
        v-if="loading.includes(tile.id) && !data[tile.id]"
        :class="{ 'header-offset': tile.showHeader }"
        class="panel-loading"
        indeterminate
    />
    <div v-else class="panel-container" :class="{ loading: loading.includes(tile.id) }">
        <div v-if="errors[tile.id]" class="panel-error">
            <VIcon name="warning" />
            {{ $t('unexpected_error') }}
            <VError :error="errors[tile.id]" />
        </div>
        <div
            v-else-if="tile.id in data && isEmpty(data[tile.id])"
            class="panel-no-data type-note"
            :class="{ 'header-offset': tile.showHeader }"
        >
            {{ $t('no_data') }}
        </div>
        <VErrorBoundary v-else :name="`panel-${tile.data.type}`">
            <component
                :is="`panel-${tile.data.type}`"
                v-bind="tile.data.options"
                :id="tile.id"
                :dashboard="primaryKey"
                :show-header="tile.showHeader"
                :height="tile.height"
                :width="tile.width"
                :now="now"
                :data="data[tile.id]"
                :grid-size="gridSize"
            />
            <template #fallback="{ error }">
                <div class="panel-error">
                    <VIcon name="warning" />
                    {{ $t('unexpected_error') }}
                    <VError :error="error" />
                </div>
            </template>
        </VErrorBoundary>
    </div>
</template>
```

### 4.6 数据刷新机制

**刷新触发**：

1. **进入仪表盘时**：
   ```typescript
   // app/src/modules/insights/index.ts:22-26
   beforeEnter(to) {
       const store = useInsightsStore();
       store.refresh(to.params.primaryKey as string);
   },
   ```

2. **手动刷新**：通过侧边栏的刷新按钮

3. **自动刷新**：通过 `refreshInterval` 设置定时刷新

**刷新流程** (`refresh`)：
```typescript
// app/src/stores/insights.ts:166-184
async function refresh(dashboard: string) {
    const panelsForDashboard = unref(panels).filter((panel) => panel.dashboard === dashboard);

    await loadPanelData(panelsForDashboard);

    // 缓存管理：最多缓存 MAX_CACHE_SIZE 个仪表盘的数据
    if (lastLoaded.includes(dashboard) === false) {
        lastLoaded.push(dashboard);

        if (lastLoaded.length > MAX_CACHE_SIZE) {
            const removed = lastLoaded.shift();

            const removedPanels = unref(panels)
                .filter((panel) => panel.dashboard === removed)
                .map(({ id }) => id);

            data.value = omit(data.value, ...removedPanels);
        }
    }
}
```

---

## 5. 完整时序流程

### 5.1 权限配置存储说明

在开始时序分析之前，明确一个关键概念：

**面板配置与权限配置是分离存储的**：

| 配置类型 | 存储位置 | 内容 | 应用时机 |
|----------|----------|------|----------|
| **面板配置** | `directus_panels` 表 | 数据源、过滤条件、聚合方式等 | 面板渲染时读取 |
| **权限配置** | `directus_permissions` 表 | 集合权限、过滤条件、字段限制等 | **查询执行时动态应用** |

**关键点**：
- 面板的 `options.filter` 是**用户配置的过滤条件**，不是权限
- 权限配置中的 `permissions.filter` 是**系统强制的权限条件**
- 查询执行时，两者会通过 `processAst` 合并为最终的 WHERE 条件

### 5.2 用户编辑面板数据源配置时序

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                      阶段1: 用户编辑面板数据源配置                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────┐     ┌──────────────────────┐     ┌─────────────────────────────┐   │
│  │   用户界面    │     │  panel-configuration │     │      insightsStore          │   │
│  │              │     │         .vue         │     │                             │   │
│  └──────┬───────┘     └──────────┬───────────┘     └──────────────┬──────────────┘   │
│         │                        │                                  │                   │
│         │  选择面板类型(metric)   │                                  │                   │
│         │───────────────────────>│                                  │                   │
│         │                        │                                  │                   │
│         │  配置数据源：           │                                  │                   │
│         │  collection: articles  │                                  │                   │
│         │  field: views          │                                  │                   │
│         │  function: sum         │                                  │                   │
│         │  filter: status=active │                                  │                   │
│         │───────────────────────>│                                  │                   │
│         │                        │                                  │                   │
│         │                        │  stagePanelUpdate()             │                   │
│         │                        │────────────────────────────────>│                   │
│         │                        │                                  │                   │
│         │                        │                                  │  检查 options 变化  │
│         │                        │                                  │  调用 panel.query() │
│         │                        │                                  │  生成查询配置       │
│         │                        │                                  │                   │
│         │                        │                                  │  loadPanelData()  │
│         │                        │                                  │─────────────┐     │
│         │                        │                                  │             │     │
│         │                        │                                  │    ┌────────▼────┐│
│         │                        │                                  │    │  构建 GraphQL ││
│         │                        │                                  │    │  查询请求     ││
│         │                        │                                  │    └────────┬────┘│
│         │                        │                                  │             │     │
│         │                        │                                  │  POST /graphql│    │
│         │                        │                                  │─────────────>│     │
│         │                        │                                  │             │     │
│         │                        │                                  │  存储预览数据 │     │
│         │                        │                                  │  data[panel.id]│   │
│         │                        │                                  │<─────────────│     │
│         │                        │                                  │             │     │
│         │  显示预览数据           │                                  │             │     │
│         │<───────────────────────│                                  │             │     │
│         │                        │                                  │             │     │
└─────────┴────────────────────────┴──────────────────────────────────┴─────────────┘   │
                                                                                          │
  【关键】此时：
  - 面板配置暂存在 edits.update 中，未写入数据库
  - 数据查询用于预览，权限已在后端动态应用
  - 用户配置的 filter 与权限配置的 filter 已合并
                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 保存面板配置到数据库时序

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                      阶段2: 保存面板配置到数据库                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────┐     ┌──────────────────────┐     ┌─────────────────────────────┐   │
│  │   用户界面    │     │     dashboard.vue    │     │      insightsStore          │   │
│  └──────┬───────┘     └──────────┬───────────┘     └──────────────┬──────────────┘   │
│         │                        │                                  │                   │
│         │  点击"保存"按钮        │                                  │                   │
│         │───────────────────────>│                                  │                   │
│         │                        │                                  │                   │
│         │                        │  saveChanges()                  │                   │
│         │                        │────────────────────────────────>│                   │
│         │                        │                                  │                   │
│         │                        │                                  │  收集请求：        │
│         │                        │                                  │  • edits.create   │
│         │                        │                                  │  • edits.update   │
│         │                        │                                  │  • edits.delete   │
│         │                        │                                  │                   │
│         │                        │                                  │  发送请求到后端    │
│         │                        │                                  │                   │
└─────────┴────────────────────────┴──────────────────────────────────┴─────────────────┘ │
                                          ↓                                                   │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         后端：保存面板配置                                                  │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐   │
│  │  panelsController    │     │    PanelsService     │     │    ItemsService     │   │
│  └──────────┬───────────┘     └──────────┬───────────┘     └──────────┬──────────┘   │
│             │                              │                              │              │
│             │  POST /panels (create)      │                              │              │
│             │─────────────────────────────>│                              │              │
│             │                              │                              │              │
│             │                              │  创建 ItemsService 实例      │              │
│             │                              │  accountability: req.account │              │
│             │                              │─────────────────────────────>│              │
│             │                              │                              │              │
│             │                              │  createMany(req.body)        │              │
│             │                              │─────────────────────────────>│              │
│             │                              │                              │              │
│             │                              │                              │  权限校验：   │
│             │                              │                              │              │
│             │                              │                              │  1. 检查用户 │
│             │                              │                              │     是否有    │
│             │                              │                              │     directus_ │
│             │                              │                              │     panels的  │
│             │                              │                              │     create权限│
│             │                              │                              │              │
│             │                              │                              │  2. process- │
│             │                              │                              │     Payload   │
│             │                              │                              │     应用 presets│
│             │                              │                              │              │
│             │                              │                              │  3. 写入数据库│
│             │                              │                              │              │
│             │                              │                              │  INSERT INTO  │
│             │                              │                              │  directus_    │
│             │                              │                              │  panels       │
│             │                              │                              │              │
│             │                              │ 返回主键列表                  │              │
│             │                              │<─────────────────────────────│              │
│             │                              │                              │              │
│             │  返回创建的面板数据          │                              │              │
│             │<─────────────────────────────│                              │              │
│             │                              │                              │              │
└─────────────┴──────────────────────────────┴──────────────────────────────┴──────────────┘ │
                                                                                          │
  【关键】此时保存到数据库的是：
  - 面板配置（options）：包含 collection、filter、function 等
  - 不包含权限配置！权限在 directus_permissions 表中
                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 请求载荷详细分析

**创建面板的请求载荷**：

```typescript
// POST /panels
// 请求体示例
{
  "name": "文章浏览量统计",
  "type": "metric",
  "position_x": 0,
  "position_y": 0,
  "width": 12,
  "height": 2,
  "dashboard": "dashboard-uuid-123",
  "options": {
    // 数据源配置，不包含权限
    "collection": "articles",
    "field": "views",
    "function": "sum",
    "filter": {
      // 用户配置的过滤条件，不是权限
      "status": { "_eq": "published" }
    }
  }
}
```

**后端处理后的存储数据**（`directus_panels` 表）：

```sql
-- 插入到 directus_panels 表
INSERT INTO directus_panels (
  id,
  name,
  type,
  position_x,
  position_y,
  width,
  height,
  dashboard,
  options,  -- JSON 格式存储数据源配置
  user_created,
  date_created
) VALUES (
  'panel-uuid-456',
  '文章浏览量统计',
  'metric',
  0,
  0,
  12,
  2,
  'dashboard-uuid-123',
  -- options 字段存储的是数据源配置，不是权限
  '{"collection":"articles","field":"views","function":"sum","filter":{"status":{"_eq":"published"}}}',
  'user-uuid-789',
  NOW()
);
```

**权限配置存储**（`directus_permissions` 表）：

```sql
-- 权限配置与面板配置分离存储
INSERT INTO directus_permissions (
  id,
  role,
  collection,  -- 权限针对的集合，不是面板
  action,      -- create/read/update/delete
  access,      -- none/partial/full
  permissions, -- 权限过滤条件（JSON）
  fields,      -- 允许的字段
  presets      -- 预设值
) VALUES (
  'perm-uuid-abc',
  'role-uuid-def',
  'articles',    -- 权限针对 articles 集合
  'read',        -- 读操作权限
  'partial',     -- 部分权限
  -- 权限过滤条件：只能看到 category = 'news' 的文章
  '{"category":{"_eq":"news"}}',
  '["id","title","views","status","category"]',
  NULL
);
```

### 5.5 再次进入页面重渲染时序

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                      阶段3: 再次进入页面重渲染                                            │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────┐     ┌──────────────────────┐     ┌─────────────────────────────┐   │
│  │   用户界面    │     │    insights/index.ts │     │      insightsStore          │   │
│  └──────┬───────┘     └──────────┬───────────┘     └──────────────┬──────────────┘   │
│         │                        │                                  │                   │
│         │  导航到仪表盘页面       │                                  │                   │
│         │───────────────────────>│                                  │                   │
│         │                        │                                  │                   │
│         │                        │  beforeEnter 钩子                │                   │
│         │                        │  store.refresh(dashboardId)    │                   │
│         │                        │────────────────────────────────>│                   │
│         │                        │                                  │                   │
│         │                        │                                  │  hydrate()        │
│         │                        │                                  │  ─────────────    │
│         │                        │                                  │  1. 从 API 加载   │
│         │                        │                                  │     仪表盘和面板   │
│         │                        │                                  │     配置数据       │
│         │                        │                                  │                   │
│         │                        │                                  │  GET /dashboards  │
│         │                        │                                  │  ?fields=*.*      │
│         │                        │                                  │──────────────────>│
│         │                        │                                  │                   │
└─────────┴────────────────────────┴──────────────────────────────────┴─────────────────┘ │
                                          ↓                                                   │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         后端：加载面板配置                                                  │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐   │
│  │ dashboardsController │     │  DashboardsService   │     │    ItemsService     │   │
│  └──────────┬───────────┘     └──────────┬───────────┘     └──────────┬──────────┘   │
│             │                              │                              │              │
│             │  GET /dashboards/:id         │                              │              │
│             │  fields=panels.*             │                              │              │
│             │─────────────────────────────>│                              │              │
│             │                              │                              │              │
│             │                              │  readOne(pk, query)          │              │
│             │                              │─────────────────────────────>│              │
│             │                              │                              │              │
│             │                              │                              │  权限校验：   │
│             │                              │                              │              │
│             │                              │                              │  1. 检查用户 │
│             │                              │                              │     是否有    │
│             │                              │                              │     directus_ │
│             │                              │                              │     dashboards│
│             │                              │                              │     的 read   │
│             │                              │                              │     权限       │
│             │                              │                              │              │
│             │                              │                              │  2. processAst│
│             │                              │                              │     注入权限   │
│             │                              │                              │     过滤条件    │
│             │                              │                              │              │
│             │                              │                              │  3. 查询数据库│
│             │                              │                              │              │
│             │                              │                              │  SELECT *     │
│             │                              │                              │  FROM directus│
│             │                              │                              │  _dashboards  │
│             │                              │                              │  JOIN directus│
│             │                              │                              │  _panels      │
│             │                              │                              │  ON dashboard │
│             │                              │                              │  = dashboard.id│
│             │                              │                              │              │
│             │                              │  返回仪表盘+面板数据          │              │
│             │                              │<─────────────────────────────│              │
│             │                              │                              │              │
│             │  返回数据到前端              │                              │              │
│             │<─────────────────────────────│                              │              │
│             │                              │                              │              │
└─────────────┴──────────────────────────────┴──────────────────────────────┴──────────────┘ │
                                                                                          │
  【关键】此时从数据库读取的是：
  - 仪表盘配置
  - 面板配置（options）：包含 collection、filter 等数据源配置
  - 不包含权限配置！权限在查询数据时动态应用
                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                          ↓                                                   │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         阶段4: 加载面板数据（权限动态应用）                                │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐   │
│  │    insightsStore     │     │   GraphQLService    │     │    ItemsService     │   │
│  └──────────┬───────────┘     └──────────┬───────────┘     └──────────┬──────────┘   │
│             │                              │                              │              │
│             │  loadPanelData()             │                              │              │
│             │  ─────────────               │                              │              │
│             │  1. 遍历所有面板             │                              │              │
│             │  2. 调用 panel.query()       │                              │              │
│             │     生成查询配置              │                              │              │
│             │  3. 构建 GraphQL 查询        │                              │              │
│             │                              │                              │              │
│             │  POST /graphql               │                              │              │
│             │  { query: "...", ... }       │                              │              │
│             │─────────────────────────────>│                              │              │
│             │                              │                              │              │
│             │                              │  execute(graphqlParams)     │              │
│             │                              │─────────────────────────────>│              │
│             │                              │                              │              │
│             │                              │  解析查询中的 collection     │              │
│             │                              │  例如：articles              │              │
│             │                              │                              │              │
│             │                              │  创建对应集合的 ItemsService │              │
│             │                              │  accountability 传递下去      │              │
│             │                              │─────────────────────────────>│              │
│             │                              │                              │              │
│             │                              │                              │  权限校验：   │
│             │                              │                              │              │
│             │                              │                              │  1. 检查用户 │
│             │                              │                              │     是否有    │
│             │                              │                              │     articles 的│
│             │                              │                              │     read 权限 │
│             │                              │                              │              │
│             │                              │                              │  2. processAst│
│             │                              │                              │     关键步骤！│
│             │                              │                              │              │
│             │                              │                              │  合并过滤条件：│
│             │                              │                              │  filter = {  │
│             │                              │                              │    _and: [   │
│             │                              │                              │      // 用户  │
│             │                              │                              │      // 配置的│
│             │                              │                              │      // 过滤  │
│             │                              │                              │      { status:│
│             │                              │                              │        _eq:   │
│             │                              │                              │        'pub'  │
│             │                              │                              │      },       │
│             │                              │                              │      // 权限  │
│             │                              │                              │      // 配置的│
│             │                              │                              │      // 过滤  │
│             │                              │                              │      { category│
│             │                              │                              │      : _eq:   │
│             │                              │                              │        'news' │
│             │                              │                              │      }        │
│             │                              │                              │    ]          │
│             │                              │                              │  }            │
│             │                              │                              │              │
│             │                              │                              │  3. 执行查询  │
│             │                              │                              │              │
│             │                              │                              │  SELECT      │
│             │                              │                              │  SUM(views)   │
│             │                              │                              │  FROM articles│
│             │                              │                              │  WHERE       │
│             │                              │                              │    status =   │
│             │                              │                              │    'published'│
│             │                              │                              │    AND        │
│             │                              │                              │    category = │
│             │                              │                              │    'news'     │
│             │                              │                              │              │
│             │                              │  返回聚合结果                │              │
│             │                              │<─────────────────────────────│              │
│             │                              │                              │              │
│             │  返回数据                    │                              │              │
│             │<─────────────────────────────│                              │              │
│             │                              │                              │              │
│             │  存储到 data[panel.id]       │                              │              │
│             │                              │                              │              │
└─────────────┴──────────────────────────────┴──────────────────────────────┴──────────────┘ │
                                                                                          │
  【关键】此时：
  - 面板数据查询时，权限条件被动态注入到查询中
  - 用户配置的 filter 与权限配置的 filter 合并为最终的 WHERE 条件
  - 权限检查是强制的，无法绕过
                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
                                          ↓                                                   │
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         阶段5: 面板渲染                                                    │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────────────┐     ┌──────────────────────┐     ┌─────────────────────┐   │
│  │     dashboard.vue    │     │   panel-metric.vue   │     │   Vue 渲染系统      │   │
│  └──────────┬───────────┘     └──────────┬───────────┘     └──────────┬──────────┘   │
│             │                              │                              │              │
│             │  遍历 tiles 数组             │                              │              │
│             │  构建渲染配置                 │                              │              │
│             │                              │                              │              │
│             │  <component                  │                              │              │
│             │    :is="panel-${type}"      │                              │              │
│             │    :data="data[panel.id]"   │                              │              │
│             │    v-bind="options"         │                              │              │
│             │  />                          │                              │              │
│             │                              │                              │              │
│             │  动态组件解析                │                              │              │
│             │  注册为 panel-metric         │                              │              │
│             │─────────────────────────────>│                              │              │
│             │                              │                              │              │
│             │                              │  接收 props:                 │                              │              │
│             │                              │  - data: 聚合结果            │                              │              │
│             │                              │  - options: 面板配置         │                              │              │
│             │                              │  - 其他布局参数              │                              │              │
│             │                              │                              │              │
│             │                              │  渲染面板内容                │                              │              │
│             │                              │  - 显示标题                  │                              │              │
│             │                              │  - 显示聚合值                │                              │              │
│             │                              │  - 应用样式                  │                              │              │
│             │                              │                              │              │
│             │                              │  触发 Vue 响应式更新        │              │
│             │                              │─────────────────────────────>│              │
│             │                              │                              │              │
│             │  页面呈现完整的仪表盘        │                              │              │
│             │  包含所有面板的渲染结果      │                              │              │
│             │                              │                              │              │
└─────────────┴──────────────────────────────┴──────────────────────────────┴──────────────┘ │
                                                                                          │
  【关键】此时：
  - 面板组件使用的是经过权限过滤的数据
  - 用户只能看到有权限访问的数据聚合结果
  - 面板配置（options）用于 UI 渲染，不直接决定数据可见性
                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.6 完整时序流程图（简化版）

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                              完整时序：面板配置 → 保存 → 重渲染                                 │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  用户编辑阶段                          保存阶段                          重渲染阶段            │
│  ────────────                          ────────                          ──────────            │
│                                                                                                │
│  ┌─────────────┐                   ┌─────────────┐                   ┌─────────────┐         │
│  │  用户界面   │                   │  用户界面   │                   │  用户界面   │         │
│  └──────┬──────┘                   └──────┬──────┘                   └──────┬──────┘         │
│         │                                 │                                 │                    │
│         │  配置面板选项                    │  点击保存                      │  导航到仪表盘        │
│         │  (collection, filter)           │                                 │                    │
│         ▼                                 ▼                                 ▼                    │
│  ┌─────────────┐                   ┌─────────────┐                   ┌─────────────┐         │
│  │insightsStore│                   │insightsStore│                   │insightsStore│         │
│  │  暂存编辑   │                   │ saveChanges │                   │   hydrate   │         │
│  │ (edits.xxx) │                   │             │                   │   refresh   │         │
│  └──────┬──────┘                   └──────┬──────┘                   └──────┬──────┘         │
│         │                                 │                                 │                    │
│         │  预览数据查询                    │  POST /panels                  │  GET /dashboards    │
│         │  POST /graphql                  │  PATCH /panels                 │  读取面板配置        │
│         │  (权限已动态应用)                │  DELETE /panels                │                    │
│         ▼                                 ▼                                 ▼                    │
│  ┌─────────────────────────────┐   ┌─────────────────────────────┐   ┌──────────────────┐   │
│  │    后端: GraphQLService     │   │    后端: PanelsService      │   │后端: Dashboards-  │   │
│  │    读操作: processAst       │   │    写操作: processPayload   │   │Service (读配置)   │   │
│  │    注入权限过滤条件          │   │    检查 directus_panels    │   │                    │   │
│  └─────────────────────────────┘   └─────────────────────────────┘   └──────────────────┘   │
│         │                                 │                                 │                    │
│         │  返回预览数据                    │  写入 directus_panels         │  返回配置数据        │
│         │  (已过滤)                       │  表（不含权限）                │  (含 options)        │
│         ▼                                 ▼                                 ▼                    │
│  ┌─────────────┐                   ┌─────────────┐                   ┌─────────────┐         │
│  │insightsStore│                   │insightsStore│                   │insightsStore│         │
│  │data[panel.id]│                   │ clearEdits  │                   │loadPanelData│         │
│  │ (预览缓存)  │                   │             │                   │ (数据查询)  │         │
│  └─────────────┘                   └─────────────┘                   └──────┬──────┘         │
│                                                                               │                    │
│                                                                               │  POST /graphql    │
│                                                                               │  (权限再次应用)    │
│                                                                               ▼                    │
│                                                                    ┌─────────────────────────────┐│
│                                                                    │    后端: GraphQLService     ││
│                                                                    │    processAst 合并过滤条件   ││
│                                                                    │    • 用户配置的 filter       ││
│                                                                    │    • 权限配置的 filter       ││
│                                                                    └─────────────────────────────┘│
│                                                                               │                    │
│                                                                               │  返回已过滤数据      │
│                                                                               ▼                    │
│                                                                    ┌─────────────────────────────┐│
│                                                                    │    dashboard.vue 渲染        ││
│                                                                    │    动态组件: panel-${type}   ││
│                                                                    │    数据: data[panel.id]      ││
│                                                                    └─────────────────────────────┘│
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.7 完整重渲染时序详解（三环节串清）

#### 5.7.1 重渲染整体流程概览

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                    再次进入页面：完整重渲染时序（三环节串清）                                   │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  【阶段1：导航与初始化】                    【阶段2：加载配置】                                │
│  ─────────────────────                    ────────────────                                │
│                                                                                                │
│  用户点击导航 ──→ 路由守卫 beforeEnter ──→ store.refresh(dashboardId)                      │
│                          │                          │                                        │
│                          ▼                          ▼                                        │
│                 检查模块级权限              hydrate() 加载仪表盘和面板配置                  │
│                 (preRegisterCheck)              │                                        │
│                                                  ▼                                        │
│                                          GET /dashboards + GET /panels                     │
│                                          (并行请求)                                          │
│                                                  │                                        │
│                                                  ▼                                        │
│                                          更新 dashboards.value + panels.value              │
│                                          (响应式数据更新)                                  │
│                                                                                                │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  【环节1：数据查询回传】                        【环节2：data[panel.id] 写入】              │
│  ───────────────────                        ─────────────────────────────              │
│                                                                                                │
│  refresh(dashboard) ──→ loadPanelData(panels) ──→ prepareQuery(panel)                    │
│         │                               │                         │                        │
│         │                               ▼                         ▼                        │
│         │                    遍历面板，调用 panel.query()    应用变量替换                  │
│         │                    生成查询配置 (collection, filter)  applyOptionsData         │
│         │                               │                         │                        │
│         │                               ▼                         ▼                        │
│         │                    构建 GraphQL 查询字符串      合并多个面板查询                 │
│         │                    queryToGqlString()              去重优化                       │
│         │                               │                         │                        │
│         │                               ▼                         ▼                        │
│         │                    POST /graphql              后端执行查询                         │
│         │                    (携带 accountability)         • 检查权限                        │
│         │                               │                    • processAst 注入过滤条件        │
│         │                               │                    • 执行数据库查询                 │
│         │                               │                         │                        │
│         │                               ▼                         ▼                        │
│         │                    后端返回数据 ◄────────────────── 返回聚合结果                  │
│         │                    (已应用权限过滤)                  (经过 filter 合并)           │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    【环节2：data[panel.id] 写入】                                │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    解析响应，映射查询 key 到面板 id                                │
│         │                    mapKeys(data.data, (_, key) => key.substring('query_'.length)) │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    results[panel.id] = data                                      │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    data.value = assign({}, data.value, results)                  │
│         │                    ┌─────────────────────────────────────────────┐              │
│         │                    │  此时：data[panel.id] 被响应式更新          │              │
│         │                    │  例如：data['panel-123'] = { sum: 1500 }  │              │
│         │                    └─────────────────────────────────────────────┘              │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    【环节3：动态组件重绘】                                      │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    Vue 响应式系统触发：data.value 变化                           │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    dashboard.vue 中的模板：                                       │
│         │                    <component                                                     │
│         │                        :is="`panel-${tile.data.type}`"                           │
│         │                        :data="data[tile.id]"    ◄─── 依赖 data[tile.id]         │
│         │                    />                                                            │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    当 data[tile.id] 变化时，Vue 自动重新渲染组件                 │
│         │                               │                                                  │
│         │                               ▼                                                  │
│         │                    面板组件 (如 panel-metric.vue) 使用 data prop 渲染内容       │
│         │                    显示聚合值、图表等                                             │
│         │                               │                                                  │
│         ▼                               ▼                                                  │
│  页面呈现完整的仪表盘 ──←── 所有面板渲染完成                                               │
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 5.7.2 环节1：查询回传详细时序

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                          环节1：数据查询回传（前端 → 后端 → 前端）                            │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  ┌─────────────────────────────────┐         ┌──────────────────────────────────────────┐   │
│  │         前端: insightsStore      │         │         后端: GraphQLService             │   │
│  └───────────────────┬─────────────┘         └───────────────────────┬──────────────────┘   │
│                      │                                                   │                       │
│                      │  loadPanelData(panels)                          │                       │
│                      │  ─────────────────────                          │                       │
│                      │                                                   │                       │
│                      │  1. 遍历所有面板                                  │                       │
│                      │     for (const panel of panels)                 │                       │
│                      │                                                   │                       │
│                      │  2. prepareQuery(panel)                          │                       │
│                      │     ┌─────────────────────────────────────┐     │                       │
│                      │     │  • 获取面板类型: panelType =       │     │                       │
│                      │     │    panelTypes.find(type === panel.type)│   │                       │
│                      │     │  • 调用 panel.query(options)          │     │                       │
│                      │     │    例如 metric 面板返回:            │     │                       │
│                      │     │    { collection: 'articles',        │     │                       │
│                      │     │      query: { aggregate: { sum: ['views'] } }│                   │
│                      │     │  • 应用变量替换: applyOptionsData() │     │                       │
│                      │     │    替换 {{ variable }} 为实际值     │     │                       │
│                      │     └─────────────────────────────────────┘     │                       │
│                      │                                                   │                       │
│                      │  3. 构建 GraphQL 查询                          │                       │
│                      │     ┌─────────────────────────────────────┐     │                       │
│                      │     │  queryToGqlString([                  │     │                       │
│                      │     │    {                                  │     │                       │
│                      │     │      key: 'query_hash123',         │     │                       │
│                      │     │      collection: 'articles',        │     │                       │
│                      │     │      query: {                        │     │                       │
│                      │     │        aggregate: { sum: ['views'] },│     │                       │
│                      │     │        filter: { status: 'published' }│    │                       │
│                      │     │      }                                │     │                       │
│                      │     │    }                                  │     │                       │
│                      │     │  ])                                   │     │                       │
│                      │     │                                      │     │                       │
│                      │     │  生成:                               │     │                       │
│                      │     │  query query_hash123 {            │     │                       │
│                      │     │    articles_aggregated(           │     │                       │
│                      │     │      filter: { status: { _eq: "published" } }│                 │
│                      │     │    ) { sum_views }                  │     │                       │
│                      │     │  }                                    │     │                       │
│                      │     └─────────────────────────────────────┘     │                       │
│                      │                                                   │                       │
│                      │  POST /graphql                                   │                       │
│                      │  body: { query: gqlString }                    │                       │
│                      │──────────────────────────────────────────────────>│                       │
│                      │                                                   │                       │
│                      │                                                   │  execute(graphqlParams) │
│                      │                                                   │  ────────────────────── │
│                      │                                                   │                       │
│                      │                                                   │  1. 解析查询中的 collection│
│                      │                                                   │     例如: 'articles'    │
│                      │                                                   │                       │
│                      │                                                   │  2. 创建 ItemsService   │
│                      │                                                   │     new ItemsService(    │
│                      │                                                   │       'articles',         │
│                      │                                                   │       {                   │
│                      │                                                   │         accountability:  │
│                      │                                                   │           req.accountability│
│                      │                                                   │           // 包含用户信息  │
│                      │                                                   │           // 和角色权限    │
│                      │                                                   │       }                   │
│                      │                                                   │     )                     │
│                      │                                                   │                       │
│                      │                                                   │  3. readByQuery(query)   │
│                      │                                                   │     ┌─────────────────────┐ │
│                      │                                                   │     │ 【关键】processAst    │ │
│                      │                                                   │     │                    │ │
│                      │                                                   │     │ 3.1 检查权限         │ │
│                      │                                                   │     │     collectionAccess  │ │
│                      │                                                   │     │     = fetchAccount-    │ │
│                      │                                                   │     │       abilityCollection │ │
│                      │                                                   │     │       Access(...)     │ │
│                      │                                                   │     │                    │ │
│                      │                                                   │     │     • 如果 access =    │ │
│                      │                                                   │     │       'none' 抛出错误  │ │
│                      │                                                   │     │                    │ │
│                      │                                                   │     │ 3.2 如果 access =     │ │
│                      │                                                   │     │     'partial' 注入    │ │
│                      │                                                   │     │     权限过滤条件      │ │
│                      │                                                   │     │                    │ │
│                      │                                                   │     │     // 权限配置中的   │ │
│                      │                                                   │     │     // permissions.filter│
│                      │                                                   │     │     permissionFilters │ │
│                      │                                                   │     │     = { category:     │ │
│                      │                                                   │     │         { _eq: "news" } }│
│                      │                                                   │     │                    │ │
│                      │                                                   │     │     // 合并过滤条件    │ │
│                      │                                                   │     │     ast.query.filter = │ │
│                      │                                                   │     │       {                │ │
│                      │                                                   │     │         _and: [        │ │
│                      │                                                   │     │           // 用户配置 │ │
│                      │                                                   │     │           { status:  │ │
│                      │                                                   │     │             { _eq:   │ │
│                      │                                                   │     │               "pub-   │ │
│                      │                                                   │     │                lished" }│
│                      │                                                   │     │           },          │ │
│                      │                                                   │     │           // 权限配置 │ │
│                      │                                                   │     │           { category:│ │
│                      │                                                   │     │             { _eq:   │ │
│                      │                                                   │     │               "news" }│
│                      │                                                   │     │           }           │ │
│                      │                                                   │     │         ]            │ │
│                      │                                                   │     │       }                │ │
│                      │                                                   │     └─────────────────────┘ │
│                      │                                                   │                       │
│                      │                                                   │  4. 执行数据库查询       │
│                      │                                                   │     SELECT SUM(views)    │
│                      │                                                   │     FROM articles          │
│                      │                                                   │     WHERE status =        │
│                      │                                                   │           'published'       │
│                      │                                                   │       AND category = 'news'│
│                      │                                                   │     // 两个条件都应用了！  │
│                      │                                                   │                       │
│                      │                                                   │  5. 返回聚合结果          │
│                      │                                                   │     { sum_views: 1500 }   │
│                      │                                                   │                       │
│                      │<──────────────────────────────────────────────────│                       │
│                      │  响应数据 (已过滤)                                 │                       │
│                      │  {                                                   │                       │
│                      │    data: {                                          │                       │
│                      │      query_hash123: { sum_views: 1500 }         │                       │
│                      │    }                                                │                       │
│                      │  }                                                   │                       │
│                      │                                                   │                       │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 5.7.3 环节2：data[panel.id] 写入详细时序

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                          环节2：data[panel.id] 写入（响应式更新）                            │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  前端: loadPanelData() 继续执行                                                               │
│  ──────────────────────────────                                                               │
│                                                                                                │
│  try {                                                                                         │
│      const responses = await Promise.all(requests);                                           │
│                                                                                                │
│      const results: { [panel: string]: Item | Item[] } = {};                                │
│                                                                                                │
│      for (const { data } of responses) {                                                      │
│          // 移除 'query_' 前缀，映射到原始 key                                               │
│          const result = mapKeys(                                                  │
│              data.data,                                                                       │
│              (_, key) => key.substring('query_'.length)                                      │
│          );                                                                                   │
│                                                                                                │
│          for (const [key, data] of Object.entries(result)) {                                │
│              const { panel, length } = queries.get(key);                                      │
│                                                                                                │
│              if (length === 1) results[panel] = data;                                        │
│              else if (!results[panel]) results[panel] = [data];                              │
│              else results[panel]?.push(data);                                                 │
│          }                                                                                     │
│      }                                                                                          │
│                                                                                                │
│      // 【关键】响应式更新                                                                     │
│      // data.value 是 ref() 创建的响应式对象                                                   │
│      // assign() 会合并数据，触发 Vue 的响应式更新                                            │
│      data.value = assign({}, data.value, results);                                            │
│                                                                                                │
│      // 此时：                                                                                  │
│      // data.value = {                                                                         │
│      //     'panel-uuid-123': { sum_views: 1500 },    ◄── 新数据                            │
│      //     'panel-uuid-456': { count: 42 },           ◄── 旧数据                            │
│      // }                                                                                       │
│                                                                                                │
│      // 更新成功/失败状态                                                                       │
│      const succeededPanels = Object.keys(results);                                             │
│      errors.value = omit(errors.value, ...succeededPanels);                                  │
│                                                                                                │
│  } finally {                                                                                   │
│      // 更新 loading 状态                                                                      │
│      loading.value = pull(unref(loading), ...Array.from(queries.values()).map(({ panel }) => panel));│
│  }                                                                                              │
│                                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  【Vue 响应式机制】                                                                         │ │
│  │                                                                                             │ │
│  │  1. data.value 是通过 ref() 创建的响应式对象                                                │ │
│  │     const data = ref<{ [panel: string]: Item | Item[] }>({});                             │ │
│  │                                                                                             │ │
│  │  2. 当执行 data.value = assign(...) 时                                                       │ │
│  │     - Vue 检测到 data.value 的引用变化                                                      │ │
│  │     - 或者检测到对象内部属性变化 (Vue 3 Proxy 机制)                                          │ │
│  │                                                                                             │ │
│  │  3. 触发所有依赖 data.value 的 computed、watch、模板渲染                                    │ │
│  │                                                                                             │ │
│  │  依赖示例：                                                                                  │ │
│  │  - dashboard.vue 模板中的 :data="data[tile.id]"                                           │ │
│  │  - 任何 watch(data, ...) 监听器                                                             │ │
│  │  - 任何 computed 中引用了 data.value                                                        │ │
│  │                                                                                             │ │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 5.7.4 环节3：动态组件重绘详细时序

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                          环节3：动态组件重绘（Vue 响应式渲染）                                 │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  【第一步：tiles 计算属性更新】                                                                 │
│  ────────────────────────────────                                                              │
│                                                                                                │
│  // dashboard.vue                                                                              │
│  const tiles = computed<AppTile[]>(() => {                                                    │
│      const panels = insightsStore.getPanelsForDashboard(props.primaryKey);                  │
│                                                                                                │
│      // 从 panels.value 获取面板配置                                                           │
│      // panels.value 也是响应式的，但在 hydrate 后已稳定                                       │
│                                                                                                │
│      const tiles: AppTile[] = panels.map((panel) => {                                         │
│          const panelType = unref(panelsInfo).find((panelType) => panelType.id === panel.type);│
│                                                                                                │
│          return {                                                                              │
│              id: panel.id,                                                                     │
│              x: panel.position_x,                                                              │
│              y: panel.position_y,                                                              │
│              width: panel.width,                                                               │
│              height: panel.height,                                                             │
│              data: {                                                                           │
│                  // 应用变量替换到 options                                                      │
│                  options: applyOptionsData(panel.options ?? {}, unref(variables), ...),      │
│                  type: panel.type,                                                             │
│              },                                                                                │
│          };                                                                                    │
│      });                                                                                        │
│                                                                                                │
│      return tiles;                                                                              │
│  });                                                                                            │
│                                                                                                │
│  【第二步：模板渲染与依赖追踪】                                                                 │
│  ────────────────────────────────                                                              │
│                                                                                                │
│  <!-- dashboard.vue 模板 -->                                                                   │
│  <VWorkspace                                                                                   │
│      v-model="editing"                                                                        │
│      :tiles="tiles"                         ◄─── 依赖 tiles 计算属性                           │
│      :zoom-to-fit="zoomToFit"                                                                 │
│  >                                                                                              │
│      <template #default="{ tile, gridSize }">                                                  │
│                                                                                                │
│          <!-- 加载状态 -->                                                                     │
│          <VProgressCircular                                                                   │
│              v-if="loading.includes(tile.id) && !data[tile.id]"  ◄─── 依赖 data[tile.id]   │
│              :class="{ 'header-offset': tile.showHeader }"                                    │
│              class="panel-loading"                                                             │
│              indeterminate                                                                      │
│          />                                                                                     │
│                                                                                                │
│          <div v-else class="panel-container" :class="{ loading: loading.includes(tile.id) }"> │
│                                                                                                │
│              <!-- 错误状态 -->                                                                 │
│              <div v-if="errors[tile.id]" class="panel-error">   ◄─── 依赖 errors[tile.id]   │
│                  <VIcon name="warning" />                                                      │
│                  {{ $t('unexpected_error') }}                                                  │
│                  <VError :error="errors[tile.id]" />                                          │
│              </div>                                                                             │
│                                                                                                │
│              <!-- 无数据状态 -->                                                                │
│              <div                                                                               │
│                  v-else-if="tile.id in data && isEmpty(data[tile.id])"                        │
│                  class="panel-no-data type-note"                                              │
│                  :class="{ 'header-offset': tile.showHeader }"                                │
│              >                                                                                  │
│                  {{ $t('no_data') }}                                                           │
│              </div>                                                                             │
│                                                                                                │
│              <!-- 【关键】动态组件渲染 -->                                                       │
│              <VErrorBoundary v-else :name="`panel-${tile.data.type}`">                        │
│                                                                                                │
│                  <component                                                                     │
│                      :is="`panel-${tile.data.type}`"     ◄─── 动态组件类型                  │
│                      v-bind="tile.data.options"            ◄─── 面板配置选项                 │
│                      :id="tile.id"                                                                 │
│                      :dashboard="primaryKey"                                                      │
│                      :show-header="tile.showHeader"                                              │
│                      :height="tile.height"                                                        │
│                      :width="tile.width"                                                          │
│                      :now="now"                                                                    │
│                      :data="data[tile.id]"                  ◄─── 【关键】依赖 data[tile.id]   │
│                      :grid-size="gridSize"                                                        │
│                  />                                                                               │
│                                                                                                │
│                  <template #fallback="{ error }">                                                │
│                      <div class="panel-error">                                                    │
│                          <VIcon name="warning" />                                                  │
│                          {{ $t('unexpected_error') }}                                              │
│                          <VError :error="error" />                                                │
│                      </div>                                                                         │
│                  </template>                                                                       │
│              </VErrorBoundary>                                                                     │
│          </div>                                                                                     │
│      </template>                                                                                   │
│  </VWorkspace>                                                                                      │
│                                                                                                │
│  【第三步：Vue 响应式触发与组件重渲染】                                                         │
│  ────────────────────────────────────────────────                                                │
│                                                                                                │
│  当 data[panel.id] 发生变化时：                                                                  │
│                                                                                                │
│  1. Vue 3 Proxy 检测到 data.value 对象的属性变化                                                │
│     data.value['panel-uuid-123'] = { sum_views: 1500 }                                        │
│                                                                                                │
│  2. 触发所有依赖该数据的组件重新渲染                                                              │
│     - 模板中的 v-if="tile.id in data"                                                          │
│     - 模板中的 :data="data[tile.id]"                                                           │
│     - 所有引用 data[tile.id] 的计算属性和监听器                                                 │
│                                                                                                │
│  3. 动态组件 `<component :is="panel-${type}">` 重新渲染                                        │
│                                                                                                │
│     例如，类型为 'metric' 的面板：                                                               │
│     - :is="`panel-metric`" 解析为组件 'panel-metric'                                           │
│     - 该组件已在 app/src/panels/index.ts 中注册                                                 │
│                                                                                                │
│  4. 面板组件 (如 panel-metric.vue) 接收到新的 data prop                                         │
│                                                                                                │
│     // panel-metric.vue 简化版                                                                 │
│     <script setup lang="ts">                                                                    │
│         const props = defineProps<{                                                             │
│             // ... 其他 props                                                                   │
│             data?: Item | Item[];             ◄─── 接收 data prop                              │
│         }>();                                                                                    │
│     </script>                                                                                    │
│                                                                                                │
│     <template>                                                                                   │
│         <div class="type-metric">                                                                │
│             <!-- 使用 data prop 渲染内容 -->                                                    │
│             <div class="metric-value" v-text="data?.[computedField]" />                       │
│             <div class="metric-label" v-text="label" />                                         │
│         </div>                                                                                   │
│     </template>                                                                                  │
│                                                                                                │
│  5. 当 data prop 变化时，Vue 自动更新 DOM                                                       │
│     - 新的数值显示                                                                               │
│     - 图表重新渲染（如果是图表面板）                                                              │
│     - 所有响应式绑定更新                                                                         │
│                                                                                                │
│  【第四步：完整的渲染结果】                                                                     │
│  ────────────────────────────────                                                              │
│                                                                                                │
│  最终用户看到：                                                                                 │
│  ┌─────────────────────────────────────────────────────────────┐                                │
│  │  ┌─────────────────────────────────────────────────────┐   │                                │
│  │  │  文章浏览量统计                          [配置] [×]  │   │                                │
│  │  │                                                     │   │                                │
│  │  │      ┌─────────────────────────────────────┐       │   │                                │
│  │  │      │                                     │       │   │                                │
│  │  │      │           15,432                    │       │   │                                │
│  │  │      │         总浏览量                    │       │   │                                │
│  │  │      │                                     │       │   │                                │
│  │  │      └─────────────────────────────────────┘       │   │                                │
│  │  │                                                     │   │                                │
│  │  │  显示的是经过权限过滤的数据：                          │   │                                │
│  │  │  - 面板配置要求：status = 'published'                 │   │                                │
│  │  │  - 权限要求：category = 'news'                        │   │                                │
│  │  │  - 实际查询：WHERE status = 'published' AND category = 'news'│                         │
│  │  │                                                     │   │                                │
│  │  └─────────────────────────────────────────────────────┘   │                                │
│  └─────────────────────────────────────────────────────────────┘                                │
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 5.8 关键发现总结

#### 5.8.1 权限不是面板独立存储的

| 维度 | 说明 |
|------|------|
| **存储位置** | 权限配置存储在 `directus_permissions` 表，与面板配置分离 |
| **应用时机** | 权限在**数据查询执行时**动态应用，不是在面板配置保存时 |
| **作用对象** | 权限针对的是**集合**（如 `articles`），不是面板 |
| **配置方式** | 通过角色权限配置，不是通过面板配置界面 |

#### 5.8.2 权限与查询的关系

```
最终查询的 WHERE 条件 = 用户配置的 filter + 权限配置的 filter

示例：
- 面板配置 (options.filter): { status: { _eq: "published" } }
- 权限配置 (permissions.filter): { category: { _eq: "news" } }

合并后的 WHERE:
WHERE status = 'published' AND category = 'news'

这意味着：
1. 即使面板配置了查询所有 status='published' 的文章
2. 如果用户的权限限制只能看 category='news' 的文章
3. 实际查询的是两个条件的交集
4. 用户无法绕过权限限制
```

#### 5.8.3 权限层级关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                        权限层级与面板的关系                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              模块级权限 (Module Level)                        │   │
│  │  控制：能否访问仪表盘模块                                      │   │
│  │  检查：directus_dashboards 的 read 权限                       │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │           配置级权限 (Config Level)                           │   │
│  │  控制：能否创建/修改/删除仪表盘和面板                          │   │
│  │  检查：                                                       │   │
│  │    • directus_dashboards 的 create/update/delete 权限        │   │
│  │    • directus_panels 的 create/update/delete 权限            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              ↓                                       │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │           数据级权限 (Data Level) ⭐ 关键                    │   │
│  │  控制：面板能查询哪些业务数据                                  │   │
│  │  检查：                                                       │   │
│  │    • 业务集合（如 articles）的 read 权限                      │   │
│  │    • processAst 动态注入权限过滤条件                          │   │
│  │    • 与面板配置的 filter 合并                                 │   │
│  │                                                               │   │
│  │  【关键点】这是面板数据源权限的核心机制！                      │   │
│  │  - 权限配置与面板配置分离存储                                  │   │
│  │  - 数据查询时动态应用权限                                      │   │
│  │  - 用户无法通过修改面板配置绕过权限                            │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 8. 结论：权限不随面板持久化，而是在查询执行阶段动态注入

### 8.1 核心结论

**结论 1：权限配置不随面板持久化存储**

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                        面板配置 vs 权限配置：分离存储                                            │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  ┌─────────────────────────────────┐              ┌─────────────────────────────────────────┐│
│  │      directus_panels 表         │              │        directus_permissions 表         ││
│  │      （面板配置存储）            │              │         （权限配置存储）                ││
│  ├─────────────────────────────────┤              ├─────────────────────────────────────────┤│
│  │  字段              内容         │              │  字段              内容                 ││
│  ├─────────────────────────────────┤              ├─────────────────────────────────────────┤│
│  │  id               uuid          │              │  id               uuid                  ││
│  │  name             面板名称      │              │  role             角色 ID               ││
│  │  type             面板类型      │              │  collection       目标集合（如 articles）││
│  │  dashboard        所属仪表盘    │              │  action           操作类型（read等）   ││
│  │  options          ⭐ 关键       │              │  access           权限级别              ││
│  │                   数据源配置     │              │  permissions      ⭐ 关键               ││
│  │                   • collection   │              │                   权限过滤条件（JSON）   ││
│  │                   • filter       │              │  fields           允许的字段列表        ││
│  │                   • aggregation  │              │  presets          创建/更新预设值        ││
│  │                   • ...          │              └─────────────────────────────────────────┘│
│  └─────────────────────────────────┘                                                  │
│                                                                                                │
│  【关键对比】                                                                                  │
│                                                                                                │
│  面板配置 (options)：                                                                          │
│  - 存储在 directus_panels.options 字段                                                        │
│  - 内容：查询哪个集合、用什么过滤条件、如何聚合                                                  │
│  - 由用户在面板配置界面设置                                                                     │
│  - 不包含任何权限信息                                                                            │
│                                                                                                │
│  权限配置 (permissions 表)：                                                                    │
│  - 存储在 directus_permissions 表（独立的表）                                                  │
│  - 内容：角色对集合的访问权限、权限过滤条件、字段限制                                           │
│  - 由管理员在角色权限界面设置                                                                   │
│  - 与面板配置完全分离                                                                            │
│                                                                                                │
│  【结论】权限配置不随面板持久化，两者完全分离存储！                                              │
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

**结论 2：权限在查询执行阶段动态注入**

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                    权限注入时机：查询执行阶段（非面板保存阶段）                                 │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  【错误认知】权限是在保存面板配置时绑定到面板的                                               │ │
│  │                                                                                             │ │
│  │  ❌ 错误流程：                                                                               │ │
│  │  用户配置面板 → 保存面板配置 → 权限绑定到面板配置 → 下次查询时使用绑定的权限               │ │
│  │                                                                                             │ │
│  │  【正确认知】权限是在每次查询执行时动态注入的                                                 │ │
│  │                                                                                             │ │
│  │  ✅ 正确流程：                                                                               │ │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐                     │ │
│  │  │用户配置 │ → │保存面板  │ → │查询数据  │ → │动态注入 │ → │返回过滤  │                     │ │
│  │  │面板     │   │配置     │   │请求     │   │权限     │   │后的数据 │                     │ │
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘                     │ │
│  │                                      │              │              │                        │ │
│  │                                      ▼              ▼              ▼                        │ │
│  │                                每次查询都    从当前用户    与面板配置的                    │ │
│  │                                重新获取权限  的角色获取    filter 合并                     │ │
│  │                                (非从面板    权限配置      (每次都重新合并)                │ │
│  │                                配置读取)                                                     │ │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                │
│  【详细时序对比】                                                                              │
│                                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  场景：用户 A 保存了一个面板配置，用户 B 也能看到这个面板                                     │ │
│  │                                                                                             │ │
│  │  面板配置（保存在数据库，两人看到的相同）：                                                   │ │
│  │  {                                                                                           │ │
│  │    type: 'metric',                                                                           │ │
│  │    options: {                                                                                │ │
│  │      collection: 'articles',                                                                 │ │
│  │      filter: { status: { _eq: 'published' } },  ◄─── 两人看到的相同                       │ │
│  │      aggregation: { sum: ['views'] }                                                         │ │
│  │    }                                                                                          │ │
│  │  }                                                                                            │ │
│  │                                                                                             │ │
│  │  ─────────────────────────────────────────────────────────────────────────────────────────  │ │
│  │                                                                                             │ │
│  │  用户 A 的权限（独立存储，与面板无关）：                                                     │ │
│  │  {                                                                                           │ │
│  │    role: 'editor',                                                                           │ │
│  │    collection: 'articles',                                                                   │ │
│  │    action: 'read',                                                                           │ │
│  │    permissions: { category: { _eq: 'technology' } },  ◄─── 权限过滤条件                    │ │
│  │    access: 'partial'                                                                         │ │
│  │  }                                                                                            │ │
│  │                                                                                             │ │
│  │  用户 A 实际查询的 WHERE 条件：                                                              │ │
│  │  WHERE status = 'published' AND category = 'technology'                                     │ │
│  │        └── 面板配置 ──┘      └───── 权限配置 ─────┘                                        │ │
│  │                                                                                             │ │
│  │  ─────────────────────────────────────────────────────────────────────────────────────────  │ │
│  │                                                                                             │ │
│  │  用户 B 的权限（独立存储，与面板无关）：                                                     │ │
│  │  {                                                                                           │ │
│  │    role: 'marketing',                                                                        │ │
│  │    collection: 'articles',                                                                   │ │
│  │    action: 'read',                                                                           │ │
│  │    permissions: { category: { _eq: 'marketing' } },   ◄─── 不同的权限过滤条件            │ │
│  │    access: 'partial'                                                                         │ │
│  │  }                                                                                            │ │
│  │                                                                                             │ │
│  │  用户 B 实际查询的 WHERE 条件：                                                              │ │
│  │  WHERE status = 'published' AND category = 'marketing'                                      │ │
│  │        └── 面板配置 ──┘      └───── 权限配置 ─────┘                                        │ │
│  │                                                                                             │ │
│  │  ─────────────────────────────────────────────────────────────────────────────────────────  │ │
│  │                                                                                             │ │
│  │  【结果】虽然两人看到的是同一个面板配置，但查询到的数据不同！                                  │ │
│  │                                                                                             │ │
│  │  用户 A 看到：technology 分类下已发布文章的总浏览量                                           │ │
│  │  用户 B 看到：marketing 分类下已发布文章的总浏览量                                            │ │
│  │                                                                                             │ │
│  │  这就是"权限动态注入"的实际效果！                                                            │ │
│  │                                                                                             │ │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 8.2 技术实现要点

#### 8.2.1 权限动态注入的核心代码路径

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                          权限动态注入的代码执行路径                                              │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  【前端发起请求】                                                                              │
│  ───────────────────                                                                          │
│  app/src/stores/insights.ts                                                                   │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  loadPanelData(panels) {                                                                   │ │
│  │      // 1. 准备查询配置                                                                     │ │
│  │      for (const panel of panels) {                                                         │ │
│  │          const req = prepareQuery(panel);  // 从面板配置生成查询                            │ │
│  │          // req = { collection: 'articles', query: { filter: {...}, aggregate: {...} } }  │ │
│  │      }                                                                                      │ │
│  │                                                                                              │ │
│  │      // 2. 构建 GraphQL 查询                                                                 │ │
│  │      const gqlString = queryToGqlString(...);                                               │ │
│  │                                                                                              │ │
│  │      // 3. 发送请求（携带用户认证信息）                                                       │ │
│  │      api.post(`/graphql`, { query: gqlString });  // ◄─── 不携带任何权限配置              │ │
│  │                                                                                              │ │
│  │      // 4. 处理响应                                                                         │ │
│  │      data.value = assign({}, data.value, results);                                           │ │
│  │  }                                                                                            │ │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                │
│                              │                                                                 │
│                              ▼                                                                 │
│                                                                                                │
│  【后端接收请求】                                                                              │
│  ───────────────────                                                                          │
│  api/src/controllers/graphql.ts                                                               │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  router.use(                                                                                │ │
│  │      '/',                                                                                   │ │
│  │      parseGraphQL,                                                                          │ │
│  │      asyncHandler(async (req, res, next) => {                                              │ │
│  │          const service = new GraphQLService({                                               │ │
│  │              accountability: req.accountability,  // ◄─── 关键：从请求中获取用户身份      │ │
│  │              schema: req.schema,                                                           │ │
│  │              scope: 'items',                                                                │ │
│  │          });                                                                                │ │
│  │                                                                                              │ │
│  │          res.locals['payload'] = await service.execute(res.locals['graphqlParams']);      │ │
│  │      }),                                                                                     │ │
│  │      respond,                                                                                │ │
│  │  );                                                                                          │ │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                │
│                              │                                                                 │
│                              ▼                                                                 │
│                                                                                                │
│  【核心：权限动态注入】                                                                        │
│  ───────────────────────                                                                        │
│  api/src/services/items.ts  (通过 GraphQLService 调用)                                         │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {                 │ │
│  │      // ...                                                                                  │ │
│  │                                                                                              │ │
│  │      let ast = await getAstFromQuery(...);  // 解析查询为 AST                                │ │
│  │                                                                                              │ │
│  │      // 【关键】processAst - 动态注入权限                                                    │ │
│  │      ast = await processAst(                                                                 │ │
│  │          { ast, action: 'read', accountability: this.accountability },  // ◄─── 用户身份   │ │
│  │          { knex: this.knex, schema: this.schema },                                          │ │
│  │      );                                                                                       │ │
│  │                                                                                              │ │
│  │      const records = await runAst(ast, ...);  // 执行查询                                   │ │
│  │                                                                                              │ │
│  │      return filteredRecords as Item[];                                                        │ │
│  │  }                                                                                            │ │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                │
│                              │                                                                 │
│                              ▼                                                                 │
│                                                                                                │
│  【processAst 内部实现】                                                                       │
│  ────────────────────────                                                                      │
│  api/src/permissions/modules/process-ast/process-ast.ts                                        │
│  ┌──────────────────────────────────────────────────────────────────────────────────────────┐ │
│  │  export async function processAst(                                                          │ │
│  │      input: { ast: QueryAST; action: PermissionsAction; accountability: Accountability | null },│
│  │      context: { knex: Knex; schema: SchemaOverview },                                      │ │
│  │  ): Promise<QueryAST> {                                                                      │ │
│  │      if (!input.accountability) return input.ast;  // 无 accountability 跳过权限检查       │ │
│  │                                                                                              │ │
│  │      // 1. 获取用户对该集合的权限配置                                                         │ │
│  │      const collectionAccess = fetchAccountabilityCollectionAccess(                           │ │
│  │          input.accountability,                                                                │ │
│  │          input.action,                                                                        │ │
│  │          input.ast.name,  // 集合名，如 'articles'                                          │ │
│  │          context.schema,                                                                      │ │
│  │      );                                                                                       │ │
│  │                                                                                              │ │
│  │      if (collectionAccess.access === 'none') {                                               │ │
│  │          throw new ForbiddenError();  // 无权限直接抛出错误                                   │ │
│  │      }                                                                                        │ │
│  │                                                                                              │ │
│  │      // 2. 如果是 partial 权限，注入过滤条件                                                 │ │
│  │      if (collectionAccess.access === 'partial') {                                             │ │
│  │          // 从权限配置中获取 filter                                                          │ │
│  │          // 例如：{ category: { _eq: "news" } }                                             │ │
│  │          const permissionFilters = collectionAccess.permissions;                              │ │
│  │                                                                                              │ │
│  │          // 【关键】合并过滤条件                                                               │ │
│  │          // 原查询的 filter (来自面板配置) + 权限配置的 filter                               │ │
│  │          input.ast.query.filter = {                                                          │ │
│  │              _and: [                                                                         │ │
│  │                  input.ast.query.filter,  // 面板配置的 filter                               │ │
│  │                  permissionFilters,          // 权限配置的 filter                               │ │
│  │              ].filter(Boolean),                                                              │ │
│  │          };                                                                                   │ │
│  │      }                                                                                        │ │
│  │                                                                                              │ │
│  │      // 3. 字段级权限检查                                                                     │ │
│  │      const allowedFields = fetchAllowedFields(...);                                           │ │
│  │      const requestedFields = extractFieldsFromQuery(input.ast);                               │ │
│  │                                                                                              │ │
│  │      for (const field of requestedFields) {                                                   │ │
│  │          if (!allowedFields.includes(field)) {                                                │ │
│  │              throw new ForbiddenError(`You don't have permission to access the "${field}" field`);│
│  │          }                                                                                    │ │
│  │      }                                                                                        │ │
│  │                                                                                              │ │
│  │      return input.ast;                                                                        │ │
│  │  }                                                                                            │ │
│  └──────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 8.3 架构设计优势

| 设计决策 | 优势 |
|----------|------|
| **权限与配置分离存储** | 1. 权限管理集中在角色层面，便于统一管理<br>2. 同一面板不同用户看到不同数据（基于权限）<br>3. 权限变更立即生效，无需修改面板配置 |
| **查询时动态注入** | 1. 安全性：无法通过修改前端配置绕过权限<br>2. 灵活性：权限变更立即反映在所有相关面板<br>3. 可审计：所有数据访问都经过权限检查 |
| **使用 accountability 对象** | 1. 统一的用户身份传递机制<br>2. 支持管理员 bypass 权限检查（admin_access）<br>3. 支持动态权限变量（如 `$CURRENT_USER`） |

### 8.4 最终结论

```
┌───────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    最终结论                                                    │
├───────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                │
│  结论 1：权限不随面板持久化                                                                     │
│  ───────────────────────────                                                                   │
│  • 面板配置存储在 directus_panels 表的 options 字段，只包含数据源配置                          │
│  • 权限配置存储在 directus_permissions 表，与面板配置完全分离                                   │
│  • 两者之间没有直接关联，通过"集合名"间接关联（都针对同一集合）                                 │
│                                                                                                │
│  结论 2：权限在查询执行阶段动态注入                                                             │
│  ───────────────────────────────────                                                           │
│  • 每次面板数据查询时，都会从当前用户的角色权限中动态获取权限配置                                │
│  • processAst 函数在查询执行前将权限过滤条件注入到查询 AST 中                                   │
│  • 注入的权限配置与面板配置的 filter 合并为最终的 WHERE 条件                                   │
│  • 权限检查发生在服务端，前端无法绕过                                                           │
│                                                                                                │
│  结论 3：同一面板不同用户看到不同数据                                                           │
│  ─────────────────────────────────────────                                                        │
│  • 虽然所有用户看到的面板配置相同（相同的 collection、filter、aggregation）                   │
│  • 但每个用户查询时会注入自己的权限过滤条件                                                    │
│  • 因此实际查询的 WHERE 条件不同，返回的数据也不同                                              │
│  • 这是 Directus 权限系统的核心设计目标                                                         │
│                                                                                                │
│  结论 4：权限变更立即生效                                                                       │
│  ───────────────────────────                                                                   │
│  • 由于权限是动态注入的，修改角色权限后立即生效                                                 │
│  • 无需重新保存或修改任何面板配置                                                               │
│  • 所有使用该集合的面板都会自动应用新的权限                                                    │
│                                                                                                │
└───────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键代码位置索引

### 6.1 前端扩展
| 功能 | 文件路径 |
|------|----------|
| 扩展加载与注册 | `app/src/extensions.ts` |
| 面板注册 | `app/src/panels/index.ts` |
| 面板定义示例 | `app/src/panels/metric/index.ts` |
| 仪表盘模块 | `app/src/modules/insights/index.ts` |
| 仪表盘视图 | `app/src/modules/insights/routes/dashboard.vue` |
| 面板配置 | `app/src/modules/insights/routes/panel-configuration.vue` |

### 6.2 数据管理
| 功能 | 文件路径 |
|------|----------|
| 仪表板面数据 Store | `app/src/stores/insights.ts` |
| 权限 Store | `app/src/stores/permissions.ts` |
| 项目级权限 | `app/src/composables/use-permissions/item/use-item-permissions.ts` |
| 变量替换工具 | `@directus/utils` (applyOptionsData) |

### 6.3 后端权限核心模块
| 功能 | 文件路径 |
|------|----------|
| 读操作权限注入 | `api/src/permissions/modules/process-ast/process-ast.ts` |
| 写操作权限验证 | `api/src/permissions/modules/validate-access/validate-access.ts` |
| 权限预设应用 | `api/src/permissions/modules/process-payload/process-payload.ts` |

### 6.4 后端服务
| 功能 | 文件路径 |
|------|----------|
| 面板服务 | `api/src/services/panels.ts` |
| 面板控制器 | `api/src/controllers/panels.ts` |
| GraphQL 服务 | `api/src/services/graphql/index.ts` |
| GraphQL 控制器 | `api/src/controllers/graphql.ts` |
| ItemsService (核心) | `api/src/services/items.ts` |

### 6.5 数据模型
| 功能 | 文件路径 |
|------|----------|
| 面板表字段 | `packages/system-data/src/fields/panels.yaml` |
| 仪表盘表字段 | `packages/system-data/src/fields/dashboards.yaml` |
| 面板 SDK 类型 | `sdk/src/schema/panel.ts` |

---

## 7. 总结

### 7.1 核心架构特点

Directus 仪表盘面板扩展机制是一个设计完善的系统，具有以下核心特点：

#### 7.1.1 模块化扩展
- 面板通过 `definePanel` 定义，支持完全自定义
- 扩展自动加载和注册，无需手动配置
- 支持内部面板和自定义面板的无缝集成

#### 7.1.2 灵活的数据查询
- 每个面板定义自己的 `query()` 函数，生成查询配置
- 使用 GraphQL 进行高效数据查询
- 支持变量替换，实现面板间的数据联动
- 系统集合和普通集合分离处理

#### 7.1.3 **权限与配置分离（关键设计）**
- **面板配置**存储在 `directus_panels.options`：包含数据源、过滤条件、聚合方式等
- **权限配置**存储在 `directus_permissions`：包含集合权限、过滤条件、字段限制等
- **权限在查询执行时动态应用**：通过 `processAst` 将权限过滤条件注入查询
- **两者合并为最终查询条件**：`WHERE = 用户配置的 filter + 权限配置的 filter`

### 7.2 权限机制总结

#### 7.2.1 权限不是面板独立存储的

| 问题 | 答案 |
|------|------|
| 权限是否存储在面板配置中？ | **否**，权限存储在 `directus_permissions` 表 |
| 权限是否与面板绑定？ | **否**，权限与角色/集合绑定，不与特定面板绑定 |
| 权限何时应用？ | **数据查询执行时**动态应用，不是面板保存时 |
| 能否通过修改面板配置绕过权限？ | **不能**，权限在服务端强制检查 |

#### 7.2.2 权限层级与面板的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限层级与面板的交互                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【模块级权限】                                                   │
│  - 检查：directus_dashboards 的 read 权限                       │
│  - 决定：用户能否进入仪表盘模块                                  │
│  - 时机：模块注册时 (preRegisterCheck)                          │
│                                                                 │
│  ↓                                                               │
│                                                                 │
│  【配置级权限】                                                   │
│  - 检查：directus_dashboards / directus_panels 的 CRUD 权限   │
│  - 决定：用户能否创建/修改/删除仪表盘和面板                      │
│  - 时机：仪表盘/面板 CRUD 操作时                                │
│  - 应用：validateAccess (检查记录权限)                          │
│        processPayload (应用预设)                                │
│                                                                 │
│  ↓                                                               │
│                                                                 │
│  【数据级权限】⭐ 最关键                                         │
│  - 检查：业务集合（如 articles）的 read 权限                    │
│  - 决定：面板能查询哪些数据                                      │
│  - 时机：面板数据查询时 (GraphQL 执行)                          │
│  - 应用：processAst                                              │
│        • 提取权限配置中的 filter                                │
│        • 与面板配置中的 filter 合并                             │
│        • 注入到查询 WHERE 条件中                                │
│                                                                 │
│  【关键】数据级权限是面板数据源权限的核心：                      │
│  - 面板配置的 filter 是"用户想要看什么"                         │
│  - 权限配置的 filter 是"用户能看什么"                           │
│  - 实际查询的是两者的交集                                        │
│  - 无法绕过：权限在服务端强制应用                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 完整流程总结

```
用户配置面板 → 暂存编辑 → 保存到数据库 → 再次进入页面 → 读取面板配置 → 数据查询 → 权限动态应用 → 渲染

         │              │              │              │              │            │              │
         ▼              ▼              ▼              ▼              ▼            ▼              ▼
    ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐  ┌─────────┐   ┌─────────┐
    │ 选择    │   │ edits.  │   │ POST    │   │ 导航    │   │ GET     │  │ POST    │   │ 动态    │
    │ 数据源  │   │ create/ │   │ /panels │   │ 到仪表盘 │   │ /dash-  │  │ /graphql│   │ 组件    │
    │ filter  │   │ update  │   │ 写入    │   │         │   │ boards  │  │ 数据查询│   │ 渲染    │
    │ 聚合等  │   │         │   │ options │   │         │   │ 读取配置│  │ process-│   │         │
    └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘  │ Ast 合  │   └─────────┘
         │              │              │              │              │     │ 并过滤  │
         │              │              │              │              │     │ 条件   │
         ▼              ▼              ▼              ▼              ▼     └─────────┘
    预览数据查询    实时预览效果    权限检查        hydrate()      options   权限与配置
    (权限已应用)   (加载临时数据)   (directus_    重新加载        包含      的 filter
                     (无需保存)      panels)       面板配置        数据源    合并为
                                                      不含权限      filter    WHERE
                                                      权限动态应用            条件
```

### 7.4 设计优势

这种架构设计具有以下优势：

1. **安全性**：权限在服务端强制应用，无法通过修改前端配置绕过
2. **灵活性**：权限配置与面板配置分离，可以独立管理
3. **可维护性**：权限逻辑集中在服务层，易于维护和审计
4. **一致性**：同一用户在不同面板查询同一集合时，应用相同的权限规则
5. **可扩展性**：支持复杂的权限场景（字段级、动态变量、预设值等）

这种架构使得 Directus 的仪表盘系统既强大又灵活，用户可以通过简单的配置实现复杂的数据可视化需求，同时保持系统的安全性和可维护性。