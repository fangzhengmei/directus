# Directus Webhook 与操作节点机制分析报告

## 1. 概述

Directus 通过 **Flow（流程）** 系统实现内部事件向外部系统的投递。Flow 系统由 **触发器（Trigger）** 和 **操作节点（Operation）** 组成，形成一个灵活的工作流引擎。

## 2. 触发器（Trigger）机制

### 2.1 触发器类型

Directus 支持以下五种触发器类型 (`packages/types/src/flows.ts:1`)：

| 触发器类型 | 描述 | 触发方式 |
|-----------|------|---------|
| `event` | 事件触发器 | 监听内部系统事件（如数据增删改） |
| `schedule` | 定时触发器 | 按预定时间/间隔触发 |
| `operation` | 操作触发器 | 由其他 Flow 的操作节点触发 |
| `webhook` | Webhook 触发器 | 通过 HTTP 请求端点触发 |
| `manual` | 手动触发器 | 在管理界面手动触发 |

### 2.2 Webhook 触发器配置

Webhook 触发器通过专门的 HTTP 端点触发 Flow 执行。

#### 触发端点
- **GET**: `GET /flows/trigger/:flowId`
- **POST**: `POST /flows/trigger/:flowId`

#### 配置选项 (`options`)

| 配置项 | 类型 | 描述 |
|-------|------|------|
| `method` | string | 允许的 HTTP 方法：`GET` 或 `POST` |
| `cacheEnabled` | boolean | 是否启用 GET 请求缓存（默认 `true`） |

#### 缓存行为

- **GET 请求**：默认启用缓存，相同请求在缓存有效期内返回相同结果
- **POST 请求**：始终忽略缓存，每次请求都会执行 Flow

#### 配置示例

```typescript
// 创建 Webhook 触发的 Flow
const flow = {
  name: '外部系统通知 Flow',
  trigger: 'webhook',
  options: {
    method: 'POST',
    cacheEnabled: false  // POST 请求始终不缓存
  },
  status: 'active',
  accountability: '$trigger'  // 使用触发者的权限
};
```

### 2.3 事件触发器（Event Trigger）

事件触发器用于监听 Directus 内部系统事件，实现数据变更后的自动处理。

#### 可监听的事件类型

| 事件类别 | 事件示例 | 描述 |
|---------|---------|------|
| 数据操作 | `items.create`, `items.update`, `items.delete` | 数据增删改事件 |
| 用户操作 | `auth.login`, `users.create` | 认证和用户管理事件 |
| 文件操作 | `files.upload`, `files.delete` | 文件管理事件 |
| 自定义事件 | 自定义事件名 | 通过扩展注册的自定义事件 |

#### 配置选项

```typescript
// 监听数据创建事件的 Flow
const flow = {
  name: '数据创建通知',
  trigger: 'event',
  options: {
    type: 'action',           // 操作类型：action（操作后）或 filter（操作前）
    scope: ['items.create'],  // 监听的事件范围
    collections: ['articles'] // 限定集合（可选）
  },
  status: 'active'
};
```

## 3. 操作节点（Operation）机制

### 3.1 操作节点类型定义

操作节点是 Flow 中的执行单元，每个节点定义一个具体的操作逻辑。

```typescript
// packages/types/src/flows.ts:17-27
interface Operation {
  id: string;                    // 唯一标识
  name: string | null;           // 节点名称
  key: string;                    // 节点键（Flow 内唯一）
  type: string;                   // 操作类型
  position_x: number;            // 工作区 X 坐标
  position_y: number;            // 工作区 Y 坐标
  options: Record<string, any>;  // 操作配置选项
  resolve: Operation | null;     // 成功时执行的下一节点
  reject: Operation | null;      // 失败时执行的下一节点
}
```

### 3.2 内置操作节点类型

Directus 提供以下内置操作类型 (`packages/specs/src/components/operation.yaml:17`)：

| 操作类型 | 描述 | 主要用途 |
|---------|------|---------|
| `log` | 日志输出 | 记录执行日志到控制台或日志系统 |
| `mail` | 发送邮件 | 发送电子邮件通知 |
| `notification` | 系统通知 | 发送 Directus 内部系统通知 |
| `create` | 创建数据 | 在数据库中创建新记录 |
| `read` | 读取数据 | 从数据库读取数据 |
| `request` | HTTP 请求 | **向外部系统发送 HTTP 请求（核心投递操作）** |
| `sleep` | 延时等待 | 暂停执行指定时间 |
| `transform` | 数据转换 | 转换/处理数据流 |
| `trigger` | 触发 Flow | 触发另一个 Flow 执行 |
| `condition` | 条件判断 | 基于条件选择执行分支 |

### 3.3 Request 操作（外部系统投递核心）

`request` 操作是 Directus 向外部系统投递事件的核心机制，用于发送 HTTP 请求到外部 API。

#### 配置选项

| 配置项 | 类型 | 描述 |
|-------|------|------|
| `url` | string | 目标 URL（支持模板变量） |
| `method` | string | HTTP 方法：GET, POST, PUT, PATCH, DELETE |
| `headers` | object | 请求头 |
| `body` | any | 请求体 |

#### 配置示例

```typescript
// 向外部系统发送数据变更通知
const requestOperation = {
  key: 'notify_external',
  type: 'request',
  name: '通知外部系统',
  options: {
    url: 'https://api.external-service.com/webhook',
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Webhook-Secret': '{{ $env.WEBHOOK_SECRET }}'
    },
    body: {
      event: '{{ $trigger.event }}',
      collection: '{{ $trigger.collection }}',
      data: '{{ $trigger.payload }}',
      timestamp: '{{ $timestamp }}'
    }
  },
  resolve: null,  // 成功后无后续操作
  reject: null    // 失败后无后续操作
};
```

## 4. 执行链路与数据流

### 4.1 Flow 执行链路结构

Flow 中的操作节点通过 `resolve` 和 `reject` 字段形成有向无环图（DAG）执行链路：

```
                    ┌─────────────┐
                    │   Trigger   │
                    │  (Webhook)  │
                    └──────┬──────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Operation1 │
                    │ (Transform) │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       ┌─────────────┐           ┌─────────────┐
       │  resolve    │           │   reject    │
       │  Operation2 │           │  Operation4 │
       │  (Request)  │           │  (Log Error)│
       └──────┬──────┘           └─────────────┘
              │
    ┌─────────┴─────────┐
    │                   │
    ▼                   ▼
┌───────────┐     ┌───────────┐
│ resolve   │     │  reject   │
│Operation3 │     │Operation5 │
│(Log Success)│   │(Retry Logic)│
└───────────┘     └───────────┘
```

### 4.2 Condition 操作（分支控制）

`condition` 操作用于实现条件分支逻辑，是实现复杂控制流的关键。

```typescript
// 条件判断操作示例
const conditionOperation = {
  key: 'check_status',
  type: 'condition',
  name: '检查响应状态',
  options: {
    filter: {
      // 使用 Directus 过滤器语法
      _and: [
        { 'notify_external.status': { _eq: 200 } },
        { 'notify_external.data.success': { _eq: true } }
      ]
    }
  },
  resolve: 'log_success',  // 条件满足时执行
  reject: 'handle_error'    // 条件不满足时执行
};
```

### 4.3 数据流与上下文变量

Flow 执行过程中，数据通过上下文变量在操作节点间传递：

| 变量 | 描述 | 示例 |
|-----|------|------|
| `$trigger` | 触发器数据 | Webhook 请求体、事件数据 |
| `$env` | 环境变量 | `$env.WEBHOOK_SECRET` |
| `$accountability` | 用户权限信息 | 当前用户角色、权限 |
| `$timestamp` | 当前时间戳 | ISO 格式时间 |
| `$last` | 上一个操作的结果 | 前一个节点的输出 |
| `$flow` | 当前 Flow 信息 | Flow ID、名称等 |
| `{operation_key}` | 指定操作的结果 | `notify_external.status` |

## 5. 失败处理与重试机制

### 5.1 错误处理机制

Directus Flow 通过 `reject` 链路实现错误处理：

1. **操作失败**：当操作节点执行失败时（如 HTTP 请求超时、返回错误状态码）
2. **执行 reject 分支**：自动跳转到 `reject` 指向的操作节点
3. **错误信息传递**：错误详情通过 `$last` 变量传递给后续节点

### 5.2 自定义重试逻辑

Directus 没有内置的自动重试机制，但可以通过 `condition` + `sleep` 操作组合实现自定义重试：

```
┌─────────────┐
│  Request    │◄──────────────────┐
│  Operation  │                   │
└──────┬──────┘                   │
       │                          │
   ┌───┴───┐                      │
   │Success│                      │
   │Failure│                      │
   └───┬───┘                      │
       │                          │
       ▼                          │
┌─────────────┐                   │
│  Condition  │                   │
│ Check Retry │                   │
│   Count     │                   │
└──────┬──────┘                   │
       │                          │
  ┌────┴────┐                     │
  │  Under  │                     │
  │  Limit  │                     │
  └────┬────┘                     │
       │                          │
   ┌───┴───┐                      │
   │  Yes  │    ┌──────────┐     │
   │       │───▶│  Sleep   │─────┘
   └───┬───┘    │ (Delay)  │
       │        └──────────┘
       ▼
┌─────────────┐
│    No       │
│  Log Error  │
│  & Notify   │
└─────────────┘
```

### 5.3 重试实现示例

使用变量追踪重试次数：

```typescript
// 1. 初始化重试计数器（Transform 操作）
const initRetry = {
  key: 'init_retry',
  type: 'transform',
  options: {
    // 设置初始重试次数
    data: { retryCount: 0, maxRetries: 3 }
  },
  resolve: 'send_request'
};

// 2. 发送请求（Request 操作）
const sendRequest = {
  key: 'send_request',
  type: 'request',
  options: {
    url: 'https://api.external.com/webhook',
    method: 'POST',
    body: '{{ $trigger.payload }}'
  },
  resolve: 'check_success',   // 成功后检查
  reject: 'check_retry'       // 失败后检查重试
};

// 3. 检查重试条件（Condition 操作）
const checkRetry = {
  key: 'check_retry',
  type: 'condition',
  options: {
    filter: {
      _and: [
        { 'init_retry.retryCount': { _lt: '{{ init_retry.maxRetries }}' } }
      ]
    }
  },
  resolve: 'increment_retry',  // 可以重试
  reject: 'log_final_error'     // 超过重试次数
};

// 4. 增加重试计数（Transform 操作）
const incrementRetry = {
  key: 'increment_retry',
  type: 'transform',
  options: {
    data: {
      retryCount: '{{ init_retry.retryCount + 1 }}'
    }
  },
  resolve: 'wait_before_retry'
};

// 5. 等待延时（Sleep 操作）
const waitBeforeRetry = {
  key: 'wait_before_retry',
  type: 'sleep',
  options: {
    // 指数退避：1s, 2s, 4s...
    duration: '{{ 1000 * (2 ^ increment_retry.retryCount) }}'
  },
  resolve: 'send_request'  // 回到请求操作
};
```

## 6. 状态管理与持久化

### 6.1 数据模型

Flow 和 Operation 的配置存储在数据库中：

#### directus_flows 表

| 字段 | 类型 | 描述 |
|-----|------|------|
| `id` | UUID | 唯一标识 |
| `name` | string | Flow 名称 |
| `status` | enum | 状态：`active` / `inactive` |
| `trigger` | enum | 触发器类型 |
| `options` | JSON | 触发器配置 |
| `operation` | UUID | 起始操作节点 ID |
| `accountability` | string | 执行权限模式 |

#### directus_operations 表

| 字段 | 类型 | 描述 |
|-----|------|------|
| `id` | UUID | 唯一标识 |
| `name` | string | 节点名称 |
| `key` | string | 节点键 |
| `type` | string | 操作类型 |
| `options` | JSON | 操作配置 |
| `resolve` | UUID | 成功时下一节点 |
| `reject` | UUID | 失败时下一节点 |
| `flow` | UUID | 所属 Flow ID |

### 6.2 Accountability（权限模式）

Flow 执行时的权限通过 `accountability` 字段配置：

| 模式 | 描述 |
|-----|------|
| `$public` | 使用公开权限（最低权限） |
| `$trigger` | 使用触发者的权限（如 Webhook 触发时的用户） |
| `$full` | 使用完全权限（跳过权限检查） |
| `<role_uuid>` | 使用指定角色的权限 |

### 6.3 执行状态追踪

Directus Flow 本身不存储执行历史，但可以通过以下方式实现状态追踪：

1. **自定义日志操作**：在关键节点添加 `log` 或自定义操作记录状态
2. **数据库记录**：使用 `create` 操作将执行状态写入自定义表
3. **外部通知**：通过 `request` 操作通知外部系统执行状态

## 7. 完整示例：外部系统 Webhook 通知 Flow

### 7.1 场景描述

当 `articles` 表有新数据创建时，自动通知外部系统，并处理可能的失败情况。

### 7.2 Flow 配置

```
Trigger: event (items.create on articles)
         │
         ▼
┌─────────────────┐
│ Transform:      │
│ Prepare Payload │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Request:        │
│ Notify External │◄──────────────────┐
│ System          │                   │
└────────┬────────┘                   │
         │                            │
    ┌────┴────┐                       │
    │Success  │                       │
    │Failure  │                       │
    └────┬────┘                       │
         │                            │
         ▼                            │
┌─────────────────┐                   │
│ Condition:      │                   │
│ Check Retry     │                   │
│    Count        │                   │
└────────┬────────┘                   │
         │                            │
    ┌────┴────┐                       │
    │  Under  │                       │
    │  Limit  │                       │
    └────┬────┘                       │
         │                            │
    ┌────┴────┐                       │
    │   Yes   │    ┌─────────────┐   │
    │         │───▶│ Sleep:      │───┘
    └────┬────┘    │ Wait (1s)  │
         │         └─────────────┘
         ▼
┌─────────────────┐
│    No / Final   │
│ Request: Notify │
│ Admin of Failure│
└─────────────────┘
```

### 7.3 配置代码示例

```typescript
// 1. 创建 Flow
const flow = {
  name: '文章创建 - 外部通知',
  trigger: 'event',
  options: {
    type: 'action',
    scope: ['items.create'],
    collections: ['articles']
  },
  status: 'active',
  accountability: '$trigger'
};

// 2. 准备数据操作
const preparePayload = {
  key: 'prepare_payload',
  type: 'transform',
  name: '准备通知数据',
  options: {
    data: {
      event: 'article.created',
      articleId: '{{ $trigger.key }}',
      title: '{{ $trigger.payload.title }}',
      author: '{{ $trigger.payload.author }}',
      createdAt: '{{ $timestamp }}'
    }
  }
};

// 3. 发送 Webhook 请求
const sendWebhook = {
  key: 'send_webhook',
  type: 'request',
  name: '发送 Webhook',
  options: {
    url: 'https://api.external-service.com/directus-webhook',
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Directus-Signature': 'sha256={{ $env.WEBHOOK_SECRET }}'
    },
    body: '{{ prepare_payload }}'
  }
};

// 4. 检查重试条件
const checkRetry = {
  key: 'check_retry',
  type: 'condition',
  name: '检查重试次数',
  options: {
    filter: {
      'prepare_payload.retryCount': { _lt: 3 }
    }
  }
};

// 5. 增加重试计数
const incrementRetry = {
  key: 'increment_retry',
  type: 'transform',
  name: '增加重试计数',
  options: {
    data: {
      ...'{{ prepare_payload }}',
      retryCount: '{{ (prepare_payload.retryCount || 0) + 1 }}'
    }
  }
};

// 6. 等待延时
const waitRetry = {
  key: 'wait_retry',
  type: 'sleep',
  name: '等待重试',
  options: {
    duration: 1000  // 1秒
  }
};

// 7. 通知管理员失败
const notifyAdmin = {
  key: 'notify_admin',
  type: 'mail',
  name: '邮件通知管理员',
  options: {
    to: ['admin@example.com'],
    subject: 'Webhook 投递失败 - 文章: {{ prepare_payload.title }}',
    body: 'Webhook 通知外部系统失败，已尝试 3 次。\n\n文章 ID: {{ prepare_payload.articleId }}\n错误: {{ send_webhook.error }}'
  }
};
```

## 8. 总结

### 8.1 核心机制总结

| 组件 | 作用 | 关键特性 |
|-----|------|---------|
| **Webhook Trigger** | 外部 HTTP 请求触发 | 支持 GET/POST，可配置缓存 |
| **Event Trigger** | 内部事件触发 | 监听 items.create/update/delete 等 |
| **Request Operation** | 外部系统投递核心 | 灵活的 HTTP 请求配置 |
| **resolve/reject** | 执行链路控制 | 有向无环图执行流 |
| **Condition Operation** | 条件分支 | 实现复杂逻辑控制 |
| **Sleep Operation** | 延时等待 | 实现重试延时 |

### 8.2 与传统 Webhook 的区别

Directus 的 Flow 系统相比传统 Webhook 具有以下优势：

1. **灵活的触发条件**：支持多种触发方式（HTTP、事件、定时等）
2. **可编程的处理流程**：通过操作节点组合实现复杂逻辑
3. **内置错误处理**：通过 resolve/reject 机制处理成功/失败分支
4. **数据转换能力**：支持在投递前对数据进行转换处理
5. **权限控制**：灵活的 accountability 配置

### 8.3 注意事项

1. **无内置重试**：需要通过 Condition + Sleep 手动实现重试逻辑
2. **无执行历史**：Flow 执行状态不自动存储，需要自行实现日志记录
3. **异步执行**：Flow 通常是异步执行的，不要依赖同步返回
4. **权限配置**：正确配置 accountability 以确保操作有足够权限

---

*报告生成时间：2026-05-02*
*基于 Directus 代码库分析*
