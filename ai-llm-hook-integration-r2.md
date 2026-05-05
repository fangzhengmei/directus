# Directus AI 字段自动化链路深度解析（R2）

本文档聚焦于 **AI 字段自动化** 场景，详细说明从 Admin 字段侧触发、API 选路、LLM 调用、到 Hook/Flow 回写的完整闭环，同时补充失败分支、权限边界，以及与通用 AI 聊天链路的差异对比。

---

## 目录

1. [核心概念区分](#1-核心概念区分)
2. [端到端自动化闭环（可复现场景）](#2-端到端自动化闭环可复现场景)
3. [/ai 接口选路机制与失败分支](#3-ai-接口选路机制与失败分支)
4. [权限边界与验证机制](#4-权限边界与验证机制)
5. [与通用 AI 聊天链路对比](#5-与通用-ai-聊天链路对比)
6. [关键代码引用](#6-关键代码引用)

---

## 1. 核心概念区分

### 1.1 两种 AI 使用场景

| 维度 | AI 字段自动化 | 通用 AI 聊天 |
|------|--------------|-------------|
| **触发方式** | 事件自动触发（数据变更）或手动流程触发 | 用户在 AI 侧边栏主动输入指令 |
| **目标** | 字段级自动化（翻译、生成描述、数据清洗） | 交互式对话、复杂数据操作 |
| **API 端点** | 主要通过 Flow 内的 `request` 操作调用外部 API，或 `/ai/object` | `POST /ai/chat` 流式对话 |
| **权限模型** | 流程级别权限（`$trigger` / `$full` / `$public` / 角色） | 用户级别权限（accountability） |
| **失败处理** | 流程错误分支（Condition + Reject 路径） | 流式错误、工具调用失败 |
| **上下文** | 事件 payload、触发数据 | 对话历史、页面上下文 |

### 1.2 触发方式分类

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AI 触发方式分类                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────┐          ┌──────────────────────┐               │
│  │   主动触发（用户）    │          │   被动触发（系统）    │               │
│  └──────────────────────┘          └──────────────────────┘               │
│           │                                   │                              │
│           ▼                                   ▼                              │
│  ┌──────────────────────┐          ┌──────────────────────┐               │
│  │ 1. AI 侧边栏对话      │          │ 1. 事件触发器        │               │
│  │    POST /ai/chat     │          │    items.create      │               │
│  │                      │          │    items.update      │               │
│  │ 2. 手动触发流程       │          │    items.delete      │               │
│  │    右键菜单 → 流程    │          │                      │               │
│  │    → POST /flows/run │          │ 2. 定时触发器        │               │
│  │                      │          │    Cron 表达式        │               │
│  │ 3. AI 工具调用流程    │          │                      │               │
│  │    trigger-flow 工具  │          │                      │               │
│  └──────────────────────┘          └──────────────────────┘               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 端到端自动化闭环（可复现场景）

### 2.1 场景定义

**场景名称**：产品标题自动翻译

**业务需求**：
- 当产品的中文标题 (`title_zh`) 创建或更新时
- 自动调用 LLM 翻译成英文
- 将翻译结果写入 `title_en` 字段
- 记录翻译时间到 `translated_at` 字段

**技术实现路径**：
```
[数据变更] → [Event Trigger] → [Flow 执行] → [LLM 调用] → [数据回写]
     │                │                │              │              │
     ▼                ▼                ▼              ▼              ▼
items.create    Flow Manager      Request      OpenAI API   Item Update
items.update    (filter/action)   Operation    (翻译)       Operation
```

### 2.2 步骤 1：环境准备

**2.2.1 集合结构**

创建 `products` 集合，包含以下字段：

| 字段名 | 类型 | 说明 |
|--------|------|------|
| `id` | Integer | 主键 |
| `title_zh` | String | 中文标题（触发字段） |
| `title_en` | String | 英文标题（目标字段） |
| `translated_at` | DateTime | 翻译时间戳 |
| `translation_status` | String | 翻译状态：`pending` / `completed` / `failed` |

**2.2.2 环境变量配置**

```bash
# .env
AI_ENABLED=true

# OpenAI 配置
AI_OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
AI_OPENAI_ALLOWED_MODELS=gpt-4o,gpt-4-turbo,gpt-3.5-turbo

# 或者使用自定义 LLM
# AI_OPENAI_COMPATIBLE_API_KEY=ollama
# AI_OPENAI_COMPATIBLE_BASE_URL=http://localhost:11434/v1
# AI_OPENAI_COMPATIBLE_NAME=Ollama
# AI_OPENAI_COMPATIBLE_MODELS=[{"id": "llama3.1", "name": "Llama 3.1", "context": 128000}]
```

**2.2.3 Admin 界面 AI 设置**

1. 进入 **Settings → AI**
2. 配置 OpenAI API Key
3. 设置允许的模型列表
4. 保存设置

### 2.3 步骤 2：创建翻译流程

**2.3.1 流程概览**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        流程："产品标题自动翻译"                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Trigger: Event (Action - After)                                           │
│   Scope: items.create, items.update                                          │
│   Collection: products                                                        │
│                                                                              │
│   Operations:                                                                │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐           │
│   │ Condition│───▶│  Request │───▶│Item Update│───▶│  (End)   │           │
│   │ 检查字段  │    │  调用LLM │    │  写回数据  │    │          │           │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘           │
│        │               │               │                                    │
│        ▼ (Reject)      ▼ (Reject)      ▼ (Reject)                          │
│   ┌──────────────────────────────────────────────────────────────┐         │
│   │              Log Operation (记录错误)                         │         │
│   │              + Item Update (更新状态为 failed)                │         │
│   └──────────────────────────────────────────────────────────────┘         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**2.3.2 触发器配置**

| 配置项 | 值 | 说明 |
|--------|-----|------|
| Trigger | `Event` | 事件触发 |
| Type | `Action (After)` | 操作执行后触发（不阻塞原始操作） |
| Scope | `items.create`, `items.update` | 创建和更新时触发 |
| Collections | `products` | 仅针对产品集合 |

**2.3.3 操作 1：条件检查（Condition）**

**目的**：检查 `title_zh` 是否有值且有变更，避免不必要的翻译

**配置**：
```json
{
  "rule": {
    "AND": [
      {
        "trigger": {
          "payload": {
            "title_zh": {
              "_nnull": true
            }
          }
        }
      }
    ]
  }
}
```

**Resolve 路径**：继续执行 LLM 调用
**Reject 路径**：记录日志并退出

**2.3.4 操作 2：调用 LLM 翻译（Request）**

**目的**：调用 OpenAI API 进行翻译

**配置**：

| 配置项 | 值 |
|--------|-----|
| Method | `POST` |
| URL | `https://api.openai.com/v1/chat/completions` |

**Headers**：
```json
[
  {
    "header": "Authorization",
    "value": "Bearer {{ $env.AI_OPENAI_API_KEY }}"
  },
  {
    "header": "Content-Type",
    "value": "application/json"
  }
]
```

**Body**：
```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "你是翻译助手。将用户提供的中文产品标题翻译成简洁的英文。只返回翻译结果，不要有解释或其他内容。"
    },
    {
      "role": "user",
      "content": "{{ $trigger.payload.title_zh }}"
    }
  ],
  "temperature": 0.3,
  "max_tokens": 100
}
```

**Resolve 路径**：继续执行数据回写
**Reject 路径**：记录错误日志，更新翻译状态为 `failed`

**2.3.5 操作 3：写回数据（Item Update）**

**目的**：将翻译结果写回 `title_en` 字段

**配置**：

| 配置项 | 值 | 说明 |
|--------|-----|------|
| Collection | `products` | 目标集合 |
| Key | `{{ $trigger.keys[0] }}` | 触发事件的主键 |
| Permissions | `$trigger` | 使用触发用户的权限 |
| Emit Events | `true` | 触发事件（可能触发其他流程） |

**Payload**：
```json
{
  "title_en": "{{ $last.data.choices[0].message.content }}",
  "translated_at": "$NOW",
  "translation_status": "completed"
}
```

**2.3.6 错误分支处理**

为每个操作的 Reject 路径配置错误处理：

**Log Operation**：
```json
{
  "message": "翻译失败: {{ $trigger.keys[0] }} - {{ $last }}",
  "level": "error"
}
```

**Item Update（更新状态）**：
```json
{
  "collection": "products",
  "key": "{{ $trigger.keys[0] }}",
  "payload": {
    "translation_status": "failed"
  }
}
```

### 2.4 步骤 3：端到端测试

**测试步骤**：

1. **创建新产品**
   ```bash
   POST /items/products
   Body: {
     "title_zh": "智能蓝牙手表",
     "translation_status": "pending"
   }
   ```

2. **观察流程执行**
   - 流程自动触发
   - 检查 Condition 通过（title_zh 有值）
   - Request 操作调用 OpenAI API
   - Item Update 写回翻译结果

3. **验证结果**
   ```bash
   GET /items/products/{id}
   
   预期响应:
   {
     "id": 1,
     "title_zh": "智能蓝牙手表",
     "title_en": "Smart Bluetooth Watch",
     "translated_at": "2026-05-05T10:30:00.000Z",
     "translation_status": "completed"
   }
   ```

### 2.5 备选方案：使用 AI 工具直接操作

如果你想在 AI 对话中实现类似功能，可以让 LLM 直接调用 `items` 工具：

**用户输入**：
> "将 products 集合中 id=1 的 title_zh 翻译成英文，写入 title_en"

**LLM 工具调用序列**：

```
1. items.read 读取原始数据
   → { id: 1, title_zh: "智能蓝牙手表", title_en: null }

2. LLM 内部翻译
   → "Smart Bluetooth Watch"

3. items.update 写回数据
   → { title_en: "Smart Bluetooth Watch" }
```

---

## 3. /ai 接口选路机制与失败分支

### 3.1 API 端点架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           /ai 端点架构                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌──────────────────────────────┐                         │
│                    │      app.use('/ai', ...)     │                         │
│                    │    (仅当 AI_ENABLED=true)     │                         │
│                    └──────────────┬───────────────┘                         │
│                                   │                                            │
│            ┌──────────────────────┼──────────────────────┐                  │
│            ▼                      ▼                      ▼                  │
│   ┌────────────────┐    ┌────────────────┐    ┌────────────────┐          │
│   │ POST /ai/chat  │    │ POST /ai/object│    │ /ai/files/*    │          │
│   │  流式对话       │    │  结构化对象生成 │    │  文件处理       │          │
│   └────────────────┘    └────────────────┘    └────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 端点功能对比

| 端点 | 用途 | 响应格式 | 工具支持 | 典型场景 |
|------|------|----------|----------|----------|
| `POST /ai/chat` | 流式对话 | SSE (Server-Sent Events) | ✅ 完整支持 | AI 侧边栏对话、复杂操作 |
| `POST /ai/object` | 结构化生成 | JSON Stream | ❌ 不支持 | 字段填充、翻译、生成描述 |
| `/ai/files/*` | 文件处理 | - | - | 图片理解、文档分析 |

### 3.3 /ai/chat 失败分支详解

**3.3.1 权限检查失败**

```typescript
// api/src/ai/chat/controllers/chat.post.ts:11-14
export const aiChatPostHandler: RequestHandler = async (req, res, _next) => {
    // 🔒 关键权限检查：必须是 App 内部请求
    if (!req.accountability?.app) {
        throw new ForbiddenError();
    }
    // ...
};
```

**失败条件**：
- `req.accountability` 为 `null` 或 `undefined`
- `req.accountability.app` 为 `false` 或 `undefined`

**错误响应**：
```json
{
  "errors": [
    {
      "message": "You don't have permission to access this.",
      "extensions": {
        "code": "FORBIDDEN"
      }
    }
  ]
}
```

**3.3.2 请求体验证失败**

```typescript
// api/src/ai/chat/controllers/chat.post.ts:16-20
const parseResult = ChatRequest.safeParse(req.body);

if (!parseResult.success) {
    throw new InvalidPayloadError({ 
        reason: fromZodError(parseResult.error).message 
    });
}
```

**ChatRequest Schema** 包含：
- `provider`: 必须是 `openai` / `anthropic` / `google` / `openai-compatible`
- `model`: 字符串
- `messages`: 非空数组
- `tools`: 工具列表（可选）
- `toolApprovals`: 工具审批配置（可选）
- `context`: 上下文信息（可选）

**3.3.3 模型权限检查失败**

```typescript
// api/src/ai/chat/controllers/chat.post.ts:26-40
const allowedModelsMap: Record<StandardProviderType, string[] | null> = {
    openai: aiSettings.openaiAllowedModels,
    anthropic: aiSettings.anthropicAllowedModels,
    google: aiSettings.googleAllowedModels,
};

// openai-compatible 跳过验证
if (provider !== 'openai-compatible') {
    const allowedModels = allowedModelsMap[provider];
    
    if (!allowedModels || allowedModels.length === 0 || !allowedModels.includes(model)) {
        throw new ForbiddenError({ 
            reason: 'Model not allowed for this provider' 
        });
    }
}
```

**失败条件**：
- 提供商没有配置允许的模型列表
- 模型不在允许列表中

**3.3.4 LLM 提供商配置缺失**

```typescript
// api/src/ai/chat/lib/create-ui-stream.ts:43-48
const configs = buildProviderConfigs(aiSettings);
const providerConfig = configs.find((c) => c.type === provider);

if (!providerConfig) {
    throw new ServiceUnavailableError({ 
        service: provider, 
        reason: 'No API key configured for LLM provider' 
    });
}
```

**失败条件**：
- 提供商的 API Key 未配置（如 `ai_openai_api_key` 为 `null`）

**3.3.5 工具执行失败**

工具执行失败会通过 `streamText` 的错误处理机制返回：

```typescript
// api/src/ai/chat/utils/chat-request-tool-to-ai-sdk-tool.ts:35-53
return tool({
    description: directusTool.description,
    inputSchema,
    needsApproval,
    execute: async (rawArgs) => {
        const coercedArgs = coerceJsonFields(rawArgs as Record<string, unknown>);

        const { error, data: args } = directusTool.validateSchema?.safeParse(coercedArgs) ?? {
            data: coercedArgs,
        };

        if (error) {
            throw new InvalidPayloadError({ 
                reason: fromZodError(error).message 
            });
        }

        return directusTool.handler({ args, accountability, schema });
    },
});
```

**工具可能的失败场景**：

| 工具 | 失败原因 | 错误类型 |
|------|----------|----------|
| `items` | 集合不存在 | `ForbiddenError` |
| `items` | 系统集合操作 | `InvalidPayloadError` |
| `items` | 无权限操作字段 | 由 ItemsService 抛出 |
| `trigger-flow` | 流程不存在 | 由 FlowsService 抛出 |
| `trigger-flow` | 流程不是手动类型 | `ForbiddenError` |
| `trigger-flow` | 必填字段缺失 | `InvalidPayloadError` |

### 3.4 /ai/object 失败分支

`/ai/object` 端点用于生成结构化对象，失败分支与 `/ai/chat` 类似，但有一些差异：

**关键差异**：
- 不支持工具调用
- 使用 `streamObject` 而非 `streamText`
- 必须指定 `outputSchema`

**失败分支**：

```typescript
// api/src/ai/chat/controllers/object.post.ts:17-77
export const aiObjectPostHandler: RequestHandler = async (req, res) => {
    // 1. 权限检查（同 chat）
    if (!req.accountability?.app) {
        throw new ForbiddenError();
    }

    // 2. 请求体验证
    const parseResult = ObjectRequest.safeParse(req.body);
    // ObjectRequest 必须包含 outputSchema

    // 3. 模型权限检查（同 chat）

    // 4. 提供商配置检查
    const providerConfig = configs.find((c) => c.type === provider);
    if (!providerConfig) {
        throw new ServiceUnavailableError({ 
            service: provider, 
            reason: 'No API key configured for LLM provider' 
        });
    }

    // 5. 调用 LLM
    const result = streamObject({
        model: languageModel,
        prompt,
        schema: jsonSchema(addAdditionalPropertiesToJsonSchema(outputSchema)),
        // ...
    });

    result.pipeTextStreamToResponse(res);
};
```

### 3.5 完整失败处理流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    POST /ai/chat 完整失败处理流程                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  请求进入                                                                     │
│      │                                                                       │
│      ▼                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 检查 1: AI_ENABLED=true?                                              │   │
│  │ (api/src/app.ts:348)                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ├── NO ──▶ 404 Not Found (端点不存在)                                   │
│      │                                                                       │
│      ▼ YES                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 检查 2: accountability.app=true?                                      │   │
│  │ (api/src/ai/chat/controllers/chat.post.ts:12)                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ├── NO ──▶ ForbiddenError: "You don't have permission to access this." │
│      │                                                                       │
│      ▼ YES                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 检查 3: 请求体格式 (ChatRequest)                                       │   │
│  │ (api/src/ai/chat/models/chat-request.ts)                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ├── NO ──▶ InvalidPayloadError: Zod 验证错误消息                        │
│      │                                                                       │
│      ▼ YES                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 检查 4: 模型是否在允许列表中?                                           │   │
│  │ (api/src/ai/chat/controllers/chat.post.ts:34-40)                    │   │
│  │ (openai-compatible 跳过此检查)                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ├── NO ──▶ ForbiddenError: "Model not allowed for this provider"       │
│      │                                                                       │
│      ▼ YES                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 检查 5: 提供商 API Key 是否配置?                                        │   │
│  │ (api/src/ai/chat/lib/create-ui-stream.ts:46-48)                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ├── NO ──▶ ServiceUnavailableError: "No API key configured"            │
│      │                                                                       │
│      ▼ YES                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 调用 LLM (streamText)                                                 │   │
│  │ (api/src/ai/chat/lib/create-ui-stream.ts:71-98)                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│      │                                                                       │
│      ├── 网络错误 ──▶ SSE 事件: error 事件                                  │
│      │                                                                       │
│      ├── 工具调用失败 ──▶ 通过 tool result 返回错误信息                      │
│      │                                                                       │
│      ▼ 成功                                                                  │
│  流式返回响应                                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 权限边界与验证机制

### 4.1 多层权限架构

Directus 的 AI 功能采用多层权限验证：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         多层权限架构                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layer 1: 环境级别                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ AI_ENABLED=true/false                                                 │   │
│  │ - false: /ai 端点完全不可用                                            │   │
│  │ - true: 端点可用，但仍需其他权限检查                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 2: 请求级别                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ accountability.app = true                                             │   │
│  │ - 必须来自 Admin App 的请求                                            │   │
│  │ - API Token 或公共访问会被拒绝                                         │   │
│  │ - 检查位置: chat.post.ts:12, object.post.ts:18                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 3: 模型级别                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 模型白名单检查                                                          │   │
│  │ - ai_openai_allowed_models                                             │   │
│  │ - ai_anthropic_allowed_models                                          │   │
│  │ - ai_google_allowed_models                                             │   │
│  │ - openai-compatible 跳过此检查                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 4: 工具级别（Flow Operations）                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ permissions 配置                                                       │   │
│  │ - $trigger: 使用触发用户的权限                                         │   │
│  │ - $full: 系统权限（绕过权限检查）                                       │   │
│  │ - $public: 公共权限                                                    │   │
│  │ - <role-uuid>: 指定角色的权限                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                      │                                       │
│                                      ▼                                       │
│  Layer 5: 数据级别                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ItemsService 权限检查                                                  │   │
│  │ - 基于 accountability 的权限检查                                       │   │
│  │ - 字段级权限验证                                                        │   │
│  │ - 集合访问权限验证                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 Layer 4: 工具权限配置详解

**4.2.1 Item Update 操作的权限配置**

```typescript
// api/src/operations/item-update/index.ts:24-35
let customAccountability: Accountability | null;

if (!permissions || permissions === '$trigger') {
    // 使用触发用户的权限
    customAccountability = accountability;
} else if (permissions === '$full') {
    // 系统权限（绕过权限检查）
    customAccountability = await getAccountabilityForRole(
        'system', 
        { database, schema, accountability }
    );
} else if (permissions === '$public') {
    // 公共权限（无用户上下文）
    customAccountability = await getAccountabilityForRole(
        null, 
        { database, schema, accountability }
    );
} else {
    // 指定角色的权限
    customAccountability = await getAccountabilityForRole(
        permissions, 
        { database, schema, accountability }
    );
}
```

**4.2.2 权限配置对比表**

| 配置值 | accountability | 权限范围 | 使用场景 | 风险等级 |
|--------|----------------|----------|----------|----------|
| `$trigger` (默认) | 触发用户的 | 与触发用户相同 | 用户操作审计、保持权限一致性 | 🔵 低 |
| `$full` | `admin: true, app: true` | 绕过所有权限检查 | 系统自动化、定时任务 | 🔴 高 |
| `$public` | `user: null, role: null` | 公共访问权限 | 公开数据操作 | 🟡 中 |
| `<role-uuid>` | 指定角色的 | 该角色的权限 | 基于角色的自动化 | 🟡 中 |

**4.2.3 $full 权限的实现**

```typescript
// api/src/utils/get-accountability-for-role.ts:19-23
} else if (role === 'system') {
    generatedAccountability = createDefaultAccountability({
        admin: true,
        app: true,
    });
}
```

当 `admin: true` 时，大多数权限检查会被跳过：

```typescript
// 示例：权限检查模式
if (accountability?.admin !== true) {
    // 执行权限验证
    await validateAccess(...);
    await fetchPermissions(...);
}
// admin: true 时跳过
```

### 4.3 Layer 5: AI 工具的权限边界

**4.3.1 Items 工具的权限检查**

```typescript
// api/src/ai/tools/items/index.ts:77-91
async handler({ args, schema, accountability }) {
    // 检查 1: 不能操作系统集合
    if (isSystemCollection(args.collection)) {
        throw new InvalidPayloadError({ 
            reason: 'Cannot provide a core collection' 
        });
    }

    // 检查 2: 集合必须存在
    if (args.collection in schema.collections === false) {
        throw new ForbiddenError();
    }

    // 创建 ItemsService，传入 accountability
    const itemsService = new ItemsService(args.collection, {
        schema,
        accountability,  // 🔑 权限上下文会传递给 ItemsService
    });

    // ItemsService 内部会进行详细的权限检查
    // ...
}
```

**4.3.2 Trigger Flow 工具的权限检查**

```typescript
// api/src/ai/tools/trigger-flow/index.ts:21-59
async handler({ args, schema, accountability }) {
    const flowsService = new FlowsService({ schema, accountability });

    // 检查 1: 流程必须存在且是手动类型
    const flow = await flowsService.readOne(args.id, {
        filter: { 
            status: { _eq: 'active' }, 
            trigger: { _eq: 'manual' }  // 只能触发手动流程
        },
        fields: ['options'],
    });

    // 检查 2: 必填字段验证
    const requiredFields = ((flow.options?.['fields'] as { field: string; meta: { required: boolean } }[]) ?? [])
        .filter((field) => field.meta?.required)
        .map((field) => field.field);

    for (const fieldName of requiredFields) {
        if (!args.data || !(fieldName in args.data)) {
            throw new InvalidPayloadError({ 
                reason: `Required field "${fieldName}" is missing` 
            });
        }
    }

    // 触发流程（流程内部会使用自己的 permissions 配置）
    const flowManager = getFlowManager();
    const { result } = await flowManager.runWebhookFlow(
        `POST-${args.id}`,
        { /* data */ },
        { accountability, schema },  // 传递 accountability
    );

    return { type: 'text', data: result };
}
```

### 4.4 权限边界总结

| 组件 | 权限检查点 | 关键代码位置 |
|------|-----------|--------------|
| **API 端点** | `accountability.app` | `chat.post.ts:12`, `object.post.ts:18` |
| **模型选择** | 允许列表 | `chat.post.ts:34-40` |
| **Items 工具** | 系统集合检查、集合存在性 | `items/index.ts:78-84` |
| **Trigger Flow 工具** | 流程类型、必填字段 | `trigger-flow/index.ts:24-41` |
| **Item Update 操作** | `permissions` 配置 | `item-update/index.ts:27-35` |
| **ItemsService** | 字段级权限、集合权限 | `services/items.ts` (多处) |

---

## 5. 与通用 AI 聊天链路对比

### 5.1 架构对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     通用 AI 聊天链路                                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│   │  AI 侧边 │───▶│ use-ai.ts│───▶│ /ai/chat │───▶│  LLM     │          │
│   │  用户输入 │    │  Store   │    │ 流式对话 │    │  (API)   │          │
│   └──────────┘    └──────────┘    └──────────┘    └────┬─────┘          │
│                                                          │                  │
│                    工具调用方向 ◀─────────────────────────┘                  │
│                                                          │                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│   │  数据库  │◀───│ItemsService│◀──│  items   │◀───│  LLM 决定 │          │
│   │          │    │          │    │  工具    │    │  调用哪个 │          │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│                                                                              │
│   特点：                                                                      │
│   - 用户主动发起                                                              │
│   - LLM 决定调用哪些工具                                                      │
│   - 流式响应，交互式体验                                                       │
│   - 权限基于用户 accountability                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                     AI 字段自动化链路                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│   │ 数据变更 │───▶│  Event   │───▶│  Flow    │───▶│ Request  │          │
│   │create/   │    │ Trigger  │    │ Manager  │    │ Operation│          │
│   │update    │    │          │    │          │    │ (LLM API)│          │
│   └──────────┘    └──────────┘    └──────────┘    └────┬─────┘          │
│                                                          │                  │
│                                                          ▼                  │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│   │  数据库  │◀───│ItemsService│◀───│ItemUpdate│◀───│ LLM 响应 │          │
│   │          │    │          │    │Operation │    │          │          │
│   └──────────┘    └──────────┘    └──────────┘    └──────────┘          │
│                                                                              │
│   特点：                                                                      │
│   - 系统自动触发（事件驱动）                                                  │
│   - 预定义操作链（配置决定执行顺序）                                           │
│   - 可配置错误分支                                                            │
│   - 权限基于流程的 permissions 配置                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 详细对比表

| 维度 | 通用 AI 聊天 | AI 字段自动化 |
|------|-------------|--------------|
| **触发方式** | 用户在 AI 侧边栏输入指令 | 事件触发（数据变更）、定时触发、手动触发 |
| **主要 API** | `POST /ai/chat` | 无直接 `/ai` 调用，通过 `request` 操作调用外部 API |
| **替代 API** | 无 | 可使用 `POST /ai/object` 进行结构化生成 |
| **工具使用** | LLM 动态决定调用哪个工具 | 预定义操作序列（无工具调用概念） |
| **响应方式** | SSE 流式响应 | 同步或异步流程执行 |
| **上下文来源** | 对话历史、页面上下文 | 事件 payload、触发数据 |
| **权限模型** | 用户 accountability | 流程 `permissions` 配置 |
| **失败处理** | 工具调用错误返回、流式错误 | 流程 Reject 分支、Condition 判断 |
| **可观察性** | 对话历史记录 | 流程日志、活动记录 |
| **配置复杂度** | 低（开箱即用） | 中（需要配置流程和操作） |
| **灵活性** | 高（LLM 自主决策） | 中（预定义路径） |
| **可控性** | 中（依赖 LLM 行为） | 高（完全配置化） |

### 5.3 选择建议

**使用通用 AI 聊天** 的场景：
- 用户需要交互式探索数据
- 复杂的多步骤操作（如：先查询、再分析、再更新）
- 自然语言驱动的临时任务
- 需要保留对话历史的场景

**使用 AI 字段自动化** 的场景：
- 字段级数据处理（翻译、格式化、生成）
- 事件驱动的自动化（数据创建时自动处理）
- 需要严格错误处理和回滚的场景
- 需要权限隔离的自动化任务
- 定时批量处理任务

### 5.4 混合使用方案

在实际项目中，可以结合两种方式：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           混合使用方案                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   场景：产品数据管理                                                          │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  自动化链路（事件触发）                                                 │   │
│   │  ─────────────────────                                                 │   │
│   │  products.create → Flow: 自动翻译标题 → 写入 title_en                 │   │
│   │  products.update → Flow: 检测价格变更 → 更新促销标签                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  交互式链路（用户发起）                                                 │   │
│   │  ─────────────────────                                                 │   │
│   │  用户: "分析最近 7 天销量下降的产品，生成优化建议"                      │   │
│   │        ↓                                                                │   │
│   │  LLM 调用工具:                                                          │   │
│   │  1. items.read (查询销量数据)                                           │   │
│   │  2. 分析数据 (LLM 内部)                                                 │   │
│   │  3. items.update (写入建议到 description)                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键代码引用

### 6.1 API 端点

| 文件路径 | 说明 |
|----------|------|
| `api/src/app.ts:348-353` | AI 端点注册（依赖 `AI_ENABLED`） |
| `api/src/ai/chat/router.ts:7-9` | AI 路由定义 |
| `api/src/ai/chat/controllers/chat.post.ts` | `/ai/chat` 控制器 |
| `api/src/ai/chat/controllers/object.post.ts` | `/ai/object` 控制器 |
| `api/src/ai/chat/middleware/load-settings.ts` | AI 配置加载中间件 |

### 6.2 权限检查

| 文件路径 | 说明 |
|----------|------|
| `api/src/ai/chat/controllers/chat.post.ts:12-14` | `accountability.app` 检查 |
| `api/src/ai/chat/controllers/chat.post.ts:34-40` | 模型白名单检查 |
| `api/src/ai/chat/lib/create-ui-stream.ts:46-48` | 提供商配置检查 |
| `api/src/operations/item-update/index.ts:27-35` | 操作权限配置处理 |
| `api/src/utils/get-accountability-for-role.ts` | 角色 accountability 生成 |
| `api/src/ai/tools/items/index.ts:78-84` | Items 工具权限检查 |
| `api/src/ai/tools/trigger-flow/index.ts:24-41` | Trigger Flow 工具权限检查 |

### 6.3 流程与操作

| 文件路径 | 说明 |
|----------|------|
| `api/src/flows.ts` | Flow Manager 核心实现 |
| `api/src/operations/item-update/index.ts` | Item Update 操作 |
| `api/src/operations/request/index.ts` | Request 操作（调用外部 API） |
| `api/src/operations/condition/index.ts` | Condition 操作（条件判断） |
| `api/src/operations/log/index.ts` | Log 操作（错误记录） |

### 6.4 错误类型

| 错误类型 | 使用场景 | 代码位置 |
|----------|----------|----------|
| `ForbiddenError` | 权限不足、模型不允许 | 多处 |
| `InvalidPayloadError` | 请求体验证失败 | 多处 |
| `ServiceUnavailableError` | 提供商未配置 | `create-ui-stream.ts:46-48` |

---

## 附录：环境变量参考

```bash
# AI 功能开关
AI_ENABLED=true

# OpenAI 配置
AI_OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxx
AI_OPENAI_ALLOWED_MODELS=gpt-4o,gpt-4-turbo,gpt-3.5-turbo

# Anthropic 配置  
AI_ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxxxxx
AI_ANTHROPIC_ALLOWED_MODELS=claude-3-5-sonnet-20240620,claude-3-opus-20240229

# Google AI 配置
AI_GOOGLE_API_KEY=xxxxxxxxxxxxxxxx
AI_GOOGLE_ALLOWED_MODELS=gemini-1.5-pro,gemini-1.5-flash

# 自定义 OpenAI Compatible（如 Ollama）
AI_OPENAI_COMPATIBLE_API_KEY=ollama
AI_OPENAI_COMPATIBLE_BASE_URL=http://localhost:11434/v1
AI_OPENAI_COMPATIBLE_NAME=Ollama
AI_OPENAI_COMPATIBLE_MODELS=[{"id": "llama3.1", "name": "Llama 3.1", "context": 128000}]
AI_OPENAI_COMPATIBLE_HEADERS=[{"header": "X-Custom-Header", "value": "custom-value"}]

# 系统提示词
AI_SYSTEM_PROMPT=你是 Directus 助手，帮助用户管理数据。
```

---

## 总结

本文档深入解析了 Directus 的 AI 字段自动化链路，核心要点：

1. **两种触发模式**：
   - **通用 AI 聊天**：用户主动交互，LLM 自主决策工具调用
   - **AI 字段自动化**：事件驱动，预定义操作链

2. **五层权限架构**：
   - Layer 1: `AI_ENABLED` 环境级别
   - Layer 2: `accountability.app` 请求级别
   - Layer 3: 模型白名单
   - Layer 4: 操作 `permissions` 配置
   - Layer 5: `ItemsService` 数据级别

3. **权限配置选项**：
   - `$trigger`：使用触发用户权限（推荐）
   - `$full`：系统权限（风险高）
   - `$public`：公共权限
   - 角色 UUID：指定角色权限

4. **失败分支处理**：
   - 每层检查都有明确的错误类型
   - 流程支持 Reject 分支进行错误处理
   - 建议结合 Log 操作记录错误

5. **链路对比**：
   - 通用 AI 聊天：灵活但可控性低
   - AI 字段自动化：可控但配置复杂度高
   - 实际项目中建议混合使用

选择哪种链路取决于你的具体需求：需要灵活性和交互性选择通用 AI 聊天，需要自动化和可控性选择 AI 字段自动化。
