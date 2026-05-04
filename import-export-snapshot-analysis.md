# Directus 数据导入导出与 Schema Snapshot 分析文档

## 1. 数据导入导出 (Import/Export) 机制

### 1.1 核心组件

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| `ImportService` | `api/src/services/import-export.ts:207-635` | 处理数据导入逻辑 |
| `ExportService` | `api/src/services/import-export.ts:637-925` | 处理数据导出逻辑 |
| `utilsController` | `api/src/controllers/utils.ts:110-181` | 导入导出的 HTTP 端点 |
| `validateAccess` | `api/src/permissions/modules/validate-access/validate-access.ts` | 权限校验 |
| `useStore` | `api/src/utils/store.ts` | 并发控制和状态存储 |

### 1.2 数据导入流程

#### 1.2.1 整体流程概览

```
HTTP Request → 权限校验 → 文件解析 → 并发控制 → 数据库写入 → 通知用户
                                    ↓
                              后台/同步执行
```

#### 1.2.2 详细流程分析

**阶段 1: 权限校验** (`import-export.ts:224-250`)

```typescript
// 系统集合需要 admin 权限
if (this.accountability?.admin !== true && isSystemCollection(collection)) throw new ForbiddenError();

// 普通集合需要 create 和 update 权限
if (this.accountability) {
    await validateAccess(
        { accountability: this.accountability, action: 'create', collection },
        { schema: this.schema, knex: this.knex },
    );
    await validateAccess(
        { accountability: this.accountability, action: 'update', collection },
        { schema: this.schema, knex: this.knex },
    );
}
```

**阶段 2: 文件格式校验** (`import-export.ts:252-254`)

```typescript
if (['application/json', 'text/csv', 'application/vnd.ms-excel'].includes(mimetype) === false) {
    throw new UnsupportedMediaTypeError({ mediaType: mimetype, where: 'file import' });
}
```

**阶段 3: 并发控制** (`import-export.ts:256-269`)

使用 `useStore` 进行分布式并发控制（支持 Redis 或本地内存）：

```typescript
const limitReached = await store(async (store) => {
    const count = (await store.get('importCount')) ?? 0;
    if (count >= Number(env['IMPORT_MAX_CONCURRENCY'])) return true;
    await store.set('importCount', count + 1);
    return false;
});

if (limitReached) {
    throw new LimitExceededError({ category: 'Concurrent import' });
}
```

**阶段 4: 文件解析与数据库写入**

**JSON 导入** (`import-export.ts:340-451`)：

- 使用 `stream-json` 进行流式解析，避免大文件内存溢出
- 使用 `async.queue` 进行并行处理
- 事务保证原子性
- 错误追踪机制

```typescript
async importJSON(collection: string, stream: Readable): Promise<void> {
    const extractJSON = StreamArray.withParser();
    const errorTracker = createErrorTracker();
    
    return transaction(this.knex, async (trx) => {
        const service = getService(collection, { knex: trx, ... });
        
        const saveQueue = queue(async (task: { data: Record<string, unknown>; rowNumber: number }) => {
            if (errorTracker.shouldStop()) return;
            try {
                return await service.upsertOne(task.data, { ... });
            } catch (error) {
                errorTracker.addCapturedError(err, task.rowNumber);
                // ...
            }
        });
        
        stream.pipe(extractJSON);
        
        extractJSON.on('data', ({ value }) => {
            saveQueue.push({ data: value, rowNumber: rowNumber++ });
        });
        // ...
    });
}
```

**CSV 导入** (`import-export.ts:453-635`)：

- 先写入临时文件，再解析
- 使用 `papaparse` 解析 CSV
- 支持嵌套字段（通过 `lodash.set`）
- 与 JSON 导入共享相同的队列和错误追踪机制

**阶段 5: 后台处理与通知** (`import-export.ts:290-337`)

```typescript
if (options?.background) {
    promise
        .then(async () => {
            await notify('Your import has been successful', `Your import in ${collection} has been successful.`);
        })
        .catch(async (error) => {
            await notify('Your import has failed', `Your import in ${collection} has failed...`);
        })
        .finally(async () => await decrementImportCount());
}
```

### 1.3 数据导出流程

#### 1.3.1 整体流程概览

```
HTTP Request → 权限校验(隐式) → 分批查询 → 格式转换 → 临时文件 → 上传文件 → 通知用户
                                    ↓
                              后台异步执行
```

#### 1.3.2 详细流程分析

**阶段 1: 权限校验**

导出权限校验是隐式的，通过 `accountability` 传递给 `ItemsService`，在查询时自动过滤无权限的数据。

**阶段 2: 分批查询** (`import-export.ts:708-757`)

```typescript
const batchesRequired = Math.ceil(count / (env['EXPORT_BATCH_SIZE'] as number));

for (let batch = 0; batch < batchesRequired; batch++) {
    const result = await service.readByQuery({
        ...query,
        limit,
        offset: batch * (env['EXPORT_BATCH_SIZE'] as number),
    });
    // 写入临时文件
    await appendFile(tmpFile.path, this.transform(result, format, { ... }));
}
```

**阶段 3: 格式转换** (`import-export.ts:830-892`)

支持的格式：`csv`, `csv_utf8`, `json`, `xml`, `yaml`

```typescript
transform(input: Record<string, any>[], format: ExportFormat, options?: {...}): string {
    if (format === 'json') { ... }
    if (format === 'xml') { ... }
    if (format.startsWith('csv')) { ... }
    if (format === 'yaml') { ... }
}
```

**阶段 4: 文件上传与通知** (`import-export.ts:760-824`)

```typescript
// 上传到文件服务
const savedFile = await filesService.uploadOne(createReadStream(tmpFile.path), fileWithDefaults);

// 发送通知
await notificationsService.createOne({
    recipient: this.accountability.user,
    subject: `Your export of ${collection} is ready`,
    message: `Your export of ${collection} is ready. <a href="${href}">Click here to view.</a>`,
});
```

### 1.4 错误追踪机制

`createErrorTracker` (`import-export.ts:64-201`) 提供了完善的错误追踪：

- **字段级错误聚合**：相同字段的错误会聚合在一起
- **行号范围优化**：连续行号会显示为范围（如 `1-5` 而不是 `1,2,3,4,5`）
- **错误限制**：`MAX_IMPORT_ERRORS` 环境变量控制最大错误数
- **泛型错误**：非字段相关的错误单独处理

---

## 2. Schema Snapshot 机制

### 2.1 核心组件

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| `SchemaService` | `api/src/services/schema.ts` | HTTP API 层的快照服务 |
| `getSnapshot` | `api/src/utils/get-snapshot.ts` | 从数据库生成快照 |
| `applySnapshot` | `api/src/utils/apply-snapshot.ts` | 应用完整快照 |
| `applyDiff` | `api/src/utils/apply-diff.ts` | 应用增量差异 |
| `getSnapshotDiff` | `api/src/utils/get-snapshot-diff.ts` | 计算快照差异 |
| `schemaController` | `api/src/controllers/schema.ts` | HTTP 端点 |
| `snapshot CLI` | `api/src/cli/commands/schema/snapshot.ts` | CLI 导出命令 |
| `apply CLI` | `api/src/cli/commands/schema/apply.ts` | CLI 应用命令 |

### 2.2 快照数据结构

定义在 `packages/types/src/snapshot.ts`：

```typescript
type Snapshot = {
    version: number;           // 快照版本 (当前为 1)
    directus: string;          // Directus 版本
    vendor?: DatabaseClient;   // 数据库类型 (postgres, mysql, etc.)
    collections: SnapshotCollection[];   // 集合定义
    fields: SnapshotField[];             // 字段定义
    systemFields: SnapshotSystemField[]; // 系统字段索引
    relations: SnapshotRelation[];       // 关系定义
};
```

### 2.3 快照创建流程

#### 2.3.1 整体流程

```
权限校验 → 读取元数据 → 过滤系统项 → 排序规范化 → 输出快照
```

#### 2.3.2 详细分析 (`get-snapshot.ts:12-57`)

```typescript
export async function getSnapshot(options?: {...}): Promise<Snapshot> {
    // 1. 获取服务实例
    const collectionsService = new CollectionsService({...});
    const fieldsService = new FieldsService({...});
    const relationsService = new RelationsService({...});
    
    // 2. 并行读取所有元数据
    const [collectionsRaw, fieldsRaw, relationsRaw] = await Promise.all([
        collectionsService.readByQuery(),
        fieldsService.readAll(),
        relationsService.readAll(),
    ]);
    
    // 3. 过滤：排除系统表和未跟踪的表
    const collectionsFiltered = collectionsRaw.filter(
        (item) => excludeSystem(item) && excludeUntracked(item)
    );
    const fieldsFiltered = fieldsRaw.filter(
        (item) => excludeSystem(item) && excludeUntracked(item)
    );
    // 系统字段只保留有索引的
    const systemFieldsFiltered = fieldsRaw.filter((item) => systemFieldWithIndex(item));
    
    // 4. 排序和规范化
    const collectionsSorted = sortBy(mapValues(collectionsFiltered, sortDeep), ['collection'])
        .map((collection) => sanitizeCollection(collection));
    // ...
    
    return {
        version: 1,
        directus: version,
        vendor,
        collections: collectionsSorted,
        fields: fieldsSorted,
        systemFields: systemFieldsSorted,
        relations: relationsSorted,
    };
}
```

### 2.4 快照应用流程

#### 2.4.1 整体流程

```
CLI/API 输入 → 解析文件 → 计算差异 → 交互式确认(可选) → 事务应用 → 缓存刷新
```

#### 2.4.2 差异计算

`getSnapshotDiff` 使用 `deep-diff` 库计算差异：

```typescript
// 差异类型
const DiffKind = {
    NEW: 'N',      // 新增
    DELETE: 'D',   // 删除
    EDIT: 'E',     // 修改
    ARRAY: 'A',    // 数组变更
};
```

#### 2.4.3 应用差异 (`apply-diff.ts:37-372`)

**执行顺序非常重要**：

1. **创建集合**（按嵌套层级递归创建）
2. **删除集合**（先清理关系）
3. **更新集合**
4. **处理字段**（创建/更新/删除）
5. **处理系统字段**
6. **处理关系**

```typescript
await transaction(database, async (trx) => {
    // 1. 创建集合（递归处理嵌套集合）
    await createCollections(snapshotDiff.collections.filter(filterCollectionsForCreation));
    
    // 2. 删除集合
    if (collectionsToDelete.length > 0) await deleteCollections(collectionsToDelete);
    
    // 3. 更新集合
    for (const { collection, diff } of snapshotDiff.collections) {
        if (diff?.[0]?.kind === DiffKind.EDIT || diff?.[0]?.kind === DiffKind.ARRAY) {
            await collectionsService.updateOne(collection, newValues, mutationOptions);
        }
    }
    
    // 4. 处理字段
    for (const { collection, field, diff } of snapshotDiff.fields) {
        if (diff?.[0]?.kind === DiffKind.NEW) {
            await fieldsService.createField(collection, rhs, ...);
        }
        // 更新和删除...
    }
    
    // 5. 处理系统字段
    for (const { collection, field, diff } of snapshotDiff.systemFields) { ... }
    
    // 6. 处理关系
    for (const { collection, field, diff } of snapshotDiff.relations) { ... }
});

// 刷新缓存
await flushCaches();
```

### 2.5 集合创建的特殊处理

**嵌套集合递归创建** (`apply-diff.ts:59-106`)：

```typescript
const getNestedCollectionsToCreate = (currentLevelCollection: string) =>
    snapshotDiff.collections.filter(
        ({ diff }) => (diff[0] as DiffNew<Collection>).rhs?.meta?.group === currentLevelCollection,
    );

const createCollections = async (collections: CollectionDelta[]) => {
    for (const { collection, diff } of collections) {
        if (diff?.[0]?.kind === DiffKind.NEW && diff[0].rhs) {
            // 1. 创建集合时同时创建主键字段
            const fields = snapshotDiff.fields
                .filter((fieldDiff) => fieldDiff.collection === collection)
                .map((fieldDiff) => (fieldDiff.diff[0] as DiffNew<Field>).rhs);
            
            await collectionsService.createOne({ ...diff[0].rhs, fields }, mutationOptions);
            
            // 2. 从待处理字段中移除
            snapshotDiff.fields = snapshotDiff.fields.filter((fieldDiff) => fieldDiff.collection !== collection);
            
            // 3. 递归创建嵌套集合
            await createCollections(getNestedCollectionsToCreate(collection));
        }
    }
};
```

### 2.6 权限校验

Schema 操作需要 **admin 权限**：

```typescript
// schema.ts:29
async snapshot(): Promise<Snapshot> {
    if (this.accountability?.admin !== true) throw new ForbiddenError();
    // ...
}

// schema.ts:37
async apply(payload: SnapshotDiffWithHash, options?: {...}): Promise<void> {
    if (this.accountability?.admin !== true) throw new ForbiddenError();
    // ...
}
```

---

## 3. 协作机制详解

### 3.1 数据导入的协作流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HTTP Request (POST /utils/import/:collection)     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. 权限校验 (validateAccess)                                                 │
│    - 系统集合: 检查 admin 权限                                               │
│    - 普通集合: 检查 create + update 权限                                     │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. 并发控制 (useStore)                                                        │
│    - 读取 importCount                                                         │
│    - 检查是否超过 IMPORT_MAX_CONCURRENCY                                     │
│    - 原子性递增 importCount                                                   │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. 文件解析 (流式)                                                             │
│    JSON: stream-json → 逐行解析 → push 到队列                                │
│    CSV: 临时文件 → papaparse → 逐行解析 → push 到队列                        │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. 数据库写入 (async.queue + 事务)                                            │
│    - 队列并行处理 (默认并发数由 async.queue 控制)                             │
│    - 每个任务调用 service.upsertOne                                           │
│    - 错误追踪: addCapturedError                                               │
│    - 达到 MAX_IMPORT_ERRORS 或泛型错误时停止                                  │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
        ┌─────────────────────┐     ┌─────────────────────┐
        │  background=false   │     │  background=true    │
        │  (同步等待)         │     │  (后台执行)         │
        └──────────┬──────────┘     └──────────┬──────────┘
                   │                             │
                   ▼                             ▼
        ┌─────────────────────┐     ┌─────────────────────┐
        │ 5. 等待完成         │     │ 5. 立即返回 200     │
        │    成功/抛出错误    │     │    后台继续执行      │
        └──────────┬──────────┘     └──────────┬──────────┘
                   │                             │
                   ▼                             ▼
        ┌─────────────────────────────────────────────────┐
        │ 6. 资源清理与通知                                │
        │    - decrementImportCount                       │
        │    - 成功: 通知成功消息                          │
        │    - 失败: 通知错误消息                          │
        └─────────────────────────────────────────────────┘
```

### 3.2 数据导出的协作流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        HTTP Request (POST /utils/export/:collection)        │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. 参数校验                                                                   │
│    - 检查 query 和 format 必填                                                │
│    - sanitizeQuery 清理查询参数                                               │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. 启动后台任务 (不 await)                                                    │
│    service.exportToFile(...)  // 没有 await，立即返回                        │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. HTTP 响应 (立即返回)                                                       │
│    return next();  // 响应 204 No Content                                    │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                    (后台继续执行)
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. 分批查询 (EXPORT_BATCH_SIZE)                                               │
│    - 先查询总数                                                                │
│    - 循环分批读取                                                              │
│    - 每次读取后写入临时文件                                                    │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. 格式转换                                                                    │
│    - JSON: JSON.stringify with header/footer control                         │
│    - CSV: json2csv with flatten transform                                    │
│    - XML: js2xmlparser                                                        │
│    - YAML: js-yaml                                                            │
└─────────────────────────────────┬───────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. 文件上传与通知                                                             │
│    - 临时文件 → FilesService.uploadOne                                        │
│    - 通知用户: "Your export of {collection} is ready"                        │
│    - 清理临时文件                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 Schema Snapshot 的协作流程

**创建快照：**

```
┌─────────────────────────────────────────────────────────────────┐
│  GET /schema/snapshot 或 CLI: directus schema snapshot          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 权限校验: admin only                                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 并行读取元数据                                                 │
│    - CollectionsService.readByQuery()                            │
│    - FieldsService.readAll()                                     │
│    - RelationsService.readAll()                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 过滤与规范化                                                   │
│    - 排除系统表 (meta.system === true)                           │
│    - 排除未跟踪表 (meta === null)                                │
│    - 排序确保一致性                                               │
│    - 移除 id (omitID)                                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 输出                                                           │
│    API: JSON 响应                                                 │
│    CLI: 文件输出 (JSON/YAML) 或 stdout                           │
└─────────────────────────────────────────────────────────────────┘
```

**应用快照：**

```
┌─────────────────────────────────────────────────────────────────┐
│  POST /schema/apply 或 CLI: directus schema apply <file>       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 解析文件 (JSON/YAML)                                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 获取当前快照 (getSnapshot)                                     │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 计算差异 (getSnapshotDiff)                                     │
│    - 比较 collections/fields/relations/systemFields              │
│    - 生成 SnapshotDiff                                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 交互式确认 (CLI only, --yes 跳过)                             │
│    - 显示将执行的变更: Create/Update/Delete                      │
│    - 询问是否继续                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 事务应用 (applyDiff)                                           │
│    1. 创建集合 (递归处理嵌套)                                      │
│    2. 删除集合 (先清理关系)                                        │
│    3. 更新集合                                                    │
│    4. 处理字段                                                    │
│    5. 处理系统字段                                                │
│    6. 处理关系                                                    │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 缓存刷新 (flushCaches)                                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. Data Export 与 Schema Snapshot 的边界

### 4.1 核心差异对比

| 维度 | Data Import/Export | Schema Snapshot |
|------|---------------------|-----------------|
| **操作对象** | 数据行 (items/records) | 元数据 (schema) |
| **数据内容** | 业务数据 | 表结构、字段定义、关系配置 |
| **权限要求** | create/update 权限 (细粒度) | admin 权限 (粗粒度) |
| **执行方式** | 支持后台异步 | 同步执行 |
| **并发控制** | 有 (IMPORT_MAX_CONCURRENCY) | 无 (依赖数据库锁) |
| **事务边界** | 单次导入一个事务 | 整个 apply 一个事务 |
| **错误处理** | 可容忍部分错误 (MAX_IMPORT_ERRORS) | 全有或全无 |
| **通知机制** | 有 (成功/失败通知) | CLI 有日志，API 无通知 |
| **使用场景** | 数据迁移、批量更新、备份恢复 | 环境同步、版本控制、CI/CD |

### 4.2 边界图示

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              Directus 系统                                    │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────┐    ┌─────────────────────────────┐       │
│  │     Schema (元数据)          │    │      Data (业务数据)         │       │
│  │  ┌───────────────────────┐  │    │  ┌───────────────────────┐  │       │
│  │  │ Collections           │  │    │  │ Table: articles       │  │       │
│  │  │ Fields                │  │    │  │ ┌─────┬────────┐     │  │       │
│  │  │ Relations             │  │    │  │ │ id  │ title  │     │  │       │
│  │  │ Permissions (meta)    │  │    │  │ ├─────┼────────┤     │  │       │
│  │  └───────────────────────┘  │    │  │ │ 1   │ Hello  │     │  │       │
│  │                              │    │  │ │ 2   │ World  │     │  │       │
│  │  操作: Schema Snapshot       │    │  │ └─────┴────────┘     │  │       │
│  │  - snapshot (导出)           │    │  │                       │  │       │
│  │  - apply (应用)              │    │  │ 操作: Import/Export   │  │       │
│  │  - diff (比较)               │    │  │  - import (导入)      │  │       │
│  │                              │    │  │  - export (导出)      │  │       │
│  └─────────────────────────────┘    └─────────────────────────────┘       │
│                                                                              │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  边界: Schema 描述 "数据如何存储", Data 是 "实际存储的数据"                  │
│                                                                              │
└────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 典型使用场景

**Data Import/Export 使用场景：**

1. **数据迁移**：从 CSV/JSON 导入产品目录
2. **批量更新**：导出数据 → 外部编辑 → 重新导入
3. **数据备份**：定期导出关键业务数据
4. **数据集成**：与外部系统交换数据

**Schema Snapshot 使用场景：**

1. **环境同步**：开发 → 测试 → 生产 的 schema 迁移
2. **版本控制**：将 schema 纳入 Git 管理
3. **CI/CD 集成**：自动化部署 schema 变更
4. **环境重建**：快速恢复/创建一致的数据库结构
5. **团队协作**：共享和评审 schema 变更

### 4.4 权限模型差异

**Import/Export 权限：**

```
权限检查点:
1. 集合级别: 需要 create + update 权限
2. 字段级别: 通过 ItemsService 隐式检查
3. 系统集合: 需要 admin 权限

适用场景:
- 业务用户导入/导出自己的数据
- 基于角色的细粒度控制
```

**Schema Snapshot 权限：**

```
权限检查点:
1. 必须是 admin 用户 (accountability.admin === true)
2. 没有更细粒度的控制

适用场景:
- 系统管理员操作
- DevOps 自动化流程
```

### 4.5 错误处理策略差异

**Import 错误处理：**

```typescript
// 设计目标: 尽可能导入更多数据，记录问题
const errorTracker = createErrorTracker();

// 1. 字段级错误聚合
// 相同字段的错误会被分组，显示行号范围

// 2. 可配置的错误容忍度
// MAX_IMPORT_ERRORS 控制何时停止

// 3. 两种错误类型
// - 字段错误: 可继续导入其他行
// - 泛型错误: 立即停止

// 结果:
// - 部分成功: 导入有效行，报告错误行
// - 完全失败: 事务回滚
```

**Snapshot Apply 错误处理：**

```typescript
// 设计目标: 全有或全无，保证 schema 一致性
await transaction(database, async (trx) => {
    // 任何一步失败都会回滚整个事务
    await createCollections(...);
    await deleteCollections(...);
    // ...
});

// 结果:
// - 成功: 所有变更应用
// - 失败: 完全回滚到原始状态
```

### 4.6 何时使用哪个？

| 需求 | 推荐方案 | 原因 |
|------|----------|------|
| 迁移 1000 条产品数据 | Import | 操作数据行，支持部分成功 |
| 复制开发环境的表结构到生产 | Snapshot | 操作元数据，保证结构一致 |
| 导出用户报表数据 | Export | 数据导出，支持多种格式 |
| 在 Git 中追踪数据库结构变更 | Snapshot | 快照可版本化，支持 diff |
| 批量更新库存数量 | Import | 支持 upsert，增量更新 |
| 自动化部署新功能的表结构 | Snapshot | CI/CD 友好，支持 dry-run |
| 备份整个系统（结构+数据） | Snapshot + Export | 两者结合，完整备份 |

---

## 5. 关键代码位置速查

### 5.1 Import/Export

| 功能 | 文件 | 行号 |
|------|------|------|
| 主入口服务 | `api/src/services/import-export.ts` | 全部 |
| ImportService.import | `api/src/services/import-export.ts` | 218-338 |
| ImportService.importJSON | `api/src/services/import-export.ts` | 340-451 |
| ImportService.importCSV | `api/src/services/import-export.ts` | 453-635 |
| ExportService.exportToFile | `api/src/services/import-export.ts` | 653-825 |
| ExportService.transform | `api/src/services/import-export.ts` | 830-892 |
| HTTP 端点 | `api/src/controllers/utils.ts` | 110-181 |
| 错误追踪 | `api/src/services/import-export.ts` | 64-201 |
| 并发控制 | `api/src/utils/store.ts` | 全部 |

### 5.2 Schema Snapshot

| 功能 | 文件 | 行号 |
|------|------|------|
| SchemaService | `api/src/services/schema.ts` | 全部 |
| getSnapshot | `api/src/utils/get-snapshot.ts` | 全部 |
| applySnapshot | `api/src/utils/apply-snapshot.ts` | 全部 |
| applyDiff | `api/src/utils/apply-diff.ts` | 全部 |
| getSnapshotDiff | `api/src/utils/get-snapshot-diff.ts` | 需查看 |
| HTTP 端点 | `api/src/controllers/schema.ts` | 全部 |
| CLI snapshot | `api/src/cli/commands/schema/snapshot.ts` | 全部 |
| CLI apply | `api/src/cli/commands/schema/apply.ts` | 全部 |
| 类型定义 | `packages/types/src/snapshot.ts` | 全部 |

---

## 6. 环境变量配置

### 6.1 Import/Export 相关

| 变量名 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `IMPORT_MAX_CONCURRENCY` | number | ？ | 最大并发导入数 |
| `IMPORT_TIMEOUT` | string | `'1h'` | 导入超时时间 |
| `IMPORT_EXPORT_NAMESPACE` | string | ？ | Redis 命名空间 |
| `EXPORT_BATCH_SIZE` | number | ？ | 导出分批大小 |
| `MAX_IMPORT_ERRORS` | number | ？ | 最大容忍错误数 |

### 6.2 Schema Snapshot 相关

无专用环境变量，依赖数据库连接配置。

---

## 7. 总结

### 7.1 核心设计理念

1. **Import/Export**: 面向数据，灵活容错
   - 流式处理大文件
   - 部分成功模式
   - 细粒度权限控制
   - 后台异步执行

2. **Schema Snapshot**: 面向结构，一致性优先
   - 全有或全无事务
   - admin 权限保护
   - diff-based 增量更新
   - 版本控制友好

### 7.2 协作模式总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           数据导入导出协作模型                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   HTTP Request                                                               │
│       │                                                                      │
│       ▼                                                                      │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐              │
│   │ 权限校验│───▶│并发控制 │───▶│流式解析 │───▶│队列处理 │              │
│   │         │    │         │    │         │    │         │              │
│   │validate │    │ useStore│    │JSON/CSV │    │ async   │              │
│   │ Access  │    │         │    │ Stream  │    │ .queue  │              │
│   └─────────┘    └─────────┘    └─────────┘    └────┬────┘              │
│                                                         │                    │
│              ┌────────────────────────────────────────┘                    │
│              │                                                                 │
│              ▼                                                                 │
│   ┌─────────────────────────────────────────────────────┐                  │
│   │              数据库事务 (Transaction)                 │                  │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │                  │
│   │  │ upsert  │  │ 错误    │  │ 事件    │             │                  │
│   │  │ One     │  │ 追踪    │  │ 缓存    │             │                  │
│   │  └─────────┘  └─────────┘  └─────────┘             │                  │
│   └─────────────────────────────────────────────────────┘                  │
│                                                         │                    │
│              ┌────────────────────────────────────────┘                    │
│              │                                                                 │
│              ▼                                                                 │
│   ┌─────────────────────────────────────────────────────┐                  │
│   │              完成处理                                  │                  │
│   │  ┌─────────┐  ┌─────────┐  ┌─────────┐             │                  │
│   │  │ 并发    │  │ 通知    │  │ 临时    │             │                  │
│   │  │ 计数-1  │  │ 用户    │  │ 文件    │             │                  │
│   │  └─────────┘  └─────────┘  └─────────┘             │                  │
│   └─────────────────────────────────────────────────────┘                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           Schema Snapshot 协作模型                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   创建 (Snapshot)                              应用 (Apply)                  │
│   ┌──────────┐                                ┌──────────┐                   │
│   │ Admin    │                                │ 解析文件 │                   │
│   │ 权限校验 │                                │ (JSON/   │                   │
│   └────┬─────┘                                │ YAML)   │                   │
│        │                                      └────┬─────┘                   │
│        ▼                                           │                          │
│   ┌──────────┐                                     ▼                          │
│   │ 并行读取 │                              ┌──────────┐                      │
│   │ 元数据   │                              │ 获取当前 │                      │
│   └────┬─────┘                              │ 快照     │                      │
│        │                                    └────┬─────┘                      │
│        ▼                                         │                          │
│   ┌──────────┐                                     ▼                          │
│   │ 过滤系统 │                              ┌──────────┐                      │
│   │ 表/字段  │                              │ 计算差异 │                      │
│   └────┬─────┘                              │ (diff)   │                      │
│        │                                    └────┬─────┘                      │
│        ▼                                         │                          │
│   ┌──────────┐                                     ▼                          │
│   │ 排序规范化│                              ┌──────────┐                      │
│   │ 移除ID   │                              │ 交互式   │                      │
│   └────┬─────┘                              │ 确认     │ (CLI only)          │
│        │                                    └────┬─────┘                      │
│        ▼                                         │                          │
│   ┌──────────┐                                     ▼                          │
│   │ 输出     │                              ┌──────────────────────┐        │
│   │ (文件/   │                              │   事务应用           │        │
│   │ API)    │                              │  按顺序执行:         │        │
│   └──────────┘                              │  创建→删除→更新     │        │
│                                             │  字段→系统字段→关系  │        │
│                                             └──────────┬───────────┘        │
│                                                        │                      │
│                                                        ▼                      │
│                                               ┌──────────┐                   │
│                                               │ 缓存刷新 │                   │
│                                               │ (flush   │                   │
│                                               │ Caches)  │                   │
│                                               └──────────┘                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 边界清晰划分

| 维度 | 数据导入导出 | Schema Snapshot |
|------|-------------|-----------------|
| **是什么** | 数据行的批量操作 | 元数据结构的版本管理 |
| **为什么** | 业务数据的迁移、备份、集成 | 环境同步、版本控制、自动化部署 |
| **怎么做** | 流式解析 + 队列处理 + 事务 + 部分成功 | 快照对比 + diff 计算 + 全有或全无事务 |
| **谁来做** | 业务用户 (基于角色权限) | 系统管理员 / DevOps |

这种清晰的边界划分使得 Directus 能够同时满足：
- **业务用户**：灵活、容错的数据操作需求
- **技术团队**：可靠、可追溯的 schema 管理需求
