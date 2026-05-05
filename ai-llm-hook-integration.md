# Directus AI 字段与 LLM 钩子集成机制

本文档详细说明 Directus 中 AI 字段（如自动翻译、生成描述等）的完整工作流程，包括 Admin 界面触发、API 路由到 LLM 服务、以及 LLM 结果通过钩子写回数据集合的机制。

---

## 目录

1. [整体架构概览](#1-整体架构概览)
2. [Admin 界面触发机制](#2-admin-界面触发机制)
3. [API 路由到 LLM 服务](#3-api-路由到-llm-服务)
4. [LLM 结果通过钩子/流程写回数据](#4-llm-结果通过钩子流程写回数据)
5. [配置示例](#5-配置示例)
6. [关键代码引用](#6-关键代码引用)

---

## 1. 整体架构概览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              Admin 界面 (Vue 3 App)                           │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────────┐     │
│  │ AI 侧边栏   │───▶│ use-ai.ts   │───▶│ DefaultChatTransport        │     │
│  │ (ai-conversation.vue) │         │ (状态管理)  │  (API 通信层)         │     │
│  └─────────────┘    └─────────────┘    └─────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼ POST /ai/chat 或 /ai/object
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API 层 (Node.js / Express)                        │
│  ┌─────────────────────┐    ┌─────────────────────────────────────────────┐ │
│  │ aiRouter            │───▶│ 控制器层                                     │ │
│  │ - /ai/chat          │    │ - aiChatPostHandler (流式对话)              │ │
│  │ - /ai/object        │    │ - aiObjectPostHandler (结构化对象生成)      │ │
│  │ - /ai/files         │    └─────────────────────────────────────────────┘ │
│  └─────────────────────┘                          │                          │
│                                                    ▼                          │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                        AI 提供商注册表 (Provider Registry)              │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────────┐   │ │
│  │  │ OpenAI   │ │ Anthropic│ │  Google  │ │ OpenAI Compatible    │   │ │
│  │  │ (GPT)    │ │ (Claude) │ │ (Gemini) │ │ (自定义 LLM)         │   │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                    │                          │
│                                                    ▼                          │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                        AI 工具系统 (Tool System)                        │ │
│  │  - items: 数据项 CRUD 操作                                               │ │
│  │  - trigger-flow: 触发手动流程                                            │ │
│  │  - files/folders: 文件/文件夹操作                                        │ │
│  │  - collections/fields: 集合/字段管理                                     │ │
│  │  - flows/operations: 流程/操作管理                                       │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                    │                          │
│                                                    ▼                          │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                        流程引擎 (Flow Manager)                           │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐   │ │
│  │  │ 事件触发器   │  │ 手动触发器   │  │ 操作执行器                   │   │ │
│  │  │ (filter/    │  │ (webhook/   │  │ - item-update: 更新数据     │   │ │
│  │  │  action)    │  │  manual)    │  │ - item-create: 创建数据     │   │ │
│  │  │             │  │             │  │ - request: 调用外部 API     │   │ │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────────┘   │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              数据库层                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────────┐     │
│  │ ItemsService│───▶│   Knex.js   │───▶│  PostgreSQL / MySQL / etc   │     │
│  │  (业务逻辑)  │    │ (ORM/查询)  │    │      (数据持久化)           │     │
│  └─────────────┘    └─────────────┘    └─────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Admin 界面触发机制

### 2.1 AI 侧边栏组件

Directus 的 Admin 界面提供了一个 AI 侧边栏，用户可以通过以下方式触发 AI 功能：

**核心组件位置**：
- `app/src/ai/components/ai-conversation.vue` - 主对话组件
- `app/src/ai/components/ai-input.vue` - 输入框组件
- `app/src/ai/components/ai-magic-button.vue` - 魔法按钮（视觉图标）

### 2.2 状态管理：use-ai.ts

AI 功能的核心状态管理由 `use-ai.ts` store 实现：

```typescript
// app/src/ai/stores/use-ai.ts:230-279
const transport = new DefaultChatTransport({
    api: '/ai/chat',  // 调用 /ai/chat API 端点
    credentials: 'include',
    body: () => {
        const tools = [...toolsStore.enabledSystemTools, ...toolsStore.localTools.map(toApiTool)];
        
        // 工具审批配置
        const approvals: Record<string, 'always' | 'ask'> = {};
        for (const [toolName, mode] of Object.entries(toolsStore.toolApprovals)) {
            if (mode === 'always' || mode === 'ask') {
                approvals[toolName] = mode;
            }
        }

        // 构建上下文
        const context = {
            page: currentPageContext.value,  // 当前页面信息（集合、条目等）
            ...(pendingContextSnapshot.value.length > 0 && { 
                attachments: pendingContextSnapshot.value 
            }),
        };

        return {
            provider: selectedModel.value?.provider,  // LLM 提供商 (openai/anthropic/google)
            model: selectedModel.value?.model,        // 具体模型 (gpt-4, claude-3, etc.)
            tools,                                      // 可用工具列表
            toolApprovals: approvals,                  // 工具审批配置
            context,                                    // 上下文信息
        };
    },
    // ...
});
```

### 2.3 聊天实例与自动工具调用

```typescript
// app/src/ai/stores/use-ai.ts:281-341
const chat = new Chat<UIMessage>({
    messages: storedMessages.value,
    transport,
    // 当工具调用完成时自动发送新请求
    sendAutomaticallyWhen: ({ messages }) =>
        lastAssistantMessageIsCompleteWithToolApprovalResponses({ messages }) ||
        lastAssistantMessageIsCompleteWithToolCalls({ messages }),
    
    // 工具调用回调
    onToolCall: async ({ toolCall }) => {
        const isServerTool = toolCall.dynamic || toolsStore.isSystemTool(toolCall.toolName);

        if (isServerTool) {
            return;  // 服务器端工具，由后端处理
        }

        // 客户端本地工具执行
        const tool = toolsStore.localTools.find((t) => t.name === toolCall.toolName);
        if (tool) {
            const output = await tool.execute(toolCall.input as Record<string, unknown>);
            chat.addToolResult({ tool: toolCall.toolName, output, toolCallId: toolCall.toolCallId });
        }
    },
});
```

### 2.4 触发流程

**用户操作**：
1. 用户打开 AI 侧边栏（右上角 AI 图标）
2. 在输入框中输入提示词（如："将标题翻译成英文"、"为这个产品生成描述"）
3. 点击发送按钮或按回车

**前端处理**：
1. `use-ai.ts` 的 `submit()` 方法被调用
2. 准备上下文数据（当前集合、当前条目、已选中的上下文）
3. 使用 `DefaultChatTransport` 发送 POST 请求到 `/ai/chat`
4. 接收流式响应并渲染到界面

---

## 3. API 路由到 LLM 服务

### 3.1 API 路由注册

AI 相关的 API 端点在 `app.ts` 中注册：

```typescript
// api/src/app.ts:14-17, 348-353
import { aiRouter } from './ai/chat/router.js';
import { aiFilesRouter } from './ai/files/router.js';

// ...

if (toBoolean(env['AI_ENABLED']) === true) {
    app.use('/ai', aiRouter);        // 主要 AI 路由
    app.use('/ai/files', aiFilesRouter);  // AI 文件处理
}
```

### 3.2 路由定义

**aiRouter** 定义了两个主要端点：

```typescript
// api/src/ai/chat/router.ts:1-9
export const aiRouter = Router()
    .post('/chat', asyncHandler(loadSettings), asyncHandler(aiChatPostHandler))
    .post('/object', asyncHandler(loadSettings), asyncHandler(aiObjectPostHandler));
```

| 端点 | 用途 | 控制器 |
|------|------|--------|
| `POST /ai/chat` | 流式对话（支持工具调用） | `aiChatPostHandler` |
| `POST /ai/object` | 结构化对象生成（如翻译、生成描述） | `aiObjectPostHandler` |

### 3.3 中间件：加载 AI 设置

`loadSettings` 中间件从数据库加载 AI 配置：

```typescript
// api/src/ai/chat/middleware/load-settings.ts:6-49
export const loadSettings: RequestHandler = async (_req, res, next) => {
    const service = new SettingsService({ schema: await getSchema() });
    
    const settings = await service.readSingleton({
        fields: [
            'ai_openai_api_key',
            'ai_anthropic_api_key', 
            'ai_google_api_key',
            'ai_openai_compatible_api_key',
            'ai_openai_compatible_base_url',
            'ai_openai_allowed_models',
            'ai_anthropic_allowed_models',
            'ai_google_allowed_models',
            'ai_system_prompt',
        ],
    });

    const aiSettings: AISettings = {
        openaiApiKey: settings['ai_openai_api_key'] ?? null,
        anthropicApiKey: settings['ai_anthropic_api_key'] ?? null,
        googleApiKey: settings['ai_google_api_key'] ?? null,
        openaiCompatibleApiKey: settings['ai_openai_compatible_api_key'] ?? null,
        openaiCompatibleBaseUrl: settings['ai_openai_compatible_base_url'] ?? null,
        // ... 其他设置
    };

    res.locals['ai'] = { settings: aiSettings, systemPrompt: settings['ai_system_prompt'] };
    return next();
};
```

### 3.4 Chat 控制器处理

`aiChatPostHandler` 处理流式对话请求：

```typescript
// api/src/ai/chat/controllers/chat.post.ts:11-82
export const aiChatPostHandler: RequestHandler = async (req, res, _next) => {
    // 权限检查：必须是 App 内部请求
    if (!req.accountability?.app) {
        throw new ForbiddenError();
    }

    // 解析请求体
    const parseResult = ChatRequest.safeParse(req.body);
    const { provider, model, messages, tools: requestedTools, toolApprovals, context } = parseResult.data;

    // 验证模型是否被允许
    const aiSettings = res.locals['ai'].settings;
    const allowedModelsMap = {
        openai: aiSettings.openaiAllowedModels,
        anthropic: aiSettings.anthropicAllowedModels,
        google: aiSettings.googleAllowedModels,
    };

    // 将请求工具转换为 AI SDK 工具
    const tools = requestedTools.reduce<{ [x: string]: Tool }>((acc, t) => {
        const name = typeof t === 'string' ? t : t.name;
        const aiTool = chatRequestToolToAiSdkTool({
            chatRequestTool: t,
            accountability: req.accountability!,
            schema: req.schema,
            ...(toolApprovals && { toolApprovals }),
        });
        acc[name] = aiTool;
        return acc;
    }, {});

    // 创建 UI 流并调用 LLM
    const stream = await createUiStream(validationResult.data, {
        provider,
        model,
        tools,
        aiSettings,
        userId: req.accountability?.user,
        role: req.accountability?.role,
        systemPrompt: res.locals['ai'].systemPrompt,
        ...(context && { context }),
    });

    // 将流式响应返回给客户端
    stream.pipeUIMessageStreamToResponse(res);
};
```

### 3.5 LLM 提供商注册表

`createAIProviderRegistry` 创建不同 LLM 提供商的客户端：

```typescript
// api/src/ai/providers/registry.ts:10-78
export function buildProviderConfigs(settings: AISettings): ProviderConfig[] {
    const configs: ProviderConfig[] = [];
    
    if (settings.openaiApiKey) {
        configs.push({ type: 'openai', apiKey: settings.openaiApiKey });
    }
    if (settings.anthropicApiKey) {
        configs.push({ type: 'anthropic', apiKey: settings.anthropicApiKey });
    }
    if (settings.googleApiKey) {
        configs.push({ type: 'google', apiKey: settings.googleApiKey });
    }
    if (settings.openaiCompatibleApiKey && settings.openaiCompatibleBaseUrl) {
        configs.push({ 
            type: 'openai-compatible', 
            apiKey: settings.openaiCompatibleApiKey, 
            baseUrl: settings.openaiCompatibleBaseUrl 
        });
    }
    return configs;
}

export function createAIProviderRegistry(configs: ProviderConfig[], settings?: AISettings) {
    const providers = {};
    
    for (const config of configs) {
        switch (config.type) {
            case 'openai':
                providers['openai'] = createOpenAI({ apiKey: config.apiKey });
                break;
            case 'anthropic':
                providers['anthropic'] = createAnthropicWithFileSupport(config.apiKey);
                break;
            case 'google':
                providers['google'] = createGoogleGenerativeAI({ apiKey: config.apiKey });
                break;
            case 'openai-compatible':
                if (config.baseUrl) {
                    providers['openai-compatible'] = createOpenAICompatible({
                        name: settings?.openaiCompatibleName ?? 'openai-compatible',
                        apiKey: config.apiKey,
                        baseURL: config.baseUrl,
                    });
                }
                break;
        }
    }
    
    return createProviderRegistry(providers);
}
```

### 3.6 创建 UI 流并调用 LLM

`createUiStream` 是调用外部 LLM 的核心函数：

```typescript
// api/src/ai/chat/lib/create-ui-stream.ts:39-101
export const createUiStream = async (
    messages: UIMessage[],
    { provider, model, tools, aiSettings, systemPrompt, userId, role, context, onUsage }: CreateUiStreamOptions,
) => {
    // 1. 构建提供商配置
    const configs = buildProviderConfigs(aiSettings);
    const providerConfig = configs.find((c) => c.type === provider);

    if (!providerConfig) {
        throw new ServiceUnavailableError({ 
            service: provider, 
            reason: 'No API key configured for LLM provider' 
        });
    }

    // 2. 创建提供商注册表和语言模型
    const registry = createAIProviderRegistry(configs, aiSettings);
    const providerOptions = getProviderOptions(provider, model, aiSettings);
    let languageModel = registry.languageModel(`${provider}:${model}`);

    // 3. 应用开发工具中间件（如果启用）
    const devToolsMiddleware = getDevToolsMiddleware();
    if (devToolsMiddleware) {
        languageModel = wrapLanguageModel({ model: languageModel, middleware: devToolsMiddleware });
    }

    // 4. 准备系统提示词
    const baseSystemPrompt = systemPrompt || SYSTEM_PROMPT;
    const contextBlock = context ? formatContextForSystemPrompt(context) : null;
    const fullSystemPrompt = contextBlock ? baseSystemPrompt + contextBlock : baseSystemPrompt;

    // 5. 调用 LLM（使用 Vercel AI SDK）
    const telemetryConfig = getAITelemetryConfig({ provider, model, userId, role });
    const finalTools = applyAnthropicToolSearch(provider, model, tools);

    const stream = streamText({
        system: baseSystemPrompt,
        model: languageModel,
        messages: await convertToModelMessages(transformFilePartsForProvider(messages)),
        stopWhen: [stepCountIs(10)],
        providerOptions,
        tools: finalTools,
        ...(telemetryConfig ? { experimental_telemetry: telemetryConfig } : {}),
        prepareStep: () => {
            if (contextBlock) {
                return { system: fullSystemPrompt };
            }
            return {};
        },
        onFinish({ usage }) {
            if (onUsage) {
                const { inputTokens, outputTokens, totalTokens } = usage;
                onUsage({ inputTokens, outputTokens, totalTokens });
            }
        },
    });

    return stream;
};
```

### 3.7 Object 端点（结构化对象生成）

`/ai/object` 端点用于生成结构化数据，适用于自动翻译、生成描述等场景：

```typescript
// api/src/ai/chat/controllers/object.post.ts:17-77
export const aiObjectPostHandler: RequestHandler = async (req, res) => {
    // 权限检查
    if (!req.accountability?.app) {
        throw new ForbiddenError();
    }

    const parseResult = ObjectRequest.safeParse(req.body);
    const { provider, model, prompt, outputSchema, maxOutputTokens } = parseResult.data;

    // 构建提供商配置
    const aiSettings: AISettings = res.locals['ai'].settings;
    const configs = buildProviderConfigs(aiSettings);
    const providerConfig = configs.find((c) => c.type === provider);

    if (!providerConfig) {
        throw new ServiceUnavailableError({ 
            service: provider, 
            reason: 'No API key configured for LLM provider' 
        });
    }

    // 创建语言模型
    const registry = createAIProviderRegistry(configs, aiSettings);
    const providerOptions = getProviderOptions(provider, model, aiSettings);
    let languageModel = registry.languageModel(`${provider}:${model}`);

    // 使用 streamObject 生成结构化输出
    const result = streamObject({
        model: languageModel,
        prompt,
        schema: jsonSchema(addAdditionalPropertiesToJsonSchema(outputSchema)),
        providerOptions,
        ...(typeof maxOutputTokens === 'number' ? { maxOutputTokens } : {}),
    });

    // 将结果流式返回
    result.pipeTextStreamToResponse(res);
};
```

---

## 4. LLM 结果通过钩子/流程写回数据

### 4.1 两种主要机制

Directus 提供了两种方式让 LLM 结果写回数据集合：

| 机制 | 适用场景 | 实现位置 |
|------|----------|----------|
| **AI 工具直接操作** | 实时对话中 LLM 主动操作数据 | `api/src/ai/tools/items/index.ts` |
| **流程 (Flow) + 操作 (Operation)** | 自动化工作流、事件触发 | `api/src/flows.ts` + `api/src/operations/` |

### 4.2 AI 工具系统

#### 4.2.1 工具定义

AI 工具是 LLM 可以调用的函数，用于与 Directus 系统交互：

```typescript
// api/src/ai/tools/types.ts:35-44
export interface ToolConfig<T> {
    name: string;           // 工具名称
    description: string;     // 工具描述（给 LLM 看）
    endpoint?: ToolEndpoint<T>;
    admin?: boolean;
    inputSchema: ZodType<any>;     // 输入参数 Schema
    validateSchema?: ZodType<T>;   // 验证 Schema
    annotations?: ToolAnnotations;
    handler: ToolHandler<T>;       // 执行函数
}
```

#### 4.2.2 已注册的 AI 工具

```typescript
// api/src/ai/tools/index.ts:15-28
export const ALL_TOOLS: ToolConfig<any>[] = [
    system,         // 系统信息
    items,          // 数据项 CRUD ⭐
    files,          // 文件操作
    folders,        // 文件夹操作
    assets,         // 资产操作
    flows,          // 流程管理
    triggerFlow,    // 触发手动流程 ⭐
    operations,     // 操作管理
    schema,         // 数据库 Schema
    collections,    // 集合管理
    fields,         // 字段管理
    relations,      // 关系管理
];
```

#### 4.2.3 Items 工具（核心数据操作）

`items` 工具允许 LLM 直接对数据进行 CRUD 操作：

```typescript
// api/src/ai/tools/items/index.ts:62-187
export const items = defineTool<z.infer<typeof ItemsValidateSchema>>({
    name: 'items',
    description: '...',  // 工具描述（告诉 LLM 这个工具能做什么）
    inputSchema: ItemsInputSchema,  // 输入参数定义
    validateSchema: ItemsValidateSchema,
    
    async handler({ args, schema, accountability }) {
        // 验证集合
        if (isSystemCollection(args.collection)) {
            throw new InvalidPayloadError({ reason: 'Cannot provide a core collection' });
        }
        if (args.collection in schema.collections === false) {
            throw new ForbiddenError();
        }

        // 创建 ItemsService 实例
        const isSingleton = schema.collections[args.collection]?.singleton ?? false;
        const itemsService = new ItemsService(args.collection, {
            schema,
            accountability,  // 权限上下文
        });

        // 根据 action 执行不同操作
        switch (args.action) {
            case 'create':
                // 创建数据
                const savedKeys = await itemsService.createMany(toArray(args.data));
                const result = await itemsService.readMany(savedKeys, sanitizedQuery);
                return { type: 'text', data: result || null };

            case 'read':
                // 读取数据
                let readResult = null;
                if (isSingleton) {
                    readResult = await itemsService.readSingleton(sanitizedQuery);
                } else if (args.keys) {
                    readResult = await itemsService.readMany(args.keys, sanitizedQuery);
                } else {
                    readResult = await itemsService.readByQuery(sanitizedQuery);
                }
                return { type: 'text', data: readResult };

            case 'update':  // ⭐ 用于写回 LLM 结果
                // 更新数据
                let updatedKeys: PrimaryKey[] = [];
                if (Array.isArray(args.data)) {
                    updatedKeys = await itemsService.updateBatch(args.data);
                } else if (args.keys) {
                    updatedKeys = await itemsService.updateMany(args.keys, args.data);
                } else {
                    updatedKeys = await itemsService.updateByQuery(sanitizedQuery, args.data);
                }
                const updatedResult = await itemsService.readMany(updatedKeys, sanitizedQuery);
                return { type: 'text', data: updatedResult };

            case 'delete':
                // 删除数据
                const deletedKeys = await itemsService.deleteMany(args.keys);
                return { type: 'text', data: deletedKeys };
        }

        throw new Error('Invalid action.');
    },
});
```

#### 4.2.4 Trigger Flow 工具

`trigger-flow` 工具允许 LLM 触发预定义的手动流程：

```typescript
// api/src/ai/tools/trigger-flow/index.ts:13-63
export const triggerFlow = defineTool<z.infer<typeof TriggerFlowValidateSchema>>({
    name: 'trigger-flow',
    description: '...',
    inputSchema: TriggerFlowInputSchema,
    validateSchema: TriggerFlowValidateSchema,
    
    async handler({ args, schema, accountability }) {
        const flowsService = new FlowsService({ schema, accountability });

        // 验证流程是否存在且为手动触发类型
        const flow = await flowsService.readOne(args.id, {
            filter: { status: { _eq: 'active' }, trigger: { _eq: 'manual' } },
            fields: ['options'],
        });

        // 验证必填字段
        const requiredFields = ((flow.options?.['fields'] as { field: string; meta: { required: boolean } }[]) ?? [])
            .filter((field) => field.meta?.required)
            .map((field) => field.field);

        for (const fieldName of requiredFields) {
            if (!args.data || !(fieldName in args.data)) {
                throw new InvalidPayloadError({ reason: `Required field "${fieldName}" is missing` });
            }
        }

        // 触发流程
        const flowManager = getFlowManager();
        const { result } = await flowManager.runWebhookFlow(
            `POST-${args.id}`,
            {
                path: `/trigger/${args.id}`,
                query: args.query ?? {},
                method: 'POST',
                body: {
                    collection: args.collection,
                    keys: args.keys,
                    ...(args.data ?? {}),  // 传递给流程的数据（包括 LLM 结果）
                },
                headers: args.headers ?? {},
            },
            { accountability, schema },
        );

        return { type: 'text', data: result };
    },
});
```

### 4.3 流程引擎 (Flow Manager)

#### 4.3.1 流程管理器核心

`FlowManager` 负责加载、执行和管理所有流程：

```typescript
// api/src/flows.ts:63-96
class FlowManager {
    private isLoaded = false;
    private flows: Record<string, Flow> = {};
    private operations: Map<string, OperationHandler> = new Map();
    
    private triggerHandlers: TriggerHandler[] = [];
    private operationFlowHandlers: Record<string, any> = {};
    private webhookFlowHandlers: Record<string, any> = {};

    // 初始化：从数据库加载所有激活的流程
    public async initialize(): Promise<void> {
        if (!this.isLoaded) {
            await this.load();
        }
    }

    // ...
}
```

#### 4.3.2 流程触发器类型

Directus 支持多种流程触发器：

```typescript
// api/src/flows.ts:169-363
private async load(): Promise<void> {
    // 从数据库读取激活的流程
    const flows = await flowsService.readByQuery({
        filter: { status: { _eq: 'active' } },
        fields: ['*', 'operations.*'],
    });

    for (const flow of flowTrees) {
        this.flows[flow.id] = flow;

        switch (flow.trigger) {
            // 1. 事件触发器 (Filter / Action)
            case 'event':
                const events: string[] = [];
                // 构建事件列表，如：items.create, items.update, items.delete
                
                if (flow.options['type'] === 'filter') {
                    // Filter 钩子：在操作执行前触发，可以修改 payload
                    const handler: FilterHandler = (payload, meta, context) =>
                        this.executeFlow(flow, { payload, ...meta }, { /* context */ });
                    events.forEach((event) => emitter.onFilter(event, handler));
                } else if (flow.options['type'] === 'action') {
                    // Action 钩子：在操作执行后触发
                    const handler: ActionHandler = (meta, context) =>
                        this.executeFlow(flow, meta, { /* context */ });
                    events.forEach((event) => emitter.onAction(event, handler));
                }
                break;

            // 2. 定时触发器 (Cron)
            case 'schedule':
                if (validateCron(flow.options['cron'])) {
                    const job = scheduleSynchronizedJob(flow.id, flow.options['cron'], async () => {
                        await this.executeFlow(flow);
                    });
                }
                break;

            // 3. 操作触发器
            case 'operation':
                const handler = (data, context) => this.executeFlow(flow, data, context);
                this.operationFlowHandlers[flow.id] = handler;
                break;

            // 4. Webhook 触发器
            case 'webhook':
                // 注册到 webhookFlowHandlers
                this.webhookFlowHandlers[`${method}-${flow.id}`] = handler;
                break;

            // 5. 手动触发器 ⭐（可被 AI 工具触发）
            case 'manual':
                // 注册到 webhookFlowHandlers，使用 POST 方法
                this.webhookFlowHandlers[`POST-${flow.id}`] = handler;
                break;
        }
    }
}
```

#### 4.3.3 执行流程

```typescript
// api/src/flows.ts:393-504
private async executeFlow(flow: Flow, data: unknown = null, context: Record<string, unknown> = {}): Promise<unknown> {
    // 准备上下文数据
    const keyedData: Record<string, unknown> = {
        [TRIGGER_KEY]: data,        // 触发器数据
        [LAST_KEY]: data,            // 最后一个操作的结果
        [ACCOUNTABILITY_KEY]: context?.['accountability'] ?? null,
        [ENV_KEY]: this.envs,
    };

    let nextOperation = flow.operation;
    
    // 循环执行操作链
    while (nextOperation !== null) {
        const { successor, data, status, options } = await this.executeOperation(
            nextOperation, 
            keyedData, 
            context
        );

        keyedData[nextOperation.key] = data;
        keyedData[LAST_KEY] = data;
        lastOperationStatus = status;
        nextOperation = successor;
    }

    // 根据配置返回结果
    if (flow.options['return'] === '$all') {
        return keyedData;
    } else if (flow.options['return']) {
        return get(keyedData, flow.options['return']);
    }

    return undefined;
}
```

#### 4.3.4 执行单个操作

```typescript
// api/src/flows.ts:506-600
private async executeOperation(
    operation: Operation,
    keyedData: Record<string, unknown>,
    context: Record<string, unknown> = {},
) {
    const handler = this.operations.get(operation.type)!;

    // 应用模板变量（如 {{ $trigger.payload.title }}）
    let options = operation.options;
    try {
        options = applyOptionsData(options, keyedData);

        // 执行操作处理器
        let result = await handler(options, {
            services,
            env: useEnv(),
            database: getDatabase(),
            logger,
            getSchema,
            data: keyedData,
            accountability: null,
            ...context,
        });

        return { 
            successor: operation.resolve,  // 成功时的下一个操作
            status: 'resolve', 
            data: result ?? null, 
            options 
        };
    } catch (error) {
        return { 
            successor: operation.reject,   // 失败时的下一个操作
            status: 'reject', 
            data: error ?? null,
            options 
        };
    }
}
```

### 4.4 关键操作类型

#### 4.4.1 Item Update 操作（写回数据核心）

```typescript
// api/src/operations/item-update/index.ts:17-70
export default defineOperationApi<Options>({
    id: 'item-update',

    handler: async (
        { collection, key, payload, query, emitEvents, permissions },
        { accountability, database, getSchema },
    ) => {
        const schema = await getSchema({ database });

        // 处理权限
        let customAccountability: Accountability | null;
        if (!permissions || permissions === '$trigger') {
            customAccountability = accountability;
        } else if (permissions === '$full') {
            customAccountability = await getAccountabilityForRole('system', { database, schema, accountability });
        } // ... 其他权限模式

        // 创建 ItemsService
        const itemsService: ItemsService = new ItemsService(collection, {
            schema: await getSchema({ database }),
            accountability: customAccountability,
            knex: database,
        });

        // 解析 payload（支持模板变量）
        const payloadObject: Partial<Item> | Partial<Item>[] | null = optionToObject(payload) ?? null;

        if (!payloadObject) {
            return null;
        }

        // 执行更新
        let result: PrimaryKey | PrimaryKey[] | null;
        if (Array.isArray(payloadObject)) {
            // 批量更新
            result = await itemsService.updateBatch(payloadObject, { emitEvents: !!emitEvents });
        } else if (!key || (Array.isArray(key) && key.length === 0)) {
            // 按查询条件更新
            result = await itemsService.updateByQuery(sanitizedQueryObject, payloadObject, { emitEvents: !!emitEvents });
        } else {
            // 按主键更新
            const keys = toArray(key);
            if (keys.length === 1) {
                result = await itemsService.updateOne(keys[0]!, payloadObject, { emitEvents: !!emitEvents });
            } else {
                result = await itemsService.updateMany(keys, payloadObject, { emitEvents: !!emitEvents });
            }
        }

        return result;
    },
});
```

#### 4.4.2 Request 操作（调用外部 API）

```typescript
// api/src/operations/request/index.ts:14-55
export default defineOperationApi<Options>({
    id: 'request',

    handler: async ({ url, method, body, headers }) => {
        // 构建请求头
        const customHeaders = headers?.reduce(
            (acc, { header, value }) => {
                acc[header] = value;
                return acc;
            },
            {} as Record<string, string>,
        ) ?? {};

        // 发送 HTTP 请求
        const axios = await getAxios();
        try {
            const result = await axios({
                url: encodeUrl(url),
                method,
                data: body,
                headers: customHeaders,
            });

            return { 
                status: result.status, 
                statusText: result.statusText, 
                headers: result.headers, 
                data: result.data 
            };
        } catch (error) {
            // 错误处理
            throw error;
        }
    },
});
```

### 4.5 数据写回流程图解

#### 场景 1：AI 工具直接更新数据

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           用户在 AI 侧边栏输入                                   │
│         "将 products 集合中 id=1 的 title 字段翻译成英文"                        │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                           LLM 理解并调用工具                                      │
│  LLM 分析用户意图，决定：                                                         │
│  1. 先调用 items.read 读取原始数据                                                │
│  2. 翻译内容                                                                     │
│  3. 调用 items.update 写回翻译结果                                               │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Items Tool Handler 执行                                    │
│                                                                                  │
│   步骤 1: items.read                                                            │
│   ┌────────────────────────────────────────────────────────────────────────┐  │
│   │ args = {                                                                │  │
│   │   action: 'read',                                                       │  │
│   │   collection: 'products',                                               │  │
│   │   keys: ['1'],                                                          │  │
│   │   query: { fields: ['id', 'title'] }                                   │  │
│   │ }                                                                       │  │
│   │                                                                         │  │
│   │ itemsService.readMany(['1'], sanitizedQuery)                          │  │
│   │ ──▶ 返回 { id: 1, title: '智能手表' }                                  │  │
│   └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   步骤 2: LLM 内部处理（翻译）                                                   │
│   ┌────────────────────────────────────────────────────────────────────────┐  │
│   │ 输入: "智能手表"                                                         │  │
│   │ 输出: "Smart Watch"                                                      │  │
│   └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   步骤 3: items.update ⭐                                                       │
│   ┌────────────────────────────────────────────────────────────────────────┐  │
│   │ args = {                                                                │  │
│   │   action: 'update',                                                     │  │
│   │   collection: 'products',                                               │  │
│   │   keys: ['1'],                                                          │  │
│   │   data: { title: 'Smart Watch' }                                        │  │
│   │ }                                                                       │  │
│   │                                                                         │  │
│   │ itemsService.updateMany(['1'], { title: 'Smart Watch' })              │  │
│   │ ──▶ 调用 Knex 更新数据库                                                 │  │
│   │ ──▶ 触发 items.update 事件钩子（如果有监听）                              │  │
│   └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### 场景 2：通过流程 + 操作写回数据

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                        配置手动触发流程                                          │
│                                                                                  │
│  流程名称: "AI 翻译并写回"                                                       │
│  触发器: Manual (手动)                                                          │
│  操作 1: Request (调用外部翻译 API)                                              │
│    - URL: https://api.openai.com/v1/chat/completions                          │
│    - Method: POST                                                               │
│    - Body: {                                                                    │
│        "model": "gpt-4",                                                        │
│        "messages": [                                                            │
│          {"role": "user", "content": "Translate to English: {{$trigger.data.text}}"}
│        ]                                                                        │
│      }                                                                          │
│                                                                                  │
│  操作 2: Item Update (写回数据库)                                                │
│    - Collection: products                                                       │
│    - Key: {{ $trigger.keys[0] }}                                               │
│    - Payload: {                                                                 │
│        "title_en": "{{ $last.data.choices[0].message.content }}"              │
│      }                                                                          │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        LLM 调用 trigger-flow 工具                               │
│                                                                                  │
│   LLM 决定触发流程：                                                             │
│   {                                                                             │
│     "name": "trigger-flow",                                                    │
│     "input": {                                                                  │
│       "id": "flow-uuid-123",                    // 流程 ID                    │
│       "collection": "products",                 // 目标集合                    │
│       "keys": ["1"],                             // 目标条目                    │
│       "data": {                                                                  │
│         "text": "智能手表"                        // 要翻译的文本               │
│       }                                                                         │
│     }                                                                           │
│   }                                                                             │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                      Flow Manager 执行流程                                      │
│                                                                                  │
│   步骤 1: runWebhookFlow                                                        │
│   ┌────────────────────────────────────────────────────────────────────────┐  │
│   │ flowManager.runWebhookFlow(                                             │  │
│   │   'POST-flow-uuid-123',                                                 │  │
│   │   {                                                                      │  │
│   │     path: '/trigger/flow-uuid-123',                                     │  │
│   │     method: 'POST',                                                      │  │
│   │     body: {                                                              │  │
│   │       collection: 'products',                                            │  │
│   │       keys: ['1'],                                                       │  │
│   │       text: '智能手表'                                                    │  │
│   │     }                                                                    │  │
│   │   },                                                                     │  │
│   │   { accountability, schema }                                             │  │
│   │ )                                                                        │  │
│   └────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   步骤 2: executeFlow (执行操作链)                                              │
│   ┌────────────────────────────────────────────────────────────────────────┐  │
│   │ 操作 1: Request (调用外部 API)                                           │  │
│   │ ──▶ 返回翻译结果 "Smart Watch"                                           │  │
│   │                                                                         │  │
│   │ 操作 2: Item Update ⭐                                                   │  │
│   │ keyedData = {                                                           │  │
│   │   '$trigger': { collection: 'products', keys: ['1'], text: '智能手表' },│
│   │   '$last': { data: { choices: [{ message: { content: 'Smart Watch' }}]}},│
│   │   'operation-1': { ... },                                               │  │
│   │   ...                                                                    │  │
│   │ }                                                                       │  │
│   │                                                                         │  │
│   │ // 应用模板变量                                                          │  │
│   │ payload = applyOptionsData({                                            │  │
│   │   "title_en": "{{ $last.data.choices[0].message.content }}"            │  │
│   │ }, keyedData);                                                          │  │
│   │ // 结果: { "title_en": "Smart Watch" }                                  │  │
│   │                                                                         │  │
│   │ itemsService.updateMany(['1'], { "title_en": "Smart Watch" })          │  │
│   └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### 场景 3：事件钩子自动触发（数据变更时自动调用 AI）

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                       配置事件触发流程                                           │
│                                                                                  │
│  流程名称: "创建产品时自动生成描述"                                               │
│  触发器: Event (事件)                                                           │
│  触发类型: Action (操作后)                                                      │
│  作用域: items.create                                                           │
│  集合: products                                                                 │
│                                                                                  │
│  操作 1: Request (调用 LLM 生成描述)                                            │
│    - URL: {{ $env.LLM_API_URL }}                                               │
│    - Body: {                                                                    │
│        "prompt": "根据产品标题生成产品描述: {{$trigger.payload.title}}"          │
│      }                                                                          │
│                                                                                  │
│  操作 2: Item Update (写回描述)                                                 │
│    - Key: {{ $trigger.keys[0] }}                                               │
│    - Payload: { "description": "{{ $last.data }}" }                           │
└──────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                         数据创建触发流程                                         │
│                                                                                  │
│   1. 用户创建新产品                                                             │
│      POST /items/products                                                      │
│      Body: { "title": "智能手表" }                                             │
│                                                                                  │
│   2. ItemsService.createOne 执行                                               │
│      ──▶ 写入数据库                                                             │
│      ──▶ 触发事件: emitter.emitAction('products.items.create', meta, context) │
│                                                                                  │
│   3. FlowManager 的事件监听器被触发                                             │
│      ┌────────────────────────────────────────────────────────────────────┐  │
│      │ // api/src/flows.ts:214-228                                        │  │
│      │ } else if (flow.options['type'] === 'action') {                    │  │
│      │   const handler: ActionHandler = (meta, context) =>                │  │
│      │       this.executeFlow(flow, meta, {                                │  │
│      │           accountability: context['accountability'],                │  │
│      │           database: getDatabase(),                                   │  │
│      │           getSchema: ...,                                            │  │
│      │       });                                                             │  │
│      │   events.forEach((event) => emitter.onAction(event, handler));     │  │
│      │ }                                                                    │  │
│      └────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   4. 流程执行                                                                    │
│      ──▶ 操作 1: 调用 LLM 生成描述                                             │
│      ──▶ 操作 2: 使用 updateMany 写回 description 字段                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. 配置示例

### 5.1 环境变量配置

```bash
# .env

# 启用 AI 功能
AI_ENABLED=true

# OpenAI 配置
AI_OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AI_OPENAI_ALLOWED_MODELS=gpt-4,gpt-4o,gpt-3.5-turbo

# Anthropic 配置
AI_ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxx
AI_ANTHROPIC_ALLOWED_MODELS=claude-3-5-sonnet-20240620,claude-3-opus-20240229

# Google AI 配置
AI_GOOGLE_API_KEY=xxxxxxxxxxxxxxxx
AI_GOOGLE_ALLOWED_MODELS=gemini-1.5-pro,gemini-1.5-flash

# 自定义 OpenAI 兼容服务（如 Ollama、本地模型）
AI_OPENAI_COMPATIBLE_API_KEY=ollama
AI_OPENAI_COMPATIBLE_BASE_URL=http://localhost:11434/v1
AI_OPENAI_COMPATIBLE_NAME=Ollama
AI_OPENAI_COMPATIBLE_MODELS=[{"id": "llama3.1", "name": "Llama 3.1", "context": 128000}]

# 系统提示词（可选）
AI_SYSTEM_PROMPT="你是 Directus 助手，可以帮助用户管理数据。当需要操作数据时，请使用提供的工具。"
```

### 5.2 流程配置示例：自动翻译

**场景**：当产品的中文标题更新时，自动翻译成英文并写入 `title_en` 字段。

**步骤 1：创建流程**

1. 进入 **Settings > Flows > Create Flow**
2. 配置触发器：
   - **Trigger**: `Event`
   - **Type**: `Action (After)`
   - **Scope**: `items.update`
   - **Collections**: `products`
3. 添加操作 1：**Request**（调用 LLM）
   - **Method**: `POST`
   - **URL**: `https://api.openai.com/v1/chat/completions`
   - **Headers**:
     - `Authorization`: `Bearer {{ $env.AI_OPENAI_API_KEY }}`
     - `Content-Type`: `application/json`
   - **Body**:
     ```json
     {
       "model": "gpt-4",
       "messages": [
         {"role": "system", "content": "你是翻译助手，只返回翻译结果，不要有其他内容。"},
         {"role": "user", "content": "将以下文本翻译成英文：{{ $trigger.payload.title }}"}
       ],
       "temperature": 0.3
     }
     ```
4. 添加操作 2：**Item Update**（写回数据）
   - **Collection**: `products`
   - **Key**: `{{ $trigger.keys[0] }}`
   - **Payload**:
     ```json
     {
       "title_en": "{{ $last.data.choices[0].message.content }}"
     }
     ```
5. 保存并激活流程

### 5.3 使用 AI 工具的对话示例

**用户输入**：
> "读取 products 集合中 id=5 的产品，将标题翻译成英文，然后更新回去"

**LLM 可能的工具调用序列**：

```json
// 工具调用 1: 读取数据
{
  "name": "items",
  "input": {
    "action": "read",
    "collection": "products",
    "keys": ["5"],
    "query": {
      "fields": ["id", "title", "title_en"]
    }
  }
}

// 工具返回结果
{
  "type": "text",
  "data": [
    { "id": 5, "title": "无线蓝牙耳机", "title_en": null }
  ]
}

// LLM 内部处理翻译
// 输入: "无线蓝牙耳机" → 输出: "Wireless Bluetooth Headphones"

// 工具调用 2: 更新数据 ⭐
{
  "name": "items",
  "input": {
    "action": "update",
    "collection": "products",
    "keys": ["5"],
    "data": {
      "title_en": "Wireless Bluetooth Headphones"
    }
  }
}

// 工具返回结果
{
  "type": "text",
  "data": [
    { "id": 5, "title": "无线蓝牙耳机", "title_en": "Wireless Bluetooth Headphones" }
  ]
}
```

---

## 6. 关键代码引用

### 6.1 前端相关文件

| 文件路径 | 说明 |
|----------|------|
| `app/src/ai/stores/use-ai.ts` | AI 状态管理核心，处理对话和工具调用 |
| `app/src/ai/components/ai-conversation.vue` | AI 对话主组件 |
| `app/src/ai/components/ai-input.vue` | AI 输入框组件 |
| `app/src/ai/stores/use-ai-tools.ts` | AI 工具管理 |
| `app/src/ai/stores/use-ai-context.ts` | AI 上下文管理 |

### 6.2 API 路由相关文件

| 文件路径 | 说明 |
|----------|------|
| `api/src/app.ts:348-353` | AI 路由注册 |
| `api/src/ai/chat/router.ts` | AI 路由定义 |
| `api/src/ai/chat/middleware/load-settings.ts` | 加载 AI 设置中间件 |
| `api/src/ai/chat/controllers/chat.post.ts` | 流式对话控制器 |
| `api/src/ai/chat/controllers/object.post.ts` | 结构化对象生成控制器 |

### 6.3 LLM 提供商相关文件

| 文件路径 | 说明 |
|----------|------|
| `api/src/ai/providers/registry.ts` | 提供商注册表 |
| `api/src/ai/providers/types.ts` | 提供商配置类型定义 |
| `api/src/ai/chat/lib/create-ui-stream.ts` | 创建 UI 流并调用 LLM |

### 6.4 AI 工具相关文件

| 文件路径 | 说明 |
|----------|------|
| `api/src/ai/tools/index.ts` | AI 工具注册中心 |
| `api/src/ai/tools/define-tool.ts` | 工具定义函数 |
| `api/src/ai/tools/types.ts` | 工具类型定义 |
| `api/src/ai/tools/items/index.ts` | 数据项 CRUD 工具 ⭐ |
| `api/src/ai/tools/trigger-flow/index.ts` | 触发流程工具 ⭐ |
| `api/src/ai/chat/utils/chat-request-tool-to-ai-sdk-tool.ts` | 工具转换函数 |

### 6.5 流程和钩子相关文件

| 文件路径 | 说明 |
|----------|------|
| `api/src/flows.ts` | 流程管理器核心 ⭐ |
| `api/src/operations/item-update/index.ts` | Item Update 操作 ⭐ |
| `api/src/operations/request/index.ts` | Request 操作 |
| `api/src/operations/item-create/index.ts` | Item Create 操作 |
| `api/src/services/items.ts` | ItemsService 业务逻辑 |

### 6.6 事件发射器

事件钩子通过 `emitter` 模块触发和监听：

```typescript
// 关键方法（在 api/src/emitter.ts 中定义）
emitter.onFilter(eventName, handler)   // 注册 Filter 钩子（操作前）
emitter.onAction(eventName, handler)   // 注册 Action 钩子（操作后）
emitter.emitFilter(eventName, ...)      // 触发 Filter 事件
emitter.emitAction(eventName, ...)      // 触发 Action 事件
```

---

## 附录：完整数据流时序图

```
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│  User   │     │  App    │     │  API    │     │  LLM    │     │   DB    │
└────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘     └────┬────┘
     │               │               │               │               │
     │  1. 输入提示词│               │               │               │
     │──────────────▶│               │               │               │
     │               │               │               │               │
     │               │ 2. POST /ai/chat              │               │
     │               │──────────────▶│               │               │
     │               │               │               │               │
     │               │               │ 3. loadSettings 读取 AI 配置    │
     │               │               │──────────────────────────────▶│
     │               │               │◀──────────────────────────────│
     │               │               │               │               │
     │               │               │ 4. 创建 Provider Registry      │
     │               │               │               │               │
     │               │               │ 5. streamText() 调用 LLM       │
     │               │               │──────────────▶│               │
     │               │               │               │               │
     │               │               │               │ 6. LLM 决定调用工具  │
     │               │               │               │               │
     │               │               │◀──────────────│               │
     │               │               │               │               │
     │               │               │ 7. 执行 items 工具             │
     │               │               │──────────────────────────────▶│
     │               │               │◀──────────────────────────────│
     │               │               │               │               │
     │               │               │ 8. 发送工具结果给 LLM          │
     │               │               │──────────────▶│               │
     │               │               │               │               │
     │               │               │               │ 9. LLM 生成最终响应  │
     │               │               │◀──────────────│               │
     │               │               │               │               │
     │               │ 10. 流式返回响应              │               │
     │               │◀──────────────│               │               │
     │               │               │               │               │
     │ 11. 显示结果 │               │               │               │
     │◀──────────────│               │               │               │
     │               │               │               │               │
     └───────────────┴───────────────┴───────────────┴───────────────┴───────────────┘
```

---

## 总结

Directus 的 AI 字段集成机制采用了三层架构：

1. **前端触发层**：通过 AI 侧边栏组件和 `use-ai.ts` store 管理用户交互，支持上下文感知和自动工具调用。

2. **API 路由层**：提供 `/ai/chat`（流式对话）和 `/ai/object`（结构化对象生成）两个端点，通过 Vercel AI SDK 抽象不同 LLM 提供商的差异。

3. **数据写回层**：通过两种机制实现数据持久化：
   - **AI 工具直接操作**：`items` 工具允许 LLM 直接调用 `ItemsService` 进行 CRUD 操作
   - **流程 + 操作**：`trigger-flow` 工具触发预定义流程，通过 `item-update` 等操作写回数据

4. **事件钩子**：流程可以配置为事件触发（`items.create`、`items.update` 等），实现数据变更时的自动 AI 处理。

这种设计既提供了灵活的交互式 AI 体验（通过对话），又支持自动化的工作流（通过流程和钩子），满足了不同场景下的 AI 集成需求。
