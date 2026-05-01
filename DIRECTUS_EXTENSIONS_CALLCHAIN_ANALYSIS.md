# Directus 三类扩展完整调用链深度分析报告

## 目录

1. [概览](#1-概览)
2. [API 端：扩展发现与入口点生成](#2-api-端扩展发现与入口点生成)
3. [App 端：扩展加载与注册流程](#3-app-端扩展加载与注册流程)
4. [依赖注入机制](#4-依赖注入机制)
5. [Layout 扩展调用链与数据流](#5-layout-扩展调用链与数据流)
6. [Interface 扩展调用链与数据流](#6-interface-扩展调用链与数据流)
7. [Display 扩展调用链与数据流](#7-display-扩展调用链与数据流)
8. [状态同步机制](#8-状态同步机制)
9. [权限与上下文传递](#9-权限与上下文传递)
10. [完整调用链时序图](#10-完整调用链时序图)
11. [关键代码索引](#11-关键代码索引)

---

## 1. 概览

Directus 扩展系统采用 **"API 端发现 + App 端加载"** 的分离架构。三类扩展（Layout、Interface、Display）的完整生命周期涉及：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           完整生命周期                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐        │
│  │  API 端      │      │  App 端      │      │  运行时      │        │
│  │  发现阶段    │─────▶│  注册阶段    │─────▶│  使用阶段    │        │
│  └──────────────┘      └──────────────┘      └──────────────┘        │
│         │                      │                      │                 │
│         ▼                      ▼                      ▼                 │
│  • 扫描文件系统          • 动态 import()        • 动态组件渲染         │
│  • 解析 package.json    • app.component()      • useExtension()      │
│  • 生成入口点 JS         • provide() 注入       • 状态同步            │
│  • 检查 enabled 状态     • 响应式引用管理       • 权限检查            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.1 三类扩展职责对比

| 维度 | Layout | Interface | Display |
|-----|--------|-----------|---------|
| **粒度** | Collection（集合）级 | Field（字段）级 | Field（字段）级 |
| **职责** | 数据列表展示与交互 | 字段值编辑 | 字段值只读展示 |
| **数据获取** | 主动获取（useItems） | 被动接收（props.value） | 被动接收（props.value） |
| **状态管理** | 复杂（selection, pagination, filters） | 简单（value 双向绑定） | 无（只读） |
| **权限依赖** | 集合级权限（CRUD） | 字段级权限 + 集合权限 | 字段级权限 |

---

## 2. API 端：扩展发现与入口点生成

### 2.1 扩展发现机制

API 端在服务启动时扫描扩展目录，解析并验证扩展配置。

#### 2.1.1 扩展来源

扩展来自三个来源：

```typescript
// packages/extensions/src/node/utils/get-extensions.ts

// 1. 本地扩展（extensions/ 目录）
export async function resolveFsExtensions(root: string): Promise<Map<string, Extension>> {
    // 扫描 root 目录下的所有子目录
    const extensionFolders = await listFolders(root, { ignoreHidden: true });
    
    // 逐个解析 package.json
    for (const folder of extensionFolders) {
        const path = join(root, folder);
        const manifest = await fse.readJSON(join(path, 'package.json'));
        
        // 验证 manifest 格式
        const parsedManifest = ExtensionManifest.parse(manifest);
        
        // 生成扩展定义
        const extensionDefinition = getExtensionDefinition(parsedManifest, { 
            path, 
            local: true 
        });
    }
}

// 2. 注册表扩展（通过市场安装）
// 3. 模块扩展（package.json dependencies 中的扩展包）
export async function resolveModuleExtensions(root: string): Promise<Map<string, Extension>> {
    const pkg = await fse.readJSON(join(root, 'package.json'));
    const dependencyNames = Object.keys(pkg.dependencies ?? {});
    
    // 检查每个依赖是否包含 EXTENSION_PKG_KEY
    for (const name of dependencyNames) {
        const path = resolvePackage(name, root);
        const manifest = await fse.readJSON(join(path, 'package.json'));
        
        // 检查 package.json 中是否有 "directus" 字段
        if (EXTENSION_PKG_KEY in manifest) {
            // 这是一个 Directus 扩展
        }
    }
}
```

#### 2.1.2 扩展配置结构

扩展在 `package.json` 中的配置格式：

```json
{
    "name": "directus-extension-my-custom-layout",
    "version": "1.0.0",
    "directus": {
        "type": "layout",
        "path": "dist/index.js",
        "source": "src/index.ts",
        "host": "^10.0.0"
    }
}
```

**关键字段说明**：

| 字段 | 类型 | 说明 |
|-----|------|-----|
| `type` | string | 扩展类型：`layout` / `interface` / `display` / `module` 等 |
| `path` | string \| object | 入口点路径。对于 hybrid 类型（如 operation），是 `{ app: "...", api: "..." }` |
| `host` | string | 兼容的 Directus 版本范围 |
| `sandbox` | boolean | 是否在沙箱中运行（仅 API 扩展） |

#### 2.1.3 扩展类型分类

```typescript
// packages/constants/src/extensions.ts

export const APP_EXTENSION_TYPES = [
    'interface', 
    'display', 
    'layout', 
    'module', 
    'panel', 
    'theme'
] as const;

export const API_EXTENSION_TYPES = [
    'hook', 
    'endpoint'
] as const;

export const HYBRID_EXTENSION_TYPES = [
    'operation'
] as const;

export const BUNDLE_EXTENSION_TYPES = [
    'bundle'
] as const;
```

### 2.2 入口点生成

API 端在运行时动态生成 App 端扩展加载的入口点文件。

#### 2.2.1 生成逻辑

```typescript
// packages/extensions/src/node/utils/generate-extensions-entrypoint.ts

export function generateExtensionsEntrypoint(
    extensionMaps: { 
        local: Map<string, Extension>; 
        registry: Map<string, Extension>; 
        module: Map<string, Extension> 
    },
    settings: ExtensionSettings[],  // 数据库中的扩展启用设置
): string {
    // 1. 过滤启用的扩展
    const appOrHybridExtensions: (AppExtension | HybridExtension)[] = [];
    
    for (const [source, extensions] of Object.entries(extensionMaps)) {
        for (const [folder, extension] of extensions.entries()) {
            // 检查数据库中的启用状态
            const settingsForExtension = settings.find(
                (setting) => setting.source === source && setting.folder === folder
            );
            
            if (!settingsForExtension) continue;
            
            // 仅保留启用的 App 或 Hybrid 扩展
            if (isIn(extension.type, [...APP_EXTENSION_TYPES, ...HYBRID_EXTENSION_TYPES]) 
                && settingsForExtension.enabled) {
                appOrHybridExtensions.push(extension);
            }
        }
    }
    
    // 2. 生成 import 语句
    const appOrHybridExtensionImports = [...APP_EXTENSION_TYPES, ...HYBRID_EXTENSION_TYPES]
        .flatMap((type) =>
            appOrHybridExtensions
                .filter((extension) => extension.type === type)
                .map(
                    (extension, i) =>
                        `import ${type}${i} from './${pathToRelativeUrl(
                            path.resolve(
                                extension.path,
                                isTypeIn(extension, HYBRID_EXTENSION_TYPES) 
                                    ? extension.entrypoint.app 
                                    : extension.entrypoint,
                            ),
                        )}';`,
                ),
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
                            extension.entries.some((entry) => entry.type === type) 
                                ? `...${type}Bundle${i}` 
                                : null,
                        )
                        .filter((e): e is string => e !== null),
                )
                .join(',')}];`,
    );
    
    // 4. 组合成完整文件
    return `${appOrHybridExtensionImports.join('')}
            ${bundleExtensionImports.join('')}
            ${extensionExports.join('')}`;
}
```

#### 2.2.2 生成的入口点示例

生成的 `extensions/sources/index.js` 示例：

```javascript
// 导入语句
import layout0 from './../../extensions/directus-extension-my-layout/dist/index.js';
import interface0 from './../../extensions/directus-extension-my-interface/dist/index.js';
import display0 from './../../extensions/directus-extension-my-display/dist/index.js';

// 导出语句（按类型分组）
export const layouts = [layout0];
export const interfaces = [interface0];
export const displays = [display0];
export const modules = [];
export const panels = [];
export const operations = [];
export const themes = [];
```

---

## 3. App 端：扩展加载与注册流程

### 3.1 应用启动入口

```typescript
// app/src/main.ts

async function init() {
    const app = createApp(App);
    
    // 1. 安装基础插件
    app.use(i18n);
    app.use(createPinia());
    app.use(createHead());
    
    // 2. 注册核心组件、指令、视图
    registerDirectives(app);
    registerComponents(app);
    registerViews(app);
    
    // 3. 加载扩展（异步）
    await loadExtensions();
    
    // 4. 注册扩展到 Vue 应用
    registerExtensions(app);
    
    // 5. 安装路由（必须在扩展注册之后，确保模块路由已注册）
    app.use(router);
    
    // 6. 注入全局依赖
    useSystem(app);
    
    // 7. 挂载应用
    app.mount('#app');
}
```

### 3.2 扩展加载（loadExtensions）

```typescript
// app/src/extensions.ts:31-42

let customExtensions: AppExtensionConfigs | null = null;

export async function loadExtensions(): Promise<void> {
    try {
        customExtensions = import.meta.env.DEV
            // 开发环境：使用 Vite 插件提供的虚拟模块
            ? await import(/* @vite-ignore */ '@directus-extensions')
            // 生产环境：使用 API 生成的入口点
            : await import(/* @vite-ignore */ `${getRootPath()}extensions/sources/index.js`);
    } catch (err: any) {
        // 错误处理：记录警告但不中断应用
        console.warn(`Couldn't load extensions`);
        console.warn(err);
    }
}
```

**加载策略说明**：

| 环境 | 加载方式 | 说明 |
|-----|---------|-----|
| 开发环境 | `import('@directus-extensions')` | Vite 插件 `@directus/extensions-sdk` 提供的虚拟模块，支持 HMR |
| 生产环境 | `import('${rootPath}extensions/sources/index.js')` | API 端动态生成的入口点文件 |

### 3.3 扩展注册（registerExtensions）

```typescript
// app/src/extensions.ts:44-95

// 全局扩展存储（响应式）
const extensions: RefRecord<AppExtensionConfigs> = {
    interfaces: shallowRef([]),
    displays: shallowRef([]),
    layouts: shallowRef([]),
    modules: shallowRef([]),
    panels: shallowRef([]),
    operations: shallowRef([]),
    themes: shallowRef([]),
};

export function registerExtensions(app: App): void {
    // 1. 获取内置扩展
    const interfaces = getInternalInterfaces();
    const displays = getInternalDisplays();
    const layouts = getInternalLayouts();
    const modules = getInternalModules();
    const panels = getInternalPanels();
    const operations = getInternalOperations();
    const themes = [];

    // 2. 合并自定义扩展（如果加载成功）
    if (customExtensions !== null) {
        interfaces.push(...customExtensions.interfaces);
        displays.push(...customExtensions.displays);
        layouts.push(...customExtensions.layouts);
        modules.push(...customExtensions.modules);
        panels.push(...customExtensions.panels);
        operations.push(...customExtensions.operations);
        themes.push(...customExtensions.themes);
    }

    // 3. 注册为 Vue 全局组件
    registerInterfaces(interfaces, app);
    registerDisplays(displays, app);
    registerLayouts(layouts, app);
    registerPanels(panels, app);
    registerOperations(operations, app);
    registerThemes(themes);

    // 4. 设置语言切换响应式更新
    watch(
        i18n.global.locale,
        () => {
            extensions.interfaces.value = translate(interfaces);
            extensions.displays.value = translate(displays);
            extensions.layouts.value = translate(layouts);
            extensions.panels.value = translate(panels);
            extensions.operations.value = translate(operations);
        },
        { immediate: true },
    );

    // 5. 模块特殊处理（注册路由）
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

### 3.4 各类扩展的具体注册

#### 3.4.1 Layout 注册

```typescript
// app/src/layouts/index.ts

// 使用 Vite 的 import.meta.glob 自动扫描内置布局
export function getInternalLayouts(): LayoutConfig[] {
    const layouts = import.meta.glob<LayoutConfig>('./*/index.ts', { 
        import: 'default', 
        eager: true 
    });
    return sortBy(Object.values(layouts), 'id');
}

// 注册为 Vue 组件
export function registerLayouts(layouts: LayoutConfig[], app: App): void {
    for (const layout of layouts) {
        // 主布局组件
        app.component(`layout-${layout.id}`, layout.component);
        // 布局选项面板（侧边栏配置）
        app.component(`layout-options-${layout.id}`, layout.slots.options);
        // 布局侧边栏内容
        app.component(`layout-sidebar-${layout.id}`, layout.slots.sidebar);
        // 布局操作栏组件（头部右侧）
        app.component(`layout-actions-${layout.id}`, layout.slots.actions);
    }
}
```

**Layout 组件命名规范**：

| 组件用途 | 命名格式 | 示例 |
|---------|---------|------|
| 主布局组件 | `layout-{id}` | `layout-tabular` |
| 选项面板 | `layout-options-{id}` | `layout-options-tabular` |
| 侧边栏内容 | `layout-sidebar-{id}` | `layout-sidebar-tabular` |
| 操作栏组件 | `layout-actions-{id}` | `layout-actions-tabular` |

#### 3.4.2 Interface 注册

```typescript
// app/src/interfaces/index.ts

export function getInternalInterfaces(): InterfaceConfig[] {
    // 扫描普通接口和系统接口目录
    const interfaces = import.meta.glob<InterfaceConfig>(
        ['./*/index.ts', './_system/*/index.ts'], 
        { import: 'default', eager: true }
    );
    return sortBy(Object.values(interfaces), 'id');
}

export function registerInterfaces(interfaces: InterfaceConfig[], app: App): void {
    for (const inter of interfaces) {
        // 主编辑组件
        app.component(`interface-${inter.id}`, inter.component);
        
        // 选项组件（仅当 options 是组件时注册）
        // 如果是函数或数组，则由字段配置界面动态渲染
        if (typeof inter.options !== 'function' 
            && Array.isArray(inter.options) === false 
            && inter.options !== null) {
            app.component(`interface-options-${inter.id}`, inter.options);
        }
    }
}
```

**Interface options 的三种形式**：

| 形式 | 类型 | 处理方式 |
|-----|------|---------|
| 数组 | `DeepPartial<AppField>[]` | 动态渲染为表单项 |
| 函数 | `(ctx) => DeepPartial<AppField>[]` | 调用函数获取配置后动态渲染 |
| 组件 | `Component` | 注册为 `interface-options-{id}` 组件 |

#### 3.4.3 Display 注册

```typescript
// app/src/displays/index.ts

export function getInternalDisplays(): DisplayConfig[] {
    const displays = import.meta.glob<DisplayConfig>('./*/index.ts', { 
        import: 'default', 
        eager: true 
    });
    return sortBy(Object.values(displays), 'id');
}

export function registerDisplays(displays: DisplayConfig[], app: App): void {
    for (const display of displays) {
        app.component(`display-${display.id}`, display.component);
        
        // 选项组件（同 Interface）
        if (typeof display.options !== 'function' 
            && Array.isArray(display.options) === false 
            && display.options !== null) {
            app.component(`display-options-${display.id}`, display.options);
        }
    }
}
```

---

## 4. 依赖注入机制

### 4.1 注入点设置

```typescript
// app/src/composables/use-system.ts

import { API_INJECT, EXTENSIONS_INJECT, SDK_INJECT, STORES_INJECT } from '@directus/constants';

export function useSystem(app: App): void {
    // 1. 注入所有 Pinia Stores
    app.provide(STORES_INJECT, {
        useAppStore,
        useCollectionsStore,
        useFieldsStore,
        useFlowsStore,
        useInsightsStore,
        useNotificationsStore,
        usePermissionsStore,
        usePresetsStore,
        useRelationsStore,
        useRequestsStore,
        useServerStore,
        useSettingsStore,
        useTranslationsStore,
        useUserStore,
    });

    // 2. 注入 API 客户端（Axios 实例）
    app.provide(API_INJECT, api);

    // 3. 注入 SDK 客户端
    app.provide(SDK_INJECT, sdk);

    // 4. 注入扩展配置（关键！）
    app.provide(EXTENSIONS_INJECT, useExtensions());
}
```

### 4.2 注入键定义

```typescript
// packages/constants/src/index.ts（推断）

export const API_INJECT = Symbol('api');
export const EXTENSIONS_INJECT = Symbol('extensions');
export const SDK_INJECT = Symbol('sdk');
export const STORES_INJECT = Symbol('stores');
```

### 4.3 扩展访问方式

#### 4.3.1 通过 useExtensions 访问

```typescript
// packages/composables/src/use-system.ts

/**
 * 从注入获取全局扩展配置
 * 在任何组件或 composable 中调用此函数即可访问所有扩展
 */
export function useExtensions(): RefRecord<AppExtensionConfigs> {
    const extensions = inject<RefRecord<AppExtensionConfigs>>(EXTENSIONS_INJECT);
    
    if (!extensions) {
        throw new Error('[useExtensions]: The extensions could not be found.');
    }
    
    return extensions;
}
```

#### 4.3.2 通过 useExtension 查找单个扩展

```typescript
// app/src/composables/use-extension.ts

import { pluralize } from '@directus/utils';

/**
 * 根据类型和 ID 查找单个扩展配置
 * @param type - 扩展类型：'interface' | 'display' | 'layout' 等
 * @param name - 扩展 ID
 * @returns 响应式的扩展配置（找不到返回 null）
 */
export function useExtension<T extends AppExtensionType | HybridExtensionType>(
    type: T | Ref<T>,
    name: string | Ref<string | null>,
): Ref<AppExtensionConfigs[Plural<T>][number] | null> {
    const extensions = useExtensions();
    
    return computed(() => {
        if (unref(name) === null) return null;
        
        // pluralize('interface') → 'interfaces'
        // pluralize('display') → 'displays'
        // pluralize('layout') → 'layouts'
        return (extensions[pluralize(unref(type))].value as any[])
            .find(({ id }) => id === unref(name)) ?? null;
    });
}
```

**使用示例**：

```typescript
// 在组件中查找特定 interface
const myInterface = useExtension('interface', 'my-custom-input');

// 响应式：当 ID 变化时自动重新查找
const interfaceId = ref('input');
const interfaceConfig = useExtension('interface', interfaceId);
```

---

## 5. Layout 扩展调用链与数据流

### 5.1 Layout 配置结构

```typescript
// packages/types/src/extensions/layouts.ts

export interface LayoutConfig<Options = any, Query = any> {
    id: string;
    name: string;
    icon: string;
    component: Component;
    slots: {
        options: Component;      // 布局选项配置面板
        sidebar: Component;      // 侧边栏内容
        actions: Component;      // 头部操作栏
    };
    smallHeader?: boolean;
    headerShadow?: boolean;
    sidebarShadow?: boolean;
    
    /**
     * 组合式 setup 函数（核心！）
     * 返回的所有属性和方法将通过 layoutState 传递给组件
     */
    setup: (
        props: LayoutProps<Options, Query>, 
        ctx: LayoutContext
    ) => Record<string, unknown>;
}

export interface LayoutProps<Options = any, Query = any> {
    collection: string | null;           // 当前集合名称
    selection: (number | string)[];      // 选中的主键数组
    layoutOptions: Options;               // 布局选项（用户配置）
    layoutQuery: Query;                   // 布局查询参数（分页、排序等）
    layoutProps: Record<string, unknown>; // 额外传递的属性
    filterUser: Filter | null;            // 用户过滤条件
    filterSystem: Filter | null;          // 系统过滤条件（如归档）
    filter: Filter | null;                // 合并后的过滤条件
    search: string | null;                 // 搜索关键词
    selectMode: boolean;                   // 是否为选择模式（从关系字段选择）
    showSelect: ShowSelect;               // 选择显示类型
    readonly: boolean;                     // 是否只读
    resetPreset?: () => Promise<void>;    // 重置预设函数
    clearFilters?: () => void;             // 清除过滤函数
}

interface LayoutContext {
    emit: (
        event: 'update:selection' | 'update:layoutOptions' | 'update:layoutQuery',
        ...args: any[]
    ) => void;
}
```

### 5.2 Layout 调用链全景图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Layout 完整调用链                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  用户访问 /content/my-collection                                             │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Router 导航到 collection.vue                                      │  │
│  │    - 解析路由参数：collection = 'my-collection'                      │  │
│  │    - 检查是否有 bookmark 参数                                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 2. usePreset() 加载布局状态                                           │  │
│  │    - 从 presetsStore 获取集合预设                                     │  │
│  │    - 或从 bookmarksStore 获取书签（如果有 bookmark ID）               │  │
│  │    - 返回：layout, layoutOptions, layoutQuery, filter, search 等    │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 3. useLayout() 创建布局包装器                                         │  │
│  │    - 调用 useExtensions() 获取所有 layouts                           │  │
│  │    - 为每个 layout 调用 createLayoutWrapper()                        │  │
│  │    - 根据 layout.value 选择对应的包装器                               │  │
│  │    - 如果找不到，默认使用 tabular                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 4. createLayoutWrapper() 创建动态组件                                 │  │
│  │    - 定义标准 props 和 emits                                          │  │
│  │    - 调用 layout.setup(props, { emit })                               │  │
│  │    - 合并返回值到响应式 state                                         │  │
│  │    - 渲染作用域插槽，传递 layoutState                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 5. Layout.setup() 执行业务逻辑                                        │  │
│  │    - useSync(props, 'selection', emit) 同步选中项                    │  │
│  │    - useSync(props, 'layoutOptions', emit) 同步选项                  │  │
│  │    - useSync(props, 'layoutQuery', emit) 同步查询                    │  │
│  │    - useCollection(collection) 获取集合信息                           │  │
│  │    - useItems(collection, query) 获取数据列表                         │  │
│  │    - 返回：items, loading, error, refresh 等                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 6. 组件渲染                                                            │  │
│  │    - <component :is="`layout-${layoutId}`" v-bind="layoutState" /> │  │
│  │    - <component :is="`layout-actions-${layoutId}`" v-bind="..." /> │  │
│  │    - <component :is="`layout-options-${layoutId}`" v-bind="..." /> │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 Layout Wrapper 实现详解

```typescript
// packages/composables/src/use-layout.ts

export function createLayoutWrapper<Options, Query>(layout: LayoutConfig): Component {
    return defineComponent({
        name: `${layout.id}-wrapper`,  // e.g., "tabular-wrapper"
        
        // 定义标准 props
        props: {
            collection: { type: String, required: true },
            selection: { type: Array, default: () => [] },
            layoutOptions: { type: Object, default: () => ({}) },
            layoutQuery: { type: Object, default: () => ({}) },
            layoutProps: { type: Object, default: () => ({}) },
            filter: { type: Object as PropType<Filter>, default: null },
            filterUser: { type: Object as PropType<Filter>, default: null },
            filterSystem: { type: Object as PropType<Filter>, default: null },
            search: { type: String as PropType<string | null>, default: null },
            showSelect: { type: String as PropType<ShowSelect>, default: 'multiple' },
            selectMode: { type: Boolean, default: false },
            readonly: { type: Boolean, default: false },
            resetPreset: { type: Function as PropType<() => Promise<void>>, default: null },
            clearFilters: { type: Function as PropType<() => void>, default: null },
        },
        
        // 定义可写属性的更新事件
        emits: ['update:selection', 'update:layoutOptions', 'update:layoutQuery'],
        
        setup(props, { emit }) {
            // 1. 调用 layout 的 setup 函数，获取业务逻辑返回值
            // 2. 合并 props 和 layout 配置（如 sidebarShadow）
            const state: Record<string, unknown> = reactive({
                ...layout.setup(props, { emit }),
                ...toRefs(props),
                sidebarShadow: layout.sidebarShadow ?? false,
            });
            
            // 3. 为每个 state 属性创建 onUpdate 处理器
            // 用于支持 v-model 绑定
            for (const key in state) {
                state[`onUpdate:${key}`] = (value: unknown) => {
                    if (isWritableProp(key)) {
                        // 可写属性：通过 emit 更新父组件
                        emit(`update:${key}`, value);
                    } else if (!Object.keys(props).includes(key)) {
                        // 非 props 属性：本地更新
                        state[key] = value;
                    }
                };
            }
            
            return { state };
        },
        
        // 4. 通过作用域插槽传递 state
        render(ctx: any) {
            return ctx.$slots.default !== undefined 
                ? ctx.$slots.default({ layoutState: ctx.state }) 
                : null;
        },
    });
}
```

### 5.4 Layout 在 Collection 页面的使用

```vue
<!-- app/src/modules/content/routes/collection.vue -->
<script setup lang="ts">
// 1. 使用 usePreset 管理持久化状态
const {
    layout,           // Ref<string> - 当前布局 ID
    layoutOptions,    // Ref<Record> - 布局选项
    layoutQuery,      // Ref<Record> - 查询参数
    filter,           // Ref<Filter | null> - 过滤条件
    search,           // Ref<string | null> - 搜索词
    resetPreset,      // Function - 重置预设
} = usePreset(collection, bookmarkID);

// 2. 获取布局包装器
const { layoutWrapper } = useLayout(layout);

// 3. 本地状态
const selection = ref<Item[]>([]);  // 选中项
const layoutRef = ref();              // 布局组件引用

// 4. 通过引用调用布局方法
async function refresh() {
    await layoutRef.value?.state?.refresh?.();
}
</script>

<template>
    <!-- 5. 使用动态组件渲染布局包装器 -->
    <component
        :is="layoutWrapper"
        ref="layoutRef"
        v-slot="{ layoutState }"
        
        <!-- 双向绑定：使用 v-model 语法 -->
        v-model:selection="selection"
        v-model:layout-options="layoutOptions"
        v-model:layout-query="layoutQuery"
        
        <!-- 单向绑定：其他 props -->
        :filter-user="filter"
        :filter-system="archiveFilter"
        :filter="mergeFilters(filter, archiveFilter)"
        :search="search"
        :collection="collection"
        :reset-preset="resetPreset"
        :clear-filters="clearFilters"
    >
        <!-- 6. 在插槽内使用 layoutState -->
        <PrivateView :sidebar-shadow="layoutState.sidebarShadow">
            <!-- 操作栏：动态组件 -->
            <template #actions:prepend>
                <component 
                    :is="`layout-actions-${layout || 'tabular'}`" 
                    v-bind="layoutState" 
                />
            </template>
            
            <!-- 主内容：动态组件 -->
            <component :is="`layout-${layout || 'tabular'}`" v-bind="layoutState">
                <template #no-results>...</template>
                <template #no-items>...</template>
                <template #error="{ error, reset }">...</template>
            </component>
            
            <!-- 侧边栏 -->
            <template #sidebar>
                <LayoutSidebarDetail v-model="layout">
                    <!-- 选项面板：动态组件 -->
                    <component :is="`layout-options-${layout || 'tabular'}`" v-bind="layoutState" />
                </LayoutSidebarDetail>
                <!-- 侧边栏内容：动态组件 -->
                <component :is="`layout-sidebar-${layout || 'tabular'}`" v-bind="layoutState" />
            </template>
        </PrivateView>
    </component>
</template>
```

### 5.5 Tabular Layout setup 示例

```typescript
// app/src/layouts/tabular/index.ts

export default defineLayout<LayoutOptions, LayoutQuery>({
    id: 'tabular',
    name: '$t:layouts.tabular.tabular',
    icon: 'table_rows',
    component: TabularLayout,
    slots: {
        options: TabularOptions,
        sidebar: () => undefined,
        actions: TabularActions,
    },
    headerShadow: false,
    
    setup(props, { emit }) {
        // ==========================================
        // 阶段 1：设置基础响应式同步
        // ==========================================
        
        // 使用 useSync 创建与父组件的双向绑定
        const selection = useSync(props, 'selection', emit);
        const layoutOptions = useSync(props, 'layoutOptions', emit);
        const layoutQuery = useSync(props, 'layoutQuery', emit);
        
        // 解构只读 props
        const { collection, filter, filterSystem, filterUser, search } = toRefs(props);
        
        // ==========================================
        // 阶段 2：获取集合和字段信息
        // ==========================================
        
        const { 
            info,                    // 集合元数据
            primaryKeyField,         // 主键字段配置
            fields: fieldsInCollection, // 所有字段
            sortField                // 排序字段（如果有）
        } = useCollection(collection);
        
        // ==========================================
        // 阶段 3：准备查询参数
        // ==========================================
        
        // 从 layoutQuery 提取或使用默认值
        const page = syncRefProperty(layoutQuery, 'page', 1);
        const limit = syncRefProperty(layoutQuery, 'limit', 25);
        const defaultSort = computed(() => {
            const field = sortField.value ?? primaryKeyField.value?.field;
            return field ? [field] : [];
        });
        const sort = syncRefProperty(layoutQuery, 'sort', defaultSort);
        
        // 计算需要获取的字段
        const fieldsDefaultValue = computed(() => {
            return fieldsInCollection.value
                .filter((field) => !field.meta?.hidden && !field.meta?.special?.includes('no-data'))
                .slice(0, 4)  // 默认显示前 4 个字段
                .map(({ field }) => field)
                .sort();
        });
        
        const fields = computed({
            get() {
                if (layoutQuery.value?.fields) {
                    return layoutQuery.value.fields.filter(
                        (field) => fieldsStore.getField(collection.value!, field)
                    );
                } else {
                    return unref(fieldsDefaultValue);
                }
            },
            set(value) {
                layoutQuery.value = Object.assign({}, layoutQuery.value, { fields: value });
            },
        });
        
        // 处理关系字段的别名
        const { aliasedFields, aliasQuery, aliasedKeys } = useAliasFields(fields, collection);
        const fieldsWithRelationalAliased = computed(() =>
            flatten(Object.values(aliasedFields.value).map(({ fields }) => fields)),
        );
        
        // ==========================================
        // 阶段 4：获取数据（核心！）
        // ==========================================
        
        const {
            items,              // 数据列表
            loading,            // 加载状态
            loadingItemCount,   // 数量加载状态
            error,              // 错误信息
            totalPages,         // 总页数
            itemCount,          // 当前过滤后的数量
            totalCount,         // 集合总数量
            changeManualSort,   // 手动排序方法
            getItems,           // 重新获取数据
            getItemCount,       // 获取数量
            getTotalCount,      // 获取总数
        } = useItems(collection, {
            sort,
            limit,
            page,
            fields: fieldsWithRelationalAliased,
            alias: aliasQuery,
            filter,
            search,
            filterSystem,
        });
        
        // ==========================================
        // 阶段 5：处理表格特定逻辑
        // ==========================================
        
        const onClick = useLayoutClickHandler({ props, selection, primaryKeyField });
        
        // 表格头部配置
        const tableHeaders = computed<HeaderRaw[]>({
            get() {
                return activeFields.value.map((field) => {
                    return {
                        text: field.name,
                        value: field.key,
                        width: localWidths.value[field.key] || layoutOptions.value?.widths?.[field.key] || 144,
                        align: layoutOptions.value?.align?.[field.key] || 'left',
                        field: {
                            // 关键：为每个字段配置 display 和 interface
                            display: field.meta?.display || getDefaultDisplayForType(field.type),
                            displayOptions: field.meta?.display_options,
                            interface: field.meta?.interface,
                            interfaceOptions: field.meta?.options,
                            type: field.type,
                            field: field.field,
                            collection: field.collection,
                        },
                        sortable: ['json', 'alias', 'presentation', 'translations'].includes(field.type) === false,
                    };
                });
            },
            // ...
        });
        
        // ==========================================
        // 阶段 6：导出方法和状态
        // ==========================================
        
        function refresh() {
            getItems();
            getTotalCount();
            getItemCount();
        }
        
        function selectAll() {
            if (!primaryKeyField.value) return;
            const pk = primaryKeyField.value;
            selection.value = items.value.map((item) => item[pk.field]);
        }
        
        function download() {
            if (!collection.value) return;
            saveAsCSV(collection.value, fields.value, items.value);
        }
        
        return {
            // 数据
            items,
            loading,
            error,
            totalPages,
            itemCount,
            totalCount,
            
            // 表格配置
            tableHeaders,
            tableSort,
            tableRowHeight,
            activeFields,
            tableSpacing,
            
            // 方法
            onRowClick: onClick,
            refresh,
            selectAll,
            download,
            resetPresetAndRefresh: async () => {
                await props?.resetPreset?.();
                refresh();
            },
            
            // 字段信息
            fieldsInCollection,
            fields,
            primaryKeyField,
            info,
            
            // 分页
            page,
            limit,
        };
    },
});
```

---

## 6. Interface 扩展调用链与数据流

### 6.1 Interface 配置结构

```typescript
// packages/types/src/extensions/interfaces.ts

export interface InterfaceConfig {
    id: string;
    name: string;
    icon: string;
    description?: string;
    component: Component;
    
    /**
     * 选项配置
     * - 数组：表单项配置列表
     * - 对象：{ standard: [...], advanced: [...] } 分组
     * - 函数：根据上下文动态生成
     * - 组件：自定义选项组件
     * - null：无选项
     */
    options:
        | DeepPartial<AppField>[]
        | { standard: DeepPartial<AppField>[]; advanced: DeepPartial<AppField>[] }
        | ((ctx: ExtensionOptionsContext) => ...)
        | Exclude<ComponentOptions, any>
        | null;
    
    types: readonly Type[];           // 支持的字段类型
    localTypes?: readonly LocalType[]; // 支持的本地类型
    group?: 'standard' | 'selection' | 'relational' | 'presentation' | 'group' | 'other';
    order?: number;
    relational?: boolean;              // 是否为关系型
    hideLabel?: boolean;               // 是否隐藏标签
    hideLoader?: boolean;              // 是否隐藏加载状态
    indicatorStyle?: 'active' | 'hidden' | 'muted';
    autoKey?: boolean;
    system?: boolean;                  // 是否为系统内置
    recommendedDisplays?: string[];    // 推荐的显示组件
    preview?: string;                  // 预览 SVG
}
```

### 6.2 Interface 调用链全景图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          Interface 完整调用链                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  用户打开字段编辑表单                                                         │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 1. v-form 渲染表单                                                    │  │
│  │    - 接收 modelValues（表单数据对象）                                │  │
│  │    - 接收 fields（字段配置列表）                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 2. getFormFields() 处理字段配置                                       │  │
│  │    - 解析字段的 meta.interface                                        │  │
│  │    - 解析字段的 meta.options                                          │  │
│  │    - 处理条件逻辑（如 conditions）                                    │  │
│  │    - 返回 FormField 数组                                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 3. form-field-interface.vue 渲染单个字段                              │  │
│  │    - 调用 useExtension('interface', field.meta.interface)            │  │
│  │    - 检查 interface 是否存在                                          │  │
│  │    - 如果不存在或 rawEditorActive，使用 SystemRawEditor              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 4. 动态渲染 Interface 组件                                            │  │
│  │    - 组件名：`interface-${id}`                                        │  │
│  │    - 传递 props：value, disabled, field-data, collection 等         │  │
│  │    - 监听事件：@input, @set-field-value                               │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 5. Interface 组件内部                                                 │  │
│  │    - 接收 props.value 作为初始值                                      │  │
│  │    - 用户交互后 emit('input', newValue)                               │  │
│  │    - 或 emit('set-field-value', { field: 'other_field', value: x }) │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 6. 父组件处理更新                                                     │  │
│  │    - @input → 更新 modelValues[field.field]                          │  │
│  │    - @set-field-value → 更新其他字段值                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 6.3 Interface 渲染组件详解

```vue
<!-- app/src/components/v-form/components/form-field-interface.vue -->
<script setup lang="ts">
const props = defineProps<{
    field: FormField;                    // 完整字段配置
    modelValue?: any;                    // 当前值（v-model）
    primaryKey?: string | number | null; // 当前记录主键
    loading?: boolean;
    disabled?: boolean;
    nonEditable?: boolean;
    autofocus?: boolean;
    rawEditorEnabled?: boolean;
    rawEditorActive?: boolean;
    direction?: string;
    version?: ContentVersionMaybeNew | null;
}>();

defineEmits(['update:modelValue', 'setFieldValue']);

// 1. 查找 Interface 配置
const inter = useExtension(
    'interface',
    computed(() => props.field?.meta?.interface ?? 'input'),
);

const interfaceExists = computed(() => !!inter.value);

// 2. 确定组件名称
const componentName = computed(() => {
    return props.field?.meta?.interface
        ? `interface-${props.field.meta.interface}`
        : `interface-${getDefaultInterfaceForType(props.field.type!)}`;
});

// 3. 处理值（默认值 fallback）
const value = computed(() =>
    props.modelValue === undefined 
        ? (props.field.schema?.default_value ?? null) 
        : props.modelValue,
);
</script>

<template>
    <div class="interface">
        <!-- 加载状态 -->
        <VSkeletonLoader v-if="loading && field.hideLoader !== true" />
        
        <!-- 正常渲染 Interface -->
        <VErrorBoundary v-if="interfaceExists && !rawEditorActive" :name="componentName">
            <component
                :is="componentName"
                
                <!-- 扩展选项（从 field.meta.options 传递） -->
                v-bind="(field.meta && field.meta.options) || {}"
                
                <!-- 标准 props -->
                :autofocus="disabled !== true && autofocus"
                :disabled="disabled"
                :non-editable="nonEditable"
                :loading="loading"
                :value="value"
                :batch-mode="batchMode"
                :batch-active="batchActive"
                :comparison-mode="!!comparison"
                :comparison-active="comparisonActive"
                :comparison-side="comparison?.side"
                :width="(field.meta && field.meta.width) || 'full'"
                :type="field.type"
                :collection="field.collection"
                :field="field.field"
                :field-data="field"
                :primary-key="primaryKey"
                :length="field.schema && field.schema.max_length"
                :direction="direction"
                :raw-editor-enabled="rawEditorEnabled"
                :version="version"
                
                <!-- 事件监听 -->
                @input="$emit('update:modelValue', $event)"
                @set-field-value="$emit('setFieldValue', $event)"
            />
            
            <template #fallback>
                <VNotice type="warning">{{ $t('unexpected_error') }}</VNotice>
            </template>
        </VErrorBoundary>
        
        <!-- 原始编辑器（用于调试或特殊场景） -->
        <InterfaceSystemRawEditor
            v-else-if="rawEditorEnabled && rawEditorActive"
            :value="value"
            :type="field.type"
            @input="$emit('update:modelValue', $event)"
        />
        
        <!-- 未找到 Interface 的 fallback -->
        <VNotice v-else type="warning">
            {{ $t('interface_not_found', { interface: field.meta && field.meta.interface }) }}
        </VNotice>
    </div>
</template>
```

### 6.4 Interface 组件 Props 完整列表

| Prop | 类型 | 说明 |
|-----|------|-----|
| `value` | any | 当前字段值 |
| `disabled` | boolean | 是否禁用 |
| `nonEditable` | boolean | 是否不可编辑（与 disabled 不同，可能只是视觉上） |
| `loading` | boolean | 是否加载中 |
| `type` | string | 字段类型（如 'string', 'integer'） |
| `collection` | string | 所属集合 |
| `field` | string | 字段名 |
| `field-data` | Field | 完整字段配置对象 |
| `primary-key` | string \| number \| null | 当前记录主键 |
| `width` | string | 显示宽度：'half' \| 'full' \| 'fill' |
| `length` | number | 字段最大长度 |
| `direction` | string | 文字方向：'ltr' \| 'rtl' |
| `batch-mode` | boolean | 是否批量编辑模式 |
| `batch-active` | boolean | 批量编辑是否激活 |
| `comparison-mode` | boolean | 是否对比模式（版本对比） |
| `comparison-active` | boolean | 对比是否激活 |
| `comparison-side` | string | 对比侧：'left' \| 'right' |
| `raw-editor-enabled` | boolean | 是否启用原始编辑器 |
| `version` | ContentVersion | 内容版本信息 |

### 6.5 Interface 组件 Events

| Event | Payload | 说明 |
|-------|---------|-----|
| `@input` | `newValue: any` | 值变更时触发，用于 v-model |
| `@set-field-value` | `{ field: string, value: any }` | 设置其他字段值时触发 |

---

## 7. Display 扩展调用链与数据流

### 7.1 Display 配置结构

```typescript
// packages/types/src/extensions/displays.ts

export interface DisplayConfig {
    id: string;
    name: string;
    icon: string;
    description?: string;
    
    component: Component;
    
    /**
     * 可选：纯文本格式化处理器
     * 当只需要简单文本格式化时使用，不需要渲染完整组件
     */
    handler?: (
        value: any,
        options: Record<string, any>,
        ctx: { 
            interfaceOptions?: Record<string, any>; 
            field?: Field; 
            collection?: string 
        }
    ) => string | null;
    
    options:
        | DeepPartial<AppField>[]
        | { standard: DeepPartial<AppField>[]; advanced: DeepPartial<AppField>[] }
        | ((ctx: ExtensionOptionsContext) => ...)
        | Exclude<ComponentOptions, any>
        | null;
    
    types: readonly Type[];           // 支持的字段类型
    localTypes?: readonly LocalType[];
    
    /**
     * 需要额外加载的关联字段
     * 用于显示关联数据（如显示用户名而不是用户 ID）
     */
    fields?: string[] | DisplayFieldsFunction;
}

export type DisplayFieldsFunction = (
    options: any,
    context: { collection: string; field: string; type: string }
) => string[];
```

### 7.2 Display 调用链全景图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Display 完整调用链                                    │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  列表渲染表格行                                                               │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Layout 组件构建 tableHeaders                                       │  │
│  │    - 遍历 activeFields                                                │  │
│  │    - 为每个字段配置：                                                 │  │
│  │      - display: field.meta.display || getDefaultDisplayForType()    │  │
│  │      - displayOptions: field.meta.display_options                    │  │
│  │      - interface: field.meta.interface                               │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 2. 表格单元格渲染 #cell 插槽                                          │  │
│  │    - 接收 item（当前行数据）和 header（列配置）                       │  │
│  │    - 从 header.field 获取 display 配置                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 3. render-display.vue 渲染 Display                                    │  │
│  │    - 调用 useExtension('display', header.field.display)              │  │
│  │    - 检查 value 是否为 null                                           │  │
│  │    - 检查 displayInfo 是否存在                                        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│              │                                                               │
│              ▼                                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ 4. 动态渲染 Display 组件                                              │  │
│  │    - 组件名：`display-${id}`                                          │  │
│  │    - 传递 props：value, options, type, collection, field 等          │  │
│  │    - 无事件（Display 是只读的）                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 Display 渲染组件详解

```vue
<!-- app/src/views/private/components/render-display.vue -->
<script setup lang="ts">
const props = defineProps<{
    display: string | null;              // Display ID
    options?: Record<string, unknown>;   // Display 选项
    interface?: string;                   // 关联的 Interface ID
    interfaceOptions?: Record<string, unknown>;
    value?: any;                          // 当前值
    type: string;                         // 字段类型
    collection: string;                   // 集合
    field: string;                        // 字段名
}>();

const { display } = toRefs(props);

// 查找 Display 配置
const displayInfo = useExtension('display', display);
</script>

<template>
    <!-- 值为 null 时显示占位符 -->
    <ValueNull v-if="value === null || value === undefined" />
    
    <!-- 未找到 Display 时显示原始文本 -->
    <VTextOverflow v-else-if="displayInfo === null" class="display" :text="value" />
    
    <!-- 正常渲染 Display 组件 -->
    <VErrorBoundary v-else :name="`display-${display}`">
        <component
            :is="`display-${display}`"
            
            <!-- Display 选项 -->
            v-bind="options"
            
            <!-- 上下文信息 -->
            :interface="interface"
            :interface-options="interfaceOptions"
            :value="value"
            :type="type"
            :collection="collection"
            :field="field"
        />
        
        <template #fallback>
            <VTextOverflow class="display" :text="value" />
        </template>
    </VErrorBoundary>
</template>
```

### 7.4 Display handler 示例

```typescript
// app/src/displays/formatted-value/index.ts

export default defineDisplay({
    id: 'formatted-value',
    name: '$t:displays.formatted-value.formatted-value',
    types: ['string', 'text', 'integer', 'float', 'decimal', 'bigInteger'],
    icon: 'text_format',
    component: DisplayFormattedValue,
    
    /**
     * 纯文本格式化处理器
     * 用于不需要 Vue 组件的场景（如导出、API 响应）
     */
    handler: (value, options) => {
        const prefix = options.prefix ?? '';
        const suffix = options.suffix ?? '';
        
        let sanitizedValue = String(value);
        // 清理 HTML 标签
        sanitizedValue = dompurify.sanitize(value, { ALLOWED_TAGS: [] });
        // 解码 HTML 实体
        sanitizedValue = decode(sanitizedValue);
        // 格式化标题
        const formattedValue = options.format ? formatTitle(sanitizedValue) : sanitizedValue;
        
        return `${prefix}${formattedValue}${suffix}`;
    },
    
    options: ({ field }) => {
        return [
            { field: 'format', name: 'Format', type: 'boolean', ... },
            { field: 'font', name: 'Font', ... },
            { field: 'bold', name: 'Bold', type: 'boolean', ... },
            { field: 'prefix', name: 'Prefix', ... },
            { field: 'suffix', name: 'Suffix', ... },
            // ...
        ];
    },
});
```

---

## 8. 状态同步机制

### 8.1 useSync：双向绑定封装

```typescript
// packages/composables/src/use-sync.ts

/**
 * @deprecated 使用 Vue 3.4+ 的 defineModel() 替代
 * 
 * 创建 computed ref，实现 prop 与 emit 的双向绑定
 */
export function useSync<T, K extends keyof T & string, E extends (event: `update:${K}`, ...args: any[]) => void>(
    props: T,
    key: K,
    emit: E,
): Ref<T[K]> {
    return computed<T[K]>({
        get() {
            return props[key];
        },
        set(newVal) {
            emit(`update:${key}` as const, newVal);
        },
    });
}
```

**在 Layout 中的使用**：

```typescript
// app/src/layouts/tabular/index.ts

setup(props, { emit }) {
    // 创建与父组件的双向绑定
    const selection = useSync(props, 'selection', emit);
    const layoutOptions = useSync(props, 'layoutOptions', emit);
    const layoutQuery = useSync(props, 'layoutQuery', emit);
    
    // 现在可以直接赋值，会自动触发 emit
    // selection.value = [1, 2, 3] → emit('update:selection', [1, 2, 3])
    
    // ...
}
```

### 8.2 syncRefProperty：对象属性同步

```typescript
// app/src/utils/sync-ref-property.ts（推断）

/**
 * 同步 ref 对象中的单个属性
 * 用于从 layoutQuery 这样的对象中提取特定属性
 */
export function syncRefProperty<T, K extends keyof T>(
    objRef: Ref<T>,
    key: K,
    defaultValue: T[K]
): WritableComputedRef<T[K]> {
    return computed({
        get() {
            return objRef.value?.[key] ?? defaultValue;
        },
        set(newVal) {
            objRef.value = {
                ...objRef.value,
                [key]: newVal,
            };
        },
    });
}
```

**使用示例**：

```typescript
// 从 layoutQuery 中同步 page 属性
const page = syncRefProperty(layoutQuery, 'page', 1);

// 读取：page.value → layoutQuery.value.page || 1
// 赋值：page.value = 2 → layoutQuery.value = { ...layoutQuery.value, page: 2 }
```

### 8.3 usePreset：持久化状态管理

```typescript
// app/src/composables/use-preset.ts

export function usePreset(
    collection: Ref<string>,
    bookmark = ref<number | null>(null),
    temporary = false,
): UsablePreset {
    const presetsStore = usePresetsStore();
    const userStore = useUserStore();
    
    const localPreset = ref<Partial<Preset>>({});
    
    // 初始化本地 preset
    function initLocalPreset() {
        const preset = { layout: 'tabular' };
        
        if (bookmark.value === null) {
            // 没有书签：使用集合预设
            assign(preset, presetsStore.getPresetForCollection(collection.value));
        } else if (bookmarkExists.value) {
            // 有书签：使用书签配置
            assign(preset, presetsStore.getBookmark(Number(bookmark.value)));
        }
        
        localPreset.value = preset;
    }
    
    // 响应式属性：从 localPreset 读写
    const layout = computed<string>({
        get: () => localPreset.value.layout || 'tabular',
        set: (layout) => updatePreset({ layout }),
    });
    
    const layoutOptions = computed<Record<string, any>>({
        get() {
            return localPreset.value.layout_options?.[layout.value] || null;
        },
        set(options) {
            const { layout_options } = localPreset.value;
            updatePreset({ 
                layout_options: assign({}, layout_options, { [layout.value]: options }) 
            });
        },
    });
    
    const layoutQuery = computed<Record<string, any>>({
        get() {
            return localPreset.value.layout_query?.[layout.value] || null;
        },
        set(query) {
            const { layout_query } = localPreset.value;
            updatePreset({ 
                layout_query: assign({}, layout_query, { [layout.value]: query }) 
            });
        },
    });
    
    // 自动保存（防抖）
    const autoSave = debounce(async (preset?: Partial<Preset>) => {
        savePreset(preset);
    }, 450);
    
    function updatePreset(preset: Partial<Preset>, immediate?: boolean) {
        localPreset.value = assign({}, localPreset.value, preset);
        
        if (immediate) {
            savePreset();
        } else {
            handleChanges();
        }
    }
    
    function handleChanges() {
        if (bookmarkExists.value) {
            // 书签模式：检查是否与保存的版本一致
            const bookmarkInStore = presetsStore.getBookmark(Number(bookmark.value));
            bookmarkSaved.value = isEqual(localPreset.value, bookmarkInStore);
        } else {
            // 普通模式：自动保存到数据库
            const preset = cloneDeep(localPreset.value);
            autoSave(preset);
        }
    }
    
    return {
        layout,
        layoutOptions,
        layoutQuery,
        filter,
        search,
        refreshInterval,
        savePreset,
        saveCurrentAsBookmark,
        resetPreset,
        // ...
    };
}
```

### 8.4 状态同步数据流图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            状态同步数据流                                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        持久化层（数据库）                              │  │
│  │  ┌─────────────┐    ┌─────────────┐                                 │  │
│  │  │  presets    │    │  bookmarks  │                                 │  │
│  │  │  (directus_ │    │  (directus_ │                                 │  │
│  │  │   presets)  │    │   presets)  │                                 │  │
│  │  └──────┬──────┘    └──────┬──────┘                                 │  │
│  └─────────┼──────────────────┼──────────────────────────────────────────┘  │
│            │                  │                                              │
│            ▼                  ▼                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         Store 层（Pinia）                             │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │ usePresetsStore()                                               │  │  │
│  │  │   - presets: Map<collection, Preset>                           │  │  │
│  │  │   - bookmarks: Map<id, Preset>                                 │  │  │
│  │  │   - hydrate() / dehydrate()                                     │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  └─────────┬────────────────────────────────────────────────────────────┘  │
│            │                                                                  │
│            ▼                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      usePreset() 组合式                               │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │ localPreset: Ref<Partial<Preset>>                              │  │  │
│  │  │                                                                  │  │  │
│  │  │ 计算属性（响应式读写）：                                         │  │  │
│  │  │   - layout        → localPreset.value.layout                    │  │  │
│  │  │   - layoutOptions → localPreset.value.layout_options[layout]  │  │  │
│  │  │   - layoutQuery   → localPreset.value.layout_query[layout]    │  │  │
│  │  │   - filter        → localPreset.value.filter                   │  │  │
│  │  │   - search        → localPreset.value.search                   │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  └─────────┬────────────────────────────────────────────────────────────┘  │
│            │                                                                  │
│            │ v-model:layout, v-model:layout-options, v-model:layout-query  │
│            ▼