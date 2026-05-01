# Directus 管理界面三类扩展注册与加载机制分析报告

## 目录
1. [概述](#1-概述)
2. [核心扩展注册入口](#2-核心扩展注册入口)
3. [三类扩展的定义与注册](#3-三类扩展的定义与注册)
4. [数据流与核心界面协作](#4-数据流与核心界面协作)
5. [总结](#5-总结)

---

## 1. 概述

Directus 管理界面通过强大的扩展机制支持自定义数据展示和编辑组件。本文详细分析了三类核心扩展：

| 扩展类型 | 用途 | 典型场景 |
|---------|------|---------|
| **Layout（布局）** | 定义数据集合的整体展示方式 | 表格、卡片、看板、日历、地图等 |
| **Interface（界面）** | 定义单个字段的编辑组件 | 输入框、选择器、日期选择器、富文本等 |
| **Display（显示）** | 定义单个字段的只读展示组件 | 格式化文本、彩色标签、图片预览等 |

---

## 2. 核心扩展注册入口

### 2.1 扩展加载流程

扩展的加载和注册在 `app/src/extensions.ts` 中统一管理，采用 **"先加载后注册"** 的两阶段模式。

#### 第一阶段：加载扩展

```typescript
// app/src/extensions.ts:31-42
export async function loadExtensions(): Promise<void> {
    try {
        customExtensions = import.meta.env.DEV
            ? await import(/* @vite-ignore */ '@directus-extensions')
            : await import(/* @vite-ignore */ `${getRootPath()}extensions/sources/index.js`);
    } catch (err: any) {
        // eslint-disable-next-line no-console
        console.warn(`Couldn't load extensions`);
        // eslint-disable-next-line no-console
        console.warn(err);
    }
}
```

**关键点**：
- 开发环境从 `@directus-extensions` 虚拟模块加载（Vite 插件提供）
- 生产环境从 `extensions/sources/index.js` 加载
- 使用动态 `import()` 实现懒加载
- 加载失败不中断应用，仅输出警告

#### 第二阶段：注册扩展

```typescript
// app/src/extensions.ts:44-95
export function registerExtensions(app: App): void {
    // 1. 获取内置扩展
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
        // ... 其他扩展类型
    }

    // 3. 注册到 Vue 应用
    registerInterfaces(interfaces, app);
    registerDisplays(displays, app);
    registerLayouts(layouts, app);
    // ...

    // 4. 国际化响应式更新
    watch(
        i18n.global.locale,
        () => {
            extensions.interfaces.value = translate(interfaces);
            extensions.displays.value = translate(displays);
            extensions.layouts.value = translate(layouts);
            // ...
        },
        { immediate: true },
    );
}
```

#### 扩展存储结构

```typescript
// app/src/extensions.ts:16-24
const extensions: RefRecord<AppExtensionConfigs> = {
    interfaces: shallowRef([]),
    displays: shallowRef([]),
    layouts: shallowRef([]),
    modules: shallowRef([]),
    panels: shallowRef([]),
    operations: shallowRef([]),
    themes: shallowRef([]),
};
```

**设计特点**：
- 使用 `shallowRef` 存储扩展配置数组，避免深层响应式开销
- 通过 `useExtensions()` 可全局访问
- 响应式支持：语言切换时自动更新翻译

---

## 3. 三类扩展的定义与注册

### 3.1 扩展定义工具函数

Directus 提供了类型安全的扩展定义函数，位于 `packages/extensions/src/shared/utils/define-extension.ts`：

```typescript
export function defineInterface<Custom extends CustomConfig<InterfaceConfig>>(
    config: ExtendedConfig<InterfaceConfig, Custom>,
): ExtendedConfig<InterfaceConfig, Custom> {
    return config;
}

export function defineDisplay<Custom extends CustomConfig<DisplayConfig>>(
    config: ExtendedConfig<DisplayConfig, Custom>,
): ExtendedConfig<DisplayConfig, Custom> {
    return config;
}

export function defineLayout<Options = any, Query = any>(
    config: LayoutConfig<Options, Query>,
): LayoutConfig<Options, Query> {
    return config;
}
```

**作用**：
- 提供 TypeScript 类型推导和检查
- 支持自定义配置扩展
- 不做任何运行时处理，仅返回原配置

### 3.2 Layout（布局）扩展

#### 配置类型定义

```typescript
// packages/types/src/extensions/layouts.ts
export interface LayoutConfig<Options = any, Query = any> {
    id: string;                          // 唯一标识
    name: string;                        // 显示名称（支持 i18n）
    icon: string;                        // Material Icons 图标名
    component: Component;                // 主布局组件
    slots: {
        options: Component;              // 布局选项面板组件
        sidebar: Component;              // 侧边栏面板组件
        actions: Component;              // 操作栏组件
    };
    smallHeader?: boolean;               // 是否使用小型头部
    headerShadow?: boolean;              // 头部阴影
    sidebarShadow?: boolean;             // 侧边栏阴影
    setup: (props: LayoutProps<Options, Query>, ctx: LayoutContext) 
           => Record<string, unknown>;   // 组合式 setup 函数
}
```

#### LayoutProps 详解

| 属性 | 类型 | 说明 |
|-----|------|-----|
| `collection` | `string \| null` | 当前数据集合名称 |
| `selection` | `(number \| string)[]` | 当前选中的项目主键数组 |
| `layoutOptions` | `Options` | 布局选项（用户配置） |
| `layoutQuery` | `Query` | 布局查询参数（分页、排序等） |
| `filter` / `filterUser` / `filterSystem` | `Filter \| null` | 过滤条件 |
| `search` | `string \| null` | 搜索关键词 |
| `selectMode` | `boolean` | 是否为选择模式 |
| `showSelect` | `'none' \| 'one' \| 'multiple'` | 选择显示类型 |
| `readonly` | `boolean` | 是否只读 |
| `resetPreset` | `() => Promise<void>` | 重置预设函数 |
| `clearFilters` | `() => void` | 清除过滤函数 |

#### LayoutContext 详解

```typescript
interface LayoutContext {
    emit: (event: 'update:selection' | 'update:layoutOptions' | 'update:layoutQuery', 
            ...args: any[]) => void;
}
```

**可触发的事件**：
- `update:selection` - 更新选中项
- `update:layoutOptions` - 更新布局选项
- `update:layoutQuery` - 更新查询参数

#### 内置布局注册机制

```typescript
// app/src/layouts/index.ts
export function getInternalLayouts(): LayoutConfig[] {
    const layouts = import.meta.glob<LayoutConfig>('./*/index.ts', { 
        import: 'default', 
        eager: true 
    });
    return sortBy(Object.values(layouts), 'id');
}

export function registerLayouts(layouts: LayoutConfig[], app: App): void {
    for (const layout of layouts) {
        // 主组件：layout-{id}
        app.component(`layout-${layout.id}`, layout.component);
        // 选项面板：layout-options-{id}
        app.component(`layout-options-${layout.id}`, layout.slots.options);
        // 侧边栏：layout-sidebar-{id}
        app.component(`layout-sidebar-${layout.id}`, layout.slots.sidebar);
        // 操作栏：layout-actions-{id}
        app.component(`layout-actions-${layout.id}`, layout.slots.actions);
    }
}
```

**动态组件命名规范**：
- `layout-{id}` - 主布局组件
- `layout-options-{id}` - 布局选项配置组件
- `layout-sidebar-{id}` - 侧边栏面板组件
- `layout-actions-{id}` - 头部操作栏组件

#### 布局实现示例：Tabular（表格）布局

```typescript
// app/src/layouts/tabular/index.ts
export default defineLayout<LayoutOptions, LayoutQuery>({
    id: 'tabular',
    name: '$t:layouts.tabular.tabular',
    icon: 'table_rows',
    component: TabularLayout,
    slots: {
        options: TabularOptions,
        sidebar: () => undefined,  // 无侧边栏
        actions: TabularActions,
    },
    headerShadow: false,
    setup(props, { emit }) {
        // 1. 响应式同步 props
        const selection = useSync(props, 'selection', emit);
        const layoutOptions = useSync(props, 'layoutOptions', emit);
        const layoutQuery = useSync(props, 'layoutQuery', emit);

        // 2. 解构核心参数
        const { collection, filter, filterSystem, filterUser, search } = toRefs(props);

        // 3. 使用 composables 获取数据
        const { info, primaryKeyField, fields: fieldsInCollection, sortField } 
            = useCollection(collection);
        
        // 4. 使用 useItems 获取数据（核心数据流）
        const {
            items,
            loading,
            error,
            totalPages,
            itemCount,
            totalCount,
            getItems,
            getTotalCount,
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

        // 5. 导出状态和方法供组件使用
        return {
            items,
            loading,
            error,
            tableHeaders,
            onRowClick: onClick,
            refresh: () => { getItems(); getTotalCount(); },
            // ... 更多导出
        };
    },
});
```

### 3.3 Interface（界面）扩展

#### 配置类型定义

```typescript
// packages/types/src/extensions/interfaces.ts
export interface InterfaceConfig {
    id: string;                              // 唯一标识
    name: string;                            // 显示名称
    icon: string;                            // 图标
    description?: string;                    // 描述
    component: Component;                    // 编辑组件
    options:                                 // 配置选项定义
        | DeepPartial<AppField>[]
        | { standard: DeepPartial<AppField>[]; advanced: DeepPartial<AppField>[] }
        | ((ctx: ExtensionOptionsContext) => ...)
        | Exclude<ComponentOptions, any>
        | null;
    types: readonly Type[];                  // 支持的字段类型（如 'string', 'integer'）
    localTypes?: readonly LocalType[];       // 支持的本地类型
    group?: 'standard' | 'selection' | 'relational' | 'presentation' | 'group' | 'other';
    order?: number;                           // 排序权重
    relational?: boolean;                     // 是否为关系型
    hideLabel?: boolean;                      // 是否隐藏标签
    hideLoader?: boolean;                     // 是否隐藏加载状态
    indicatorStyle?: 'active' | 'hidden' | 'muted';
    autoKey?: boolean;
    system?: boolean;                         // 是否为系统内置
    recommendedDisplays?: string[];          // 推荐的显示组件
    preview?: string;                         // 预览 SVG
}
```

#### 内置界面注册机制

```typescript
// app/src/interfaces/index.ts
export function getInternalInterfaces(): InterfaceConfig[] {
    const interfaces = import.meta.glob<InterfaceConfig>(
        ['./*/index.ts', './_system/*/index.ts'], 
        { import: 'default', eager: true }
    );
    return sortBy(Object.values(interfaces), 'id');
}

export function registerInterfaces(interfaces: InterfaceConfig[], app: App): void {
    for (const inter of interfaces) {
        app.component(`interface-${inter.id}`, inter.component);
        
        // 选项非函数、非数组且非 null 时注册为组件
        if (typeof inter.options !== 'function' 
            && Array.isArray(inter.options) === false 
            && inter.options !== null) {
            app.component(`interface-options-${inter.id}`, inter.options);
        }
    }
}
```

#### 界面实现示例：Input 输入框

```typescript
// app/src/interfaces/input/index.ts
export default defineInterface({
    id: 'input',
    name: '$t:interfaces.input.input',
    description: '$t:interfaces.input.description',
    icon: 'text_fields',
    component: InterfaceInput,
    types: ['string', 'uuid', 'bigInteger', 'integer', 'float', 'decimal', 'text'],
    group: 'standard',
    options: ({ field }) => {
        // 根据字段类型返回不同选项
        if (field.type && APP_NUMERIC_TYPES.includes(field.type)) {
            return [
                { field: 'min', name: '$t:interfaces.input.minimum_value', ... },
                { field: 'max', name: '$t:interfaces.input.maximum_value', ... },
                { field: 'step', name: '$t:interfaces.input.step_interval', ... },
                // ...
            ];
        }
        
        return {
            standard: [
                { field: 'placeholder', name: '$t:placeholder', ... },
                { field: 'iconLeft', name: '$t:icon_left', ... },
                { field: 'iconRight', name: '$t:icon_right', ... },
            ],
            advanced: [
                { field: 'softLength', name: '$t:soft_length', ... },
                { field: 'font', name: '$t:font', ... },
                { field: 'trim', name: '$t:interfaces.input.trim', ... },
                // ...
            ],
        };
    },
    preview: PreviewSVG,
});
```

**Interface 组件接收的 Props**（来自 `form-field-interface.vue`）：

| Prop | 类型 | 说明 |
|-----|------|-----|
| `value` | any | 当前字段值 |
| `disabled` | boolean | 是否禁用 |
| `readonly` | boolean | 是否只读 |
| `type` | string | 字段类型 |
| `collection` | string | 所属集合 |
| `field` | string | 字段名 |
| `field-data` | Field | 完整字段配置 |
| `primary-key` | string \| number \| null | 当前记录主键 |
| `width` | string | 显示宽度（'half', 'full' 等） |
| `batch-mode` | boolean | 是否批量编辑模式 |
| `comparison-mode` | boolean | 是否对比模式 |
| `version` | ContentVersion | 内容版本 |

**Interface 组件触发的 Events**：
- `@input` - 值变更时触发
- `@set-field-value` - 设置其他字段值时触发

### 3.4 Display（显示）扩展

#### 配置类型定义

```typescript
// packages/types/src/extensions/displays.ts
export interface DisplayConfig {
    id: string;                              // 唯一标识
    name: string;                            // 显示名称
    icon: string;                            // 图标
    description?: string;                    // 描述
    component: Component;                    // 展示组件
    handler?: (                               // 格式化处理器（可选）
        value: any,
        options: Record<string, any>,
        ctx: { interfaceOptions?: Record<string, any>; field?: Field; collection?: string }
    ) => string | null;
    options:                                 // 配置选项定义
        | DeepPartial<AppField>[]
        | { standard: DeepPartial<AppField>[]; advanced: DeepPartial<AppField>[] }
        | ((ctx: ExtensionOptionsContext) => ...)
        | Exclude<ComponentOptions, any>
        | null;
    types: readonly Type[];                  // 支持的字段类型
    localTypes?: readonly LocalType[];       // 支持的本地类型
    fields?: string[] | DisplayFieldsFunction; // 需要额外加载的关联字段
}
```

#### 内置显示组件注册机制

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
        
        if (typeof display.options !== 'function' 
            && Array.isArray(display.options) === false 
            && display.options !== null) {
            app.component(`display-options-${display.id}`, display.options);
        }
    }
}
```

#### 显示实现示例：Formatted Value（格式化值）

```typescript
// app/src/displays/formatted-value/index.ts
export default defineDisplay({
    id: 'formatted-value',
    name: '$t:displays.formatted-value.formatted-value',
    description: '$t:displays.formatted-value.description',
    types: ['string', 'text', 'integer', 'float', 'decimal', 'bigInteger'],
    icon: 'text_format',
    component: DisplayFormattedValue,
    
    // 可选的格式化处理器：纯文本格式化时使用
    handler: (value, options) => {
        const prefix = options.prefix ?? '';
        const suffix = options.suffix ?? '';
        
        let sanitizedValue = String(value);
        // 清理 HTML 标签
        sanitizedValue = dompurify.sanitize(value, { ALLOWED_TAGS: [] });
        // 解码 HTML 实体
        sanitizedValue = decode(sanitizedValue);
        // 格式化标题（如 "hello_world" -> "Hello World"）
        const formattedValue = options.format ? formatTitle(sanitizedValue) : sanitizedValue;
        
        return `${prefix}${formattedValue}${suffix}`;
    },
    
    options: ({ field }) => {
        const isString = ['string', 'text'].includes(field.type ?? 'unknown');
        const stringOperators = ['eq', 'neq', 'contains', 'starts_with', 'ends_with'];
        const numberOperators = ['eq', 'neq', 'gt', 'gte', 'lt', 'lte'];
        
        return [
            { field: 'format', name: '$t:displays.formatted-value.format', type: 'boolean', ... },
            { field: 'font', name: '$t:displays.formatted-value.font', ... },
            { field: 'bold', name: '$t:displays.formatted-value.bold', type: 'boolean', ... },
            { field: 'italic', name: '$t:displays.formatted-value.italic', type: 'boolean', ... },
            { field: 'prefix', name: '$t:displays.formatted-value.prefix', ... },
            { field: 'suffix', name: '$t:displays.formatted-value.suffix', ... },
            { field: 'color', name: '$t:displays.formatted-value.color', ... },
            { field: 'background', name: '$t:displays.formatted-value.background', ... },
            { field: 'icon', name: '$t:displays.formatted-value.icon', ... },
            { field: 'border', name: '$t:displays.formatted-value.border', type: 'boolean', ... },
            { field: 'masked', name: '$t:displays.formatted-value.mask', type: 'boolean', ... },
            // 条件格式化
            { 
                field: 'conditionalFormatting', 
                type: 'json', 
                name: '$t:conditional_styles',
                meta: {
                    interface: 'list',
                    options: {
                        template: '{{operator}} {{value}}',
                        fields: [
                            { field: 'operator', name: '$t:operator', ... },
                            { field: 'value', name: '$t:value', ... },
                            { field: 'color', name: '$t:displays.formatted-value.color', ... },
                            // ...
                        ],
                    },
                },
            },
        ];
    },
});
```

**Display 组件接收的 Props**（来自 `render-display.vue`）：

| Prop | 类型 | 说明 |
|-----|------|-----|
| `value` | any | 当前字段值 |
| `options` | Record<string, unknown> | 显示配置选项 |
| `interface` | string | 关联的界面组件 ID |
| `interfaceOptions` | Record<string, unknown> | 界面组件配置 |
| `type` | string | 字段类型 |
| `collection` | string | 所属集合 |
| `field` | string | 字段名 |

---

## 4. 数据流与核心界面协作

### 4.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Vue 应用初始化阶段                             │
├─────────────────────────────────────────────────────────────────────┤
│  1. loadExtensions() → 动态导入自定义扩展                            │
│  2. registerExtensions(app)                                           │
│     ├── 获取内置扩展 (getInternal*)                                   │
│     ├── 合并自定义扩展                                                 │
│     ├── 注册为 Vue 组件 (app.component)                               │
│     └── 提供响应式扩展引用 (extensions.*)                             │
│  3. provide(EXTENSIONS_INJECT, extensions)                           │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│                        运行时组件使用阶段                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │   Layout     │    │  Interface   │    │   Display    │         │
│  │  集合视图层   │    │  字段编辑层   │    │  字段展示层   │         │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘         │
│         │                    │                    │                  │
│         ▼                    ▼                    ▼                  │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │              核心 Composables 层                          │       │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐ │       │
│  │  │ useLayout   │  │useExtension │  │    useItems      │ │       │
│  │  │ (包装器)    │  │ (查找扩展)  │  │   (数据获取)     │ │       │
│  │  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘ │       │
│  │         │                 │                    │           │       │
│  │         ▼                 ▼                    ▼           │       │
│  │  ┌──────────────────────────────────────────────────┐    │       │
│  │  │           useExtensions() (注入获取)              │    │       │
│  │  │  ┌──────────────────────────────────────────┐   │    │       │
│  │  │  │  inject<RefRecord<AppExtensionConfigs>>( │   │    │       │
│  │  │  │    EXTENSIONS_INJECT                      │   │    │       │
│  │  │  │  )                                        │   │    │       │
│  │  │  └──────────────────────────────────────────┘   │    │       │
│  │  └──────────────────────────────────────────────────┘    │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Layout 数据流详解

#### 核心包装器机制

Layout 使用 `useLayout` composable 和 `createLayoutWrapper` 函数创建动态包装器，实现标准的 props/emit 接口。

```typescript
// packages/composables/src/use-layout.ts
export function createLayoutWrapper<Options, Query>(layout: LayoutConfig): Component {
    return defineComponent({
        name: `${layout.id}-${NAME_SUFFIX}`, // e.g., "tabular-wrapper"
        
        // 定义标准 props
        props: {
            collection: { type: String, required: true },
            selection: { type: Array, default: () => [] },
            layoutOptions: { type: Object, default: () => ({}) },
            layoutQuery: { type: Object, default: () => ({}) },
            // ... 其他 props
        },
        
        emits: WRITABLE_PROPS.map((prop) => `update:${prop}` as const),
        // 即: ['update:selection', 'update:layoutOptions', 'update:layoutQuery']
        
        setup(props, { emit }) {
            // 1. 调用 layout 的 setup 函数，合并返回值
            const state: Record<string, unknown> = reactive({
                ...layout.setup(props, { emit }),
                ...toRefs(props),
                sidebarShadow: layout.sidebarShadow ?? false,
            });
            
            // 2. 创建 onUpdate 处理器，支持双向绑定
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
        
        // 3. 渲染：通过作用域插槽传递 state
        render(ctx: any) {
            return ctx.$slots.default !== undefined 
                ? ctx.$slots.default({ layoutState: ctx.state }) 
                : null;
        },
    });
}
```

#### useLayout Composable

```typescript
export function useLayout<Options = any, Query = any>(
    layoutId: Ref<string | null>,
): { layoutWrapper: ComputedRef<Component> } {
    const { layouts } = useExtensions();  // 注入获取所有布局
    
    // 为每个布局创建包装器
    const layoutWrappers = computed(() => 
        layouts.value.map((layout) => createLayoutWrapper<Options, Query>(layout))
    );
    
    // 根据 ID 选择对应的包装器，默认返回 tabular
    const layoutWrapper = computed(() => {
        const layout = layoutWrappers.value.find(
            (layout) => layout.name === `${layoutId.value}-${NAME_SUFFIX}`
        );
        
        if (layout === undefined) {
            return layoutWrappers.value.find((layout) => layout.name === `tabular-${NAME_SUFFIX}`)!;
        }
        
        return layout;
    });
    
    return { layoutWrapper };
}
```

#### Layout 在 Collection 页面的使用

```vue
<!-- app/src/modules/content/routes/collection.vue -->
<script setup lang="ts">
// 1. 使用 usePreset 管理布局状态（来自 URL/书签）
const {
    layout,           // 当前布局 ID: Ref<string>
    layoutOptions,    // 布局选项
    layoutQuery,      // 查询参数
    filter,           // 用户过滤
    search,           // 搜索词
    resetPreset,      // 重置预设函数
} = usePreset(collection, bookmarkID);

// 2. 获取布局包装器组件
const { layoutWrapper } = useLayout(layout);

// 3. 定义其他状态
const selection = ref<Item[]>([]);  // 选中项
const layoutRef = ref();             // 布局组件引用

// 4. 刷新数据（通过调用 layout 导出的方法）
async function refresh() {
    await layoutRef.value?.state?.refresh?.();
}
</script>

<template>
    <!-- 5. 使用动态组件渲染布局 -->
    <component
        :is="layoutWrapper"
        ref="layoutRef"
        v-slot="{ layoutState }"
        v-model:selection="selection"
        v-model:layout-options="layoutOptions"
        v-model:layout-query="layoutQuery"
        :filter-user="filter"
        :filter-system="archiveFilter"
        :filter="mergeFilters(filter, archiveFilter)"
        :search="search"
        :collection="collection"
        :reset-preset="resetPreset"
        :clear-filters="clearFilters"
    >
        <!-- 6. 在插槽中使用 layoutState 渲染子组件 -->
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
                    <!-- 布局选项：动态组件 -->
                    <component :is="`layout-options-${layout || 'tabular'}`" v-bind="layoutState" />
                </LayoutSidebarDetail>
                <!-- 侧边栏内容：动态组件 -->
                <component :is="`layout-sidebar-${layout || 'tabular'}`" v-bind="layoutState" />
            </template>
        </PrivateView>
    </component>
</template>
```

**关键点**：
1. `v-model:selection` 等使用 `v-model` 指令实现双向绑定
2. `layoutState` 包含 layout setup 返回的所有属性和方法
3. 四个动态组件（`layout-{id}`、`layout-actions-{id}`、`layout-options-{id}`、`layout-sidebar-{id}`）共同构成完整布局

### 4.3 Interface 数据流详解

#### Interface 查找机制

```typescript
// app/src/composables/use-extension.ts
export function useExtension<T extends AppExtensionType | HybridExtensionType>(
    type: T | Ref<T>,
    name: string | Ref<string | null>,
): Ref<AppExtensionConfigs[Plural<T>][number] | null> {
    const extensions = useExtensions();
    
    return computed(() => {
        if (unref(name) === null) return null;
        // pluralize('interface') → 'interfaces'
        // 根据 ID 在 extensions.interfaces 中查找
        return (extensions[pluralize(unref(type))].value as any[])
            .find(({ id }) => id === unref(name)) ?? null;
    });
}
```

#### Interface 在表单中的使用

```vue
<!-- app/src/components/v-form/components/form-field-interface.vue -->
<script setup lang="ts">
const props = defineProps<{
    field: FormField;           // 字段配置
    modelValue?: any;           // 当前值
    primaryKey?: string | number | null;
    loading?: boolean;
    disabled?: boolean;
    // ...
}>();

defineEmits(['update:modelValue', 'setFieldValue']);

// 1. 查找对应的 Interface 配置
const inter = useExtension(
    'interface',
    computed(() => props.field?.meta?.interface ?? 'input'),
);

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
        
        <!-- 动态渲染 Interface 组件 -->
        <VErrorBoundary v-if="interfaceExists && !rawEditorActive" :name="componentName">
            <component
                :is="componentName"
                <!-- 传递 meta.options 配置 -->
                v-bind="(field.meta && field.meta.options) || {}"
                <!-- 传递标准 props -->
                :autofocus="disabled !== true && autofocus"
                :disabled="disabled"
                :non-editable="nonEditable"
                :loading="loading"
                :value="value"
                :batch-mode="batchMode"
                :batch-active="batchActive"
                :comparison-mode="!!comparison"
                :type="field.type"
                :collection="field.collection"
                :field="field.field"
                :field-data="field"
                :primary-key="primaryKey"
                :length="field.schema && field.schema.max_length"
                :direction="direction"
                :version="version"
                <!-- 监听事件 -->
                @input="$emit('update:modelValue', $event)"
                @set-field-value="$emit('setFieldValue', $event)"
            />
            
            <template #fallback>
                <VNotice type="warning">{{ $t('unexpected_error') }}</VNotice>
            </template>
        </VErrorBoundary>
        
        <!-- 原始编辑器 -->
        <InterfaceSystemRawEditor
            v-else-if="rawEditorEnabled && rawEditorActive"
            :value="value"
            :type="field.type"
            @input="$emit('update:modelValue', $event)"
        />
        
        <!-- 未找到提示 -->
        <VNotice v-else type="warning">
            {{ $t('interface_not_found', { interface: field.meta && field.meta.interface }) }}
        </VNotice>
    </div>
</template>
```

**Interface 数据流总结**：
```
父组件 (v-form)
    │
    │ modelValue, field config
    ▼
form-field-interface.vue
    │
    │ 1. useExtension('interface', field.meta.interface)
    │    └─► 从 extensions.interfaces 查找配置
    │
    │ 2. 构建组件名: `interface-${id}`
    │
    │ 3. 动态渲染 <component :is="componentName" ... />
    │    ├── props: value, disabled, type, collection, field, field-data, ...
    │    └── events: @input → update:modelValue, @set-field-value
    │
    ▼
Interface 组件 (如 interface-input)
    │
    ├── 读取 props.value
    ├── 读取 props.field.meta.options 配置
    ├── 用户编辑
    └── emit('input', newValue) → 触发父组件更新
```

### 4.4 Display 数据流详解

#### Display 在列表中的使用

```vue
<!-- app/src/views/private/components/render-display.vue -->
<script setup lang="ts">
const props = defineProps<{
    display: string | null;              // Display ID
    options?: Record<string, unknown>;   // 显示配置选项
    interface?: string;                   // 关联的 Interface ID
    interfaceOptions?: Record<string, unknown>;
    value?: any;                          // 当前值
    type: string;                         // 字段类型
    collection: string;                   // 集合
    field: string;                        // 字段名
}>();

const { display } = toRefs(props);

// 1. 查找 Display 配置
const displayInfo = useExtension('display', display);
</script>

<template>
    <!-- 值为 null 时显示占位 -->
    <ValueNull v-if="value === null || value === undefined" />
    
    <!-- 未找到 Display 时显示原始文本 -->
    <VTextOverflow v-else-if="displayInfo === null" class="display" :text="value" />
    
    <!-- 动态渲染 Display 组件 -->
    <VErrorBoundary v-else :name="`display-${display}`">
        <component
            :is="`display-${display}`"
            v-bind="options"           <!-- 显示选项 -->
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

#### Display 在 Tabular Layout 中的集成

```typescript
// app/src/layouts/tabular/index.ts (简化版)
function useTable() {
    const tableHeaders = computed<HeaderRaw[]>({
        get() {
            return activeFields.value.map((field) => {
                return {
                    text: field.name,
                    value: field.key,
                    // ...
                    field: {
                        // 关键：从字段元数据获取 display 配置
                        display: field.meta?.display || getDefaultDisplayForType(field.type),
                        displayOptions: field.meta?.display_options,
                        interface: field.meta?.interface,
                        interfaceOptions: field.meta?.options,
                        type: field.type,
                        field: field.field,
                        collection: field.collection,
                    },
                    // ...
                } as HeaderRaw;
            });
        },
        // ...
    });
}
```

在 Tabular 组件的表格单元格中：
```vue
<!-- 伪代码示意 -->
<v-table>
    <template #cell="{ item, header }">
        <!-- header.field 包含 display 配置 -->
        <render-display
            :display="header.field.display"
            :options="header.field.displayOptions"
            :interface="header.field.interface"
            :interface-options="header.field.interfaceOptions"
            :value="item[header.field.field]"
            :type="header.field.type"
            :collection="header.field.collection"
            :field="header.field.field"
        />
    </template>
</v-table>
```

**Display 数据流总结**：
```
Layout (如 tabular)
    │
    │ 从 field.meta 获取: display, display_options
    ▼
render-display.vue
    │
    │ 1. useExtension('display', displayId)
    │    └─► 从 extensions.displays 查找配置
    │
    │ 2. 构建组件名: `display-${id}`
    │
    │ 3. 动态渲染 <component :is="componentName" ... />
    │    └── props: value, options, type, collection, field, ...
    │
    ▼
Display 组件 (如 display-formatted-value)
    │
    ├── 读取 props.value
    ├── 读取 props.options (格式化配置)
    ├── 可选：调用 handler() 格式化纯文本
    └── 渲染为 Vue 组件（支持图标、颜色、样式等）
```

### 4.5 三类扩展对比总结

| 维度 | Layout | Interface | Display |
|-----|--------|-----------|---------|
| **职责** | 集合级数据展示与交互 | 单字段编辑 | 单字段只读展示 |
| **作用范围** | 整个页面/集合 | 单个表单字段 | 列表/详情中的单个字段 |
| **数据获取** | 通过 `useItems` 自主获取 | 从表单接收 value | 从父组件接收 value |
| **状态管理** | 复杂：selection, pagination, filters | 简单：value 双向绑定 | 简单：value 只读 |
| **组件数量** | 4 个（主组件 + 3 个插槽组件） | 1-2 个（主组件 + 可选选项组件） | 1-2 个（主组件 + 可选选项组件） |
| **典型配置** | layoutOptions, layoutQuery | field.meta.options | field.meta.display_options |
| **触发更新** | emit('update:selection/layoutOptions/layoutQuery') | emit('input', value) | 无（只读） |

---

## 5. 总结

### 5.1 注册加载流程总览

```
┌────────────────────────────────────────────────────────────────┐
│                        应用启动阶段                              │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  1. loadExtensions()                                           │
│     ├── 开发环境: import('@directus-extensions')             │
│     └── 生产环境: import(`${rootPath}extensions/sources/index.js`)│
│                                                                │
│  2. registerExtensions(app)                                    │
│     ├── getInternal*() ──► Vite import.meta.glob() 扫描目录  │
│     ├── 合并 customExtensions.interfaces/displays/layouts     │
│     ├── register*() ──► app.component(`{type}-{id}`, ...)   │
│     └── provide(EXTENSIONS_INJECT, extensions)               │
│                                                                │
└────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────────┐
│                        运行时阶段                                │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  组件使用流程:                                                  │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ 1. useExtensions() ──► inject(EXTENSIONS_INJECT)        ││
│  │    └─► 获取响应式扩展引用                                 ││
│  └──────────────────────────────────────────────────────────┘│
│                              │                                 │
│                              ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ 2. useExtension(type, name)                               ││
│  │    └─► 根据 ID 在对应类型数组中 find()                    ││
│  └──────────────────────────────────────────────────────────┘│
│                              │                                 │
│                              ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ 3. 构建组件名: `${type}-${id}`                            ││
│  │    └─► e.g., 'layout-tabular', 'interface-input'         ││
│  └──────────────────────────────────────────────────────────┘│
│                              │                                 │
│                              ▼                                 │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ 4. 动态渲染: <component :is="componentName" ... />       ││
│  │    ├── props: 标准属性 + 扩展配置                          ││
│  │    └── events: v-model 或 @input 监听                     ││
│  └──────────────────────────────────────────────────────────┘│
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 5.2 核心设计模式

1. **配置驱动**：所有扩展通过配置对象定义，组件注册全自动化
2. **约定优于配置**：组件命名遵循 `${type}-${id}` 规范
3. **依赖注入**：通过 Vue 的 `provide/inject` 实现全局扩展访问
4. **响应式设计**：扩展列表使用 `shallowRef`，支持语言切换时自动更新翻译
5. **容错机制**：
   - 扩展加载失败不中断应用
   - 组件渲染错误使用 `VErrorBoundary` 捕获并降级
   - 未找到扩展时有默认 fallback（如 layout 默认 tabular）

### 5.3 扩展开发指南（基于分析）

#### 创建自定义 Layout
```typescript
// 1. 使用 defineLayout 定义
export default defineLayout<MyOptions, MyQuery>({
    id: 'my-custom-layout',
    name: 'My Layout',
    icon: 'dashboard',
    component: MyLayoutComponent,
    slots: {
        options: MyLayoutOptions,
        sidebar: () => undefined,
        actions: MyLayoutActions,
    },
    setup(props, { emit }) {
        // 使用 useItems 获取数据
        const { items, loading, getItems } = useItems(props.collection, {
            // 从 props.layoutQuery 获取分页、排序等
        });
        
        // 使用 useSync 同步可写属性
        const selection = useSync(props, 'selection', emit);
        
        return {
            items,
            loading,
            refresh: getItems,
            // ...
        };
    },
});
```

#### 创建自定义 Interface
```typescript
export default defineInterface({
    id: 'my-custom-input',
    name: 'My Custom Input',
    icon: 'edit',
    component: MyInputComponent,
    types: ['string', 'text'],
    group: 'standard',
    options: [
        { field: 'placeholder', name: 'Placeholder', meta: { interface: 'input' } },
        // ...
    ],
});

// Interface 组件中
// props: { value, disabled, type, collection, field, field-data, ... }
// emit: 'input' (值变更)
```

#### 创建自定义 Display
```typescript
export default defineDisplay({
    id: 'my-custom-display',
    name: 'My Custom Display',
    icon: 'visibility',
    component: MyDisplayComponent,
    types: ['string', 'integer'],
    handler: (value, options) => {
        // 可选：纯文本格式化
        return options.prefix + value + options.suffix;
    },
    options: [
        { field: 'prefix', name: 'Prefix', meta: { interface: 'input' } },
        // ...
    ],
});
```

### 5.4 关键文件索引

| 功能 | 文件路径 |
|-----|---------|
| 扩展注册入口 | `app/src/extensions.ts` |
| 内置 Layout 注册 | `app/src/layouts/index.ts` |
| 内置 Interface 注册 | `app/src/interfaces/index.ts` |
| 内置 Display 注册 | `app/src/displays/index.ts` |
| 扩展定义函数 | `packages/extensions/src/shared/utils/define-extension.ts` |
| Layout 类型定义 | `packages/types/src/extensions/layouts.ts` |
| Interface 类型定义 | `packages/types/src/extensions/interfaces.ts` |
| Display 类型定义 | `packages/types/src/extensions/displays.ts` |
| Layout 包装器 | `packages/composables/src/use-layout.ts` |
| 扩展查找 | `app/src/composables/use-extension.ts` |
| 全局扩展注入 | `packages/composables/src/use-system.ts` |
| Collection 页面使用 Layout | `app/src/modules/content/routes/collection.vue` |
| 表单使用 Interface | `app/src/components/v-form/components/form-field-interface.vue` |
| 列表使用 Display | `app/src/views/private/components/render-display.vue` |
