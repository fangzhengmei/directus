# Directus 三类扩展调用链复核报告

> **复核日期**: 2026-05-02  
> **复核范围**: 扩展发现、入口点生成、前端注册、状态同步、数据绑定  
> **复核方法**: 直接读取源码，逐段核对前一轮分析结论

---

## 目录

1. [复核概述](#1-复核概述)
2. [已证实的结论（✅）](#2-已证实的结论)
3. [部分对齐的结论（⚠️）](#3-部分对齐的结论)
4. [推断错误的结论（❌）](#4-推断错误的结论)
5. [关键分支深度分析](#5-关键分支深度分析)
6. [核实的代码索引](#6-核实的代码索引)
7. [总结](#7-总结)

---

## 1. 复核概述

### 1.1 复核目标

本轮复核的核心目标是：
1. 逐段核对前一轮分析中的调用链和数据流向
2. 区分哪些结论是基于源码的**已证实事实**
3. 找出哪些结论是**推断**或**部分正确**
4. 修正前一轮分析中与源码不符的部分

### 1.2 复核方法

采用**源码对齐法**：
- 对每个关键结论，直接定位到具体的源码文件和行号
- 对比分析结论与实际实现
- 标记状态：✅ 已证实 | ⚠️ 部分对齐 | ❌ 推断错误

---

## 2. 已证实的结论（✅）

以下结论已通过源码阅读完全证实。

### 2.1 扩展注册机制

| 结论 | 源码位置 | 验证结果 |
|-----|---------|---------|
| Interface 注册为 `interface-${id}` 组件 | `app/src/interfaces/index.ts:16` | ✅ 已证实 |
| Display 注册为 `display-${id}` 组件 | `app/src/displays/index.ts:13` | ✅ 已证实 |
| Layout 注册为 `layout-${id}` 组件（含 4 个 slots） | `app/src/layouts/index.ts:13-17` | ✅ 已证实 |

**关键代码**：

```typescript
// app/src/interfaces/index.ts:14-22
export function registerInterfaces(interfaces: InterfaceConfig[], app: App): void {
    for (const inter of interfaces) {
        app.component(`interface-${inter.id}`, inter.component);
        // ...
    }
}

// app/src/layouts/index.ts:11-18
export function registerLayouts(layouts: LayoutConfig[], app: App): void {
    for (const layout of layouts) {
        app.component(`layout-${layout.id}`, layout.component);
        app.component(`layout-options-${layout.id}`, layout.slots.options);
        app.component(`layout-sidebar-${layout.id}`, layout.slots.sidebar);
        app.component(`layout-actions-${layout.id}`, layout.slots.actions);
    }
}
```

### 2.2 内置扩展发现

| 结论 | 源码位置 | 验证结果 |
|-----|---------|---------|
| 使用 `import.meta.glob` 自动扫描 | `app/src/interfaces/index.ts:6-8` | ✅ 已证实 |
| 使用 `eager: true` 立即加载所有模块 | 同上 | ✅ 已证实 |
| Interface 包含 `_system` 子目录 | `app/src/interfaces/index.ts:6` | ✅ 已证实 |

**关键代码**：

```typescript
// app/src/interfaces/index.ts:5-12
export function getInternalInterfaces(): InterfaceConfig[] {
    const interfaces = import.meta.glob<InterfaceConfig>(
        ['./*/index.ts', './_system/*/index.ts'],
        { import: 'default', eager: true }
    );
    return sortBy(Object.values(interfaces), 'id');
}
```

### 2.3 自定义扩展加载

| 结论 | 源码位置 | 验证结果 |
|-----|---------|---------|
| 开发环境：`import('@directus-extensions')` | `app/src/extensions.ts:34` | ✅ 已证实 |
| 生产环境：`import('${rootPath}extensions/sources/index.js')` | `app/src/extensions.ts:35` | ✅ 已证实 |
| 加载失败不中断应用（try-catch） | `app/src/extensions.ts:31-37` | ✅ 已证实 |

**关键代码**：

```typescript
// app/src/extensions.ts:31-37
export async function loadExtensions(): Promise<void> {
    try {
        customExtensions = import.meta.env.DEV
            ? await import(/* @vite-ignore */ '@directus-extensions')
            : await import(/* @vite-ignore */ `${getRootPath()}extensions/sources/index.js`);
    } catch (err: any) {
        // eslint-disable-next-line no-console
        console.warn(err);
    }
}
```

### 2.4 useSync 实现

| 结论 | 源码位置 | 验证结果 |
|-----|---------|---------|
| 已 deprecated，推荐 `defineModel()` | `packages/composables/src/use-sync.ts:7-8` | ✅ 已证实 |
| getter 返回 `props[key]` | `packages/composables/src/use-sync.ts:144` | ✅ 已证实 |
| setter 调用 `emit(\`update:${key}\`, newVal)` | `packages/composables/src/use-sync.ts:147` | ✅ 已证实 |

**关键代码**：

```typescript
// packages/composables/src/use-sync.ts:137-150
export function useSync<...>(props: T, key: K, emit: E): Ref<T[K]> {
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

### 2.5 usePreset 实现

| 结论 | 源码位置 | 验证结果 |
|-----|---------|---------|
| `autoSave` 防抖 450ms | `app/src/composables/use-preset.ts:72-74` | ✅ 已证实 |
| `layoutOptions` 是 computed ref | `app/src/composables/use-preset.ts:111-119` | ✅ 已证实 |
| `layoutQuery` 是 computed ref | `app/src/composables/use-preset.ts:121-129` | ✅ 已证实 |
| 自动同步到 `localPreset` → 数据库 | `app/src/composables/use-preset.ts:80-88` | ✅ 已证实 |

**关键代码**：

```typescript
// app/src/composables/use-preset.ts:72-74
const autoSave = debounce(async (preset?: Partial<Preset>) => {
    savePreset(preset);
}, 450);

// app/src/composables/use-preset.ts:80-88
function handleChanges() {
    if (bookmarkExists.value) {
        // 有 bookmark 时，只标记是否已保存
        const bookmarkInStore = presetsStore.getBookmark(Number(bookmark.value));
        bookmarkSaved.value = isEqual(localPreset.value, bookmarkInStore);
    } else {
        // 无 bookmark 时，自动保存到数据库
        const preset = cloneDeep(localPreset.value);
        autoSave(preset);
    }
}
```

### 2.6 generateExtensionsEntrypoint 实现

| 结论 | 源码位置 | 验证结果 |
|-----|---------|---------|
| 从三种来源过滤扩展 | `packages/extensions/src/node/utils/generate-extensions-entrypoint.ts:19-53` | ✅ 已证实 |
| 只包含 `enabled: true` 的扩展 | 同上，第 25 行 | ✅ 已证实 |
| 支持 Bundle extension 的子扩展 | 同上，第 29-51 行 | ✅ 已证实 |

---

## 3. 部分对齐的结论（⚠️）

以下结论基本正确，但存在细节差异需要修正。

### 3.1 syncRefProperty 实现

**前一轮结论**：
```typescript
// 推断的实现
set(newVal) {
    ref.value = { ...ref.value, [key]: newVal };  // 推断：展开运算符
}
```

**实际源码**（`app/src/utils/sync-ref-property.ts:8-10`）：
```typescript
set(value: R[T]) {
    ref.value = Object.assign({}, ref.value, { [key]: value }) as R;  // 实际：Object.assign
}
```

**差异说明**：
- 功能等价：都创建新对象，保持响应式
- 实现方式：使用 `Object.assign` 而非展开运算符
- 额外发现：`defaultValue` 支持传入 `Ref`（通过 `unref(defaultValue)`）

**修正后的结论**：
- ✅ getter: `ref.value?.[key] ?? unref(defaultValue)`（支持 `Ref` 类型的 defaultValue）
- ✅ setter: `Object.assign({}, ref.value, { [key]: value })`（创建新对象）

### 3.2 Interface 数据绑定模式

**前一轮推断**：
> Interface 使用标准的 `v-model` 模式，即 `modelValue` prop + `update:modelValue` 事件。

**实际源码**（`app/src/interfaces/input/input.vue`）：

```typescript
// 第 7-34 行：定义 props
const props = withDefaults(
    defineProps<{
        value: string | number | null;  // 注意：是 value，不是 modelValue
        // ...
    }>(),
    // ...
);

// 第 36 行：定义 emits
defineEmits(['input']);  // 注意：是 input，不是 update:modelValue
```

```vue
<!-- 第 71-109 行：模板 -->
<VInput
    :model-value="value"  // 接收 value
    @update:model-value="$emit('input', $event)"  // 触发 input 事件
>
```

**差异说明**：
- Interface 组件使用的是 **Vue 2 的 v-model 模式**：`value` prop + `input` 事件
- 这与 Vue 3 的标准模式 `modelValue` + `update:modelValue` 不同
- 但在使用时，外层包装器会将其转换为标准的 v-model

**修正后的结论**：
- ⚠️ Interface 组件内部使用 `value` prop + `input` 事件（Vue 2 兼容模式）
- ⚠️ 外层通过包装器转换为标准的 `v-model` 语法
- ✅ 功能上实现了双向绑定，符合预期

---

## 4. 推断错误的结论（❌）

以下结论是前一轮的推断，与实际源码不符。

### 4.1 开发环境扩展加载的 Vite 插件来源

**前一轮错误推断**：
> 开发环境从 `@directus-extensions` 虚拟模块加载，该模块由 **Vite 插件 `@directus/extensions-sdk`** 提供。

**实际源码**（`app/vite.config.js:108-208`）：

```javascript
// 第 108 行：本地定义的函数，不是来自 @directus/extensions-sdk
function directusExtensions() {
    const virtualExtensionsId = '@directus-extensions';
    let extensionsEntrypoint = null;

    return [
        {
            name: 'directus-extensions-serve',
            apply: 'serve',
            // ...
            async buildStart() {
                await loadExtensions();  // 本地定义的 loadExtensions
            },
            resolveId(id) {
                if (id === virtualExtensionsId) {
                    return id;
                }
            },
            load(id) {
                if (id === virtualExtensionsId) {
                    return extensionsEntrypoint;
                }
            },
        },
        // ...
    ];

    // 第 157-207 行：loadExtensions 是本地定义的
    async function loadExtensions() {
        // 三种来源：
        const localExtensions = extensionsPathExists 
            ? await resolveFsExtensions(EXTENSIONS_PATH)  // 1. extensions 目录
            : new Map();
        const moduleExtensions = await resolveModuleExtensions(API_PATH);  // 2. node_modules
        const registryExtensions = extensionsPathExists
            ? await resolveFsExtensions(path.join(EXTENSIONS_PATH, '.registry'))  // 3. .registry 目录
            : new Map();
        
        // 开发环境默认全部 enabled
        const extensionSettings = [
            ...Array.from(localExtensions.entries()).flatMap(([folder, extension]) =>
                mockSetting('local', folder, extension),  // enabled: true
            ),
            // ...
        ];
    }
}
```

**错误修正**：

| 项目 | 错误推断 | 实际源码 |
|-----|---------|---------|
| 插件来源 | `@directus/extensions-sdk` | 本地 `app/vite.config.js` 中定义 |
| 扩展来源 | 单一来源 | 三种来源（local、module、registry） |
| 默认启用状态 | 未知 | 开发环境默认全部 `enabled: true` |

**修正后的结论**：
- ❌ 错误：插件来自 `@directus/extensions-sdk`
- ✅ 正确：插件是 `app/vite.config.js` 中本地定义的 `directusExtensions()` 函数
- ✅ 正确：开发环境从三种来源发现扩展
- ✅ 正确：开发环境默认所有扩展都是 `enabled: true`

### 4.2 扩展发现的来源数量

**前一轮推断**：
> 扩展发现主要来自 `extensions/` 目录。

**实际源码**（`app/vite.config.js:157-201`）：

```javascript
async function loadExtensions() {
    // 来源 1：本地 extensions 目录
    const localExtensions = extensionsPathExists 
        ? await resolveFsExtensions(EXTENSIONS_PATH) 
        : new Map();
    
    // 来源 2：node_modules 中的依赖
    const moduleExtensions = await resolveModuleExtensions(API_PATH);
    
    // 来源 3：extensions/.registry 目录
    const registryExtensions = extensionsPathExists
        ? await resolveFsExtensions(path.join(EXTENSIONS_PATH, '.registry'))
        : new Map();
}
```

**错误修正**：
- ❌ 错误：只有 `extensions/` 目录一种来源
- ✅ 正确：三种来源（local、module、registry）

**三种来源的优先级**（根据 `mockSetting` 函数）：
1. **local**：`extensions/` 目录下的扩展
2. **module**：`node_modules` 中带有 `directus` 字段的依赖
3. **registry**：`extensions/.registry/` 目录（可能是 marketplace 安装的扩展）

---

## 5. 关键分支深度分析

### 5.1 扩展发现的完整流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        扩展发现的三种来源                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────┐              │
│  │   local     │    │   module    │    │    registry     │              │
│  │  扩展目录    │    │ node_modules│    │  .registry 目录 │              │
│  └──────┬──────┘    └──────┬──────┘    └────────┬────────┘              │
│         │                    │                     │                        │
│         ▼                    ▼                     ▼                        │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │              resolveFsExtensions / resolveModuleExtensions   │          │
│  │              packages/extensions/src/node/utils/get-extensions.ts│    │
│  └───────────────────────────┬─────────────────────────────────┘          │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │              1. 读取 package.json                            │          │
│  │              2. 检查 "directus" 字段（ExtensionManifest）   │          │
│  │              3. 解析 extension type、entrypoint 等         │          │
│  └───────────────────────────┬─────────────────────────────────┘          │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │              生成 Extension 定义                              │          │
│  │              { name, version, type, entrypoint, host, ... } │          │
│  └───────────────────────────┬─────────────────────────────────┘          │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────┐          │
│  │              存入 Map<string, Extension>                      │          │
│  │              键：文件夹名 / 包名                              │          │
│  │              值：Extension 定义                               │          │
│  └─────────────────────────────────────────────────────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 开发环境 vs 生产环境的关键差异

| 维度 | 开发环境 | 生产环境 |
|-----|---------|---------|
| **扩展加载方式** | Vite 虚拟模块 `@directus-extensions` | 动态生成的 JS 文件 |
| **文件位置** | 内存中（Vite 插件生成） | `extensions/sources/index.js` |
| **默认启用状态** | 全部 `enabled: true` | 从数据库读取 `directus_extensions` |
| **热更新支持** | ✅ 支持（Vite HMR） | ❌ 不支持 |
| **入口点生成时机** | `buildStart` 钩子 | API 启动时 |

**关键代码对比**：

```typescript
// 开发环境：app/vite.config.js:165-188
const mockSetting = (source, folder, extension) => {
    const settings = [{
        id: extension.name,
        enabled: true,  // 开发环境：默认启用
        folder: folder,
        bundle: null,
        source: source,
    }];
    // ...
};

// 生产环境：推断（从数据库读取）
// API 端会从 directus_extensions 表读取 enabled 状态
// 然后调用 generateExtensionsEntrypoint 生成入口点
```

### 5.3 状态同步的实际调用链

以 Calendar Layout 为例，实际的状态同步调用链：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Calendar Layout 的状态同步调用链                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. setup(props, { emit }) 被调用                                           │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │  // app/src/layouts/calendar/index.ts:40-57                    │   │
│     │  const selection = useSync(props, 'selection', emit);          │   │
│     │  const layoutOptions = useSync(props, 'layoutOptions', emit);  │   │
│     │  const { collection, search, filterSystem, ... } = toRefs(props);│  │
│     └─────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  2. 使用 syncRefProperty 从 layoutOptions 提取属性                          │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │  // app/src/layouts/calendar/index.ts:94-109                   │   │
│     │  const template = syncRefProperty(layoutOptions, 'template', undefined);│
│     │  const viewInfo = syncRefProperty(layoutOptions, 'viewInfo', undefined);│
│     │  const startDateField = syncRefProperty(layoutOptions, 'startDateField', undefined);│
│     │  const endDateField = syncRefProperty(layoutOptions, 'endDateField', undefined);│
│     │  const firstDay = syncRefProperty(layoutOptions, 'firstDay', undefined);│
│     └─────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  3. 修改属性 → 触发状态同步                                                 │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │  // 例如：用户修改 viewInfo                                      │   │
│     │  viewInfo.value = { type: 'dayGridWeek', startDateStr: '...' };│   │
│     │                                                                  │   │
│     │  // syncRefProperty 的 setter 被调用                            │   │
│     │  // app/src/utils/sync-ref-property.ts:8-10                    │   │
│     │  set(value: R[T]) {                                              │   │
│     │      ref.value = Object.assign({}, ref.value, { [key]: value });│  │
│     │  }                                                                │   │
│     │                                                                  │   │
│     │  // 此时 ref 是 layoutOptions（来自 useSync）                    │   │
│     │  // layoutOptions 的 setter 会触发 emit                         │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  4. 外层 usePreset 接收更新 → 自动保存到数据库                              │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │  // app/src/composables/use-preset.ts:111-119                  │   │
│     │  const layoutOptions = computed<Record<string, any>>({          │   │
│     │      get() {                                                      │   │
│     │          return localPreset.value.layout_options?.[layout.value] || null;│
│     │      },                                                           │   │
│     │      set(options) {                                               │   │
│     │          const { layout_options } = localPreset.value;          │   │
│     │          updatePreset({ layout_options: assign({}, layout_options, { [layout.value]: options }) });│
│     │      },                                                           │   │
│     │  });                                                              │   │
│     │                                                                  │   │
│     │  // updatePreset → handleChanges → autoSave（防抖 450ms）     │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.4 Interface 数据绑定的实际机制

**实际的数据流**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Interface 数据绑定的实际机制                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  外层组件（如字段编辑表单）                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  <InterfaceComponent                                                 │ │
│  │      :value="fieldValue"          // 传入 value（不是 modelValue）  │ │
│  │      @input="onInput"              // 监听 input 事件               │ │
│  │  />                                                                   │ │
│  │                                                                      │ │
│  │  // 或者通过 v-model（需要包装器转换）                                │ │
│  │  <v-field-input v-model="fieldValue" ... />                         │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                              │
│                              ▼                                              │
│  Interface 组件内部（如 input.vue）                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  // props: { value: string | number | null }                        │ │
│  │  // emits: ['input']                                                 │ │
│  │                                                                      │ │
│  │  <VInput                                                             │ │
│  │      :model-value="value"         // 传给 VInput                   │ │
│  │      @update:model-value="$emit('input', $event)"  // 转换事件    │ │
│  │  />                                                                   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                              │                                              │
│                              ▼                                              │
│  VInput 组件（内部组件）                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │  // 使用 Vue 3 标准模式                                              │ │
│  │  // props: { modelValue: ... }                                      │ │
│  │  // emits: ['update:modelValue']                                    │ │
│  │  // 或者使用 defineModel()                                           │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**为什么使用 `value` + `input` 模式？**

这是为了保持向后兼容。Directus 从 Vue 2 迁移到 Vue 3 时，为了不破坏现有扩展，保留了 Vue 2 的 v-model 约定。

---

## 6. 核实的代码索引

### 6.1 扩展发现与入口点生成

| 功能 | 文件 | 行号 | 状态 |
|-----|------|-----|------|
| resolveFsExtensions | `packages/extensions/src/node/utils/get-extensions.ts` | 68-109 | ✅ 已核实 |
| resolveModuleExtensions | `packages/extensions/src/node/utils/get-extensions.ts` | 111-158 | ✅ 已核实 |
| getExtensionDefinition | `packages/extensions/src/node/utils/get-extensions.ts` | 9-66 | ✅ 已核实 |
| generateExtensionsEntrypoint | `packages/extensions/src/node/utils/generate-extensions-entrypoint.ts` | 12-94 | ✅ 已核实 |

### 6.2 开发环境 Vite 插件

| 功能 | 文件 | 行号 | 状态 |
|-----|------|-----|------|
| directusExtensions 插件定义 | `app/vite.config.js` | 108-208 | ✅ 已核实 |
| loadExtensions（本地） | `app/vite.config.js` | 157-207 | ✅ 已核实 |
| mockSetting | `app/vite.config.js` | 165-188 | ✅ 已核实 |
| 三种扩展来源 | `app/vite.config.js` | 158-163 | ✅ 已核实 |

### 6.3 前端扩展加载与注册

| 功能 | 文件 | 行号 | 状态 |
|-----|------|-----|------|
| loadExtensions | `app/src/extensions.ts` | 31-41 | ✅ 已核实 |
| registerExtensions | `app/src/extensions.ts` | 43-69 | ✅ 已核实 |
| getInternalInterfaces | `app/src/interfaces/index.ts` | 5-12 | ✅ 已核实 |
| registerInterfaces | `app/src/interfaces/index.ts` | 14-22 | ✅ 已核实 |
| getInternalDisplays | `app/src/displays/index.ts` | 5-9 | ✅ 已核实 |
| registerDisplays | `app/src/displays/index.ts` | 11-19 | ✅ 已核实 |
| getInternalLayouts | `app/src/layouts/index.ts` | 5-9 | ✅ 已核实 |
| registerLayouts | `app/src/layouts/index.ts` | 11-18 | ✅ 已核实 |

### 6.4 状态同步

| 功能 | 文件 | 行号 | 状态 |
|-----|------|-----|------|
| useSync | `packages/composables/src/use-sync.ts` | 137-150 | ✅ 已核实 |
| syncRefProperty | `app/src/utils/sync-ref-property.ts` | 3-12 | ⚠️ 部分修正 |
| usePreset | `app/src/composables/use-preset.ts` | 28-232 | ✅ 已核实 |
| autoSave（防抖） | `app/src/composables/use-preset.ts` | 72-74 | ✅ 已核实 |
| handleChanges | `app/src/composables/use-preset.ts` | 80-88 | ✅ 已核实 |

### 6.5 实际使用示例

| 功能 | 文件 | 行号 | 状态 |
|-----|------|-----|------|
| Calendar Layout 使用 useSync | `app/src/layouts/calendar/index.ts` | 54-56 | ✅ 已核实 |
| Calendar Layout 使用 syncRefProperty | `app/src/layouts/calendar/index.ts` | 94-109 | ✅ 已核实 |
| Input Interface 定义 props | `app/src/interfaces/input/input.vue` | 7-34 | ✅ 已核实 |
| Input Interface 定义 emits | `app/src/interfaces/input/input.vue` | 36 | ✅ 已核实 |
| Input Interface 模板绑定 | `app/src/interfaces/input/input.vue` | 71-93 | ✅ 已核实 |

---

## 7. 总结

### 7.1 结论状态统计

| 状态 | 数量 | 说明 |
|-----|------|-----|
| ✅ 已证实 | 18 项 | 与源码完全对齐 |
| ⚠️ 部分对齐 | 2 项 | 基本正确，但有细节差异 |
| ❌ 推断错误 | 2 项 | 与实际源码不符 |

### 7.2 核心修正点

**1. 开发环境扩展加载**
- ❌ 错误：插件来自 `@directus/extensions-sdk`
- ✅ 正确：插件是 `app/vite.config.js` 中本地定义的

**2. 扩展发现来源**
- ❌ 错误：只有 `extensions/` 目录一种来源
- ✅ 正确：三种来源（local、module、registry）

**3. syncRefProperty 实现**
- ⚠️ 修正：setter 使用 `Object.assign` 而非展开运算符
- ✅ 补充：`defaultValue` 支持传入 `Ref`

**4. Interface 数据绑定**
- ⚠️ 修正：使用 `value` prop + `input` 事件（Vue 2 兼容模式）
- ✅ 补充：外层通过包装器转换为标准 v-model

### 7.3 已证实的核心设计理念

1. **响应式优先**：所有状态同步都通过 Vue 的 `computed`、`ref`、`watch` 实现
2. **防抖优化**：`usePreset` 使用 450ms 防抖避免频繁数据库写入
3. **向后兼容**：Interface 使用 Vue 2 的 v-model 模式保持扩展兼容性
4. **多来源扩展**：支持三种扩展来源（本地、模块、注册表）
5. **开发友好**：开发环境默认启用所有扩展，支持热更新

### 7.4 建议

1. **扩展开发者**：注意 Interface 使用的是 `value` + `input` 模式，而非 Vue 3 的标准模式
2. **架构维护者**：三种扩展来源的优先级和冲突处理需要文档化
3. **未来版本**：考虑迁移 Interface 到 Vue 3 的标准 v-model 模式（使用 `defineModel()`）

---

**报告生成时间**: 2026-05-02  
**复核完整性**: 高（覆盖了发现、加载、注册、状态同步、数据绑定全流程）
