# Directus Webhook 与操作节点投递链路分析报告（纠偏版）

## 1. 概述

本报告基于 Directus 代码库的实际实现，重新梳理并纠偏内部事件经操作节点投递到外部系统的完整链路。报告重点关注：
- 触发类型命名与权限模式的真实取值
- Operation 触发型流程如何承接上游上下文
- Webhook GET/POST 在缓存和执行上的差异
- 失败排查和重试边界说明

每个结论都提供**实现侧**（源代码定义）和**测试侧**（测试代码验证）的依据。

---

## 2. 触发类型命名与权限模式的真实取值

### 2.1 TriggerType（触发器类型）

#### 实现侧依据

**类型定义** (`packages/types/src/flows.ts:1`)：

```typescript
export type TriggerType = 'event' | 'schedule' | 'operation' | 'webhook' | 'manual';
```

**Flow 接口定义** (`packages/types/src/flows.ts:5-15`)：

```typescript
export interface Flow {
    id: string;
    name: string | null;
    // ...
    trigger: TriggerType | null;  // 触发器类型
    options: Record<string, any>;
    operation: Operation | null;
    accountability: 'all' | 'activity' | null;  // 注意：这里类型定义有问题！
}
```

#### 文档侧差异（Specs 文档）

**OpenAPI 组件定义** (`packages/specs/src/components/flow.yaml:31`)：

```yaml
trigger:
    description: Type of trigger for the flow. One of `hook`, `webhook`, `operation`, `schedule`, `manual`.
    type: string
    example: manual
```

#### 纠偏结论

| 来源 | 取值 | 说明 |
|-----|------|------|
| **类型定义（实现侧）** | `'event'` | 正确的触发器类型名 |
| **Specs 文档** | `'hook'` | 可能是文档错误，应以类型定义为准 |

**真实的 TriggerType 取值**：

| 类型值 | 描述 | 触发方式 |
|-------|------|---------|
| `'event'` | 事件触发器 | 监听内部系统事件（如数据增删改） |
| `'schedule'` | 定时触发器 | 按预定时间/间隔触发 |
| `'operation'` | 操作触发器 | 由其他 Flow 的操作节点触发 |
| `'webhook'` | Webhook 触发器 | 通过 HTTP 请求端点触发 |
| `'manual'` | 手动触发器 | 在管理界面手动触发 |

### 2.2 Accountability（权限模式）

#### 重要发现：两种不同的 Accountability 概念

Directus 代码中存在两种不同的 `accountability` 概念，容易混淆：

##### 1. Collection.accountability（集合审计范围）

**类型定义** (`packages/types/src/collection.ts:28`)：

```typescript
export type CollectionMeta = {
    // ...
    accountability: 'all' | 'activity' | null;
    // ...
};
```

**用途**：控制集合的**审计日志记录范围**

| 取值 | 描述 |
|-----|------|
| `'all'` | 记录所有操作（默认值） |
| `'activity'` | 只记录有用户活动的操作 |
| `null` | 不记录审计日志 |

**测试侧依据** (`packages/composables/src/use-collection.test.ts:15,50`)：

```typescript
// 测试用例 1
const mockCollectionInfo = {
    // ...
    accountability: 'all',
    // ...
};

// 测试用例 2
const mockCollectionInfo2 = {
    // ...
    accountability: 'activity',
    // ...
};
```

##### 2. Flow.accountability（流程权限模式）

**类型定义问题** (`packages/types/src/flows.ts:14`)：

```typescript
// ⚠️ 类型定义错误！使用了 Collection 的 accountability 类型
export interface Flow {
    // ...
    accountability: 'all' | 'activity' | null;  // ❌ 错误！
    // ...
}
```

**正确的文档定义** (`packages/specs/src/components/flow.yaml:40`)：

```yaml
accountability:
    description: The permission used during the flow. One of `$public`, `$trigger`, `$full`, or UUID of a role.
    type: string
    example: '$trigger'
```

#### 纠偏结论

| 概念 | 所属对象 | 真实取值 | 用途 |
|-----|---------|---------|------|
| **Collection.accountability** | 集合 | `'all'` \| `'activity'` \| `null` | 控制审计日志记录范围 |
| **Flow.accountability** | 流程 | `'$public'` \| `'$trigger'` \| `'$full'` \| `<role_uuid>` | 控制 Flow 执行时的权限上下文 |

**Flow.accountability 的真实取值**：

| 取值 | 描述 | 权限级别 |
|-----|------|---------|
| `'$public'` | 使用公开权限 | 最低权限（匿名用户权限） |
| `'$trigger'` | 使用触发者的权限 | 继承触发 Flow 的用户权限 |
| `'$full'` | 使用完全权限 | 跳过权限检查（最高权限） |
| `<role_uuid>` | 使用指定角色的权限 | 自定义角色权限 |

**测试侧依据** (`tests/blackbox/tests/db/routes/flows/webhook.test.ts:63`)：

```typescript
const payloadFlowCreate = {
    name: 'webhook flow',
    // ...
    accountability: null,  // 测试中使用 null
    trigger: 'webhook',
    options: {},
};
```

---

## 3. Operation 触发型流程如何承接上游上下文

### 3.1 OperationContext（操作上下文）定义

#### 实现侧依据

**OperationContext 接口** (`packages/types/src/extensions/operations.ts:8-12`)：

```typescript
export type OperationContext = ApiExtensionContext & {
    data: Record<string, unknown>;           // 上下文数据
    accountability: Accountability | null;    // 权限信息
    flow?: Flow;                               // 当前 Flow 信息
};
```

**Accountability 类型** (`packages/types/src/accountability.ts:6-17`)：

```typescript
export type Accountability = {
    role: string | null;        // 当前角色
    roles: string[];            // 所有角色
    user: string | null;        // 用户 ID
    admin: boolean;             // 是否管理员
    app: boolean;               // 是否应用上下文
    share?: string;             // 分享链接 ID
    ip: string | null;          // IP 地址
    userAgent?: string;         // 用户代理
    origin?: string;            // 来源
    session?: string;           // 会话 ID
};
```

**OperationHandler 类型** (`packages/types/src/extensions/operations.ts:14-17`)：

```typescript
export type OperationHandler<Options = Record<string, unknown>> = (
    options: Options,           // 操作配置选项
    context: OperationContext,  // 操作上下文
) => unknown | Promise<unknown> | void;
```

### 3.2 上下文传递链路

#### Operation 触发器的工作原理

当一个 Flow 的操作节点（`type: 'trigger'`）触发另一个 Flow 时，上下文传递链路如下：

```
┌─────────────────────────────────────────────────────────────────┐
│                        上游 Flow A                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────┐    ┌──────────┐    ┌──────────────────────────┐ │
│  │ Trigger  │───▶│ Operation│───▶│ Trigger Operation        │ │
│  │ (任意)   │    │ (处理)   │    │ (type: 'trigger')       │ │
│  └──────────┘    └──────────┘    └──────────────────────────┘ │
│                                              │                    │
│                                              ▼                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              传递的上下文数据                                │ │
│  │  - data: 上游操作的输出结果                                 │ │
│  │  - accountability: 上游的权限上下文                        │ │
│  │  - flow: 上游 Flow 信息（可选）                           │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        下游 Flow B                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Operation Trigger                            │  │
│  │  (trigger: 'operation')                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           承接的上下文数据                                │  │
│  │  - $trigger: 上游传递的 data                             │  │
│  │  - $accountability: 上游传递的 accountability            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    下游 Operation 节点                    │  │
│  │  可通过模板变量访问上游数据：                              │  │
│  │  - '{{ $trigger.field }}'                                │  │
│  │  - '{{ $accountability.user }}'                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 上下文数据的访问方式

#### 模板变量

在操作节点的 `options` 配置中，可以使用模板变量访问上下文数据：

| 模板变量 | 描述 | 示例 |
|---------|------|------|
| `{{ $trigger }}` | 触发器数据 | `{{ $trigger.payload }}` |
| `{{ $accountability }}` | 权限上下文 | `{{ $accountability.user }}` |
| `{{ $env }}` | 环境变量 | `{{ $env.API_KEY }}` |
| `{{ $timestamp }}` | 当前时间戳 | `{{ $timestamp }}` |
| `{{ $last }}` | 上一个操作的结果 | `{{ $last.status }}` |
| `{{ <operation_key> }}` | 指定操作的结果 | `{{ my_operation.data }}` |

#### 实现侧依据

**OperationContext 中的 data 字段** (`packages/types/src/extensions/operations.ts:9`)：

```typescript
export type OperationContext = ApiExtensionContext & {
    data: Record<string, unknown>;  // 上游传递的上下文数据
    accountability: Accountability | null;
    flow?: Flow;
};
```

### 3.4 配置示例

#### 上游 Flow（触发者）

```typescript
// Flow A: 数据处理后触发另一个 Flow
const flowA = {
    name: '数据处理 Flow',
    trigger: 'event',
    options: {
        type: 'action',
        scope: ['items.create'],
        collections: ['articles']
    },
    accountability: '$trigger',
    operation: {
        key: 'process_data',
        type: 'transform',
        options: {
            data: {
                articleId: '{{ $trigger.key }}',
                title: '{{ $trigger.payload.title }}',
                processedAt: '{{ $timestamp }}'
            }
        },
        resolve: {
            key: 'trigger_another_flow',
            type: 'trigger',  // 触发操作
            options: {
                flow: 'flow-b-uuid',  // 下游 Flow ID
                // 上游的数据会自动传递到下游
            }
        }
    }
};
```

#### 下游 Flow（被触发者）

```typescript
// Flow B: 被上游触发，承接上游上下文
const flowB = {
    name: '外部通知 Flow',
    trigger: 'operation',  // 操作触发器
    options: {},
    accountability: '$trigger',  // 继承上游的 accountability
    operation: {
        key: 'send_notification',
        type: 'request',
        options: {
            url: 'https://api.external.com/webhook',
            method: 'POST',
            body: {
                // 访问上游传递的数据
                article: '{{ $trigger }}',  // 上游 process_data 的输出
                triggeredBy: '{{ $accountability.user }}',  // 上游的用户
                timestamp: '{{ $timestamp }}'
            }
        }
    }
};
```

---

## 4. Webhook GET/POST 在缓存和执行上的差异

### 4.1 测试侧依据（Blackbox 测试）

#### 测试代码分析

**测试文件** (`tests/blackbox/tests/db/routes/flows/webhook.test.ts`)：

```typescript
describe('Webhook Trigger', () => {
    describe('cacheEnabled works for GET', () => {
        it.each(vendors)('%s', async (vendor) => {
            // Setup
            const payloadFlowCreate = {
                name: 'webhook flow',
                trigger: 'webhook',
                options: {},  // 默认配置
                accountability: null,
            };

            // 不同的缓存配置
            const flowCacheEnabledId = (/* ... */)
                .send({
                    options: { ...payloadFlowCreate.options, cacheEnabled: true },
                });

            const flowCacheDisabledId = (/* ... */)
                .send({
                    options: { ...payloadFlowCreate.options, cacheEnabled: false },
                });

            // Action: 两次 GET 请求
            const responseDefault = await request(getUrl(vendor, env)).get(`/flows/trigger/${flowId}`);
            await sleep(100);
            const responseDefault2 = await request(getUrl(vendor, env)).get(`/flows/trigger/${flowId}`);

            // Assert
            // 默认配置：两次请求结果相同（启用缓存）
            expect(responseDefault.body).toEqual(responseDefault2.body);
            
            // cacheEnabled: true：两次请求结果相同（启用缓存）
            expect(responseCacheEnabled.body).toEqual(responseCacheEnabled2.body);
            
            // cacheEnabled: false：两次请求结果不同（禁用缓存）
            expect(responseCacheDisabled.body).not.toEqual(responseCacheDisabled2.body);
        });
    });

    describe('ignores cacheEnabled for POST', () => {
        it.each(vendors)('%s', async (vendor) => {
            // Setup
            const payloadFlowCreate = {
                name: 'POST webhook flow',
                trigger: 'webhook',
                options: { method: 'POST' },  // 指定 POST 方法
                accountability: null,
            };

            // Action: 两次 POST 请求
            const responseDefault = await request(getUrl(vendor, env)).post(`/flows/trigger/${flowId}`);
            await sleep(100);
            const responseDefault2 = await request(getUrl(vendor, env)).post(`/flows/trigger/${flowId}`);

            // Assert
            // 无论 cacheEnabled 设置如何，POST 始终不缓存
            expect(responseDefault.body).not.toEqual(responseDefault2.body);
        });
    });
});
```

### 4.2 测试侧依据（E2E 测试）

**测试文件** (`tests/e2e/tests/flows/webhook.test.ts`)：

```typescript
test('trigger webhook', async () => {
    const flow = await api.request(
        createFlow({
            name: 'webhook flow default',
            trigger: 'webhook',
            options: {},  // 默认配置
        }),
    );

    const operation = await api.request(
        createOperation({
            flow: flow!.id,
            type: 'exec',
            options: { code: 'module.exports = async function() { return { epoch: Date.now() }; }' },
        }),
    );

    // 两次 GET 请求
    const result1 = (await api.request(triggerFlow('GET', flow.id))) as { epoch: number };
    const result2 = (await api.request(triggerFlow('GET', flow.id))) as { epoch: number };

    // 断言
    if (options.cache) {
        // 启用缓存时，两次结果相同
        expect(result2.epoch).toBe(result1.epoch);
    } else {
        // 禁用缓存时，两次结果不同
        expect(result2.epoch).toBeGreaterThan(result1.epoch);
    }
});

test('trigger webhook with cacheEnabled set to false', async () => {
    const flow = await api.request(
        createFlow({
            name: 'webhook flow cache enabled',
            trigger: 'webhook',
            options: { cacheEnabled: false },  // 显式禁用缓存
        }),
    );

    // 两次 GET 请求
    const result1 = (await api.request(triggerFlow('GET', flow.id))) as { epoch: number };
    const result2 = (await api.request(triggerFlow('GET', flow.id))) as { epoch: number };

    // 断言：始终不缓存
    expect(result2.epoch).toBeGreaterThan(result1.epoch);
});
```

### 4.3 纠偏结论

#### GET 请求缓存行为

| 配置 | 缓存行为 | 测试验证 |
|-----|---------|---------|
| `options: {}`（默认） | **启用缓存** | 两次请求返回相同结果 |
| `options: { cacheEnabled: true }` | **启用缓存** | 两次请求返回相同结果 |
| `options: { cacheEnabled: false }` | **禁用缓存** | 两次请求返回不同结果 |

#### POST 请求缓存行为

| 配置 | 缓存行为 | 测试验证 |
|-----|---------|---------|
| 任意 `cacheEnabled` 设置 | **始终不缓存** | 两次请求返回不同结果 |

#### 完整对比表

| HTTP 方法 | cacheEnabled | 是否缓存 | 执行次数 | 说明 |
|----------|-------------|---------|---------|------|
| GET | 未设置（默认） | ✅ 是 | 1 次 | 后续请求使用缓存 |
| GET | `true` | ✅ 是 | 1 次 | 显式启用缓存 |
| GET | `false` | ❌ 否 | 每次都执行 | 显式禁用缓存 |
| POST | 任意值 | ❌ 否 | 每次都执行 | POST 始终不缓存 |

### 4.4 配置示例

```typescript
// 1. GET - 默认启用缓存
const flow1 = {
    name: '缓存 GET Webhook',
    trigger: 'webhook',
    options: {},  // 默认启用缓存
};

// 2. GET - 显式启用缓存
const flow2 = {
    name: '显式缓存 GET Webhook',
    trigger: 'webhook',
    options: { cacheEnabled: true },
};

// 3. GET - 禁用缓存
const flow3 = {
    name: '不缓存 GET Webhook',
    trigger: 'webhook',
    options: { cacheEnabled: false },
};

// 4. POST - 始终不缓存
const flow4 = {
    name: 'POST Webhook',
    trigger: 'webhook',
    options: { 
        method: 'POST',
        cacheEnabled: true  // 此设置对 POST 无效
    },
};
```

---

## 5. 失败排查和重试边界说明

### 5.1 执行链路与错误处理

#### 操作节点的成功/失败分支

**实现侧依据** (`packages/types/src/flows.ts:17-27`)：

```typescript
export interface Operation {
    id: string;
    name: string | null;
    key: string;
    type: string;
    position_x: number;
    position_y: number;
    options: Record<string, any>;
    resolve: Operation | null;  // ✅ 成功时执行的下一节点
    reject: Operation | null;   // ❌ 失败时执行的下一节点
}
```

#### 执行链路示意图

```
┌─────────────────────────────────────────────────────────────────┐
│                         Flow 执行链路                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌──────────┐                                                 │
│   │  Trigger │                                                 │
│   │ (入口点)  │                                                 │
│   └────┬─────┘                                                 │
│        │                                                       │
│        ▼                                                       │
│   ┌──────────────┐                                             │
│   │  Operation 1 │                                             │
│   │   (Request)  │                                             │
│   └──────┬───────┘                                             │
│          │                                                     │
│    ┌─────┴─────┐                                               │
│    │           │                                               │
│    ▼           ▼                                               │
│ ┌────────┐  ┌────────┐                                         │
│ │resolve │  │ reject │                                         │
│ │ (成功)  │  │ (失败)  │                                         │
│ └────┬───┘  └────┬───┘                                         │
│      │           │                                               │
│      ▼           ▼                                               │
│ ┌──────────┐ ┌──────────┐                                       │
│ │Operation2│ │Operation4│                                       │
│ │(Log Succ)│ │(Log Fail)│                                       │
│ └──────────┘ └──────────┘                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 失败判定条件

#### 操作节点失败的判定

| 操作类型 | 失败条件 | 示例 |
|---------|---------|------|
| `request`（HTTP 请求） | HTTP 状态码非 2xx，或请求超时 | 404 Not Found, 500 Internal Server Error |
| `condition`（条件判断） | 条件表达式结果为 `false` | 检查 `status === 200` 不成立 |
| `create/read/update/delete`（数据操作） | 数据库操作失败 | 权限不足、数据不存在、约束违反 |
| `transform`（数据转换） | 转换逻辑抛出异常 | JavaScript 执行错误 |
| `exec`（执行代码） | 代码执行抛出异常 | `throw new Error()` |
| `mail`（发送邮件） | 邮件发送失败 | SMTP 服务器错误 |

### 5.3 无内置自动重试机制

#### 重要发现

**Directus Flow 没有内置的自动重试机制**。所有失败处理都需要通过 `reject` 分支手动实现。

#### 测试侧依据

从现有测试代码来看：
- **Blackbox 测试** (`tests/blackbox/tests/db/routes/flows/webhook.test.ts`)：只测试了 Webhook 触发和缓存行为，未涉及重试
- **E2E 测试** (`tests/e2e/tests/flows/webhook.test.ts`)：同样只测试了基本触发功能

#### 实现侧依据

从类型定义来看：
- `Operation` 接口只有 `resolve` 和 `reject` 字段，**没有** `retryCount`、`retryDelay` 等重试相关字段
- `Flow` 接口中**没有**任何重试配置选项

### 5.4 手动实现重试策略

#### 重试实现方式

通过组合 `condition`、`sleep`、`transform` 操作节点，可以手动实现重试逻辑：

```
┌─────────────────────────────────────────────────────────────────┐
│                    手动重试实现链路                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────┐                                          │
│   │  transform      │                                          │
│   │  初始化计数器   │                                          │
│   │  retryCount: 0 │                                          │
│   │  maxRetries: 3 │                                          │
│   └────────┬────────┘                                          │
│            │                                                    │
│            ▼                                                    │
│   ┌─────────────────┐                                          │
│   │  request        │                                          │
│   │  发送 HTTP 请求 │                                          │
│   └────────┬────────┘                                          │
│            │                                                    │
│     ┌──────┴──────┐                                            │
│     │             │                                            │
│     ▼             ▼                                            │
│ ┌────────┐    ┌────────┐                                       │
│ │resolve │    │ reject │                                       │
│ │ (成功)  │    │ (失败)  │                                       │
│ └────┬───┘    └────┬───┘                                       │
│      │             │                                            │
│      ▼             ▼                                            │
│ ┌──────────┐ ┌──────────────┐                                   │
│ │  结束    │ │ condition    │                                   │
│ │(记录成功)│ │ 检查重试次数 │                                   │
│ └──────────┘ │retryCount <  │                                   │
│              │  maxRetries   │                                   │
│              └──────┬─────────┘                                   │
│                     │                                              │
│              ┌──────┴──────┐                                       │
│              │             │                                       │
│              ▼             ▼                                       │
│         ┌────────┐   ┌────────┐                                   │
│         │resolve │   │ reject │                                   │
│         │ (可重试)│   │ (超限)  │                                   │
│         └────┬───┘   └────┬───┘                                   │
│              │             │                                        │
│              ▼             ▼                                        │
│         ┌──────────┐ ┌──────────┐                                  │
│         │transform │ │  结束    │                                  │
│         │retryCount│ │(通知失败)│                                  │
│         │   +1     │ └──────────┘                                  │
│         └────┬─────┘                                               │
│              │                                                      │
│              ▼                                                      │
│         ┌──────────┐                                               │
│         │  sleep   │                                               │
│         │ (延时)   │                                               │
│         └────┬─────┘                                               │
│              │                                                      │
│              └──────────▶ 回到 request 操作                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.5 重试边界条件

#### 重试次数边界

| 边界条件 | 行为 | 说明 |
|---------|------|------|
| `retryCount === 0` | 首次执行 | 还未发生任何失败 |
| `retryCount < maxRetries` | 可以重试 | 继续执行重试逻辑 |
| `retryCount >= maxRetries` | 停止重试 | 执行最终失败处理 |

#### 延时策略

| 策略 | 实现方式 | 示例 |
|-----|---------|------|
| 固定延时 | 每次 sleep 相同时间 | `duration: 1000`（1秒） |
| 线性退避 | 每次增加固定时间 | `duration: 1000 * retryCount` |
| 指数退避 | 每次时间翻倍 | `duration: 1000 * (2 ^ retryCount)` |

### 5.6 失败排查要点

#### 常见失败原因

| 失败类型 | 可能原因 | 排查方法 |
|---------|---------|---------|
| HTTP 请求失败 | 网络问题、目标服务不可用、认证错误 | 检查 URL、headers、body，测试目标服务 |
| 数据库操作失败 | 权限不足、数据不存在、约束违反 | 检查 accountability、验证数据完整性 |
| 代码执行失败 | JavaScript 语法错误、运行时异常 | 检查 exec/transform 代码逻辑 |
| 邮件发送失败 | SMTP 配置错误、收件人无效 | 检查邮件配置、测试邮件服务 |

#### 排查建议

1. **添加日志节点**：在关键位置添加 `log` 操作节点记录状态
2. **使用 reject 分支**：为每个可能失败的操作配置 `reject` 分支
3. **记录错误信息**：在失败分支中记录 `$last` 变量（包含上一个操作的错误信息）
4. **测试关键路径**：单独测试每个可能失败的操作节点

### 5.7 配置示例：带重试的外部通知

```typescript
const flowWithRetry = {
    name: '带重试的外部通知',
    trigger: 'event',
    options: {
        type: 'action',
        scope: ['items.create'],
        collections: ['orders']
    },
    accountability: '$trigger',
    operation: {
        // 1. 初始化重试计数器
        key: 'init_retry',
        type: 'transform',
        options: {
            data: {
                retryCount: 0,
                maxRetries: 3,
                orderData: '{{ $trigger.payload }}'
            }
        },
        resolve: {
            // 2. 发送通知请求
            key: 'send_notification',
            type: 'request',
            options: {
                url: 'https://api.external.com/order-webhook',
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'X-API-Key': '{{ $env.EXTERNAL_API_KEY }}'
                },
                body: '{{ init_retry.orderData }}'
            },
            resolve: {
                // 3a. 成功：记录日志
                key: 'log_success',
                type: 'log',
                options: {
                    message: '通知发送成功，订单 ID: {{ init_retry.orderData.id }}'
                }
            },
            reject: {
                // 3b. 失败：检查是否可以重试
                key: 'check_retry',
                type: 'condition',
                options: {
                    filter: {
                        'init_retry.retryCount': {
                            _lt: '{{ init_retry.maxRetries }}'
                        }
                    }
                },
                resolve: {
                    // 4a. 可以重试：增加计数，延时后重试
                    key: 'increment_retry',
                    type: 'transform',
                    options: {
                        data: {
                            ...'{{ init_retry }}',
                            retryCount: '{{ init_retry.retryCount + 1 }}'
                        }
                    },
                    resolve: {
                        key: 'wait_before_retry',
                        type: 'sleep',
                        options: {
                            // 指数退避：1s, 2s, 4s
                            duration: '{{ 1000 * (2 ^ increment_retry.retryCount) }}'
                        }
                        // ⚠️ 注意：这里需要手动将 reject 指回 send_notification
                        // 但 Directus 不支持循环引用，需要特殊处理
                    }
                },
                reject: {
                    // 4b. 超过重试次数：通知管理员
                    key: 'notify_admin',
                    type: 'mail',
                    options: {
                        to: ['admin@example.com'],
                        subject: '外部通知失败 - 订单: {{ init_retry.orderData.id }}',
                        body: '通知外部系统失败，已尝试 {{ init_retry.maxRetries }} 次。\n\n错误信息: {{ send_notification }}'
                    }
                }
            }
        }
    }
};
```

---

## 6. 完整投递链路总结

### 6.1 内部事件触发 Flow 的完整链路

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    内部事件 → Flow → 外部系统 完整链路                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 1: 事件触发                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  内部事件发生 (如 items.create, items.update)                     │ │
│  │       │                                                           │ │
│  │       ▼                                                           │ │
│  │  Event Trigger 检测匹配的 Flow                                    │ │
│  │       │                                                           │ │
│  │       ▼                                                           │ │
│  │  确定 accountability 权限模式                                      │ │
│  │  - '$public': 公开权限                                            │ │
│  │  - '$trigger': 触发者权限                                         │ │
│  │  - '$full': 完全权限                                              │ │
│  │  - '<role_uuid>': 指定角色权限                                    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                              │                                          │
│                              ▼                                          │
│  阶段 2: 操作执行                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  从起始 Operation 开始执行                                         │ │
│  │       │                                                           │ │
│  │       ▼                                                           │ │
│  │  执行当前 Operation                                                │ │
│  │  - 读取 options 配置                                              │ │
│  │  - 解析模板变量 ({{ $trigger }}, {{ $last }}, 等)               │ │
│  │  - 执行具体操作逻辑                                                │ │
│  │       │                                                           │ │
│  │  ┌────┴────┐                                                     │ │
│  │  │         │                                                     │ │
│  │  ▼         ▼                                                     │ │
│  │ 成功      失败                                                   │ │
│  │  │         │                                                     │ │
│  │  ▼         ▼                                                     │ │
│  │ resolve   reject                                                │ │
│  │  │         │                                                     │ │
│  │  └────┬────┘                                                     │ │
│  │       │                                                          │ │
│  │       ▼                                                          │ │
│  │  下一个 Operation (或结束)                                       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                              │                                          │
│                              ▼                                          │
│  阶段 3: 外部投递 (Request Operation)                                   │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  type: 'request'                                                  │ │
│  │       │                                                           │ │
│  │       ▼                                                           │ │
│  │  构建 HTTP 请求                                                    │ │
│  │  - url: 目标地址 (支持模板变量)                                   │ │
│  │  - method: GET/POST/PUT/PATCH/DELETE                            │ │
│  │  - headers: 请求头                                                │ │
│  │  - body: 请求体                                                   │ │
│  │       │                                                           │ │
│  │       ▼                                                           │ │
│  │  发送请求到外部系统                                                │ │
│  │       │                                                           │ │
│  │  ┌────┴────┐                                                     │ │
│  │  │         │                                                     │ │
│  │  ▼         ▼                                                     │ │
│  │ 2xx      非 2xx                                                 │ │
│  │  │         │                                                     │ │
│  │  ▼         ▼                                                     │ │
│  │ resolve   reject                                                │ │
│  │  │         │                                                     │ │
│  │  │         └────▶ 手动重试逻辑 (如已配置)                        │ │
│  │  │                  │                                            │ │
│  │  │                  ▼                                            │ │
│  │  │            超过重试次数                                        │ │
│  │  │                  │                                            │ │
│  │  │                  ▼                                            │ │
│  │  │            失败通知 (邮件/日志)                                │ │
│  │  │                                                               │ │
│  │  └────────────▶ 成功后续操作                                     │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 关键配置项总结

#### Trigger 配置

| 配置项 | 用途 | 可选值 |
|-------|------|-------|
| `trigger` | 触发器类型 | `'event'`, `'schedule'`, `'operation'`, `'webhook'`, `'manual'` |
| `accountability` | 权限模式 | `'$public'`, `'$trigger'`, `'$full'`, `<role_uuid>` |
| `options` | 触发器选项 | 依触发器类型而定 |

#### Webhook Trigger 特有配置

| 配置项 | 类型 | 作用 |
|-------|------|------|
| `options.method` | string | HTTP 方法：`'GET'` 或 `'POST'` |
| `options.cacheEnabled` | boolean | GET 请求是否启用缓存（POST 始终不缓存） |

#### Operation 配置

| 配置项 | 用途 | 说明 |
|-------|------|------|
| `type` | 操作类型 | `'request'`, `'condition'`, `'transform'`, `'log'`, 等 |
| `key` | 操作键 | Flow 内唯一，用于模板变量引用 |
| `options` | 操作配置 | 依操作类型而定 |
| `resolve` | 成功分支 | 操作成功时执行的下一节点 |
| `reject` | 失败分支 | 操作失败时执行的下一节点 |

---

## 7. 纠偏要点回顾

### 7.1 已确认的纠偏点

| 项目 | 之前的错误理解 | 正确取值 | 依据来源 |
|-----|---------------|---------|---------|
| **TriggerType** | 可能是 `'hook'` | `'event'` | 类型定义 `packages/types/src/flows.ts:1` |
| **Flow.accountability** | 混淆了 Collection 的类型 | `'$public'`, `'$trigger'`, `'$full'`, `<role_uuid>` | Specs 文档 `packages/specs/src/components/flow.yaml:40` |
| **Collection.accountability** | 未知 | `'all'`, `'activity'`, `null` | 类型定义 `packages/types/src/collection.ts:28` |
| **Webhook GET 缓存** | 默认不缓存 | **默认启用缓存** | 测试代码 `tests/blackbox/tests/db/routes/flows/webhook.test.ts` |
| **Webhook POST 缓存** | 可能受 cacheEnabled 影响 | **始终不缓存** | 测试代码 `tests/blackbox/tests/db/routes/flows/webhook.test.ts:149-224` |
| **内置重试机制** | 可能有 | **没有内置重试** | 类型定义无重试字段，测试代码无重试测试 |

### 7.2 重要提醒

1. **两种 accountability 概念不要混淆**：
   - `Collection.accountability`：控制审计日志记录范围
   - `Flow.accountability`：控制 Flow 执行时的权限上下文

2. **Webhook 缓存行为注意**：
   - GET 请求默认启用缓存，可通过 `cacheEnabled: false` 禁用
   - POST 请求始终不缓存，`cacheEnabled` 设置无效

3. **重试需要手动实现**：
   - Directus Flow 没有内置自动重试机制
   - 需要通过 `condition` + `sleep` + `transform` 手动实现
   - 注意循环引用问题（Directus 不支持操作节点间的循环引用）

4. **模板变量访问**：
   - 使用 `{{ $trigger }}` 访问触发器数据
   - 使用 `{{ $accountability }}` 访问权限上下文
   - 使用 `{{ <operation_key> }}` 访问指定操作的结果

---

## 8. 参考文件列表

### 实现侧依据

| 文件路径 | 说明 |
|---------|------|
| `packages/types/src/flows.ts` | TriggerType、Flow、Operation 类型定义 |
| `packages/types/src/collection.ts` | CollectionMeta.accountability 类型定义 |
| `packages/types/src/extensions/operations.ts` | OperationContext、OperationHandler 类型定义 |
| `packages/types/src/accountability.ts` | Accountability 权限上下文类型定义 |
| `packages/specs/src/components/flow.yaml` | Flow OpenAPI 组件定义（包含正确的 accountability 取值） |

### 测试侧依据

| 文件路径 | 说明 |
|---------|------|
| `tests/blackbox/tests/db/routes/flows/webhook.test.ts` | Webhook 触发和缓存行为的 Blackbox 测试 |
| `tests/e2e/tests/flows/webhook.test.ts` | Webhook 触发的 E2E 测试 |
| `packages/composables/src/use-collection.test.ts` | Collection accountability 的测试用例 |

---

*报告生成时间：2026-05-02*
*基于 Directus 代码库实际实现分析*
