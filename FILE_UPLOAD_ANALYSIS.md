# Directus 文件上传系统分析报告

> 分析日期：2026-05-02  
> 分析版本：Directus 源码分析

---

## 目录

1. [概述](#1-概述)
2. [文件上传完整链路](#2-文件上传完整链路)
   - 2.1 客户端上传
   - 2.2 控制器层处理
   - 2.3 服务层核心逻辑
3. [存储驱动系统](#3-存储驱动系统)
   - 3.1 抽象接口设计
   - 3.2 驱动注册机制
   - 3.3 存储位置配置
   - 3.4 具体驱动实现
4. [资产变换系统](#4-资产变换系统)
   - 4.1 变换配置方式
   - 4.2 变换处理流程
   - 4.3 预设与动态变换
5. [访问权限校验](#5-访问权限校验)
   - 5.1 权限校验机制
   - 5.2 系统公共文件例外
6. [环境变量配置](#6-环境变量配置)
7. [关键代码参考](#7-关键代码参考)

---

## 1. 概述

Directus 的文件上传系统是一个多层架构的完整文件管理解决方案，涵盖了从客户端上传到服务端存储、缩略图变换、权限校验的完整链路。

### 核心组件

| 组件 | 路径 | 职责 |
|------|------|------|
| FilesController | `api/src/controllers/files.ts` | 处理 HTTP 上传请求 |
| FilesService | `api/src/services/files.ts` | 上传业务逻辑 |
| AssetsController | `api/src/controllers/assets.ts` | 资产访问和变换 API |
| AssetsService | `api/src/services/assets.ts` | 缩略图生成和资产获取 |
| StorageManager | `packages/storage/src/index.ts` | 存储驱动管理器 |
| Driver 接口 | `packages/storage/src/index.ts` | 存储驱动抽象接口 |

---

## 2. 文件上传完整链路

### 2.1 客户端上传

客户端通过以下方式上传文件：

1. **POST /files** - 使用 `multipart/form-data` 上传文件
2. **POST /files/import** - 通过 URL 导入文件
3. **PATCH /files/:pk** - 替换已存在的文件

**注意事项**：
- 表单字段顺序很重要：所有字段必须在文件之前提供
- 这允许服务端在上传实际文件前先确定存储位置和创建数据库记录

### 2.2 控制器层处理

#### Multipart 处理

`FilesController` 使用 **Busboy** 库处理 `multipart/form-data` 请求：

```typescript
// api/src/controllers/files.ts:26-133
export const multipartHandler: RequestHandler = (req, res, next) => {
    const busboy = Busboy({
        headers,
        defParamCharset: 'utf8',
        limits: {
            fileSize: bytes.parse(env['FILES_MAX_UPLOAD_SIZE'] as string),
        },
    });

    // 处理表单字段
    busboy.on('field', (fieldname, val) => {
        payload[fieldname] = fieldValue;
    });

    // 处理文件流
    busboy.on('file', async (_fieldname, fileStream, { filename, mimeType }) => {
        // MIME 类型校验
        const allowedPatterns = toArray(env['FILES_MIME_TYPE_ALLOW_LIST']);
        const mimeTypeAllowed = allowedPatterns.some((pattern) => minimatch(mimeType, pattern));
        
        // 调用 FilesService 上传
        const primaryKey = await service.uploadOne(fileStream, payloadWithRequiredFields, existingPrimaryKey);
    });

    req.pipe(busboy);
};
```

**关键校验点**：
1. **文件大小限制**：通过 `FILES_MAX_UPLOAD_SIZE` 环境变量配置
2. **MIME 类型白名单**：通过 `FILES_MIME_TYPE_ALLOW_LIST` 配置
3. **文件名必填校验**：拒绝无文件名的上传

### 2.3 服务层核心逻辑

#### FilesService.uploadOne 方法

这是文件上传的核心方法，位于 `api/src/services/files.ts:81-252`：

**完整执行流程**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    uploadOne 执行流程                             │
├─────────────────────────────────────────────────────────────────┤
│  1. 获取 StorageManager 实例                                      │
│         ↓                                                         │
│  2. 检查是否为替换上传（已有 primaryKey）                          │
│         ↓                                                         │
│  3. 合并现有文件数据与新 payload                                   │
│         ↓                                                         │
│  4. 获取存储位置（默认使用第一个配置的存储位置）                    │
│         ↓                                                         │
│  5. 新建上传：创建数据库记录                                        │
│         ↓                                                         │
│  6. 生成文件名：{primaryKey}.{extension}                          │
│         ↓                                                         │
│  7. 写入存储（新建→最终位置 / 替换→临时位置）                      │
│         ↓                                                         │
│  8. 检查文件是否被截断（stream.truncated）                        │
│         ↓                                                         │
│  9. 替换上传处理：更新数据库、删除旧文件、移动临时文件              │
│         ↓                                                         │
│ 10. 获取文件大小并更新 filesize                                    │
│         ↓                                                         │
│ 11. 提取元数据（图片尺寸、EXIF 等）                                │
│         ↓                                                         │
│ 12. 使用 sudo 权限更新数据库记录                                   │
│         ↓                                                         │
│ 13. 触发 files.upload 事件                                        │
└─────────────────────────────────────────────────────────────────┘
```

**关键设计亮点**：

1. **替换上传的原子性**：
   - 先写入临时文件 `temp_{filename}`
   - 更新数据库成功后再删除旧文件
   - 最后将临时文件移动到最终位置
   - 失败时自动清理（删除临时文件/数据库记录）

2. **文件名策略**：
   ```typescript
   // api/src/services/files.ts:128-134
   const fileExtension = path.extname(payload.filename_download!) || 
                          (payload.type && '.' + extension(payload.type)) || '';
   const filenameDisk = primaryKey + (fileExtension || '');
   ```
   - 使用 UUID (primaryKey) 作为文件名主体
   - 确保文件名唯一性，避免冲突

3. **元数据提取**：
   ```typescript
   // api/src/services/files.ts:222
   const metadata = await extractMetadata(payload.storage, payload);
   ```
   - 图片文件自动提取宽高、EXIF、标题、描述、标签等信息

#### 元数据提取

`extractMetadata` 函数位于 `api/src/services/files/lib/extract-metadata.ts`：

**支持的格式**：
- `image/jpeg`
- `image/png`
- `image/webp`
- `image/gif`
- `image/tiff`
- `image/avif`

**提取的字段**：
| 字段 | 来源 |
|------|------|
| `width`, `height` | 图片实际尺寸 |
| `metadata` | EXIF 数据 |
| `description` | 图片描述（如果存在） |
| `title` | 图片标题（如果存在） |
| `tags` | 图片标签（如果存在） |

---

## 3. 存储驱动系统

### 3.1 抽象接口设计

存储驱动系统采用**接口抽象 + 多实现**的设计模式。

#### Driver 接口

位于 `packages/storage/src/index.ts:35-46`：

```typescript
export declare class Driver {
    constructor(config: Record<string, unknown>);
    
    // 核心读写操作
    read(filepath: string, options?: ReadOptions): Promise<Readable>;
    write(filepath: string, content: Readable, type?: string): Promise<void>;
    delete(filepath: string): Promise<void>;
    
    // 文件信息操作
    stat(filepath: string): Promise<Stat>;      // 获取文件大小和修改时间
    exists(filepath: string): Promise<boolean>;  // 检查文件是否存在
    
    // 文件操作
    move(src: string, dest: string): Promise<void>;
    copy(src: string, dest: string): Promise<void>;
    
    // 列表操作
    list(prefix?: string): AsyncIterable<string>;
}
```

#### TusDriver 接口（支持分块上传）

位于 `packages/storage/src/index.ts:48-55`：

```typescript
export interface TusDriver extends Driver {
    get tusExtensions(): string[];  // 支持的 TUS 扩展
    
    // 分块上传生命周期
    createChunkedUpload(filepath: string, context: ChunkedUploadContext): Promise<ChunkedUploadContext>;
    writeChunk(filepath: string, content: Readable, offset: number, context: ChunkedUploadContext): Promise<number>;
    finishChunkedUpload(filepath: string, context: ChunkedUploadContext): Promise<void>;
    deleteChunkedUpload(filepath: string, context: ChunkedUploadContext): Promise<void>;
}
```

#### StorageManager

位于 `packages/storage/src/index.ts:4-33`：

```typescript
export class StorageManager {
    private drivers = new Map<string, typeof Driver>();      // 已注册的驱动类
    private locations = new Map<string, Driver>();            // 已实例化的存储位置
    
    // 注册驱动类型
    registerDriver(name: string, driver: typeof Driver): void;
    
    // 注册存储位置（使用指定驱动和配置）
    registerLocation(name: string, config: DriverConfig): void;
    
    // 获取存储位置实例
    location(name: string): Driver;
}
```

### 3.2 驱动注册机制

#### 驱动别名映射

位于 `api/src/storage/get-storage-driver.ts:3-10`：

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

#### 驱动动态加载

位于 `api/src/storage/get-storage-driver.ts:12-20`：

```typescript
export const getStorageDriver = async (driverName: string): Promise<typeof Driver> => {
    if (driverName in _aliasMap) {
        driverName = _aliasMap[driverName]!;
    } else {
        throw new Error(`Driver "${driverName}" doesn't exist.`);
    }
    return (await import(driverName)).default;  // 动态 import
};
```

#### 自动注册使用的驱动

位于 `api/src/storage/register-drivers.ts:5-21`：

```typescript
export const registerDrivers = async (storage: StorageManager) => {
    const env = useEnv();
    const usedDrivers: string[] = [];

    // 扫描环境变量中使用的驱动
    for (const [key, value] of Object.entries(env)) {
        if ((key.startsWith('STORAGE_') && key.endsWith('_DRIVER')) === false) continue;
        if (value && usedDrivers.includes(value as string) === false) {
            usedDrivers.push(value as string);
        }
    }

    // 动态加载并注册
    for (const driverName of usedDrivers) {
        const storageDriver = await getStorageDriver(driverName);
        if (storageDriver) {
            storage.registerDriver(driverName, storageDriver);
        }
    }
};
```

### 3.3 存储位置配置

#### 位置注册

位于 `api/src/storage/register-locations.ts:7-22`：

```typescript
export const registerLocations = async (storage: StorageManager) => {
    const locations = toArray(env['STORAGE_LOCATIONS'] as string);

    locations.forEach((location: string) => {
        location = location.trim();
        // 从环境变量 STORAGE_{LOCATION}_* 获取配置
        const driverConfig = getConfigFromEnv(`STORAGE_${location.toUpperCase()}_`);
        const { driver, ...options } = driverConfig;
        storage.registerLocation(location, { 
            driver, 
            options: { ...options, tus }  // 注入 TUS 配置
        });
    });
};
```

#### 环境变量配置示例

```bash
# 定义可用的存储位置（逗号分隔）
STORAGE_LOCATIONS="local,s3"

# Local 存储配置
STORAGE_LOCAL_DRIVER="local"
STORAGE_LOCAL_ROOT="./uploads"

# S3 存储配置
STORAGE_S3_DRIVER="s3"
STORAGE_S3_KEY="your-access-key"
STORAGE_S3_SECRET="your-secret-key"
STORAGE_S3_BUCKET="your-bucket"
STORAGE_S3_REGION="us-east-1"
STORAGE_S3_ENDPOINT="https://s3.amazonaws.com"
```

#### 默认存储位置

位于 `api/src/services/files.ts:104`：

```typescript
const payload = {
    storage: toArray(env['STORAGE_LOCATIONS'] as string)[0]!,  // 第一个配置的位置作为默认
    ...(existingFile ?? {}),
    ...clone(data),
};
```

### 3.4 具体驱动实现

#### DriverLocal（本地文件系统）

位于 `packages/storage-driver-local/src/index.ts`：

**配置选项**：
```typescript
export type DriverLocalConfig = {
    root: string;  // 存储根目录路径
};
```

**核心实现特点**：
1. **路径安全**：使用 `path.resolve` 和 `path.join` 确保路径不跳出 root 目录
2. **目录自动创建**：`ensureDir` 方法在写入时自动创建父目录
3. **完整 TUS 支持**：实现了 `TusDriver` 接口，支持分块上传

#### DriverS3（AWS S3 兼容）

位于 `packages/storage-driver-s3/src/index.ts`：

**配置选项**：
```typescript
export type DriverS3Config = {
    root?: string;
    key?: string;
    secret?: string;
    bucket: string;
    acl?: ObjectCannedACL;
    serverSideEncryption?: ServerSideEncryption;
    serverSideEncryptionKmsKeyId?: string;
    endpoint?: string;
    region?: string;
    forcePathStyle?: boolean;
    tus?: { chunkSize?: number };
    connectionTimeout?: number;
    socketTimeout?: number;
    maxSockets?: number;
    keepAlive?: boolean;
};
```

**核心实现特点**：
1. **多部分上传**：使用 S3 的 Multipart Upload API 实现大文件分块上传
2. **信号量控制**：`partUploadSemaphore` 限制并发上传数量（默认 60）
3. **HTTP 客户端优化**：
   - 增加 `maxSockets` 到 500（默认 50）
   - 支持 `keepAlive` 连接复用
   - 可配置连接超时和 socket 超时
4. **兼容多种 S3 实现**：通过 `endpoint` 和 `forcePathStyle` 支持 MinIO、DigitalOcean Spaces 等

#### 其他可用驱动

| 驱动 | 包名 | 适用场景 |
|------|------|----------|
| **GCS** | `@directus/storage-driver-gcs` | Google Cloud Storage |
| **Azure** | `@directus/storage-driver-azure` | Azure Blob Storage |
| **Cloudinary** | `@directus/storage-driver-cloudinary` | Cloudinary 图片 CDN |
| **Supabase** | `@directus/storage-driver-supabase` | Supabase Storage |

---

## 4. 资产变换系统

### 4.1 变换配置方式

#### 系统预设变换

位于 `api/src/constants.ts:10-41`：

```typescript
export const SYSTEM_ASSET_ALLOW_LIST: TransformationParams[] = [
    {
        key: 'system-small-cover',
        format: 'auto',
        transforms: [['resize', { width: 64, height: 64, fit: 'cover' }]],
    },
    {
        key: 'system-medium-cover',
        format: 'auto',
        transforms: [['resize', { width: 300, height: 300, fit: 'cover' }]],
    },
    {
        key: 'system-large-cover',
        format: 'auto',
        transforms: [['resize', { width: 800, height: 800, fit: 'cover' }]],
    },
    // ... 更多预设
];
```

#### 自定义预设配置

通过数据库 `directus_settings` 表的 `storage_asset_presets` 字段配置。

#### 变换参数

位于 `api/src/constants.ts:43-54`：

```typescript
export const ASSET_TRANSFORM_QUERY_KEYS = [
    'key',              // 使用预设的 key
    'transforms',       // 自定义变换数组（JSON）
    'width',            // 目标宽度
    'height',           // 目标高度
    'format',           // 输出格式 (jpg/png/webp/avif/auto)
    'fit',              // 适应模式 (cover/contain/inside/outside/fill)
    'quality',          // 质量 (1-100)
    'withoutEnlargement', // 禁止放大
    'focal_point_x',    // 焦点 X 坐标（用于裁剪）
    'focal_point_y',    // 焦点 Y 坐标
] as const;
```

### 4.2 变换处理流程

#### AssetsService.getAsset 方法

位于 `api/src/services/assets.ts:193-410`：

**完整执行流程**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    getAsset 执行流程                              │
├─────────────────────────────────────────────────────────────────┤
│  1. 检查 UUID 有效性                                              │
│         ↓                                                         │
│  2. 权限校验（系统公共文件跳过）                                   │
│         ↓                                                         │
│  3. 读取文件记录（sudo 权限）                                      │
│         ↓                                                         │
│  4. 检查文件是否存在于存储                                         │
│         ↓                                                         │
│  5. 解析变换参数                                                   │
│         ↓                                                         │
│  6. 检查是否支持变换格式                                           │
│         ↓                                                         │
│  7. 生成变换后的文件名：{id}__{hash}.{ext}                       │
│         ↓                                                         │
│  8. 检查缓存是否存在                                               │
│         │                                                         │
│         ├── 存在 → 直接返回缓存文件                               │
│         │                                                         │
│         └── 不存在 → 执行变换                                     │
│                    ↓                                              │
│              8.1 检查图片尺寸限制                                  │
│                    ↓                                              │
│              8.2 检查并发限制                                      │
│                    ↓                                              │
│              8.3 创建 Sharp 变换器                                │
│                    ↓                                              │
│              8.4 应用变换（resize/rotate/format 等）             │
│                    ↓                                              │
│              8.5 写入变换后的文件到存储                            │
│         ↓                                                         │
│  9. 返回文件流                                                     │
└─────────────────────────────────────────────────────────────────┘
```

#### 变换处理核心代码

```typescript
// api/src/services/assets.ts:351-392
const transformer = getSharpInstance();

// 自动旋转（根据 EXIF）
if (transforms.find((transform) => transform[0] === 'rotate') === undefined) {
    transformer.rotate();
}

// 应用所有变换
for (const [method, ...args] of transforms) {
    (transformer[method] as any).apply(transformer, args);
}

// 管道处理：读取 → 变换 → 写入
const readStream = await storage.location(file.storage).read(file.filename_disk);
await storage.location(file.storage).write(
    assetFilename, 
    readStream.pipe(transformer), 
    type
);
```

### 4.3 预设与动态变换

#### 变换策略配置

通过 `storage_asset_transform` 设置（位于 `directus_settings`）：

| 策略值 | 说明 |
|--------|------|
| `all` | 允许所有变换（预设 + 动态参数） |
| `presets` | 只允许使用预设的变换 |
| `none` / 其他 | 只允许系统预设 |

#### 控制器中的策略校验

位于 `api/src/controllers/assets.ts:237-259`：

```typescript
if (assetSettings.storage_asset_transform === 'all') {
    // 允许所有变换，但 key 必须在允许列表中
    if (transformation['key'] && allKeys.includes(transformation['key'] as string) === false) {
        throw new InvalidQueryError({ reason: `Key "${transformation['key']}" isn't configured` });
    }
} else if (assetSettings.storage_asset_transform === 'presets') {
    // 只允许预设，且不能有额外参数
    if (allKeys.includes(transformation['key'] as string) && Object.keys(transformation).length === 1) {
        return next();
    }
    throw new InvalidQueryError({ reason: `Only configured presets can be used in asset generation` });
} else {
    // 只允许系统预设
    if (transformation['key'] && systemKeys.includes(transformation['key'] as string) && 
        Object.keys(transformation).length === 1) {
        return next();
    }
    throw new InvalidQueryError({ reason: `Dynamic asset generation has been disabled` });
}
```

#### 支持的变换方法

位于 `api/src/utils/transformations.ts`：

| 方法 | 说明 |
|------|------|
| `resize` | 调整尺寸（支持 fit、width、height、withoutEnlargement） |
| `rotate` | 旋转 |
| `extract` | 裁剪区域 |
| `toFormat` | 转换格式 |

#### 焦点裁剪

当同时指定 `width`、`height`、`focal_point_x`、`focal_point_y` 时，会使用焦点裁剪算法：

```typescript
// api/src/utils/transformations.ts:51-90
if (
    (transformationParams.fit === undefined || transformationParams.fit === 'cover') &&
    toWidth && toHeight &&
    toFocalPointX !== null && toFocalPointY !== null
) {
    // 1. 先按比例缩放到中间尺寸
    // 2. 以焦点为中心裁剪到目标尺寸
    const transformArgs = getResizeArguments(
        { w: file.width, h: file.height },
        { w: toWidth, h: toHeight },
        { x: toFocalPointX, y: toFocalPointY },
    );
    
    transforms.push(
        ['resize', { width: transformArgs.width, height: transformArgs.height, ... }],
        ['extract', transformArgs.region]  // 以焦点为中心裁剪
    );
}
```

#### 缓存命名策略

变换后的文件使用哈希值命名，确保相同变换参数的文件可以复用：

```typescript
// api/src/services/assets.ts:413-416
const getAssetSuffix = (transforms: Transformation[]) => {
    if (Object.keys(transforms).length === 0) return '';
    return `__${hash(transforms)}`;  // 使用 object-hash 生成唯一标识
};

// 最终文件名：{originalId}__{hash}.{ext}
const assetFilename =
    path.basename(file.filename_disk, path.extname(file.filename_disk)) +
    getAssetSuffix(transforms) +
    (maybeNewFormat ? `.${maybeNewFormat}` : path.extname(file.filename_disk));
```

---

## 5. 访问权限校验

### 5.1 权限校验机制

#### 校验入口

**上传时校验**（通过 ItemsService 继承）：

`FilesService` 继承自 `ItemsService`，所有 CRUD 操作都会自动触发权限校验。

**访问时校验**（AssetsService）：

位于 `api/src/services/assets.ts:231-251`：

```typescript
if (!systemPublicKeys.includes(id) && this.accountability && this.accountability.admin !== true) {
    const { allowedRootFields, accessAllowed } = await validateItemAccess(
        {
            accountability: this.accountability,
            action: 'read',
            collection: 'directus_files',
            primaryKeys: [id],
            returnAllowedRootFields: true,
        },
        { knex: this.knex, schema: this.schema },
    );

    if (!accessAllowed) {
        throw new ForbiddenError({
            reason: `You don't have permission to perform "read" for collection "directus_files"`,
        });
    }

    allowedFields = allowedRootFields;
}
```

#### validateItemAccess 实现

位于 `api/src/permissions/modules/validate-access/lib/validate-item-access.ts`：

**核心逻辑**：
1. 构建 AST（抽象语法树）查询
2. 注入权限过滤条件（通过 `processAst`）
3. 执行查询，检查返回结果数量
4. 如果指定 `returnAllowedRootFields`，还会返回允许访问的字段列表

```typescript
// 构建查询
const ast: AST = {
    type: 'root',
    name: options.collection,
    query: { limit: options.primaryKeys!.length },
    children: [],  // 字段列表
    cases: [],
};

// 注入权限
await processAst({ ast, ...options }, context);

// 添加主键过滤
ast.query.filter = {
    [primaryKeyField]: { _in: options.primaryKeys! },
};

// 执行查询
const items = await fetchPermittedAstRootFields(ast, { ... });

// 检查访问权限：必须能查到所有请求的主键
const expectedCount = options.primaryKeys!.length;
const hasAccess = items && items.length === expectedCount;
```

#### 条件请求的预校验

位于 `api/src/controllers/assets.ts:336-387`：

当启用 `ASSETS_CACHE_REVALIDATE` 时，在处理 `If-None-Match` 和 `If-Modified-Since` 头时会先校验权限：

```typescript
if (revalidate) {
    const ifNoneMatch = req.headers['if-none-match'];
    const ifModifiedSince = req.headers['if-modified-since'];

    if (ifNoneMatch || ifModifiedSince) {
        // 先校验权限
        if (req.accountability) {
            await validateAccess(
                {
                    accountability: req.accountability,
                    action: 'read',
                    collection: 'directus_files',
                    primaryKeys: [id],
                },
                { knex: getDatabase(), schema: req.schema },
            );
        }

        // 再检查缓存条件
        const fileRecord = await filesService.readOne(id, { fields: ['modified_on'] });
        // ... 304 响应处理
    }
}
```

### 5.2 系统公共文件例外

#### 系统公共文件列表

位于 `api/src/services/assets.ts:215-220`：

```typescript
const publicSettings = await this.knex
    .select('project_logo', 'public_background', 'public_foreground', 'public_favicon')
    .from('directus_settings')
    .first();

const systemPublicKeys: string[] = Object.values(publicSettings || {});
```

#### 例外逻辑

```typescript
// api/src/services/assets.ts:231
if (!systemPublicKeys.includes(id) && this.accountability && this.accountability.admin !== true) {
    // 执行权限校验
}
```

**含义**：
- 如果文件是项目 logo、背景、前景或 favicon
- **任何人都可以访问**，无需登录或权限
- 这确保了登录页面、公共页面可以正常显示这些图片

#### 管理员例外

```typescript
this.accountability.admin !== true
```

管理员用户跳过所有文件权限校验。

---

## 6. 环境变量配置

### 文件上传配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `FILES_MAX_UPLOAD_SIZE` | - | 单个文件最大大小（支持 `10MB`, `1GB` 等格式） |
| `FILES_MAX_UPLOAD_CONCURRENCY` | - | 最大并发上传数 |
| `FILES_MIME_TYPE_ALLOW_LIST` | - | 允许的 MIME 类型（支持 glob 模式） |
| `FILES_DELETE_ORIGINAL_ON_MOVE` | - | 重命名文件时是否删除原文件 |

### 存储配置

| 变量名 | 说明 |
|--------|------|
| `STORAGE_LOCATIONS` | 可用存储位置列表（逗号分隔） |
| `STORAGE_{LOCATION}_DRIVER` | 该位置使用的驱动类型 |
| `STORAGE_{LOCATION}_*` | 驱动特定的配置选项 |

### 资产变换配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `ASSETS_TRANSFORM_MAX_OPERATIONS` | - | 单次请求允许的最大变换操作数 |
| `ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION` | - | 允许变换的最大图片尺寸 |
| `ASSETS_TRANSFORM_MAX_CONCURRENT` | - | 最大并发变换数 |
| `ASSETS_TRANSFORM_TIMEOUT` | - | 变换操作超时时间 |
| `ASSETS_CACHE_TTL` | - | 缓存有效期 |
| `ASSETS_CACHE_REVALIDATE` | - | 是否启用缓存验证 |
| `ASSETS_CONTENT_SECURITY_POLICY` | - | 资产响应的 CSP 策略 |

### 分块上传（TUS）配置

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `TUS_ENABLED` | `false` | 是否启用 TUS 分块上传 |
| `TUS_CHUNK_SIZE` | - | 分块大小 |
| `TUS_UPLOAD_EXPIRATION` | `600000` (10min) | 未完成上传的过期时间 |
| `TUS_CLEANUP_SCHEDULE` | - | 清理过期上传的 cron 表达式 |

---

## 7. 关键代码参考

### 7.1 核心模块路径

| 模块 | 路径 |
|------|------|
| **文件上传控制器** | `api/src/controllers/files.ts` |
| **文件服务** | `api/src/services/files.ts` |
| **资产访问控制器** | `api/src/controllers/assets.ts` |
| **资产服务** | `api/src/services/assets.ts` |
| **存储抽象** | `packages/storage/src/index.ts` |
| **存储管理器** | `api/src/storage/index.ts` |
| **驱动注册** | `api/src/storage/register-drivers.ts` |
| **位置注册** | `api/src/storage/register-locations.ts` |
| **本地驱动** | `packages/storage-driver-local/src/index.ts` |
| **S3 驱动** | `packages/storage-driver-s3/src/index.ts` |
| **变换工具** | `api/src/utils/transformations.ts` |
| **权限校验** | `api/src/permissions/modules/validate-access/` |
| **常量定义** | `api/src/constants.ts` |

### 7.2 关键方法速查

| 方法 | 文件位置 | 功能 |
|------|----------|------|
| `multipartHandler` | `files.ts:26` | 处理 multipart 上传 |
| `FilesService.uploadOne` | `files.ts:81` | 核心上传逻辑 |
| `FilesService.importOne` | `files.ts:261` | 从 URL 导入文件 |
| `AssetsService.getAsset` | `assets.ts:193` | 获取资产（含变换） |
| `StorageManager.location` | `storage/index.ts:24` | 获取存储位置 |
| `registerDrivers` | `storage/register-drivers.ts:5` | 注册驱动 |
| `registerLocations` | `storage/register-locations.ts:7` | 注册存储位置 |
| `validateAccess` | `validate-access.ts:22` | 权限校验入口 |
| `validateItemAccess` | `validate-item-access.ts:44` | 项目级别权限校验 |
| `resolvePreset` | `transformations.ts:5` | 解析变换参数 |
| `extractMetadata` | `files/lib/extract-metadata.ts:6` | 提取图片元数据 |

---

## 总结

Directus 的文件上传系统设计具有以下特点：

### 架构优势

1. **分层清晰**：Controller → Service → Storage Driver 三层架构，职责分明
2. **驱动抽象**：统一的 `Driver` 接口，支持多种存储后端无缝切换
3. **变换灵活**：支持预设变换和动态变换，通过缓存提升性能
4. **权限严格**：基于 RBAC 的细粒度访问控制，同时支持系统公共文件例外

### 关键设计决策

1. **UUID 文件名**：避免文件名冲突，简化文件管理
2. **替换上传原子性**：先写临时文件，成功后再替换，保证数据一致性
3. **变换缓存**：通过哈希命名实现变换结果复用，减少重复计算
4. **流式处理**：全程使用 Node.js Stream，避免大文件内存溢出

### 可扩展点

1. **自定义存储驱动**：实现 `Driver` 或 `TusDriver` 接口即可添加新的存储后端
2. **自定义变换**：扩展 `TransformationMethods` 可以添加新的图片变换操作
3. **权限策略**：通过 Directus 的权限系统可以灵活配置文件的读写权限

---

*报告生成完毕*
