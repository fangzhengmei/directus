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

**关键事务语义**：整个导入过程在**单个数据库事务**中执行。

**JSON 导入** (`import-export.ts:340-451`)：

- 使用 `stream-json` 进行流式解析，避免大文件内存溢出
- 使用 `async.queue` 进行并行处理
- **整个导入在一个事务中**，任何错误都会导致完全回滚
- 错误追踪机制用于收集和聚合错误信息，便于报告

```typescript
async importJSON(collection: string, stream: Readable): Promise<void> {
    const extractJSON = StreamArray.withParser();
    const nestedActionEvents: ActionEventParams[] = [];
    const errorTracker = createErrorTracker();
    const isSingleton = this.schema.collections[collection]?.singleton ?? false;
    let timeout: NodeJS.Timeout;

    // 整个导入在一个事务中执行
    return transaction(this.knex, async (trx) => {
        const service = getService(collection, {
            knex: trx,
            schema: this.schema,
            accountability: this.accountability,
        });

        try {
            await new Promise<void>((resolve, reject) => {
                let rowNumber = 1;

                const saveQueue = queue(async (task: { data: Record<string, unknown>; rowNumber: number }) => {
                    if (errorTracker.shouldStop()) return;

                    try {
                        if (isSingleton) {
                            return await service.upsertSingleton(task.data, {
                                bypassEmitAction: (params) => nestedActionEvents.push(params),
                            });
                        } else {
                            return await service.upsertOne(task.data, {
                                bypassEmitAction: (params) => nestedActionEvents.push(params),
                            });
                        }
                    } catch (error) {
                        // 收集错误信息
                        for (const err of toArray(error)) {
                            errorTracker.addCapturedError(err, task.rowNumber);

                            if (errorTracker.shouldStop()) {
                                break;
                            }
                        }

                        // 达到错误限制或发生泛型错误时，拒绝 Promise
                        if (errorTracker.shouldStop()) {
                            saveQueue.kill();
                            destroyPipedStream(extractJSON, stream);
                            reject();  // 这会导致事务回滚
                        }

                        return;
                    }
                });

                stream.pipe(extractJSON);

                extractJSON.on('data', ({ value }: Record<string, any>) => {
                    // ... 处理数据
                    saveQueue.push({ data: value, rowNumber: rowNumber++ });
                });

                extractJSON.on('end', () => {
                    saveQueue.drain(() => {
                        // 如果有任何错误，拒绝 Promise → 事务回滚
                        if (errorTracker.hasErrors()) {
                            return reject();  // 事务回滚
                        }

                        // 只有完全没有错误时才提交事务
                        for (const nestedActionEvent of nestedActionEvents) {
                            emitter.emitAction(nestedActionEvent.event, nestedActionEvent.meta, nestedActionEvent.context);
                        }

                        return resolve();  // 事务提交
                    });
                });
                // ...
            });
        } catch (error) {
            if (!error && errorTracker.hasErrors()) {
                // 构建详细的错误信息返回给用户
                throw errorTracker.buildFinalErrors();
            }

            throw error;
        } finally {
            clearTimeout(timeout);
        }
    });
}
```

**CSV 导入** (`import-export.ts:453-635`)：

与 JSON 导入共享相同的事务语义和错误处理机制。

**阶段 5: 后台处理与通知** (`import-export.ts:290-337`)

```typescript
if (options?.background) {
    promise
        .then(async () => {
            await notify('Your import has been successful', `Your import in ${collection} has been successful.`);
        })
        .catch(async (error) => {
            logger.error(error, `Background import to ${collection} failed`);

            await notify(
                'Your import has failed',
                `Your import in ${collection} has failed.\n\n${(error as any).message ?? ''}`,
            );
        })
        .finally(async () => await decrementImportCount());
}
```

### 1.3 数据导出流程

#### 1.3.1 整体流程概览

```
HTTP Request → 权限校验(隐式) → 分批查询 → 格式转换 → 临时文件 → 上传文件 → 通知用户
                                    ↓
                              后台异步执行 (始终)
```

#### 1.3.2 详细流程分析

**阶段 1: 权限校验**

导出权限校验是隐式的，通过 `accountability` 传递给 `ItemsService`，在查询时自动过滤无权限的数据。

**阶段 2: 分批查询** (`import-export.ts:708-757`)

```typescript
const batchesRequired = Math.ceil(count / (env['EXPORT_BATCH_SIZE'] as number));

for (let batch = 0; batch < batchesRequired; batch++) {
    let limit = env['EXPORT_BATCH_SIZE'] as number;

    if (requestedLimit > 0 && (env['EXPORT_BATCH_SIZE'] as number) > requestedLimit - readCount) {
        limit = requestedLimit - readCount;
    }

    const result = await service.readByQuery({
        ...query,
        sort,
        limit,
        offset: batch * (env['EXPORT_BATCH_SIZE'] as number),
    });

    readCount += result.length;

    if (result.length) {
        // 写入临时文件
        await appendFile(
            tmpFile.path,
            this.transform(result, format, {
                includeHeader: batch === 0,
                includeFooter: batch + 1 === batchesRequired,
                fields: csvHeadings,
            }),
        );
    }
}
```

**阶段 3: 格式转换** (`import-export.ts:830-892`)

支持的格式：`csv`, `csv_utf8`, `json`, `xml`, `yaml`

**阶段 4: 文件上传与通知** (`import-export.ts:760-824`)

```typescript
// 上传到文件服务
const savedFile = await filesService.uploadOne(createReadStream(tmpFile.path), fileWithDefaults);

// 发送通知（只有当有用户上下文时）
if (this.accountability?.user) {
    const notificationsService = new NotificationsService({
        schema: this.schema,
    });

    const usersService = new UsersService({
        schema: this.schema,
    });

    const user = await usersService.readOne(this.accountability.user, {
        fields: ['first_name', 'last_name', 'email'],
    });

    const href = new Url(env['PUBLIC_URL'] as string).addPath('admin', 'files', savedFile).toString();

    const message = `
Hello ${userName(user)},

Your export of ${collection} is ready. <a href="${href}">Click here to view.</a>
`;

    await notificationsService.createOne({
        recipient: this.accountability.user,
        sender: this.accountability.user,
        subject: `Your export of ${collection} is ready`,
        message,
        collection: `directus_files`,
        item: savedFile,
    });
}
```

### 1.4 错误追踪机制

`createErrorTracker` (`import-export.ts:64-201`) 的作用：

> **重要**：错误追踪的目的是**收集和聚合错误信息以便更好地报告给用户**，而不是支持部分成功。整个导入在一个事务中执行，任何错误都会导致完全回滚。

- **字段级错误聚合**：相同字段的错误会聚合在一起
- **行号范围优化**：连续行号会显示为范围（如 `1-5` 而不是 `1,2,3,4,5`）
- **错误限制**：`MAX_IMPORT_ERRORS` 环境变量控制何时停止收集错误并回滚
- **泛型错误**：非字段相关的错误会立即导致回滚

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

## 3. 执行状态与进度反馈机制

### 3.1 数据导入的状态反馈

#### 3.1.1 同步模式 (`background=false`，默认)

**执行方式**：HTTP 请求同步等待导入完成

**状态反馈**：

| 阶段 | 反馈方式 | 说明 |
|------|----------|------|
| 进行中 | HTTP 连接保持 | 客户端处于等待状态 |
| 成功 | HTTP 200 OK | 响应体为空 |
| 失败 | HTTP 错误响应 | 包含详细错误信息（字段、行号等） |

**错误响应示例**：

```json
{
  "errors": [
    {
      "message": "Field \"title\" is required",
      "extensions": {
        "code": "FAILED_VALIDATION",
        "field": "title",
        "type": "any.required",
        "rows": [
          { "type": "range", "start": 1, "end": 5 },
          { "type": "lines", "rows": [7, 9, 12] }
        ]
      }
    }
  ]
}
```

#### 3.1.2 后台模式 (`background=true`)

**执行方式**：HTTP 立即返回，导入在后台异步执行

**状态反馈**：

| 阶段 | 反馈方式 | 说明 |
|------|----------|------|
| 请求接收 | HTTP 200 OK | 立即返回，导入开始 |
| 进行中 | 无主动反馈 | 客户端需要轮询或等待通知 |
| 成功 | 站内通知 | `NotificationsService` 发送 "Your import has been successful" |
| 失败 | 站内通知 | `NotificationsService` 发送 "Your import has failed"，包含错误信息 |

**重要限制**：
- 后台模式没有进度条或百分比反馈
- 只能通过最终的成功/失败通知了解结果
- 并发状态通过 `importCount` 追踪，但不对外暴露

### 3.2 数据导出的状态反馈

#### 3.2.1 执行方式

**始终是后台异步执行**（控制器中没有 `await`）：

```typescript
// controllers/utils.ts:173-176
// We're not awaiting this, as it's supposed to run async in the background
service.exportToFile(req.params['collection']!, sanitizedQuery, req.body.format, {
    file: req.body.file,
});

return next();  // 立即返回
```

#### 3.2.2 状态反馈

| 阶段 | 反馈方式 | 说明 |
|------|----------|------|
| 请求接收 | HTTP 204 No Content | 立即返回，导出开始 |
| 进行中 | 无主动反馈 | 客户端需要等待通知 |
| 成功 | 站内通知 + 文件记录 | 通知包含下载链接，文件保存到 `directus_files` |
| 失败 | 站内通知 | 通知 "Your export of {collection} failed" |

**成功通知示例**：

```
Hello {User Name},

Your export of articles is ready. <a href="/admin/files/{file_id}">Click here to view.</a>
```

### 3.3 Schema Snapshot 的状态反馈

#### 3.3.1 HTTP API 方式

**执行方式**：同步执行，HTTP 请求等待完成

| 操作 | 端点 | 成功响应 | 失败响应 |
|------|------|----------|----------|
| 创建快照 | `GET /schema/snapshot` | HTTP 200 + JSON 快照 | HTTP 错误响应 |
| 计算差异 | `POST /schema/diff` | HTTP 200 + diff JSON | HTTP 错误响应 |
| 应用差异 | `POST /schema/apply` | HTTP 204 No Content | HTTP 错误响应 |

**特点**：
- 没有后台执行选项
- 没有通知机制
- 客户端同步等待结果

#### 3.3.2 CLI 方式

**执行方式**：同步执行，输出到控制台

| 操作 | 命令 | 状态反馈 |
|------|------|----------|
| 创建快照 | `directus schema snapshot` | 输出到 stdout 或保存到文件 |
| 应用快照 | `directus schema apply <file>` | 控制台显示变更列表，交互式确认，执行日志 |

**CLI 应用快照的输出示例**：

```
The following changes will be applied:

Collections:
  - Create articles
  - Update categories

Fields:
  - Create articles.title
  - Update articles.status

Relations:
  - Create articles.category → categories

? Would you like to continue? (Y/n)
```

**特点**：
- 支持 `--dry-run` 预览变更
- 支持 `--yes` 跳过交互式确认
- 详细的控制台日志输出
- 没有通知机制

### 3.4 状态反馈机制对比表

| 特性 | 数据导入 (同步) | 数据导入 (后台) | 数据导出 | Schema Snapshot (API) | Schema Snapshot (CLI) |
|------|-----------------|-----------------|----------|------------------------|------------------------|
| **执行模式** | 同步 | 异步 | 异步 | 同步 | 同步 |
| **HTTP 响应时机** | 完成后 | 立即 | 立即 | 完成后 | N/A |
| **进度反馈** | 无 (等待) | 无 | 无 | 无 (等待) | 有 (控制台) |
| **成功通知** | 无 (HTTP 200) | 站内通知 | 站内通知 | 无 (HTTP 204) | 控制台日志 |
| **失败通知** | HTTP 错误响应 | 站内通知 | 站内通知 | HTTP 错误响应 | 控制台错误 |
| **详细错误信息** | 有 (响应体) | 有 (通知) | 无 (仅提示失败) | 有 (响应体) | 有 (控制台) |
| **预览/ dry-run** | 无 | 无 | 无 | 无 | 有 (`--dry-run`) |

### 3.5 通知机制详解

**通知服务** (`NotificationsService`) 用于后台操作的状态反馈：

**适用场景**：
- 数据导入 (`background=true`)
- 数据导出 (始终后台)

**不适用场景**：
- Schema Snapshot (无通知)
- 同步模式的导入导出

**通知内容**：

| 操作 | 成功主题 | 失败主题 |
|------|----------|----------|
| 导入 | "Your import has been successful" | "Your import has failed" |
| 导出 | "Your export of {collection} is ready" | "Your export of {collection} failed" |

**技术实现**：

```typescript
// services/import-export.ts:291-316
const notify = async (subject: string, message: string) => {
    try {
        if (!this.accountability?.user) return;

        const notificationsService = new NotificationsService({
            schema: this.schema,
        });

        const usersService = new UsersService({
            schema: this.schema,
        });

        const user = await usersService.readOne(this.accountability.user, {
            fields: ['first_name', 'last_name', 'email'],
        });

        await notificationsService.createOne({
            recipient: this.accountability.user,
            sender: this.accountability.user,
            subject,
            message: `Hello ${userName(user)},\n\n${message}\n`,
        });
    } catch (error) {
        logger.error(error, `Failed to notify user`);
    }
};
```

---

## 4. Data Export 与 Schema Snapshot 的边界

### 4.1 核心差异对比

| 维度 | Data Import/Export | Schema Snapshot |
|------|---------------------|-----------------|
| **操作对象** | 数据行 (items/records) | 元数据 (schema) |
| **数据内容** | 业务数据 | 表结构、字段定义、关系配置 |
| **权限要求** | create/update 权限 (细粒度) | admin 权限 (粗粒度) |
| **执行方式** | 导入: 同步/后台可选; 导出: 始终后台 | 始终同步 |
| **并发控制** | 有 (IMPORT_MAX_CONCURRENCY) | 无 (依赖数据库锁) |
| **事务边界** | 单次导入一个事务 | 整个 apply 一个事务 |
| **回滚语义** | 全有或全无 (任何错误都回滚) | 全有或全无 |
| **错误追踪** | 有 (用于报告，非部分成功) | 无详细错误聚合 |
| **通知机制** | 有 (后台模式) | 无 |
| **进度反馈** | 无 (仅最终状态) | CLI 有控制台输出 |
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

**Data Import/Export 使用场景**：

1. **数据迁移**：从 CSV/JSON 导入产品目录
2. **批量更新**：导出数据 → 外部编辑 → 重新导入
3. **数据备份**：定期导出关键业务数据
4. **数据集成**：与外部系统交换数据

**Schema Snapshot 使用场景**：

1. **环境同步**：开发 → 测试 → 生产 的 schema 迁移
2. **版本控制**：将 schema 纳入 Git 管理
3. **CI/CD 集成**：自动化部署 schema 变更
4. **环境重建**：快速恢复/创建一致的数据库结构
5. **团队协作**：共享和评审 schema 变更

### 4.4 权限模型差异

**Import/Export 权限**：

```
权限检查点:
1. 集合级别: 需要 create + update 权限
2. 字段级别: 通过 ItemsService 隐式检查
3. 系统集合: 需要 admin 权限

适用场景:
- 业务用户导入/导出自己的数据
- 基于角色的细粒度控制
```

**Schema Snapshot 权限**：

```
权限检查点:
1. 必须是 admin 用户 (accountability.admin === true)
2. 没有更细粒度的控制

适用场景:
- 系统管理员操作
- DevOps 自动化流程
```

### 4.5 事务与回滚语义对比

**Import 事务语义**：

```typescript
// 关键代码: import-export.ts:347-450
return transaction(this.knex, async (trx) => {
    // ... 所有写入操作
    
    // 队列处理完成后检查
    saveQueue.drain(() => {
        // 有任何错误 → reject → 事务回滚
        if (errorTracker.hasErrors()) {
            return reject();  // 事务回滚
        }
        
        // 完全没有错误 → resolve → 事务提交
        return resolve();  // 事务提交
    });
});
```

**关键点**：
- 整个导入在**一个数据库事务**中执行
- `errorTracker.hasErrors()` 检测到任何错误 → **事务回滚**
- 只有当**完全没有错误**时才提交事务
- `errorTracker` 的作用是**收集错误信息用于报告**，不是支持部分成功

**Snapshot Apply 事务语义**：

```typescript
// apply-diff.ts:56-356
await transaction(database, async (trx) => {
    // 1. 创建集合
    await createCollections(...);
    
    // 2. 删除集合
    await deleteCollections(...);
    
    // 3. 更新集合
    // 4. 处理字段
    // 5. 处理系统字段
    // 6. 处理关系
    
    // 任何一步抛出异常 → 事务回滚
});
```

**关键点**：
- 整个 apply 在**一个数据库事务**中执行
- 任何操作失败 → **完全回滚**
- 没有错误收集机制，失败即停止

### 4.6 何时使用哪个？

| 需求 | 推荐方案 | 原因 |
|------|----------|------|
| 迁移 1000 条产品数据 | Import | 操作数据行，事务保证一致性 |
| 复制开发环境的表结构到生产 | Snapshot | 操作元数据，保证结构一致 |
| 导出用户报表数据 | Export | 数据导出，支持多种格式，后台通知 |
| 在 Git 中追踪数据库结构变更 | Snapshot | 快照可版本化，支持 diff |
| 批量更新库存数量 | Import | 支持 upsert，事务保证 |
| 自动化部署新功能的表结构 | Snapshot | CI/CD 友好，CLI 支持 dry-run |
| 备份整个系统（结构+数据） | Snapshot + Export | 两者结合，完整备份 |
| 需要预览变更再确认 | Snapshot (CLI) | 支持 `--dry-run` |
| 需要后台执行不阻塞用户 | Import (background) / Export | 支持通知机制 |

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
| 通知服务 | `api/src/services/notifications.ts` | 需查看 |

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
| `MAX_IMPORT_ERRORS` | number | ？ | 收集错误数达到此值时停止并回滚 |

### 6.2 Schema Snapshot 相关

无专用环境变量，依赖数据库连接配置。

---

## 7. 总结

### 7.1 核心设计理念

1. **Import/Export**: 面向数据，事务一致
   - 流式处理大文件
   - **全有或全无事务**（任何错误都回滚）
   - 细粒度权限控制
   - 后台异步执行 + 通知机制
   - 错误追踪用于报告，非部分成功

2. **Schema Snapshot**: 面向结构，一致性优先
   - 全有或全无事务
   - admin 权限保护
   - diff-based 增量更新
   - 版本控制友好
   - CLI 支持 dry-run 预览

### 7.2 事务与回滚语义总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        导入事务语义 (关键修正)                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   之前错误理解: "部分成功模式"                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  行 1 ✅ 成功  │ 行 2 ❌ 失败  │ 行 3 ✅ 成功  │ 行 4 ❌ 失败      │   │
│   │  结果: 行 1, 3 保留，报告行 2, 4 错误                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   实际正确语义: "全有或全无"                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  行 1 ✅ 成功  │ 行 2 ❌ 失败  │ 行 3 ✅ 成功  │ 行 4 ❌ 失败      │   │
│   │  结果: 全部回滚，报告所有错误                                        │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   关键代码:                                                                   │
│   saveQueue.drain(() => {                                                    │
│       if (errorTracker.hasErrors()) {                                        │
│           return reject();  // ← 任何错误都导致事务回滚                      │
│       }                                                                       │
│       return resolve();  // ← 只有完全无错误才提交                           │
│   });                                                                         │
│                                                                              │
│   errorTracker 的作用:                                                        │
│   - 收集和聚合错误信息                                                        │
│   - 优化行号显示 (范围: 1-5, 离散: 7,9,12)                                  │
│   - 用于构建友好的错误响应                                                    │
│   - ❌ 不支持部分成功                                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3 协作模式总结

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
│   │  ┌─────────────────────────────────────────────┐     │                  │
│   │  │  关键: 全有或全无                             │     │                  │
│   │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐    │     │                  │
│   │  │  │ upsert  │  │ 错误    │  │ 事件    │    │     │                  │
│   │  │  │ One     │  │ 追踪    │  │ 缓存    │    │     │                  │
│   │  │  └─────────┘  └────┬────┘  └─────────┘    │     │                  │
│   │  │                      │                       │     │                  │
│   │  │  有错误? ──Yes──▶ 回滚                     │     │                  │
│   │  │                      │                       │     │                  │
│   │  │              No ──▶ 提交                    │     │                  │
│   │  └─────────────────────────────────────────────┘     │                  │
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
│                                             │                      │        │
│                                             │  任何失败 → 完全回滚 │        │
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

### 7.4 状态反馈机制总结

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           执行状态反馈机制                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        数据导入 (同步模式)                             │   │
│  │  HTTP Request ──▶ 等待 ──▶ 完成/失败                                  │   │
│  │                                      │                                  │   │
│  │                    ┌─────────────────┴─────────────────┐             │   │
│  │                    ▼                                   ▼             │   │
│  │              HTTP 200 OK                    HTTP 错误响应              │   │
│  │              (空响应体)                    (详细错误信息)               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                      数据导入 (后台模式) / 数据导出                    │   │
│  │  HTTP Request ──▶ 立即返回 200/204 ──▶ 后台执行                     │   │
│  │                                                         │              │   │
│  │                                    ┌────────────────────┴────────────┐ │   │
│  │                                    ▼                                 ▼ │ │   │
│  │                           站内通知 (成功)                   站内通知 (失败)│ │   │
│  │                           "导入成功"                          "导入失败" │ │   │
│  │                           "导出就绪 + 链接"                    "导出失败" │ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Schema Snapshot (HTTP API)                     │   │
│  │  HTTP Request ──▶ 等待 ──▶ 完成/失败                                  │   │
│  │                                      │                                  │   │
│  │                    ┌─────────────────┴─────────────────┐             │   │
│  │                    ▼                                   ▼             │   │
│  │              HTTP 200/204                    HTTP 错误响应              │   │
│  │              (快照JSON / 空)                  (错误信息)                │   │
│  │                                                                  │   │
│  │  ❌ 无后台模式  ❌ 无通知机制                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Schema Snapshot (CLI)                          │   │
│  │  命令执行 ──▶ 控制台输出 ──▶ 变更列表 ──▶ 交互式确认 ──▶ 执行        │   │
│  │                                                                         │   │
│  │  ✅ 支持 --dry-run 预览                                                │   │
│  │  ✅ 详细控制台日志                                                      │   │
│  │  ❌ 无通知机制                                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.5 边界清晰划分

| 维度 | 数据导入导出 | Schema Snapshot |
|------|-------------|-----------------|
| **是什么** | 数据行的批量操作 | 元数据结构的版本管理 |
| **为什么** | 业务数据的迁移、备份、集成 | 环境同步、版本控制、自动化部署 |
| **怎么做** | 流式解析 + 队列处理 + **全有或全无事务** + 错误报告 | 快照对比 + diff 计算 + 全有或全无事务 |
| **谁来做** | 业务用户 (基于角色权限) | 系统管理员 / DevOps |
| **状态反馈** | HTTP 响应 + 站内通知 (后台) | HTTP 响应 + CLI 控制台输出 |

这种清晰的边界划分使得 Directus 能够同时满足：
- **业务用户**：灵活、可靠的数据操作需求（事务保证）
- **技术团队**：可靠、可追溯的 schema 管理需求
