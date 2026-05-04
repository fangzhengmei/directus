# Directus 文件上传、图片变换与资产权限分析

## 1. 文件上传处理流程

### 1.1 入口点与请求解析

文件上传的入口点位于 `/api/src/controllers/files.ts`，使用 `Busboy` 库解析 `multipart/form-data` 请求。

**关键代码位置：** `files.ts:26-133`

```typescript
const multipartHandler: RequestHandler = (req, res, next) => {
    if (req.is('multipart/form-data') === false) return next();
    
    const busboy = Busboy({
        headers,
        defParamCharset: 'utf8',
        limits: {
            fileSize: bytes.parse(env['FILES_MAX_UPLOAD_SIZE'] as string) ?? undefined,
        },
    });
    // ... 处理文件和字段
};
```

### 1.2 上传流程步骤

1. **请求解析**：Busboy 解析请求体，区分字段和文件流
2. **MIME 类型验证**：检查文件类型是否在允许列表中
   ```typescript
   const allowedPatterns = toArray(env['FILES_MIME_TYPE_ALLOW_LIST'] as string | string[]);
   const mimeTypeAllowed = allowedPatterns.some((pattern) => minimatch(mimeType, pattern));
   ```
   `files.ts:77-82`

3. **服务层调用**：调用 `FilesService.uploadOne()` 处理实际上传
4. **存储写入**：将文件流写入配置的存储适配器
5. **元数据提取**：提取文件元数据（尺寸、格式等）
6. **数据库记录**：更新 `directus_files` 表记录

### 1.3 FilesService.uploadOne 方法详解

**关键代码位置：** `files.ts:81-252`

主要流程：

1. **存储初始化**：获取配置的存储位置
2. **现有文件检查**：如果是替换上传，检查现有文件记录
3. **数据库记录创建**：先创建数据库记录，获取主键
4. **文件名生成**：使用主键 + 文件扩展名作为磁盘文件名
   ```typescript
   const filenameDisk = primaryKey + (fileExtension || '');
   ```
   `files.ts:131`

5. **文件写入**：
   - 新文件：直接写入最终位置
   - 替换文件：先写入临时位置，成功后替换
6. **截断检查**：检查文件流是否被意外截断
7. **元数据提取**：调用 `extractMetadata()` 提取文件元数据
8. **记录更新**：更新数据库记录，包含文件大小、元数据等
9. **事件触发**：触发 `files.upload` 事件

---

## 2. 图片变换（裁剪和缩放）实现

### 2.1 核心技术栈

- **图像处理库**：Sharp 高性能图像处理库
- **变换定义**：`@directus/types` 中的 `TransformationMethods` 枚举
- **预设管理**：通过 `storage_asset_presets` 配置预设变换

### 2.2 入口点与请求处理

**关键代码位置：** `assets.ts:279-468`

图片变换在资产访问时动态处理，入口是 `GET /assets/:id/:filename?` 端点。

**请求参数验证**：`assets.ts:158-260`

```typescript
// 验证变换参数
const transformation = pick(req.query, ASSET_TRANSFORM_QUERY_KEYS);

// 检查 transforms JSON 数组
if ('transforms' in transformation) {
    let transforms = parseJSON(transformation['transforms'] as string);
    // 验证变换数量和类型
    if (transforms.length > Number(env['ASSETS_TRANSFORM_MAX_OPERATIONS'])) {
        throw new InvalidQueryError({ ... });
    }
}
```

### 2.3 变换模式配置

系统支持三种变换模式（`storage_asset_transform`）：

1. **all**：允许所有动态变换和预设
2. **presets**：仅允许配置的预设变换
3. **none**：禁用动态变换（仅系统预设）

### 2.4 核心变换实现

**关键代码位置：** `assets.ts:303-400`

```typescript
if (type && transforms.length > 0 && SUPPORTED_IMAGE_TRANSFORM_FORMATS.includes(type)) {
    // 生成变换后的文件名（带哈希后缀）
    const assetFilename = 
        path.basename(file.filename_disk, path.extname(file.filename_disk)) +
        getAssetSuffix(transforms) +
        (maybeNewFormat ? `.${maybeNewFormat}` : path.extname(file.filename_disk));
    
    // 检查缓存是否存在
    const exists = await storage.location(file.storage).exists(assetFilename);
    
    if (exists) {
        // 直接返回缓存文件
        const assetStream = () => storage.location(file.storage).read(assetFilename, { range });
        return { stream: assetStream, file, stat };
    }
    
    // 图像尺寸检查
    if (width > (env['ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION'] as number) ||
        height > (env['ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION'] as number)) {
        throw new IllegalAssetTransformationError({ invalidTransformations: ['width', 'height'] });
    }
    
    // 创建 Sharp 实例并应用变换
    const transformer = getSharpInstance();
    transformer.rotate(); // 默认自动旋转（基于 EXIF）
    
    for (const [method, ...args] of transforms) {
        (transformer[method] as any).apply(transformer, args);
    }
    
    // 执行变换并保存
    await storage.location(file.storage).write(
        assetFilename, 
        readStream.pipe(transformer), 
        type
    );
}
```

### 2.5 Sharp 实例配置

**关键代码位置：** `files/lib/get-sharp-instance.ts:1-12`

```typescript
export function getSharpInstance(): Sharp {
    const env = useEnv();
    
    return sharp({
        limitInputPixels: Math.trunc(Math.pow(env['ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION'] as number, 2)),
        sequentialRead: true,
        failOn: env['ASSETS_INVALID_IMAGE_SENSITIVITY_LEVEL'] as FailOnOptions,
    });
}
```

### 2.6 支持的变换方法

通过 `TransformationMethods` 枚举定义，包括：
- `resize`：调整大小
- `crop`：裁剪
- `rotate`：旋转
- `flip`：翻转
- `flop`：水平翻转
- `sharpen`：锐化
- `blur`：模糊
- `greyscale`：灰度
- `normalize`：归一化
- `quality`：质量设置
- `format`：格式转换

### 2.7 缓存机制

- **文件名格式**：`{primaryKey}__{hash}{ext}`
- **哈希计算**：`object-hash` 对变换参数数组进行哈希
- **缓存检查**：每次请求先检查变换后的文件是否存在
- **自动过期**：通过文件修改时间和版本号管理缓存

```typescript
const getAssetSuffix = (transforms: Transformation[]) => {
    if (Object.keys(transforms).length === 0) return '';
    return `__${hash(transforms)}`;
};
```
`assets.ts:413-416`

### 2.8 安全限制

1. **最大并发变换**：`ASSETS_TRANSFORM_MAX_CONCURRENT`
2. **变换超时**：`ASSETS_TRANSFORM_TIMEOUT`（1-3600 秒）
3. **最大变换次数**：`ASSETS_TRANSFORM_MAX_OPERATIONS`
4. **最大图像尺寸**：`ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION`（防止内存溢出）

---

## 3. 资产访问权限控制

### 3.1 权限检查入口

**关键代码位置：** `assets.ts:231-251`

```typescript
let allowedFields: string[] = ['*'];

if (!systemPublicKeys.includes(id) && this.accountability && this.accountability.admin !== true) {
    // 使用 validateItemAccess 检查权限并获取允许字段
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
            reason: `You don't have permission to perform "read" for collection "directus_files" or it does not exist.`,
        });
    }
    
    allowedFields = allowedRootFields;
}
```

### 3.2 权限检查层次

#### 第一层：系统公共文件豁免

```typescript
const publicSettings = await this.knex
    .select('project_logo', 'public_background', 'public_foreground', 'public_favicon')
    .from('directus_settings')
    .first();

const systemPublicKeys: string[] = Object.values(publicSettings || {});

if (!systemPublicKeys.includes(id) && ...) {
    // 进行权限检查
}
```
`assets.ts:215-231`

系统公共文件（项目 Logo、背景等）无需权限检查。

#### 第二层：管理员豁免

```typescript
if (this.accountability.admin !== true) {
    // 非管理员需要权限检查
}
```

管理员账号跳过权限检查。

#### 第三层：基于权限系统的检查

使用 `validateItemAccess` 函数，检查：
- 用户对 `directus_files` 集合的 `read` 权限
- 具体文件 ID 的访问权限

### 3.3 字段级权限控制

**关键代码位置：** `assets.ts:57-74`

```typescript
private sanitizeFields(file: File, allowedFields: string[]): Partial<File> {
    if (allowedFields.includes('*')) {
        return file;
    }
    
    const bypassFields: (keyof File)[] = ['type', 'filesize'];
    const fieldsToKeep = new Set<string>([...allowedFields, ...bypassFields]);
    
    const filteredFile: Partial<File> = {};
    
    for (const field of fieldsToKeep) {
        if (field in file) {
            (filteredFile as Record<string, unknown>)[field] = file[field as keyof File];
        }
    }
    
    return filteredFile;
}
```

返回给用户的文件信息会根据权限系统配置的 `allowedRootFields` 进行过滤。

### 3.4 额外的安全检查

#### 1. UUID 格式验证

```typescript
if (!isValidUuid(id)) throw new ForbiddenError();
```
`assets.ts:227`

防止 SQL 注入和无效 ID 查询。

#### 2. 文件存在性验证

```typescript
const exists = await storage.location(file.storage).exists(file.filename_disk);

if (!exists) throw new ForbiddenError();
```
`assets.ts:255-257`

即使数据库记录存在，物理文件不存在也返回 403。

#### 3. 条件请求前置检查

**关键代码位置：** `assets.ts:336-387`

对于带 `If-None-Match` 或 `If-Modified-Since` 头的请求：
- 先执行权限检查
- 再检查文件修改时间
- 如果缓存有效，返回 304 而非文件内容

```typescript
if (revalidate) {
    const ifNoneMatch = req.headers['if-none-match'];
    const ifModifiedSince = req.headers['if-modified-since'];
    
    if (ifNoneMatch || ifModifiedSince) {
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
        
        // 检查 ETag 和 Last-Modified
        const etag = `"${Math.floor(modifiedOnTime / 1000)}"`;
        
        if (ifNoneMatch === etag) {
            res.setHeader('Cache-Control', 'max-age=0, must-revalidate');
            res.setHeader('ETag', etag);
            res.status(304);
            return res.end();
        }
    }
}
```

### 3.5 权限控制流程图

```
请求资产
    ↓
检查是否为系统公共文件？
    ├── 是 → 跳过权限检查
    └── 否 → 检查是否为管理员？
                  ├── 是 → 跳过权限检查
                  └── 否 → 调用 validateItemAccess
                                ↓
                          检查 read 权限？
                                ├── 否 → 抛出 ForbiddenError
                                └── 是 → 获取 allowedRootFields
                                              ↓
                                        检查文件物理存在？
                                              ├── 否 → 抛出 ForbiddenError
                                              └── 是 → 过滤返回字段
                                                          ↓
                                                    返回资产内容
```

---

## 4. 关键配置项

### 4.1 文件上传配置

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `FILES_MAX_UPLOAD_SIZE` | 最大上传文件大小 | 取决于环境 |
| `FILES_MIME_TYPE_ALLOW_LIST` | 允许的 MIME 类型列表 | - |
| `FILES_DELETE_ORIGINAL_ON_MOVE` | 移动文件时是否删除原文件 | - |
| `STORAGE_LOCATIONS` | 存储位置列表 | `local` |

### 4.2 图片变换配置

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION` | 最大图像尺寸 | - |
| `ASSETS_TRANSFORM_MAX_OPERATIONS` | 单次请求最大变换次数 | - |
| `ASSETS_TRANSFORM_MAX_CONCURRENT` | 最大并发变换数 | - |
| `ASSETS_TRANSFORM_TIMEOUT` | 变换超时时间（秒） | - |
| `ASSETS_INVALID_IMAGE_SENSITIVITY_LEVEL` | 无效图像敏感度 | - |
| `STORAGE_ASSET_TRANSFORM` | 变换模式 | `all` |
| `STORAGE_ASSET_PRESETS` | 变换预设 | `[]` |

### 4.3 缓存和安全配置

| 配置项 | 说明 | 默认值 |
|--------|------|--------|
| `ASSETS_CACHE_TTL` | 资产缓存 TTL | - |
| `ASSETS_CACHE_REVALIDATE` | 是否启用重新验证 | `false` |
| `ASSETS_CONTENT_SECURITY_POLICY` | 内容安全策略 | - |

---

## 5. 关键文件索引

| 功能 | 文件路径 | 说明 |
|------|----------|------|
| 文件上传控制器 | `api/src/controllers/files.ts` | HTTP 端点和请求解析 |
| 文件服务 | `api/src/services/files.ts` | 上传、元数据提取核心逻辑 |
| 资产控制器 | `api/src/controllers/assets.ts` | 资产访问和变换端点 |
| 资产服务 | `api/src/services/assets.ts` | 权限检查和图片变换 |
| Sharp 实例 | `api/src/services/files/lib/get-sharp-instance.ts` | 图像处理库配置 |
| 元数据提取 | `api/src/services/files/lib/extract-metadata.ts` | 文件元数据提取 |
| 权限验证 | `api/src/permissions/modules/validate-access/` | 核心权限检查逻辑 |
| 变换工具 | `api/src/utils/transformations.ts` | 变换参数解析和预设管理 |

---

## 6. 总结

### 6.1 架构特点

1. **分层设计**：Controller → Service → Storage 的三层架构
2. **流式处理**：文件上传和变换都使用流，避免内存溢出
3. **缓存优化**：变换后的图像自动缓存，提升性能
4. **权限细粒度**：集合级 + 字段级 + 记录级的三层权限控制

### 6.2 安全机制

1. **MIME 类型白名单**：防止恶意文件上传
2. **文件大小限制**：防止拒绝服务攻击
3. **UUID 验证**：防止 SQL 注入
4. **并发控制**：防止资源耗尽
5. **尺寸限制**：防止内存溢出

### 6.3 扩展性

1. **存储适配器**：支持多种存储后端（S3、Azure、GCS 等）
2. **变换预设**：可配置预设变换，简化前端调用
3. **权限系统**：基于 Directus 权限系统，支持复杂的访问控制规则
