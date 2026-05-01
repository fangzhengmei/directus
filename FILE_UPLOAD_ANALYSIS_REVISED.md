# Directus 文件上传系统分析报告（修正版）

> 分析日期：2026-05-02  
> 分析版本：Directus 源码深度追踪  
> 本文档修正了之前分析中的关键错误，基于实际代码执行路径进行了精确追踪

---

## 目录

1. [执行摘要：关键修正点](#1-执行摘要关键修正点)
2. [中间件链与路由注册](#2-中间件链与路由注册)
3. [文件上传完整流程](#3-文件上传完整流程)
4. [权限校验机制的真实实现](#4-权限校验机制的真实实现)
5. [存储驱动系统](#5-存储驱动系统)
6. [资产变换系统](#6-资产变换系统)
7. [存储驱动切换的衔接机制](#7-存储驱动切换的衔接机制)
8. [关键代码路径速查](#8-关键代码路径速查)

---

## 1. 执行摘要：关键修正点

### ⚠️ 之前分析的错误 vs 实际实现

| 主题 | 之前的理解 | 实际实现 |
|------|-----------|---------|
| **createOne 权限校验** | 自动触发完整权限校验 | 通过 `processPayload` 检查集合和字段权限，**不调用 `validateAccess`** |
| **updateMany 权限校验** | 与 createOne 相同 | **明确调用 `validateAccess`** 进行完整校验 |
| **readByQuery 权限校验** | 在服务层显式校验 | 通过 `processAst` **注入权限过滤条件**到 SQL 查询中 |
| **FilesService 元数据更新** | 需要用户有 update 权限 | 使用 **sudo 模式**（无 accountability），绕过权限校验 |
| **存储位置选择** | 全局使用默认位置 | **每个文件有独立的 `storage` 字段**，访问时根据该字段选择驱动 |
| **TUS 分块上传** | 支持多存储位置 | **硬编码只使用第一个存储位置** |
| **useCollection 中间件** | 进行权限校验 | **仅设置 `req.collection`**，不做任何权限检查 |

---

## 2. 中间件链与路由注册

### 2.1 完整中间件执行顺序

根据 `api/src/app.ts` 的实际代码，请求经过的中间件顺序：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           HTTP 请求进入                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. express.json()                                                           │
│     - 解析 JSON 请求体                                                        │
│     - 限制：MAX_PAYLOAD_SIZE                                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. cookieParser()                                                           │
│     - 解析 Cookie 头                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. extractToken (api/src/middleware/extract-token.ts)                      │
│     - 从以下位置提取 token：                                                  │
│       • Authorization: Bearer <token>                                        │
│       • access_token 查询参数                                                 │
│       • directus_access_token Cookie                                          │
│     - 设置 req.token                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. authenticate (api/src/middleware/authenticate.ts)                       │
│     - 验证 token 的有效性                                                     │
│     - 加载用户信息和角色                                                      │
│     - ⚠️ **设置 req.accountability**（关键！后续权限依赖此值）               │
│     - accountability = null 表示未认证用户                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. schema (api/src/middleware/schema.ts)                                   │
│     - 加载数据库 schema 信息                                                  │
│     - 设置 req.schema                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  6. sanitizeQuery (api/src/middleware/sanitize-query.ts)                    │
│     - 清理和标准化查询参数                                                    │
│     - 设置 req.sanitizedQuery                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  7. requestCounter / cache                                                    │
│     - 请求计数和缓存中间件                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│  8. 路由处理                                                                  │
│     • /files → filesRouter (api/src/controllers/files.ts)                   │
│     • /assets → assetsRouter (api/src/controllers/assets.ts)                │
│     • /files/tus → tusRouter (仅 TUS_ENABLED=true)                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 useCollection 中间件的真实作用

**位置**：`api/src/middleware/use-collection.ts`

```typescript
const useCollection = (collection: string): RequestHandler =>
    asyncHandler(async (req, _res, next) => {
        req.collection = collection;  // ⚠️ 仅设置 collection，不做任何权限校验！
        next();
    });
```

**关键修正**：
- ❌ 之前认为：`useCollection` 进行权限校验
- ✅ 实际实现：**仅设置 `req.collection` 属性**，供其他中间件使用，本身不做任何权限检查

### 2.3 路由挂载顺序

根据 `api/src/app.ts:321-380`：

```typescript
// 认证相关（不需要 full schema）
app.use('/auth', authRouter);
app.use('/graphql', graphqlRouter);

// 常规 API 路由
app.use('/activity', activityRouter);
app.use('/access', accessRouter);
app.use('/assets', assetsRouter);      // 资产访问
app.use('/collections', collectionsRouter);
// ...

// ⚠️ 注意：TUS 路由在 files 之前
if (env['TUS_ENABLED'] === true) {
    app.use('/files/tus', tusRouter);
}

app.use('/files', filesRouter);        // 文件上传
app.use('/flows', flowsRouter);
// ...
```

---

## 3. 文件上传完整流程

### 3.1 普通上传（POST /files）

#### 流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      POST /files 上传流程                                     │
└─────────────────────────────────────────────────────────────────────────────┘

客户端
   │
   │ POST /files (multipart/form-data)
   │ 注意：字段必须在文件之前提供！
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  multipartHandler (api/src/controllers/files.ts:26)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 使用 Busboy 解析 multipart/form-data                                      │
│  2. busboy.on('field') → 收集表单字段到 payload                              │
│  3. busboy.on('file') → 处理文件流：                                          │
│     • 检查 MIME 类型白名单 (FILES_MIME_TYPE_ALLOW_LIST)                       │
│     • 检查文件大小限制 (FILES_MAX_UPLOAD_SIZE)                               │
│     • 生成 title（从 filename 格式化）                                         │
│     • 调用 FilesService.uploadOne()                                           │
│  4. 所有文件处理完成后，保存主键到 res.locals['savedFiles']                   │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  FilesService.uploadOne() (api/src/services/files.ts:81)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 1：初始化与准备                                                    │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 1.1 获取 StorageManager 实例                                           │   │
│  │     const storage = await getStorage();                               │   │
│  │                                                                       │   │
│  │ 1.2 如果是替换上传（有 primaryKey）：                                   │   │
│  │     • 从数据库读取现有文件记录                                           │   │
│  │     • 合并现有数据到 payload                                            │   │
│  │                                                                       │   │
│  │ 1.3 设置默认存储位置：                                                  │   │
│  │     payload.storage = toArray(env['STORAGE_LOCATIONS'])[0]!;       │   │
│  │     ⚠️ 新上传默认使用第一个存储位置                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 2：创建数据库记录（权限校验点）                                     │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 如果是新上传或不是替换：                                                 │   │
│  │                                                                       │   │
│  │ primaryKey = await this.createOne(payload, { emitEvents: false });  │   │
│  │                                                                       │   │
│  │ ⚠️ 关键：此处通过 ItemsService.createOne 触发权限校验                  │   │
│  │    → 调用 processPayload() 进行集合和字段权限检查                      │   │
│  │    → 不调用 validateAccess()（与 update/delete 不同）                 │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 3：生成文件名                                                       │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ const fileExtension =                                                  │   │
│  │     path.extname(payload.filename_download!) ||                       │   │
│  │     (payload.type && '.' + extension(payload.type)) || '';           │   │
│  │                                                                       │   │
│  │ const filenameDisk = primaryKey + (fileExtension || '');             │   │
│  │                                                                       │   │
│  │ ⚠️ 文件名格式：{primaryKey}.{extension}                                │   │
│  │    例如：550e8400-e29b-41d4-a716-446655440000.jpg                   │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 4：写入存储                                                         │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 获取存储驱动实例：                                                       │   │
│  │ const disk = storage.location(payload.storage);                       │   │
│  │                                                                       │   │
│  │ 如果是替换上传：                                                         │   │
│  │   • 先写入临时文件 temp_{filenameDisk}                                  │   │
│  │   • 后续成功后再移动到最终位置                                           │   │
│  │                                                                       │   │
│  │ 如果是新上传：                                                           │   │
│  │   await disk.write(payload.filename_disk, stream, payload.type);    │   │
│  │                                                                       │   │
│  │ 检查文件是否被截断：                                                     │   │
│  │ if ('truncated' in stream && stream.truncated === true)             │   │
│  │     throw new ContentTooLargeError();                                 │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 5：替换上传的原子性处理                                             │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 如果是替换上传 (isReplacement === true)：                               │   │
│  │                                                                       │   │
│  │ 5.1 更新数据库记录：                                                    │   │
│  │     await this.updateOne(primaryKey, payload, { emitEvents: false });│   │
│  │                                                                       │   │
│  │ 5.2 删除旧文件和缩略图：                                                │   │
│  │     for await (const filepath of disk.list(String(primaryKey))) {   │   │
│  │         await disk.delete(filepath);                                 │   │
│  │     }                                                                 │   │
│  │                                                                       │   │
│  │ 5.3 移动临时文件到最终位置：                                             │   │
│  │     await disk.move(tempFilenameDisk, payload.filename_disk);       │   │
│  │                                                                       │   │
│  │ ⚠️ 关键：如果任何步骤失败，调用 cleanUp() 删除临时文件/数据库记录        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 6：更新文件元数据（sudo 模式）                                      │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 6.1 获取文件大小：                                                      │   │
│  │     const { size } = await storage.location(payload.storage)         │   │
│  │         .stat(payload.filename_disk);                                 │   │
│  │     payload.filesize = size;                                          │   │
│  │                                                                       │   │
│  │ 6.2 提取图片元数据：                                                    │   │
│  │     const metadata = await extractMetadata(                          │   │
│  │         payload.storage,                                              │   │
│  │         payload as Parameters<typeof extractMetadata>[1]             │   │
│  │     );                                                                 │   │
│  │     → 包含 width, height, description, title, tags 等                 │   │
│  │                                                                       │   │
│  │ 6.3 ⚠️ 使用 sudo 权限更新（绕过权限校验）：                               │   │
│  │     const sudoFilesItemsService = new ItemsService(                 │   │
│  │         'directus_files',                                             │   │
│  │         { knex: this.knex, schema: this.schema }                     │   │
│  │         // ⚠️ 注意：没有 accountability！                              │   │
│  │     );                                                                 │   │
│  │                                                                       │   │
│  │     await sudoFilesItemsService.updateOne(                          │   │
│  │         primaryKey,                                                   │   │
│  │         { ...payload, ...metadata },                                  │   │
│  │         { emitEvents: false }                                         │   │
│  │     );                                                                 │   │
│  │                                                                       │   │
│  │ 为什么使用 sudo？                                                      │   │
│  │ • 用户可能只有 create 权限，没有 update 权限                           │   │
│  │ • 但文件上传后需要自动设置 filesize、width、height 等元数据            │   │
│  │ • 这些是系统自动生成的字段，不应该受用户权限限制                        │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 7：触发事件                                                        │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ if (opts?.emitEvents !== false) {                                    │   │
│  │     emitter.emitAction(                                               │   │
│  │         'files.upload',                                               │   │
│  │         { payload, key: primaryKey, collection: this.collection },  │   │
│  │         { database, schema, accountability }                         │   │
│  │     );                                                                │   │
│  │ }                                                                     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 TUS 分块上传

#### 关键限制

**位置**：`api/src/services/tus/server.ts:31-32`

```typescript
const location = toArray(env['STORAGE_LOCATIONS'] as string)[0]!;  // ⚠️ 硬编码只使用第一个位置！
const driver: Driver | TusDriver = storage.location(location);
```

**关键修正**：
- ❌ 之前认为：TUS 支持多存储位置
- ✅ 实际实现：**TUS 硬编码只使用 `STORAGE_LOCATIONS` 中的第一个位置**

#### 数据流

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  TUS 分块上传流程（简化版）                                                   │
└─────────────────────────────────────────────────────────────────────────────┘

客户端 (TUS 协议)
   │
   │ POST /files/tus (创建上传会话)
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  TusDataStore.create() (api/src/services/tus/data-store.ts:48)            │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 创建数据库记录（通过 ItemsService.createOne，带权限校验）                 │
│  2. 设置 fileData.storage = this.location（第一个存储位置）                   │
│  3. 调用 storageDriver.createChunkedUpload()                                 │
│  4. 更新 tus_id 和 tus_data 字段                                              │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   │ PATCH /files/tus/:id (上传数据块)
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  TusDataStore.write() (api/src/services/tus/data-store.ts:152)            │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 通过 tus_id 查找文件记录（带用户权限过滤）                                 │
│  2. 调用 storageDriver.writeChunk()                                          │
│  3. 使用 sudo 模式更新 tus_data.offset                                        │
│  4. 如果所有块上传完成：                                                       │
│     • 调用 storageDriver.finishChunkedUpload()                               │
│     • 如果是替换上传，删除旧文件并移动新文件                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 权限校验机制的真实实现

### 4.1 三种不同的权限校验方式

根据 `api/src/services/items.ts` 的实际代码，不同操作有不同的权限校验机制：

| 操作 | 校验方式 | 关键代码位置 |
|------|---------|-------------|
| **createOne** | `processPayload()` | items.ts:172-186 |
| **updateMany** | `validateAccess()` | items.ts:752-766 |
| **deleteMany** | `validateAccess()` | items.ts:1098-1111 |
| **readByQuery** | `processAst()` 注入过滤 | items.ts:530-533 |

### 4.2 createOne 的权限校验

**位置**：`api/src/services/items.ts:172-186`

```typescript
const payloadWithPresets = this.accountability
    ? await processPayload(
            {
                accountability: this.accountability,
                action: 'create',
                collection: this.collection,
                payload: payloadAfterHooks,
                nested: this.nested,
            },
            { knex: trx, schema: this.schema }
        )
    : payloadAfterHooks;  // ⚠️ 没有 accountability 时跳过所有校验
```

**processPayload 的校验逻辑**（`api/src/permissions/modules/process-payload/process-payload.ts`）：

```typescript
export async function processPayload(options: ProcessPayloadOptions, context: Context) {
    if (!options.accountability.admin) {
        // 1. 获取用户的权限策略
        policies = await fetchPolicies(options.accountability, context);
        
        permissions = await fetchPermissions(
            { action: options.action, policies, collections: [options.collection], ... },
            context
        );

        // 2. 检查是否有集合级别的权限
        if (permissions.length === 0) {
            throw createCollectionForbiddenError('', options.collection);
        }

        // 3. 检查字段级别权限
        const fieldsAllowed = uniq(permissions.map(({ fields }) => fields ?? []).flat());
        
        if (fieldsAllowed.includes('*') === false) {
            const fieldsUsed = Object.keys(options.payload);
            const notAllowed = difference(fieldsUsed, fieldsAllowed);
            
            if (notAllowed.length > 0) {
                throw createFieldsForbiddenError('', options.collection, notAllowed);
            }
        }

        // 4. 获取 validation 规则用于后续校验
        permissionValidationRules = permissions.map(({ validation }) => validation);
    }

    // 5. 应用权限预设 (presets)
    const presets = (permissions ?? []).map((permission) => permission.presets);
    const payloadWithPresets = assign({}, ...presets, options.payload);

    // 6. 校验 validation 规则和字段非空约束
    if (validationRules.length > 0) {
        const validationErrors = validatePayload({ _and: validationRules }, payloadWithPresets);
        if (validationErrors.length > 0) throw validationErrors;
    }

    return payloadWithPresets;
}
```

**关键修正**：
- ❌ 之前认为：`createOne` 调用 `validateAccess()`
- ✅ 实际实现：`createOne` **不调用 `validateAccess()`**，而是通过 `processPayload()` 进行：
  1. 集合级别权限检查（是否有该集合的 create 权限）
  2. 字段级别权限检查（payload 中的字段是否都被允许）
  3. 应用权限预设（presets）
  4. 校验 validation 规则

### 4.3 updateMany 的权限校验

**位置**：`api/src/services/items.ts:752-766`

```typescript
if (this.accountability) {
    await validateAccess(
        {
            accountability: this.accountability,
            action: 'update',
            collection: this.collection,
            primaryKeys: keys,
            fields: Object.keys(payloadAfterHooks),
        },
        { schema: this.schema, knex: this.knex }
    );
}
```

**validateAccess 的行为**（`api/src/permissions/modules/validate-access/validate-access.ts`）：

```typescript
export async function validateAccess(options: ValidateAccessOptions, context: Context) {
    if (options.accountability.admin === true) {
        return;  // 管理员直接通过
    }

    let access: boolean;

    if (options.primaryKeys) {
        // ⚠️ 有主键时，通过实际查询来验证
        const result = await validateItemAccess(options as Required<ValidateAccessOptions>, context);
        access = result.accessAllowed;
    } else {
        // 没有主键时，只检查集合级别的权限
        access = await validateCollectionAccess(options, context);
    }

    if (!access) {
        throw new ForbiddenError({ reason: `You don't have permission...` });
    }
}
```

**validateItemAccess 的工作方式**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  validateItemAccess 执行流程                                                 │
└─────────────────────────────────────────────────────────────────────────────┘

输入：primaryKeys = [id1, id2, id3]
      action = 'update'
      collection = 'directus_files'

   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 构建 AST 查询                                                             │
│  ast = {                                                                      │
│      type: 'root',                                                           │
│      name: 'directus_files',                                                 │
│      query: { limit: 3 },                                                    │
│      children: [...],  // 字段列表                                           │
│  }                                                                           │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. processAst() - 注入权限过滤条件                                          │
│     • 检查字段权限                                                            │
│     • 注入权限过滤条件（基于 permissions 表的 presets 和 validation）        │
│     • 例如：如果用户只能访问自己上传的文件，                                   │
│            会添加 { uploaded_by: { _eq: $CURRENT_USER } } 过滤              │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. 添加主键过滤                                                              │
│  ast.query.filter = { id: { _in: [id1, id2, id3] } }                       │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 执行查询 fetchPermittedAstRootFields()                                   │
│     → 生成类似这样的 SQL：                                                    │
│        SELECT * FROM directus_files                                          │
│        WHERE id IN ('id1', 'id2', 'id3')                                    │
│        AND (uploaded_by = 'current_user_id')  -- 权限过滤条件               │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  5. 检查结果数量                                                              │
│  expectedCount = 3                                                           │
│  hasAccess = items.length === expectedCount                                  │
│                                                                              │
│  ⚠️ 关键逻辑：                                                                │
│  - 如果用户有权限访问所有 3 个文件 → items.length = 3 → accessAllowed = true│
│  - 如果用户只能访问其中 2 个 → items.length = 2 → accessAllowed = false    │
│  - 因为权限校验要求用户对 ALL 请求的主键都有权限                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 readByQuery 的权限校验

**位置**：`api/src/services/items.ts:518-539`

```typescript
async readByQuery(query: Query, opts?: QueryOptions): Promise<Item[]> {
    // ... emitFilter 钩子 ...

    // 1. 从查询构建 AST
    let ast = await getAstFromQuery(
        { collection: this.collection, query: updatedQuery, accountability: this.accountability },
        { schema: this.schema, knex: this.knex }
    );

    // 2. ⚠️ 关键：processAst 注入权限过滤条件
    ast = await processAst(
        { ast, action: 'read', accountability: this.accountability },
        { knex: this.knex, schema: this.schema }
    );

    // 3. 执行查询（已包含权限过滤）
    const records = await runAst(ast, this.schema, this.accountability, { ... });

    // 4. 如果没有返回记录，抛出 ForbiddenError
    if (records === null) {
        throw new ForbiddenError();
    }

    return records;
}
```

**关键修正**：
- ❌ 之前认为：read 操作在服务层显式校验权限
- ✅ 实际实现：**权限过滤直接注入到 SQL 查询中**
  - 用户查询文件时，只会返回有权限访问的记录
  - 如果没有任何记录有权限，返回空数组或抛出 `ForbiddenError`
  - 这是**行级权限**的实现方式

### 4.5 资产访问的权限校验

**位置**：`api/src/services/assets.ts:231-251`

```typescript
async getAsset(id: string, transformation?: TransformationSet, ...) {
    // ...

    // 1. 获取系统公共文件列表（logo、favicon 等）
    const publicSettings = await this.knex
        .select('project_logo', 'public_background', 'public_foreground', 'public_favicon')
        .from('directus_settings')
        .first();
    const systemPublicKeys: string[] = Object.values(publicSettings || {});

    // 2. ⚠️ 权限校验条件
    if (!systemPublicKeys.includes(id) &&  // 不是系统公共文件
        this.accountability &&              // 已认证（不是 null）
        this.accountability.admin !== true  // 不是管理员
    ) {
        // 3. 执行完整的权限校验
        const { allowedRootFields, accessAllowed } = await validateItemAccess(
            {
                accountability: this.accountability,
                action: 'read',
                collection: 'directus_files',
                primaryKeys: [id],
                returnAllowedRootFields: true,
            },
            { knex: this.knex, schema: this.schema }
        );

        if (!accessAllowed) {
            throw new ForbiddenError({
                reason: `You don't have permission to perform "read" for collection "directus_files"...`,
            });
        }

        allowedFields = allowedRootFields;
    }

    // 4. 使用 sudo 模式读取文件记录
    const file = (await this.sudoFilesService.readOne(id, { limit: 1 })) as File;

    // ...
}
```

**权限校验逻辑总结**：

| 条件 | 是否校验 |
|------|---------|
| 文件是系统公共文件（logo、favicon 等） | ❌ 跳过校验，任何人可访问 |
| accountability = null（未认证用户） | ❌ 跳过校验（取决于后续的数据库检查） |
| accountability.admin = true（管理员） | ❌ 跳过校验 |
| 其他情况 | ✅ 执行完整的 validateItemAccess 校验 |

### 4.6 sudo 模式的使用场景

**什么是 sudo 模式**：
- 创建 Service 时**不传递 accountability**
- 所有权限校验都会被跳过（因为 `this.accountability = null` 或 `accountability.admin = true` 的检查逻辑）

**FilesService 中的 sudo 模式使用**：

| 位置 | 用途 | 原因 |
|------|------|------|
| `uploadOne` 中更新元数据 | 设置 filesize、width、height、uploaded_on 等 | 用户可能只有 create 权限，没有 update 权限，但这些是系统自动生成的字段 |
| `AssetsService.sudoFilesService` | 读取文件完整信息 | 权限校验已在之前完成，需要获取完整的文件记录（包括敏感字段） |
| `TusDataStore` 中更新 tus_data | 记录分块上传进度 | 内部系统操作，不应受用户权限限制 |

**sudo 模式示例**：

```typescript
// 普通模式（带权限校验）
const filesService = new FilesService({
    accountability: req.accountability,  // ⚠️ 传递 accountability
    schema: req.schema,
});

// sudo 模式（跳过权限校验）
const sudoFilesService = new ItemsService('directus_files', {
    knex: this.knex,
    schema: this.schema,
    // ⚠️ 没有 accountability！
});
```

---

## 5. 存储驱动系统

### 5.1 存储位置的真实工作机制

**关键发现**：每个文件有独立的 `storage` 字段

**位置**：
- `directus_files` 表的 `storage` 字段
- `api/src/services/files.ts:104`（新文件默认值）
- `api/src/services/files.ts:109`（获取驱动）

```typescript
// uploadOne 中
const payload = {
    storage: toArray(env['STORAGE_LOCATIONS'] as string)[0]!,  // 新上传默认使用第一个位置
    ...(existingFile ?? {}),  // ⚠️ 现有文件保留原有的 storage 字段
    ...clone(data),
};

const disk = storage.location(payload.storage);  // 根据文件的 storage 字段选择驱动
```

**访问文件时**：

```typescript
// AssetsService.getAsset 中
const exists = await storage.location(file.storage).exists(file.filename_disk);
//                                    ↑
//                         从文件记录读取 storage 字段

// FilesService.deleteMany 中
const disk = storage.location(file['storage']);
```

### 5.2 存储驱动的初始化流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  StorageManager 初始化流程（单例模式）                                        │
└─────────────────────────────────────────────────────────────────────────────┘

首次调用 getStorage()
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  1. 创建 StorageManager 实例                                                  │
│     const storage = new StorageManager();                                    │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  2. 注册驱动 (registerDrivers)                                               │
│     api/src/storage/register-drivers.ts                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│  2.1 扫描环境变量中使用的驱动：                                               │
│      for (const [key, value] of Object.entries(env)) {                     │
│          if (key.startsWith('STORAGE_') && key.endsWith('_DRIVER')) {      │
│              // 例如：STORAGE_LOCAL_DRIVER, STORAGE_S3_DRIVER               │
│              usedDrivers.push(value);                                        │
│          }                                                                   │
│      }                                                                       │
│                                                                              │
│  2.2 动态 import 驱动类：                                                    │
│      for (const driverName of usedDrivers) {                                │
│          const storageDriver = await getStorageDriver(driverName);         │
│          // getStorageDriver 会：                                           │
│          // • 检查别名映射：local → @directus/storage-driver-local         │
│          // • 动态 import 对应的包                                           │
│          // • 返回默认导出的 Driver 类                                       │
│                                                                              │
│          storage.registerDriver(driverName, storageDriver);                │
│      }                                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  3. 注册存储位置 (registerLocations)                                          │
│     api/src/storage/register-locations.ts                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  3.1 获取配置的位置列表：                                                     │
│      const locations = toArray(env['STORAGE_LOCATIONS'] as string);        │
│      // 例如：['local', 's3']                                                │
│                                                                              │
│  3.2 为每个位置创建驱动实例：                                                 │
│      locations.forEach((location) => {                                      │
│          location = location.trim();                                         │
│                                                                              │
│          // 从环境变量读取配置：                                              │
│          // STORAGE_{LOCATION}_DRIVER                                       │
│          // STORAGE_{LOCATION}_ROOT (for local)                             │
│          // STORAGE_{LOCATION}_KEY, STORAGE_{LOCATION}_SECRET (for s3)     │
│          // ... 等等                                                          │
│          const driverConfig = getConfigFromEnv(                            │
│              `STORAGE_${location.toUpperCase()}_`                           │
│          );                                                                   │
│                                                                              │
│          const { driver, ...options } = driverConfig;                       │
│                                                                              │
│          // 注册位置（创建 Driver 实例）                                      │
│          storage.registerLocation(location, {                               │
│              driver,                                                         │
│              options: { ...options, tus }                                    │
│          });                                                                 │
│      });                                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  4. 缓存实例                                                                  │
│     _cache.storage = storage;                                                │
│     // 后续调用 getStorage() 直接返回缓存实例                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5.3 驱动别名映射

**位置**：`api/src/storage/get-storage-driver.ts:3-10`

```typescript
export const _aliasMap: Record<string, string> = {
    local: '@directus/storage-driver-local',
    s3: '@directus/storage-driver-s3',
    supabase: '@directus/storage-driver-supabase',
    gcs: '@directus/storage-driver-gcs',
    azure: '@directus/storage-driver-azure',
    cloudinary: '@directus/storage-driver-cloudinary',
};
```

### 5.4 驱动接口定义

**位置**：`packages/storage/src/index.ts:35-59`

```typescript
// 基础驱动接口（所有驱动必须实现）
export declare class Driver {
    constructor(config: Record<string, unknown>);
    
    // 核心读写
    read(filepath: string, options?: ReadOptions): Promise<Readable>;
    write(filepath: string, content: Readable, type?: string): Promise<void>;
    delete(filepath: string): Promise<void>;
    
    // 文件信息
    stat(filepath: string): Promise<Stat>;      // { size, modified }
    exists(filepath: string): Promise<boolean>;
    
    // 文件操作
    move(src: string, dest: string): Promise<void>;
    copy(src: string, dest: string): Promise<void>;
    
    // 列表操作（用于删除缩略图等）
    list(prefix?: string): AsyncIterable<string>;
}

// TUS 扩展接口（支持分块上传的驱动实现）
export interface TusDriver extends Driver {
    get tusExtensions(): string[];  // 支持的 TUS 扩展
    
    createChunkedUpload(filepath: string, context: ChunkedUploadContext): Promise<ChunkedUploadContext>;
    writeChunk(filepath: string, content: Readable, offset: number, context: ChunkedUploadContext): Promise<number>;
    finishChunkedUpload(filepath: string, context: ChunkedUploadContext): Promise<void>;
    deleteChunkedUpload(filepath: string, context: ChunkedUploadContext): Promise<void>;
}
```

---

## 6. 资产变换系统

### 6.1 变换触发时机

**流程**：

```
请求 GET /assets/:id?width=300&height=300&fit=cover
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  AssetsController 校验层 (api/src/controllers/assets.ts:155-259)           │
├─────────────────────────────────────────────────────────────────────────────┤
│  1. 从数据库读取资产变换配置：                                                 │
│     • storage_asset_presets（自定义预设）                                    │
│     • storage_asset_transform（变换策略）                                    │
│                                                                              │
│  2. 解析和校验变换参数：                                                      │
│     • 如果有 transforms 参数，解析 JSON 数组                                  │
│     • 检查变换数量不超过 ASSETS_TRANSFORM_MAX_OPERATIONS                    │
│     • 检查变换方法是否在允许列表中                                            │
│                                                                              │
│  3. 根据策略校验是否允许变换：                                                │
│                                                                              │
│     storage_asset_transform = 'all'：                                        │
│       • 允许所有变换（预设 + 动态参数）                                       │
│       • 但 key 必须在允许列表中                                               │
│                                                                              │
│     storage_asset_transform = 'presets'：                                    │
│       • 只允许使用预设的变换                                                  │
│       • 只能传递 key 参数，不能有其他变换参数                                 │
│                                                                              │
│     storage_asset_transform = 'none' 或其他：                                │
│       • 只允许系统预设（system-small-cover 等）                              │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  AssetsService.getAsset() (api/src/services/assets.ts:193)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 1：权限校验（见第 4.5 节）                                         │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 2：解析变换参数                                                    │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ const transforms = transformation                                     │   │
│  │     ? TransformationUtils.resolvePreset(transformation, file)        │   │
│  │     : [];                                                              │   │
│  │                                                                       │   │
│  │ resolvePreset 会：                                                     │   │
│  │ • 合并预设参数和动态参数                                                │   │
│  │ • 处理 width/height/fit 等参数为 resize 变换                          │   │
│  │ • 处理 format/quality 参数为 toFormat 变换                             │   │
│  │ • 处理焦点裁剪（focal_point_x/focal_point_y）                         │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 3：检查是否需要变换                                                │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 支持变换的格式：                                                        │   │
│  │ SUPPORTED_IMAGE_TRANSFORM_FORMATS = [                                 │   │
│  │     'image/jpeg', 'image/png', 'image/webp',                          │   │
│  │     'image/tiff', 'image/avif'                                         │   │
│  │ ]                                                                       │   │
│  │                                                                       │   │
│  │ if (type && transforms.length > 0 &&                                   │   │
│  │     SUPPORTED_IMAGE_TRANSFORM_FORMATS.includes(type)) {               │   │
│  │     // 执行变换流程                                                     │   │
│  │ } else {                                                                │   │
│  │     // 直接返回原图                                                     │   │
│  │     const assetStream = () => storage.location(file.storage)          │   │
│  │         .read(file.filename_disk, { range, version });               │   │
│  │ }                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 4：检查缓存（关键！）                                               │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 4.1 生成变换后的文件名：                                                 │   │
│  │                                                                       │   │
│  │     const assetFilename =                                              │   │
│  │         path.basename(file.filename_disk, ext) +                     │   │
│  │         getAssetSuffix(transforms) +                                  │   │
│  │         (maybeNewFormat ? `.${maybeNewFormat}` : ext);               │   │
│  │                                                                       │   │
│  │     // getAssetSuffix 使用 object-hash 生成唯一标识：                  │   │
│  │     const getAssetSuffix = (transforms) => {                          │   │
│  │         if (Object.keys(transforms).length === 0) return '';         │   │
│  │         return `__${hash(transforms)}`;                               │   │
│  │     };                                                                 │   │
│  │                                                                       │   │
│  │     示例：                                                              │   │
│  │     原文件：550e8400-...-446655440000.jpg                           │   │
│  │     变换后：550e8400-...-446655440000__a1b2c3d4.jpg                  │   │
│  │                                                                       │   │
│  │ 4.2 检查缓存是否存在：                                                   │   │
│  │     const exists = await storage.location(file.storage)               │   │
│  │         .exists(assetFilename);                                        │   │
│  │                                                                       │   │
│  │ 4.3 如果缓存存在：                                                       │   │
│  │     // 直接返回缓存文件，跳过变换                                        │   │
│  │     const assetStream = () => storage.location(file.storage)          │   │
│  │         .read(assetFilename, { range });                               │   │
│  │                                                                       │   │
│  │     return { stream, file, stat };                                     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                              ↓                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │ 阶段 5：执行变换（缓存不存在时）                                         │   │
│  ├──────────────────────────────────────────────────────────────────────┤   │
│  │ 5.1 安全检查：                                                          │   │
│  │     • 图片尺寸不能超过 ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION           │   │
│  │     • 并发变换数不能超过 ASSETS_TRANSFORM_MAX_CONCURRENT              │   │
│  │                                                                       │   │
│  │ 5.2 创建 Sharp 变换器：                                                 │   │
│  │     const transformer = getSharpInstance();                            │   │
│  │                                                                       │
│  │     // 自动旋转（根据 EXIF）                                            │   │
│  │     if (transforms.find(t => t[0] === 'rotate') === undefined)       │   │
│  │         transformer.rotate();                                          │   │
│  │                                                                       │
│  │ 5.3 应用所有变换：                                                       │   │
│  │     for (const [method, ...args] of transforms) {                    │   │
│  │         (transformer[method] as any).apply(transformer, args);       │   │
│  │     }                                                                 │   │
│  │                                                                       │
│  │     支持的变换方法（来自 TransformationMethods）：                       │   │
│  │     • resize - 调整尺寸                                                 │   │
│  │     • rotate - 旋转                                                     │   │
│  │     • extract - 裁剪区域                                                 │   │
│  │     • toFormat - 转换格式                                               │   │
│  │                                                                       │
│  │ 5.4 管道处理并写入存储：                                                 │   │
│  │     const readStream = await storage.location(file.storage)           │   │
│  │         .read(file.filename_disk, { range, version });                │   │
│  │                                                                       │
│  │     await storage.location(file.storage).write(                       │   │
│  │         assetFilename,                                                  │   │
│  │         readStream.pipe(transformer),  // 读取 → 变换 → 写入          │   │
│  │         type                                                           │   │
│  │     );                                                                 │   │
│  │                                                                       │
│  │ 5.5 返回变换后的文件流：                                                 │   │
│  │     const assetStream = () => storage.location(file.storage)          │   │
│  │         .read(assetFilename, { range, version });                     │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 系统预设变换

**位置**：`api/src/constants.ts:10-41`

```typescript
export const SYSTEM_ASSET_ALLOW_LIST: TransformationParams[] = [
    {
        key: 'system-small-cover',
        format: 'auto',
        transforms: [['resize', { width: 64, height: 64, fit: 'cover' }]],
    },
    {
        key: 'system-small-contain',
        format: 'auto',
        transforms: [['resize', { width: 64, fit: 'contain' }]],
    },
    {
        key: 'system-medium-cover',
        format: 'auto',
        transforms: [['resize', { width: 300, height: 300, fit: 'cover' }]],
    },
    {
        key: 'system-medium-contain',
        format: 'auto',
        transforms: [['resize', { width: 300, fit: 'contain' }]],
    },
    {
        key: 'system-large-cover',
        format: 'auto',
        transforms: [['resize', { width: 800, height: 800, fit: 'cover' }]],
    },
    {
        key: 'system-large-contain',
        format: 'auto',
        transforms: [['resize', { width: 800, fit: 'contain' }]],
    },
];
```

### 6.3 变换参数说明

**位置**：`api/src/constants.ts:43-54`

```typescript
export const ASSET_TRANSFORM_QUERY_KEYS = [
    'key',              // 使用预设的 key
    'transforms',       // 自定义变换数组（JSON 格式）
    'width',            // 目标宽度
    'height',           // 目标高度
    'format',           // 输出格式：jpg/png/webp/avif/auto
    'fit',              // 适应模式：cover/contain/inside/outside/fill
    'quality',          // 质量：1-100
    'withoutEnlargement', // 禁止放大图片
    'focal_point_x',    // 焦点 X 坐标（用于裁剪）
    'focal_point_y',    // 焦点 Y 坐标
] as const;
```

---

## 7. 存储驱动切换的衔接机制

### 7.1 关键发现：每个文件独立追踪存储位置

**核心机制**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  存储位置切换的核心原理                                                        │
└─────────────────────────────────────────────────────────────────────────────┘

数据库表：directus_files
┌─────────────────────────────────────────────────────────────────────────────┐
│ id          │ filename_disk                    │ storage  │ ...            │
├─────────────────────────────────────────────────────────────────────────────┤
│ uuid-001    │ uuid-001.jpg                    │ local    │ ...            │
│ uuid-002    │ uuid-002.png                    │ s3       │ ...            │
│ uuid-003    │ uuid-003.webp                   │ local    │ ...            │
│ uuid-004    │ uuid-004.tiff                   │ gcs      │ ...            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↑
                           每个文件有独立的 storage 字段
```

### 7.2 访问文件时的驱动选择

**代码位置**：多处

```typescript
// FilesService.uploadOne 中写入文件
const disk = storage.location(payload.storage);  // 根据当前文件的 storage 字段
await disk.write(payload.filename_disk, stream, payload.type);

// FilesService.deleteMany 中删除文件
for (const file of files) {
    const disk = storage.location(file['storage']);  // ⚠️ 每个文件使用自己的存储位置
    // ... 删除文件和缩略图
}

// AssetsService.getAsset 中读取文件
const exists = await storage.location(file.storage).exists(file.filename_disk);
const stream = await storage.location(file.storage).read(file.filename_disk, { version });
```

### 7.3 存储位置切换的场景分析

#### 场景 1：修改默认存储位置（新文件使用新位置）

**操作**：修改 `STORAGE_LOCATIONS` 环境变量，将新位置放在第一位

```
之前：STORAGE_LOCATIONS=local
之后：STORAGE_LOCATIONS=s3,local

结果：
┌─────────────────────────────────────────────────────────────────────────────┐
│ 文件 uuid-001 (storage=local) → 仍从 local 读取                            │
│ 文件 uuid-002 (storage=s3)    → 仍从 s3 读取                               │
│ 新上传的文件                    → storage=s3（使用第一个位置）                │
└─────────────────────────────────────────────────────────────────────────────┘
```

**关键**：**已有文件不受影响**，因为它们的 `storage` 字段记录了原始位置。

#### 场景 2：完全切换存储（迁移所有文件）

**需要执行的操作**：

```
1. 保持两个存储位置都在 STORAGE_LOCATIONS 中
   STORAGE_LOCATIONS=local,s3

2. 编写迁移脚本：
   • 遍历 directus_files 表中 storage='local' 的所有记录
   • 从 local 读取文件
   • 写入 s3
   • 更新数据库记录的 storage 字段为 's3'
   • 删除 local 中的原文件

3. 迁移完成后，可以修改 STORAGE_LOCATIONS（可选）
```

#### 场景 3：移除不再使用的存储位置

**风险点**：

```
❌ 错误操作：
STORAGE_LOCATIONS=s3  (移除了 local)

后果：
• 文件 uuid-001 (storage=local) 尝试访问时：
  storage.location('local') 
  → 抛出 Error: Location "local" doesn't exist.
```

**正确操作**：
1. 先迁移所有文件到新位置
2. 更新数据库记录的 `storage` 字段
3. 确认没有文件使用旧位置后，再从 `STORAGE_LOCATIONS` 中移除

### 7.4 更新文件存储位置的代码

**位置**：`api/src/services/files.ts:363-452`（updateMany 方法中的特殊处理）

```typescript
// 当更新 filename_disk 时的处理（单文件操作）
if (keys.length === 1 && data.filename_disk) {
    // ...
    
    for (const key of keys) {
        await transaction(this.knex, async (trx) => {
            // ...
            
            const storage = await getStorage();
            const file = updatedFiles.get(key);
            
            if (!file || !file.filename_disk) return;
            
            const disk = storage.location(file['storage']);  // ⚠️ 使用文件当前的存储位置
            
            // 检查目标文件是否已存在
            const remoteFileExists = await disk.exists(data.filename_disk);
            
            for await (const filePath of disk.list(filePrefixPath)) {
                if (filePath === existingFilePath) {
                    if (!remoteFileExists) {
                        // 目标不存在：移动文件
                        await disk.move(filePath, updatedFilePath);
                        continue;
                    } else if (toBoolean(env['FILES_DELETE_ORIGINAL_ON_MOVE']) === false) {
                        // 目标存在且不删除原文件：跳过
                        continue;
                    }
                }
                
                // 其他文件（缩略图等）：总是删除
                await disk.delete(filePath);
            }
        });
    }
}
```

**关键发现**：
- `filename_disk` 可以更新，但**`storage` 字段的更新不会触发文件迁移**
- 如果要更改文件的存储位置，需要：
  1. 手动从旧位置读取文件
  2. 写入新位置
  3. 更新 `storage` 和 `filename_disk` 字段
  4. 删除旧位置的文件

### 7.5 缩略图的位置关联

**关键机制**：

```
原文件：
  storage: local
  filename_disk: uuid-001.jpg

变换后的缩略图命名：
  uuid-001__{hash}.jpg  (同样存储在 local)

存储驱动的 list 方法：
  disk.list('uuid-001')  // 列出所有以 uuid-001 开头的文件
                          // 包括原文件和所有缩略图
```

**代码位置**：
- `api/src/services/assets.ts:413-416` - 缩略文件名生成
- `api/src/services/files.ts:485-488` - 删除时列出所有关联文件

```typescript
// 删除文件时
for (const file of files) {
    const disk = storage.location(file['storage']);
    const filePrefix = path.parse(file['filename_disk']).name;
    
    // 删除原文件 + 所有缩略图
    for await (const filepath of disk.list(filePrefix)) {
        await disk.delete(filepath);
    }
}
```

**关键**：缩略图和原文件**总是存储在同一个位置**，通过文件名前缀关联。

---

## 8. 关键代码路径速查

### 8.1 文件上传流程

| 阶段 | 文件路径 | 关键方法/行号 |
|------|----------|--------------|
| 路由注册 | `api/src/app.ts:339` | `app.use('/files', filesRouter)` |
| Multipart 解析 | `api/src/controllers/files.ts:26` | `multipartHandler` |
| 核心上传逻辑 | `api/src/services/files.ts:81` | `uploadOne()` |
| 创建数据库记录 | `api/src/services/files.ts:125` | `this.createOne(payload, { emitEvents: false })` |
| 写入存储 | `api/src/services/files.ts:176-179` | `disk.write()` |
| 元数据提取 | `api/src/services/files.ts:222` | `extractMetadata()` |
| Sudo 更新元数据 | `api/src/services/files.ts:228-233` | `sudoFilesItemsService.updateOne()` |

### 8.2 权限校验

| 操作 | 文件路径 | 关键方法/行号 |
|------|----------|--------------|
| createOne 校验 | `api/src/services/items.ts:172` | `processPayload()` |
| updateMany 校验 | `api/src/services/items.ts:752` | `validateAccess()` |
| deleteMany 校验 | `api/src/services/items.ts:1098` | `validateAccess()` |
| readByQuery 注入过滤 | `api/src/services/items.ts:530` | `processAst()` |
| 资产访问校验 | `api/src/services/assets.ts:231` | `validateItemAccess()` |
| processPayload 实现 | `api/src/permissions/modules/process-payload/process-payload.ts:29` | `processPayload()` |
| validateAccess 实现 | `api/src/permissions/modules/validate-access/validate-access.ts:22` | `validateAccess()` |
| processAst 实现 | `api/src/permissions/modules/process-ast/process-ast.ts:19` | `processAst()` |

### 8.3 存储驱动

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| Driver 接口 | `packages/storage/src/index.ts:35` | 基础驱动接口定义 |
| TusDriver 接口 | `packages/storage/src/index.ts:48` | 分块上传扩展接口 |
| StorageManager | `packages/storage/src/index.ts:4` | 驱动管理器 |
| getStorage | `api/src/storage/index.ts:10` | 获取存储管理器（单例） |
| 驱动别名映射 | `api/src/storage/get-storage-driver.ts:3` | 驱动别名到包名映射 |
| 注册驱动 | `api/src/storage/register-drivers.ts:5` | 动态 import 并注册驱动 |
| 注册位置 | `api/src/storage/register-locations.ts:7` | 创建存储位置实例 |
| Local 驱动 | `packages/storage-driver-local/src/index.ts` | 本地文件系统驱动 |
| S3 驱动 | `packages/storage-driver-s3/src/index.ts` | S3 兼容存储驱动 |

### 8.4 资产变换

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| 变换参数解析 | `api/src/utils/transformations.ts:5` | `resolvePreset()` |
| 系统预设 | `api/src/constants.ts:10` | `SYSTEM_ASSET_ALLOW_LIST` |
| 变换参数键 | `api/src/constants.ts:43` | `ASSET_TRANSFORM_QUERY_KEYS` |
| 支持的格式 | `api/src/constants.ts:85` | `SUPPORTED_IMAGE_TRANSFORM_FORMATS` |
| 缩略文件名 | `api/src/services/assets.ts:413` | `getAssetSuffix()` |
| Sharp 实例 | `api/src/services/files/lib/get-sharp-instance.ts` | 获取 Sharp 实例 |
| 元数据提取 | `api/src/services/files/lib/extract-metadata.ts:6` | `extractMetadata()` |

### 8.5 TUS 分块上传

| 组件 | 文件路径 | 说明 |
|------|----------|------|
| TUS 服务端 | `api/src/services/tus/server.ts:24` | `TusServer` 类 |
| 数据存储 | `api/src/services/tus/data-store.ts:28` | `TusDataStore` 类 |
| ⚠️ 硬编码位置 | `api/src/services/tus/server.ts:31` | `toArray(env['STORAGE_LOCATIONS'])[0]` |

---

## 总结

### 关键修正点回顾

1. **权限校验不是统一的**：
   - `createOne` → `processPayload()`（集合+字段权限检查）
   - `updateMany`/`deleteMany` → `validateAccess()`（完整校验）
   - `readByQuery` → `processAst()`（注入过滤条件到 SQL）

2. **Sudo 模式的重要性**：
   - 文件上传后，元数据（filesize、width、height 等）使用 sudo 模式更新
   - 即使用户只有 create 权限，没有 update 权限，也能正常上传

3. **存储位置是文件级别的**：
   - 每个文件记录有独立的 `storage` 字段
   - 访问时根据该字段选择驱动
   - 修改 `STORAGE_LOCATIONS` 不影响已有文件

4. **TUS 有限制**：
   - 硬编码只使用第一个存储位置
   - 不支持多存储位置的分块上传

5. **useCollection 不做权限校验**：
   - 仅设置 `req.collection` 属性
   - 权限校验发生在 Service 层内部

---

*报告生成完毕*
