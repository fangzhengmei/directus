# Directus 三类扩展完整调用链深度分析报告（续篇）

> 本文是 `DIRECTUS_EXTENSIONS_CALLCHAIN_ANALYSIS.md` 的续篇，包含剩余的核心内容。

---

## 9. 权限与上下文传递

### 9.1 权限系统架构

Directus 的权限系统基于 `directus_permissions` 表，通过 Pinia Store 在前端管理。

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                            权限系统架构                                        │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      数据库层                                          │  │
│  │  ┌────────────────────────────────────────────────────────────────┐  │  │
│  │  │ directus_permissions 表                                         │  │  │
│  │  │   - role: 角色 ID                                                │  │  │
│  │  │   - collection: 集合名称                                         │  │  │
│  │  │   - action: read / create / update / delete / share            │  │  │
│  │  │   - access: none / partial / full                                │  │  │
│  │  │   - fields: 有权限的字段列表                                     │  │  │
│  │  │   - presets: 字段预设值                                          │  │  │
│  │  │   - permissions: 字段级权限                                      │  │  │
│  │  │   - validation: 数据验证规则                                     │  │  │
│  │  │   - presets: 创建/更新时的默认值                                 │  │  │
│  │  └────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      API 层                                            │  │
│  │  GET /permissions/me                                                  │  │
│  │    - 返回当前用户的所有权限                                           │  │
│  │    - 解析 $CURRENT_USER, $CURRENT_ROLE 等动态变量                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      Store 层（Pinia）                                │  │
│  │  usePermissionsStore()                                                │  │
│  │    - state.permissions: CollectionAccess                             │  │
│  │    - hydrate(): 从 API 加载权限                                      │  │
│  │    - dehydrate(): 清空权限                                           │  │
│  │    - getPermission(collection, action): 获取权限配置                 │  │
│  │    - hasPermission(collection, action): 检查是否有权限              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                                    ▼                                         │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                      Composable 层                                    │  │
│  │  useCollectionPermissions(collection)                                 │  │
│  │    - readAllowed: ComputedRef<boolean>                               │  │
│  │    - createAllowed: ComputedRef<boolean>                             │  │
│  │    - updateAllowed: ComputedRef<boolean>                             │  │
│  │    - deleteAllowed: ComputedRef<boolean>                             │  │
│  │    - sortAllowed: ComputedRef<boolean>                               │  │
│  │    - archiveAllowed: ComputedRef<boolean>                            │  │
│  │    - revisionsAllowed: ComputedRef<boolean>                          │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 9.2 Permissions Store 实现

```typescript
// app/src/stores/permissions.ts

export const usePermissionsStore = defineStore({
    id: 'permissionsStore',
    
    state: () => ({
        permissions: {} as CollectionAccess,
    }),
    
    actions: {
        /**
         * 从 API 加载当前用户的权限
         */
        async hydrate() {
            const userStore = useUserStore();
            
            // 调用 /permissions/me 获取权限
            const response = await api.get('/permissions/me');
            
            // 解析权限中的动态变量（如 $CURRENT_USER）
            const fields = getNestedDynamicVariableFields(response.data.data);
            
            // 如果权限中引用了用户字段，需要额外加载用户信息
            if (fields.length > 0) {
                await userStore.hydrateAdditionalFields(fields);
            }
            
            // 处理预设值（解析 $CURRENT_USER 等变量）
            this.permissions = mapValues(
                response.data.data as CollectionAccess,
                (collectionPermission: CollectionPermission) => {
                    Object.values(collectionPermission).forEach((actionPermission) => {
                        if (actionPermission.presets) {
                            actionPermission.presets = parsePreset(actionPermission.presets);
                        }
                    });
                    return collectionPermission;
                },
            ) as CollectionAccess;
        },
        
        /**
         * 清空权限
         */
        dehydrate() {
            this.$reset();
        },
        
        /**
         * 获取指定集合和操作的权限配置
         */
        getPermission(collection: string, action: PermissionsAction) {
            return this.permissions[collection]?.[action] ?? null;
        },
        
        /**
         * 检查是否有指定集合和操作的权限
         * 管理员始终返回 true
         */
        hasPermission(collection: string, action: PermissionsAction) {
            const userStore = useUserStore();
            
            // 管理员拥有所有权限
            if (userStore.isAdmin) return true;
            
            // 检查权限配置中的 access 字段
            return (this.getPermission(collection, action)?.access ?? 'none') !== 'none';
        },
    },
});
```

### 9.3 集合权限 Composable

```typescript
// app/src/composables/use-permissions/collection/use-collection-permissions.ts

export type UsableCollectionPermissions = {
    readAllowed: ComputedRef<boolean>;
    createAllowed: ComputedRef<boolean>;
    updateAllowed: ComputedRef<boolean>;
    deleteAllowed: ComputedRef<boolean>;
    sortAllowed: ComputedRef<boolean>;
    archiveAllowed: ComputedRef<boolean>;
    revisionsAllowed: ComputedRef<boolean>;
};

export function useCollectionPermissions(
    collection: Collection  // Ref<string> 或 string
): UsableCollectionPermissions {
    // 使用 isActionAllowed 检查各操作权限
    const readAllowed = isActionAllowed(collection, 'read');
    const createAllowed = isActionAllowed(collection, 'create');
    const updateAllowed = isActionAllowed(collection, 'update');
    const deleteAllowed = isActionAllowed(collection, 'delete');
    
    // 检查排序权限
    const sortAllowed = isSortAllowed(collection);
    
    // 检查归档权限（需要 update 权限 + 归档字段配置）
    const archiveAllowed = isArchiveAllowed(collection, updateAllowed);
    
    // 检查版本控制权限
    const revisionsAllowed = isRevisionsAllowed();
    
    return {
        readAllowed,
        createAllowed,
        updateAllowed,
        deleteAllowed,
        sortAllowed,
        archiveAllowed,
        revisionsAllowed,
    };
}
```

### 9.4 权限检查辅助函数

```typescript
// app/src/composables/use-permissions/collection/lib/is-action-allowed.ts

/**
 * 检查是否有指定集合和操作的权限
 * 返回 ComputedRef 以支持响应式更新
 */
export const isActionAllowed = (collection: Collection, action: PermissionsAction) => {
    const { hasPermission } = usePermissionsStore();
    
    return computed(() => {
        const collectionValue = unref(collection);
        
        if (!collectionValue) return false;
        
        return hasPermission(collectionValue, action);
    });
};
```

### 9.5 权限在 Collection 页面的使用

```typescript
// app/src/modules/content/routes/collection.vue

// 获取集合权限
const {
    readAllowed,           // 读取权限（自动检查）
    createAllowed,         // 创建权限
    updateAllowed: batchEditAllowed,    // 批量编辑权限
    deleteAllowed: batchDeleteAllowed,  // 批量删除权限
} = useCollectionPermissions(collection);

// 使用权限控制 UI
const canInviteUsers = computed(() => {
    if (serverStore.auth.disableDefault === true) return false;
    
    // 检查 directus_users 的 create 权限
    return unref(createAllowed) 
        && userStore.currentUser?.role?.app_access_dashboard === true;
});
```

### 9.6 上下文传递机制

#### 9.6.1 全局上下文注入

```typescript
// app/src/composables/use-system.ts

export function useSystem(app: App): void {
    // 注入所有 Stores（包含权限 Store）
    app.provide(STORES_INJECT, {
        useAppStore,
        useCollectionsStore,
        useFieldsStore,
        usePermissionsStore,    // 权限 Store
        usePresetsStore,
        useUserStore,           // 用户 Store
        // ...
    });

    // 注入 API 客户端
    app.provide(API_INJECT, api);

    // 注入 SDK 客户端
    app.provide(SDK_INJECT, sdk);

    // 注入扩展配置
    app.provide(EXTENSIONS_INJECT, useExtensions());
}
```

#### 9.6.2 访问全局上下文

```typescript
// 在任何组件或 composable 中访问

/**
 * 获取所有 Stores 的访问器
 */
export function useStores(): Record<string, any> {
    const stores = inject<Record<string, any>>(STORES_INJECT);
    
    if (!stores) throw new Error('[useStores]: The stores could not be found.');
    
    return stores;
}

/**
 * 获取 API 客户端
 */
export function useApi(): AxiosInstance {
    const api = inject<AxiosInstance>(API_INJECT);
    
    if (!api) throw new Error('[useApi]: The api could not be found.');
    
    return api;
}

/**
 * 获取 SDK 客户端
 */
export function useSdk<Schema extends object = any>(): DirectusClient<Schema> & RestClient<Schema> {
    const sdk = inject<DirectusClient<Schema> & RestClient<Schema>>(SDK_INJECT);
    
    if (!sdk) throw new Error('[useSdk]: The sdk could not be found.');
    
    return sdk;
}

/**
 * 获取扩展配置
 */
export function useExtensions(): RefRecord<AppExtensionConfigs> {
    const extensions = inject<RefRecord<AppExtensionConfigs>>(EXTENSIONS_INJECT);
    
    if (!extensions) throw new Error('[useExtensions]: The extensions could not be found.');
    
    return extensions;
}
```

#### 9.6.3 上下文在扩展中的传递

**Layout 组件接收的上下文**：

```typescript
// Layout Props 中包含的上下文信息
interface LayoutProps {
    collection: string | null;    // 当前集合名称
    selectMode: boolean;           // 是否为选择模式
    showSelect: ShowSelect;       // 选择显示类型
    readonly: boolean;             // 是否只读
    // ...
}

// Layout 可以通过 composables 获取更多上下文
setup(props, { emit }) {
    // 获取用户信息
    const userStore = useUserStore();
    
    // 获取权限
    const permissionsStore = usePermissionsStore();
    
    // 获取集合信息
    const { info, primaryKeyField } = useCollection(props.collection);
    
    // 获取 API 客户端
    const api = useApi();
    
    // ...
}
```

**Interface 组件接收的上下文**：

```typescript
// Interface 组件接收的 Props
{
    type: string;              // 字段类型
    collection: string;        // 所属集合
    field: string;             // 字段名
    field-data: Field;         // 完整字段配置
    primary-key: string | number | null;  // 当前记录主键
    version: ContentVersion;   // 内容版本
    // ...
}

// Interface 组件可以通过 composables 获取更多上下文
// 但通常不需要，因为所有必要信息都通过 props 传递
```

**Display 组件接收的上下文**：

```typescript
// Display 组件接收的 Props
{
    type: string;              // 字段类型
    collection: string;        // 所属集合
    field: string;             // 字段名
    interface: string;         // 关联的 Interface ID
    interfaceOptions: Record;  // Interface 选项
    // ...
}

// Display handler 函数接收的上下文
handler: (value, options, ctx) => {
    // ctx 包含：
    // - interfaceOptions?: Record
    // - field?: Field
    // - collection?: string
}
```

---

## 10. 完整调用链时序图

### 10.1 应用启动与扩展注册时序

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                        应用启动与扩展注册时序图                                           │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  main.ts                    loadExtensions()          registerExtensions()    useSystem()│
│     │                            │                           │                    │       │
│     │                            │                           │                    │       │
│     │───1. createApp()──────────▶│                           │                    │       │
│     │                            │                           │                    │       │
│     │───2. await loadExtensions()▶                           │                    │       │
│     │                            │                           │                    │       │
│     │                            │───2.1 import('@directus-extensions')───▶       │       │
│     │                            │◀──2.2 返回扩展配置─────────────────────────       │       │
│     │                            │                           │                    │       │
│     │───3. registerExtensions(app)─────────────────────────▶│                    │       │
│     │                            │                           │                    │       │
│     │                            │───3.1 getInternalInterfaces()────────────▶       │       │
│     │                            │◀──返回内置 interfaces────────────────────────       │       │
│     │                            │                           │                    │       │
│     │                            │───3.2 getInternalDisplays()──────────────▶       │       │
│     │                            │◀──返回内置 displays──────────────────────────       │       │
│     │                            │                           │                    │       │
│     │                            │───3.3 getInternalLayouts()───────────────▶       │       │
│     │                            │◀──返回内置 layouts───────────────────────────       │       │
│     │                            │                           │                    │       │
│     │                            │───3.4 合并 customExtensions───────────────▶       │       │
│     │                            │                           │                    │       │
│     │                            │───3.5 registerInterfaces()────────────────▶       │       │
│     │                            │       app.component('interface-*', ...)  │       │
│     │                            │                           │                    │       │
│     │                            │───3.6 registerDisplays()──────────────────▶       │       │
│     │                            │       app.component('display-*', ...)    │       │
│     │                            │                           │                    │       │
│     │                            │───3.7 registerLayouts()───────────────────▶       │       │
│     │                            │       app.component('layout-*', ...)     │       │
│     │                            │                           │                    │       │
│     │───4. useSystem(app)────────────────────────────────────────────────────▶│       │
│     │                            │                           │                    │       │
│     │                            │                           │───4.1 provide(STORES_INJECT, ...)   │
│     │                            │                           │                    │       │
│     │                            │                           │───4.2 provide(API_INJECT, api)    │
│     │                            │                           │                    │       │
│     │                            │                           │───4.3 provide(SDK_INJECT, sdk)    │
│     │                            │                           │                    │       │
│     │                            │                           │───4.4 provide(EXTENSIONS_INJECT,  │
│     │                            │                           │        useExtensions())              │
│     │                            │                           │                    │       │
│     │───5. app.mount('#app')───▶│                           │                    │       │
│     │                            │                           │                    │       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 10.2 Layout 完整调用时序

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                           Layout 完整调用时序图                                           │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  User                        Router          collection.vue     usePreset     useLayout  │
│   │                            │                  │                │              │       │
│   │───1. 点击集合导航──────────▶│                  │                │              │       │
│   │                            │                  │                │              │       │
│   │                            │───2. 导航到 /content/:collection▶             │       │
│   │                            │                  │                │              │       │
│   │                            │                  │───3. setup()──▶│              │       │
│   │                            │                  │                │              │       │
│   │                            │                  │                │───3.1 usePreset(collection, bookmarkID)│
│   │                            │                  │                │              │       │
│   │                            │                  │                │    3.1.1 presetsStore.getPresetForCollection()│
│   │                            │                  │                │              │       │
│   │                            │                  │                │    3.1.2 返回 layout, layoutOptions, layoutQuery│
│   │                            │                  │                │              │       │
│   │                            │                  │                │              │       │
│   │                            │                  │───3.2 useLayout(layout)──────▶│       │
│   │                            │                  │                │              │       │
│   │                            │                  │                │              │───3.2.1 useExtensions()│
│   │                            │                  │                │              │       │
│   │                            │                  │                │              │───3.2.2 createLayoutWrapper()│
│   │                            │                  │                │              │       │
│   │                            │                  │                │              │    3.2.3 返回 layoutWrapper│
│   │                            │                  │                │              │       │
│   │                            │                  │───4. render()──▶│              │       │
│   │                            │                  │                │              │       │
│   │                            │                  │    <component :is="layoutWrapper">│
│   │                            │                  │                │              │       │
│   │                            │                  │    layoutWrapper.setup() 执行:   │
│   │                            │                  │                │              │       │
│   │                            │                  │    ├───4.1 调用 layout.setup(props, { emit })│
│   │                            │                  │                │              │       │
│   │                            │                  │    │       ├───useSync 同步 selection│
│   │                            │                  │    │       ├───useCollection 获取集合信息│
│   │                            │                  │    │       └───useItems 获取数据列表│
│   │                            │                  │    │              │              │       │
│   │                            │                  │    │              │───4.1.1 useApi() 获取 API│
│   │                            │                  │    │              │              │       │
│   │                            │                  │    │              │───4.1.2 api.get('/items/:collection')│
│   │                            │                  │    │              │              │       │
│   │                            │                  │    │              │◀──4.1.3 返回数据列表      │
│   │                            │                  │    │              │              │       │
│   │                            │                  │    └───4.2 合并到响应式 state    │
│   │                            │                  │                │              │       │
│   │                            │                  │───5. 插槽传递 layoutState        │
│   │                            │                  │                │              │       │
│   │                            │                  │    <component :is="`layout-${layoutId}`"│
│   │                            │                  │                │              │       │
│   │◀──6. 页面渲染完成───────────│◀─────────────────│                │              │       │
│   │                            │                  │                │              │       │
│   │                            │                  │                │              │       │
│   │───7. 用户编辑布局选项──────▶│                  │                │              │       │
│   │                            │                  │                │              │       │
│   │                            │                  │    layoutOptions.value = { ... } │
│   │                            │                  │                │              │       │
│   │                            │                  │    ├───v-model:layout-options 触发更新│
│   │                            │                  │    ├───useSync.set() 触发 emit  │
│   │                            │                  │    ├───collection.vue 接收更新   │
│   │                            │                  │    ├───usePreset.updatePreset() │
│   │                            │                  │    └───autoSave() 保存到数据库   │
│   │                            │                  │                │              │       │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

### 10.3 Interface 完整调用时序

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                          Interface 完整调用时序图                                          │
├──────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  User                      v-form         form-field-interface   Interface 组件   API  │
│   │                         │                    │                   │              │    │
│   │───1. 打开编辑表单───────▶│                    │                   │              │    │
│   │                         │                    │                   │              │    │
│   │                         │───2. render()─────▶│                   │              │    │
│   │                         │                    │                   │              │    │
│   │                         │                    │───2.1 useExtension('interface', id)│
│   │                         │                    │                   │              │    │
│   │                         │                    │    2.1.1 useExtensions() 获取扩展列表│
│   │                         │                    │                   │              │    │
│   │                         │                    │    2.1.2 find({ id }) 查找配置    │
│   │                         │                    │                   │              │    │
│   │                         │                    │                   │              │    │
│   │                         │                    │───2.2 构建组件名: `interface-${id}`│
│   │                         │                    │                   │              │    │
│   │                         │                    │───2.3 动态渲染组件──────────────▶│    │
│   │                         │                    │                   │              │    │
│   │                         │                    │    传递 props:                  │    │
│   │                         │                    │    - value: 当前字段值            │    │
│   │                         │                    │    - field-data: 完整字段配置     │    │
│   │                         │                    │    - collection: 集合名称         │    │
│   │                         │                    │    - ... 其他上下文              │    │
│   │                         │                    │                   │              │    │
│   │◀──3. 表单渲染完成───────│◀───────────────────│◀──────────────────│              │    │
│   │                         │                    │                   │              │    │
│   │                         │                    │                   │              │    │
│   │───4. 用户编辑字段值─────▶│                    │                   │              │    │
│   │                         │                    │                   │              │    │
│   │                         │                    │                   │───4.1 用户交互更新本地值│
│   │                         │                    │                   │              │    │
│   │                         │                    │                   │───4.2 emit('input', newValue)│
│   │                         │                    │                   │              │    │
│   │                         │                    │───4.3 接收 @input 事件───────────│    │
│   │                         │                    │                   │              │    │
│   │                         │                    │    $emit('update:modelValue', $event)│
│   │                         │                    │                   │              │    │
│   │                         │───4.4 更新 modelValues[field.field]───│              │    │
│   │                         │                    │                   │              │    │
│   │                         │                    │                   │              │    │
│   │───5. 点击保存───────────▶│                    │                   │              │    │
│   │                         │                    │                   │              │    │
│   │                         │───5.1 收集所有字段值───────────────────│              │    │
│   │                         │                    │                   │              │    │
│   │                         │───5.2 api.patch('/items/:collection/:id')──────────────▶│
│   │                         │                    │                   │              │    │
│   │                         │◀──5.3 返回更新结果─────────────────────────────────────────│
│   │                         │                    │                   │              │    │
│   │◀──6. 保存完成提示───────▶│                    │                   │              │    │
│   │                         │                    │                   │              │    │
└──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. 关键代码索引

### 11.1 应用启动与扩展注册

| 功能 | 文件路径 | 关键函数/代码 |
|-----|---------|--------------|
| 应用入口 | `app/src/main.ts` | `init()`, `loadExtensions()`, `registerExtensions()` |
| 扩展加载 | `app/src/extensions.ts` | `loadExtensions()`, `registerExtensions()` |
| 全局注入 | `app/src/composables/use-system.ts` | `useSystem()`, `provide()` |
| Layout 注册 | `app/src/layouts/index.ts` | `getInternalLayouts()`, `registerLayouts()` |
| Interface 注册 | `app/src/interfaces/index.ts` | `getInternalInterfaces()`, `registerInterfaces()` |
| Display 注册 | `app/src/displays/index.ts` | `getInternalDisplays()`, `registerDisplays()` |
| 扩展定义 | `packages/extensions/src/shared/utils/define-extension.ts` | `defineLayout()`, `defineInterface()`, `defineDisplay()` |
| 扩展类型 | `packages/types/src/extensions/*.ts` | `LayoutConfig`, `InterfaceConfig`, `DisplayConfig` |

### 11.2 API 端扩展发现

| 功能 | 文件路径 | 关键函数/代码 |
|-----|---------|--------------|
| 扩展发现 | `packages/extensions/src/node/utils/get-extensions.ts` | `resolveFsExtensions()`, `resolveModuleExtensions()` |
| 入口点生成 | `packages/extensions/src/node/utils/generate-extensions-entrypoint.ts` | `generateExtensionsEntrypoint()` |
| 扩展常量 | `packages/constants/src/extensions.ts` | `APP_EXTENSION_TYPES`, `HYBRID_EXTENSION_TYPES` |

### 11.3 运行时扩展使用

| 功能 | 文件路径 | 关键函数/代码 |
|-----|---------|--------------|
| 扩展查找 | `app/src/composables/use-extension.ts` | `useExtension()` |
| Layout 包装器 | `packages/composables/src/use-layout.ts` | `useLayout()`, `createLayoutWrapper()` |
| 状态同步 | `packages/composables/src/use-sync.ts` | `useSync()` |
| 预设管理 | `app/src/composables/use-preset.ts` | `usePreset()` |
| 数据获取 | `packages/composables/src/use-items.ts` | `useItems()` |
| 集合信息 | `packages/composables/src/use-collection.ts` | `useCollection()` |
| Layout 使用 | `app/src/modules/content/routes/collection.vue` | `usePreset()`, `useLayout()` |
| Interface 渲染 | `app/src/components/v-form/components/form-field-interface.vue` | `useExtension()`, 动态组件渲染 |
| Display 渲染 | `app/src/views/private/components/render-display.vue` | `useExtension()`, 动态组件渲染 |

### 11.4 权限系统

| 功能 | 文件路径 | 关键函数/代码 |
|-----|---------|--------------|
| 权限 Store | `app/src/stores/permissions.ts` | `usePermissionsStore()`, `hasPermission()` |
| 集合权限 | `app/src/composables/use-permissions/collection/use-collection-permissions.ts` | `useCollectionPermissions()` |
| 权限检查 | `app/src/composables/use-permissions/collection/lib/is-action-allowed.ts` | `isActionAllowed()` |

### 11.5 注入键定义

| 注入键 | 用途 | 提供位置 |
|-------|------|---------|
| `STORES_INJECT` | 所有 Pinia Stores | `app/src/composables/use-system.ts` |
| `API_INJECT` | Axios API 客户端 | `app/src/composables/use-system.ts` |
| `SDK_INJECT` | Directus SDK 客户端 | `app/src/composables/use-system.ts` |
| `EXTENSIONS_INJECT` | 扩展配置 | `app/src/composables/use-system.ts` |

---

## 12. 总结

### 12.1 核心设计理念

1. **配置驱动**：所有扩展通过配置对象定义，组件注册全自动化
2. **约定优于配置**：组件命名遵循 `${type}-${id}` 规范
3. **依赖注入**：通过 Vue 的 `provide/inject` 实现全局上下文访问
4. **响应式设计**：使用 `computed`, `ref`, `watch` 实现状态自动同步
5. **分层架构**：API 端发现 → App 端注册 → 运行时使用，各层职责清晰
6. **权限集成**：权限检查与 UI 控制紧密集成，支持字段级权限

### 12.2 三类扩展对比

| 维度 | Layout | Interface | Display |
|-----|--------|-----------|---------|
| **职责粒度** | 集合级 | 字段级 | 字段级 |
| **数据流向** | 主动获取（useItems） | 双向绑定（v-model） | 单向接收（props） |
| **状态管理** | 复杂（selection, pagination） | 简单（value） | 无 |
| **权限依赖** | 集合级 CRUD 权限 | 字段级权限 | 字段级权限 |
| **上下文接收** | Props + Composables | Props（完整） | Props（部分） |
| **注册组件数** | 4 个（主组件 + 3 插槽） | 1-2 个 | 1-2 个 |

### 12.3 完整调用链要点

```
扩展完整生命周期：
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 发现阶段（API 端）                                                       │
│     ├── 扫描文件系统：extensions/ 目录                                      │
│     ├── 解析 package.json：检查 "directus" 字段                            │
│     ├── 验证 manifest：ExtensionManifest.parse()                            │
│     ├── 检查 enabled 状态：从数据库 settings 读取                           │
│     └── 生成入口点：generateExtensionsEntrypoint()                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. 加载阶段（App 端启动时）                                                 │
│     ├── 开发环境：import('@directus-extensions')                          │
│     ├── 生产环境：import('${rootPath}extensions/sources/index.js')        │
│     └── 容错处理：加载失败不中断应用，仅输出警告                           │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. 注册阶段（App 端启动时）                                                 │
│     ├── 获取内置扩展：import.meta.glob() 扫描目录                         │
│     ├── 合并自定义扩展：customExtensions.interfaces/displays/layouts       │
│     ├── 注册 Vue 组件：app.component(`${type}-${id}`, component)          │
│     ├── 设置响应式引用：extensions.* = shallowRef(...)                     │
│     └── 语言切换支持：watch(i18n.locale, () => translate(extensions))     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 注入阶段（App 端启动时）                                                 │
│     ├── provide(STORES_INJECT, { usePermissionsStore, useUserStore, ... })│
│     ├── provide(API_INJECT, api)                                           │
│     ├── provide(SDK_INJECT, sdk)                                           │
│     └── provide(EXTENSIONS_INJECT, useExtensions())                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 使用阶段（运行时）                                                       │
│     ┌─────────────────────────────────────────────────────────────────────┐│
│     │ Layout 使用：                                                        ││
│     │   ├── usePreset() 加载持久化状态（layout, layoutOptions, ...）     ││
│     │   ├── useLayout() 创建包装器                                        ││
│     │   ├── v-model:selection/layoutOptions/layoutQuery 双向绑定         ││
│     │   ├── layout.setup() 执行：useCollection, useItems, useSync        ││
│     │   └── 动态组件渲染：<component :is="`layout-${id}`" v-bind="state">││
│     └─────────────────────────────────────────────────────────────────────┘│
│     ┌─────────────────────────────────────────────────────────────────────┐│
│     │ Interface 使用：                                                     ││
│     │   ├── useExtension('interface', field.meta.interface)              ││
│     │   ├── 构建组件名：`interface-${id}`                                 ││
│     │   ├── v-model 或 @input 事件处理                                    ││
│     │   └── 传递上下文 props：value, field-data, collection, ...         ││
│     └─────────────────────────────────────────────────────────────────────┘│
│     ┌─────────────────────────────────────────────────────────────────────┐│
│     │ Display 使用：                                                       ││
│     │   ├── useExtension('display', field.meta.display)                  ││
│     │   ├── 构建组件名：`display-${id}`                                   ││
│     │   └── 传递只读 props：value, options, type, collection, field      ││
│     └─────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  6. 状态同步阶段（运行时）                                                   │
│     ├── useSync()：prop ↔ emit 双向绑定                                    │
│     ├── syncRefProperty()：对象属性同步                                    │
│     ├── usePreset()：localPreset ↔ 数据库自动保存（防抖）                 │
│     └── v-model: 语法糖简化双向绑定                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  7. 权限检查阶段（运行时）                                                   │
│     ├── useCollectionPermissions()：集合级权限检查                         │
│     ├── usePermissionsStore().hasPermission()：细粒度权限检查              │
│     ├── 管理员豁免：userStore.isAdmin → 始终返回 true                      │
│     └── UI 控制：权限 → disabled 状态 / 按钮隐藏                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

**报告完成时间**：2026-05-02

**相关文件**：
- `DIRECTUS_EXTENSIONS_CALLCHAIN_ANALYSIS.md` - 报告前篇
- `DIRECTUS_EXTENSIONS_ANALYSIS.md` - 基础分析报告