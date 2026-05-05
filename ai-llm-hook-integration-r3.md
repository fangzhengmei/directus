# Directus AI 字段自动化链路深度解析（R3）

**本文档聚焦于实际代码验证的真实路径**

---

## 关键发现前置

在深入分析代码后，得出以下核心结论：

| 端点 | 存在性 | 前端调用 | 实际用途 |
|------|--------|----------|----------|
| `POST /ai/chat` | ✅ 存在 | ✅ 大量调用 | **唯一实际使用的 AI 端点** |
| `POST /ai/object` | ✅ 存在 | ❌ 无调用 | 预留端点，前端未使用 |

**真实的"字段级 AI 生成"路径**：
- 不是字段旁的独立按钮
- 而是通过 **AI 侧边栏 + 上下文菜单 + 工具调用** 实现
- 数据操作通过 **`items` 工具** 完成，**不依赖 flow/request 操作**

---

## 目录

1. [真实触发入口：AI 侧边栏 + 上下文菜单](#1-真实触发入口ai-侧边栏--上下文菜单)
2. [API 选路：为何只有 /ai/chat 被使用](#2-api-选路为何只有-aichat-被使用)
3. [端到端闭环时序（不依赖 request 操作）](#3-端到端闭环时序不依赖-request-操作)
4. [失败分支与权限边界](#4-失败分支与权限边界)
5. [与 Flow/Request 自动化链路的分工对比](#5-与-flowrequest-自动化链路的分工对比)
6. [关键代码引用](#6-关键代码引用)

---

## 1. 真实触发入口：AI 侧边栏 + 上下文菜单

### 1.1 触发入口架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Admin 界面触发入口                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                     AI 侧边栏 (ai-conversation.vue)                   │   │
│   │  ┌───────────────────────────────────────────────────────────────┐  │   │
│   │  │                                                                 │  │   │
│   │  │    ┌─────────────┐                                             │  │   │
│   │  │    │  输入框     │◀── 用户输入指令                               │  │   │
│   │  │    │             │     "将 title 翻译成英文"                    │  │   │
│   │  │    └──────┬──────┘                                             │  │   │
│   │  │           │                                                      │  │   │
│   │  │           ▼                                                      │  │   │
│   │  │    ┌─────────────────────────────────────────────────────────┐ │  │   │
│   │  │    │              AI 上下文菜单 (ai-context-menu.vue)         │ │  │   │
│   │  │    │                                                           │ │  │   │
│   │  │    │  按钮: ➕ Add Content                                    │ │  │   │
│   │  │    │  选项:                                                    │ │  │   │
│   │  │    │    🪄 Prompts        - 插入可复用提示词                   │ │  │   │
│   │  │    │    📦 Content        - 插入数据项上下文 ⭐               │ │  │   │
│   │  │    │    📁 File Library   - 附加文件                          │ │  │   │
│   │  │    │    📤 Upload File    - 上传文件                          │ │  │   │
│   │  │    │                                                           │ │  │   │
│   │  │    └─────────────────────────────────────────────────────────┘ │  │   │
│   │  │                                                                 │  │   │
│   │  └───────────────────────────────────────────────────────────────┘  │   │
│   │                                                                      │   │
│   │  用户操作流程:                                                        │   │
│   │  1. 打开 AI 侧边栏                                                   │   │
│   │  2. 点击 ➕ Add Content → 选择 📦 Content                          │   │
│   │  3. 选择集合 → 选择具体条目 (如 products:1)                        │   │
│   │  4. 输入指令: "将这个产品的 title_zh 翻译成英文，写入 title_en"    │   │
│   │  5. 点击发送                                                        │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 上下文菜单代码解析

```typescript
// app/src/ai/components/ai-context-menu.vue:83-89
// 核心菜单选项
options.push({
    id: 'items',
    icon: 'box',
    title: t('ai.content'),
    subtitle: t('ai.insert_item_context'),
    action: () => openList('items'),
});

// 选择集合后打开抽屉
function handleCollectionSelect(collection: Collection) {
    selectedCollection.value = collection.collection;
    mainMenuOpen.value = false;
    showItemDrawer.value = true;
}

// 选择条目后添加到上下文
async function handleItemSelect(ids: (string | number)[] | null) {
    showItemDrawer.value = false;
    if (selectedCollection.value) {
        await stageItems(selectedCollection.value, ids);  // 添加到待处理上下文
    }
    selectedCollection.value = null;
}
```

### 1.3 当前页面上下文自动注入

除了手动选择上下文，AI 还会自动感知当前页面：

```typescript
// app/src/ai/stores/use-ai.ts:209-224
// 从路由自动提取当前页面上下文
const currentPageContext = computed(() => {
    const path = route.path;
    const collection = route.params.collection as string | undefined;
    const item = route.params.primaryKey as string | number | undefined;
    const pathParts = path.split('/').filter(Boolean);
    const module = pathParts[0];

    return {
        path,
        ...(collection && { collection }),
        ...(item !== undefined && { item }),
        ...(module && { module }),
    };
});

// 发送请求时携带上下文
// app/src/ai/stores/use-ai.ts:245-255
const context = {
    page: currentPageContext.value,  // 自动注入的页面上下文
    ...(pendingContextSnapshot.value.length > 0 && { 
        attachments: pendingContextSnapshot.value  // 手动添加的上下文
    }),
};
```

---

## 2. API 选路：为何只有 /ai/chat 被使用

### 2.1 两个端点的对比

| 维度 | `POST /ai/chat` | `POST /ai/object` |
|------|-----------------|-------------------|
| **存在性** | ✅ 存在 | ✅ 存在 |
| **前端调用** | ✅ `use-ai.ts` 中直接使用 | ❌ 无调用记录 |
| **工具支持** | ✅ 完整支持（items、trigger-flow 等） | ❌ 不支持 |
| **响应方式** | SSE 流式响应 | JSON Stream |
| **请求体要求** | `messages`, `tools`, `context` | `prompt`, `outputSchema` |
| **典型场景** | 交互式对话、数据操作 | 结构化输出（预留） |

### 2.2 /ai/chat 是唯一实际使用的端点

**前端代码证据**：

```typescript
// app/src/ai/stores/use-ai.ts:230-264
// 唯一的 AI 端点配置
const transport = new DefaultChatTransport({
    api: '/ai/chat',  // 🔑 硬编码为 /ai/chat
    credentials: 'include',
    body: () => {
        return {
            provider: selectedModel.value?.provider,
            model: selectedModel.value?.model,
            tools,           // 包含可用工具
            toolApprovals: approvals,
            context,         // 上下文数据
        };
    },
    // ...
});
```

**搜索结果验证**：
- 搜索 `/ai/object` 在 `app/src` 目录：**0 matches**
- 搜索 `/ai/chat` 在 `app/src` 目录：**多处匹配**

### 2.3 /ai/object 的定位（预留功能）

`/ai/object` 端点存在但未被前端使用，其设计意图是：

```typescript
// api/src/ai/chat/controllers/object.post.ts:68-75
const result = streamObject({
    model: languageModel,
    prompt,
    schema: jsonSchema(addAdditionalPropertiesToJsonSchema(outputSchema)),
    providerOptions,
    // ...
});
```

**特点**：
- 需要 `outputSchema` 参数定义输出结构
- 不支持工具调用
- 使用 `streamObject` 而非 `streamText`

**可能的用途**：
- 未来的字段级专用 AI 功能
- 程序化调用（扩展开发）
- 但当前版本前端未集成

### 2.4 选路决策树

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          API 端点选路决策树                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   用户触发 AI 功能                                                            │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────┐                                                │
│   │ 是否需要工具调用？       │                                                │
│   │ (数据读写、触发流程等)   │                                                │
│   └──────────┬──────────────┘                                                │
│              │                                                                │
│        ┌─────┴─────┐                                                        │
│        │           │                                                        │
│        ▼ YES       ▼ NO (仅需结构化输出)                                     │
│   ┌──────────┐   ┌──────────┐                                                │
│   │ /ai/chat │   │/ai/object│                                                │
│   │ ⭐ 实际使用│   │  预留功能 │                                                │
│   └────┬─────┘   └──────────┘                                                │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  工具调用能力:                                                         │  │
│   │  - items: 数据 CRUD ⭐ (用于字段读写)                                  │  │
│   │  - trigger-flow: 触发手动流程                                          │  │
│   │  - files/folders: 文件操作                                             │  │
│   │  - collections/fields: 集合/字段管理                                   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 端到端闭环时序（不依赖 request 操作）

### 3.1 场景定义

**场景**：用户在 AI 侧边栏选择 `products` 集合中 `id=1` 的条目，输入指令"将 title_zh 翻译成英文，写入 title_en"

**核心机制**：
- **不依赖 Flow**
- **不依赖 Request 操作**
- **完全通过 `/ai/chat` + `items` 工具实现**

### 3.2 完整时序图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    端到端闭环时序（通过 items 工具）                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │  用户    │  │  App     │  │  API     │  │  LLM     │  │  数据库  │    │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘    │
│       │             │             │             │             │            │
│       │ 1. 选择上下文│             │             │             │            │
│       │────────────▶│             │             │             │            │
│       │             │             │             │             │            │
│       │ 2. 输入指令  │             │             │             │            │
│       │ "翻译title_zh│             │             │             │            │
│       │  写入title_en│             │             │             │            │
│       │────────────▶│             │             │             │            │
│       │             │             │             │             │            │
│       │             │ 3. POST /ai/chat          │             │            │
│       │             │────────────▶│             │             │            │
│       │             │             │             │             │            │
│       │             │             │ 4. 验证权限  │             │            │
│       │             │             │ • app=true  │             │            │
│       │             │             │ • 模型白名单 │             │            │
│       │             │             │ • 提供商配置 │             │            │
│       │             │             │             │             │            │
│       │             │             │ 5. 调用 LLM │             │            │
│       │             │             │────────────▶│             │            │
│       │             │             │             │             │            │
│       │             │             │             │ 6. LLM 分析 │            │
│       │             │             │             │ 意图: 需要  │            │
│       │             │             │             │ 先读取数据   │            │
│       │             │             │             │ 再翻译       │            │
│       │             │             │             │ 再写入       │            │
│       │             │             │             │             │            │
│       │             │             │             │ 7. 决定调用  │            │
│       │             │             │             │ items 工具  │            │
│       │             │             │◀────────────│             │            │
│       │             │             │             │             │            │
│       │             │             │ 8. 执行 items│             │            │
│       │             │             │    .read    │             │            │
│       │             │             │             │             │ 9. 查询   │
│       │             │             │             │             │───────────▶│
│       │             │             │             │             │            │
│       │             │             │             │             │ 10. 返回  │
│       │             │             │             │             │◀───────────│
│       │             │             │             │             │ {id:1,    │
│       │             │             │             │             │ title_zh:  │
│       │             │             │             │             │ "智能手表"}│
│       │             │             │             │             │            │
│       │             │             │ 11. 返回工具│             │            │
│       │             │             │    结果     │             │            │
│       │             │             │────────────▶│             │            │
│       │             │             │             │             │            │
│       │             │             │             │ 12. LLM 翻译│            │
│       │             │             │             │ "智能手表"  │            │
│       │             │             │             │ →"Smart Watch"│           │
│       │             │             │             │             │            │
│       │             │             │             │ 13. 调用    │             │
│       │             │             │             │ items.update│            │
│       │             │             │◀────────────│             │            │
│       │             │             │             │             │            │
│       │             │             │ 14. 执行    │             │             │
│       │             │             │ items.update│             │             │
│       │             │             │             │             │ 15. 更新   │
│       │             │             │             │             │───────────▶│
│       │             │             │             │             │ title_en=  │
│       │             │             │             │             │"Smart Watch"│
│       │             │             │             │             │            │
│       │             │             │             │             │ 16. 确认   │
│       │             │             │             │             │◀───────────│
│       │             │             │             │             │            │
│       │             │             │ 17. 返回工具│             │            │
│       │             │             │    结果     │             │            │
│       │             │             │────────────▶│             │            │
│       │             │             │             │             │            │
│       │             │             │             │ 18. 生成最终│            │
│       │             │             │             │ 响应消息    │            │
│       │             │◀────────────│◀────────────│             │            │
│       │             │             │             │             │            │
│       │ 19. 显示结果│             │             │             │            │
│       │◀────────────│             │             │             │            │
│       │             │             │             │             │            │
└───────┴─────────────┴─────────────┴─────────────┴─────────────┴────────────┘
```

### 3.3 关键步骤详解

#### 步骤 3-5: 发送请求到 /ai/chat

```typescript
// app/src/ai/stores/use-ai.ts:230-264
const transport = new DefaultChatTransport({
    api: '/ai/chat',
    credentials: 'include',
    body: () => {
        const tools = [...toolsStore.enabledSystemTools, ...toolsStore.localTools.map(toApiTool)];
        
        return {
            provider: 'openai',           // 选中的提供商
            model: 'gpt-4o',              // 选中的模型
            tools,                         // 可用工具列表
            toolApprovals: approvals,      // 工具审批配置
            context: {
                page: {                    // 当前页面上下文
                    path: '/content/products/1',
                    collection: 'products',
                    item: 1,
                    module: 'content'
                },
                attachments: [{            // 手动添加的上下文
                    type: 'item',
                    collection: 'products',
                    item: 1,
                    fields: ['id', 'title_zh', 'title_en']
                }]
            },
            messages: [                    // 对话历史
                {
                    role: 'user',
                    content: '将 title_zh 翻译成英文，写入 title_en'
                }
            ]
        };
    },
});
```

#### 步骤 6-7: LLM 决定调用工具

LLM 收到请求后，分析用户意图：

1. 理解指令："翻译 title_zh → 写入 title_en"
2. 意识到需要：
   - 先读取数据获取 `title_zh` 的值
   - 翻译内容
   - 写入 `title_en`
3. 决定调用 `items` 工具

**LLM 生成的工具调用**：

```json
{
    "name": "items",
    "arguments": {
        "action": "read",
        "collection": "products",
        "keys": ["1"],
        "query": {
            "fields": ["id", "title_zh", "title_en"]
        }
    }
}
```

#### 步骤 8-10: 执行 items.read

```typescript
// api/src/ai/tools/items/index.ts:122-138
if (args.action === 'read') {
    const sanitizedQuery = await buildSanitizedQueryFromArgs(args, schema, accountability);
    
    // 1. 创建 ItemsService，传入 accountability（权限上下文）
    const itemsService = new ItemsService(args.collection, {
        schema,
        accountability,  // 🔑 关键：权限上下文
    });

    // 2. 执行查询
    const result = await itemsService.readMany(args.keys, sanitizedQuery);
    
    // 3. 返回结果
    return {
        type: 'text',
        data: result,  // [{ id: 1, title_zh: "智能手表", title_en: null }]
    };
}
```

#### 步骤 11-13: LLM 翻译后调用 items.update

工具结果返回给 LLM 后，LLM：
1. 提取 `title_zh: "智能手表"`
2. 翻译为 `"Smart Watch"`
3. 决定调用 `items.update` 写入

**LLM 生成的工具调用**：

```json
{
    "name": "items",
    "arguments": {
        "action": "update",
        "collection": "products",
        "keys": ["1"],
        "data": {
            "title_en": "Smart Watch"
        }
    }
}
```

#### 步骤 14-16: 执行 items.update

```typescript
// api/src/ai/tools/items/index.ts:140-174
if (args.action === 'update') {
    const sanitizedQuery = await buildSanitizedQueryFromArgs(args, schema, accountability);

    const itemsService = new ItemsService(args.collection, {
        schema,
        accountability,  // 🔑 权限上下文
    });

    // 1. 执行更新
    const updatedKeys = await itemsService.updateMany(
        args.keys,           // ["1"]
        args.data            // { title_en: "Smart Watch" }
    );

    // 2. 读取更新后的数据返回
    const result = await itemsService.readMany(updatedKeys, sanitizedQuery);

    return {
        type: 'text',
        data: result,  // [{ id: 1, title_zh: "智能手表", title_en: "Smart Watch" }]
    };
}
```

#### 步骤 17-19: 生成最终响应

LLM 收到更新成功的结果后，生成最终响应消息：

```
已完成翻译：
- 原始标题 (title_zh): 智能手表
- 英文标题 (title_en): Smart Watch

数据已更新到 products 集合的 id=1 条目。
```

### 3.4 闭环的核心：自动工具调用机制

```typescript
// app/src/ai/stores/use-ai.ts:282-329
// Chat 实例配置
const chat = new Chat<UIMessage>({
    messages: storedMessages.value,
    transport,
    
    // 🔑 自动工具调用的关键配置
    sendAutomaticallyWhen: ({ messages }) =>
        // 当助理消息包含完整的工具调用时，自动发送新请求
        lastAssistantMessageIsCompleteWithToolApprovalResponses({ messages }) ||
        lastAssistantMessageIsCompleteWithToolCalls({ messages }),
    
    // 工具调用处理
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

**自动调用流程**：

```
1. 用户发送消息
   ↓
2. LLM 返回："我需要调用 items.read 工具"
   ↓
3. 前端检测到工具调用，自动发送新请求
   ↓
4. 后端执行 items.read，返回数据
   ↓
5. LLM 返回："我需要调用 items.update 工具"
   ↓
6. 前端再次自动发送新请求
   ↓
7. 后端执行 items.update，返回结果
   ↓
8. LLM 返回最终响应消息
```

---

## 4. 失败分支与权限边界

### 4.1 五层权限检查

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          五层权限检查架构                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layer 1: 环境级别                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AI_ENABLED=true/false                                                 │   │
│  │                                                                        │   │
│  │ 检查位置: api/src/app.ts:348                                          │   │
│  │                                                                        │   │
│  │ if (toBoolean(env['AI_ENABLED']) === true) {                          │   │
│  │     app.use('/ai', aiRouter);                                          │   │
│  │ }                                                                      │   │
│  │                                                                        │   │
│  │ 失败: 404 Not Found (端点不存在)                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 2: 请求来源级别                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ accountability.app = true                                             │   │
│  │                                                                        │   │
│  │ 检查位置: chat.post.ts:12, object.post.ts:18                         │   │
│  │                                                                        │   │
│  │ if (!req.accountability?.app) {                                        │   │
│  │     throw new ForbiddenError();                                        │   │
│  │ }                                                                      │   │
│  │                                                                        │   │
│  │ 含义: 必须来自 Admin App 的请求                                         │   │
│  │       API Token、外部调用都会被拒绝                                     │   │
│  │                                                                        │   │
│  │ 失败: 403 Forbidden                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 3: 模型级别                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 模型白名单检查                                                          │   │
│  │                                                                        │   │
│  │ 检查位置: chat.post.ts:34-40, object.post.ts:38-44                  │   │
│  │                                                                        │   │
│  │ const allowedModels = allowedModelsMap[provider];                      │   │
│  │                                                                        │   │
│  │ if (!allowedModels || !allowedModels.includes(model)) {                │   │
│  │     throw new ForbiddenError({ reason: 'Model not allowed' });        │   │
│  │ }                                                                      │   │
│  │                                                                        │   │
│  │ 例外: openai-compatible 提供商跳过此检查                                │   │
│  │                                                                        │   │
│  │ 失败: 403 Forbidden: "Model not allowed for this provider"             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 4: 提供商级别                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ API Key 配置检查                                                       │   │
│  │                                                                        │   │
│  │ 检查位置: create-ui-stream.ts:46-48                                   │   │
│  │                                                                        │   │
│  │ const providerConfig = configs.find((c) => c.type === provider);      │   │
│  │                                                                        │   │
│  │ if (!providerConfig) {                                                 │   │
│  │     throw new ServiceUnavailableError({                                │   │
│  │         service: provider,                                             │   │
│  │         reason: 'No API key configured for LLM provider'               │   │
│  │     });                                                                │   │
│  │ }                                                                      │   │
│  │                                                                        │   │
│  │ 失败: 503 Service Unavailable                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 5: 数据级别（工具执行时）                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ItemsService 权限检查                                                  │   │
│  │                                                                        │   │
│  │ 检查位置: items/index.ts (多处), services/items.ts (多处)            │   │
│  │                                                                        │   │
│  │ const itemsService = new ItemsService(args.collection, {              │   │
│  │     schema,                                                             │   │
│  │     accountability,  // 🔑 权限上下文传递给 ItemsService              │   │
│  │ });                                                                    │   │
│  │                                                                        │   │
│  │ ItemsService 内部会检查:                                                │   │
│  │ - 集合访问权限                                                          │   │
│  │ - 字段读写权限                                                          │   │
│  │ - 数据行级权限                                                          │   │
│  │                                                                        │   │
│  │ 失败: 由 ItemsService 抛出相应错误                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 完整失败分支流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          完整失败分支流程                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   用户发送 AI 请求                                                           │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Layer 1: AI_ENABLED=true?                                            │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ├── NO ──▶ 404 Not Found (端点不存在)                                │
│        │                                                                     │
│        ▼ YES                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Layer 2: accountability.app=true?                                    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ├── NO ──▶ 403 Forbidden: "You don't have permission to access this."│
│        │                                                                     │
│        ▼ YES                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Layer 3: 请求体验证 (Zod Schema)                                      │  │
│   │ - provider 是否有效?                                                   │  │
│   │ - messages 是否非空?                                                   │  │
│   │ - tools 格式是否正确?                                                  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ├── NO ──▶ 400 InvalidPayloadError                                   │
│        │                                                                     │
│        ▼ YES                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Layer 4: 模型白名单检查 (openai-compatible 除外)                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ├── NO ──▶ 403 Forbidden: "Model not allowed for this provider"     │
│        │                                                                     │
│        ▼ YES                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ Layer 5: 提供商 API Key 配置检查                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ├── NO ──▶ 503 ServiceUnavailable: "No API key configured"          │
│        │                                                                     │
│        ▼ YES                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ 调用外部 LLM API                                                       │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ├── 网络错误 ──▶ SSE error 事件                                      │
│        ├── 超时错误 ──▶ SSE error 事件                                      │
│        ├── 429 限流 ──▶ SSE error 事件                                      │
│        │                                                                     │
│        ▼ 成功                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ LLM 决定调用工具                                                        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ▼                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ 工具执行前置检查 (items 工具)                                          │  │
│   │                                                                         │  │
│   │ 检查 1: 是否是系统集合?                                                 │  │
│   │ if (isSystemCollection(args.collection)) {                             │  │
│   │     throw new InvalidPayloadError({                                    │  │
│   │         reason: 'Cannot provide a core collection'                     │  │
│   │     });                                                                │  │
│   │ }                                                                      │  │
│   │                                                                         │  │
│   │ 检查 2: 集合是否存在?                                                   │  │
│   │ if (args.collection in schema.collections === false) {                 │  │
│   │     throw new ForbiddenError();                                        │  │
│   │ }                                                                      │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ├── 系统集合 ──▶ 400 InvalidPayloadError                             │
│        ├── 集合不存在 ──▶ 403 Forbidden                                     │
│        │                                                                     │
│        ▼ 通过                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ ItemsService 权限检查                                                  │  │
│   │ (由 ItemsService 内部处理)                                              │  │
│   │                                                                         │  │
│   │ 可能的失败:                                                             │  │
│   │ - 无权限访问集合: ForbiddenError                                       │  │
│   │ - 无权限读取字段: ForbiddenError                                       │  │
│   │ - 无权限更新字段: ForbiddenError                                       │  │
│   │ - 数据不存在: RecordNotUniqueError / RecordNotExistError              │  │
│   │ - 数据校验失败: InvalidPayloadError                                    │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│        │                                                                     │
│        ▼ 成功                                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │ 工具执行成功，返回结果                                                  │  │
│   │ 结果通过 SSE 流式返回给客户端                                           │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 错误类型汇总表

| 检查层级 | 错误类型 | 错误代码 | 典型原因 |
|----------|----------|----------|----------|
| Layer 1 | - | 404 | `AI_ENABLED=false` |
| Layer 2 | `ForbiddenError` | 403 | 非 App 请求（API Token、外部调用） |
| Layer 3 | `InvalidPayloadError` | 400 | 请求体格式错误 |
| Layer 4 | `ForbiddenError` | 403 | 模型不在白名单 |
| Layer 5 | `ServiceUnavailableError` | 503 | 提供商 API Key 未配置 |
| 网络层 | SSE error | - | 网络错误、超时、限流 |
| 工具层 | `InvalidPayloadError` | 400 | 操作系统集合 |
| 工具层 | `ForbiddenError` | 403 | 集合不存在 |
| 数据层 | `ForbiddenError` | 403 | 无字段/集合权限 |
| 数据层 | `RecordNotExistError` | 404 | 数据不存在 |
| 数据层 | `InvalidPayloadError` | 400 | 数据校验失败 |

### 4.4 accountability 权限上下文传递链

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     accountability 权限上下文传递链                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 请求进入时，accountability 由中间件设置                                   │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │ accountability = {                                                   │  │
│     │     user: "user-uuid-123",      // 当前用户 ID                      │  │
│     │     role: "role-uuid-456",      // 当前角色 ID                      │  │
│     │     admin: true/false,          // 是否管理员                        │  │
│     │     app: true/false,            // 是否来自 App (🔑 关键)           │  │
│     │     ip: "192.168.1.1",          // 客户端 IP                        │  │
│     │     userAgent: "Mozilla/5.0...",// User-Agent                       │  │
│     │ }                                                                     │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                   │                                          │
│                                   ▼                                          │
│  2. AI 控制器检查 accountability.app                                         │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │ // chat.post.ts:12                                                  │  │
│     │ if (!req.accountability?.app) {                                     │  │
│     │     throw new ForbiddenError();  // ❌ 非 App 请求被拒绝            │  │
│     │ }                                                                    │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                   │                                          │
│                                   ▼                                          │
│  3. 工具调用时传递 accountability                                            │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │ // chat-request-tool-to-ai-sdk-tool.ts:35-53                       │  │
│     │ return tool({                                                        │  │
│     │     execute: async (rawArgs) => {                                    │  │
│     │         return directusTool.handler({                                 │  │
│     │             args,                                                     │  │
│     │             accountability,  // 🔑 传递给工具 handler                │  │
│     │             schema,                                                   │  │
│     │         });                                                           │  │
│     │     },                                                                │  │
│     │ });                                                                   │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                   │                                          │
│                                   ▼                                          │
│  4. Items 工具创建 ItemsService                                              │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │ // items/index.ts:88-91                                              │  │
│     │ const itemsService = new ItemsService(args.collection, {             │  │
│     │     schema,                                                            │  │
│     │     accountability,  // 🔑 传递给 ItemsService                       │  │
│     │ });                                                                   │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                   │                                          │
│                                   ▼                                          │
│  5. ItemsService 内部进行权限检查                                            │
│     ┌────────────────────────────────────────────────────────────────────┐  │
│     │ // services/items.ts (多处)                                          │  │
│     │                                                                        │  │
│     │ // 示例：检查字段权限                                                  │  │
│     │ if (accountability?.admin !== true) {                                │  │
│     │     // 非管理员，需要检查权限                                          │  │
│     │     const permissions = await fetchPermissions({                      │  │
│     │         accountability,                                                │  │
│     │         collection: args.collection,                                   │  │
│     │         action: 'read',  // 或 'create' / 'update' / 'delete'      │  │
│     │     });                                                                │  │
│     │                                                                        │  │
│     │     if (!hasPermission(permissions)) {                                │  │
│     │         throw new ForbiddenError();                                    │  │
│     │     }                                                                  │  │
│     │ }                                                                      │  │
│     │                                                                        │  │
│     │ // 管理员 (admin: true) 跳过权限检查                                   │  │
│     └────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.5 工具审批机制

除了权限检查，系统还支持工具审批配置：

```typescript
// app/src/ai/stores/use-ai.ts:236-243
// 工具审批配置
const approvals: Record<string, 'always' | 'ask'> = {};

for (const [toolName, mode] of Object.entries(toolsStore.toolApprovals)) {
    if (mode === 'always' || mode === 'ask') {
        approvals[toolName] = mode;
    }
}
```

**审批模式**：

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `always` | 自动批准，无需用户确认 | 信任的自动化场景 |
| `ask` | 每次调用都需要用户确认 | 敏感操作（如删除数据） |
| `disabled` | 禁止使用该工具 | 高风险操作 |

**审批流程**：

```
LLM 决定调用工具
    │
    ▼
检查工具审批模式
    │
    ├── disabled ──▶ 不调用工具，LLM 重新规划
    │
    ├── ask ──▶ 在 AI 侧边栏显示确认对话框
    │              │
    │              ├── 用户批准 ──▶ 执行工具
    │              │
    │              └── 用户拒绝 ──▶ 不执行，通知 LLM
    │
    └── always ──▶ 自动执行工具
```

---

## 5. 与 Flow/Request 自动化链路的分工对比

### 5.1 核心差异总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    两种链路的分工对比                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────┐  ┌───────────────────────────────┐│
│  │     AI 聊天 + items 工具链路        │  │     Flow + Request 链路       ││
│  │     (交互式、用户驱动)               │  │     (自动化、事件驱动)         ││
│  ├─────────────────────────────────────┤  ├───────────────────────────────┤│
│  │                                     │  │                               ││
│  │  触发方式:                           │  │  触发方式:                     ││
│  │  ┌─────────────────────────────┐   │  │  ┌─────────────────────────┐ ││
│  │  │ 用户在 AI 侧边栏输入指令     │   │  │ │ 事件触发:               │ ││
│  │  │ 或添加上下文后发送          │   │  │ │ items.create/update     │ ││
│  │  └─────────────────────────────┘   │  │ │ 定时触发: Cron 表达式   │ ││
│  │                                     │  │ │ 手动触发: UI 按钮       │ ││
│  │  API 端点:                          │  │ └─────────────────────────┘ ││
│  │  ┌─────────────────────────────┐   │  │                               ││
│  │  │ POST /ai/chat              │   │  │  核心组件:                   ││
│  │  │ (唯一实际使用)              │   │  │  ┌─────────────────────────┐ ││
│  │  └─────────────────────────────┘   │  │ │ FlowManager (流程管理)  │ ││
│  │                                     │  │ │ Request Operation (HTTP) │ ││
│  │  数据操作:                           │  │ │ ItemUpdate Operation    │ ││
│  │  ┌─────────────────────────────┐   │  │ └─────────────────────────┘ ││
│  │  │ items 工具 (CRUD)           │   │  │                               ││
│  │  │ • read: 读取数据            │   │  │  数据操作:                   ││
│  │  │ • update: 更新数据 ⭐       │   │  │  ┌─────────────────────────┐ ││
│  │  │ • create: 创建数据          │   │  │ │ ItemUpdate Operation     │ ││
│  │  │ • delete: 删除数据          │   │  │ │ (通过 ItemsService)      │ ││
│  │  └─────────────────────────────┘   │  │ └─────────────────────────┘ ││
│  │                                     │  │                               ││
│  │  权限模型:                           │  │  权限模型:                   ││
│  │  ┌─────────────────────────────┐   │  │  ┌─────────────────────────┐ ││
│  │  │ accountability.app=true     │   │  │ │ 操作 permissions 配置    │ ││
│  │  │ 用户级别权限                │   │  │ │ • $trigger (触发用户)    │ ││
│  │  │ 工具审批机制                │   │  │ │ • $full (系统权限)       │ ││
│  │  └─────────────────────────────┘   │  │ │ • $public (公共权限)    │ ││
│  │                                     │  │ │ • <role-uuid> (指定角色) │ ││
│  │  适用场景:                           │  │ └─────────────────────────┘ ││
│  │  • 交互式数据操作                    │  │                               ││
│  │  • 临时数据处理                      │  │  适用场景:                   ││
│  │  • 复杂多步骤任务                    │  │  • 自动化数据处理          ││
│  │  • 需要 LLM 智能决策                 │  │  • 事件驱动的自动化       ││
│  │                                     │  │  • 定时任务                 ││
│  └─────────────────────────────────────┘  └───────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 详细对比表

| 维度 | AI 聊天 + items 工具 | Flow + Request |
|------|---------------------|----------------|
| **触发方式** | 用户在 AI 侧边栏主动交互 | 事件触发、定时触发、手动触发 |
| **核心端点** | `POST /ai/chat` | 无直接 `/ai` 调用，使用 `request` 操作 |
| **数据操作** | `items` 工具（后端实现） | `item-update` 操作（Flow 内置） |
| **LLM 调用** | 内置支持，通过 Provider Registry | 需要手动配置 Request 操作 |
| **智能决策** | LLM 动态决定调用哪些工具 | 预定义操作序列，无智能决策 |
| **上下文感知** | 自动感知当前页面 + 手动添加上下文 | 通过 `$trigger`、`$last` 等变量 |
| **权限模型** | `accountability.app` + 用户权限 | 操作级 `permissions` 配置 |
| **审批机制** | 工具审批 (`always`/`ask`/`disabled`) | 无内置审批，需自行实现 |
| **错误处理** | 工具错误通过 SSE 返回 | Flow Reject 分支 + Log 操作 |
| **配置复杂度** | 低（开箱即用） | 高（需要配置 Flow 和多个 Operation） |
| **灵活性** | 高（LLM 自主决策） | 中（预定义路径） |
| **可控性** | 中（依赖 LLM 行为） | 高（完全配置化） |
| **可审计性** | 对话历史记录 | Flow 活动日志 |

### 5.3 场景选择建议

#### 推荐使用 AI 聊天 + items 工具的场景

| 场景 | 原因 |
|------|------|
| **交互式数据探索** | 用户需要与数据进行对话式交互 |
| **复杂多步骤操作** | LLM 可以动态规划步骤，如"先查 A，再计算，最后更新 B" |
| **临时数据处理** | 不需要配置流程，快速处理一次性任务 |
| **需要上下文理解** | LLM 可以理解自然语言指令的意图 |
| **非结构化输入** | 用户输入自然语言指令，而非选择预定义操作 |

#### 推荐使用 Flow + Request 的场景

| 场景 | 原因 |
|------|------|
| **自动化数据处理** | 数据变更时自动执行，无需用户干预 |
| **事件驱动流程** | `items.create`、`items.update` 等事件触发 |
| **定时批量任务** | Cron 表达式定时执行 |
| **需要严格错误处理** | 可配置 Reject 分支、回滚操作 |
| **需要权限隔离** | 可配置 `$full`、`$public`、特定角色权限 |
| **需要审计追踪** | Flow 活动日志记录完整执行过程 |
| **需要审批工作流** | 可结合 Condition 操作实现审批逻辑 |

### 5.4 混合使用示例

在实际项目中，可以结合两种链路：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          混合使用方案                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   场景：产品多语言内容管理                                                    │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  自动化链路 (Flow + Request)                                          │  │
│   │  ─────────────────────────                                             │  │
│   │                                                                         │  │
│   │  触发: items.create (products 集合)                                   │  │
│   │                                                                         │  │
│   │  操作 1: Request → 调用 LLM 翻译 title_zh → title_en                 │  │
│   │  操作 2: Item Update → 写入 title_en                                  │  │
│   │  操作 3: Log → 记录翻译日志                                           │  │
│   │                                                                         │  │
│   │  效果: 创建产品时自动翻译标题                                          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  交互式链路 (AI 聊天 + items 工具)                                    │  │
│   │  ─────────────────────────                                             │  │
│   │                                                                         │  │
│   │  用户操作:                                                              │  │
│   │  1. 打开 AI 侧边栏                                                     │  │
│   │  2. 添加 products:5 作为上下文                                         │  │
│   │  3. 输入: "为这个产品生成 100 字的产品描述，写入 description 字段"    │  │
│   │                                                                         │  │
│   │  执行流程:                                                              │  │
│   │  1. LLM 调用 items.read 读取产品数据                                  │  │
│   │  2. LLM 基于标题、价格等信息生成描述                                    │  │
│   │  3. LLM 调用 items.update 写入 description                             │  │
│   │                                                                         │  │
│   │  效果: 用户可以灵活处理各种临时任务                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.5 关键代码差异

#### AI 聊天 + items 工具路径

```typescript
// 前端: use-ai.ts
const transport = new DefaultChatTransport({
    api: '/ai/chat',  // 单一端点
    body: () => ({
        provider,
        model,
        tools,           // 包含 items 工具
        context,         // 页面+手动上下文
        messages,        // 对话历史
    }),
});

// 后端: items 工具
// api/src/ai/tools/items/index.ts
export const items = defineTool({
    name: 'items',
    async handler({ args, schema, accountability }) {
        const itemsService = new ItemsService(args.collection, {
            schema,
            accountability,  // 用户权限上下文
        });
        
        // 根据 action 执行 CRUD
        switch (args.action) {
            case 'read':   return itemsService.readMany(...);
            case 'update': return itemsService.updateMany(...);
            case 'create': return itemsService.createMany(...);
            case 'delete': return itemsService.deleteMany(...);
        }
    },
});
```

#### Flow + Request 路径

```typescript
// 后端: Flow 配置 (存储在数据库中)
{
    id: 'flow-auto-translate',
    name: '自动翻译标题',
    trigger: 'event',
    options: {
        type: 'action',
        scope: ['items.create'],
        collections: ['products'],
    },
    operations: [
        {
            key: 'operation-1',
            type: 'request',
            options: {
                url: 'https://api.openai.com/v1/chat/completions',
                method: 'POST',
                headers: { Authorization: 'Bearer {{ $env.OPENAI_KEY }}' },
                body: {
                    model: 'gpt-4o',
                    messages: [
                        { role: 'user', content: '翻译: {{ $trigger.payload.title_zh }}' }
                    ],
                },
            },
        },
        {
            key: 'operation-2',
            type: 'item-update',
            options: {
                collection: 'products',
                key: '{{ $trigger.keys[0] }}',
                permissions: '$trigger',  // 权限配置
                payload: {
                    title_en: '{{ $last.data.choices[0].message.content }}',
                },
            },
        },
    ],
}

// 后端: 执行
// api/src/flows.ts
flowManager.executeFlow(flow, triggerData, { accountability, schema });
```

---

## 6. 关键代码引用

### 6.1 前端触发层

| 文件路径 | 说明 |
|----------|------|
| `app/src/ai/stores/use-ai.ts:230-264` | `/ai/chat` 端点配置（唯一实际使用） |
| `app/src/ai/stores/use-ai.ts:282-329` | 自动工具调用机制 |
| `app/src/ai/components/ai-context-menu.vue:83-89` | 上下文菜单选项（添加数据上下文） |
| `app/src/ai/components/ai-context-menu.vue:137-185` | 选择集合和条目的处理逻辑 |
| `app/src/ai/stores/use-ai.ts:209-224` | 当前页面上下文自动提取 |

### 6.2 API 路由层

| 文件路径 | 说明 |
|----------|------|
| `api/src/app.ts:348-353` | AI 端点注册（依赖 `AI_ENABLED`） |
| `api/src/ai/chat/router.ts:7-9` | AI 路由定义 (`/chat` + `/object`) |
| `api/src/ai/chat/controllers/chat.post.ts:12-14` | `accountability.app` 权限检查 |
| `api/src/ai/chat/controllers/chat.post.ts:34-40` | 模型白名单检查 |
| `api/src/ai/chat/controllers/object.post.ts:17-20` | `/ai/object` 端点（前端未调用） |

### 6.3 工具与数据操作层

| 文件路径 | 说明 |
|----------|------|
| `api/src/ai/tools/index.ts:15-28` | 所有 AI 工具注册 |
| `api/src/ai/tools/items/index.ts:62-187` | `items` 工具核心实现 |
| `api/src/ai/tools/items/index.ts:78-84` | 系统集合检查 |
| `api/src/ai/tools/items/index.ts:88-91` | `ItemsService` 创建（传递 accountability） |
| `api/src/ai/tools/items/index.ts:140-174` | `update` 操作实现 |
| `api/src/ai/chat/utils/chat-request-tool-to-ai-sdk-tool.ts:35-53` | 工具调用时传递 accountability |

### 6.4 权限与错误处理

| 文件路径 | 说明 |
|----------|------|
| `api/src/ai/chat/middleware/load-settings.ts:6-49` | AI 配置加载中间件 |
| `api/src/ai/providers/registry.ts:10-78` | 提供商注册表 |
| `api/src/ai/chat/lib/create-ui-stream.ts:46-48` | 提供商 API Key 检查 |
| `api/src/utils/get-accountability-for-role.ts` | accountability 生成（用于 Flow） |
| `app/src/ai/stores/use-ai.ts:236-243` | 工具审批配置 |

### 6.5 Flow 对比层（供参考）

| 文件路径 | 说明 |
|----------|------|
| `api/src/flows.ts` | Flow Manager 核心 |
| `api/src/operations/request/index.ts` | Request 操作（Flow 使用） |
| `api/src/operations/item-update/index.ts` | Item Update 操作（Flow 使用） |

---

## 总结

### 核心发现

1. **`/ai/object` 端点存在但未被前端使用**
   - 搜索 `app/src` 目录：0 个匹配
   - 可能是预留功能或程序化调用接口

2. **`/ai/chat` 是唯一实际使用的 AI 端点**
   - `use-ai.ts` 中硬编码
   - 支持流式响应和工具调用

3. **数据操作完全通过 `items` 工具实现**
   - 不依赖 Flow
   - 不依赖 Request 操作
   - 直接使用 `ItemsService` 进行 CRUD

4. **五层权限检查确保安全**
   - Layer 1: `AI_ENABLED` 环境变量
   - Layer 2: `accountability.app` 必须为 true
   - Layer 3: 请求体验证
   - Layer 4: 模型白名单
   - Layer 5: `ItemsService` 数据权限

5. **与 Flow 链路的核心区别**
   - AI 聊天：用户驱动、交互式、LLM 智能决策
   - Flow：事件驱动、自动化、预定义路径

### 建议

1. **字段级 AI 生成**：使用 AI 侧边栏 + 上下文菜单 + `items` 工具
2. **自动化场景**：使用 Flow + Request 操作
3. **混合方案**：根据场景灵活选择，或结合使用

文档已验证基于实际代码分析，而非假设或文档推导。
