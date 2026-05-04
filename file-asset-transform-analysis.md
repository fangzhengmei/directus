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

### 2.4 裁剪与缩放的真实执行顺序

#### 2.4.1 变换参数解析与构造

**关键代码位置：** `utils/transformations.ts:5-94`

变换参数由 `resolvePreset` 函数构造，**构造顺序**如下：

1. **初始化 transforms 数组**：如果请求中已有 `transforms` 参数，先复制到数组
   ```typescript
   const transforms = transformationParams.transforms ? [...transformationParams.transforms] : [];
   ```
   `transformations.ts:6`

2. **添加格式/质量变换**：如果有 `format` 或 `quality` 参数，添加 `toFormat` 变换
   ```typescript
   if (transformationParams.format || transformationParams.quality) {
       transforms.push([
           'toFormat',
           getFormat(file, transformationParams.format, acceptFormat),
           { quality: transformationParams.quality ? Number(transformationParams.quality) : undefined },
       ]);
   }
   ```
   `transformations.ts:8-16`

3. **添加 resize 变换**：如果有 `width` 或 `height` 参数，添加 resize 相关变换
   - 这是**最复杂的部分**，会决定是否触发焦点裁剪

#### 2.4.2 焦点裁剪的触发条件

**关键代码位置：** `transformations.ts:51-57`

焦点裁剪（基于 focal_point 的智能裁剪）**必须同时满足以下所有条件**才会触发：

```typescript
if (
    (transformationParams.fit === undefined || transformationParams.fit === 'cover') &&
    toWidth &&
    toHeight &&
    toFocalPointX !== null &&
    toFocalPointY !== null
)
```

| 条件 | 说明 | 来源 |
|------|------|------|
| `fit` 为 `undefined` 或 `'cover'` | 适配模式必须是 cover（或默认） | 请求参数 `fit` |
| `toWidth` 和 `toHeight` 都有值 | **必须同时指定 width 和 height** | 请求参数 `width` + `height` |
| `toFocalPointX` 和 `toFocalPointY` 都不为 null | 焦点坐标存在 | 请求参数 `focal_point_x/y` **或** 文件元数据 `focal_point_x/y` |

**关键点**：
- 如果只指定了 `width` 或只指定了 `height`，**不会触发焦点裁剪**，只会执行普通 resize
- 焦点坐标优先级：请求参数 > 文件元数据
- 焦点坐标以像素为单位，相对于原始图像

#### 2.4.3 焦点裁剪的执行顺序

**关键代码位置：** `transformations.ts:64-77`

当焦点裁剪条件满足时，会向 transforms 数组**添加两个变换**：

```typescript
transforms.push(
    [
        'resize',
        {
            width: transformArgs.width,
            height: transformArgs.height,
            fit: transformationParams.fit,
            withoutEnlargement: transformationParams.withoutEnlargement
                ? Boolean(transformationParams.withoutEnlargement)
                : undefined,
        },
    ],
    ['extract', transformArgs.region],
);
```

**这意味着焦点裁剪的执行顺序是：**

1. **第一步：resize（缩放）**
   - 先将图像缩放到**中间尺寸**
   - 中间尺寸计算：保持宽高比，按较小边缩放
   - 例如：原图 1920x1080，目标 400x400，中间尺寸约为 711x400

2. **第二步：extract（裁剪）**
   - 从缩放后的图像中**提取以焦点为中心的区域**
   - 提取区域计算：
     - 将原始焦点坐标除以缩放因子，得到新坐标
     - 以新坐标为中心，裁剪出目标尺寸的区域
     - 使用 `clamp` 确保边界不越界

**算法原理**（来自 `transformations.ts:129-196`）：

```typescript
// 1. 计算中间尺寸（缩放因子）
function getIntermediateDimensions(original: Dimensions, target: Dimensions) {
    const hRatio = original.h / target.h;
    const wRatio = original.w / target.w;
    
    // 选择较小的比率作为缩放因子
    if (hRatio < wRatio) {
        factor = hRatio;
        height = target.h;
        width = original.w / factor;
    } else {
        factor = wRatio;
        width = target.w;
        height = original.h / factor;
    }
}

// 2. 计算裁剪区域
function getExtractionRegion(factor: number, focalPoint: FocalPoint, target: Dimensions, intermediate: Dimensions) {
    // 将焦点坐标转换到缩放后的坐标系
    const newXCenter = focalPoint.x / factor;
    const newYCenter = focalPoint.y / factor;
    
    // 以新坐标为中心裁剪目标尺寸
    return {
        left: clamp(Math.round(newXCenter - target.w / 2), 0, intermediate.w - target.w),
        top: clamp(Math.round(newYCenter - target.h / 2), 0, intermediate.h - target.h),
        width: target.w,
        height: target.h,
    };
}
```

#### 2.4.4 实际执行顺序（Sharp 管道）

**关键代码位置：** `assets.ts:357-369`

在 `AssetsService.getAsset()` 中，变换的**实际执行顺序**如下：

```typescript
// 1. 默认自动旋转（如果没有显式的 rotate 变换）
if (transforms.find((transform) => transform[0] === 'rotate') === undefined) {
    transformer.rotate();
}

// 2. 按顺序执行 transforms 数组中的所有变换
try {
    for (const [method, ...args] of transforms) {
        (transformer[method] as any).apply(transformer, args);
    }
} catch (error) {
    // ... 错误处理
}
```

**完整执行顺序总结**：

| 步骤 | 变换 | 条件 | 说明 |
|------|------|------|------|
| 1 | `rotate()` | 总是执行（除非有显式 rotate） | 基于 EXIF 方向自动旋转 |
| 2 | 自定义 `transforms` 数组 | 请求有 `transforms` 参数时 | 按数组顺序执行 |
| 3 | `toFormat` | 有 `format` 或 `quality` 参数时 | 格式转换和质量设置 |
| 4a | `resize`（普通） | 非焦点裁剪时 | 普通缩放 |
| 4b | `resize`（中间尺寸） | 焦点裁剪时 | 先缩放到中间尺寸 |
| 5 | `extract` | 焦点裁剪时 | 提取焦点区域 |

### 2.5 缓存命中与限流超时的执行顺序

**关键代码位置：** `assets.ts:303-400`

当需要执行图片变换时，**检查和执行顺序**如下：

```
请求变换
    ↓
[1] 生成变换后文件名（包含 hash）
    ↓
[2] 检查缓存是否存在？
    ├── 是 → 直接返回缓存文件（跳过所有后续检查）
    └── 否 → 继续
              ↓
        [3] 检查图像尺寸
              │
              ├── 尺寸超限 → 抛出 IllegalAssetTransformationError
              └── 正常 → 继续
                        ↓
                  [4] 检查并发限流
                        │
                        ├── 超限 → 抛出 ServiceUnavailableError
                        └── 正常 → 继续
                                  ↓
                            [5] 创建 Sharp 实例
                                  ↓
                            [6] 设置超时
                                  ↓
                            [7] 应用变换到管道
                                  ↓
                            [8] 执行变换（流式处理）
                                  ↓
                            [9] 保存结果到缓存
                                  ↓
                            [10] 返回结果
```

#### 详细步骤说明

**步骤 2：缓存检查**（`assets.ts:311-325`）
```typescript
const exists = await storage.location(file.storage).exists(assetFilename);

if (exists) {
    // 缓存命中，直接返回
    const assetStream = () => storage.location(file.storage).read(assetFilename, { range });
    return {
        stream: deferStream ? assetStream : await assetStream(),
        file: this.sanitizeFields(file, allowedFields),
        stat: await storage.location(file.storage).stat(assetFilename),
    };
}
```
- **缓存命中时**：直接返回，**不执行**后续的尺寸检查、限流检查和变换
- **缓存未命中时**：继续执行后续流程

**步骤 3：图像尺寸检查**（`assets.ts:332-340`）
```typescript
const { width, height } = file;

if (
    !width ||
    !height ||
    width > (env['ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION'] as number) ||
    height > (env['ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION'] as number)
) {
    logger.warn(`Image is too large to be transformed, or image size couldn't be determined.`);
    throw new IllegalAssetTransformationError({ invalidTransformations: ['width', 'height'] });
}
```
- 检查图像尺寸是否超过 `ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION`
- 防止处理超大图像导致内存溢出

**步骤 4：并发限流检查**（`assets.ts:342-349`）
```typescript
const { queue, process } = sharp.counters();

if (queue + process > (env['ASSETS_TRANSFORM_MAX_CONCURRENT'] as number)) {
    throw new ServiceUnavailableError({
        service: 'files',
        reason: 'Server too busy',
    });
}
```
- 使用 `sharp.counters()` 获取当前排队和正在处理的变换任务数
- 如果超过 `ASSETS_TRANSFORM_MAX_CONCURRENT`，返回 503 Service Unavailable

**步骤 5-6：Sharp 实例创建与超时设置**（`assets.ts:351-355`）
```typescript
const transformer = getSharpInstance();

transformer.timeout({
    seconds: clamp(Math.round(getMilliseconds(env['ASSETS_TRANSFORM_TIMEOUT'], 0) / 1000), 1, 3600),
});
```
- 超时时间范围：1-3600 秒（强制限制）
- 超时触发时抛出 `ServiceUnavailableError`（`assets.ts:387-388`）

**步骤 7-9：执行变换并缓存**
- 使用流式处理：`readStream.pipe(transformer)`
- 变换结果写入 `assetFilename`（带 hash 的缓存文件名）

### 2.6 缓存机制详解

**关键代码位置：** `assets.ts:413-416`

```typescript
const getAssetSuffix = (transforms: Transformation[]) => {
    if (Object.keys(transforms).length === 0) return '';
    return `__${hash(transforms)}`;
};
```

**缓存文件名格式**：
```
{primaryKey}__{transforms_hash}{.ext}
```

例如：
- 原始文件：`550e8400-e29b-41d4-a716-446655440000.jpg`
- 变换后（宽 400，高 300，cover）：`550e8400-e29b-41d4-a716-446655440000__a1b2c3d4.jpg`

**缓存特性**：
1. **唯一性**：不同的变换参数生成不同的 hash
2. **持久性**：缓存文件不会自动过期，需要手动清理或在文件更新时删除
3. **版本控制**：通过 `modified_on` 时间戳生成 version 参数，用于缓存失效

### 2.7 Sharp 实例配置

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

| 配置项 | 说明 |
|--------|------|
| `limitInputPixels` | 最大像素数限制（尺寸的平方） |
| `sequentialRead` | 顺序读取模式，优化内存使用 |
| `failOn` | 无效图像敏感度级别 |

### 2.8 支持的变换方法

通过 `TransformationMethods` 枚举定义，包括：

| 方法 | 说明 |
|------|------|
| `resize` | 调整大小（核心变换） |
| `extract` | 裁剪（焦点裁剪使用） |
| `rotate` | 旋转 |
| `flip` | 垂直翻转 |
| `flop` | 水平翻转 |
| `sharpen` | 锐化 |
| `blur` | 模糊 |
| `greyscale` | 灰度 |
| `normalize` | 归一化 |
| `toFormat` | 格式转换 |

### 2.9 安全限制汇总

| 检查点 | 配置项 | 错误类型 | 检查时机 |
|--------|--------|----------|----------|
| 变换数量限制 | `ASSETS_TRANSFORM_MAX_OPERATIONS` | `InvalidQueryError` | 参数解析时 |
| 图像尺寸限制 | `ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION` | `IllegalAssetTransformationError` | 缓存未命中后 |
| 并发变换限制 | `ASSETS_TRANSFORM_MAX_CONCURRENT` | `ServiceUnavailableError` | 尺寸检查后 |
| 变换超时限制 | `ASSETS_TRANSFORM_TIMEOUT` | `ServiceUnavailableError` | 执行中触发 |
| 变换模式限制 | `STORAGE_ASSET_TRANSFORM` | `InvalidQueryError` | 参数验证时 |

---

## 3. 资产访问权限控制

### 3.1 权限检查入口

**关键代码位置：** `assets.ts:227-251`

```typescript
// 1. UUID 格式验证（在权限检查之前）
if (!isValidUuid(id)) throw new ForbiddenError();

let allowedFields: string[] = ['*'];

// 2. 核心权限检查逻辑
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
            reason: `You don't have permission to perform "read" for collection "directus_files" or it does not exist.`,
        });
    }
    
    allowedFields = allowedRootFields;
}
```

### 3.2 资产可见性的五层权限边界

Directus 对资产访问有**严格的五层检查**，任何一层不通过都会返回 403 Forbidden。

```
请求资产
    ↓
[第0层] UUID 格式验证
    │
    ├── 格式无效 → ForbiddenError
    └── 有效 → 继续
              ↓
        [第1层] 系统公共文件豁免？
              │
              ├── 是（项目 Logo 等）→ 跳过所有权限检查，直接访问
              └── 否 → 继续
                        ↓
                  [第2层] 管理员豁免？
                        │
                        ├── 是（accountability.admin === true）→ 跳过权限检查
                        └── 否 → 继续
                                  ↓
                            [第3层] 权限系统检查（validateItemAccess）
                                  │
                                  ├── 无权限 → ForbiddenError
                                  └── 有权限 → 继续
                                            ↓
                                      [第4层] 文件物理存在性验证
                                            │
                                            ├── 不存在 → ForbiddenError
                                            └── 存在 → 继续
                                                      ↓
                                                [第5层] 字段级权限过滤
                                                      │
                                                      └── 过滤后返回文件信息
```

#### 第 0 层：UUID 格式验证

**关键代码位置：** `assets.ts:227`

```typescript
if (!isValidUuid(id)) throw new ForbiddenError();
```

**边界说明**：
- 在**任何其他检查之前**执行
- 防止 SQL 注入和无效 ID 查询
- 即使是管理员，ID 格式无效也会被拒绝

#### 第 1 层：系统公共文件豁免

**关键代码位置：** `assets.ts:215-220`

```typescript
const publicSettings = await this.knex
    .select('project_logo', 'public_background', 'public_foreground', 'public_favicon')
    .from('directus_settings')
    .first();

const systemPublicKeys: string[] = Object.values(publicSettings || {});
```

**豁免的文件类型**：

| 设置项 | 用途 |
|--------|------|
| `project_logo` | 项目 Logo |
| `public_background` | 公共背景 |
| `public_foreground` | 公共前景 |
| `public_favicon` | 网站图标 |

**边界说明**：
- 这些文件**无需任何权限**即可访问
- 用于登录页面、公共页面等无需认证的场景
- 豁免后跳过第 2、3 层检查，但仍需通过第 0、4 层

#### 第 2 层：管理员豁免

**关键代码位置：** `assets.ts:231`

```typescript
if (!systemPublicKeys.includes(id) && this.accountability && this.accountability.admin !== true) {
    // 执行权限检查
}
```

**边界说明**：
- 条件：`this.accountability.admin === true`
- 管理员跳过第 3 层的权限系统检查
- 但仍需通过第 0、1、4 层检查

#### 第 3 层：权限系统检查（核心）

**关键代码位置：** `validate-item-access.ts:44-172`

这是最复杂的一层，使用 `validateItemAccess` 函数执行**多级权限检查**。

##### 3.1 validateItemAccess 内部流程

```
调用 validateItemAccess
    ↓
[1] 构建查询 AST
    ↓
[2] processAst：注入权限规则
    │
    ├── 集合级权限检查
    └── 字段级权限检查
    ↓
[3] 注入主键过滤条件
    ↓
[4] fetchPermittedAstRootFields：实际查询数据库
    ↓
[5] 检查返回数量是否匹配
    │
    ├── 不匹配 → accessAllowed = false
    └── 匹配 → 继续
              ↓
        [6] 计算 allowedRootFields
              ↓
        [7] 返回结果
```

##### 3.2 权限检查的三个维度

| 维度 | 说明 | 检查方式 |
|------|------|----------|
| **集合级** | 是否有权访问 `directus_files` 集合 | 检查是否有 `read` 权限的策略 |
| **字段级** | 可以访问哪些字段 | `fetchAllowedFields` 获取允许字段 |
| **记录级** | 可以访问哪些具体文件 | 构建带权限过滤的查询，验证返回数量 |

##### 3.3 记录级权限验证原理

**关键代码位置：** `validate-item-access.ts:127-135`

```typescript
const items = await fetchPermittedAstRootFields(ast, {
    schema: context.schema,
    accountability: options.accountability,
    knex: context.knex,
    action: options.action,
});

const expectedCount = isSingleton && !hasPrimaryKeys ? 1 : options.primaryKeys!.length;
const hasAccess = items && items.length === expectedCount;
```

**原理说明**：
1. 构建一个包含**所有权限过滤条件**的查询
2. 注入 `id IN [请求的 ID]` 过滤条件
3. 执行查询，检查返回的记录数
4. **如果返回数量 == 期望数量** → 有权限
5. **如果返回数量 < 期望数量** → 部分或全部记录无权限 → 拒绝访问

**示例**：
- 请求 ID：`[A, B, C]`
- 期望数量：3
- 实际返回：`[A, C]`（只有 2 条）
- 结果：`accessAllowed = false`（因为 B 无权限）

##### 3.4 字段级权限计算

**关键代码位置：** `validate-item-access.ts:153-168`

```typescript
// 如果 returnAllowedRootFields，返回交集 of allowed fields across all items
if (options.returnAllowedRootFields) {
    // 如果没有记录级规则，直接返回 permissioned fields
    if (!hasItemRules) {
        return {
            accessAllowed,
            allowedRootFields: permissionedFields!,
        };
    }
    
    // 有记录级规则时，计算所有记录的字段交集
    const allowedRootFields =
        items.length > 0 ? Object.keys(items[0]!).filter((field) => items.every((item: any) => item[field] === 1)) : [];
    
    return {
        accessAllowed,
        allowedRootFields,
    };
}
```

**边界说明**：
- 如果没有记录级权限规则 → 所有用户看到相同的字段集
- 如果有记录级权限规则 → 计算所有请求记录的**字段交集**
- 只有**所有记录都允许访问**的字段才会返回

#### 第 4 层：文件物理存在性验证

**关键代码位置：** `assets.ts:253-257`

```typescript
const file = (await this.sudoFilesService.readOne(id, { limit: 1 })) as File;

const exists = await storage.location(file.storage).exists(file.filename_disk);

if (!exists) throw new ForbiddenError();
```

**边界说明**：
- 即使数据库记录存在、权限检查通过
- 如果**物理文件不存在**，仍返回 403
- 这是安全措施：防止通过数据库记录猜测文件存在性

#### 第 5 层：字段级权限过滤

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

**边界说明**：
- `type` 和 `filesize` 字段**始终返回**（用于 HTTP 响应头）
- 其他字段根据 `allowedRootFields` 过滤
- 即使有权限访问文件，某些元数据字段可能被过滤

### 3.3 条件请求的前置检查

**关键代码位置：** `assets.ts:336-387`（控制器层面）

对于带 `If-None-Match` 或 `If-Modified-Since` 头的条件请求，有一个**额外的权限检查**：

```typescript
if (revalidate) {
    const ifNoneMatch = req.headers['if-none-match'];
    const ifModifiedSince = req.headers['if-modified-since'];
    
    if (ifNoneMatch || ifModifiedSince) {
        // 先执行权限检查
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
        
        // 再检查缓存
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

**边界说明**：
- 条件请求（304 Not Modified）也需要**先通过权限检查**
- 防止未授权用户通过 304 响应猜测文件修改时间

### 3.4 权限检查的关键边界总结

| 检查层 | 代码位置 | 检查内容 | 豁免条件 |
|--------|----------|----------|----------|
| 0. UUID 格式 | `assets.ts:227` | ID 是否为有效 UUID | 无 |
| 1. 系统公共文件 | `assets.ts:215-220` | ID 是否在公共文件列表中 | 公共文件 ID |
| 2. 管理员 | `assets.ts:231` | `accountability.admin === true` | 管理员 |
| 3. 权限系统 | `validate-item-access.ts` | 集合+字段+记录级权限 | 无（非豁免用户必须通过） |
| 4. 物理存在 | `assets.ts:255-257` | 文件是否存在于存储 | 无 |
| 5. 字段过滤 | `assets.ts:57-74` | 过滤返回字段 | `type`, `filesize` 始终返回 |

### 3.5 典型权限场景

#### 场景 1：公共文件访问
```
请求：GET /assets/{project_logo_id}
检查：
  0. UUID 有效 ✓
  1. 在公共文件列表中 ✓ → 跳过 2、3 层
  4. 物理存在 ✓
结果：成功访问
```

#### 场景 2：管理员访问任意文件
```
请求：GET /assets/{any_file_id}
检查：
  0. UUID 有效 ✓
  1. 非公共文件 → 继续
  2. 是管理员 ✓ → 跳过第 3 层
  4. 物理存在 ✓
结果：成功访问
```

#### 场景 3：普通用户无权限
```
请求：GET /assets/{restricted_file_id}
检查：
  0. UUID 有效 ✓
  1. 非公共文件 → 继续
  2. 非管理员 → 继续
  3. validateItemAccess → 返回 0 条记录
     → accessAllowed = false
结果：ForbiddenError
```

#### 场景 4：数据库存在但文件已删除
```
请求：GET /assets/{file_with_missing_physical_file}
检查：
  0. UUID 有效 ✓
  1. 非公共文件 → 继续
  2. 管理员/有权限 ✓
  3. 权限检查通过 ✓
  4. storage.exists() → false
结果：ForbiddenError
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
| `ASSETS_TRANSFORM_IMAGE_MAX_DIMENSION` | 最大图像尺寸（像素） | - |
| `ASSETS_TRANSFORM_MAX_OPERATIONS` | 单次请求最大变换次数 | - |
| `ASSETS_TRANSFORM_MAX_CONCURRENT` | 最大并发变换数 | - |
| `ASSETS_TRANSFORM_TIMEOUT` | 变换超时时间（毫秒） | - |
| `ASSETS_INVALID_IMAGE_SENSITIVITY_LEVEL` | 无效图像敏感度 | `warning` |
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
| 变换工具 | `api/src/utils/transformations.ts` | 变换参数解析、焦点裁剪算法 |
| 权限验证入口 | `api/src/permissions/modules/validate-access/validate-access.ts` | 权限检查入口 |
| 项权限验证 | `api/src/permissions/modules/validate-access/lib/validate-item-access.ts` | 记录级权限验证核心 |
| 元数据提取 | `api/src/services/files/lib/extract-metadata.ts` | 文件元数据提取 |

---

## 6. 总结

### 6.1 图片变换执行顺序总结

| 阶段 | 顺序 | 操作 | 条件 |
|------|------|------|------|
| 参数构造 | 1 | 初始化 transforms 数组 | 有 `transforms` 参数 |
| 参数构造 | 2 | 添加 `toFormat` | 有 `format` 或 `quality` |
| 参数构造 | 3 | 添加 resize/extract | 有 `width`/`height` |
| 实际执行 | 1 | 自动 `rotate()` | 无显式 rotate |
| 实际执行 | 2+ | 按数组顺序执行 | transforms 数组 |

**焦点裁剪特殊顺序**：resize → extract

### 6.2 缓存与限流检查顺序

```
缓存检查（命中则返回）
    ↓
尺寸检查
    ↓
并发限流检查
    ↓
创建 Sharp 实例 + 设置超时
    ↓
执行变换
```

### 6.3 权限边界总结

| 层级 | 检查内容 | 豁免 |
|------|----------|------|
| 0 | UUID 格式 | 无 |
| 1 | 系统公共文件 | 公共文件 ID |
| 2 | 管理员 | 管理员账号 |
| 3 | 权限系统（集合+字段+记录） | 无 |
| 4 | 物理文件存在 | 无 |
| 5 | 字段过滤 | `type`, `filesize` |

### 6.4 架构特点

1. **分层设计**：Controller → Service → Storage 的三层架构
2. **流式处理**：文件上传和变换都使用流，避免内存溢出
3. **缓存优化**：变换后的图像自动缓存，提升性能
4. **权限多层防护**：六层权限检查，确保资产安全
5. **焦点裁剪智能**：基于焦点坐标的智能缩放 + 裁剪组合

### 6.5 安全机制

1. **MIME 类型白名单**：防止恶意文件上传
2. **文件大小限制**：防止拒绝服务攻击
3. **UUID 验证**：防止 SQL 注入
4. **并发控制**：防止资源耗尽
5. **尺寸限制**：防止内存溢出
6. **多层权限**：防止越权访问
7. **物理存在验证**：防止通过数据库记录猜测
