# Directus Schema 迁移双入口机制分析

## 概述

Directus 提供了两种主要方式来管理数据结构（Schema）：
1. **Admin 界面（REST API 路径）**：通过 Web 界面或 REST API 进行两阶段提交（diff → apply），带哈希验证
2. **命令行工具**：通过 `directus schema` 命令进行单阶段提交，每次基于当前结构重算差异

这两种方式共享相同的底层服务，但在执行路径和冲突处理上有本质区别。

---

## 核心架构

### 共享的底层服务

两种方式都依赖于以下核心服务：

| 服务 | 路径 | 职责 |
|------|------|------|
| CollectionsService | [`api/src/services/collections.ts`](api/src/services/collections.ts) | 集合的增删改查 |
| FieldsService | [`api/src/services/fields.ts`](api/src/services/fields.ts) | 字段的增删改查 |
| RelationsService | [`api/src/services/relations.ts`](api/src/services/relations.ts) | 关系的增删改查 |
| SchemaService | [`api/src/services/schema.ts`](api/src/services/schema.ts) | 快照、差异计算、应用（REST API 专用） |

### 核心工具函数

| 函数 | 路径 | 职责 |
|------|------|------|
| getSnapshot | [`api/src/utils/get-snapshot.ts`](api/src/utils/get-snapshot.ts) | 生成当前数据库结构的快照 |
| getSnapshotDiff | [`api/src/utils/get-snapshot-diff.ts`](api/src/utils/get-snapshot-diff.ts) | 计算两个快照之间的差异 |
| applyDiff | [`api/src/utils/apply-diff.ts`](api/src/utils/apply-diff.ts) | 应用具体的差异变更 |
| getVersionedHash | [`api/src/utils/get-versioned-hash.ts`](api/src/utils/get-versioned-hash.ts) | 生成带版本信息的快照哈希 |
| validateApplyDiff | [`api/src/utils/validate-diff.ts`](api/src/utils/validate-diff.ts) | 验证差异的合法性（REST API 专用） |

---

## 两条路径的完整实现

### 1. Admin 界面 / REST API 路径（两阶段提交）

这是 Admin 界面和程序化 API 调用使用的路径，采用 **diff → apply** 两阶段提交模式，带哈希验证机制。

#### 端点概览

| 端点 | 方法 | 功能 | 支持 force 参数 |
|------|------|------|----------------|
| `/schema/snapshot` | GET | 获取当前结构快照 | 否 |
| `/schema/diff` | POST | 计算与目标快照的差异，返回带哈希的 diff | 是 |
| `/schema/apply` | POST | 应用带哈希的 diff | 是 |

#### SDK 对应的命令

```typescript
// 获取快照
schemaSnapshot()

// 计算差异（可选 force）
schemaDiff(snapshot, force = false)

// 应用差异（可选 force）
schemaApply(diff, force = false)
```

#### 完整执行链路

**第一阶段：生成差异（POST /schema/diff）**

**入口点**：[`api/src/controllers/schema.ts:102-117`](api/src/controllers/schema.ts#L102-L117)

**执行步骤**：

1. **接收快照**：从请求体解析目标快照（支持 JSON/YAML，支持 multipart/form-data）
2. **获取当前快照**：`service.snapshot()` → `getSnapshot({ database })`
3. **验证快照**：`validateSnapshot(snapshot, force)`
   - `force=true` 时绕过版本和数据库厂商限制
4. **计算差异**：`getSnapshotDiff(currentSnapshot, targetSnapshot)`
5. **生成哈希**：`getVersionedHash(currentSnapshot)`
   - 基于当前快照内容 + Directus 版本生成唯一哈希
   - 实现：`hash({ item, version })`
6. **返回结果**：`{ hash: currentSnapshotHash, diff: snapshotDiff }`

**关键代码**：
```typescript
// controllers/schema.ts
const currentSnapshot = await service.snapshot();
const snapshotDiff = await service.diff(snapshot, { currentSnapshot, force: 'force' in req.query });

const currentSnapshotHash = getVersionedHash(currentSnapshot);
res.locals['payload'] = { data: { hash: currentSnapshotHash, diff: snapshotDiff } };
```

**哈希生成逻辑** ([`api/src/utils/get-versioned-hash.ts`](api/src/utils/get-versioned-hash.ts))：
```typescript
export function getVersionedHash(item: Record<string, any>): string {
    return hash({ item, version });  // 结合快照内容和 Directus 版本
}
```

---

**第二阶段：应用差异（POST /schema/apply）**

**入口点**：[`api/src/controllers/schema.ts:119-129`](api/src/controllers/schema.ts#L119-L129)

**执行步骤**：

1. **接收带哈希的 diff**：`{ hash: string, diff: SnapshotDiff }`
2. **重新获取当前快照**：`service.snapshot()` → `getSnapshot({ database })`
3. **重新计算当前快照的哈希**：`getVersionedHash(currentSnapshot)`
4. **验证 diff**：`validateApplyDiff(payload, snapshotWithHash, force)`
   - **核心验证**：检查携带的哈希与当前哈希是否匹配
   - `force=true` 时绕过哈希验证
5. **应用变更**：`applyDiff(currentSnapshot, payload.diff, { database })`

**关键代码**（SchemaService.apply）：
```typescript
// services/schema.ts
async apply(payload: SnapshotDiffWithHash, options?: { force?: boolean }): Promise<void> {
    const currentSnapshot = await this.snapshot();
    const snapshotWithHash = this.getHashedSnapshot(currentSnapshot);  // 重新计算哈希

    if (!validateApplyDiff(payload, snapshotWithHash, options?.force)) return;

    await applyDiff(currentSnapshot, payload.diff, { database: this.knex });
}
```

#### 哈希不匹配时的失败分支

**核心验证逻辑** ([`api/src/utils/validate-diff.ts:72-202`](api/src/utils/validate-diff.ts#L72-L202))

**快速路径检查**（第 91 行）：
```typescript
// Diff can be applied due to matching hash
if (applyDiff.hash === currentSnapshotWithHash.hash || force) return true;
```

**失败分支 1：哈希不匹配 + force=false → 进入详细检查**

当哈希不匹配时，代码会逐一检查每个差异是否与当前状态冲突：

| 差异类型 | 检查逻辑 | 错误消息 |
|---------|---------|---------|
| 创建集合 (NEW) | 检查集合是否已存在 | `"Provided diff is trying to create collection \"${collection}\" but it already exists..."` |
| 删除集合 (DELETE) | 检查集合是否不存在 | `"Provided diff is trying to delete collection \"${collection}\" but it does not exist..."` |
| 创建字段 (NEW) | 检查字段是否已存在 | `"Provided diff is trying to create field \"${field}\" but it already exists..."` |
| 删除字段 (DELETE) | 检查字段是否不存在 | `"Provided diff is trying to delete field \"${field}\" but it does not exist..."` |
| 创建关系 (NEW) | 检查关系是否已存在 | `"Provided diff is trying to create relation \"${relation}\" but it already exists..."` |
| 删除关系 (DELETE) | 检查关系是否不存在 | `"Provided diff is trying to delete relation \"${relation}\" but it does not exist..."` |
| 系统字段操作 | 只允许 EDIT schema.is_indexed | `"Provided diff is trying to ${action} field... but this action is not supported"` |

**失败分支 2：详细检查通过 → 抛出通用哈希不匹配错误**

如果上述详细检查都没有发现明显冲突（例如只是修改了某些属性），最后会抛出：

```typescript
throw new InvalidPayloadError({
    reason: `Provided hash does not match the current instance's schema hash, indicating the schema has changed after this diff was generated. Please generate a new diff and try again or use the "force" query parameter to bypass this check`,
});
```

**验证流程图**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    validateApplyDiff 执行流程                     │
├─────────────────────────────────────────────────────────────────┤
│  1. Joi Schema 验证（必填字段、格式）                              │
│      ↓ 失败 → throw "hash is required" 等                        │
│  2. 检查是否为空 diff                                              │
│      ↓ 为空 → return false（无变更）                               │
│  3. 快速路径检查 ⭐ 核心                                          │
│      if (hash 匹配 || force=true) → return true ✅                │
│      ↓ 不匹配且 force=false                                        │
│  4. 详细冲突检查（逐个检查每个差异）                                 │
│      ├─ 集合：NEW 检查是否已存在，DELETE 检查是否不存在            │
│      ├─ 字段：NEW 检查是否已存在，DELETE 检查是否不存在            │
│      ├─ 系统字段：只允许 EDIT schema.is_indexed                   │
│      └─ 关系：NEW 检查是否已存在，DELETE 检查是否不存在            │
│      ↓ 任一检查失败 → throw 具体错误消息                          │
│  5. 详细检查通过但哈希仍不匹配                                      │
│      → throw "Provided hash does not match..." 通用错误           │
└─────────────────────────────────────────────────────────────────┘
```

#### force 参数的放行条件

**force 参数的作用**：

| 阶段 | force=true 的影响 | 代码位置 |
|------|------------------|---------|
| `/schema/diff` | 绕过版本和数据库厂商限制 | `validateSnapshot(snapshot, force)` |
| `/schema/apply` | **绕过哈希验证** | `validateApplyDiff(..., force)` |

**force 放行的关键代码** ([`api/src/utils/validate-diff.ts:91`](api/src/utils/validate-diff.ts#L91))：
```typescript
// Diff can be applied due to matching hash
if (applyDiff.hash === currentSnapshotWithHash.hash || force) return true;
```

**force=true 时仍会执行的验证**：
1. **Joi Schema 验证**：确保 diff 格式正确（必须有 hash、diff 等字段）
2. **空 diff 检查**：无变更时不执行
3. **系统字段操作限制**：只允许修改 `schema.is_indexed`

**force 参数的使用场景**：

| 场景 | 建议 | 风险 |
|------|------|------|
| 确定 diff 生成后 schema 未变化 | 不需要 force | 无 |
| 明确知道有并发修改但仍要应用 | 使用 force ⚠️ | 可能覆盖他人修改 |
| 自动化脚本/CI/CD 环境 | 谨慎使用 | 需要确保独占访问 |
| 紧急修复 | 可考虑使用 | 需事后同步快照 |

---

### 2. 命令行路径（单阶段提交）

命令行工具通过 `directus schema` 命令管理 Schema，采用**单阶段提交**模式，**每次都基于当前数据库结构重算差异**，**无哈希验证机制**。

#### 命令概览

| 命令 | 功能 | 关键参数 |
|------|------|---------|
| `directus schema snapshot [file]` | 生成结构快照 | `--format json/yaml`, `--yes` |
| `directus schema apply <file>` | 应用结构快照 | `--dry-run`, `--yes`, `--ignore-rules` |

#### 快照命令 (snapshot)

**实现文件**：[`api/src/cli/commands/schema/snapshot.ts`](api/src/cli/commands/schema/snapshot.ts)

**执行流程**：
1. 获取数据库连接
2. 调用 `getSnapshot({ database })` 生成当前结构快照
3. 支持 JSON 和 YAML 两种格式
4. 可选保存到文件或输出到标准输出

**与 REST API 路径的差异**：
- 命令行的 `snapshot` 命令与 REST API 的 `GET /schema/snapshot` 底层相同
- 都调用 `getSnapshot` 函数
- 但命令行不生成用于验证的哈希（哈希只在 `/schema/diff` 中生成）

#### 应用命令 (apply)

**实现文件**：[`api/src/cli/commands/schema/apply.ts`](api/src/cli/commands/schema/apply.ts)

**这是与 REST API 路径最关键的区别所在**。

**执行流程**：

1. **读取快照文件**：从磁盘读取目标快照
2. **获取当前快照**：`getSnapshot({ database })`
3. **计算差异**：`getSnapshotDiff(currentSnapshot, targetSnapshot)`
4. **预览或执行**：
   - `--dry-run`：仅显示差异，不执行
   - 正常模式：显示差异，询问确认后执行
   - `--yes`：跳过确认，直接执行
5. **应用变更**：`applySnapshot(snapshot, { current, diff, database })`

**关键代码**：
```typescript
// cli/commands/schema/apply.ts
let snapshot: Snapshot;
// ... 读取文件 ...

const currentSnapshot = await getSnapshot({ database });
let snapshotDiff = getSnapshotDiff(currentSnapshot, snapshot);  // ⭐ 每次都重算！

// ... 预览或确认 ...

await applySnapshot(snapshot, { current: currentSnapshot, diff: snapshotDiff, database });
```

**与 REST API 路径的核心区别**：

| 特性 | REST API 路径 (diff → apply) | 命令行路径 (apply) |
|------|------------------------------|-------------------|
| **提交模式** | 两阶段 | 单阶段 |
| **差异计算时机** | 第一阶段（diff） | 执行前立即计算 |
| **哈希验证** | ✅ 有（验证 diff 时状态与 apply 时一致） | ❌ 无 |
| **并发保护** | ✅ 有（哈希不匹配失败） | ❌ 无（直接基于当前状态） |
| **force 参数** | 绕过哈希验证 | 无对应概念（`--yes` 是跳过确认） |

---

## 两条路径的详细对比

### 执行路径对比图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        REST API 路径（两阶段，带哈希保护）                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  时间点 T1                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  POST /schema/diff                                                   │   │
│  │  ├─ 读取目标快照                                                      │   │
│  │  ├─ getSnapshot() → current_SNAPSHOT_T1                             │   │
│  │  ├─ getSnapshotDiff(current_T1, target) → DIFF                      │   │
│  │  └─ getVersionedHash(current_T1) → HASH_T1                          │   │
│  │  返回: { hash: HASH_T1, diff: DIFF }                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                              ↓ （用户持有 hash+diff）                          │
│  ⏰ 时间流逝... 可能有并发修改！                                               │
│                              ↓                                                │
│  时间点 T2                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  POST /schema/apply  (携带 HASH_T1 + DIFF)                           │   │
│  │  ├─ getSnapshot() → current_SNAPSHOT_T2  ⭐ 重新获取！              │   │
│  │  ├─ getVersionedHash(current_T2) → HASH_T2                           │   │
│  │  ├─ 验证: HASH_T1 === HASH_T2 ?                                       │   │
│  │  │   ├─ ✅ 匹配 → 执行 applyDiff                                      │   │
│  │  │   └─ ❌ 不匹配 → 检查 force 参数                                   │   │
│  │  │       ├─ force=true → 放行 ⚠️                                     │   │
│  │  │       └─ force=false → 详细检查 → throw Error ❌                  │   │
│  │  └─ 执行 applyDiff (仅验证通过后)                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        命令行路径（单阶段，无哈希保护）                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  时间点 T1 (执行 directus schema apply 时)                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  执行 apply 命令                                                       │   │
│  │  ├─ 读取目标快照文件                                                    │   │
│  │  ├─ getSnapshot() → current_SNAPSHOT_T1  ⭐ 立即获取当前状态         │   │
│  │  ├─ getSnapshotDiff(current_T1, target) → DIFF_T1  ⭐ 立即计算      │   │
│  │  ├─ (可选) 预览差异                                                    │   │
│  │  └─ 直接执行 applyDiff(current_T1, DIFF_T1)                          │   │
│  │                                                                       │   │
│  │  ⚠️ 关键：没有任何机制保护"用户意图"与"实际执行"之间的一致性          │   │
│  │     用户以为在应用"快照与某个历史状态的差异"                          │   │
│  │     实际上在应用"快照与执行时刻当前状态的差异"                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 核心差异总结

| 维度 | REST API 路径 | 命令行路径 |
|------|--------------|-----------|
| **设计目标** | 程序化调用、前端界面、并发安全 | 手动操作、版本控制同步 |
| **一致性保证** | 哈希验证保证 diff 时状态 = apply 时状态 | 无，每次基于执行时状态 |
| **并发修改感知** | 能检测到（哈希变化） | 不能（会基于新状态重算） |
| **用户意图保留** | 保留（应用的是 diff 时计算的差异） | 不保留（应用的是执行时计算的差异） |
| **失败模式** | 哈希不匹配时失败（保护数据） | 不会因并发失败（但可能覆盖） |

---

## 并发改动下两条路径的实际后果

### 场景说明

假设以下时间线：

```
时间轴 ──────────────────────────────────────────────────────────────►

T0:  数据库初始状态
     - 集合: articles, users
     - 字段: articles.title, articles.content, users.name

T1:  操作 A 开始
     - 路径 X: 调用 POST /schema/diff (目标: 删除 articles.content)
     - 或
     - 路径 Y: 准备执行 directus schema apply (目标快照: 无 articles.content)

T2:  操作 B 并发执行（Admin 界面或其他客户端）
     - 创建新字段: articles.status (string, default: 'draft')

T3:  操作 A 继续执行
     - 路径 X: 调用 POST /schema/apply (携带 T1 时的 hash + diff)
     - 或
     - 路径 Y: directus schema apply 开始执行

T4:  最终结果？
```

### 场景 1：REST API 路径（diff → apply）

**操作 A 在 T1 执行 diff**：
- 当前快照：有 `articles.content`，无 `articles.status`
- 目标快照：无 `articles.content`
- 计算的 diff：`DELETE articles.content`
- 生成的 hash：HASH_T1（基于 T1 状态）

**操作 B 在 T2 并发修改**：
- 新增 `articles.status` 字段
- 数据库状态改变

**操作 A 在 T3 执行 apply（携带 HASH_T1 + DELETE diff）**：

**子场景 1a：不带 force 参数**

```
执行流程：
1. 重新获取当前快照 (T3 状态)
   - 有 articles.content (未被修改)
   - 有 articles.status (新增的)
2. 重新计算 hash: HASH_T3
3. 验证: HASH_T1 === HASH_T3 ?
   - ❌ 不匹配！因为多了 articles.status
4. 详细检查:
   - diff 是 DELETE articles.content
   - 检查 articles.content 是否存在？→ 存在 ✓
   - 无明显冲突...
5. 最终抛出:
   "Provided hash does not match the current instance's schema hash,
    indicating the schema has changed after this diff was generated..."
```

**结果**：❌ 操作失败，**不会删除 articles.content**

**子场景 1b：带 force=true 参数**

```
执行流程：
1. 重新获取当前快照 (T3 状态)
2. 重新计算 hash: HASH_T3
3. 验证: HASH_T1 === HASH_T3 || force ?
   - ✅ force=true → 直接放行！
4. 执行 applyDiff:
   - 应用 DELETE articles.content
   - 完全忽略 articles.status 的存在
```

**结果**：
- ✅ `articles.content` 被删除
- ⚠️ `articles.status` 保留（不受影响）
- ⚠️ **但这不是原子操作**：操作 A 意图删除 content，但操作 B 新增了 status，最终两者都生效了

### 场景 2：命令行路径（directus schema apply）

**操作 A 在 T3 才开始执行 apply**：

```
执行流程：
1. 读取目标快照文件（无 articles.content）
2. 立即获取当前快照 (T3 状态)
   - 有 articles.content
   - 有 articles.status ⭐ 能看到并发新增的字段！
3. 计算差异 getSnapshotDiff(current_T3, target)
   - 比较：
     current_T3: 有 content, 有 status
     target:     无 content, 无 status ⭐ 目标快照也没有 status！
   - 计算出的 diff:
     DELETE articles.content （原本意图）
     DELETE articles.status  ⭐ 额外删除！因为目标快照没有这个字段
4. 预览或直接执行
5. 应用这两个 DELETE 操作
```

**结果**：
- ✅ `articles.content` 被删除（符合预期）
- ❌ `articles.status` **也被删除**（非预期！）
- ⚠️ 命令行"看不到"并发操作，它只是简单地将数据库**同步到目标快照的状态**

### 更多并发场景分析

#### 场景 3：两边修改同一字段的不同属性

```
T1: 操作 A diff → 目标: 修改 articles.title 长度 255 → 500
T2: 操作 B → 修改 articles.title 长度 255 → 1000
T3: 操作 A apply
```

**REST API 路径（无 force）**：
- 哈希不匹配 → 失败
- 不会执行任何修改

**REST API 路径（force=true）**：
- 放行，应用 T1 时计算的 diff
- 结果：`articles.title` 长度变为 **500**（覆盖 B 的修改）

**命令行路径**：
- 执行时比较：
  - 当前：长度 1000（B 修改后）
  - 目标：长度 500（快照中的值）
- diff：修改 1000 → 500
- 结果：`articles.title` 长度变为 **500**（覆盖 B 的修改）

#### 场景 4：操作 A 创建集合，操作 B 也创建同名集合

```
T1: 操作 A diff → 目标: 新建集合 products
T2: 操作 B → 新建集合 products (带字段 id, name)
T3: 操作 A apply
```

**REST API 路径（无 force）**：
- 哈希不匹配
- 详细检查：
  - diff 是 `NEW products`
  - 检查集合是否已存在？→ 已存在 ❌
- 抛出：`"Provided diff is trying to create collection \"products\" but it already exists..."`
- 结果：失败

**REST API 路径（force=true）**：
- 放行
- 但 `applyDiff` 会尝试创建已存在的集合
- 结果：取决于 `CollectionsService.createOne` 的实现
  - 代码中有检查：`if (existingCollections.includes(payload.collection)) throw`
  - 所以即使 force，也可能在执行阶段失败

**命令行路径**：
- 执行时比较：
  - 当前：有 products（B 创建的）
  - 目标：有 products（快照中的）
- 计算 diff：
  - 如果快照中的 products 与当前完全一致 → 无 diff
  - 如果有差异 → EDIT 类型的 diff
- 结果：不会创建（已存在），但会同步字段差异

#### 场景 5：操作 A 删除集合，操作 B 修改该集合的字段

```
T1: 操作 A diff → 目标: 删除集合 articles
T2: 操作 B → 修改 articles.title 长度
T3: 操作 A apply
```

**REST API 路径（无 force）**：
- 哈希不匹配
- 详细检查：
  - diff 是 `DELETE articles`
  - 检查集合是否不存在？→ 存在 ✓
- 继续检查... 但哈希仍不匹配
- 最终抛出通用哈希不匹配错误
- 结果：失败

**REST API 路径（force=true）**：
- 放行
- 执行 `DELETE articles`
- 结果：✅ 整个集合被删除，B 的修改也一起消失

**命令行路径**：
- 执行时比较：
  - 当前：有 articles（B 修改了 title）
  - 目标：无 articles
- 计算 diff：`DELETE articles`
- 结果：✅ 整个集合被删除，B 的修改也一起消失

---

## 并发场景后果总结表

| 场景 | REST API (无 force) | REST API (force=true) | 命令行 |
|------|---------------------|----------------------|--------|
| **A 删除字段，B 新增另一字段** | ❌ 失败 | ⚠️ 执行 A 的删除，B 的新增保留 | ⚠️ 执行 A 的删除，**同时删除 B 的新增** ⚠️ |
| **A 修改字段属性，B 修改同一字段其他属性** | ❌ 失败 | ⚠️ 覆盖为 A 的值 | ⚠️ 覆盖为快照中的值 |
| **A 创建集合，B 也创建同名** | ❌ 失败（明确错误） | ⚠️ 可能执行阶段失败 | ✅ 无冲突（已存在则跳过或同步） |
| **A 删除集合，B 修改该集合** | ❌ 失败 | ⚠️ 删除整个集合（B 修改丢失） | ⚠️ 删除整个集合（B 修改丢失） |
| **A 新增字段，B 也新增同名字段** | ❌ 失败（明确错误） | ⚠️ 可能执行阶段失败 | ✅ 无冲突（已存在则同步） |

### 关键发现

1. **REST API 路径（无 force）是最安全的**：
   - 能检测到任何并发修改
   - 失败时不会产生部分变更
   - 提供明确的错误信息

2. **REST API 路径（force=true）是危险的**：
   - 绕过哈希验证，但仍可能在执行阶段失败
   - 结果不可预测，取决于具体变更类型

3. **命令行路径是"同步"而非"应用差异"**：
   - 它的行为是**将数据库同步到快照状态**
   - 不是**应用快照与历史状态的差异**
   - 这意味着：
     - 任何不在快照中的新增内容都会被删除
     - 任何在快照中被删除的内容都会被删除
     - 并发修改的结果取决于它们是否存在于目标快照中

---

## 最佳实践建议

### 选择正确的路径

| 使用场景 | 推荐路径 | 原因 |
|---------|---------|------|
| 程序化调用 / 自动化脚本 | REST API (diff → apply) | 有哈希验证，能检测并发冲突 |
| CI/CD 流水线 | REST API + force（谨慎） | 需要确保独占访问，或接受覆盖风险 |
| 开发环境版本控制 | 命令行 | 简单直接，通常是单用户环境 |
| 生产环境紧急修复 | Admin 界面交互式操作 | 可控，可预览，不易出错 |
| 多开发者协作 | 命令行 + 代码审查 + 定期同步 | 以代码仓库中的快照为准 |

### 避免冲突的策略

1. **单一管理入口原则**：
   - 团队约定：要么只用命令行，要么只用 Admin 界面管理 schema
   - 建议：开发环境用命令行（快照文件纳入版本控制），生产环境用 Admin 界面（仅紧急修改）

2. **快照同步流程**：
   ```
   开发环境修改 → 生成快照 → 提交代码 → 代码审查 → 应用到其他环境
   ```

3. **使用 REST API 路径的最佳实践**：
   - **总是先 diff，检查返回的 diff 是否符合预期**
   - **不建议在生产环境使用 force=true**
   - 如果必须使用 force，确保：
     - 没有其他并发操作
     - 已备份数据库
     - 已通过 dry-run 预览

4. **使用命令行路径的最佳实践**：
   - **每次 apply 前都要 --dry-run**
   - **仔细检查 diff，特别注意是否有意外的 DELETE**
   - **理解命令行是"同步"而非"应用"**：
     - 如果目标快照没有某个字段，即使它是刚刚新增的，也会被删除
     - 如果需要保留并发新增的内容，必须先更新目标快照

### 冲突检测与解决流程

**使用 REST API 路径时的冲突解决**：

```
1. 执行 diff → apply 时收到哈希不匹配错误
         ↓
2. 重新获取当前快照：GET /schema/snapshot
         ↓
3. 比较当前快照与目标快照
         ↓
4. 判断：
   ├─ 冲突是他人的有效修改 → 需要合并
   │   ├─ 更新目标快照，包含他人的修改
   │   └─ 重新执行 diff → apply
   │
   ├─ 冲突是无效/错误修改 → 需要覆盖
   │   └─ 谨慎使用 force=true（确保已备份）
   │
   └─ 需要保留两边的修改 → 手动合并
       ├─ 创建新的合并快照
       └─ 执行 diff → apply
```

**使用命令行路径时的冲突解决**：

```
1. 执行 --dry-run 查看将要执行的操作
         ↓
2. 检查是否有意外的 DELETE 或 CREATE
         ↓
3. 判断：
   ├─ 所有操作都符合预期 → 执行 apply
   │
   ├─ 有意外的 DELETE（并发新增的内容）
   │   ├─ 停止操作
   │   ├─ 更新目标快照，添加需要保留的内容
   │   └─ 重新 --dry-run 检查
   │
   └─ 有意外的 CREATE（并发删除的内容）
       ├─ 停止操作
       ├─ 更新目标快照，移除需要删除的内容
       └─ 重新 --dry-run 检查
```

---

## 代码引用

### 核心实现文件

| 功能 | 文件路径 |
|------|---------|
| REST API 控制器 | [`api/src/controllers/schema.ts`](api/src/controllers/schema.ts) |
| Schema 服务 | [`api/src/services/schema.ts`](api/src/services/schema.ts) |
| 命令行快照 | [`api/src/cli/commands/schema/snapshot.ts`](api/src/cli/commands/schema/snapshot.ts) |
| 命令行应用 | [`api/src/cli/commands/schema/apply.ts`](api/src/cli/commands/schema/apply.ts) |
| 快照生成 | [`api/src/utils/get-snapshot.ts`](api/src/utils/get-snapshot.ts) |
| 差异计算 | [`api/src/utils/get-snapshot-diff.ts`](api/src/utils/get-snapshot-diff.ts) |
| 差异应用 | [`api/src/utils/apply-diff.ts`](api/src/utils/apply-diff.ts) |
| 差异验证 | [`api/src/utils/validate-diff.ts`](api/src/utils/validate-diff.ts) |
| 哈希生成 | [`api/src/utils/get-versioned-hash.ts`](api/src/utils/get-versioned-hash.ts) |
| 集合服务 | [`api/src/services/collections.ts`](api/src/services/collections.ts) |
| Admin 界面快照 | [`app/src/modules/settings/routes/data-model/collections/collections.vue`](app/src/modules/settings/routes/data-model/collections/collections.vue) |
| SDK 快照命令 | [`sdk/src/rest/commands/schema/snapshot.ts`](sdk/src/rest/commands/schema/snapshot.ts) |
| SDK 差异命令 | [`sdk/src/rest/commands/schema/diff.ts`](sdk/src/rest/commands/schema/diff.ts) |
| SDK 应用命令 | [`sdk/src/rest/commands/schema/apply.ts`](sdk/src/rest/commands/schema/apply.ts) |

### 关键代码位置

| 关键逻辑 | 文件:行号 |
|---------|----------|
| 哈希验证快速路径 | [`api/src/utils/validate-diff.ts:91`](api/src/utils/validate-diff.ts#L91) |
| 哈希不匹配通用错误 | [`api/src/utils/validate-diff.ts:200-202`](api/src/utils/validate-diff.ts#L200-L202) |
| SchemaService.apply 重新获取快照 | [`api/src/services/schema.ts:39`](api/src/services/schema.ts#L39) |
| 命令行每次重算差异 | [`api/src/cli/commands/schema/apply.ts:65`](api/src/cli/commands/schema/apply.ts#L65) |
| /schema/diff 返回哈希 | [`api/src/controllers/schema.ts:112-113`](api/src/controllers/schema.ts#L112-L113) |

---

## 总结

### 两条路径的本质区别

1. **REST API 路径（diff → apply）**：
   - **设计理念**：两阶段提交，确保"计算差异时的状态"与"应用差异时的状态"一致
   - **核心保护**：哈希验证机制
   - **失败模式**：检测到不一致时失败，保护数据完整性
   - **适用场景**：程序化调用、自动化、需要并发安全的环境

2. **命令行路径**：
   - **设计理念**：单阶段提交，将数据库同步到目标快照状态
   - **核心行为**：每次执行时基于当前数据库状态重新计算差异
   - **保护机制**：无（除了 `--dry-run` 预览）
   - **适用场景**：开发环境、版本控制同步、单用户操作

### 并发场景下的行为对比

| 路径 | 并发感知 | 数据保护 | 结果可预测性 |
|------|---------|---------|-------------|
| REST API (无 force) | ✅ 能感知（哈希变化） | ✅ 失败保护 | ✅ 高（要么全成，要么全败） |
| REST API (force=true) | ❌ 不能感知 | ⚠️ 部分保护（执行阶段可能失败） | ⚠️ 中（取决于具体变更） |
| 命令行 | ❌ 不能感知（但能看到当前状态） | ❌ 无保护（会同步到快照状态） | ⚠️ 中（但行为是"同步"而非"应用"） |

### 最重要的建议

1. **理解命令行的"同步"语义**：
   - `directus schema apply` 不是"应用快照与上次 apply 时的差异"
   - 而是"将数据库同步到快照描述的状态"
   - 这意味着任何不在快照中的内容都会被删除

2. **REST API 路径是更安全的选择**：
   - 对于任何自动化或程序化操作，优先使用 REST API 路径
   - 利用哈希验证保护数据一致性
   - 只有在完全理解风险的情况下才使用 `force=true`

3. **`--dry-run` 是你的朋友**：
   - 无论使用哪条路径，在实际执行前都要预览
   - 特别注意是否有意外的 `DELETE` 操作
   - 命令行的 `--dry-run` 和 REST API 的 `diff` 阶段都能提供预览

4. **团队协作需要清晰的流程**：
   - 约定谁、在什么环境、使用什么方式管理 schema
   - 将快照文件纳入版本控制
   - 定期同步开发环境与生产环境的快照
