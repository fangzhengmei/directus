# Directus 扩展生命周期分析

本文档详细分析 Directus 扩展系统的加载、注册、隔离机制，以及后台 UI 如何发现和配置扩展。

## 目录

1. [扩展类型概述](#1-扩展类型概述)
2. [扩展加载流程](#2-扩展加载流程)
3. [扩展注册机制](#3-扩展注册机制)
4. [扩展隔离机制](#4-扩展隔离机制)
5. [前端扩展发现与配置](#5-前端扩展发现与配置)
6. [扩展生命周期事件](#6-扩展生命周期事件)
7. [扩展配置完整链路](#7-扩展配置完整链路)
8. [失败与异常处理](#8-失败与异常处理)
9. [关键代码位置](#9-关键代码位置)

---

## 1. 扩展类型概述

Directus 支持多种类型的扩展，按运行环境分为：

### 1.1 API 扩展（仅后端运行）

| 类型 | 说明 | 核心功能 |
|------|------|----------|
| **endpoint** | 自定义端点 | 注册 Express 路由，提供自定义 API |
| **hook** | 钩子扩展 | 监听系统事件（filter/action/init/schedule） |
| **operation** | 工作流操作 | 为 Flow 系统提供自定义操作节点 |

### 1.2 App 扩展（仅前端运行）

| 类型 | 说明 | 核心功能 |
|------|------|----------|
| **interface** | 字段界面 | 自定义字段输入组件 |
| **display** | 字段显示 | 自定义字段值渲染组件 |
| **layout** | 数据布局 | 自定义列表/卡片等数据展示方式 |
| **module** | 功能模块 | 完整的后台页面模块（带路由） |
| **panel** | 仪表盘面板 | 自定义仪表盘组件 |
| **theme** | 主题 | 自定义 UI 主题样式 |

### 1.3 混合扩展（Hybrid）

**operation** 是唯一的混合扩展类型，同时包含：
- API 部分：后端执行逻辑
- App 部分：前端配置界面

### 1.4 Bundle 扩展

Bundle 是一种特殊的扩展类型，可以包含多个子扩展：
- 多个 hook
- 多个 endpoint
- 多个 operation
- 任何 App 扩展类型

Bundle 扩展支持 `partial` 模式，允许单独启用/禁用其中的子扩展。

### 1.5 扩展来源

扩展有三个来源，由 `ExtensionManager` 统一管理：

```typescript
// api/src/extensions/manager.ts:87-93
private localExtensions: Map<string, Extension> = new Map();    // 本地文件系统
private registryExtensions: Map<string, Extension> = new Map(); // 市场安装
private moduleExtensions: Map<string, Extension> = new Map();   // npm 依赖
```

| 来源 | 路径/方式 | 说明 |
|------|-----------|------|
| **local** | `extensions/` 目录 | 开发环境常用，直接放置扩展目录 |
| **registry** | `extensions/.registry/` | 通过市场安装的扩展 |
| **module** | `package.json` 依赖 | 作为 npm 包安装的扩展 |

---

## 2. 扩展加载流程

### 2.1 初始化入口

扩展加载从 `ExtensionManager.initialize()` 开始，在 API 启动时调用：

```typescript
// api/src/extensions/manager.ts:189-226
public async initialize(options: Partial<ExtensionManagerOptions> = {}): Promise<void> {
    // 1. 配置选项合并
    this.options = { ...defaultOptions, ...options };
    
    // 2. 初始化文件监视器（开发模式）
    if (this.options.watch && !wasWatcherInitialized) {
        this.initializeWatcher();
    }
    
    // 3. 首次加载扩展
    if (!this.isLoaded) {
        await this.load({ forceSync: true });
        logger.info(`Loaded extensions: ${this.extensions.map((ext) => ext.name).join(', ')}`);
    }
    
    // 4. 订阅跨进程重载消息
    this.messenger.subscribe(this.reloadChannel, (payload) => {
        // 忽略自身触发的消息
        if (payload.origin === this.processId) return;
        this.reload(options);
    });
}
```

### 2.2 核心加载流程

`load()` 方法是扩展加载的核心：

```typescript
// api/src/extensions/manager.ts:276-314
private async load(options?: ExtensionSyncOptions): Promise<void> {
    const logger = useLogger();

    // 1. 同步扩展（从 EXTENSIONS_LOCATION 复制）
    if (env['EXTENSIONS_LOCATION']) {
        await syncExtensions(options);
    }

    try {
        // 2. 从磁盘发现所有扩展
        const { local, registry, module } = await getExtensions();
        this.localExtensions = local;
        this.registryExtensions = registry;
        this.moduleExtensions = module;

        // 3. 获取扩展设置（从数据库）
        this.extensionsSettings = await getExtensionsSettings({ local, registry, module });
    } catch (error) {
        this.handleExtensionError({ error, reason: `Couldn't load extensions` });
    }

    // 4. 并行注册内部操作和 API 扩展
    await Promise.all([this.registerInternalOperations(), this.registerApiExtensions()]);

    // 5. 生成 App 扩展 bundle（如果启用了 SERVE_APP）
    if (env['SERVE_APP']) {
        await this.generateExtensionBundle();
    }

    this.isLoaded = true;

    // 6. 触发扩展加载事件
    emitter.emitAction('extensions.load', { extensions: this.extensions });
    logger.info('Extensions loaded');
}
```

### 2.3 扩展发现机制

扩展发现由 `getExtensions()` 函数处理：

```typescript
// api/src/extensions/lib/get-extensions.ts:6-16
export const getExtensions = async () => {
    const env = useEnv();

    // 1. 本地扩展：从 extensions/ 目录读取
    const localExtensions = await resolveFsExtensions(getExtensionsPath());
    
    // 2. 注册表扩展：从 extensions/.registry/ 目录读取
    const registryExtensions = await resolveFsExtensions(join(getExtensionsPath(), '.registry'));
    
    // 3. 模块扩展：从 package.json 依赖中解析
    const moduleExtensions = await resolveModuleExtensions(env['PACKAGE_FILE_LOCATION'] as string);

    return { local: localExtensions, registry: registryExtensions, module: moduleExtensions };
};
```

**扩展发现规则**：
- 通过 `package.json` 中的 `directus:extension` 字段识别
- 字段包含：`type`、`entrypoint`（或 `entrypoint.app/api` 用于混合扩展）
- Bundle 扩展额外包含 `entries` 数组

### 2.4 热重载机制

开发模式下支持文件变化自动重载：

```typescript
// api/src/extensions/manager.ts:474-526
private initializeWatcher(): void {
    this.watcher = chokidar.watch([path.resolve('package.json'), extensionDirPath], {
        ignoreInitial: true,
        depth: 1,
        // 忽略 node_modules，只监视特定文件
        ignored: (val: string, stats?: Stats) => {
            if (val.includes('node_modules')) return true;
            // 允许 package.json 和 dist 入口文件
            if (val.endsWith('package.json')) return false;
            if (this.isWatchedExtensionPath(val)) return false;
            return true;
        },
    });

    // 防抖处理，避免频繁重载
    this.watcher
        .on('add', debounce(() => this.reload(), 500))
        .on('change', debounce(() => this.reload(), 650))
        .on('unlink', debounce(() => this.reload(), 2000));
}
```

**重载流程**：
1. 先调用 `unload()` 卸载所有扩展
2. 再调用 `load()` 重新加载
3. 触发 `extensions.reload` 事件
4. 通过消息总线通知其他进程

---

## 3. 扩展注册机制

### 3.1 API 扩展注册

`registerApiExtensions()` 处理所有 API 类型扩展的注册：

```typescript
// api/src/extensions/manager.ts:676-713
private async registerApiExtensions(): Promise<void> {
    const sources = {
        module: this.moduleExtensions,
        registry: this.registryExtensions,
        local: this.localExtensions,
    } as const;

    await Promise.all(
        Object.entries(sources).map(async ([source, extensions]) => {
            await Promise.all(
                Array.from(extensions.entries()).map(async ([folder, extension]) => {
                    // 检查扩展是否启用
                    const { id, enabled } = this.extensionsSettings.find(
                        (settings) => settings.source === source && settings.folder === folder,
                    ) ?? { enabled: false };

                    if (!enabled) return;

                    // 根据类型注册
                    switch (extension.type) {
                        case 'hook':
                            await this.registerHookExtension(extension);
                            break;
                        case 'endpoint':
                            await this.registerEndpointExtension(extension);
                            break;
                        case 'operation':
                            await this.registerOperationExtension(extension);
                            break;
                        case 'bundle':
                            await this.registerBundleExtension(extension, source, id);
                            break;
                    }
                }),
            );
        }),
    );
}
```

### 3.2 Hook 扩展注册

Hook 扩展通过 `registerHook()` 方法注册：

```typescript
// api/src/extensions/manager.ts:885-972
private registerHook(hookRegistrationCallback: HookConfig, name: string): PromiseCallback[] {
    const logger = useLogger();
    const unregisterFunctions: PromiseCallback[] = [];

    // 创建 Hook 注册上下文
    const hookRegistrationContext = {
        // Filter 钩子：修改数据
        filter: <T = unknown>(event: string, handler: FilterHandler<T>) => {
            emitter.onFilter(event, handler);
            unregisterFunctions.push(() => emitter.offFilter(event, handler));
        },
        // Action 钩子：响应事件
        action: (event: string, handler: ActionHandler) => {
            emitter.onAction(event, handler);
            unregisterFunctions.push(() => emitter.offAction(event, handler));
        },
        // Init 钩子：系统初始化时执行
        init: (event: string, handler: InitHandler) => {
            emitter.onInit(event, handler);
            unregisterFunctions.push(() => emitter.offInit(name, handler));
        },
        // Schedule 钩子：定时任务
        schedule: (cron: string, handler: ScheduleHandler) => {
            if (validateCron(cron)) {
                const job = scheduleSynchronizedJob(`${name}:${scheduleIndex}`, cron, async () => {
                    if (this.options.schedule) await handler();
                });
                unregisterFunctions.push(async () => await job.stop());
            }
        },
        // Embed 钩子：注入 HTML 到前端
        embed: (position: 'head' | 'body', code: string | EmbedHandler) => {
            const content = typeof code === 'function' ? code() : code;
            if (position === 'head') {
                const index = this.hookEmbedsHead.length;
                this.hookEmbedsHead.push(content);
                unregisterFunctions.push(() => this.hookEmbedsHead.splice(index, 1));
            } else {
                const index = this.hookEmbedsBody.length;
                this.hookEmbedsBody.push(content);
                unregisterFunctions.push(() => this.hookEmbedsBody.splice(index, 1));
            }
        },
    };

    // 执行扩展的注册回调
    hookRegistrationCallback(hookRegistrationContext, {
        services,
        env,
        database: getDatabase(),
        emitter: this.localEmitter,  // 隔离的事件发射器
        logger,
        getSchema,
    });

    return unregisterFunctions;
}
```

### 3.3 Endpoint 扩展注册

Endpoint 扩展通过 `registerEndpoint()` 方法注册：

```typescript
// api/src/extensions/manager.ts:977-1002
private registerEndpoint(config: EndpointConfig, name: string): PromiseCallback {
    const logger = useLogger();

    // 解析配置
    const endpointRegistrationCallback = typeof config === 'function' ? config : config.handler;
    const nameWithoutType = name.includes(':') ? name.split(':')[0] : name;
    const routeName = typeof config === 'function' ? nameWithoutType : config.id;

    // 创建隔离的 Router
    const scopedRouter = express.Router();
    this.endpointRouter.use(`/${routeName}`, scopedRouter);

    // 执行扩展的路由注册
    endpointRegistrationCallback(scopedRouter, {
        services,
        env,
        database: getDatabase(),
        emitter: this.localEmitter,
        logger,
        getSchema,
    });

    // 返回注销函数
    const unregisterFunction = () => {
        this.endpointRouter.stack = this.endpointRouter.stack.filter((layer) => scopedRouter !== layer.handle);
    };

    return unregisterFunction;
}
```

**Endpoint 路由结构**：
- 每个扩展获得一个独立的 `scopedRouter`
- 挂载到全局 `endpointRouter` 的 `/${routeName}` 路径
- 注销时通过引用比较精确移除

### 3.4 Operation 扩展注册

Operation 扩展注册到 Flow Manager：

```typescript
// api/src/extensions/manager.ts:1007-1017
private registerOperation(config: OperationApiConfig): PromiseCallback {
    const flowManager = getFlowManager();
    flowManager.addOperation(config.id, config.handler);
    
    const unregisterFunction = () => {
        flowManager.removeOperation(config.id);
    };
    
    return unregisterFunction;
}
```

### 3.5 Bundle 扩展注册

Bundle 扩展需要特殊处理，遍历其包含的所有子扩展：

```typescript
// api/src/extensions/manager.ts:801-862
private async registerBundleExtension(
    bundle: BundleExtension,
    source: 'local' | 'registry' | 'module',
    bundleId: string,
) {
    // 检查子扩展是否启用的辅助函数
    const extensionEnabled = (extensionName: string) => {
        const settings = this.extensionsSettings.find(
            (settings) => settings.source === source && 
                         settings.folder === extensionName && 
                         settings.bundle === bundleId,
        );
        return settings?.enabled ?? false;
    };

    const bundlePath = path.resolve(bundle.path, bundle.entrypoint.api);
    const bundleInstances = await importFileUrl(bundlePath, import.meta.url, { fresh: true });
    const configs = getModuleDefault(bundleInstances);

    const unregisterFunctions: PromiseCallback[] = [];

    // 注册所有 hooks
    for (const { config, name } of configs.hooks) {
        if (!extensionEnabled(name)) continue;
        const unregisters = this.registerHook(config, name);
        unregisterFunctions.push(...unregisters);
    }

    // 注册所有 endpoints
    for (const { config, name } of configs.endpoints) {
        if (!extensionEnabled(name)) continue;
        const unregister = this.registerEndpoint(config, name);
        unregisterFunctions.push(unregister);
    }

    // 注册所有 operations
    for (const { config, name } of configs.operations) {
        if (!extensionEnabled(name)) continue;
        const unregister = this.registerOperation(config);
        unregisterFunctions.push(unregister);
    }

    // 保存注销函数
    this.unregisterFunctionMap.set(bundle.name, async () => {
        await Promise.all(unregisterFunctions.map((fn) => fn()));
        deleteFromRequireCache(bundlePath);
    });
}
```

---

## 4. 扩展隔离机制

Directus 提供两种运行模式：**沙箱模式**和**本地模式**。

### 4.1 沙箱模式（Sandbox）

使用 `isolated-vm` 库创建完全隔离的 JavaScript 执行环境：

```typescript
// api/src/extensions/manager.ts:616-674
private async registerSandboxedApiExtension(extension: ApiExtension | HybridExtension) {
    const logger = useLogger();

    // 沙箱配置
    const sandboxMemory = Number(env['EXTENSIONS_SANDBOX_MEMORY']);
    const sandboxTimeout = Number(env['EXTENSIONS_SANDBOX_TIMEOUT']);

    // 1. 读取扩展代码
    const entrypointPath = path.resolve(extension.path, /* ... */);
    const extensionCode = await readFile(entrypointPath, 'utf-8');

    // 2. 创建隔离环境
    const isolate = new ivm.Isolate({
        memoryLimit: sandboxMemory,  // 内存限制
        onCatastrophicError: (error) => {
            logger.error(`Error in API extension sandbox of ${extension.type} "${extension.name}"`);
            process.abort();  // 严重错误时终止进程
        },
    });

    // 3. 创建上下文
    const context = await isolate.createContext();
    context.global.setSync('process', { env: { NODE_ENV: process.env['NODE_ENV'] ?? 'production' } }, { copy: true });

    // 4. 编译并实例化模块
    const module = await isolate.compileModule(extensionCode, { filename: `file://${entrypointPath}` });

    // 5. 实例化沙箱 SDK（唯一允许的导入）
    const sdkModule = await instantiateSandboxSdk(isolate, extension.sandbox?.requestedScopes ?? {});

    // 6. 链接模块依赖
    await module.instantiate(context, (specifier) => {
        if (specifier !== 'directus:api') {
            throw new Error('Imports other than "directus:api" are prohibited in API extension sandboxes');
        }
        return sdkModule;
    });

    // 7. 执行模块
    await module.evaluate({ timeout: sandboxTimeout });

    // 8. 获取默认导出并执行注册
    const cb = await module.namespace.get('default', { reference: true });
    const { code, hostFunctions, unregisterFunction } = generateApiExtensionsSandboxEntrypoint(
        extension.type,
        extension.name,
        this.endpointRouter,
    );

    // 在上下文中执行注册代码
    await context.evalClosure(code, [cb, ...hostFunctions.map((fn) => new ivm.Reference(fn))], {
        timeout: sandboxTimeout,
        filename: '<extensions-sandbox>',
    });

    // 9. 保存注销函数
    this.unregisterFunctionMap.set(extension.name, async () => {
        await unregisterFunction();
        if (!isolate.isDisposed) isolate.dispose();  // 释放隔离环境
    });
}
```

**沙箱隔离特性**：

| 特性 | 说明 |
|------|------|
| **内存限制** | 通过 `EXTENSIONS_SANDBOX_MEMORY` 配置，防止内存泄漏 |
| **执行超时** | 通过 `EXTENSIONS_SANDBOX_TIMEOUT` 配置，防止无限循环 |
| **受限导入** | 只允许导入 `directus:api`，禁止访问 Node.js 内置模块 |
| **受限环境** | `process.env` 只暴露 `NODE_ENV`，不暴露系统环境变量 |
| **请求作用域** | 可通过 `requestedScopes` 控制扩展能访问的 API 范围 |

### 4.2 本地模式（Non-Sandbox）

不使用沙箱，直接在主进程中执行：

```typescript
// api/src/extensions/manager.ts:715-739
private async registerHookExtension(hook: ApiExtension) {
    try {
        if (hook.sandbox?.enabled) {
            await this.registerSandboxedApiExtension(hook);
        } else {
            // 本地模式：直接导入
            const hookPath = path.resolve(hook.path, hook.entrypoint);
            const hookInstance = await importFileUrl(hookPath, import.meta.url, { fresh: true });
            const config = getModuleDefault(hookInstance);
            
            const unregisterFunctions = this.registerHook(config, hook.name);
            
            this.unregisterFunctionMap.set(hook.name, async () => {
                await Promise.all(unregisterFunctions.map((fn) => fn()));
                deleteFromRequireCache(hookPath);  // 清理缓存
            });
        }
    } catch (error) {
        this.handleExtensionError({ error, reason: `Couldn't register hook "${hook.name}"` });
    }
}
```

**本地模式特点**：
- 完全访问 Node.js API 和 Directus 内部模块
- 无内存和超时限制
- 重载时需要手动清理 require 缓存
- 适合开发环境和受信任的扩展

### 4.3 沙箱路由注册

沙箱中的 endpoint 注册通过桥接函数实现：

```typescript
// api/src/extensions/lib/sandbox/register/route.ts:8-65
export function registerRouteGenerator(endpointName: string, endpointRouter: Router) {
    const router = express.Router();
    endpointRouter.use(`/${endpointName}`, router);

    // 供沙箱调用的注册函数
    const registerRoute = (
        path: Reference<string>,
        method: Reference<'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE'>,
        cb: Reference<(req: {...}) => {...}>,
    ) => {
        // 类型检查
        if (path.typeof !== 'string') throw new TypeError('Route path has to be of type string');
        if (method.typeof !== 'string') throw new TypeError('Route method has to be of type string');
        if (cb.typeof !== 'function') throw new TypeError('Route handler has to be of type function');

        const pathCopied = path.copySync();
        const methodCopied = method.copySync();

        // 创建桥接 handler
        const handler: RequestHandler = asyncHandler(async (req, res) => {
            // 1. 将请求数据复制到沙箱
            const request = { url: req.url, headers: req.headers, body: req.body };
            
            // 2. 调用沙箱中的 handler
            const response = await callReference(cb, [request]);
            
            // 3. 将响应从沙箱复制出来
            const responseCopied = await response.copy();
            
            // 4. 发送响应
            res.status(responseCopied.status).send(responseCopied.body);
        });

        // 注册到主 Router
        switch (methodCopied) {
            case 'GET': router.get(pathCopied, handler); break;
            // ... 其他方法
        }
    };

    // 注销函数
    const unregisterFunction = () => {
        endpointRouter.stack = endpointRouter.stack.filter((layer) => router !== layer.handle);
    };

    return { register: registerRoute, unregisterFunction };
}
```

### 4.4 事件隔离

扩展使用独立的 `localEmitter`，与系统核心事件隔离：

```typescript
// api/src/extensions/manager.ts:116
private localEmitter: Emitter = new Emitter();

// 注册 hook 时传入隔离的 emitter
hookRegistrationCallback(hookRegistrationContext, {
    // ...
    emitter: this.localEmitter,  // 不是全局的 emitter
    // ...
});
```

**隔离效果**：
- 扩展之间可以通过 `localEmitter` 通信
- 不会影响核心系统的事件
- 卸载时一次性清除所有监听器

### 4.5 路由隔离

Endpoint 扩展使用独立的 `endpointRouter`：

```typescript
// api/src/extensions/manager.ts:122
private endpointRouter: Router = Router();

// 提供给主应用使用
public getEndpointRouter(): Router {
    return this.endpointRouter;
}
```

**隔离效果**：
- 扩展的路由不会污染全局 Express 应用
- 卸载时可以精确移除特定扩展的路由
- 支持热重载而不影响其他路由

---

## 5. 前端扩展发现与配置

### 5.1 App 扩展 Bundle 生成

后端在加载时为前端生成统一的扩展 bundle：

```typescript
// api/src/extensions/manager.ts:569-614
private async generateExtensionBundle(): Promise<void> {
    const logger = useLogger();
    const env = useEnv();

    // 1. 获取共享依赖映射（Vue、Pinia 等）
    const sharedDepsMapping = await getSharedDepsMapping(APP_SHARED_DEPS);
    const internalImports = Object.entries(sharedDepsMapping).map(([name, path]) => ({
        find: name,
        replacement: path,
    }));

    // 2. 生成入口代码
    const entrypoint = generateExtensionsEntrypoint(
        { module: this.moduleExtensions, registry: this.registryExtensions, local: this.localExtensions },
        this.extensionsSettings,
    );

    try {
        // 3. 使用 Rollup/Rolldown 打包
        const rollDirection = (env['EXTENSIONS_ROLLDOWN'] ?? false) ? rolldown : rollup;
        
        const bundle = await rollDirection({
            input: 'entry',
            external: Object.values(sharedDepsMapping),  // 共享依赖不打包
            plugins: [
                virtual({ entry: entrypoint }),  // 虚拟入口模块
                alias({ entries: internalImports }),
                nodeResolve({ browser: true }),
            ],
        });

        // 4. 写入临时目录
        const tempDir = join(env['TEMP_PATH'] as string, 'app-extensions');
        const { output } = await bundle.write({ format: 'es', dir: tempDir });

        // 5. 记录生成的 chunk 文件名
        this.appExtensionChunks = output.reduce<string[]>((acc, chunk) => {
            if (chunk.type === 'chunk') acc.push(chunk.fileName);
            return acc;
        }, []);

        await bundle.close();
    } catch (error) {
        logger.warn(`Couldn't bundle App extensions`);
        logger.warn(error);
    }
}
```

### 5.2 入口代码生成

`generateExtensionsEntrypoint` 生成动态导入的入口代码：

```typescript
// packages/extensions/src/node/utils/generate-extensions-entrypoint.ts:12-93
export function generateExtensionsEntrypoint(
    extensionMaps: { local: Map<string, Extension>; registry: Map<string, Extension>; module: Map<string, Extension> },
    settings: ExtensionSettings[],
): string {
    const appOrHybridExtensions: (AppExtension | HybridExtension)[] = [];
    const bundleExtensions: BundleExtension[] = [];

    // 1. 过滤已启用的扩展
    for (const [source, extensions] of Object.entries(extensionMaps)) {
        for (const [folder, extension] of extensions.entries()) {
            const settingsForExtension = settings.find(
                (setting) => setting.source === source && setting.folder === folder,
            );
            if (!settingsForExtension) continue;

            // 收集 App/混合扩展
            if (isIn(extension.type, [...APP_EXTENSION_TYPES, ...HYBRID_EXTENSION_TYPES]) && settingsForExtension.enabled) {
                appOrHybridExtensions.push(extension as AppExtension | HybridExtension);
            }

            // 收集 Bundle 扩展
            if (extension.type === 'bundle') {
                const appBundle: BundleExtension = {
                    ...extension,
                    entries: extension.entries.filter((entry) => {
                        const isApp = isIn(entry.type, [...APP_EXTENSION_TYPES, ...HYBRID_EXTENSION_TYPES]);
                        if (isApp === false) return false;
                        
                        const enabled = settings.find(
                            (setting) => setting.source === source &&
                                         setting.folder === entry.name &&
                                         setting.bundle === settingsForExtension.id,
                        )?.enabled ?? false;
                        return enabled;
                    }),
                };
                if (appBundle.entries.length > 0) bundleExtensions.push(appBundle);
            }
        }
    }

    // 2. 生成 import 语句
    const appOrHybridExtensionImports = [...APP_EXTENSION_TYPES, ...HYBRID_EXTENSION_TYPES].flatMap((type) =>
        appOrHybridExtensions
            .filter((extension) => extension.type === type)
            .map((extension, i) =>
                `import ${type}${i} from './${pathToRelativeUrl(
                    path.resolve(extension.path, /* entrypoint */),
                )}';`,
            ),
    );

    const bundleExtensionImports = bundleExtensions.map((extension, i) =>
        extension.entries.length > 0
            ? `import {${[...APP_EXTENSION_TYPES, ...HYBRID_EXTENSION_TYPES]
                    .filter((type) => extension.entries.some((entry) => entry.type === type))
                    .map((type) => `${pluralize(type)} as ${type}Bundle${i}`)
                    .join(',')}} from './${pathToRelativeUrl(path.resolve(extension.path, extension.entrypoint.app))}';`
            : '',
    );

    // 3. 生成 export 语句
    const extensionExports = [...APP_EXTENSION_TYPES, ...HYBRID_EXTENSION_TYPES].map(
        (type) =>
            `export const ${pluralize(type)} = [${appOrHybridExtensions
                .filter((extension) => extension.type === type)
                .map((_, i) => `${type}${i}`)
                .concat(
                    bundleExtensions
                        .map((extension, i) =>
                            extension.entries.some((entry) => entry.type === type) ? `...${type}Bundle${i}` : null,
                        )
                        .filter((e): e is string => e !== null),
                )
                .join(',')}];`,
    );

    // 4. 合并为最终代码
    return `${appOrHybridExtensionImports.join('')}${bundleExtensionImports.join('')}${extensionExports.join('')}`;
}
```

**生成的代码示例**：
```javascript
import interface0 from './extensions/my-interface/dist/app.js';
import module0 from './extensions/my-module/dist/app.js';
import {interfaces as interfacesBundle0, modules as modulesBundle0} from './extensions/my-bundle/dist/app.js';
export const interfaces = [interface0,...interfacesBundle0];
export const modules = [module0,...modulesBundle0];
// ... 其他类型
```

### 5.3 前端加载扩展

前端通过 `loadExtensions()` 函数加载扩展 bundle：

```typescript
// app/src/extensions.ts:31-42
export async function loadExtensions(): Promise<void> {
    try {
        customExtensions = import.meta.env.DEV
            ? await import(/* @vite-ignore */ '@directus-extensions')           // 开发环境：Vite 插件
            : await import(/* @vite-ignore */ `${getRootPath()}extensions/sources/index.js`); // 生产环境
    } catch (err: any) {
        console.warn(`Couldn't load extensions`);
        console.warn(err);
    }
}
```

**加载路径**：
- 开发环境：`@directus-extensions`（Vite 虚拟模块）
- 生产环境：`/extensions/sources/index.js`（后端提供）

### 5.4 前端注册扩展

`registerExtensions()` 将扩展注册到 Vue 应用：

```typescript
// app/src/extensions.ts:44-95
export function registerExtensions(app: App): void {
    // 1. 获取内部扩展
    const interfaces = getInternalInterfaces();
    const displays = getInternalDisplays();
    const layouts = getInternalLayouts();
    const modules = getInternalModules();
    const panels = getInternalPanels();
    const operations = getInternalOperations();
    const themes = [];

    // 2. 合并自定义扩展
    if (customExtensions !== null) {
        interfaces.push(...customExtensions.interfaces);
        displays.push(...customExtensions.displays);
        layouts.push(...customExtensions.layouts);
        modules.push(...customExtensions.modules);
        panels.push(...customExtensions.panels);
        operations.push(...customExtensions.operations);
        themes.push(...customExtensions.themes);
    }

    // 3. 注册各类扩展
    registerInterfaces(interfaces, app);
    registerDisplays(displays, app);
    registerLayouts(layouts, app);
    registerPanels(panels, app);
    registerOperations(operations, app);
    registerThemes(themes);

    // 4. 监听语言变化，更新翻译
    watch(
        i18n.global.locale,
        () => {
            extensions.interfaces.value = translate(interfaces);
            extensions.displays.value = translate(displays);
            // ...
        },
        { immediate: true },
    );

    // 5. 注册模块（特殊处理，需要动态路由）
    const { registeredModules, onHydrateModules, onDehydrateModules } = registerModules(modules);
    
    watch(
        [i18n.global.locale, registeredModules],
        () => {
            extensions.modules.value = translate(registeredModules.value);
            allModules.value = translate(modules);
        },
        { immediate: true },
    );

    onHydrateCallbacks.push(onHydrateModules);
    onDehydrateCallbacks.push(onDehydrateModules);
}
```

### 5.5 Interface 注册机制

Interface 扩展注册为 Vue 组件：

```typescript
// app/src/interfaces/index.ts:14-22
export function registerInterfaces(interfaces: InterfaceConfig[], app: App): void {
    for (const inter of interfaces) {
        // 注册主组件：interface-{id}
        app.component(`interface-${inter.id}`, inter.component);

        // 注册选项组件（如果有）
        if (typeof inter.options !== 'function' && Array.isArray(inter.options) === false && inter.options !== null) {
            app.component(`interface-options-${inter.id}`, inter.options);
        }
    }
}
```

**使用方式**：
```vue
<!-- 动态组件，根据字段类型选择 -->
<component :is="`interface-${field.interface}`" :field="field" />
```

### 5.6 Module 注册机制

Module 扩展需要动态添加路由：

```typescript
// app/src/modules/index.ts:15-61
export function registerModules(modules: ModuleConfig[]): {
    registeredModules: ShallowRef<ModuleConfig[]>;
    onHydrateModules: () => Promise<void>;
    onDehydrateModules: () => Promise<void>;
} {
    const registeredModules = shallowRef<ModuleConfig[]>([]);

    // 用户登录后执行
    const onHydrateModules = async () => {
        const userStore = useUserStore();
        const permissionsStore = usePermissionsStore();

        if (!userStore.currentUser) return;

        // 1. 权限检查
        registeredModules.value = (
            await Promise.all(
                modules.map(async (module) => {
                    if (!module.preRegisterCheck) return module;
                    
                    const allowed = await module.preRegisterCheck(
                        userStore.currentUser, 
                        permissionsStore.permissions,
                    );
                    
                    if (allowed) return module;
                    return null;
                }),
            )
        ).filter((module): module is ModuleConfig => module !== null);

        // 2. 动态添加路由
        for (const module of registeredModules.value) {
            router.addRoute({
                name: module.id,
                path: `/${module.id}`,
                component: RouterPass,
                children: module.routes,
            });
        }
    };

    // 用户登出时执行
    const onDehydrateModules = async () => {
        for (const module of modules) {
            router.removeRoute(module.id);
        }
        registeredModules.value = [];
    };

    return { registeredModules, onHydrateModules, onDehydrateModules };
}
```

**Module 生命周期**：
1. **Hydrate**：用户登录后，检查权限并添加路由
2. **Dehydrate**：用户登出后，移除路由

### 5.7 UI 扩展管理界面

后台通过 `/settings/extensions` 页面管理扩展：

```vue
<!-- app/src/modules/settings/routes/extensions/extensions.vue -->
<script setup lang="ts">
const extensionsStore = useExtensionsStore();
const { extensions, loading } = storeToRefs(extensionsStore);

// 区分 bundle 和普通扩展
const bundled = computed(() => extensionsStore.extensions.filter(({ bundle }) => bundle !== null));
const regular = computed(() => extensionsStore.extensions.filter(({ bundle }) => bundle === null));

// 按类型分组
const extensionsByType = computed(() => {
    const groups = groupBy(regular.value, 'schema.type');
    // ...
    return groups as ExtensionsMap;
});
</script>

<template>
    <div v-for="(list, type) in extensionsByType" :key="`${type}-list`">
        <ExtensionGroupDivider :type="type" />
        <VList>
            <template v-for="extension in list" :key="extension.name">
                <ExtensionItem
                    :extension
                    :children="extension.schema?.type === 'bundle' 
                        ? bundled.filter(({ bundle }) => bundle === extension.id) 
                        : []"
                />
            </template>
        </VList>
    </div>
</template>
```

### 5.8 扩展 Store

前端通过 Store 管理扩展状态：

```typescript
// 从 API 获取扩展列表
// GET /extensions
```

**API 端点**：
- `GET /extensions` - 获取已安装扩展列表
- `GET /extensions/registry` - 搜索市场扩展
- `POST /extensions/registry/install` - 安装扩展
- `DELETE /extensions/registry/uninstall/:id` - 卸载扩展
- `PATCH /extensions/:id` - 更新扩展设置（启用/禁用）
- `GET /extensions/sources/:chunk` - 获取扩展 bundle 文件

---

## 6. 扩展生命周期事件

### 6.1 核心事件

扩展系统触发以下事件：

| 事件名 | 触发时机 | 数据 |
|--------|----------|------|
| `extensions.load` | 扩展首次加载完成 | `{ extensions }` |
| `extensions.unload` | 扩展卸载完成 | `{ extensions }` |
| `extensions.reload` | 扩展重载完成 | `{ extensions, added, removed }` |
| `extensions.installed` | 新扩展安装完成 | `{ extensions, versionId }` |
| `extensions.uninstalled` | 扩展卸载完成 | `{ extensions, folder }` |

### 6.2 事件使用示例

```typescript
// 在其他服务中监听扩展变化
import emitter from '../emitter.js';

emitter.onAction('extensions.load', ({ extensions }) => {
    console.log('Extensions loaded:', extensions.map(e => e.name));
});

emitter.onAction('extensions.reload', ({ added, removed }) => {
    if (added.length > 0) console.log('Added:', added);
    if (removed.length > 0) console.log('Removed:', removed);
});
```

### 6.3 重载流程

```
┌─────────────────────────────────────────────────────────────┐
│                      扩展重载流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 文件变化检测                                              │
│     ┌─────────────┐                                          │
│     │ chokidar    │ ──debounce──► reload()                  │
│     │ 监视文件变化 │                                          │
│     └─────────────┘                                          │
│                                                              │
│  2. 执行重载                                                  │
│     ┌────────────────────────────────────────────────────┐  │
│     │ reloadQueue (concurrency: 1)                        │  │
│     │ 确保重载串行执行，避免竞态条件                        │  │
│     └────────────────────────────────────────────────────┘  │
│                                                              │
│  3. 卸载旧扩展                                                │
│     ┌─────────────┐                                          │
│     │ unload()    │ ──► unregisterApiExtensions()          │
│     │             │     执行所有 unregisterFunction         │
│     │             │     清理 localEmitter                   │
│     └─────────────┘                                          │
│                                                              │
│  4. 加载新扩展                                                │
│     ┌─────────────┐                                          │
│     │ load()      │ ──► 重新发现、注册、打包                 │
│     └─────────────┘                                          │
│                                                              │
│  5. 通知其他进程                                              │
│     ┌─────────────┐                                          │
│     │ messenger   │ ──► publish('extensions.reload')        │
│     │ 消息总线    │     多进程环境下同步                     │
│     └─────────────┘                                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 关键代码位置

### 7.1 后端核心文件

| 文件路径 | 说明 |
|----------|------|
| `api/src/extensions/manager.ts` | 扩展管理器核心实现 |
| `api/src/extensions/index.ts` | 扩展管理器单例导出 |
| `api/src/extensions/lib/get-extensions.ts` | 扩展发现逻辑 |
| `api/src/extensions/lib/sandbox/` | 沙箱模式实现 |
| `api/src/extensions/lib/installation/manager.ts` | 扩展安装管理器 |
| `api/src/services/extensions.ts` | 扩展 API 服务 |
| `api/src/controllers/extensions.ts` | 扩展 HTTP 端点 |

### 7.2 前端核心文件

| 文件路径 | 说明 |
|----------|------|
| `app/src/extensions.ts` | 前端扩展加载与注册 |
| `app/src/interfaces/index.ts` | Interface 注册 |
| `app/src/displays/index.ts` | Display 注册 |
| `app/src/modules/index.ts` | Module 注册 |
| `app/src/modules/settings/routes/extensions/` | 扩展管理 UI |

### 7.3 共享包

| 包名 | 说明 |
|------|------|
| `@directus/extensions` | 扩展类型定义和工具函数 |
| `@directus/types` | 共享类型定义 |

---

## 总结

Directus 扩展系统设计特点：

1. **分层架构**：API 扩展和 App 扩展分离，各自有独立的加载和注册机制
2. **多来源支持**：本地文件、市场安装、npm 依赖三种扩展来源统一管理
3. **灵活隔离**：沙箱模式提供安全隔离，本地模式提供完整能力
4. **热重载支持**：开发环境下文件变化自动重载，提升开发效率
5. **动态配置**：通过数据库存储扩展设置，支持运行时启用/禁用
6. **模块化设计**：Bundle 扩展允许组合多个子扩展，支持部分启用

这套机制使得 Directus 具备强大的扩展性，同时保持了系统的稳定性和安全性。
