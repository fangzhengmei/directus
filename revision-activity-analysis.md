# Directus 内容修订记录、活动日志与回滚操作关系分析

## 概述

本文档分析 Directus 系统中内容修订记录（Revisions）、活动日志（Activity）和回滚操作（Rollback/Revert）三个机制之间的关系和工作原理。

---

## 一、核心概念

### 1. Activity（活动日志）

**定义**：活动日志记录系统中的操作事件，回答"谁在什么时间对什么做了什么"。

**核心字段**（`directus_activity` 表）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | 自增主键 | 日志唯一标识 |
| `action` | string(45) | 操作类型：create/update/delete/login/version-save |
| `user` | uuid | 执行操作的用户 ID |
| `timestamp` | timestamp | 操作时间 |
| `collection` | string | 操作涉及的集合名称 |
| `item` | string | 操作涉及的条目主键 |
| `ip` | string(50) | 客户端 IP 地址 |
| `user_agent` | string(255) | 客户端 UA 信息 |
| `origin` | string | 请求来源 |
| `comment` | text | 备注/评论 |

**操作类型枚举**（Action）：
- `create` - 创建条目
- `update` - 更新条目
- `delete` - 删除条目
- `login` - 用户登录
- `version-save` - 保存版本草稿

**相关代码**：
- 服务定义：`api/src/services/activity.ts:4-7`
- 数据库结构：`packages/system-data/src/fields/activity.yaml`

---

### 2. Revisions（修订记录）

**定义**：修订记录保存数据变更的具体内容快照，用于版本对比和回滚。

**核心字段**（`directus_revisions` 表）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | 自增主键 | 修订记录唯一标识 |
| `activity` | number | **关联 Activity 表的外键** |
| `collection` | string | 所属集合 |
| `item` | string/number | 所属条目主键 |
| `data` | JSON | 完整数据快照（用于回滚） |
| `delta` | JSON | 变更差异（仅变化的字段） |
| `parent` | number | 父修订记录（用于嵌套关系） |
| `version` | number | 关联的版本草稿 ID |

**相关代码**：
- 服务定义：`api/src/services/revisions.ts:5-57`
- 类型定义：`api/src/types/revision.ts:1-7`
- 数据库结构：`packages/system-data/src/fields/revisions.yaml`

---

### 3. Rollback/Revert（回滚操作）

**定义**：将数据恢复到某个修订记录保存的状态。

**实现位置**：`api/src/services/revisions.ts:10-24`

```typescript
async revert(pk: PrimaryKey): Promise<void> {
    const revision = await super.readOne(pk);
    
    if (!revision) throw new ForbiddenError();
    
    if (!revision['data']) 
        throw new InvalidPayloadError({ 
            reason: `Revision doesn't contain data to revert to` 
        });
    
    const service = new ItemsService(revision['collection'], {
        accountability: this.accountability,
        knex: this.knex,
        schema: this.schema,
    });
    
    await service.updateOne(revision['item'], revision['data']);
}
```

**工作原理**：
1. 读取指定 revision 记录
2. 验证 revision 存在且包含 `data` 字段
3. 使用 `ItemsService.updateOne()` 将数据恢复到 revision.data 的状态

---

## 二、三者关系架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         数据操作流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  触发操作 (create/update/delete/promote)                         │
│           │                                                      │
│           ▼                                                      │
│  ┌─────────────────┐      Accountability 配置                    │
│  │ ActivityService │ ──── (activity / activity+rev / null) ────► │
│  │  .createOne()   │                                            │
│  └────────┬────────┘                                            │
│           │                                                      │
│           │  返回 activity ID                                    │
│           ▼                                                      │
│  ┌─────────────────┐      accountability === 'all'              │
│  │ RevisionsService│ ──────────────────────────────────────►    │
│  │  .createOne()   │      才创建修订记录                         │
│  └────────┬────────┘                                            │
│           │                                                      │
│           │  revision.activity = activity.id                     │
│           ▼                                                      │
│  ┌──────────────────────────────────────────┐                   │
│  │  数据关系：                                │                   │
│  │                                          │                   │
│  │  directus_activity                      │                   │
│  │  ┌──────────────────────────────────┐    │                   │
│  │  │ id: 123                         │    │                   │
│  │  │ action: 'update'               │    │                   │
│  │  │ collection: 'articles'         │    │                   │
│  │  │ item: '45'                      │    │                   │
│  │  │ revisions: [o2m 关联]          │◄───┼───┐                │
│  │  └──────────────────────────────────┘    │   │                │
│  │                                          │   │                │
│  │  directus_revisions                    │   │                │
│  │  ┌──────────────────────────────────┐    │   │                │
│  │  │ id: 789                         │    │   │                │
│  │  │ activity: 123  ◄────────────────┼────┘   │                │
│  │  │ collection: 'articles'         │        │                │
│  │  │ item: '45'                      │        │                │
│  │  │ data: {...}  ◄─────────────────┼────────┘ 回滚使用        │
│  │  │ delta: {...}                   │        (版本对比使用)     │
│  │  └──────────────────────────────────┘    │                   │
│  │                                          │                   │
│  └──────────────────────────────────────────┘                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、创建流程分析

### 3.1 创建条目 (createOne)

**代码位置**：`api/src/services/items.ts:298-391`

**流程**：

```
1. 处理嵌套关系数据 (M2O/A2O/O2M)
        │
        ▼
2. 检查 accountability 配置
   if (opts.skipTracking !== true 
       && this.accountability 
       && this.schema.collections[this.collection]!.accountability !== null)
        │
        ├─── accountability === null ───► 跳过追踪
        │
        └─── accountability !== null ───► 继续
                 │
                 ▼
   3. 创建 Activity 记录
      ┌─────────────────────────────────┐
      │ activityService.createOne({     │
      │   action: Action.CREATE,        │
      │   user: this.accountability.user│
      │   collection: this.collection,  │
      │   ip: ...,                      │
      │   user_agent: ...,              │
      │   origin: ...,                  │
      │   item: primaryKey              │
      │ })                              │
      └─────────────────────────────────┘
                 │
                 ▼
   4. 检查是否需要创建 Revision
      if (this.schema.collections[this.collection]!.accountability === 'all')
                 │
                 ├─── accountability === 'activity' ───► 不创建 Revision
                 │
                 └─── accountability === 'all' ─────────► 创建 Revision
                          │
                          ▼
             ┌──────────────────────────────────────┐
             │ revisionsService.createOne({          │
             │   activity: activity,    ◄── 关联     │
             │   collection: this.collection,        │
             │   item: primaryKey,                   │
             │   data: revisionPayload, ◄── 快照     │
             │   delta: revisionPayload  ◄── 差异     │
             │ })                                    │
             └──────────────────────────────────────┘
```

### 3.2 更新条目 (updateOne/updateMany)

**代码位置**：`api/src/services/items.ts:850-946`

**与 createOne 的区别**：

1. 批量更新时使用 `createMany` 创建多条 activity 记录
2. 需要先读取当前数据作为 `data` 快照
3. `data` 是更新**前**的完整数据
4. `delta` 是本次变更的内容

```typescript
// 关键代码片段
const snapshots = await itemsService.readMany(keys, {
    fields: snapshotFields.length > 0 ? snapshotFields : ['*'],
});

const revisions = (
    await Promise.all(
        activity.map(async (activity, index) => ({
            activity: activity,                    // 关联 activity
            collection: this.collection,
            item: keys[index],
            data: snapshots[index] ? ... : null,   // 更新前的快照
            delta: payloadWithTypeCasting          // 更新的内容
        })),
    )
).filter((revision) => revision.delta);
```

### 3.3 删除条目 (deleteOne/deleteMany)

**代码位置**：`api/src/services/items.ts:1122-1185`

**关键点**：
- **只创建 Activity 记录，不创建 Revision 记录**
- 原因：数据已被删除，无法保存快照，也无法回滚

```typescript
await activityService.createMany(
    keysAfterHooks.map((key) => ({
        action: Action.DELETE,
        user: this.accountability!.user,
        collection: this.collection,
        ip: this.accountability!.ip,
        user_agent: this.accountability!.userAgent,
        origin: this.accountability!.origin,
        item: key,
    })),
    { bypassLimits: true },
);
// 注意：这里没有 RevisionsService 的调用
```

### 3.4 版本保存 (VersionsService.save)

**代码位置**：`api/src/services/versions.ts:220-303`

**流程**：
1. 创建 `action: 'version-save'` 的 Activity
2. 如果 accountability === 'all'，创建 Revision 并关联 version

```typescript
const activity = await activityService.createOne({
    action: Action.VERSION_SAVE,  // 特殊的 action 类型
    user: this.accountability?.user ?? null,
    collection,
    ...
});

if (trackingAccountability === 'all') {
    await revisionsService.createOne({
        activity,
        version: key,  // 关联版本草稿
        collection,
        item,
        data: revisionDelta,
        delta: revisionDelta,
    });
}
```

---

## 四、Accountability 配置详解

### 4.1 配置选项

集合级别的 accountability 配置决定追踪粒度：

| 配置值 | Activity | Revision | 说明 |
|--------|----------|----------|------|
| `null` | ❌ 不创建 | ❌ 不创建 | 完全不追踪 |
| `'activity'` | ✅ 创建 | ❌ 不创建 | 只记录操作，不记录内容 |
| `'all'` | ✅ 创建 | ✅ 创建 | 完整追踪，支持回滚 |

### 4.2 配置检查逻辑

```typescript
// 是否创建 Activity
if (
    opts.skipTracking !== true &&
    this.accountability &&
    this.schema.collections[this.collection]!.accountability !== null
)

// 是否创建 Revision
if (this.schema.collections[this.collection]!.accountability === 'all')
```

---

## 五、回滚机制深度分析

### 5.1 回滚触发条件

回滚操作通过 `RevisionsService.revert(pk)` 执行，需要满足：

1. Revision 记录存在
2. Revision 包含 `data` 字段（完整快照）
3. 用户有权限执行更新操作

### 5.2 回滚数据来源

| 操作类型 | revision.data 来源 | 可回滚？ |
|----------|-------------------|---------|
| create | 创建时的 payload | ✅ 可以 |
| update | 更新前的快照 | ✅ 可以（恢复到更新前） |
| delete | 无（不创建 revision） | ❌ 不可以 |
| version-save | 版本 delta | ✅ 可以 |

### 5.3 回滚的副作用

回滚本身是一个 `update` 操作，会触发：
1. 创建新的 Activity 记录（action: update）
2. 如果 accountability === 'all'，创建新的 Revision 记录

这意味着回滚操作本身也会被记录，可以再回滚到回滚之前的状态。

---

## 六、前端使用场景

### 6.1 修订记录列表

**位置**：`app/src/composables/use-revisions.ts`

前端通过 API 获取修订记录时，会关联查询 activity：

```typescript
// 按日期分组修订记录
const revisionsGroupedByDate = groupBy(
    response.data.data.filter((revision) => !!revision.activity),
    (revision) => {
        const date = new Date(new Date(revision.activity.timestamp).toDateString());
        return date;
    },
);
```

### 6.2 版本对比

**位置**：`app/src/views/private/components/comparison/use-comparison.ts`

使用 revision 的 `data` 和 `delta` 字段进行版本对比：

```typescript
const revisionDelta = Object.keys(revision.delta ?? {});
const revisionFields = new Set(getRevisionFields(revisionDelta, fields));
```

---

## 七、关键代码引用汇总

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| ActivityService 定义 | `api/src/services/activity.ts` | 4-7 |
| RevisionsService 定义 | `api/src/services/revisions.ts` | 5-57 |
| RevisionsService.revert | `api/src/services/revisions.ts` | 10-24 |
| createOne 追踪逻辑 | `api/src/services/items.ts` | 320-375 |
| updateMany 追踪逻辑 | `api/src/services/items.ts` | 850-940 |
| deleteMany 追踪逻辑 | `api/src/services/items.ts` | 1122-1150 |
| VersionsService.save | `api/src/services/versions.ts` | 220-303 |
| Activity 表结构 | `packages/system-data/src/fields/activity.yaml` | 1-80 |
| Revisions 表结构 | `packages/system-data/src/fields/revisions.yaml` | 1-34 |
| Revision 类型定义 | `api/src/types/revision.ts` | 1-7 |

---

## 八、总结

### 8.1 关系总结

```
活动日志 (Activity) ───o2m───► 修订记录 (Revisions)
       │                          │
       │ 记录"谁做了什么"          │ 记录"具体改了什么"
       │                          │
       ▼                          ▼
  审计追踪用途              版本对比 & 回滚
```

### 8.2 设计原则

1. **分离关注点**：Activity 负责审计，Revisions 负责数据恢复
2. **可选追踪**：通过 accountability 配置灵活控制追踪粒度
3. **事务一致性**：Activity 和 Revision 在同一事务中创建
4. **不可回滚的删除**：删除操作只记录日志，不保存快照

### 8.3 使用建议

- 对于需要审计但不需要回滚的集合，使用 `accountability: 'activity'`
- 对于需要完整版本控制的集合，使用 `accountability: 'all'`
- 对于临时或不重要的数据，使用 `accountability: null` 提升性能
