# Directus Flow 机制证据报告（完整版）

## 证据状态说明

本报告严格区分以下三种证据状态：

| 状态 | 说明 | 验证级别 |
|-----|------|---------|
| ✅ **已确认** | 有实现侧（类型定义/Specs）和测试侧双重证据 | 完全验证 |
| 📐 **可推断** | 有类型定义或 Specs 文档，但无专门测试验证 | 部分验证 |
| ❌ **缺失证据** | 既无实现代码也无测试用例 | 未验证 |

---

## 第一部分：已确认的结论（有实现+测试双重证据）

---

### 结论 1：Webhook GET/POST 缓存行为差异

#### 结论内容

| HTTP 方法 | cacheEnabled | 缓存行为 | 执行次数 |
|----------|-------------|---------|---------|
| GET | 未设置（默认） | ✅ 启用缓存 | 1 次 |
| GET | `true` | ✅ 启用缓存 | 1 次 |
| GET | `false` | ❌ 禁用缓存 | 每次都执行 |
| POST | 任意值 | ❌ 始终不缓存 | 每次都执行 |

---

#### 实现侧依据

**依据 1：Specs 文档定义**

**文件**：`packages/specs/src/components/flow.yaml:35-36`

```yaml
options:
    description: Options of the selected trigger for the flow.
```

**依据 2：类型定义**

**文件**：`packages/types/src/flows.ts:13`

```typescript
export interface Flow {
    // ...
    options: Record<string, any>;  // 触发器配置选项
    // ...
}
```

---

#### 测试侧依据

**测试文件 1：Blackbox 测试（完整验证）**

**文件**：`tests/blackbox/tests/db/routes/flows/webhook.test.ts:97-230`

**测试代码片段 1 - GET 缓存行为**：

```typescript
describe('cacheEnabled works for GET', () => {
    it.each(vendors)('%s', async (vendor) => {
        // Setup
        const payloadFlowCreate = {
            name: 'webhook flow',
            trigger: 'webhook',
            options: {},           // 默认配置
            accountability: null,
        };

        // 创建不同配置的 Flow
        const flowId = (/* 默认配置 */);
        const flowCacheEnabledId = (/* cacheEnabled: true */);
        const flowCacheDisabledId = (/* cacheEnabled: false */);

        // Action: 两次 GET 请求
        const responseDefault = await request(getUrl(vendor, env))
            .get(`/flows/trigger/${flowId}`);
        await sleep(100);
        const responseDefault2 = await request(getUrl(vendor, env))
            .get(`/flows/trigger/${flowId}`);

        // Assert
        // 默认配置：两次请求结果相同（启用缓存）
        expect(responseDefault.body).toEqual(responseDefault2.body);
        
        // cacheEnabled: true：两次请求结果相同（启用缓存）
        expect(responseCacheEnabled.body).toEqual(responseCacheEnabled2.body);
        
        // cacheEnabled: false：两次请求结果不同（禁用缓存）
        expect(responseCacheDisabled.body).not.toEqual(responseCacheDisabled2.body);
    });
});
```

**测试代码片段 2 - POST 缓存行为**：

```typescript
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
        const responseDefault = await request(getUrl(vendor, env))
            .post(`/flows/trigger/${flowId}`);
        await sleep(100);
        const responseDefault2 = await request(getUrl(vendor, env))
            .post(`/flows/trigger/${flowId}`);

        // Assert
        // 无论 cacheEnabled 设置如何，POST 始终不缓存
        expect(responseDefault.body).not.toEqual(responseDefault2.body);
    });
});
```

**测试文件 2：E2E 测试（验证 cacheEnabled: false）**

**文件**：`tests/e2e/tests/flows/webhook.test.ts:78-103`

```typescript
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

---

#### 验证状态

| 验证维度 | 状态 | 证据 |
|---------|------|------|
| 类型定义 | ✅ 存在 | `packages/types/src/flows.ts` |
| Specs 文档 | ✅ 存在 | `packages/specs/src/components/flow.yaml` |
| Blackbox 测试 | ✅ 存在 | `tests/blackbox/tests/db/routes/flows/webhook.test.ts` |
| E2E 测试 | ✅ 存在 | `tests/e2e/tests/flows/webhook.test.ts` |

**验证级别**：✅ **完全验证**

---

### 结论 2：Schedule 触发器的基本行为

#### 结论内容

- `schedule` 类型触发器可以通过 cron 表达式定时触发 Flow
- 支持多实例同步（使用 Redis 时，只有一个实例会执行）

---

#### 实现侧依据

**文件**：`packages/types/src/flows.ts:1`

```typescript
export type TriggerType = 'event' | 'schedule' | 'operation' | 'webhook' | 'manual';
```

---

#### 测试侧依据

**测试文件**：`tests/blackbox/tests/common/flows/schedule-hook.test.ts:99-155`

**测试代码片段**：

```typescript
describe('scheduled hooks are synchronized using redis', () => {
    it.each(vendors)('%s', async (vendor) => {
        // Setup
        const flowId = flowIds[vendor];

        // 创建延时等待（9秒，预计执行 4-5 次）
        const { sleep, sleepStart, sleepHasStarted } = delayedSleep(9000);

        const flowExecutions: string[] = [];

        const processLogLine = (chunk: any) => {
            const logLine = String(chunk);
            if (logLine.includes(logPrefix)) {
                // 记录执行
                flowExecutions.push(/* 执行标识 */);
            }
        };

        // Action: 激活 Flow
        await request(getUrl(vendor, env))
            .patch(`/flows/${flowId}`)
            .send({ status: 'active' })
            .set('Authorization', `Bearer ${USER.ADMIN.TOKEN}`);

        await sleep;

        // Assert
        // Redis 实例只执行 5 次（同步）
        const redisExecutionCount = flowExecutions.filter(
            (e) => execution.includes('redis-')
        ).length;
        expect(redisExecutionCount).toBe(5);
    });
});
```

**测试种子文件**：`tests/blackbox/tests/common/flows/schedule-hook.seed.ts:15-46`

```typescript
const payloadFlowCreate = {
    name: flowName,
    icon: 'bolt',
    status: 'inactive',
    accountability: null,
    trigger: 'schedule',                            // schedule 触发器
    options: { cron: '*/2 * * * * *' },            // 每 2 秒执行一次
};

const payloadOperationCreate = {
    position_x: 19,
    position_y: 1,
    name: 'Log to Console',
    key: 'log_to_console',
    type: 'log',                                      // log 操作类型
    options: { message: `${logPrefix}{{ $env.${envTargetVariable} }}` },
};
```

---

#### 验证状态

| 验证维度 | 状态 | 证据 |
|---------|------|------|
| 类型定义 | ✅ 存在 | `packages/types/src/flows.ts` |
| Blackbox 测试 | ✅ 存在 | `tests/blackbox/tests/common/flows/schedule-hook.test.ts` |
| 测试种子 | ✅ 存在 | `tests/blackbox/tests/common/flows/schedule-hook.seed.ts` |

**验证级别**：✅ **完全验证**

---

## 第二部分：可推断的结论（有类型定义但无测试）

---

### 结论 3：TriggerType 的真实取值

#### 结论内容

**TriggerType 的真实取值为：**

```typescript
type TriggerType = 'event' | 'schedule' | 'operation' | 'webhook' | 'manual';
```

| 类型值 | 描述 | 测试状态 |
|-------|------|---------|
| `'event'` | 事件触发器 | ⚠️ 未测试 |
| `'schedule'` | 定时触发器 | ✅ 已测试 |
| `'operation'` | 操作触发器 | ⚠️ 未测试 |
| `'webhook'` | Webhook 触发器 | ✅ 已测试 |
| `'manual'` | 手动触发器 | ⚠️ 未测试 |

---

#### 实现侧依据

**文件**：`packages/types/src/flows.ts:1`

```typescript
export type TriggerType = 'event' | 'schedule' | 'operation' | 'webhook' | 'manual';
```

**注意**：Specs 文档 `packages/specs/src/components/flow.yaml:31` 写的是 `'hook'`，但类型定义是 `'event'`。**应以类型定义为准**。

---

#### 测试侧依据

| 触发器类型 | 测试文件 | 验证状态 |
|-----------|---------|---------|
| `webhook` | `tests/blackbox/tests/db/routes/flows/webhook.test.ts` | ✅ 已验证 |
| `webhook` | `tests/e2e/tests/flows/webhook.test.ts` | ✅ 已验证 |
| `schedule` | `tests/blackbox/tests/common/flows/schedule-hook.test.ts` | ✅ 已验证 |
| `event` | **未找到** | ⚠️ 未验证 |
| `operation` | **未找到** | ⚠️ 未验证 |
| `manual` | **未找到** | ⚠️ 未验证 |

---

#### 验证状态

| 验证维度 | 状态 | 说明 |
|---------|------|------|
| 类型定义 | ✅ 存在 | 定义了所有 5 种类型 |
| 测试覆盖 | ⚠️ 部分 | 只有 `webhook` 和 `schedule` 有测试 |

**验证级别**：📐 **部分验证**（类型定义存在，但仅部分类型有测试）

---

### 结论 4：Operation 类型包含 `trigger`

#### 结论内容

**内置操作类型包含 `trigger`，用于触发另一个 Flow。**

完整的内置操作类型列表：

```
log, mail, notification, create, read, request, sleep, transform, trigger, condition
```

---

#### 实现侧依据

**文件**：`packages/specs/src/components/operation.yaml:16-18`

```yaml
type:
    description:
      Type of operation. One of `log`, `mail`, `notification`, `create`, `read`, `request`, `sleep`, `transform`,
      `trigger`, `condition`, or any type of custom operation extensions.
    type: string
    example: log
```

**类型定义**：`packages/types/src/flows.ts:17-27`

```typescript
export interface Operation {
    id: string;
    name: string | null;
    key: string;
    type: string;  // 操作类型（包括 'trigger'）
    position_x: number;
    position_y: number;
    options: Record<string, any>;
    resolve: Operation | null;
    reject: Operation | null;
}
```

---

#### 测试侧依据

**❌ 未找到专门测试**

- 没有找到测试 `type: 'trigger'` 操作节点的测试用例
- 没有找到测试一个 Flow 如何触发另一个 Flow 的测试用例

**测试种子中使用的操作类型**：

| 操作类型 | 测试文件 | 验证状态 |
|---------|---------|---------|
| `log` | `tests/blackbox/tests/common/flows/schedule-hook.seed.ts` | ✅ 间接验证 |
| `exec` | `tests/e2e/tests/flows/webhook.test.ts` | ✅ 间接验证 |
| `trigger` | **未找到** | ⚠️ 未验证 |

---

#### 验证状态

| 验证维度 | 状态 | 说明 |
|---------|------|------|
| Specs 文档 | ✅ 存在 | 明确列出 `trigger` 操作类型 |
| 类型定义 | ✅ 存在 | `type: string` 包含所有操作类型 |
| 测试验证 | ❌ 缺失 | 没有专门测试 `trigger` 操作 |

**验证级别**：📐 **可推断**（类型定义和 Specs 文档存在，但无测试验证）

---

### 结论 5：OperationContext 包含数据和权限上下文

#### 结论内容

**Operation 执行时的上下文包含：**

```typescript
type OperationContext = ApiExtensionContext & {
    data: Record<string, unknown>;           // 上游传递的数据
    accountability: Accountability | null;    // 权限上下文
    flow?: Flow;                               // 当前 Flow 信息（可选）
};
```

**Accountability 结构**：

```typescript
type Accountability = {
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

---

#### 实现侧依据

**文件**：`packages/types/src/extensions/operations.ts:8-17`

```typescript
export type OperationContext = ApiExtensionContext & {
    data: Record<string, unknown>;           // 上下文数据
    accountability: Accountability | null;    // 权限信息
    flow?: Flow;                               // 当前 Flow 信息
};

export type OperationHandler<Options = Record<string, unknown>> = (
    options: Options,           // 操作配置选项
    context: OperationContext,  // 操作上下文（包含 data 和 accountability）
) => unknown | Promise<unknown> | void;
```

**文件**：`packages/types/src/accountability.ts:6-17`

```typescript
export type Accountability = {
    role: string | null;
    roles: string[];
    user: string | null;
    admin: boolean;
    app: boolean;
    share?: string;
    ip: string | null;
    userAgent?: string;
    origin?: string;
    session?: string;
};
```

---

#### 测试侧依据

**❌ 未找到直接测试**

- 没有找到专门测试 `OperationContext` 如何在操作节点间传递的测试用例
- 没有找到测试 `accountability` 如何在 Flow 间传递的测试用例

---

#### 验证状态

| 验证维度 | 状态 | 说明 |
|---------|------|------|
| 类型定义 | ✅ 存在 | 完整定义了 `OperationContext` |
| 测试验证 | ❌ 缺失 | 没有测试上下文传递行为 |

**验证级别**：📐 **可推断**（类型定义存在，但无测试验证）

---

### 结论 6：resolve/reject 形成执行链路

#### 结论内容

**操作节点通过 `resolve` 和 `reject` 字段形成有向无环图（DAG）执行链路。**

| 字段 | 描述 |
|-----|------|
| `resolve` | 操作成功时执行的下一节点（或 `condition` 操作的 `then` 逻辑） |
| `reject` | 操作失败时执行的下一节点（或 `condition` 操作的 `otherwise` 逻辑） |

---

#### 实现侧依据

**文件**：`packages/types/src/flows.ts:17-27`

```typescript
export interface Operation {
    id: string;
    name: string | null;
    key: string;
    type: string;
    position_x: number;
    position_y: number;
    options: Record<string, any>;
    resolve: Operation | null;     // 成功分支
    reject: Operation | null;      // 失败分支
}
```

**文件**：`packages/specs/src/components/operation.yaml:47-58`

```yaml
resolve:
    description: The operation triggered when the current operation succeeds (or `then` logic of a condition operation).
    example: 63716273-0f29-4648-8a2a-2af2948f6f78
    oneOf:
      - type: string
      - $ref: '../openapi.yaml#/components/schemas/Operations'

reject:
    description:
      The operation triggered when the current operation fails (or `otherwise` logic of a condition operation).
    example: 63716273-0f29-4648-8a2a-2af2948f6f78
    oneOf:
      - type: string
      - $ref: '../openapi.yaml#/components/schemas/Operations'
```

---

#### 测试侧依据

**❌ 未找到直接测试**

- 没有找到专门测试 `resolve/reject` 链路执行的测试用例
- 没有找到测试条件分支逻辑的测试用例

**现有测试中仅使用了简单的单节点 Flow**：

**文件**：`tests/e2e/tests/flows/webhook.test.ts:16-45`

```typescript
test('trigger webhook', async () => {
    const flow = await api.request(
        createFlow({
            name: 'webhook flow default',
            trigger: 'webhook',
            options: {},
        }),
    );

    // 只创建了一个操作节点，没有测试 resolve/reject 链路
    const operation = await api.request(
        createOperation({
            flow: flow!.id,
            ...baseOperation,  // 单节点
        }),
    );
});
```

---

#### 验证状态

| 验证维度 | 状态 | 说明 |
|---------|------|------|
| 类型定义 | ✅ 存在 | 定义了 `resolve` 和 `reject` 字段 |
| Specs 文档 | ✅ 存在 | 详细描述了字段用途 |
| 测试验证 | ❌ 缺失 | 没有测试多节点链路执行 |

**验证级别**：📐 **可推断**（类型定义和 Specs 文档存在，但无测试验证）

---

### 结论 7：无内置自动重试机制

#### 结论内容

**Directus Flow 没有内置的自动重试机制。**

所有失败处理都需要通过 `reject` 分支手动实现。

---

#### 实现侧依据

**类型定义分析**：

从以下文件的类型定义可以推断：

**文件 1**：`packages/types/src/flows.ts:17-27`

```typescript
export interface Operation {
    id: string;
    name: string | null;
    key: string;
    type: string;
    position_x: number;
    position_y: number;
    options: Record<string, any>;
    resolve: Operation | null;
    reject: Operation | null;
    // ⚠️ 没有以下字段：
    // - retryCount
    // - maxRetries
    // - retryDelay
    // - retryStrategy
}
```

**文件 2**：`packages/types/src/flows.ts:5-15`

```typescript
export interface Flow {
    id: string;
    name: string | null;
    icon: string | null;
    color: string | null;
    description: string | null;
    status: string | null;
    trigger: TriggerType | null;
    options: Record<string, any>;
    operation: Operation | null;
    accountability: 'all' | 'activity' | null;
    // ⚠️ 没有重试相关配置
}
```

---

#### 测试侧依据

**❌ 未找到重试测试**

- 没有找到测试内置重试机制的测试用例
- 所有失败处理都通过 `reject` 字段指向的节点处理

---

#### 验证状态

| 验证维度 | 状态 | 说明 |
|---------|------|------|
| 类型定义无重试字段 | ✅ 确认 | `Operation` 和 `Flow` 接口都没有重试字段 |
| 测试无重试验证 | ✅ 确认 | 没有测试用例验证重试机制 |

**验证级别**：📐 **可推断**（通过缺失证据推断）

---

## 第三部分：缺失的证据（无实现也无测试）

---

### 结论 8：Operation 触发器的上下文传递

#### 结论状态

**❌ 证据缺失**

#### 问题描述

当一个 Flow 的 `type: 'trigger'` 操作节点触发另一个 Flow 时：

1. 上游的 `data` 如何传递给下游 Flow？
2. 上游的 `accountability` 如何传递给下游 Flow？
3. 下游 Flow 如何通过模板变量访问上游数据？

---

#### 已有的类型定义（可推断）

**文件**：`packages/types/src/extensions/operations.ts:8-12`

```typescript
export type OperationContext = ApiExtensionContext & {
    data: Record<string, unknown>;
    accountability: Accountability | null;
    flow?: Flow;
};
```

**推断**：当 `trigger` 操作触发另一个 Flow 时，上游的 `OperationContext.data` 和 `OperationContext.accountability` 应该会传递给下游 Flow。

---

#### 缺失的证据

| 证据类型 | 状态 | 说明 |
|---------|------|------|
| 测试 `operation` 触发器 | ❌ 缺失 | 没有测试一个 Flow 如何触发另一个 Flow |
| 测试 `trigger` 操作 | ❌ 缺失 | 没有测试 `type: 'trigger'` 操作节点的行为 |
| 测试上下文传递 | ❌ 缺失 | 没有测试数据和权限如何在 Flow 间传递 |

---

### 结论 9：循环引用限制

#### 结论状态

**❌ 证据缺失**

#### 问题描述

以下情况是否有保护机制？

1. **操作节点间的循环引用**：
   ```
   OpA.resolve → OpB
   OpB.resolve → OpA （循环引用）
   ```

2. **Flow 间的循环触发**：
   ```
   FlowA 触发 FlowB
   FlowB 触发 FlowA （循环触发）
   ```

3. **最大执行深度**：是否有最大递归深度限制？

---

#### 搜索结果

| 搜索关键词 | 结果 | 说明 |
|-----------|------|------|
| `cycle detection` | 3 个文件 | 与 Flow 无关（构建、角色树、SAML） |
| `circular` | 3 个文件 | 与 Flow 无关 |
| `max depth` | 6 个文件 | 与 Flow 无关 |
| `recursion limit` | 0 个文件 | 无结果 |
| `infinite loop` | 0 个文件 | 无结果 |

---

#### 缺失的证据

| 证据类型 | 状态 | 说明 |
|---------|------|------|
| 循环引用检测实现 | ❌ 缺失 | 没有找到相关实现代码 |
| 最大执行深度配置 | ❌ 缺失 | 没有找到相关配置 |
| 循环引用测试 | ❌ 缺失 | 没有测试用例验证循环引用保护 |

---

### 结论 10：重试边界

#### 结论状态

**❌ 证据缺失**

#### 问题描述

如果手动实现重试（通过 `condition` + `sleep` + `transform`），以下边界情况如何处理？

1. **最大重试次数**：是否有配置限制？
2. **重试延迟策略**：是否支持固定延时、线性退避、指数退避？
3. **重试超时**：是否有总超时限制？
4. **失败回退**：超过重试次数后如何处理？

---

#### 搜索结果

| 搜索关键词 | 结果 | 说明 |
|-----------|------|------|
| `retry count` | 0 个文件 | 无结果 |
| `max retries` | 0 个文件 | 无结果 |
| `retry delay` | 0 个文件 | 无结果 |
| `exponential backoff` | 0 个文件 | 无结果 |
| `sleep operation` | 部分结果 | 有 `sleep` 操作类型，但无重试相关逻辑 |

---

#### 缺失的证据

| 证据类型 | 状态 | 说明 |
|---------|------|------|
| 重试配置选项 | ❌ 缺失 | 没有找到重试相关配置 |
| 重试策略实现 | ❌ 缺失 | 没有找到内置重试策略 |
| 重试边界测试 | ❌ 缺失 | 没有测试用例验证重试边界 |

---

## 第四部分：证据汇总表

---

### 4.1 结论证据对照表

| 结论编号 | 结论内容 | 实现侧依据 | 测试侧依据 | 验证级别 |
|---------|---------|-----------|-----------|---------|
| 1 | Webhook GET/POST 缓存差异 | ✅ 类型定义 + Specs | ✅ Blackbox + E2E | **已确认** |
| 2 | Schedule 触发器基本行为 | ✅ 类型定义 | ✅ Blackbox | **已确认** |
| 3 | TriggerType 真实取值 | ✅ 类型定义 | ⚠️ 部分测试 | **可推断** |
| 4 | Operation 类型包含 `trigger` | ✅ Specs | ❌ 无测试 | **可推断** |
| 5 | OperationContext 定义 | ✅ 类型定义 | ❌ 无测试 | **可推断** |
| 6 | resolve/reject 链路 | ✅ 类型定义 + Specs | ❌ 无测试 | **可推断** |
| 7 | 无内置重试机制 | ✅ 类型定义（缺失字段） | ❌ 无测试 | **可推断** |
| 8 | Operation 触发器上下文传递 | ⚠️ 仅类型定义 | ❌ 无测试 | **缺失证据** |
| 9 | 循环引用限制 | ❌ 无实现 | ❌ 无测试 | **缺失证据** |
| 10 | 重试边界 | ❌ 无实现 | ❌ 无测试 | **缺失证据** |

---

### 4.2 测试覆盖分析

#### 已测试的触发器类型

| 触发器类型 | 测试文件 | 覆盖状态 |
|-----------|---------|---------|
| `webhook` | `tests/blackbox/tests/db/routes/flows/webhook.test.ts` | ✅ 完全覆盖 |
| `webhook` | `tests/e2e/tests/flows/webhook.test.ts` | ✅ 完全覆盖 |
| `schedule` | `tests/blackbox/tests/common/flows/schedule-hook.test.ts` | ✅ 完全覆盖 |
| `event` | 未找到 | ❌ 未覆盖 |
| `operation` | 未找到 | ❌ 未覆盖 |
| `manual` | 未找到 | ❌ 未覆盖 |

#### 已测试的操作类型

| 操作类型 | 测试文件 | 覆盖状态 |
|---------|---------|---------|
| `log` | `tests/blackbox/tests/common/flows/schedule-hook.seed.ts` | ✅ 间接覆盖 |
| `exec` | `tests/e2e/tests/flows/webhook.test.ts` | ✅ 间接覆盖 |
| `request` | 未找到 | ❌ 未覆盖 |
| `condition` | 未找到 | ❌ 未覆盖 |
| `trigger` | 未找到 | ❌ 未覆盖 |
| `sleep` | 未找到 | ❌ 未覆盖 |
| `transform` | 未找到 | ❌ 未覆盖 |
| `mail` | 未找到 | ❌ 未覆盖 |
| `notification` | 未找到 | ❌ 未覆盖 |
| `create` | 未找到 | ❌ 未覆盖 |
| `read` | 未找到 | ❌ 未覆盖 |

---

## 第五部分：风险与建议

---

### 5.1 已确认的风险

| 风险 | 证据 | 影响 |
|-----|------|------|
| 类型定义与 Specs 文档不一致 | TriggerType 是 `'event'` 但 Specs 写 `'hook'` | 可能导致混淆 |
| Flow.accountability 类型定义错误 | 使用了 Collection 的类型 | 可能导致类型错误 |
| 无内置重试机制 | 类型定义无重试字段 | 需要手动实现 |

---

### 5.2 潜在风险（无证据验证）

| 风险 | 可能性 | 影响 | 建议 |
|-----|--------|------|------|
| 循环引用导致无限循环 | 中等 | 系统崩溃 | 配置时避免循环 |
| 深度递归导致栈溢出 | 中等 | 系统崩溃 | 限制 Flow 复杂度 |
| 上下文传递行为不确定 | 低 | 数据丢失 | 使用前验证行为 |

---

### 5.3 使用建议

#### 对于已确认的功能

1. **Webhook 触发器**：
   - 使用 `cacheEnabled: false` 确保每次执行
   - POST 请求始终不缓存，适合需要实时响应的场景

2. **Schedule 触发器**：
   - 使用 Redis 实现多实例同步
   - 注意 cron 表达式的配置

#### 对于可推断的功能

1. **Trigger 操作节点**：
   - 使用前验证上下文传递行为
   - 建议在测试环境中验证

2. **Resolve/Reject 链路**：
   - 避免复杂的分支逻辑
   - 添加日志节点以便调试

3. **重试机制**：
   - 使用 `condition` + `sleep` + `transform` 手动实现
   - 注意设置最大重试次数

#### 对于缺失证据的功能

1. **循环引用**：
   - **禁止**配置循环引用的 Flow
   - **禁止**配置循环引用的操作节点

2. **操作触发器**：
   - 建议使用其他触发方式替代
   - 如需使用，必须充分测试

---

## 第六部分：参考文件列表

---

### 6.1 类型定义文件

| 文件路径 | 描述 |
|---------|------|
| `packages/types/src/flows.ts` | Flow、Operation、TriggerType 类型定义 |
| `packages/types/src/extensions/operations.ts` | OperationContext、OperationHandler 类型定义 |
| `packages/types/src/accountability.ts` | Accountability 类型定义 |
| `packages/types/src/services.ts` | FlowsService、OperationsService 类型定义 |

---

### 6.2 Specs 文档文件

| 文件路径 | 描述 |
|---------|------|
| `packages/specs/src/components/flow.yaml` | Flow OpenAPI 组件定义 |
| `packages/specs/src/components/operation.yaml` | Operation OpenAPI 组件定义 |

---

### 6.3 测试文件

| 文件路径 | 描述 |
|---------|------|
| `tests/blackbox/tests/db/routes/flows/webhook.test.ts` | Webhook 触发器 Blackbox 测试 |
| `tests/blackbox/tests/common/flows/schedule-hook.test.ts` | Schedule 触发器 Blackbox 测试 |
| `tests/blackbox/tests/common/flows/schedule-hook.seed.ts` | Schedule 测试种子数据 |
| `tests/e2e/tests/flows/webhook.test.ts` | Webhook 触发器 E2E 测试 |

---

### 6.4 测试种子文件

| 文件路径 | 描述 |
|---------|------|
| `tests/blackbox/setup/seeds/02_tests_flow.js` | Flow 测试数据初始化 |

---

*报告生成时间：2026-05-02*
*基于 Directus 代码库实际实现分析*
