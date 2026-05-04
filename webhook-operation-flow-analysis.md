# Directus Webhook & Flow Operation 数据流分析

## 一、整体架构概览

### 1.1 核心组件

| 组件 | 文件路径 | 职责 |
|------|----------|------|
| **FlowManager** | `api/src/flows.ts` | 管理所有 Flow 和 Operation，负责执行和生命周期管理 |
| **Operations** | `api/src/operations/*/index.ts` | 具体的操作实现（request, condition, trigger 等） |
| **FlowsController** | `api/src/controllers/flows.ts` | Webhook 触发器的 HTTP 入口 |
| **FlowsService** | `api/src/services/flows.ts` | Flow 的 CRUD 操作管理 |

### 1.2 触发器类型

Directus 支持以下 5 种触发器类型 (`TriggerType`)：

```typescript
export type TriggerType = 'event' | 'schedule' | 'operation' | 'webhook' | 'manual';
```

- **event**: 数据事件（如 items.create, items.update, items.delete）
- **schedule**: 定时任务（Cron 表达式）
- **operation**: 被其他 Flow 作为 Operation 调用
- **webhook**: HTTP Webhook 触发
- **manual**: 手动触发

---

## 二、Flow Manager 核心实现

### 2.1 初始化与加载流程

**文件**: `api/src/flows.ts:39-102`

```typescript
export function getFlowManager(): FlowManager {
    if (flowManager) {
        return flowManager;
    }
    flowManager = new FlowManager();
    return flowManager;
}
```

**加载流程 (`load()` 方法)**:

1. 从数据库读取所有 `status = 'active'` 的 Flow
2. 使用 `constructFlowTree()` 构建操作树结构
3. 根据 `flow.trigger` 类型注册不同的处理器：

```
┌─────────────────────────────────────────────────────────────────┐
│                      FlowManager.load()                          │
├─────────────────────────────────────────────────────────────────┤
│  1. 读取活跃 Flow                                                 │
│     ↓                                                             │
│  2. 构建 Flow 树结构                                              │
│     ↓                                                             │
│  3. 根据 trigger 类型注册处理器:                                  │
│     ├── 'event' → emitter.onAction / onFilter                   │
│     ├── 'schedule' → scheduleSynchronizedJob                     │
│     ├── 'operation' → operationFlowHandlers[flow.id]            │
│     ├── 'webhook' → webhookFlowHandlers[`${method}-${id}`]     │
│     └── 'manual' → webhookFlowHandlers[`POST-${id}`]            │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 关键数据结构

**Flow 类型定义** (`packages/types/src/flows.ts`):

```typescript
export interface Flow {
    id: string;
    name: string | null;
    icon: string | null;
    description: string | null;
    status: 'active' | 'inactive';
    trigger: TriggerType | null;
    options: Record<string, any>;      // 触发器配置
    operation: Operation | null;        // 首个操作
    accountability: 'all' | 'activity' | null;  // 审计级别
}

export interface Operation {
    id: string;
    name: string | null;
    key: string;                         // 操作唯一标识，用于数据引用
    type: string;                        // 操作类型: request, condition 等
    options: Record<string, any>;        // 操作配置
    resolve: Operation | null;           // 成功时的下一个操作
    reject: Operation | null;            // 失败时的下一个操作
}
```

---

## 三、Payload 组装机制

### 3.1 核心数据容器

**文件**: `api/src/flows.ts:393-402`

Flow 执行时，所有数据存储在 `keyedData` 对象中：

```typescript
const keyedData: Record<string, unknown> = {
    [TRIGGER_KEY]: data,           // '$trigger' - 触发时的数据
    [LAST_KEY]: data,               // '$last' - 上一个操作的结果
    [ACCOUNTABILITY_KEY]: context?.['accountability'] ?? null,  // '$accountability'
    [ENV_KEY]: this.envs,           // '$env' - 环境变量（需在 FLOWS_ENV_ALLOW_LIST 中）
};
```

### 3.2 模板变量替换 (`applyOptionsData`)

**文件**: `packages/utils/shared/apply-options-data.ts`

这是最核心的 payload 组装函数，用于将操作配置中的模板变量替换为实际数据。

**实现原理**:

```typescript
export function applyOptionsData(
    options: Record<string, any>,
    data: Record<string, any>,
    skipUndefinedKeys: string[] = [],
): Record<string, any> {
    return Object.fromEntries(
        Object.entries(options).map(([key, value]) => 
            [key, renderMustache(value, data, skipUndefinedKeys.includes(key))]
        ),
    );
}
```

**支持的模板语法**:

| 语法 | 说明 | 示例 |
|------|------|------|
| `{{ $trigger }}` | 完整引用对象 | 返回整个 trigger 数据 |
| `{{ $trigger.payload }}` | 嵌套路径访问 | 访问 trigger 中的 payload 字段 |
| `{{ $last }}` | 上一个操作的结果 | 获取最近操作的返回值 |
| `{{ operation_key }}` | 指定操作的结果 | 使用操作的 key 访问历史数据 |

**特殊处理**:
- 如果字符串完全匹配 `{{ path }}` 格式，直接返回对应值（保持类型）
- 否则使用 `micromustache` 进行字符串渲染
- 对象值会被自动 `JSON.stringify`

### 3.3 数据事件触发器的 Payload

**文件**: `api/src/flows.ts:172-228`

当 `trigger === 'event'` 时，根据事件类型组装不同的 payload：

**Action 事件（items.create, items.update, items.delete）**:

```typescript
const handler: ActionHandler = (meta, context) =>
    this.executeFlow(flow, meta, {
        accountability: context['accountability'],
        database: getDatabase(),
        getSchema: context['schema'] ? () => context['schema'] : getSchema,
    });
```

**Action 事件的 meta 数据结构**:
```typescript
{
    payload: any,           // 变更的数据
    keys: PrimaryKey[],     // 受影响的主键
    collection: string,     // 集合名称
    // ... 其他元数据
}
```

**Filter 事件**（在操作执行前触发，可修改 payload）:

```typescript
const handler: FilterHandler = (payload, meta, context) =>
    this.executeFlow(
        flow,
        { payload, ...meta },  // 组合 payload 和 meta
        { ...context }
    );
```

---

## 四、条件判断机制

### 4.1 Condition Operation 实现

**文件**: `api/src/operations/condition/index.ts`

```typescript
export default defineOperationApi<Options>({
    id: 'condition',

    handler: ({ filter }, { data, accountability }) => {
        // 解析过滤器（支持动态变量和权限检查）
        const parsedFilter = parseFilter(filter, accountability, undefined, true);

        if (!parsedFilter) {
            return null;  // 无过滤条件，视为通过
        }

        // 验证数据是否符合过滤条件
        const errors = validatePayload(parsedFilter, data, { requireAll: true });

        if (errors.length > 0) {
            // 验证失败，抛出错误 → 走 reject 分支
            throw validationErrors;
        } else {
            // 验证通过 → 走 resolve 分支
            return null;
        }
    },
});
```

### 4.2 过滤规则语法 (`parseFilter`)

Condition 操作使用 Directus 标准的 Filter 语法，支持：

- **逻辑运算**: `_and`, `_or`
- **比较运算**: `_eq`, `_neq`, `_gt`, `_gte`, `_lt`, `_lte`, `_in`, `_contains` 等
- **动态变量**: 支持 `{{ $trigger }}`, `{{ $last }}` 等模板语法

**示例 Filter 配置**:
```json
{
    "_and": [
        {
            "$trigger.payload.status": {
                "_eq": "published"
            }
        },
        {
            "$last.data": {
                "_contains": "keyword"
            }
        }
    ]
}
```

### 4.3 操作执行流程中的条件分支

**文件**: `api/src/flows.ts:506-600`

每个操作执行后，根据结果状态决定下一个操作：

```typescript
private async executeOperation(operation, keyedData, context): Promise<{
    successor: Operation | null;
    status: 'resolve' | 'reject' | 'unknown';
    data: unknown;
    options: Record<string, any> | null;
}> {
    try {
        // 1. 应用模板变量替换
        options = applyOptionsData(options, optionData);

        // 2. 执行操作处理器
        let result = await handler(options, { ...context });

        // 3. 序列化验证（确保结果可 JSON 序列化）
        JSON.stringify(result ?? null);

        // 4. 替换 undefined 为 null（JSON 不支持 undefined）
        if (typeof result === 'object' && result !== null) {
            result = deepMap(result, (value) => value === undefined ? null : value);
        }

        // 成功 → 返回 resolve 分支
        return { successor: operation.resolve, status: 'resolve', data: result ?? null, options };
    } catch (error) {
        // 错误处理
        let data;
        if (error instanceof Error) {
            delete error.stack;  // 不暴露堆栈
            data = error;
        } else if (typeof error === 'string') {
            data = isValidJSON(error) ? parseJSON(error) : error;
        } else {
            data = error ?? null;
        }

        // 失败 → 返回 reject 分支
        return { successor: operation.reject, status: 'reject', data, options };
    }
}
```

**执行流程图**:

```
┌────────────────────────────────────────────────────────────────────┐
│                    executeOperation() 执行流程                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ┌──────────────┐                                                  │
│   │  1. 模板替换  │  applyOptionsData(options, keyedData)          │
│   └──────┬───────┘                                                  │
│          ↓                                                           │
│   ┌──────────────┐                                                  │
│   │ 2. 执行操作   │  handler(options, context)                      │
│   └──────┬───────┘                                                  │
│          ↓                                                           │
│   ┌──────────────────┐                                               │
│   │ 3. 序列化验证     │  JSON.stringify(result)                      │
│   └────────┬─────────┘                                               │
│            ↓                                                          │
│    ┌───────┴───────┐                                                 │
│    │   成功?        │                                                 │
│    └───────┬───────┘                                                 │
│       Yes /   \ No                                                   │
│           /     \                                                     │
│   ┌────────┐   ┌────────┐                                            │
│   │ resolve│   │ reject │                                            │
│   │ 分支   │   │ 分支   │                                            │
│   └────┬───┘   └────┬───┘                                            │
│        ↓             ↓                                                 │
│   operation.resolve  operation.reject                                 │
│   (下一个操作)       (错误处理操作)                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 五、外部服务调用 (Webhook/Request)

### 5.1 Request Operation 实现

**文件**: `api/src/operations/request/index.ts`

这是用于调用外部 HTTP 服务的核心操作：

```typescript
type Options = {
    url: string;
    method: string;
    body: Record<string, any> | string | null;
    headers?: { header: string; value: string }[] | null;
};

export default defineOperationApi<Options>({
    id: 'request',

    handler: async ({ url, method, body, headers }) => {
        // 1. 构建自定义请求头
        const customHeaders =
            headers?.reduce(
                (acc, { header, value }) => {
                    acc[header] = value;
                    return acc;
                },
                {} as Record<string, string>,
            ) ?? {};

        // 2. 自动设置 Content-Type（如果是 JSON body）
        if (!customHeaders['Content-Type'] && 
            (typeof body === 'object' || isValidJSON(body))) {
            customHeaders['Content-Type'] = 'application/json';
        }

        // 3. 使用 Axios 发送请求
        const axios = await getAxios();

        try {
            const result = await axios({
                url: encodeUrl(url),
                method,
                data: body,
                headers: customHeaders,
            });

            // 4. 返回响应数据
            return { 
                status: result.status, 
                statusText: result.statusText, 
                headers: result.headers, 
                data: result.data 
            };
        } catch (error: unknown) {
            // 5. 错误处理 - 将 HTTP 错误转换为可序列化格式
            if (isAxiosError(error) && error.response) {
                throw JSON.stringify({
                    status: error.response.status,
                    statusText: error.response.statusText,
                    headers: error.response.headers,
                    data: error.response.data,
                });
            } else {
                throw error;
            }
        }
    },
});
```

### 5.2 Webhook 触发器的 HTTP 入口

**文件**: `api/src/controllers/flows.ts:18-46`

Webhook 触发器的路由和数据组装：

```typescript
const webhookFlowHandler = asyncHandler(async (req, res, next) => {
    const flowManager = getFlowManager();

    // 组装 trigger 数据
    const { result, cacheEnabled } = await flowManager.runWebhookFlow(
        `${req.method}-${req.params['pk']}`,  // 路由标识符: GET-xxx / POST-xxx
        {
            path: req.path,           // 请求路径
            query: req.query,         // URL 查询参数
            body: req.body,           // 请求体
            method: req.method,       // HTTP 方法
            headers: req.headers,     // 请求头
        },
        {
            accountability: req.accountability,
            schema: req.schema,
        },
    );

    // 缓存控制
    if (!cacheEnabled) {
        res.locals['cache'] = false;
    }

    // 返回结果
    res.locals['payload'] = result;
    return next();
});

// 注册路由
router.get(`/trigger/:pk(${UUID_REGEX})`, webhookFlowHandler, respond);
router.post(`/trigger/:pk(${UUID_REGEX})`, webhookFlowHandler, respond);
```

### 5.3 Webhook Flow 处理器

**文件**: `api/src/flows.ts:247-263`

```typescript
} else if (flow.trigger === 'webhook') {
    const method = flow.options?.['method'] ?? 'GET';

    const handler = async (data: unknown, context: Record<string, unknown>) => {
        let cacheEnabled = true;

        // GET 请求支持缓存配置
        if (method === 'GET') {
            cacheEnabled = flow.options['cacheEnabled'] !== false;
        }

        // 异步执行模式
        if (flow.options['async']) {
            this.executeFlow(flow, data, context);  // 不等待
            return { result: undefined, cacheEnabled };
        } else {
            // 同步执行模式
            return { result: await this.executeFlow(flow, data, context), cacheEnabled };
        }
    };

    // Webhook 默认返回 $last
    flow.options['return'] = flow.options['return'] ?? '$last';

    this.webhookFlowHandlers[`${method}-${flow.id}`] = handler;
}
```

---

## 六、结果记录与审计

### 6.1 执行结果存储机制

**文件**: `api/src/flows.ts:416-482`

Flow 执行完成后，根据 `flow.accountability` 配置记录执行结果：

```typescript
private async executeFlow(flow: Flow, data: unknown = null, context: Record<string, unknown> = {}): Promise<unknown> {
    // ... 执行流程 ...

    // 记录执行步骤
    const steps: {
        operation: string;
        key: string;
        status: 'resolve' | 'reject' | 'unknown';
        options: Record<string, any> | null;
    }[] = [];

    while (nextOperation !== null) {
        const { successor, data, status, options } = await this.executeOperation(nextOperation, keyedData, context);
        
        // 保存操作结果到 keyedData
        keyedData[nextOperation.key] = data;
        keyedData[LAST_KEY] = data;
        lastOperationStatus = status;
        
        // 记录步骤
        steps.push({ operation: nextOperation!.id, key: nextOperation.key, status, options });

        nextOperation = successor;
    }

    // 根据 accountability 配置记录审计日志
    if (flow.accountability !== null) {
        const activityService = new ActivityService({
            knex: database,
            schema: schema,
        });

        const accountability = context?.['accountability'] as Accountability | undefined;

        // 1. 创建 Activity 记录
        const activity = await activityService.createOne({
            action: Action.RUN,
            user: accountability?.user ?? null,
            collection: 'directus_flows',
            ip: accountability?.ip ?? null,
            user_agent: accountability?.userAgent ?? null,
            origin: accountability?.origin ?? null,
            item: flow.id,
        });

        // 2. 如果 accountability === 'all'，创建详细的 Revision 记录
        if (flow.accountability === 'all') {
            const revisionsService = new RevisionsService({
                knex: database,
                schema: schema,
            });

            await revisionsService.createOne({
                activity: activity,
                collection: 'directus_flows',
                item: flow.id,
                data: {
                    // 记录执行步骤（脱敏处理）
                    steps: steps.map((step) => 
                        redactObject(step, { values: this.envs }, getRedactedString)
                    ),
                    // 记录完整数据（敏感信息脱敏）
                    data: redactObject(
                        keyedData,
                        {
                            keys: [
                                ['**', 'headers', 'authorization'],
                                ['**', 'headers', 'cookie'],
                                ['**', 'query', 'access_token'],
                                ['**', 'payload', 'password'],
                                ['**', 'payload', 'token'],
                                ['**', 'payload', 'tfa_secret'],
                                ['**', 'payload', 'external_identifier'],
                                ['**', 'payload', 'auth_data'],
                                ['**', 'payload', 'credentials'],
                                ['**', 'payload', 'ai_openai_api_key'],
                                ['**', 'payload', 'ai_anthropic_api_key'],
                                ['**', 'payload', 'ai_google_api_key'],
                                ['**', 'payload', 'ai_openai_compatible_api_key'],
                            ],
                            values: this.envs,
                        },
                        getRedactedString,
                    ),
                },
            });
        }
    }

    // ... 返回结果 ...
}
```

### 6.2 Accountability 级别说明

| 级别 | 说明 | 记录内容 |
|------|------|----------|
| `null` | 不记录 | 无任何记录 |
| `'activity'` | 仅记录活动 | 在 `directus_activity` 表记录基本执行信息 |
| `'all'` | 完整记录 | Activity + Revision（包含所有步骤和数据，脱敏后） |

### 6.3 敏感信息脱敏规则

**文件**: `api/src/utils/redact-object.ts`

在记录 Revision 时，以下敏感信息会被替换为 `[REDACTED]`：

**按路径匹配的 keys**:
- `**.headers.authorization`
- `**.headers.cookie`
- `**.query.access_token`
- `**.payload.password`
- `**.payload.token`
- `**.payload.tfa_secret`
- `**.payload.external_identifier`
- `**.payload.auth_data`
- `**.payload.credentials`
- `**.payload.ai_*_api_key`

**按值匹配的 values**:
- 环境变量值（`$env` 中的值）

### 6.4 返回值配置

Flow 的返回值由 `flow.options['return']` 控制：

```typescript
if (flow.options['return'] === '$all') {
    return keyedData;  // 返回所有数据
} else if (flow.options['return']) {
    return get(keyedData, flow.options['return']);  // 返回指定路径的数据
}

return undefined;  // 默认返回 undefined
```

**Webhook 和 Manual 触发器的默认返回值**:
```typescript
flow.options['return'] = flow.options['return'] ?? '$last';  // 默认返回最后一个操作的结果
```

---

## 七、完整执行流程图

### 7.1 数据事件 (Event Trigger) 完整流程

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    items.create / items.update / items.delete               │
└──────────────────────────────────────┬─────────────────────────────────────┘
                                       │
                                       ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                         emitter.emitAction() / emitFilter()                 │
│                         (api/src/emitter.ts)                                 │
└──────────────────────────────────────┬─────────────────────────────────────┘
                                       │
                                       ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    FlowManager 注册的事件处理器                               │
│                    (api/src/flows.ts:196-228)                               │
│                                                                               │
│  Action Handler:                                                              │
│  (meta, context) => executeFlow(flow, meta, context)                        │
│                                                                               │
│  Filter Handler:                                                              │
│  (payload, meta, context) => executeFlow(flow, {payload, ...meta}, context)│
└──────────────────────────────────────┬─────────────────────────────────────┘
                                       │
                                       ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                        executeFlow() 执行流程                                 │
│                        (api/src/flows.ts:393-504)                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 初始化 keyedData:                                                        │
│     {                                                                        │
│       $trigger: data,           // 触发事件的数据                            │
│       $last: data,              // 上一个操作结果（初始为 trigger 数据）      │
│       $accountability: {...},   // 用户权限信息                              │
│       $env: {...}               // 允许的环境变量                            │
│     }                                                                        │
│                                                                              │
│  2. 循环执行操作链:                                                           │
│     while (nextOperation !== null) {                                        │
│         result = executeOperation(nextOperation, keyedData, context)       │
│         keyedData[operation.key] = result.data  // 保存到操作 key           │
│         keyedData[$last] = result.data       // 更新 $last                  │
│         nextOperation = result.successor     // 根据状态决定下一个操作        │
│     }                                                                        │
│                                                                              │
│  3. 记录审计日志 (如果 accountability !== null):                             │
│     - 创建 Activity 记录                                                      │
│     - 如果 accountability === 'all'，创建 Revision 记录（脱敏后）            │
│                                                                              │
│  4. 返回结果 (根据 flow.options['return']):                                  │
│     - '$all' → 返回完整 keyedData                                            │
│     - 其他路径 → 使用 micromustache.get() 获取指定值                         │
│                                                                              │
└──────────────────────────────────────┬─────────────────────────────────────┘
```

### 7.2 Webhook Trigger 完整流程

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    HTTP 请求: GET/POST /flows/trigger/:id                    │
└──────────────────────────────────────┬─────────────────────────────────────┘
                                       │
                                       ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    webhookFlowHandler() 控制器                               │
│                    (api/src/controllers/flows.ts:18-46)                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  组装 $trigger 数据:                                                          │
│  {                                                                           │
│    path: req.path,           // 请求路径                                     │
│    query: req.query,         // 查询参数                                     │
│    body: req.body,           // 请求体                                       │
│    method: req.method,       // HTTP 方法                                    │
│    headers: req.headers      // 请求头                                       │
│  }                                                                           │
│                                                                              │
└──────────────────────────────────────┬─────────────────────────────────────┘
                                       │
                                       ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    FlowManager.runWebhookFlow()                              │
│                    (api/src/flows.ts:133-150)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  查找对应的 handler:                                                          │
│  key = `${method}-${flow.id}`  // 如 "GET-abc123" 或 "POST-def456"        │
│  handler = webhookFlowHandlers[key]                                          │
│                                                                              │
└──────────────────────────────────────┬─────────────────────────────────────┘
                                       │
                                       ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    Webhook Handler 执行                                       │
│                    (api/src/flows.ts:250-263)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  检查配置:                                                                    │
│  - flow.options['async']: 是否异步执行                                        │
│  - flow.options['cacheEnabled']: GET 请求是否启用缓存                        │
│                                                                              │
│  异步模式:                                                                    │
│  if (async) {                                                                │
│      this.executeFlow(flow, data, context);  // 不等待，立即返回            │
│      return { result: undefined, cacheEnabled };                            │
│  }                                                                           │
│                                                                              │
│  同步模式:                                                                    │
│  else {                                                                      │
│      return { result: await executeFlow(...), cacheEnabled };               │
│  }                                                                           │
│                                                                              │
└──────────────────────────────────────┬─────────────────────────────────────┘
                                       │
                                       ▼
                         (同 Event Trigger 的 executeFlow 流程)
```

---

## 八、关键代码位置索引

| 功能模块 | 文件路径 | 关键方法/行号 |
|----------|----------|---------------|
| **Flow 管理器** | `api/src/flows.ts` | |
| ├─ 初始化加载 | | `load()` (156-366) |
| ├─ 执行 Flow | | `executeFlow()` (393-504) |
| ├─ 执行 Operation | | `executeOperation()` (506-600) |
| ├─ Event 触发器注册 | | 172-228 |
| ├─ Webhook 触发器注册 | | 247-269 |
| └─ 审计记录 | | 427-482 |
| **Operations** | `api/src/operations/*/index.ts` | |
| ├─ Request (HTTP 调用) | `request/index.ts` | |
| ├─ Condition (条件判断) | `condition/index.ts` | |
| ├─ Trigger (触发其他 Flow) | `trigger/index.ts` | |
| └─ 其他操作 | `*/index.ts` | |
| **工具函数** | | |
| ├─ 模板替换 | `packages/utils/shared/apply-options-data.ts` | `applyOptionsData()` |
| ├─ 敏感信息脱敏 | `api/src/utils/redact-object.ts` | `redactObject()` |
| **控制器** | | |
| ├─ Webhook 入口 | `api/src/controllers/flows.ts` | `webhookFlowHandler` (18-46) |
| **类型定义** | | |
| └─ Flow/Operation | `packages/types/src/flows.ts` | |

---

## 九、配置示例

### 9.1 数据事件触发 Webhook 通知

**场景**: 当 `articles` 表有新文章创建时，自动通知 Slack。

**Flow 配置**:
```typescript
{
    trigger: 'event',
    options: {
        type: 'action',
        scope: ['items.create'],
        collections: ['articles']
    },
    accountability: 'activity',  // 仅记录活动
    operation: {
        key: 'notify_slack',
        type: 'request',
        options: {
            method: 'POST',
            url: 'https://hooks.slack.com/services/xxx/yyy/zzz',
            body: {
                text: '新文章发布: {{ $trigger.payload.title }}'
            },
            headers: [
                { header: 'Content-Type', value: 'application/json' }
            ]
        },
        resolve: null,
        reject: null
    }
}
```

### 9.2 Webhook 触发带条件判断的数据处理

**场景**: 接收外部 Webhook，验证数据后更新数据库。

**Flow 配置**:
```typescript
{
    trigger: 'webhook',
    options: {
        method: 'POST',
        async: false,
        cacheEnabled: false,
        error_on_reject: true,
        return: '$last'
    },
    accountability: 'all',  // 完整记录
    operation: {
        key: 'validate_input',
        type: 'condition',
        options: {
            filter: {
                '$trigger.body.secret': { _eq: 'my-secret-key' }
            }
        },
        resolve: {  // 验证通过
            key: 'update_record',
            type: 'item-update',
            options: {
                collection: 'products',
                key: '{{ $trigger.body.product_id }}',
                payload: {
                    status: 'processed',
                    external_id: '{{ $trigger.body.external_id }}'
                }
            },
            resolve: null,
            reject: null
        },
        reject: {  // 验证失败
            key: 'throw_error',
            type: 'throw-error',
            options: {
                message: 'Invalid secret key'
            },
            resolve: null,
            reject: null
        }
    }
}
```

---

## 十、注意事项

### 10.1 数据安全

1. **环境变量访问**: 只有在 `FLOWS_ENV_ALLOW_LIST` 中配置的环境变量才能通过 `$env` 访问
2. **敏感信息脱敏**: 记录 Revision 时，密码、token、API key 等敏感信息会被自动脱敏
3. **权限检查**: Manual 触发器会检查用户对目标集合的读取权限

### 10.2 性能考虑

1. **异步执行**: Webhook 和 Manual 触发器支持 `async: true` 配置，可避免阻塞 HTTP 请求
2. **缓存控制**: GET Webhook 支持 `cacheEnabled` 配置，可利用 HTTP 缓存
3. **序列化验证**: 所有操作结果必须可 JSON 序列化，否则会失败

### 10.3 调试建议

1. 使用 `log` operation 输出调试信息（会自动脱敏敏感数据）
2. 设置 `accountability: 'all'` 查看完整的执行历史和数据
3. 注意 `error_on_reject` 配置：设为 `true` 时，最后一个操作 reject 会抛出错误

---

## 十一、相关测试文件

| 测试文件 | 覆盖范围 |
|----------|----------|
| `api/src/flows.test.ts` | Flow 管理器核心逻辑 |
| `api/src/operations/request/index.test.ts` | HTTP 请求操作 |
| `api/src/operations/condition/index.test.ts` | 条件判断操作 |
| `api/src/operations/trigger/index.test.ts` | 触发其他 Flow 操作 |
| `packages/utils/shared/apply-options-data.test.ts` | 模板替换工具函数 |
