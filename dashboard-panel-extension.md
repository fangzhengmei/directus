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

### 3.1 前端权限检查

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

### 3.2 项目级权限检查

**Composable**：`app/src/composables/use-permissions/item/use-item-permissions.ts`

对于具体项目的权限检查，使用 `useItemPermissions`：

1. **权限获取**：
   - 调用 `fetchItemPermissions` 获取项目级权限
   - 检查 `update`、`delete`、`share`、`archive` 等操作权限

2. **字段级权限**：
   - 通过 `getFields` 获取可访问的字段列表

**核心代码**：
```typescript
// app/src/composables/use-permissions/item/use-item-permissions.ts:19-37
export function useItemPermissions(
    collection: Collection,
    primaryKey: PrimaryKey,
    isNew: IsNew,
    isVersion: MaybeRef<boolean> = false,
): UsableItemPermissions {
    const { loading, fetchedItemPermissions, refresh } = fetchItemPermissions(collection, primaryKey);

    const updateAllowed = isActionAllowed(collection, isNew, fetchedItemPermissions, 'update', isVersion);
    const deleteAllowed = isActionAllowed(collection, isNew, fetchedItemPermissions, 'delete');
    const shareAllowed = isActionAllowed(collection, isNew, fetchedItemPermissions, 'share');
    const archiveAllowed = isArchiveAllowed(collection, updateAllowed);
    const fields = getFields(collection, isNew, fetchedItemPermissions, isVersion);

    return { loading, refresh, updateAllowed, deleteAllowed, shareAllowed, archiveAllowed, fields };
}
```

### 3.3 后端权限校验

**核心机制**：`accountability` 对象

后端通过 `accountability` 对象进行权限校验，该对象包含：

- `user`：用户信息
- `role`：角色信息
- `permissions`：权限配置
- `ip`：请求 IP
- `admin`：是否为管理员

**服务层权限应用**：

`PanelsService` 继承自 `ItemsService`，自动应用权限校验：

```typescript
// api/src/services/panels.ts:4-7
export class PanelsService extends ItemsService {
    constructor(options: AbstractServiceOptions) {
        super('directus_panels', options);
    }
}
```

**控制器中的使用**：

```typescript
// api/src/controllers/panels.ts:16-53
router.post(
    '/',
    asyncHandler(async (req, res, next) => {
        const service = new PanelsService({
            accountability: req.accountability,
            schema: req.schema,
        });

        // 调用服务方法时会自动应用权限校验
        if (Array.isArray(req.body)) {
            const keys = await service.createMany(req.body);
            savedKeys.push(...keys);
        } else {
            const key = await service.createOne(req.body);
            savedKeys.push(key);
        }
        // ...
    }),
    respond,
);
```

### 3.4 模块访问权限

**模块定义**：`app/src/modules/insights/index.ts`

模块级别通过 `preRegisterCheck` 进行权限检查：

```typescript
// app/src/modules/insights/index.ts:42-49
preRegisterCheck(user, permissions) {
    const admin = user.admin_access;

    if (admin) return true;

    const access = permissions['directus_dashboards']?.['read']?.access;
    return access === 'partial' || access === 'full';
},
```

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
| `options` | json | 面板特定配置选项 |
| `dashboard` | uuid | 所属仪表盘 ID |
| `date_created` | timestamp | 创建时间 |
| `user_created` | uuid | 创建用户 |

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
    loadPanelData(panel);
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

## 5. 完整协作流程图

### 5.1 面板配置与保存流程

```
用户配置面板
    ↓
前端: panel-configuration.vue
    ↓ 选择面板类型、配置选项
stagePanelCreate / stagePanelUpdate
    ↓
edits.create / edits.update (暂存)
    ↓ 实时预览（如果查询变化）
loadPanelData
    ↓ 准备查询
prepareQuery → 调用面板的 query() 函数
    ↓ 应用变量替换
applyOptionsData
    ↓ 转换为 GraphQL
queryToGqlString
    ↓ 发送请求
POST /graphql
    ↓
后端: GraphQLService
    ↓ 权限校验
accountability 传递到 ItemsService
    ↓ 执行查询
readByQuery / readOne
    ↓ 返回数据
    ↓
前端: data.value[panel.id] = 结果
    ↓ 面板组件渲染
<component :is="panel-${type}" :data="data" />
    ↓ 用户点击保存
saveChanges
    ↓
POST /panels (创建)
PATCH /panels (更新)
DELETE /panels (删除)
    ↓
后端: PanelsService → ItemsService
    ↓ 权限校验
accountability 检查
    ↓ 数据库操作
directus_panels 表 CRUD
    ↓ 返回结果
    ↓
前端: hydrate() 重新加载
    ↓
clearEdits() 清空暂存
```

### 5.2 权限校验流程

```
用户请求操作
    ↓
前端权限检查 (可选)
    ├── usePermissionsStore.hasPermission()
    └── useItemPermissions.updateAllowed
    ↓
API 请求
    ↓
后端中间件
    ├── 解析 accountability (用户、角色、权限)
    └── 传递到服务层
    ↓
服务层 (PanelsService → ItemsService)
    ↓ 权限校验
    ├── 检查 collection 权限
    ├── 检查 action 权限 (create/read/update/delete)
    ├── 应用权限预设 (presets)
    └── 字段级权限过滤
    ↓
数据库操作
    ↓ 返回结果
    ↓
前端处理响应
```

### 5.3 变量面板与依赖面板协作流程

```
用户设置变量面板值
    ↓
setVariable(field, value)
    ↓
variables.value[field] = value
    ↓ 查找依赖此变量的面板
regex = /{{\s*field\s*}}/
    ↓ 检查查询是否变化
oldQuery vs newQuery (应用新变量值)
    ↓ 如果变化
loadPanelData(dependentPanels)
    ↓ 重新查询数据
    ↓
依赖面板重新渲染
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

### 6.3 后端服务
| 功能 | 文件路径 |
|------|----------|
| 面板服务 | `api/src/services/panels.ts` |
| 面板控制器 | `api/src/controllers/panels.ts` |
| GraphQL 服务 | `api/src/services/graphql/index.ts` |
| GraphQL 控制器 | `api/src/controllers/graphql.ts` |
| 扩展管理器 | `api/src/extensions/manager.ts` |

### 6.4 数据模型
| 功能 | 文件路径 |
|------|----------|
| 面板表字段 | `packages/system-data/src/fields/panels.yaml` |
| 面板 SDK 类型 | `sdk/src/schema/panel.ts` |

---

## 7. 总结

Directus 仪表盘面板扩展机制是一个设计完善的系统，具有以下特点：

1. **模块化扩展**：
   - 面板通过 `definePanel` 定义，支持完全自定义
   - 扩展自动加载和注册，无需手动配置
   - 支持内部面板和自定义面板的无缝集成

2. **灵活的数据查询**：
   - 每个面板定义自己的 `query()` 函数，生成查询配置
   - 使用 GraphQL 进行高效数据查询
   - 支持变量替换，实现面板间的数据联动
   - 系统集合和普通集合分离处理

3. **多层权限校验**：
   - 前端：预检查权限，优化用户体验
   - 后端：服务层强制权限校验，确保安全
   - 模块级、集合级、项目级、字段级多层权限控制
   - `accountability` 对象贯穿整个请求生命周期

4. **完善的编辑体验**：
   - 暂存编辑机制，支持撤销和预览
   - 实时数据刷新，配置变化立即反映
   - 缓存管理，优化性能
   - 错误边界处理，提升稳定性

5. **数据驱动渲染**：
   - 面板配置完全存储在数据库
   - 动态组件渲染，支持运行时切换面板类型
   - 变量和选项分离，支持灵活配置

这种架构使得 Directus 的仪表盘系统既强大又灵活，用户可以通过简单的配置实现复杂的数据可视化需求，同时保持系统的安全性和可维护性。
