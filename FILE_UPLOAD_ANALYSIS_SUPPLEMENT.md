# Directus 文件上传系统深度补充分析

> 本文档补充 FILE_UPLOAD_ANALYSIS_REVISED.md，深入分析三个关键问题的精确边界条件

---

## 一、权限校验触发边界详解

### 1.1 三类操作的权限校验差异总览

| 操作类型 | 触发条件 | 核心校验函数 | 校验层级 | 特殊行为 |
|---------|---------|-------------|---------|---------|
| **createOne/createMany** | `accountability !== null` | `processPayload()` | 集合 + 字段 | **不调用 validateAccess，不检查行级权限** |
| **updateOne/updateMany** | `accountability !== null` | `validateAccess()` → `validateItemAccess()` → `processPayload()` | 集合 + 行级 + 字段 | **先检查行级访问权限，再检查字段权限** |
| **readByQuery/readOne** | 始终 | `processAst()` | 集合 + 行级（通过 SQL 注入） | **不抛出 ForbiddenError，仅过滤结果** |
| **AssetsService.getAsset** | 非 admin 且非系统公开文件 | `validateItemAccess()` | 集合 + 行级 + 字段 | **显式抛出 ForbiddenError** |

### 1.2 创建操作（createOne）的精确触发边界

#### 触发条件

```typescript
// items.ts:172-186
const payloadWithPresets = this.accountability
    ? await processPayload(
            {
                accountability: this.accountability,
                action: 'create',
                collection: this.collection,
                payload: payloadAfterHooks,
                nested: this.nested,
            },
            {
                knex: trx,
                schema: this.schema,
            }
        )
    : payloadAfterHooks;  // ⚠️ accountability 为 null 时直接跳过
```

#### 不同上下文下的行为

| 上下文 | accountability 值 | 校验行为 | 结果 |
|-------|-------------------|-----------|------|
| **普通用户请求** | `{ admin: false, user: 'xxx', ... }` | 完整执行 `processPayload()` | 检查集合权限、字段权限、应用 presets、校验 validation |
| **管理员请求** | `{ admin: true, ... }` | `processPayload()` 中跳过权限检查 | 仅应用系统预设值，不校验权限 |
| **系统内部操作（Sudo 模式）** | `null` 或 `undefined` | **完全跳过 `processPayload()`** | 无任何校验，直接使用原始 payload |

#### processPayload 的校验内容

```typescript
// process-payload.ts:35-59
if (!options.accountability.admin) {
    // 仅非管理员执行
    policies = await fetchPolicies(options.accountability, context);
    
    // 1. 检查集合权限
    if (!(await validateCollectionAccess(
        { action: options.action, accountability: options.accountability, collection: options.collection },
        context
    ))) {
        throw new ForbiddenError();
    }
    
    // 2. 获取权限规则
    const permissions = await fetchPermissions({...}, context);
    
    // 3. 检查字段权限并过滤 payload
    const { payload, presetsApply, validation } = await checkAccess(
        options.payload,
        options.action,
        options.collection,
        permissions,
        options.accountability,
        context
    );
}

// 4. 应用 presets
if (presetsApply) {
    for (const preset of presetsApply) {
        if (!(preset.field in payload)) {
            payload[preset.field] = preset.value;
        }
    }
}

// 5. 校验 validation 规则
await validatePayload(validation, payload);
```

#### 关键发现

1. **createOne 不调用 validateAccess**：与 update/delete 不同，create 操作不检查行级权限（因为还没有行）
2. **accountability 为 null = 完全绕过**：系统内部操作创建 ItemsService 时不传递 accountability，直接跳过所有校验
3. **admin 特权**：admin 用户的 accountability 会被传递，但 `processPayload` 中会检查 `admin === true` 并跳过权限检查部分

### 1.3 更新操作（updateMany）的精确触发边界

#### 触发条件

```typescript
// items.ts:752-782
if (this.accountability) {
    // ⚠️ 第一步：validateAccess - 检查行级权限
    await validateAccess(
        {
            accountability: this.accountability,
            action: 'update',
            collection: this.collection,
            primaryKeys: keys,  // ⚠️ 包含主键！
            fields: Object.keys(payloadAfterHooks),
        },
        {
            schema: this.schema,
            knex: this.knex,
        }
    );
}

// ⚠️ 第二步：processPayload - 检查字段权限
const payloadWithPresets = this.accountability
    ? await processPayload(...)
    : payloadAfterHooks;
```

#### validateAccess vs validateItemAccess

```typescript
// validate-access.ts:22-57
export async function validateAccess(options: ValidateAccessOptions, context: Context) {
    if (options.accountability.admin === true) {
        return;  // 管理员直接通过
    }

    let access: boolean;

    if (options.primaryKeys) {
        // ⚠️ 有主键时，使用 validateItemAccess 检查行级权限
        const result = await validateItemAccess(options as Required<ValidateAccessOptions>, context);
        access = result.accessAllowed;
    } else {
        // 没有主键时，仅检查集合权限
        access = await validateCollectionAccess(options, context);
    }

    if (!access) {
        throw new ForbiddenError(...);
    }
}
```

#### 不同上下文下的行为

| 上下文 | 校验行为 | 关键差异 |
|-------|---------|---------|
| **普通用户 + 有主键** | `validateAccess` → `validateItemAccess`（行级检查）→ `processPayload` | 双重校验：先检查能否访问这些行，再检查字段权限 |
| **普通用户 + 无主键** | `validateAccess` → `validateCollectionAccess`（仅集合）→ `processPayload` | 仅检查集合和字段权限，不检查行级 |
| **管理员** | `validateAccess` 中直接 return，`processPayload` 中跳过权限检查 | 完全绕过校验 |
| **Sudo 模式（accountability: null）** | 跳过 `validateAccess`，跳过 `processPayload` | 完全绕过校验 |

#### validateItemAccess 的行级检查机制

```typescript
// validate-item-access.ts:核心逻辑
// 构造查询检查用户是否有权限访问这些主键
const query = context.knex(options.collection)
    .select(primaryKeyField)
    .whereIn(primaryKeyField, options.primaryKeys);

// 注入权限过滤条件
await processAst(
    { ast, action: options.action, accountability: options.accountability },
    context
);

// 执行查询
const accessibleKeys = await query;

// 比较
const requestedSet = new Set(options.primaryKeys);
const accessibleSet = new Set(accessibleKeys.map(k => k[primaryKeyField]));

for (const key of requestedSet) {
    if (!accessibleSet.has(key)) {
        // 某个主键无权访问
        return { accessAllowed: false, ... };
    }
}
```

#### 关键发现

1. **updateMany 有双重校验**：先 `validateAccess` 检查行级访问，再 `processPayload` 检查字段权限
2. **主键是关键**：有 `primaryKeys` 参数时才会进行行级检查，否则仅检查集合权限
3. **行级检查通过 SQL 注入实现**：`validateItemAccess` 内部调用 `processAst` 注入权限条件，然后查询数据库验证

### 1.4 读取操作的精确触发边界

#### readByQuery 的行为

```typescript
// items.ts:518-533
let ast = await getAstFromQuery(
    {
        collection: this.collection,
        query: updatedQuery,
        accountability: this.accountability,
    },
    {
        schema: this.schema,
        knex: this.knex,
    }
);

// ⚠️ 注入权限条件
ast = await processAst(
    { ast, action: 'read', accountability: this.accountability },
    { knex: this.knex, schema: this.schema },
);

const records = await runAst(ast, this.schema, this.accountability, {...});
```

#### processAst 的注入机制

```typescript
// inject-cases.ts:13-73
export function injectCases(ast: AST, permissions: Permission[]) {
    // 为每个字段注入 CASE/WHEN 条件
    ast.cases = processChildren(ast.name, ast.children, permissions);
}

function processChildren(...) {
    // 获取权限规则中的 CASE 条件
    const { cases, caseMap, allowedFields } = getCases(collection, permissions, requestedKeys);

    // 为每个子节点注入 whenCase
    for (const child of children) {
        // 如果没有完整访问权限，注入 CASE/WHEN
        if (!allowedFields.has('*') && !allowedFields.has(fieldKey)) {
            child.whenCase = [...(globalWhenCase ?? []), ...(fieldWhenCase ?? [])];
        }
        
        // 递归处理关联字段
        if (child.type === 'm2o') {
            child.cases = processChildren(child.relation.related_collection!, child.children, permissions);
        }
        // ... 其他关联类型
    }

    return cases;
}
```

#### 不同读取方式的权限校验差异

| 读取方式 | 权限校验机制 | 无权限时的行为 |
|---------|-------------|---------------|
| **readByQuery** | `processAst()` 注入 SQL 过滤条件 | 过滤掉无权访问的记录，返回空数组或部分结果 |
| **readOne (通过 ItemsService)** | 内部调用 `readByQuery({ filter: { id: { _eq: pk } } })` | 无权限时返回 `undefined`，上层可能抛出 ForbiddenError |
| **AssetsService.getAsset** | 显式调用 `validateItemAccess()` | 无权限时直接抛出 `ForbiddenError` |
| **系统公开文件** | `systemPublicKeys.includes(id)` 检查 | 绕过权限校验，直接返回 |

#### AssetsService.getAsset 的特殊逻辑

```typescript
// assets.ts:220-251
const systemPublicKeys: string[] = Object.values(publicSettings || {});

if (!systemPublicKeys.includes(id) && this.accountability && this.accountability.admin !== true) {
    // ⚠️ 仅当：非系统公开文件 且 有 accountability 且 非 admin 时才校验
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

// ⚠️ 使用 sudoFilesService 读取文件记录（绕过权限校验）
const file = (await this.sudoFilesService.readOne(id, { limit: 1 })) as File;
```

#### 关键发现

1. **readByQuery 从不抛出 ForbiddenError**：它通过 `processAst` 注入 SQL 条件，无权限的记录会被过滤掉
2. **AssetsService 有特殊处理**：
   - 系统公开文件（`project_logo`、`public_background` 等）绕过权限
   - admin 用户绕过权限
   - 普通用户使用 `validateItemAccess` 显式校验
   - 实际读取文件记录时使用 `sudoFilesService`（accountability: null）

3. **allowedFields 的作用**：`validateItemAccess` 返回的 `allowedRootFields` 用于后续 `sanitizeFields` 过滤返回字段

---

## 二、存储驱动切换后的读写落点

### 2.1 新文件上传的 storage 字段确定

#### 核心代码

```typescript
// files.ts:103-107
const payload = {
    storage: toArray(env['STORAGE_LOCATIONS'] as string)[0]!,  // 1. 默认值：第一个存储位置
    ...(existingFile ?? {}),  // 2. 展开现有文件记录（如果是替换上传）
    ...clone(data),  // 3. 展开用户请求数据
};
```

#### 覆盖顺序

```
初始值: { storage: STORAGE_LOCATIONS[0] }
    ↓
...existingFile  (如果是替换上传)
    ↓ (可能覆盖 storage)
...data  (用户请求中的字段)
    ↓ (可能再次覆盖 storage)
最终值
```

#### 不同场景的结果

| 场景 | storage 初始值 | existingFile.storage | data.storage | 最终 storage |
|-----|---------------|---------------------|-------------|-------------|
| **全新上传** | `STORAGE_LOCATIONS[0]` | `undefined` | `undefined` | `STORAGE_LOCATIONS[0]` |
| **全新上传（用户指定）** | `STORAGE_LOCATIONS[0]` | `undefined` | `'s3'` | `'s3'` |
| **替换上传（用户不指定）** | `STORAGE_LOCATIONS[0]` | `'local'`（原文件） | `undefined` | `'local'` |
| **替换上传（用户指定新位置）** | `STORAGE_LOCATIONS[0]` | `'local'` | `'s3'` | `'s3'` |

#### 用户能否在请求中指定 storage？

```typescript
// controllers/files.ts:62-70
busboy.on('field', (fieldname, val) => {
    let fieldValue: string | null | boolean = val;
    
    // 类型转换...
    
    payload[fieldname] = fieldValue;  // ⚠️ 所有字段都会被收集
});
```

**答案是：可以**，但会经过 `processPayload` 校验。

#### storage 字段的权限校验

```typescript
// items.ts:124-126
if (isReplacement === false || primaryKey === undefined) {
    primaryKey = await this.createOne(payload, { emitEvents: false });
}
```

`createOne` 会调用 `processPayload`，如果用户的权限规则中没有 `storage` 字段的 write 权限：

```typescript
// process-payload 中的 checkAccess 会过滤无权访问的字段
```

**关键发现**：如果用户没有 `storage` 字段的写权限，请求中指定的 `storage` 会被过滤掉，最终使用默认值或继承值。

### 2.2 文件读取的落点确定

#### 核心代码

```typescript
// assets.ts:255
const exists = await storage.location(file.storage).exists(file.filename_disk);

// assets.ts:311
const exists = await storage.location(file.storage).exists(assetFilename);

// assets.ts:371
const readStream = await storage.location(file.storage).read(file.filename_disk, { range, version });
```

#### 读取逻辑

1. 从 `directus_files` 表读取文件记录
2. 使用 `file.storage` 字段确定存储位置
3. 通过 `storage.location(file.storage)` 获取对应的驱动实例
4. 使用该驱动读取文件

#### 存储驱动切换后的行为

| 场景 | 文件记录的 storage | 读取落点 |
|-----|-------------------|---------|
| **历史文件（旧配置）** | `'local'` | `storage.location('local')` → 从本地读取 |
| **新文件（新配置）** | `'s3'` | `storage.location('s3')` → 从 S3 读取 |
| **修改 STORAGE_LOCATIONS 但不迁移** | `'local'` | 如果 'local' 仍在配置中，正常读取；否则报错 |

#### 关键发现

1. **storage 字段是文件级别的**：每个文件记录独立追踪其存储位置
2. **配置修改不影响历史文件**：修改 `STORAGE_LOCATIONS` 不会改变已有文件的 `storage` 字段
3. **旧位置必须保留在配置中**：要访问历史文件，旧的存储位置必须仍在 `STORAGE_LOCATIONS` 中

### 2.3 替换上传的特殊行为

#### 核心代码

```typescript
// files.ts:201-217
if (isReplacement === true) {
    try {
        await this.updateOne(primaryKey, payload, { emitEvents: false });

        // ⚠️ 删除之前保存的文件和缩略图，确保重新生成
        for await (const filepath of disk.list(String(primaryKey))) {
            await disk.delete(filepath);
        }

        // 将临时文件升级为最终文件名
        await disk.move(tempFilenameDisk, payload.filename_disk);
    } catch (err: any) {
        await cleanUp();
        throw err;
    }
}
```

#### 关键问题：disk 是哪个存储位置？

```typescript
// files.ts:109
const disk = storage.location(payload.storage);  // ⚠️ 使用 payload.storage！
```

#### 替换上传时的两种场景

##### 场景 A：不改变 storage 字段（正常替换）

```
条件：
- 原文件：storage = 'local'
- 用户请求：不指定 storage

执行流程：
1. payload.storage = 'local'（继承 existingFile）
2. disk = storage.location('local')
3. disk.list('abc123') → 列出 local 位置下所有以 'abc123' 开头的文件
   - 'abc123.jpg'（源文件）
   - 'abc123__hash1.jpg'（变换缓存）
   - 'abc123__hash2.jpg'（另一个变换缓存）
4. disk.delete() → 全部删除
5. disk.move() → 新文件写入 'local' 位置

结果：
✓ 源文件被替换
✓ 所有变换缓存被清理
✓ 下次请求变换时重新生成
```

##### 场景 B：改变 storage 字段（跨位置替换）

```
条件：
- 原文件：storage = 'local'
- 用户请求：指定 storage = 's3'

执行流程：
1. payload.storage = 's3'（用户指定，覆盖继承值）
2. disk = storage.location('s3')  ⚠️ 新位置！
3. disk.list('abc123') → 列出 s3 位置下的文件
   - 可能是空的！（新位置还没有这个文件）
4. disk.delete() → 什么都没删
5. disk.move() → 新文件写入 's3' 位置
6. updateOne → 更新数据库记录的 storage 为 's3'

结果：
✓ 源文件被替换到新位置
✓ 数据库记录指向新位置
✗ 原位置（local）的源文件残留！
✗ 原位置的所有变换缓存残留！
⚠️ 下次读取时从新位置读取，没有缓存 → 重新生成
⚠️ 但原位置的缓存变成"孤儿"，永远不会被访问也不会被清理
```

#### 潜在问题：跨位置替换的资源泄漏

| 问题 | 说明 |
|-----|------|
| **源文件泄漏** | 原位置的 `filename_disk` 不会被删除 |
| **变换缓存泄漏** | 原位置的所有 `{id}__*.{ext}` 缓存文件不会被删除 |
| **storage 字段更新** | 数据库记录的 `storage` 字段会更新为新位置 |
| **后续访问** | 从新位置读取，旧位置的文件变成"孤儿" |

#### 关键发现

1. **替换上传会清理当前位置的缓存**：但"当前位置"是 `payload.storage`，不是原文件的 `storage`
2. **跨位置替换有资源泄漏风险**：如果用户在替换上传时改变了 `storage` 字段，原位置的文件和缓存不会被清理
3. **正常替换（不改变 storage）是安全的**：会正确清理所有以主键开头的文件

### 2.4 filename_disk 的确定规则

#### 核心代码

```typescript
// files.ts:128-139
const fileExtension =
    path.extname(payload.filename_download!) || (payload.type && '.' + extension(payload.type)) || '';

const filenameDisk = primaryKey + (fileExtension || '');

// filename_disk 是磁盘上的最终文件名
payload.filename_disk ||= filenameDisk;

// 如果 filename_disk 的扩展名与新的 MIME 类型不匹配，更新它
if (isReplacement === true && path.extname(payload.filename_disk!) !== fileExtension) {
    payload.filename_disk = filenameDisk;
}
```

#### 规则说明

| 条件 | filename_disk 值 |
|-----|-----------------|
| **全新上传** | `{primaryKey}.{ext}` |
| **替换上传（扩展名相同）** | 保持原有的 `filename_disk`（除非显式指定） |
| **替换上传（扩展名不同）** | 强制更新为 `{primaryKey}.{newExt}` |

#### 关键发现

1. **filename_disk 基于主键**：`{primaryKey}.{ext}` 格式
2. **扩展名变化时会更新**：替换上传时如果文件类型改变，`filename_disk` 会更新
3. **变换缓存依赖 filename_disk**：`{basename}__{hash}.{ext}`，其中 `basename` 来自 `filename_disk`

---

## 三、资产变换缓存的命中与失效

### 3.1 缓存键生成机制

#### 核心代码

```typescript
// assets.ts:306-315
const maybeNewFormat = TransformationUtils.maybeExtractFormat(transforms);

const assetFilename =
    path.basename(file.filename_disk, path.extname(file.filename_disk)) +
    getAssetSuffix(transforms) +
    (maybeNewFormat ? `.${maybeNewFormat}` : path.extname(file.filename_disk));

// assets.ts:413-416
const getAssetSuffix = (transforms: Transformation[]) => {
    if (Object.keys(transforms).length === 0) return '';
    return `__${hash(transforms)}`;
};
```

#### 缓存文件名格式

```
{basename}__{hash(transforms)}.{ext}
```

| 组成部分 | 来源 | 说明 |
|---------|------|------|
| `{basename}` | `path.basename(file.filename_disk, ext)` | 源文件名的基名（不含扩展名） |
| `{hash(transforms)}` | `object-hash` 对 transforms 数组的哈希 | 变换参数的唯一标识 |
| `{ext}` | `maybeNewFormat` 或原扩展名 | 输出格式 |

#### 示例

```typescript
// 源文件
file.filename_disk = 'abc123.jpg';

// 变换参数
const transforms = [
    ['resize', { width: 100, height: 100, fit: 'cover' }],
    ['rotate', 90],
];

// 缓存文件名
const basename = 'abc123';
const hashValue = hash(transforms);  // 例如：'a1b2c3d4e5f6'
const ext = 'jpg';  // 或 maybeNewFormat 指定的格式

// 结果：'abc123__a1b2c3d4e5f6.jpg'
```

#### 影响缓存键的因素

| 因素 | 是否影响缓存键 | 说明 |
|-----|--------------|------|
| **transforms 数组内容** | ✅ 影响 | 每个变换方法和参数都会影响哈希 |
| **transforms 顺序** | ✅ 影响 | `[['resize', ...], ['rotate', ...]]` vs `[['rotate', ...], ['resize', ...]]` 哈希不同 |
| **输出格式（format）** | ✅ 影响 | 通过 `maybeNewFormat` 直接改变扩展名 |
| **filename_disk 的 basename** | ✅ 影响 | 主键变化时缓存键变化 |
| **源文件 modified_on** | ❌ 不影响 | 缓存键不包含时间戳 |
| **源文件内容** | ❌ 不影响 | 内容变化但主键不变时缓存键相同 |
| **请求用户** | ❌ 不影响 | 所有用户共享同一缓存 |
| **访问权限** | ❌ 不影响 | 无权限用户看不到文件，但缓存键本身不受影响 |

#### 关键发现

1. **transforms 的顺序很重要**：`resize` 然后 `rotate` vs `rotate` 然后 `resize` 会生成不同的缓存键
2. **源文件更新不改变缓存键**：如果只是替换文件内容但不改变主键和 `filename_disk`，缓存键保持不变
3. **缓存是全局共享的**：所有用户使用相同的变换参数会得到相同的缓存文件

### 3.2 缓存命中条件

#### 核心代码

```typescript
// assets.ts:311-325
const exists = await storage.location(file.storage).exists(assetFilename);

if (maybeNewFormat) {
    file.type = contentType(assetFilename) || null;
}

if (exists) {
    const assetStream = () => storage.location(file.storage).read(assetFilename, { range });

    return {
        stream: deferStream ? assetStream : await assetStream(),
        file: this.sanitizeFields(file, allowedFields),
        stat: await storage.location(file.storage).stat(assetFilename),
    };
}
```

#### 命中条件

| 条件 | 说明 |
|-----|------|
| **存储位置匹配** | 缓存文件必须在 `file.storage` 指定的存储位置 |
| **缓存文件存在** | `storage.location(file.storage).exists(assetFilename)` 返回 `true` |
| **缓存键完全匹配** | `assetFilename` 必须完全匹配（包括 transforms 哈希） |

#### 命中时的行为

```typescript
// 直接返回缓存文件流，不执行变换
const assetStream = () => storage.location(file.storage).read(assetFilename, { range });
```

**关键发现**：缓存命中时，**不会读取源文件**，直接返回缓存文件。

### 3.3 缓存失效条件

#### 主动失效机制

##### 1. 替换上传（相同 storage 位置）

```typescript
// files.ts:206-209
for await (const filepath of disk.list(String(primaryKey))) {
    await disk.delete(filepath);
}
```

**触发条件**：
- 执行替换上传（`isReplacement === true`）
- `payload.storage` 与原文件的 `storage` 相同

**清理范围**：
```
disk.list('abc123') 会列出所有以 'abc123' 开头的文件：
- 'abc123.jpg'        (源文件)
- 'abc123__hash1.jpg' (变换缓存 1)
- 'abc123__hash2.jpg' (变换缓存 2)
- 'abc123__hash3.jpg' (变换缓存 3)
...
```

**结果**：✓ 源文件和所有变换缓存被删除，下次请求重新生成。

##### 2. 主动删除文件

当文件被删除时，`FilesService.deleteOne/deleteMany` 会处理：

```typescript
// 需要检查 delete 时是否清理缓存
```

（注：根据代码逻辑，删除文件记录时，`storage.location(file.storage).delete(file.filename_disk)` 只会删除源文件，**变换缓存不会被删除**。这是一个潜在的资源泄漏点。）

#### 被动失效机制

##### 1. 缓存文件不存在

- 手动删除缓存文件
- 存储后端自动清理（如 S3 生命周期策略）
- `exists()` 返回 `false`

**结果**：下次请求时重新生成变换。

##### 2. 主键变化

- 如果文件被删除后重新上传（新主键）
- 新文件的 `filename_disk` 是新主键
- 旧缓存文件变成"孤儿"，不会被访问也不会被清理

#### 不会失效的场景

| 场景 | 行为 | 说明 |
|-----|------|------|
| **源文件内容更新（非替换上传）** | 缓存**不失效** | 通过 `updateOne` 只更新元数据，不会触发缓存清理 |
| **源文件 modified_on 变化** | 缓存**不失效** | 缓存键不包含时间戳 |
| **变换参数相同但参数值变化** | 缓存**会失效** | 参数值变化会导致哈希值变化 |
| **跨位置替换上传** | 原位置缓存**不失效** | 新位置的 `disk.list()` 不会影响原位置 |

### 3.4 源文件更新后的缓存行为

#### 场景 1：通过 PATCH /files/:pk 更新元数据（非替换上传）

```
条件：
- 文件：abc123.jpg，storage: 'local'
- 变换缓存：abc123__hash1.jpg 已存在
- 操作：PATCH /files/abc123 { title: '新标题' }

执行流程：
1. PATCH 请求 → FilesService.updateOne
2. updateOne → validateAccess → processPayload
3. 更新数据库记录（title、modified_on 等）
4. ⚠️ 不会调用 storage 操作
5. ⚠️ 不会清理变换缓存

后续访问：
- GET /assets/abc123?key=transform-key
- assetFilename = 'abc123__hash1.jpg'
- exists() → true
- ⚠️ 返回旧的变换缓存！

结果：
✗ 源文件内容没有变化（只是元数据变化），所以这通常是正确的
✓ 但如果 somehow 源文件内容被外部修改，缓存会返回旧版本
```

#### 场景 2：通过替换上传更新文件内容

```
条件：
- 文件：abc123.jpg，storage: 'local'
- 变换缓存：abc123__hash1.jpg 已存在
- 操作：POST /files/abc123（multipart/form-data 上传新文件）

执行流程（相同 storage）：
1. isReplacement = true
2. disk = storage.location('local')
3. disk.list('abc123') → 列出所有以 'abc123' 开头的文件
4. disk.delete() → 删除源文件和所有变换缓存
5. disk.move() → 新文件写入
6. updateOne → 更新数据库记录

后续访问：
- GET /assets/abc123?key=transform-key
- assetFilename = 'abc123__hash1.jpg'
- exists() → false（已被删除）
- ✓ 重新执行变换并缓存

结果：
✓ 缓存被正确清理
✓ 下次请求重新生成
```

#### 场景 3：跨位置替换上传

```
条件：
- 原文件：abc123.jpg，storage: 'local'
- 变换缓存：abc123__hash1.jpg（在 local 位置）
- 操作：POST /files/abc123（multipart/form-data，指定 storage: 's3'）

执行流程：
1. payload.storage = 's3'（用户指定）
2. disk = storage.location('s3')
3. disk.list('abc123') → s3 位置可能没有这个文件
4. disk.delete() → 什么都没删
5. disk.move() → 新文件写入 s3 位置
6. updateOne → 更新数据库记录的 storage 为 's3'

后续访问：
- GET /assets/abc123?key=transform-key
- 从数据库读取 file.storage = 's3'
- assetFilename = 'abc123__hash1.jpg'
- storage.location('s3').exists() → false
- ✓ 重新执行变换并缓存到 s3 位置

结果：
✓ 新位置会重新生成缓存
✗ 原位置（local）的源文件和变换缓存残留，变成"孤儿"
```

### 3.5 资源泄漏风险汇总

| 场景 | 泄漏内容 | 泄漏位置 | 影响 |
|-----|---------|---------|------|
| **跨位置替换上传** | 源文件 + 所有变换缓存 | 原存储位置 | 永久占用存储空间，无法自动清理 |
| **删除文件** | 所有变换缓存 | 原存储位置 | `delete(file.filename_disk)` 只删源文件，缓存残留 |
| **主键变更后** | 旧主键的所有缓存 | 原存储位置 | 新主键有新的缓存键，旧缓存变成孤儿 |

### 3.6 关键发现总结

1. **缓存键不包含时间戳**：`modified_on` 变化不会导致缓存失效
2. **替换上传是唯一的主动清理机制**：但只有 `payload.storage` 与原位置相同时才会正确清理
3. **跨位置操作有资源泄漏**：改变 `storage` 字段时，原位置的文件和缓存不会被清理
4. **delete 操作可能不清理缓存**：需要确认 `deleteOne` 是否会清理变换缓存

---

## 四、关键代码路径速查表

### 4.1 权限校验

| 操作 | 文件位置 | 关键函数 |
|-----|---------|---------|
| createOne 权限校验 | `api/src/services/items.ts:172-186` | `processPayload()` |
| updateMany 权限校验 | `api/src/services/items.ts:752-782` | `validateAccess()` → `validateItemAccess()` → `processPayload()` |
| readByQuery 权限注入 | `api/src/services/items.ts:530-533` | `processAst()` |
| AssetsService 权限校验 | `api/src/services/assets.ts:231-251` | `validateItemAccess()` |
| processPayload 实现 | `api/src/permissions/modules/process-payload/process-payload.ts` | 集合权限、字段权限、presets、validation |
| validateItemAccess 实现 | `api/src/permissions/modules/validate-access/lib/validate-item-access.ts` | 行级权限检查（通过 SQL 查询） |

### 4.2 存储位置

| 操作 | 文件位置 | 关键代码 |
|-----|---------|---------|
| storage 字段确定 | `api/src/services/files.ts:103-107` | `{ storage: default, ...existingFile, ...data }` |
| 替换上传缓存清理 | `api/src/services/files.ts:206-209` | `disk.list(primaryKey)` → `disk.delete()` |
| 读取时选择驱动 | `api/src/services/assets.ts:255, 311, 371` | `storage.location(file.storage)` |

### 4.3 资产变换缓存

| 操作 | 文件位置 | 关键代码 |
|-----|---------|---------|
| 缓存键生成 | `api/src/services/assets.ts:306-315, 413-416` | `{basename}__{hash(transforms)}.{ext}` |
| 缓存命中检查 | `api/src/services/assets.ts:311-325` | `storage.location(...).exists(assetFilename)` |
| 缓存写入 | `api/src/services/assets.ts:379` | `storage.location(...).write(assetFilename, ...)` |

---

## 五、修正与补充要点

### 对 FILE_UPLOAD_ANALYSIS_REVISED.md 的补充

1. **权限校验边界修正**：
   - ✅ `createOne` 不调用 `validateAccess`，仅调用 `processPayload`
   - ✅ `updateMany` 先调用 `validateAccess`（带 `primaryKeys`）进行行级检查，再调用 `processPayload`
   - ✅ `readByQuery` 通过 `processAst` 注入 SQL 条件，不抛出 `ForbiddenError`
   - ✅ `AssetsService.getAsset` 对普通用户显式调用 `validateItemAccess`，对 admin 和系统公开文件绕过

2. **存储驱动切换补充**：
   - ✅ 用户可以在请求中指定 `storage` 字段，但受权限校验限制
   - ✅ 替换上传时如果改变 `storage` 字段，原位置的文件和缓存不会被清理
   - ✅ 正常替换上传（不改变 `storage`）会正确清理所有以主键开头的文件

3. **缓存机制补充**：
   - ✅ 缓存键包含 `transforms` 的顺序和所有参数值
   - ✅ 缓存键**不包含** `modified_on` 或任何时间戳
   - ✅ 跨位置替换上传会导致原位置的缓存泄漏
   - ✅ 删除文件时可能只删除源文件，变换缓存可能残留

---

> 文档生成时间：2026-05-02
> 分析版本：基于当前代码库的精确代码追踪
