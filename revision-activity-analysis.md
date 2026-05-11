# Directus 内容修订记录、活动日志与回滚操作关系分析

## 概述

本文档深度分析 Directus 系统中内容修订记录（Revisions）、活动日志（Activity）和回滚操作（Rollback/Revert）三个机制之间的关系、数据含义、API 入口及权限链路。

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
| `revisions` | o2m | 关联的修订记录列表（一对多） |

**操作类型枚举**（Action）：
- `create` - 创建条目
- `update` - 更新条目
- `delete` - 删除条目
- `login` - 用户登录
- `version-save` - 保存版本草稿

**代码引用**：
- 服务定义：`api/src/services/activity.ts:4-7`
- 数据库结构：`packages/system-data/src/fields/activity.yaml:1-80`

---

### 2. Revisions（修订记录）

**定义**：修订记录保存数据变更的具体内容快照，用于版本对比和回滚。

**核心字段**（`directus_revisions` 表）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | 自增主键 | 修订记录唯一标识 |
| `activity` | number | **关联 Activity 表的外键**（m2o） |
| `collection` | string | 所属集合 |
| `item` | string/number | 所属条目主键 |
| `data` | JSON | 完整数据快照（用于回滚） |
| `delta` | JSON | 变更差异（仅变化的字段） |
| `parent` | number | 父修订记录（用于嵌套关系） |
| `version` | number | 关联的版本草稿 ID |

**代码引用**：
- 服务定义：`api/src/services/revisions.ts:5-57`
- 类型定义：`api/src/types/revision.ts:1-7`
- 数据库结构：`packages/system-data/src/fields/revisions.yaml:1-34`

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
┌─────────────────────────────────────────────────────────────────────┐
│                          数据操作流程                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  触发操作 (create/update/delete/version-save)                         │
│           │                                                          │
│           ▼                                                          │
│  ┌─────────────────┐      Accountability 配置                        │
│  │ ActivityService │ ──── (activity / activity+rev / null) ────────► │
│  │  .createOne()   │                                                │
│  └────────┬────────┘                                                │
│           │                                                          │
│           │  返回 activity ID                                        │
│           ▼                                                          │
│  ┌─────────────────┐      accountability === 'all'                  │
│  │ RevisionsService│ ─────────────────────────────────────────────► │
│  │  .createOne()   │      才创建修订记录                             │
│  └────────┬────────┘                                                │
│           │                                                          │
│           │  revision.activity = activity.id                         │
│           ▼                                                          │
│  ┌──────────────────────────────────────────────────────┐           │
│  │  数据关系：                                           │           │
│  │                                                      │           │
│  │  directus_activity                                 │           │
│  │  ┌──────────────────────────────────────────────┐   │           │
│  │  │ id: 123                                     │   │           │
│  │  │ action: 'update'                           │   │           │
│  │  │ collection: 'articles'                     │   │           │
│  │  │ item: '45'                                  │   │           │
│  │  │ revisions: [o2m → directus_revisions]      │◄──┼───┐       │
│  │  └──────────────────────────────────────────────┘   │   │       │
│  │                                                      │   │       │
│  │  directus_revisions                               │   │       │
│  │  ┌──────────────────────────────────────────────┐   │   │       │
│  │  │ id: 789                                     │   │   │       │
│  │  │ activity: 123  ◄────────────────────────────┼───┘   │       │
│  │  │ collection: 'articles'                     │       │       │
│  │  │ item: '45'                                  │       │       │
│  │  │ data: {...}  ◄─────────────────────────────┼───────┘ 回滚  │
│  │  │ delta: {...}                               │        使用   │
│  │  │ version: null (或版本ID)                    │               │
│  │  └──────────────────────────────────────────────┘               │
│  │                                                      │           │
│  └──────────────────────────────────────────────────────┘           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 三、Revision.data 与 Revision.delta 的精确含义

### 3.1 Create（创建）场景

**代码位置**：`api/src/services/items.ts:352-362`

```typescript
const revisionPayload = await payloadService.prepareDelta(
    omit(payloadWithPresets, relationalFields)
);

const revision = await revisionsService.createOne({
    activity: activity,
    collection: this.collection,
    item: primaryKey,
    data: revisionPayload,   // 创建时的 payload
    delta: revisionPayload,  // 同 data
});
```

**含义表**：

| 字段 | 内容来源 | 含义 |
|------|---------|------|
| `revision.data` | `payloadWithPresets`（用户提交的创建数据） | 创建时的完整数据快照 |
| `revision.delta` | 同 data | 同 data（创建时 delta = data） |

**回滚效果**：恢复到刚创建时的状态

---

### 3.2 Update（更新）场景

**代码位置**：`api/src/services/items.ts:814-911`

```typescript
// 步骤 1：先执行数据库 UPDATE
if (Object.keys(payloadWithTypeCasting).length > 0) {
    try {
        await trx(this.collection).update(payloadWithTypeCasting).whereIn(primaryKeyField, keys);
    } catch (err: any) {
        throw await translateDatabaseError(err, data);
    }
}

// ... 处理 O2M 关系 ...

// 步骤 2：在同一事务中读取更新后的数据作为快照
const snapshots = await itemsService.readMany(keys, {
    fields: snapshotFields.length > 0 ? snapshotFields : ['*'],
});

// 步骤 3：构建 revision
const revisions = (
    await Promise.all(
        activity.map(async (activity, index) => ({
            activity: activity,
            collection: this.collection,
            item: keys[index],
            data: Array.isArray(snapshots) && snapshots[index]
                ? await payloadService.prepareDelta(snapshots[index])
                : null,                    // 更新后的完整快照
            delta: await payloadService.prepareDelta(payloadWithTypeCasting),
                                        // 用户提交的更新内容
        })),
    )
).filter((revision) => revision.delta);
```

**关键执行顺序**（同一事务内）：

```
时间轴：─────────────────────────────────────────────────►

T1: 数据库中的当前数据状态（更新前）
     ↓
T2: trx(this.collection).update(payloadWithTypeCasting)  ───► 执行 UPDATE SQL
     ↓
T3: itemsService.readMany(keys)  ───► 读取到的是更新后的数据（同一事务可见）
     ↓
T4: 构建 revision：
     - revision.data = snapshots[index]（更新后的状态）
     - revision.delta = payloadWithTypeCasting（用户提交的变更）
```

**事务可见性说明**：

数据库 UPDATE（T2）和 snapshots 读取（T3）使用**同一个事务对象 `trx`**。在数据库事务中，后续的 SELECT 可以看到同一事务内之前执行的 UPDATE 结果。因此 `snapshots` 读取到的是**更新后**的数据。

**含义表**：

| 字段 | 内容来源 | 含义 | 示例 |
|------|---------|------|------|
| `revision.data` | `snapshots[index]`（更新后从数据库读取） | **更新后**的完整状态快照（该修订执行后的状态） | `{ title: "新标题", status: "draft" }` |
| `revision.delta` | `payloadWithTypeCasting`（用户提交的更新） | **本次变更**的字段内容 | `{ title: "新标题" }` |

**前端对比逻辑验证**：`app/src/views/private/components/comparison/use-comparison.ts:398-415`

```typescript
let incoming = revision.data || {};  // 当前修订的状态作为"新状态"
if (compareToOption === 'Previous') {
    previousRevision = findPreviousRevision(revision);
    if (previousRevision && previousRevision.data) {
        base = previousRevision.data;  // 前一个修订的状态作为"基准状态"
    }
}
```

前端用 **前一个 revision.data** 作为 base（旧状态），**当前 revision.data** 作为 incoming（新状态）进行对比。这验证了 `revision.data` 代表"该修订执行后的状态"。

**回滚效果**：恢复到该修订执行后的状态

**回滚验证代码**：`api/src/services/revisions.ts:10-24`

```typescript
// revert 方法直接使用 revision.data 进行恢复
await service.updateOne(revision['item'], revision['data']);
// 即：将数据恢复到 revision.data 的状态（该修订执行后的状态）
```

**回滚场景示例**：

假设修订历史：
- R1 (create): data = `{ title: "A", status: "draft" }`
- R2 (update: title="B"): data = `{ title: "B", status: "draft" }`
- R3 (update: status="published"): data = `{ title: "B", status: "published" }`

当前数据库状态 = R3.data

执行 `revert(R2)`：
- 使用 R2.data = `{ title: "B", status: "draft" }`
- 更新后数据库 = `{ title: "B", status: "draft" }`
- 即：**恢复到 R2 执行后的状态**

执行 `revert(R1)`：
- 使用 R1.data = `{ title: "A", status: "draft" }`
- 更新后数据库 = `{ title: "A", status: "draft" }`
- 即：**恢复到初始创建状态**

---

### 3.3 Delete（删除）场景

**代码位置**：`api/src/services/items.ts:1122-1154`

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

---

### 3.4 Version-save（版本草稿保存）场景

**代码位置**：`api/src/services/versions.ts:237-268`

```typescript
const activity = await activityService.createOne({
    action: Action.VERSION_SAVE,  // 特殊的 action 类型
    user: this.accountability?.user ?? null,
    collection,
    ip: this.accountability?.ip ?? null,
    user_agent: this.accountability?.userAgent ?? null,
    origin: this.accountability?.origin ?? null,
    item,
});

if (trackingAccountability === 'all') {
    const revisionsService = new RevisionsService({
        knex: this.knex,
        schema: this.schema,
    });

    await revisionsService.createOne({
        activity,
        version: key,      // 关联版本草稿 ID
        collection,
        item,
        data: revisionDelta,   // 版本变更内容
        delta: revisionDelta,  // 同 data
    });
}
```

**含义表**：

| 字段 | 内容来源 | 含义 |
|------|---------|------|
| `revision.data` | `revisionDelta`（版本变更内容） | 版本草稿中的变更内容 |
| `revision.delta` | 同 data | 同 data |
| `revision.version` | `key`（版本 ID） | 关联的版本草稿 |

---

### 3.5 汇总对比表

| 操作类型 | revision.data | revision.delta | revision.version | 可回滚？ |
|---------|--------------|---------------|-----------------|---------|
| **create** | 创建时的完整状态 | 同 data | null | ✅ 恢复到创建状态 |
| **update** | **更新后**的完整状态快照（该修订执行后的状态） | 本次提交的变更内容 | null | ✅ 恢复到该修订后的状态 |
| **delete** | 无（不创建 revision） | 无 | 无 | ❌ |
| **version-save** | 版本变更内容 | 同 data | 版本 ID | ⚠️ 有限制（只包含版本 delta） |

---

## 四、Version-save 关联 Revision 的回滚行为分析

### 4.1 Version-save Revision 的特殊性

**代码位置**：`api/src/services/versions.ts:259-266`

```typescript
await revisionsService.createOne({
    activity,
    version: key,      // ← 设置了 version 字段
    collection,
    item,
    data: revisionDelta,
    delta: revisionDelta,
});
```

### 4.2 回滚时的实际行为

当对 version-save 的 revision 执行 `revert()` 时：

**代码链路**：
1. `RevisionsService.revert(pk)` → `api/src/services/revisions.ts:10-24`
2. 读取 revision 记录（包含 `version` 字段但 revert 方法不使用它）
3. 调用 `ItemsService.updateOne(revision['item'], revision['data'])`
4. `ItemsService.updateOne` → `api/src/services/items.ts:647-650`
5. 实际执行更新，触发新的 activity 和 revision

### 4.3 行为限制

| 限制项 | 说明 | 代码位置 |
|--------|------|---------|
| **不更新版本草稿** | revert 只更新主条目，不修改版本草稿（`directus_versions` 表） | `api/src/services/revisions.ts:10-24` |
| **使用 revision.data** | 使用的是版本保存时的 delta，不是完整快照 | `api/src/services/versions.ts:233` |
| **版本状态不变** | 版本草稿的 `delta` 字段不会被清除或更新 | 无相关逻辑 |
| **产生新记录** | 回滚会产生新的 activity（action: update）和 revision | `api/src/services/items.ts:867-913` |

### 4.4 场景示例

假设场景：
1. 主条目状态：`{ title: "原文", status: "draft" }`
2. 创建版本草稿 v1，修改为：`{ title: "版本标题" }`
3. 保存版本草稿 → 产生 revision R1（version=v1.id, data={title:"版本标题"}）
4. 之后又有其他更新

执行 `revert(R1)`：
- 主条目变为：`{ title: "版本标题" }` （注意：status 字段**不会**被恢复，因为 R1.data 中没有它）
- 版本草稿 v1 保持不变
- 产生新的 activity 和 revision 记录这次"回滚更新"

---

## 五、REST 与 GraphQL 回滚入口

### 5.1 REST API 入口

**路由定义**：`api/src/controllers/utils.ts:96-108`

```typescript
router.post(
    '/revert/:revision',
    asyncHandler(async (req, _res, next) => {
        const service = new RevisionsService({
            accountability: req.accountability,
            schema: req.schema,
        });

        await service.revert(req.params['revision']!);
        next();
    }),
    respond,
);
```

**路由注册**：`api/src/app.ts:374`

```typescript
app.use('/utils', utilsRouter);
```

**完整端点**：
```
POST /utils/revert/{revision_id}
```

**请求示例**：
```http
POST /utils/revert/123
Authorization: Bearer <token>
```

**响应**：
- 成功：`200 OK`，无响应体
- 失败：标准错误响应

---

### 5.2 GraphQL 入口

**定义位置**：`api/src/services/graphql/resolvers/system-global.ts:406-420`

```typescript
utils_revert: {
    type: GraphQLBoolean,
    args: {
        revision: new GraphQLNonNull(GraphQLID),
    },
    resolve: async (_, args) => {
        const service = new RevisionsService({
            accountability: gql.accountability,
            schema: gql.schema,
        });

        await service.revert(args['revision']);
        return true;
    },
},
```

**GraphQL Mutation**：
```graphql
mutation {
    utils_revert(revision: "123")
}
```

**响应**：
```json
{
    "data": {
        "utils_revert": true
    }
}
```

---

### 5.3 入口对比表

| 维度 | REST | GraphQL |
|------|------|---------|
| 端点 | `POST /utils/revert/{revision_id}` | `mutation { utils_revert(revision: ID!) }` |
| 代码位置 | `api/src/controllers/utils.ts:96-108` | `api/src/services/graphql/resolvers/system-global.ts:406-420` |
| 参数 | URL 路径参数 | GraphQL 变量 |
| 返回值 | 200 OK（空） | `true` |

---

## 六、权限检查链路

### 6.1 权限链路总览

```
REST: POST /utils/revert/:id
  │
  ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 路由层（无权限检查）                                      │
│     api/src/controllers/utils.ts:96-108                      │
│     直接调用 RevisionsService.revert()                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. RevisionsService.revert()                               │
│     api/src/services/revisions.ts:10-24                     │
│     ├─ 读取 revision（继承 ItemsService.readOne）            │
│     │   → 需要 directus_revisions 集合的 read 权限           │
│     ├─ 检查 revision.data 存在性                             │
│     └─ 调用 ItemsService.updateOne()                         │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. ItemsService.updateOne()                               │
│     api/src/services/items.ts:647-650                      │
│     调用 updateMany([key], data, opts)                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. ItemsService.updateMany()                              │
│     api/src/services/items.ts:752-766                      │
│     权限检查点：                                             │
│     ┌─────────────────────────────────────────────────┐    │
│     │ if (this.accountability) {                      │    │
│     │     await validateAccess({                      │    │
│     │         action: 'update',                       │    │
│     │         collection: this.collection,            │    │
│     │         primaryKeys: keys,                      │    │
│     │         fields: Object.keys(payloadAfterHooks), │    │
│     │     }, ...);                                    │    │
│     │ }                                               │    │
│     └─────────────────────────────────────────────────┘    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. validateAccess()                                       │
│     api/src/permissions/modules/validate-access/           │
│     validate-access.ts:1-57                                │
│     ├─ admin 用户直接通过                                   │
│     ├─ 检查 collection 访问权限                             │
│     └─ 检查 item-level 权限（通过实际查询）                  │
└─────────────────────────────────────────────────────────────┘
```

---

### 6.2 各层权限检查细节

#### 层 1：路由层（无权限检查）

**代码**：`api/src/controllers/utils.ts:96-108`

```typescript
router.post(
    '/revert/:revision',
    asyncHandler(async (req, _res, next) => {
        const service = new RevisionsService({
            accountability: req.accountability,  // 传入 accountability
            schema: req.schema,
        });

        await service.revert(req.params['revision']!);
        next();
    }),
    respond,
);
```

**说明**：路由层不做任何权限检查，直接将 `req.accountability` 传给 Service 层。

---

#### 层 2：RevisionsService.revert()

**代码**：`api/src/services/revisions.ts:10-24`

```typescript
async revert(pk: PrimaryKey): Promise<void> {
    // 步骤 1：读取 revision（需要 directus_revisions 的 read 权限）
    const revision = await super.readOne(pk);
    // super.readOne 继承自 ItemsService，会触发权限检查

    if (!revision) throw new ForbiddenError();

    if (!revision['data']) 
        throw new InvalidPayloadError({ 
            reason: `Revision doesn't contain data to revert to` 
        });

    // 步骤 2：创建目标集合的 ItemsService
    const service = new ItemsService(revision['collection'], {
        accountability: this.accountability,  // 传入 accountability
        knex: this.knex,
        schema: this.schema,
    });

    // 步骤 3：执行更新（会触发目标集合的 update 权限检查）
    await service.updateOne(revision['item'], revision['data']);
}
```

**权限点**：
1. 读取 `directus_revisions` 集合 → 需要 `read` 权限
2. 更新目标集合（`revision['collection']`）→ 需要 `update` 权限

---

#### 层 3：ItemsService.updateMany() 权限检查

**代码**：`api/src/services/items.ts:752-766`

```typescript
if (this.accountability) {
    await validateAccess(
        {
            accountability: this.accountability,
            action: 'update',
            collection: this.collection,
            primaryKeys: keys,
            fields: Object.keys(payloadAfterHooks),  // 检查所有要更新的字段
        },
        {
            schema: this.schema,
            knex: this.knex,
        },
    );
}
```

**检查维度**：
| 维度 | 说明 |
|------|------|
| `action` | 必须是 `'update'` |
| `collection` | 目标集合（从 revision.collection 获取） |
| `primaryKeys` | 目标条目 ID（从 revision.item 获取） |
| `fields` | `revision.data` 中的所有字段（字段级权限检查） |

---

#### 层 4：validateAccess() 核心逻辑

**代码**：`api/src/permissions/modules/validate-access/validate-access.ts:1-57`

```typescript
export async function validateAccess(options: ValidateAccessOptions, context: Context) {
    // 集合不存在检查
    if (!options.skipCollectionExistsCheck 
        && options.collection in context.schema.collections === false) {
        throw createCollectionForbiddenError('', options.collection);
    }

    // Admin 用户直接通过
    if (options.accountability.admin === true) {
        return;
    }

    let access: boolean;

    // 有 primaryKeys 时，通过实际查询验证 item-level 权限
    if (options.primaryKeys) {
        const result = await validateItemAccess(options as Required<ValidateAccessOptions>, context);
        access = result.accessAllowed;
    } else {
        // 无 primaryKeys 时，只检查 collection-level 权限
        access = await validateCollectionAccess(options, context);
    }

    if (!access) {
        // 根据是否有 fields 参数抛出不同错误
        if (options.fields?.length ?? 0 > 0) {
            throw new ForbiddenError({
                reason: `You don't have permissions to perform "${options.action}" 
                        for the field(s) ${options.fields!.map(...).join(', ')} 
                        in collection "${options.collection}"...`,
            });
        }
        throw new ForbiddenError({
            reason: `You don't have permission to perform "${options.action}" 
                    for collection "${options.collection}"...`,
        });
    }
}
```

---

### 6.3 权限检查汇总表

| 检查阶段 | 代码位置 | 权限类型 | 检查内容 |
|---------|---------|---------|---------|
| 读取 revision | `api/src/services/revisions.ts:11` | `directus_revisions` 的 `read` | 能否读取该 revision 记录 |
| 目标集合权限 | `api/src/services/items.ts:752-766` | 目标集合的 `update` | 能否更新目标集合 |
| 目标条目权限 | `api/src/permissions/.../validate-item-access.ts` | 目标条目的 `update` | 能否更新该特定条目（item-level 权限） |
| 字段权限 | `api/src/services/items.ts:759` | 字段级 `update` | 能否更新 revision.data 中的所有字段 |

---

### 6.4 权限错误示例

| 场景 | 错误类型 | 错误信息 |
|------|---------|---------|
| 无目标集合 update 权限 | `ForbiddenError` | `"You don't have permission to perform \"update\" for collection \"articles\"..."` |
| 无某字段 update 权限 | `ForbiddenError` | `"You don't have permissions to perform \"update\" for the field(s) \"title\", \"status\" in collection \"articles\"..."` |
| 无 item-level 权限 | `ForbiddenError` | `"You don't have permission to perform \"update\" for collection \"articles\"..."` |
| 无 directus_revisions read 权限 | `ForbiddenError` | （继承自 ItemsService.readOne 的错误） |

---

## 七、前端回滚流程

### 7.1 前端触发链路

```
用户点击"恢复"按钮
    │
    ▼
revisions-sidebar-detail.vue 触发 @confirm
    │
    ▼
comparison-modal.vue 计算 restoreData
    │
    ├─ 模式 1：直接 REST API 回滚
    │   POST /utils/revert/{revision_id}
    │
    └─ 模式 2：前端计算差异后更新
        emit('confirm', restoreData)
        → 父组件接收后调用 API 更新
```

### 7.2 关键组件

**组件 1**：`app/src/views/private/components/revisions-sidebar-detail.vue:25,138`

```typescript
defineEmits(['revert']);

// 在模板中
@confirm="$emit('revert', $event)"
```

**组件 2**：`app/src/views/private/components/comparison/comparison-modal.vue:215-231`

```typescript
// revision 模式下：计算差异，通过 emit 传递给父组件
const restoreData: Record<string, any> = {};
const selectedFields = unref(selectedComparisonFields);

const delta = comparisonData.value!.incoming;
const base = comparisonData.value!.base;

for (const [field, newValue] of Object.entries(delta)) {
    if (selectedFields.length > 0 && !selectedFields.includes(field)) continue;
    const previousValue = base[field] ?? null;
    if (isEqual(newValue, previousValue)) continue;
    restoreData[field] = newValue;
}

emit('confirm', restoreData);  // 传递给父组件处理
```

**父组件处理（示例）**：`app/src/modules/content/routes/item.vue:602-607`

```typescript
function revert(values: Record<string, any>) {
    edits.value = {
        ...edits.value,
        ...values,
    };
}
```

---

## 八、Accountability 配置详解

### 8.1 配置选项

集合级别的 accountability 配置决定追踪粒度：

| 配置值 | Activity | Revision | 说明 |
|--------|----------|----------|------|
| `null` | ❌ 不创建 | ❌ 不创建 | 完全不追踪 |
| `'activity'` | ✅ 创建 | ❌ 不创建 | 只记录操作，不记录内容 |
| `'all'` | ✅ 创建 | ✅ 创建 | 完整追踪，支持回滚 |

### 8.2 配置检查逻辑

**代码位置**：
- Activity 创建检查：`api/src/services/items.ts:322-326`
- Revision 创建检查：`api/src/services/items.ts:346`

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

## 九、关键代码引用汇总

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| ActivityService 定义 | `api/src/services/activity.ts` | 4-7 |
| RevisionsService 定义 | `api/src/services/revisions.ts` | 5-57 |
| **RevisionsService.revert** | `api/src/services/revisions.ts` | **10-24** |
| **createOne revision.data 赋值** | `api/src/services/items.ts` | **352-362** |
| **updateMany revision.data/delta 赋值** | `api/src/services/items.ts` | **886-911** |
| **REST 回滚入口** | `api/src/controllers/utils.ts` | **96-108** |
| **GraphQL 回滚入口** | `api/src/services/graphql/resolvers/system-global.ts` | **406-420** |
| **权限检查 validateAccess** | `api/src/permissions/modules/validate-access/validate-access.ts` | **1-57** |
| **ItemsService.updateMany 权限检查** | `api/src/services/items.ts` | **752-766** |
| VersionsService.save | `api/src/services/versions.ts` | 220-303 |
| Activity 表结构 | `packages/system-data/src/fields/activity.yaml` | 1-80 |
| Revisions 表结构 | `packages/system-data/src/fields/revisions.yaml` | 1-34 |
| 路由注册 | `api/src/app.ts` | 374 |

---

## 十、总结

### 10.1 核心结论

| 问题 | 结论 | 代码依据 |
|------|------|---------|
| **update 场景下 revision.data** | **更新后**的完整状态快照（该修订执行后的状态） | `api/src/services/items.ts:816`（UPDATE 在先）、`api/src/services/items.ts:889-891`（snapshots 读取在后，同一事务可见） |
| **update 场景下 revision.delta** | 本次提交的变更内容 | `api/src/services/items.ts:908` |
| **回滚语义** | 恢复到目标修订执行后的状态 | `api/src/services/revisions.ts:91`（`service.updateOne(revision['item'], revision['data'])`） |
| **前端对比逻辑** | 用前一个 revision.data 作 base，当前 revision.data 作 incoming | `app/src/views/private/components/comparison/use-comparison.ts:398-415` |
| **回滚 REST 入口** | `POST /utils/revert/{revision_id}` | `api/src/controllers/utils.ts:96-108` |
| **回滚 GraphQL 入口** | `mutation { utils_revert(revision: ID!) }` | `api/src/services/graphql/resolvers/system-global.ts:406-420` |
| **version-save revision 回滚** | 只更新主条目，不修改版本草稿 | `api/src/services/revisions.ts:10-24` |
| **权限检查位置** | `ItemsService.updateOne()` 内部的 `validateAccess()` | `api/src/services/items.ts:752-766` |

### 10.2 回滚副作用

回滚本身是一个 `update` 操作，会触发：
1. 创建新的 Activity 记录（action: `update`）
2. 如果 accountability === 'all'，创建新的 Revision 记录

这意味着回滚操作本身也会被记录，可以再回滚到回滚之前的状态。

### 10.3 设计原则

1. **分离关注点**：Activity 负责审计，Revisions 负责数据恢复
2. **延迟权限检查**：回滚入口不检查权限，实际权限在 ItemsService.updateOne 中验证
3. **可选追踪**：通过 accountability 配置灵活控制追踪粒度
4. **事务一致性**：Activity 和 Revision 在同一事务中创建
5. **不可回滚的删除**：删除操作只记录日志，不保存快照

### 10.4 使用建议

- 对于需要审计但不需要回滚的集合，使用 `accountability: 'activity'`
- 对于需要完整版本控制的集合，使用 `accountability: 'all'`
- 对于临时或不重要的数据，使用 `accountability: null` 提升性能
- 回滚 version-save 的 revision 时，注意它只包含版本变更的 delta，不是完整快照
