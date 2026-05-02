# Directus 流程（Flow）与扩展事件系统深度分析

## 1. 核心架构概览

### 1.1 系统组成

Directus 的流程与事件系统由以下核心组件构成：

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| **FlowManager** | `api/src/flows.ts` | 流程管理核心，负责触发器注册、流程执行、操作节点调度 |
| **Emitter** | `api/src/emitter.ts` | 事件发射器，实现 Filter/Action/Init 三种事件类型的发布订阅 |
| **ExtensionManager** | `api/src/extensions/manager.ts` | 扩展管理器，负责加载和注册操作节点、Hook 扩展 |
| **ItemsService** | `api/src/services/items.ts` | 数据操作服务，在数据写入的关键阶段触发事件 |

### 1.2 架构关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                        数据写入请求                                │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ItemsService (数据服务层)                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │ emitFilter() │───▶│  实际写入     │───▶│ emitAction() │     │
│  │ (写入前拦截)  │    │  (事务内)     │    │ (写入后通知)  │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
└───────────────────────┬─────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Emitter (事件发射器)                         │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │ filterEmitter   │    │ actionEmitter   │    │ initEmitter │ │
│  │ (同步/可修改)    │    │ (异步/仅监听)    │    │(初始化事件) │ │
│  └────────┬────────┘    └────────┬────────┘    └─────────────┘ │
└───────────┼────────────────────────┼──────────────────────────────┘
            │                        │
            ▼                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                    事件监听器注册中心                              │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ FlowManager (流程触发器)                                     │ │
│  │   - Event Trigger: 注册 filter/action 事件监听器            │ │
│  │   - Schedule Trigger: Cron 调度任务                          │ │
│  │   - Operation/Webhook/Manual Trigger: 其他触发方式           │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │ Extension Hooks (扩展 Hook)                                  │ │
│  │   - defineHook() 中通过 filter()/action() API 注册监听器    │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 触发器注册与监听机制

### 2.1 FlowManager 触发器类型

FlowManager 在 `load()` 方法中根据流程配置注册不同类型的触发器：

```typescript
// 源码位置: api/src/flows.ts:156-366
private async load(): Promise<void> {
    // 1. 从数据库读取所有激活的流程
    const flows = await flowsService.readByQuery({
        filter: { status: { _eq: 'active' } },
        fields: ['*', 'operations.*'],
    });
    
    // 2. 为每个流程构建执行树并注册触发器
    for (const flow of flowTrees) {
        this.flows[flow.id] = flow;
        
        // 根据 trigger 类型注册不同的触发器
        if (flow.trigger === 'event') {
            this.registerEventTrigger(flow);
        } else if (flow.trigger === 'schedule') {
            this.registerScheduleTrigger(flow);
        } else if (flow.trigger === 'operation') {
            this.registerOperationTrigger(flow);
        } else if (flow.trigger === 'webhook') {
            this.registerWebhookTrigger(flow);
        } else if (flow.trigger === 'manual') {
            this.registerManualTrigger(flow);
        }
    }
}
```

### 2.2 事件触发器（Event Trigger）注册详解

事件触发器是最常用的类型，支持 **Filter** 和 **Action** 两种子类型：

#### 2.2.1 Filter 类型触发器（同步阻塞）

```typescript
// 源码位置: api/src/flows.ts:196-213
if (flow.options['type'] === 'filter') {
    // Filter Handler: 接收 payload，可修改并返回
    const handler: FilterHandler = (payload, meta, context) =>
        this.executeFlow(
            flow,
            { payload, ...meta },  // 将 payload 合并到触发数据中
            {
                accountability: context['accountability'],
                database: context['database'],
                getSchema: context['schema'] ? () => context['schema'] : getSchema,
            },
        );

    // 注册到 filterEmitter
    events.forEach((event) => emitter.onFilter(event, handler));

    this.triggerHandlers.push({
        id: flow.id,
        events: events.map((event) => ({ type: 'filter', name: event, handler })),
    });
}
```

**关键特性**：
- **同步执行**：阻塞主流程，直到 handler 返回
- **可修改数据**：返回值会作为新的 payload 继续传递
- **在事务内执行**：可以参与事务回滚

#### 2.2.2 Action 类型触发器（异步非阻塞）

```typescript
// 源码位置: api/src/flows.ts:214-228
else if (flow.options['type'] === 'action') {
    // Action Handler: 只接收 meta，不返回值
    const handler: ActionHandler = (meta, context) =>
        this.executeFlow(flow, meta, {
            accountability: context['accountability'],
            database: getDatabase(),
            getSchema: context['schema'] ? () => context['schema'] : getSchema,
        });

    // 注册到 actionEmitter
    events.forEach((event) => emitter.onAction(event, handler));

    this.triggerHandlers.push({
        id: flow.id,
        events: events.map((event) => ({ type: 'action', name: event, handler })),
    });
}
```

**关键特性**：
- **异步执行**：使用 `emitAsync`，不阻塞主流程
- **不可修改数据**：只接收通知，无法修改原始数据
- **独立数据库连接**：使用新的 `getDatabase()`，不参与原事务

### 2.3 事件名称映射规则

事件触发器的 scope 会根据集合类型转换为实际事件名：

```typescript
// 源码位置: api/src/flows.ts:172-194
if (flow.trigger === 'event') {
    let events: string[] = [];

    if (flow.options?.['scope']) {
        events = toArray(flow.options['scope'])
            .map((scope: string) => {
                // items.create/items.update/items.delete 特殊处理
                if (['items.create', 'items.update', 'items.delete'].includes(scope)) {
                    if (!flow.options?.['collections']) return [];

                    return toArray(flow.options['collections']).map((collection: string) => {
                        // 系统集合: 去掉 directus_ 前缀
                        if (isSystemCollection(collection)) {
                            const action = scope.split('.')[1];
                            return collection.substring(9) + '.' + action;
                            // 例: directus_users.create → users.create
                        }
                        // 用户集合: 组合成 {collection}.{scope}
                        return `${collection}.${scope}`;
                        // 例: articles.items.create
                    });
                }
                return scope;
            })
            .flat();
    }
}
```

**事件名映射示例**：

| 配置项 | 用户集合 `articles` | 系统集合 `directus_users` |
|--------|---------------------|---------------------------|
| `scope: items.create` | `articles.items.create` | `users.create` |
| `scope: items.update` | `articles.items.update` | `users.update` |
| `scope: items.delete` | `articles.items.delete` | `users.delete` |

### 2.4 扩展 Hook 注册机制

除了 Flow 的事件触发器，扩展可以通过 Hook API 直接注册事件监听器：

```typescript
// 源码位置: api/src/extensions/manager.ts:885-926
private registerHook(hookRegistrationCallback: HookConfig, name: string): PromiseCallback[] {
    const unregisterFunctions: PromiseCallback[] = [];

    const hookRegistrationContext = {
        // Filter Hook: 同步，可修改 payload
        filter: <T = unknown>(event: string, handler: FilterHandler<T>) => {
            emitter.onFilter(event, handler);
            unregisterFunctions.push(() => {
                emitter.offFilter(event, handler);
            });
        },
        // Action Hook: 异步，仅监听
        action: (event: string, handler: ActionHandler) => {
            emitter.onAction(event, handler);
            unregisterFunctions.push(() => {
                emitter.offAction(event, handler);
            });
        },
        // Init Hook: 初始化事件
        init: (event: string, handler: InitHandler) => {
            emitter.onInit(event, handler);
            unregisterFunctions.push(() => {
                emitter.offInit(name, handler);
            });
        },
        // Schedule Hook: Cron 调度
        schedule: (cron: string, handler: ScheduleHandler) => {
            // ...
        },
    };

    // 执行扩展的 Hook 注册回调
    hookRegistrationCallback(hookRegistrationContext);

    return unregisterFunctions;
}
```

**扩展中使用示例**：
```typescript
// 扩展的 src/index.ts
import { defineHook } from '@directus/extensions-sdk';

export default defineHook(({ filter, action }) => {
    // Filter Hook: 在数据写入前修改 payload
    filter('articles.items.create', async (payload, meta, context) => {
        return {
            ...payload,
            slug: payload.title.toLowerCase().replace(/\s+/g, '-'),
        };
    });

    // Action Hook: 数据写入后发送通知
    action('articles.items.create', async (meta, context) => {
        await sendNotification(`New article created: ${meta.key}`);
    });
});
```

---

## 3. 数据写入阶段的事件介入点

### 3.1 Emitter 核心实现

```typescript
// 源码位置: api/src/emitter.ts:6-114
export class Emitter {
    private filterEmitter;   // 同步 Filter 事件
    private actionEmitter;   // 异步 Action 事件
    private initEmitter;     // Init 事件

    constructor() {
        const emitterOptions = {
            wildcard: true,           // 支持通配符: items.*
            verboseMemoryLeak: true,
            delimiter: '.',            // 事件名分隔符
            ignoreErrors: true,        // 忽略未指定事件的错误
        };

        this.filterEmitter = new ee2.EventEmitter2(emitterOptions);
        this.actionEmitter = new ee2.EventEmitter2(emitterOptions);
        this.initEmitter = new ee2.EventEmitter2(emitterOptions);
    }
```

### 3.2 Filter 事件发射（同步阻塞）

```typescript
// 源码位置: api/src/emitter.ts:34-60
public async emitFilter<T>(
    event: string | string[],
    payload: T,
    meta: Record<string, any>,
    context: EventContext | null = null,
): Promise<T> {
    const events = Array.isArray(event) ? event : [event];

    const eventListeners = events.map((event) => ({
        event,
        listeners: this.filterEmitter.listeners(event) as FilterHandler<T>[],
    }));

    let updatedPayload = payload;

    // 顺序执行所有监听器，链式传递 payload
    for (const { event, listeners } of eventListeners) {
        for (const listener of listeners) {
            const result = await listener(updatedPayload, { event, ...meta }, context ?? this.getDefaultContext());

            // 如果返回值不为 undefined，则更新 payload
            if (result !== undefined) {
                updatedPayload = result;
            }
        }
    }

    return updatedPayload;
}
```

**关键行为**：
1. **顺序执行**：多个监听器按注册顺序执行
2. **链式传递**：前一个监听器的返回值作为下一个的输入
3. **可选修改**：返回 `undefined` 表示不修改，保持原 payload

### 3.3 Action 事件发射（异步非阻塞）

```typescript
// 源码位置: api/src/emitter.ts:62-72
public emitAction(event: string | string[], meta: Record<string, any>, context: EventContext | null = null): void {
    const logger = useLogger();
    const events = Array.isArray(event) ? event : [event];

    for (const event of events) {
        // 使用 emitAsync，异步执行
        this.actionEmitter.emitAsync(event, { event, ...meta }, context ?? this.getDefaultContext())
            .catch((err) => {
                // Action 事件的错误仅记录日志，不影响主流程
                logger.warn(`An error was thrown while executing action "${event}"`);
                logger.warn(err);
            });
    }
}
```

**关键行为**：
1. **异步执行**：`emitAsync` 返回 Promise，但不等待
2. **错误隔离**：Action handler 的错误被 catch 并仅记录日志
3. **无返回值**：Action handler 的返回值被忽略

### 3.4 ItemsService 中的事件触发时序

#### 3.4.1 创建数据（createOne）

```
┌────────────────────────────────────────────────────────────────────┐
│                    createOne 执行时序                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. 开始事务                                                        │
│      │                                                              │
│      ▼                                                              │
│  2. emitFilter('items.create', payload, { collection })           │
│     ├── 同步执行所有 Filter Hook/Flow                               │
│     ├── 可修改 payload                                               │
│     └── 在事务内，失败可回滚                                          │
│      │                                                              │
│      ▼                                                              │
│  3. processPayload (权限处理)                                       │
│      │                                                              │
│      ▼                                                              │
│  4. 实际数据库 INSERT                                                │
│      │                                                              │
│      ▼                                                              │
│  5. 处理关系数据 / 嵌套写入                                          │
│      │                                                              │
│      ▼                                                              │
│  6. 提交事务                                                        │
│      │                                                              │
│      ▼                                                              │
│  7. emitAction('items.create', { payload, key, collection })      │
│     ├── 异步执行所有 Action Hook/Flow                               │
│     ├── 无法修改数据                                                 │
│     └── 使用独立数据库连接                                           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

**源码位置**: `api/src/services/items.ts:127-419`

```typescript
async createOne(data: Partial<Item>, opts: MutationOptions = {}): Promise<PrimaryKey> {
    const primaryKey: PrimaryKey = await transaction(this.knex, async (trx) => {
        // ===== 阶段1: Filter 事件 (写入前) =====
        const payloadAfterHooks =
            opts.emitEvents !== false
                ? await emitter.emitFilter(
                        this.eventScope === 'items'
                            ? ['items.create', `${this.collection}.items.create`]
                            : `${this.eventScope}.create`,
                        payload,
                        { collection: this.collection },
                        {
                            database: trx,           // 使用事务连接
                            schema: this.schema,
                            accountability: this.accountability,
                        },
                    )
                : payload;

        // ===== 阶段2: 数据处理和实际写入 =====
        const payloadWithPresets = this.accountability
            ? await processPayload(...)
            : payloadAfterHooks;

        // ... 实际 INSERT 操作 ...
        const primaryKey = await payloadService.processValues(...);

        // 准备 Action 事件数据
        actionEvent = {
            event: this.eventScope === 'items' ? 'items.create' : `${this.eventScope}.create`,
            meta: {
                payload: actionHookPayload,
                key: primaryKey,
                collection: this.collection,
            },
            context: {
                database: this.knex,
                schema: this.schema,
                accountability: this.accountability,
            },
        };

        return primaryKey;
    });

    // ===== 阶段3: Action 事件 (写入后，事务已提交) =====
    if (opts.emitEvents !== false) {
        if (opts.bypassEmitAction) {
            opts.bypassEmitAction(actionEvent);
        } else {
            emitter.emitAction(actionEvent.event, actionEvent.meta, actionEvent.context);
        }
        // ... 嵌套关系的 Action 事件 ...
    }

    return primaryKey;
}
```

#### 3.4.2 更新和删除的事件触发

**更新 (updateOne/updateMany)**：
```typescript
// 写入前
const payloadAfterHooks = await emitter.emitFilter(
    this.eventScope === 'items'
        ? ['items.update', `${this.collection}.items.update`]
        : `${this.eventScope}.update`,
    payload,
    { keys, collection },
    { database: trx, ... },
);

// 写入后（事务提交后）
emitter.emitAction('items.update', { payload, keys, collection }, context);
```

**删除 (deleteOne/deleteMany)**：
```typescript
// 删除前
await emitter.emitFilter(
    this.eventScope === 'items'
        ? ['items.delete', `${this.collection}.items.delete`]
        : `${this.eventScope}.delete`,
    keys,
    { collection },
    { database: trx, ... },
);

// 删除后（事务提交后）
emitter.emitAction('items.delete', { payload: keys, collection }, context);
```

### 3.5 支持的事件类型完整列表

| 事件分类 | 事件名 | 触发时机 | Filter 可用 | Action 可用 |
|----------|--------|----------|-------------|-------------|
| **数据写入** | `items.create` / `{collection}.items.create` | 创建数据前后 | ✅ | ✅ |
| | `items.update` / `{collection}.items.update` | 更新数据前后 | ✅ | ✅ |
| | `items.delete` / `{collection}.items.delete` | 删除数据前后 | ✅ | ✅ |
| **数据读取** | `items.query` / `{collection}.items.query` | 查询执行前 | ✅ | ❌ |
| | `items.read` / `{collection}.items.read` | 查询返回前后 | ✅ | ✅ |
| **认证** | `auth.login` | 登录成功 | ❌ | ✅ |
| | `auth.logout` | 登出 | ❌ | ✅ |
| **系统** | `server.start` | 服务器启动 | ❌ | ❌ (Init) |
| | `server.stop` | 服务器停止 | ❌ | ❌ (Init) |
| | `extensions.load` | 扩展加载完成 | ❌ | ✅ |
| | `extensions.unload` | 扩展卸载 | ❌ | ✅ |
| **响应** | `response` | 响应发送前 | ✅ | ✅ |
| **自定义** | 任意字符串 | 手动触发 | ✅ | ✅ |

---

## 4. 操作节点执行顺序与错误处理

### 4.1 流程执行核心（executeFlow）

```typescript
// 源码位置: api/src/flows.ts:393-504
private async executeFlow(flow: Flow, data: unknown = null, context: Record<string, unknown> = {}): Promise<unknown> {
    // 初始化数据流上下文
    const keyedData: Record<string, unknown> = {
        [TRIGGER_KEY]: data,        // $trigger: 原始触发数据
        [LAST_KEY]: data,            // $last: 上一个操作的结果
        [ACCOUNTABILITY_KEY]: context?.['accountability'] ?? null,
        [ENV_KEY]: this.envs,        // $env: 环境变量
    };

    let nextOperation = flow.operation;  // 起始操作节点
    let lastOperationStatus: 'resolve' | 'reject' | 'unknown' = 'unknown';

    const steps: { operation: string; key: string; status: ... }[] = [];

    // ===== 主循环：顺序执行操作节点 =====
    while (nextOperation !== null) {
        // 执行单个操作节点
        const { successor, data, status, options } = await this.executeOperation(
            nextOperation, 
            keyedData, 
            context
        );

        // 更新数据流
        keyedData[nextOperation.key] = data;  // 按 key 存储
        keyedData[LAST_KEY] = data;            // 更新 $last
        lastOperationStatus = status;
        steps.push({ operation: nextOperation!.id, key: nextOperation.key, status, options });

        // 确定下一个操作节点
        nextOperation = successor;
    }

    // ===== 后置处理：活动记录、错误处理、返回值 =====
    
    // 1. 记录活动日志（如果配置了 accountability）
    if (flow.accountability !== null) {
        const activity = await activityService.createOne({
            action: Action.RUN,
            user: accountability?.user ?? null,
            collection: 'directus_flows',
            item: flow.id,
        });

        // 完整记录：记录所有步骤和数据（带敏感信息脱敏）
        if (flow.accountability === 'all') {
            await revisionsService.createOne({
                activity: activity,
                collection: 'directus_flows',
                item: flow.id,
                data: {
                    steps: steps.map((step) => redactObject(step, ...)),
                    data: redactObject(keyedData, ...),  // 脱敏敏感信息
                },
            });
        }
    }

    // 2. 错误处理：特定情况下抛出错误
    if (
        (flow.trigger === 'manual' || flow.trigger === 'webhook') &&
        flow.options['async'] !== true &&
        flow.options['error_on_reject'] === true &&
        lastOperationStatus === 'reject'
    ) {
        throw keyedData[LAST_KEY];
    }

    // Filter 类型的 Flow 如果 reject，抛出错误中断原流程
    if (flow.trigger === 'event' && flow.options['type'] === 'filter' && lastOperationStatus === 'reject') {
        throw keyedData[LAST_KEY];
    }

    // 3. 确定返回值
    if (flow.options['return'] === '$all') {
        return keyedData;                    // 返回所有数据
    } else if (flow.options['return']) {
        return get(keyedData, flow.options['return']);  // 按路径取值
    }

    return undefined;
}
```

### 4.2 单操作节点执行（executeOperation）

```typescript
// 源码位置: api/src/flows.ts:506-600
private async executeOperation(
    operation: Operation,
    keyedData: Record<string, unknown>,
    context: Record<string, unknown> = {},
): Promise<{
    successor: Operation | null;
    status: 'resolve' | 'reject' | 'unknown';
    data: unknown;
    options: Record<string, any> | null;
}> {
    const logger = useLogger();

    // 1. 检查操作节点是否已注册
    if (!this.operations.has(operation.type)) {
        logger.warn(`Couldn't find operation ${operation.type}`);
        return { successor: null, status: 'unknown', data: null, options: null };
    }

    const handler = this.operations.get(operation.type)!;

    // 2. 准备操作配置：使用模板语法解析变量
    // 例如: options.title = "{{ $trigger.payload.title }}"
    let optionData = keyedData;
    
    // Log 操作特殊处理：脱敏敏感信息
    if (operation.type === 'log') {
        optionData = redactObject(keyedData, { keys: [/* 敏感字段列表 */] }, getRedactedString);
    }

    let options = operation.options;

    try {
        // 3. 解析配置中的模板变量
        options = applyOptionsData(options, optionData);

        // 4. 执行操作 handler
        let result = await handler(options, {
            services,           // 所有服务: ItemsService, UsersService 等
            env: useEnv(),
            database: getDatabase(),
            logger,
            getSchema,
            data: keyedData,
            accountability: null,
            ...context,
        });

        // 5. 验证结果可序列化（必须是 JSON 兼容）
        JSON.stringify(result ?? null);

        // 6. 处理 undefined 值（JSON 不支持）
        if (typeof result === 'object' && result !== null) {
            result = deepMap(result, (value) => (value === undefined ? null : value));
        }

        // ===== Resolve 分支 =====
        return { 
            successor: operation.resolve,   // 下一个操作（成功分支）
            status: 'resolve', 
            data: result ?? null, 
            options 
        };

    } catch (error) {
        // ===== Reject 分支 =====
        let data;

        if (error instanceof Error) {
            delete error.stack;  // 移除堆栈信息，避免暴露内部细节
            data = error;
        } else if (typeof error === 'string') {
            // JSON 字符串尝试解析
            data = isValidJSON(error) ? parseJSON(error) : error;
        } else {
            data = error ?? null;
        }

        return {
            successor: operation.reject,    // 下一个操作（失败分支）
            status: 'reject',
            data,
            options,
        };
    }
}
```

### 4.3 操作节点注册机制

#### 4.3.1 内置操作节点加载

```typescript
// 源码位置: api/src/extensions/manager.ts:868-880
private async registerInternalOperations(): Promise<void> {
    // 读取 operations 目录下所有子目录
    const internalOperations = await readdir(path.join(__dirname, '..', 'operations'));

    for (const operation of internalOperations) {
        // 动态导入每个操作的 index.js
        const operationInstance: OperationApiConfig | { default: OperationApiConfig } = await import(
            `../operations/${operation}/index.js`
        );

        const config = getModuleDefault(operationInstance);

        // 注册到 FlowManager
        this.registerOperation(config);
    }
}
```

#### 4.3.2 单个操作节点注册

```typescript
// 源码位置: api/src/extensions/manager.ts:1007-1017
private registerOperation(config: OperationApiConfig): PromiseCallback {
    const flowManager = getFlowManager();

    // 添加到 FlowManager 的 operations Map
    flowManager.addOperation(config.id, config.handler);

    // 返回注销函数
    const unregisterFunction = () => {
        flowManager.removeOperation(config.id);
    };

    return unregisterFunction;
}
```

### 4.4 内置操作节点列表

| 操作类型 | ID | 功能描述 |
|----------|-----|----------|
| **条件** | `condition` | 根据条件判断走 resolve 或 reject 分支 |
| **创建数据** | `item-create` | 创建新的数据项 |
| **读取数据** | `item-read` | 读取数据项 |
| **更新数据** | `item-update` | 更新数据项 |
| **删除数据** | `item-delete` | 删除数据项 |
| **发送邮件** | `mail` | 发送电子邮件 |
| **发送通知** | `notification` | 发送系统通知 |
| **HTTP 请求** | `request` | 发起 HTTP 请求 |
| **数据转换** | `transform` | 使用 JMESPath 转换数据 |
| **触发器** | `trigger` | 触发另一个流程 |
| **抛出错误** | `throw-error` | 手动抛出错误，进入 reject 分支 |
| **延迟** | `sleep` | 暂停执行指定时间 |
| **日志** | `log` | 输出日志 |
| **JWT** | `json-web-token` | 生成或解析 JWT |
| **执行脚本** | `exec` | 执行系统命令（需配置） |

### 4.5 流程执行流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        流程执行状态机                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────┐                                                               │
│   │  开始    │                                                               │
│   └────┬─────┘                                                               │
│        │                                                                      │
│        ▼                                                                      │
│   ┌──────────────────────┐                                                    │
│   │  执行当前 Operation  │                                                    │
│   │                      │                                                    │
│   │  options = applyOptionsData()  ◄───── 解析模板变量                      │
│   │  result = handler(options)         使用 $trigger, $last 等              │
│   │                      │                                                    │
│   └──────────┬───────────┘                                                    │
│              │                                                                │
│     ┌────────┴────────┐                                                       │
│     │                 │                                                       │
│     ▼                 ▼                                                       │
│  ┌─────────┐      ┌─────────┐                                                 │
│  │ Success │      │  Error  │                                                 │
│  │ Resolve │      │ Reject  │                                                 │
│  └────┬────┘      └────┬────┘                                                 │
│       │                 │                                                       │
│       ▼                 ▼                                                       │
│  ┌──────────────────────────────────┐                                          │
│  │  选择下一个 Operation:            │                                          │
│  │                                  │                                          │
│  │  Resolve 状态: operation.resolve │                                          │
│  │  Reject  状态: operation.reject  │                                          │
│  │                                  │                                          │
│  │  null = 流程结束                  │                                          │
│  └──────────────┬───────────────────┘                                          │
│                 │                                                                │
│          ┌──────┴──────┐                                                        │
│          │             │                                                        │
│          ▼             ▼                                                        │
│    ┌──────────┐   ┌──────────┐                                                 │
│    │ 有后续   │   │ 无后续   │                                                 │
│    │ Operation│   │  结束    │                                                 │
│    └────┬─────┘   └──────────┘                                                 │
│         │                                                                       │
│         └──────────────────────────────► 回到循环开始                          │
│                                                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  特殊的 Condition 操作:                                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  handler({ filter }, { data, accountability }) {                     │  │
│  │      const parsedFilter = parseFilter(filter, accountability, ...);  │  │
│  │      const errors = validatePayload(parsedFilter, data, ...);       │  │
│  │                                                                       │  │
│  │      if (errors.length > 0) {                                        │  │
│  │          throw validationErrors;  // 进入 Reject 分支               │  │
│  │      } else {                                                         │  │
│  │          return null;              // 进入 Resolve 分支              │  │
│  │      }                                                                │  │
│  │  }                                                                    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.6 错误传播机制

#### 4.6.1 操作内部错误处理

```
操作执行过程中的任何异常都会被捕获并转换为 Reject 状态:

┌────────────────────────────────────────────────────────────┐
│  try {                                                       │
│      result = await handler(options, context);              │
│      // 成功 → Resolve 分支                                  │
│      return { successor: operation.resolve, status: 'resolve' };
│  } catch (error) {                                          │
│      // 任何异常 → Reject 分支                               │
│      return { successor: operation.reject, status: 'reject', data: error };
│  }                                                           │
└────────────────────────────────────────────────────────────┘
```

#### 4.6.2 流程级错误处理

流程结束时根据触发类型决定是否抛出错误：

```typescript
// 源码位置: api/src/flows.ts:484-495
if (
    (flow.trigger === 'manual' || flow.trigger === 'webhook') &&
    flow.options['async'] !== true &&
    flow.options['error_on_reject'] === true &&
    lastOperationStatus === 'reject'
) {
    throw keyedData[LAST_KEY];
}

// Filter 类型的事件触发器会中断原流程
if (flow.trigger === 'event' && flow.options['type'] === 'filter' && lastOperationStatus === 'reject') {
    throw keyedData[LAST_KEY];
}
```

**错误传播场景**：

| 触发类型 | 配置 | 最后状态 | 行为 |
|----------|------|----------|------|
| Event (Filter) | - | reject | **抛出错误**，中断原数据操作 |
| Event (Filter) | - | resolve | 正常返回，原操作继续 |
| Event (Action) | - | reject | **静默忽略**，不影响原流程 |
| Manual | `error_on_reject: true` + 同步 | reject | **抛出错误** |
| Manual | `error_on_reject: false` 或异步 | reject | **静默忽略** |
| Webhook | `error_on_reject: true` + 同步 | reject | **抛出错误** |
| Webhook | 异步 | reject | **静默忽略** |
| Operation | - | 任意 | 返回结果给调用方 |
| Schedule | - | 任意 | 错误仅记录日志 |

### 4.7 数据流上下文

流程执行过程中维护一个上下文对象，包含以下特殊键：

| 键 | 说明 | 示例 |
|----|------|------|
| `$trigger` | 原始触发数据 | `{ payload: {...}, collection: 'articles' }` |
| `$last` | 上一个操作的结果 | `{ id: 1, title: 'Hello' }` |
| `$accountability` | 当前用户上下文 | `{ user: 'uuid', role: 'uuid', admin: false }` |
| `$env` | 环境变量（白名单内） | `{ API_URL: '...' }` |
| `{operation.key}` | 每个操作的结果按 key 存储 | `{ get_user: {...}, send_email: {...} }` |

**模板语法示例**：
```javascript
// 在操作配置中使用模板引用上下文
{
    "recipient": "{{ $accountability.email }}",
    "subject": "New article: {{ $trigger.payload.title }}",
    "article_id": "{{ $last.id }}"
}
```

---

## 5. 总结与最佳实践

### 5.1 Filter vs Action 事件选择指南

| 场景 | 推荐类型 | 原因 |
|------|----------|------|
| 数据验证 | Filter | 可以在写入前拒绝无效数据 |
| 数据转换/填充 | Filter | 可以修改 payload 后再写入 |
| 权限二次检查 | Filter | 可以抛出错误阻止操作 |
| 发送通知 | Action | 不阻塞主流程，失败不影响 |
| 异步处理 | Action | 异步执行，不影响响应时间 |
| 操作日志 | Action | 只在操作成功后记录 |
| 触发外部系统 | Action | 解耦，外部系统故障不影响内部 |

### 5.2 关键设计决策

1. **Filter 同步阻塞**：
   - 保证数据一致性
   - 支持事务回滚
   - 可能影响性能（注意 handler 执行时间）

2. **Action 异步非阻塞**：
   - 提升响应速度
   - 错误隔离
   - 不参与事务（注意数据一致性）

3. **操作节点链式执行**：
   - 清晰的执行顺序
   - 内置错误分支（resolve/reject）
   - 灵活的数据流传递

4. **模板化配置**：
   - 无需编码即可实现简单逻辑
   - 上下文变量访问（`$trigger`, `$last` 等）
   - 敏感信息自动脱敏（Log 操作）

### 5.3 性能与可靠性建议

1. **Filter Handler 性能**：
   - 避免耗时操作（如外部 HTTP 调用）
   - 考虑使用 Action + 异步补偿
   - 注意错误处理，避免影响正常业务

2. **Action Handler 可靠性**：
   - 实现幂等性（可能重复执行）
   - 错误重试机制（如需）
   - 考虑使用消息队列解耦

3. **流程设计建议**：
   - 避免过长的流程链
   - 使用 Condition 操作处理分支逻辑
   - 适当使用 Throw Error 操作进行验证
   - 配置 `accountability: all` 以便调试

4. **安全注意事项**：
   - 谨慎使用 Exec 操作
   - 注意 `$env` 白名单配置
   - 敏感信息会自动脱敏，但要注意自定义日志
   - 沙箱隔离的扩展有额外的安全限制

### 5.4 相关文件索引

| 文件路径 | 说明 |
|----------|------|
| `api/src/flows.ts` | 流程管理器核心 |
| `api/src/emitter.ts` | 事件发射器实现 |
| `api/src/services/items.ts` | 数据服务层事件触发点 |
| `api/src/extensions/manager.ts` | 扩展管理器 |
| `api/src/operations/*/index.ts` | 内置操作节点实现 |
| `api/src/extensions/lib/sandbox/register/action.ts` | 沙箱 Action Hook 注册 |
| `api/src/extensions/lib/sandbox/register/filter.ts` | 沙箱 Filter Hook 注册 |
| `api/src/extensions/lib/sandbox/register/operation.ts` | 沙箱操作节点注册 |

---

*报告生成时间：2026-05-02*
*基于 Directus 代码库版本分析*
