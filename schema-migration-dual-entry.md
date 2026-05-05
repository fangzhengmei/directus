# Directus Schema 迁移双入口机制分析

## 概述

Directus 提供了两种主要方式来管理数据结构（Schema）：
1. **Admin 界面**：通过 Web 界面进行交互式修改
2. **命令行工具**：通过 `directus schema` 命令进行快照管理

这两种方式共享相同的底层服务，但在执行路径和冲突处理上有所不同。

## 核心架构

### 共享的底层服务

两种方式都依赖于以下核心服务：

- **CollectionsService** ([`api/src/services/collections.ts`](api/src/services/collections.ts)): 处理集合的增删改查
- **FieldsService**: 处理字段的增删改查
- **RelationsService**: 处理关系的增删改查

### 核心工具函数

- **getSnapshot** ([`api/src/utils/get-snapshot.ts`](api/src/utils/get-snapshot.ts)): 生成当前数据库结构的快照
- **getSnapshotDiff** ([`api/src/utils/get-snapshot-diff.ts`](api/src/utils/get-snapshot-diff.ts)): 计算两个快照之间的差异
- **applySnapshot** ([`api/src/utils/apply-snapshot.ts`](api/src/utils/apply-snapshot.ts)): 应用快照变更
- **applyDiff** ([`api/src/utils/apply-diff.ts`](api/src/utils/apply-diff.ts)): 应用具体的差异变更

## 两条路径的实现

### 1. Admin 界面路径

Admin 界面通过 REST API 来管理 Schema，主要调用以下端点：

- **GET /schema/snapshot**: 获取当前结构快照
- **POST /schema/apply**: 应用结构变更

**实现细节**：

1. **快照获取**：
   - Admin 界面通过 SDK 的 `schemaSnapshot` 函数调用 API
   - 最终调用 `getSnapshot` 函数生成快照
   - 在 [`app/src/modules/settings/routes/data-model/collections/collections.vue`](app/src/modules/settings/routes/data-model/collections/collections.vue) 中可以看到 `downloadSnapshot` 函数的实现

2. **变更应用**：
   - Admin 界面通过 SDK 的 `schemaApply` 函数调用 API
   - 支持 `force` 参数控制是否强制执行
   - API 端点最终会调用 `applySnapshot` 函数

3. **交互式修改**：
   - Admin 界面的交互式修改（如创建/编辑/删除集合、字段）直接调用对应的 Service 方法
   - 例如：
     - 创建集合：`CollectionsService.createOne()`
     - 更新集合：`CollectionsService.updateOne()`
     - 删除集合：`CollectionsService.deleteOne()`

### 2. 命令行路径

命令行工具通过 `directus schema` 命令管理 Schema，主要子命令：

- **snapshot**: 生成结构快照
- **apply**: 应用结构快照

#### 快照命令 (snapshot)

**实现文件**：[`api/src/cli/commands/schema/snapshot.ts`](api/src/cli/commands/schema/snapshot.ts)

**执行流程**：
1. 获取数据库连接
2. 调用 `getSnapshot` 函数生成当前结构快照
3. 支持 JSON 和 YAML 两种格式
4. 可选保存到文件或输出到标准输出

**核心代码逻辑**：
```typescript
const snapshot = await getSnapshot({ database });
// 转换为指定格式（JSON 或 YAML）
// 保存到文件或输出到 stdout
```

#### 应用命令 (apply)

**实现文件**：[`api/src/cli/commands/schema/apply.ts`](api/src/cli/commands/schema/apply.ts)

**执行流程**：
1. 读取快照文件
2. 获取当前数据库结构快照
3. 计算差异 (`getSnapshotDiff`)
4. 应用变更或预览

## 冲突处理机制

### 冲突检测

Directus 的冲突检测基于**快照比较**机制：

1. **当前快照**：应用前从数据库生成的当前结构状态
2. **目标快照**：从文件读取的期望结构状态
3. **差异计算**：使用 `deep-diff` 库比较两个快照

**差异类型** (DiffKind)：
- `DiffKind.NEW`: 新增的集合/字段/关系
- `DiffKind.DELETE`: 删除的集合/字段/关系
- `DiffKind.EDIT`: 修改的集合/字段/关系
- `DiffKind.ARRAY`: 数组类型的修改

**冲突场景**：
1. **同一对象被两边修改**：Admin 界面修改了某个字段，同时命令行快照也修改了该字段
2. **对象被删除又被修改**：Admin 界面删除了某个集合，命令行快照尝试修改该集合
3. **对象被新增又被修改**：Admin 界面新增了某个集合，命令行快照也新增同名集合

### 冲突解决策略

#### 1. 命令行的默认策略

从代码分析来看，Directus 命令行采用**覆盖式**策略：

**核心逻辑** ([`api/src/utils/apply-diff.ts`](api/src/utils/apply-diff.ts))：
- 对于 `NEW` 类型的差异：直接创建
- 对于 `DELETE` 类型的差异：直接删除
- 对于 `EDIT` 类型的差异：计算新值后更新
- 整个操作在**事务**中执行（MySQL 可能不支持 DDL 事务）

**关键代码片段**：
```typescript
// 对于编辑类型的差异，计算新值后直接更新
const newValues = diff.reduce((acc, currentDiff) => {
    deepDiff.applyChange(acc, undefined, currentDiff);
    return acc;
}, cloneDeep(currentCollection));

await collectionsService.updateOne(collection, newValues, mutationOptions);
```

#### 2. 强制执行 vs 预览模式

**预览模式 (--dry-run)**：
- 仅计算并显示差异，不实际执行
- 输出将要执行的变更列表
- 最后以 `process.exit(0)` 退出

**强制执行 (--yes)**：
- 跳过交互式确认
- 直接应用所有变更
- 不询问用户确认

**正常模式**：
- 显示差异预览
- 询问用户是否继续
- 用户确认后才执行

## 执行路径详细分析

### 命令行预览路径 (--dry-run)

**入口点**：[`api/src/cli/commands/schema/apply.ts:82-199`](api/src/cli/commands/schema/apply.ts#L82-L199)

**执行步骤**：
1. 读取快照文件
2. 获取当前快照：`getSnapshot({ database })`
3. 计算差异：`getSnapshotDiff(currentSnapshot, snapshot)`
4. 格式化差异输出（使用颜色区分不同类型的变更）
5. 输出差异信息到控制台
6. **退出程序，不执行任何变更**：`process.exit(0)`

**关键代码**：
```typescript
const dryRun = options?.dryRun === true;

if (dryRun || promptForChanges) {
    // 格式化并显示差异
    const message = 'The following changes will be applied:\n\n' + sections.join('\n\n');
    
    if (dryRun) {
        console.log(message);
        process.exit(0);  // 预览模式直接退出
    }
    
    // 正常模式会询问用户确认
}
```

**变更类型显示**：
- 绿色：Create（新增）
- 红色：Delete（删除）
- 洋红色：Update（更新）

### 命令行强制执行路径 (--yes)

**入口点**：[`api/src/cli/commands/schema/apply.ts:83-215`](api/src/cli/commands/schema/apply.ts#L83-L215)

**执行步骤**：
1. 读取快照文件
2. 获取当前快照
3. 计算差异
4. 跳过确认（`promptForChanges = false`）
5. **直接执行** `applySnapshot` 函数

**关键代码**：
```typescript
const promptForChanges = !dryRun && options?.yes !== true;

if (dryRun || promptForChanges) {
    // 如果 --yes 为 true，promptForChanges 为 false，跳过此块
}

// 直接执行
await applySnapshot(snapshot, { current: currentSnapshot, diff: snapshotDiff, database });
```

### applySnapshot 执行流程

**入口点**：[`api/src/utils/apply-snapshot.ts`](api/src/utils/apply-snapshot.ts)

**执行步骤**：
1. 获取数据库连接和 Schema 概述
2. 如果未提供当前快照，重新获取
3. 如果未提供差异，重新计算
4. 调用 `applyDiff` 应用具体变更
5. 刷新缓存

### applyDiff 详细流程

**入口点**：[`api/src/utils/apply-diff.ts`](api/src/utils/apply-diff.ts)

**事务包裹**：
```typescript
await transaction(database, async (trx) => {
    // 所有变更在事务中执行
});
```

**执行顺序**：
1. **创建集合**：按层级顺序创建（先创建无父级的集合）
2. **删除集合**：处理关系后删除
3. **更新集合**：应用集合级别的修改
4. **处理字段**：
   - 创建新字段
   - 更新现有字段
   - 删除字段
5. **处理系统字段**：更新系统字段的索引等属性
6. **处理关系**：
   - 创建新关系
   - 更新现有关系
   - 删除关系

**特殊处理**：
- **嵌套集合创建**：自动处理集合之间的层级依赖
- **关系清理**：删除集合时自动清理相关关系
- **字段类型转换**：处理 alias 字段的特殊逻辑
- **UUID 类型适配**：跨数据库类型的兼容性处理

## Admin 界面与命令行的交互

### 共享的底层机制

两种方式都使用相同的核心服务，但触发方式不同：

| 操作 | Admin 界面触发 | 命令行触发 | 底层服务 |
|------|---------------|-----------|---------|
| 获取快照 | 下载快照按钮 | `directus schema snapshot` | getSnapshot |
| 应用快照 | （通过 API） | `directus schema apply` | applySnapshot |
| 创建集合 | 界面操作 | （通过快照） | CollectionsService.createOne |
| 删除集合 | 界面操作 | （通过快照） | CollectionsService.deleteOne |

### 潜在冲突场景

**场景 1：Admin 界面修改 + 命令行应用旧快照**

1. 用户通过 Admin 界面修改了集合 A 的字段
2. 用户尝试应用一个在修改前生成的快照
3. **结果**：命令行会将数据库状态**回滚**到快照中的状态

**场景 2：命令行应用快照 + Admin 界面实时修改**

1. 命令行开始执行 `directus schema apply`
2. 在事务执行期间，Admin 界面尝试修改同一对象
3. **结果**：取决于数据库的隔离级别，可能出现：
   - 事务回滚
   - 后提交的覆盖先提交的
   - 死锁（取决于具体操作）

**场景 3：两边同时创建同名对象**

1. Admin 界面创建集合 `test`
2. 命令行快照也包含集合 `test`
3. 命令行执行 `apply`
4. **结果**：
   - 如果集合已存在且快照中是 `NEW` 类型：可能会尝试创建失败
   - 实际代码中会检查是否已存在，然后执行更新或创建

## 最佳实践建议

### 避免冲突的策略

1. **单一管理入口**：
   - 选择一种方式作为主要管理方式
   - 开发环境用命令行，生产环境用 Admin 界面，或反之

2. **快照同步**：
   - 每次通过 Admin 界面修改后，立即生成新快照
   - 保持代码库中的快照文件与实际数据库同步

3. **先预览后执行**：
   - 总是使用 `--dry-run` 预览变更
   - 确认预期后再执行

4. **事务保护**：
   - 利用数据库事务（如果支持）
   - 失败时自动回滚

### 冲突解决流程

当检测到潜在冲突时：

1. **生成当前快照**：
   ```bash
   directus schema snapshot current-snapshot.json
   ```

2. **比较差异**：
   - 比较当前快照与要应用的快照
   - 识别冲突点

3. **手动合并**：
   - 编辑快照文件，保留两边的有效修改
   - 或者选择以某一边为准

4. **应用合并后的快照**：
   ```bash
   directus schema apply merged-snapshot.json
   ```

## 代码引用

### 核心实现文件

1. **命令行快照**：[`api/src/cli/commands/schema/snapshot.ts`](api/src/cli/commands/schema/snapshot.ts)
2. **命令行应用**：[`api/src/cli/commands/schema/apply.ts`](api/src/cli/commands/schema/apply.ts)
3. **快照生成**：[`api/src/utils/get-snapshot.ts`](api/src/utils/get-snapshot.ts)
4. **差异计算**：[`api/src/utils/get-snapshot-diff.ts`](api/src/utils/get-snapshot-diff.ts)
5. **快照应用**：[`api/src/utils/apply-snapshot.ts`](api/src/utils/apply-snapshot.ts)
6. **差异应用**：[`api/src/utils/apply-diff.ts`](api/src/utils/apply-diff.ts)
7. **集合服务**：[`api/src/services/collections.ts`](api/src/services/collections.ts)
8. **Admin 界面快照**：[`app/src/modules/settings/routes/data-model/collections/collections.vue`](app/src/modules/settings/routes/data-model/collections/collections.vue)
9. **SDK 快照命令**：[`sdk/src/rest/commands/schema/snapshot.ts`](sdk/src/rest/commands/schema/snapshot.ts)
10. **SDK 应用命令**：[`sdk/src/rest/commands/schema/apply.ts`](sdk/src/rest/commands/schema/apply.ts)

## 总结

Directus 的 Schema 迁移双入口机制设计灵活，但也带来了冲突风险：

1. **两条路径共享底层**：Admin 界面和命令行最终都调用相同的 Service 层
2. **覆盖式策略**：默认采用后执行覆盖先执行的策略
3. **预览与强制执行**：
   - 预览模式 (`--dry-run`)：仅显示差异，不执行
   - 强制执行模式 (`--yes`)：跳过确认，直接执行
4. **事务保护**：变更在事务中执行（取决于数据库支持）

**建议**：在团队协作环境中，建立清晰的 Schema 管理流程，优先使用命令行快照进行版本控制，Admin 界面仅用于生产环境的紧急调整或数据查看。
