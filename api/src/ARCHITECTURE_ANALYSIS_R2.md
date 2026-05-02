# Directus 数据读写架构深度分析报告（补充版）

## 1. 服务层到数据库的完整转发链路

### 1.1 读取操作的完整链路

读取操作的链路最为复杂，涉及多层抽象和转换。以下是从服务层到数据库执行的完整路径：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           服务层 (ItemsService)                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                      readByQuery 方法                                     │   │
│  │                                                                           │   │
│  │  1. 触发 filter 钩子 (emitter.emitFilter)                                │   │
│  │  2. 调用 getAstFromQuery → 构造 AST                                       │   │
│  │  3. 调用 processAst → 应用权限限制                                        │   │
│  │  4. 调用 runAst → 执行查询                                                │   │
│  │  5. 触发 filter 钩子 (emitter.emitFilter)                                │   │
│  │  6. 触发 action 钩子 (emitter.emitAction)                                │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        AST 构造层 (getAstFromQuery)                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  1. 创建根 AST 节点                                                       │   │
│  │     {                                                                     │   │
│  │       type: 'root',                                                       │   │
│  │       name: collection,                                                   │   │
│  │       query: options.query,                                               │   │
│  │       children: [],                                                        │   │
│  │       cases: [],                                                           │   │
│  │     }                                                                      │   │
│  │                                                                           │   │
│  │  2. 处理字段选择                                                          │   │
│  │     - 默认使用 ['*']                                                       │   │
│  │     - 聚合查询时清空 fields                                                │   │
│  │     - 分组查询时使用 group 作为 fields                                     │   │
│  │                                                                           │   │
│  │  3. 处理排序                                                              │   │
│  │     - 调用 getAllowedSort 获取默认排序                                    │   │
│  │     - 纯聚合查询时忽略排序                                                │   │
│  │                                                                           │   │
│  │  4. 调用 parseFields → 解析字段和关系                                     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       字段解析层 (parseFields)                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  1. 转换通配符 (convertWildcards)                                        │   │
│  │     - 将 '*' 展开为实际字段列表                                           │   │
│  │     - 考虑权限限制（非管理员用户）                                        │   │
│  │                                                                           │   │
│  │  2. 遍历每个字段，判断类型                                                │   │
│  │                                                                           │   │
│  │     【普通字段】                                                          │   │
│  │     - 非函数调用、非关系字段                                              │   │
│  │     - 创建 FieldNode: { type: 'field', name, fieldKey, whenCase: [] }  │   │
│  │                                                                           │   │
│  │     【函数字段】                                                          │   │
│  │     - 检测到 '(' 和 ')'                                                  │   │
│  │     - 特殊处理:                                                           │   │
│  │       * count(related_items): 关系计数                                   │   │
│  │       * json(...): JSON 函数                                             │   │
│  │     - 规范化: json(m2m.data, color) → m2m.json(data, color)            │   │
│  │     - 创建 FunctionFieldNode                                              │   │
│  │                                                                           │   │
│  │     【关系字段】                                                          │   │
│  │     - 包含 '.' 或是一对多别名字段                                         │   │
│  │     - 构建 relationalStructure 进行聚合处理                               │   │
│  │                                                                           │   │
│  │  3. 处理关系字段 (遍历 relationalStructure)                               │   │
│  │                                                                           │   │
│  │     【任意对一关系 (A2O)】                                               │   │
│  │     - 检测: relationType === 'a2o'                                       │   │
│  │     - 获取允许的集合列表 (one_allowed_collections)                       │   │
│  │     - 权限过滤: 非管理员用户只保留有权限的集合                            │   │
│  │     - 创建 A2ONode:                                                       │   │
│  │       {                                                                    │   │
│  │         type: 'a2o',                                                       │   │
│  │         names: allowedCollections,                                         │   │
│  │         children: {},                                                      │   │
│  │         query: {},                                                         │   │
│  │         relatedKey: {},                                                    │   │
│  │         parentKey: primaryKey,                                             │   │
│  │         fieldKey: fieldKey,                                                │   │
│  │         relation: relation,                                                │   │
│  │         cases: {},                                                         │   │
│  │         whenCase: [],                                                      │   │
│  │       }                                                                    │   │
│  │     - 递归解析每个允许集合的子字段                                          │   │
│  │                                                                           │   │
│  │     【多对一/一对多关系 (M2O/O2M)】                                       │   │
│  │     - 获取关联集合 (getRelatedCollection)                                 │   │
│  │     - 权限检查: 非管理员用户验证读取权限                                   │   │
│  │     - 创建 NestedCollectionNode:                                          │   │
│  │       {                                                                    │   │
│  │         type: relationType, // 'm2o' | 'o2m' | 'm2m'                    │   │
│  │         name: relatedCollection,                                           │   │
│  │         fieldKey: fieldKey,                                                │   │
│  │         parentKey: parentCollection.primary,                               │   │
│  │         relatedKey: relatedCollection.primary,                             │   │
│  │         relation: relation,                                                │   │
│  │         query: getDeepQuery(deep?.[fieldKey]),                            │   │
│  │         children: 递归解析子字段,                                          │   │
│  │         cases: [],                                                         │   │
│  │         whenCase: [],                                                      │   │
│  │       }                                                                    │   │
│  │                                                                           │   │
│  │     【一对多关系特殊处理】                                                 │   │
│  │     - 设置默认排序: getAllowedSort                                         │   │
│  │     - 分组查询特殊处理: group 需包含外键                                   │   │
│  │                                                                           │   │
│  │  4. 去重: 移除同时作为普通字段和关系字段的重复项                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        权限处理层 (processAst)                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  1. 提取字段映射 (fieldMapFromAst)                                        │   │
│  │     - 从 AST 中提取所有字段路径                                            │   │
│  │     - 分类: read（只读）和 other（其他操作）                              │   │
│  │                                                                           │   │
│  │  2. 管理员用户特殊处理                                                     │   │
│  │     - 只验证字段存在性 (validatePathExistence)                            │   │
│  │     - 跳过权限验证和条件注入                                               │   │
│  │     - 直接返回原始 AST                                                     │   │
│  │                                                                           │   │
│  │  3. 非管理员用户权限处理                                                  │   │
│  │                                                                           │   │
│  │     【获取权限】                                                          │   │
│  │     - 调用 fetchPolicies: 获取用户的策略列表                              │   │
│  │     - 调用 fetchPermissions: 根据策略获取具体权限                         │   │
│  │     - 如果不是读取操作，额外获取读取权限                                   │   │
│  │                                                                           │   │
│  │     【验证字段存在性】                                                    │   │
│  │     - 遍历 fieldMap.read 和 fieldMap.other                                │   │
│  │     - 调用 validatePathExistence                                          │   │
│  │                                                                           │   │
│  │     【验证字段权限】                                                      │   │
│  │     - 非读取字段: 使用当前操作的权限验证                                  │   │
│  │     - 只读字段: 使用读取权限验证                                          │   │
│  │     - 调用 validatePathPermissions                                         │   │
│  │                                                                           │   │
│  │     【注入权限条件】                                                      │   │
│  │     - 调用 injectCases(options.ast, permissions)                          │   │
│  │     - 将权限条件注入到 AST 的 cases 数组中                                │   │
│  │     - 每个条件对应一个权限规则的 filter                                    │   │
│  │     - 字段节点记录 whenCase 索引，用于后续查询构建                         │   │
│  │                                                                           │   │
│  │  4. 返回处理后的 AST                                                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       查询构建层 (getDBQuery)                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  【整体流程】                                                            │   │
│  │  1. 准备工作                                                            │   │
│  │     - 创建别名映射 (aliasMap)                                            │   │
│  │     - 获取列预处理器 (getColumnPreprocessor)                             │   │
│  │     - 克隆查询对象 (queryCopy)                                           │   │
│  │     - 检测是否有权限条件 (hasCaseWhen)                                   │   │
│  │     - 设置默认限制 (QUERY_LIMIT_DEFAULT)                                 │   │
│  │                                                                           │   │
│  │  2. 分支处理                                                            │   │
│  │                                                                           │   │
│  │     【分支 A: 聚合查询或分组查询】                                        │   │
│  │     - 检测: query.aggregate 或 query.group                               │   │
│  │                                                                           │   │
│  │     处理步骤:                                                             │   │
│  │     a. 构建字段节点映射                                                   │   │
│  │     b. 处理分组字段的 case/when                                           │   │
│  │     c. 计算聚合函数数量                                                   │   │
│  │     d. 计算分组列位置（用于 group by）                                    │   │
│  │     e. 调用 applyQuery 应用查询条件                                       │   │
│  │     f. 添加字段选择 (fieldNodes.map(preProcess))                         │   │
│  │     g. 返回查询构建器                                                     │   │
│  │                                                                           │   │
│  │     【分支 B: 普通查询】                                                  │   │
│  │     - 非聚合、非分组查询                                                  │   │
│  │                                                                           │   │
│  │     处理步骤:                                                             │   │
│  │     a. 获取主键字段 (primaryKey)                                         │   │
│  │     b. 创建基础查询: knex.from(table)                                    │   │
│  │     c. 处理排序 (如果有 query.sort)                                      │   │
│  │        - 调用 applySort                                                   │   │
│  │        - 记录排序记录 (sortRecords)                                       │   │
│  │        - 检测多关系排序 (hasMultiRelationalSort)                         │   │
│  │                                                                           │   │
│  │     d. 应用查询条件 (applyQuery)                                          │   │
│  │        - 应用 limit/offset/page                                           │   │
│  │        - 应用 filter (调用 applyFilter)                                   │   │
│  │        - 应用 group by                                                    │   │
│  │        - 应用 search                                                      │   │
│  │        - 应用 aggregate                                                   │   │
│  │        - 检测多关系过滤 (hasMultiRelationalFilter)                       │   │
│  │                                                                           │   │
│  │     e. 判断是否需要内部查询 (needsInnerQuery)                             │   │
│  │        - 条件: hasMultiRelationalSort 或 hasMultiRelationalFilter       │   │
│  │                                                                           │   │
│  │  3. 内部查询处理 (needsInnerQuery === true)                               │   │
│  │                                                                           │   │
│  │     【目的】                                                              │   │
│  │     - 多关系排序/过滤会导致重复行                                          │   │
│  │     - 需要先获取唯一主键，再关联查询完整数据                               │   │
│  │                                                                           │   │
│  │     【子分支 A: 无 case/when】                                           │   │
│  │     - 只选择主键: dbQuery.select(`${table}.${primaryKey}`)               │   │
│  │     - 添加 DISTINCT: dbQuery.distinct()                                  │   │
│  │                                                                           │   │
│  │     【子分支 B: 有 case/when】                                           │   │
│  │     - 选择预处理后的字段                                                  │   │
│  │     - 添加 O2M 字段的权限标志                                             │   │
│  │     - 使用 GROUP BY 代替 DISTINCT                                        │   │
│  │       * 原因: 某些数据库不支持某些数据类型的 DISTINCT                     │   │
│  │       * 技巧: 使用 COUNT + GROUP BY 实现类似效果                          │   │
│  │                                                                           │   │
│  │     【分组特殊处理】                                                      │   │
│  │     - 某些数据库要求 SELECT 的列都在 GROUP BY 中                          │   │
│  │     - 调用 helpers.schema.addInnerSortFieldsToGroupBy                    │   │
│  │     - 将排序列添加到 GROUP BY                                             │   │
│  │                                                                           │   │
│  │  4. 外部查询包装 (wrapperQuery)                                           │   │
│  │                                                                           │   │
│  │     【构建】                                                              │   │
│  │     - knex.from(table).innerJoin(innerQuery, primaryKey)                │   │
│  │                                                                           │   │
│  │     【子分支 A: 无 case/when】                                           │   │
│  │     - 直接选择预处理后的字段                                               │   │
│  │                                                                           │   │
│  │     【子分支 B: 有 case/when】                                           │   │
│  │     - 区分普通字段和带权限的字段                                          │   │
│  │     - 普通字段: 直接选择                                                  │   │
│  │     - 带权限字段: 使用内部查询的标志判断                                  │   │
│  │       * 内部查询: COUNT(CASE WHEN condition THEN 1 END) as flag         │   │
│  │       * 外部查询: CASE WHEN inner.flag > 0 THEN column END as alias     │   │
│  │                                                                           │   │
│  │     【排序处理】                                                          │   │
│  │     - 使用内部查询的排序列                                                 │   │
│  │     - 多关系排序特殊处理: WHERE inner.directus_row_number = 1           │   │
│  │                                                                           │   │
│  │  5. 返回最终查询构建器                                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         查询执行层 (runAst)                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  【整体流程】                                                            │   │
│  │  1. 克隆 AST (避免修改原始对象)                                           │   │
│  │  2. 获取数据库连接 (knex = options.knex \|\| getDatabase())             │   │
│  │                                                                           │   │
│  │  3. 分支处理                                                            │   │
│  │                                                                           │   │
│  │     【分支 A: 任意对一关系 (A2O)】                                       │   │
│  │     - 遍历所有允许的集合                                                  │   │
│  │     - 对每个集合单独执行查询                                              │   │
│  │     - 返回按集合分组的结果                                                │   │
│  │                                                                           │   │
│  │     【分支 B: 普通查询/其他关系】                                         │   │
│  │     - 调用内部 run 函数执行                                               │   │
│  │                                                                           │   │
│  │  【内部 run 函数详细流程】                                                │   │
│  │                                                                           │   │
│  │  步骤 1: 解析当前层级                                                     │   │
│  │          调用 parseCurrentLevel                                           │   │
│  │          - 提取字段节点 (fieldNodes)                                      │   │
│  │          - 提取主键字段 (primaryKeyField)                                 │   │
│  │          - 提取嵌套关系节点 (nestedCollectionNodes)                       │   │
│  │          - 分离一对多节点 (o2mNodes)                                     │   │
│  │                                                                           │   │
│  │  步骤 2: 获取权限（非管理员用户）                                         │   │
│  │          - 调用 fetchPolicies                                             │   │
│  │          - 调用 fetchPermissions (action: 'read')                         │   │
│  │                                                                           │   │
│  │  步骤 3: 构建数据库查询                                                   │   │
│  │          调用 getDBQuery                                                  │   │
│  │          参数: { table, fieldNodes, o2mNodes, query, cases, permissions }│   │
│  │                                                                           │   │
│  │  步骤 4: 执行查询获取原始数据                                             │   │
│  │          const rawItems: Item | Item[] = await dbQuery;                 │   │
│  │          - 这是 Knex 查询构建器的执行                                     │   │
│  │          - 返回数据库原始格式的数据                                       │   │
│  │                                                                           │   │
│  │  步骤 5: 空结果处理                                                       │   │
│  │          if (!rawItems) return null;                                      │   │
│  │                                                                           │   │
│  │  步骤 6: 数据类型转换                                                     │   │
│  │          创建 PayloadService 实例                                         │   │
│  │          构建别名映射 (aliasMap)                                          │   │
│  │          调用 payloadService.processValues('read', rawItems, aliasMap, aggregate)│   │
│  │                                                                           │   │
│  │          转换内容包括:                                                    │   │
│  │          - 日期/时间格式标准化                                            │   │
│  │          - JSON 数据解析                                                  │   │
│  │          - UUID 格式标准化                                                │   │
│  │          - 数值类型转换                                                   │   │
│  │          - 布尔值处理                                                     │   │
│  │                                                                           │   │
│  │  步骤 7: 空结果再次检查                                                   │   │
│  │          if (!items \|\| (Array.isArray(items) && items.length === 0))  │   │
│  │            return items;                                                  │   │
│  │                                                                           │   │
│  │  步骤 8: 处理嵌套关系                                                     │   │
│  │          调用 applyParentFilters                                          │   │
│  │          - 为嵌套关系添加父级过滤条件                                     │   │
│  │          - 使用外键关联父表                                               │   │
│  │                                                                           │   │
│  │          遍历每个嵌套节点:                                                │   │
│  │                                                                           │   │
│  │          【一对多关系 (O2M) 特殊处理】                                   │   │
│  │          - 支持批量查询 (RELATIONAL_BATCH_SIZE)                          │   │
│  │          - 目的: 避免 N+1 查询问题                                       │   │
│  │                                                                           │   │
│  │          权限标志处理:                                                    │   │
│  │          - 检测 hasWhenCase: 嵌套节点有权限条件                          │   │
│  │          - 提取标志值: fieldAllowed = item[nestedNode.fieldKey]          │   │
│  │          - 从结果中移除标志字段                                           │   │
│  │                                                                           │   │
│  │          批量查询循环:                                                    │   │
│  │          while (hasMore) {                                               │   │
│  │            - 构建批次查询 (limit, offset)                                │   │
│  │            - 递归调用 runAst 获取嵌套数据                                 │   │
│  │            - 合并嵌套数据到父项 (mergeWithParentItems)                   │   │
│  │            - 检查是否还有更多数据                                         │   │
│  │            - 批次计数 +1                                                  │   │
│  │          }                                                                │   │
│  │                                                                           │   │
│  │          【其他关系 (M2O/A2O) 处理】                                     │   │
│  │          - 单次查询获取所有嵌套数据                                       │   │
│  │          - limit: -1 (无限制)                                            │   │
│  │          - 递归调用 runAst                                                │   │
│  │          - 合并到父项                                                     │   │
│  │                                                                           │   │
│  │  步骤 9: 移除临时字段                                                     │   │
│  │          条件: options?.nested !== true && options?.stripNonRequested !== false│   │
│  │                                                                           │   │
│  │          移除内容包括:                                                    │   │
│  │          - 为嵌套关系添加的外键字段                                       │   │
│  │          - 权限检查标志字段                                               │   │
│  │          - 未在查询中请求的字段                                           │   │
│  │                                                                           │   │
│  │  步骤 10: 返回最终结果                                                    │   │
│  │                                                                           │   │
│  │  【特殊说明: 嵌套查询中的权限检查】                                       │   │
│  │  在处理一对多关系时，权限检查通过以下方式实现:                            │   │
│  │                                                                           │   │
│  │  1. 在 getDBQuery 中，为带权限的 O2M 字段添加标志列:                    │   │
│  │     dbQuery.select(                                                      │   │
│  │       o2mNodes                                                           │   │
│  │         .filter((node) => node.whenCase && node.whenCase.length > 0)   │   │
│  │         .map((node) => {                                                 │   │
│  │           return applyCaseWhen(                                          │   │
│  │             { column: knex.raw(1), ... },                               │   │
│  │             { knex, schema }                                             │   │
│  │           );                                                              │   │
│  │         })                                                                │   │
│  │     );                                                                    │   │
│  │                                                                           │   │
│  │  2. applyCaseWhen 生成类似:                                              │   │
│  │     CASE WHEN (permission_condition) THEN 1 END AS fieldKey             │   │
│  │                                                                           │   │
│  │  3. 在 runAst 中，根据标志决定是否显示嵌套数据:                           │   │
│  │     items = mergeWithParentItems(                                        │   │
│  │       schema, nestedItems, items!, nestedNode, fieldAllowed             │   │
│  │     );                                                                    │   │
│  │                                                                           │   │
│  │  4. mergeWithParentItems 中:                                             │   │
│  │     - 如果 fieldAllowed 为 false，嵌套字段设为 null                      │   │
│  │     - 如果 fieldAllowed 为数组，逐个检查每个父项                         │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          数据库层 (Knex.js)                                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │  【数据库连接管理】                                                      │   │
│  │                                                                           │   │
│  │  getDatabase() 函数:                                                     │   │
│  │  - 单例模式: 第一次调用时创建连接，后续复用                               │   │
│  │  - 从环境变量读取配置 (DB_* 前缀)                                        │   │
│  │  - 支持的数据库客户端:                                                    │   │
│  │    * sqlite3                                                              │   │
│  │    * mysql → 实际使用 mysql2                                             │   │
│  │    * pg (PostgreSQL)                                                     │   │
│  │    * cockroachdb                                                         │   │
│  │    * oracledb                                                            │   │
│  │    * mssql (MS SQL Server)                                               │   │
│  │                                                                           │   │
│  │  【连接池配置】                                                          │   │
│  │  - 可通过 DB_POOL_* 环境变量配置                                         │   │
│  │  - 某些数据库有特殊的 afterCreate 钩子:                                  │   │
│  │    * SQLite: 启用外键支持 (PRAGMA foreign_keys = ON)                    │   │
│  │    * CockroachDB: 设置序列化模式和整数大小                               │   │
│  │    * OracleDB: 设置日期格式                                              │   │
│  │    * MySQL: 调整时区处理                                                 │   │
│  │    * MSSQL: 禁用 UTC 转换                                                │   │
│  │                                                                           │   │
│  │  【查询执行】                                                            │   │
│  │                                                                           │   │
│  │  Knex 查询构建器执行流程:                                                │   │
│  │  1. 构建 SQL 语句                                                        │   │
│  │     - 将链式调用转换为 SQL 字符串                                         │   │
│  │     - 处理参数绑定 (避免 SQL 注入)                                       │   │
│  │     - 根据数据库方言生成不同语法                                          │   │
│  │                                                                           │   │
│  │  2. 获取连接                                                              │   │
│  │     - 从连接池获取可用连接                                                │   │
│  │     - 如果是事务，使用事务连接                                            │   │
│  │                                                                           │   │
│  │  3. 执行查询                                                              │   │
│  │     - 调用底层数据库驱动执行 SQL                                          │   │
│  │     - 返回原始结果集                                                      │   │
│  │                                                                           │   │
│  │  4. 处理结果                                                              │   │
│  │     - 将数据库结果转换为 JavaScript 对象                                  │   │
│  │     - 处理类型映射                                                        │   │
│  │     - 释放连接回连接池                                                    │   │
│  │                                                                           │   │
│  │  【性能监控】                                                            │   │
│  │  - 监听 query 事件: 记录开始时间                                          │   │
│  │  - 监听 query-response 事件: 计算执行时间                                │   │
│  │  - 发送指标到 metrics 系统                                                │   │
│  │  - 记录 trace 级别日志                                                   │   │
│  │                                                                           │   │
│  │  【事务支持】                                                            │   │
│  │  Knex 原生支持事务:                                                       │   │
│  │  - trx.commit(): 提交事务                                                │   │
│  │  - trx.rollback(): 回滚事务                                              │   │
│  │  - Directus 的 transaction() 函数封装了自动提交/回滚                     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 写入操作的完整链路

写入操作（创建、更新、删除）的链路相对简单，但同样需要经过多层处理：

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      服务层 (ItemsService - 写入操作)                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                           │   │
│  │  【通用流程 - 所有写入操作】                                              │   │
│  │                                                                           │   │
│  │  步骤 1: 初始化突变追踪器                                                 │   │
│  │          if (!opts.mutationTracker)                                      │   │
│  │            opts.mutationTracker = this.createMutationTracker();         │   │
│  │                                                                           │   │
│  │          作用: 限制批量操作的最大数量 (MAX_BATCH_MUTATION)               │   │
│  │                                                                           │   │
│  │  步骤 2: 触发 filter 钩子 (操作前)                                       │   │
│  │          emitter.emitFilter(                                             │   │
│  │            ['items.create', `${collection}.items.create`],              │   │
│  │            payload,                                                       │   │
│  │            { collection },                                                │   │
│  │            { database, schema, accountability }                          │   │
│  │          );                                                               │   │
│  │                                                                           │   │
│  │          钩子可以:                                                        │   │
│  │          - 修改输入数据                                                   │   │
│  │          - 验证数据                                                       │   │
│  │          - 抛出错误终止操作                                               │   │
│  │                                                                           │   │
│  │  步骤 3: 权限校验                                                        │   │
│  │                                                                           │   │
│  │          【更新/删除操作】                                                │   │
│  │          if (this.accountability) {                                      │   │
│  │            await validateAccess(                                         │   │
│  │              {                                                            │   │
│  │                accountability: this.accountability,                      │   │
│  │                action: 'update' | 'delete',                              │   │
│  │                collection: this.collection,                               │   │
│  │                primaryKeys: keys,                                         │   │
│  │                fields: Object.keys(payload),  // 仅更新                   │   │
│  │              },                                                           │   │
│  │              { schema, knex }                                             │   │
│  │            );                                                             │   │
│  │          }                                                                │   │
│  │                                                                           │   │
│  │          【创建/更新操作 - 数据处理】                                     │   │
│  │          const payloadWithPresets = this.accountability                  │   │
│  │            ? await processPayload(                                        │   │
│  │                {                                                          │   │
│  │                  accountability: this.accountability,                    │   │
│  │                  action: 'create' | 'update',                            │   │
│  │                  collection: this.collection,                             │   │
│  │                  payload: payloadAfterHooks,                              │   │
│  │                  nested: this.nested,                                      │   │
│  │                },                                                         │   │
│  │                { knex, schema }                                           │   │
│  │              )                                                            │   │
│  │            : payloadAfterHooks;                                           │   │
│  │                                                                           │   │
│  │          processPayload 作用:                                             │   │
│  │          - 验证字段权限                                                   │   │
│  │          - 应用预设值 (presets)                                           │   │
│  │          - 处理只读字段                                                   │   │
│  │          - 过滤无权访问的字段                                             │   │
│  │                                                                           │   │
│  │  步骤 4: 事务包装                                                        │   │
│  │          await transaction(this.knex, async (trx) => {                  │   │
│  │            // 所有数据库操作都在这个回调中执行                            │   │
│  │            // 如果抛出错误，自动回滚                                       │   │
│  │            // 如果正常返回，自动提交                                       │   │
│  │          });                                                              │   │
│  │                                                                           │   │
│  │  步骤 5: PayloadService 数据处理 (事务内)                                │   │
│  │          const payloadService = new PayloadService(                      │   │
│  │            this.collection,                                               │   │
│  │            { accountability, knex: trx, schema, nested, ... }           │   │
│  │          );                                                               │   │
│  │                                                                           │   │
│  │          【关系处理顺序】                                                 │   │
│  │          a. 处理多对一关系 (M2O)                                         │   │
│  │             payloadService.processM2O(payload, opts)                    │   │
│  │             - 查找或创建关联记录                                          │   │
│  │             - 设置外键值                                                  │   │
│  │                                                                           │   │
│  │          b. 处理任意对一关系 (A2O)                                       │   │
│  │             payloadService.processA2O(payload, opts)                    │   │
│  │             - 根据 item_type 确定集合                                     │   │
│  │             - 查找或创建关联记录                                          │   │
│  │             - 设置外键值                                                  │   │
│  │                                                                           │   │
│  │          c. 数据类型转换                                                  │   │
│  │             payloadService.processValues('create'|'update', payload)    │   │
│  │                                                                           │   │
│  │          d. 执行主表操作                                                  │   │
│  │             - 创建: trx.insert(payload).into(collection)                │   │
│  │             - 更新: trx(collection).update(payload).whereIn(...)        │   │
│  │             - 删除: trx(collection).whereIn(...).delete()               │   │
│  │                                                                           │   │
│  │          e. 处理一对多关系 (O2M) - 主表操作后                           │   │
│  │             payloadService.processO2M(payload, primaryKey, opts)        │   │
│  │             - 同步关联表数据                                              │   │
│  │             - 处理新增、更新、删除的关联记录                              │   │
│  │                                                                           │   │
│  │  步骤 6: 用户完整性验证 (可选)                                            │   │
│  │          if (userIntegrityCheckFlags) {                                  │   │
│  │            await validateUserCountIntegrity({ flags, knex: trx });      │   │
│  │          }                                                                │   │
│  │                                                                           │   │
│  │          作用: 验证用户相关的完整性约束                                   │   │
│  │                                                                           │   │
│  │  步骤 7: 问责制追踪 (可选)                                                │   │
│  │          if (opts.skipTracking !== true &&                               │   │
│  │              this.accountability &&                                       │   │
│  │              this.schema.collections[collection].accountability !== null)│   │
│  │          {                                                                │   │
│  │            // 创建活动记录 (ActivityService)                              │   │
│  │            // 创建修订记录 (RevisionsService) - 如果 accountability === 'all'│   │
│  │          }                                                                │   │
│  │                                                                           │   │
│  │  步骤 8: 事务结束 (自动提交或回滚)                                        │   │
│  │                                                                           │   │
│  │  步骤 9: 缓存清理 (事务后)                                                │   │
│  │          if (shouldClearCache(this.cache, opts, this.collection)) {      │   │
│  │            await this.cache.clear();                                      │   │
│  │          }                                                                │   │
│  │                                                                           │   │
│  │  步骤 10: 触发 action 钩子 (操作后)                                      │   │
│  │          emitter.emitAction(                                              │   │
│  │            ['items.create', `${collection}.items.create`],              │   │
│  │            { payload, key, collection },                                  │   │
│  │            { database, schema, accountability }                          │   │
│  │          );                                                               │   │
│  │                                                                           │   │
│  │  步骤 11: 返回结果                                                       │   │
│  │                                                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    事务管理层 (transaction 函数)                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                           │   │
│  │  Directus 的 transaction 函数封装了 Knex 的事务机制:                      │   │
│  │                                                                           │   │
│  │  export async function transaction<T>(                                   │   │
│  │    db: Knex,                                                              │   │
│  │    callback: (trx: Knex.Transaction) => Promise<T>                      │   │
│  │  ): Promise<T> {                                                          │   │
│  │    try {                                                                  │   │
│  │      return await db.transaction(async (trx) => {                        │   │
│  │        try {                                                              │   │
│  │          const result = await callback(trx);                              │   │
│  │          return result;                                                   │   │
│  │        } catch (error) {                                                  │   │
│  │          // 回滚已经由 Knex 处理                                          │   │
│  │          throw error;                                                     │   │
│  │        }                                                                  │   │
│  │      });                                                                  │   │
│  │    } catch (error) {                                                      │   │
│  │      // 转换数据库错误                                                    │   │
│  │      throw await translateDatabaseError(error);                          │   │
│  │    }                                                                      │   │
│  │  }                                                                        │   │
│  │                                                                           │   │
│  │  【关键点】                                                              │   │
│  │  1. 自动提交/回滚                                                        │   │
│  │     - callback 正常返回 → Knex 自动 commit                                │   │
│  │     - callback 抛出错误 → Knex 自动 rollback                              │   │
│  │                                                                           │   │
│  │  2. 错误转换                                                              │   │
│  │     - 捕获数据库特定错误                                                  │   │
│  │     - 转换为 Directus 标准错误                                            │   │
│  │     - 例如: 唯一约束冲突 → RecordNotUniqueError                          │   │
│  │                                                                           │   │
│  │  3. 服务实例分支                                                          │   │
│  │     - 在事务中需要使用 trx 作为 knex 实例                                 │   │
│  │     - ItemsService.fork({ knex: trx }) 创建分支实例                      │   │
│  │     - 确保嵌套操作使用同一个事务连接                                      │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────┬──────────────────────────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        数据库层 (Knex.js - 写入操作)                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                           │   │
│  │  【创建操作】                                                            │   │
│  │  trx.insert(payloadWithoutAliases)                                       │   │
│  │     .into(this.collection)                                                │   │
│  │     .returning(primaryKeyField, returningOptions)                        │   │
│  │     .then((result) => result[0]);                                         │   │
│  │                                                                           │   │
│  │  关键点:                                                                  │   │
│  │  - returning: 获取插入后的主键值                                          │   │
│  │  - MSSQL 特殊处理: includeTriggerModifications 支持触发器修改            │   │
│  │  - MySQL/SQLite: 不支持 returning，需额外处理                            │   │
│  │                                                                           │   │
│  │  【更新操作】                                                            │   │
│  │  trx(this.collection)                                                     │   │
│  │     .update(payloadWithTypeCasting)                                       │   │
│  │     .whereIn(primaryKeyField, keys);                                      │   │
│  │                                                                           │   │
│  │  关键点:                                                                  │   │
│  │  - 批量更新使用 whereIn                                                   │   │
│  │  - 主键列表已排序 (keys.sort())                                           │   │
│  │                                                                           │   │
│  │  【删除操作】                                                            │   │
│  │  trx(this.collection)                                                     │   │
│  │     .whereIn(primaryKeyField, keysAfterHooks)                             │   │
│  │     .delete();                                                            │   │
│  │                                                                           │   │
│  │  关键点:                                                                  │   │
│  │  - 使用 filter 钩子处理后的 keys                                          │   │
│  │  - 级联删除依赖数据库外键约束                                              │   │
│  │                                                                           │   │
│  │  【主键获取策略】                                                        │   │
│  │                                                                           │   │
│  │  创建操作后获取主键:                                                      │   │
│  │  1. 优先使用 returning 返回的值                                           │   │
│  │  2. 如果是 UUID 类型，格式化处理                                          │   │
│  │  3. 如果没有返回值 (MySQL/SQLite)，使用 max(primaryKey) 查询             │   │
│  │                                                                           │   │
│  │  特殊情况处理:                                                            │   │
│  │  - 手动提供主键值: 验证并使用                                             │   │
│  │  - 自增主键: 数据库自动生成                                               │   │
│  │  - UUID 主键: 可以手动提供或数据库生成                                    │   │
│  │                                                                           │   │
│  │  【自增序列重置】                                                        │   │
│  │  当手动提供整数主键时，可能需要重置自增序列:                              │   │
│  │  if (autoIncrementSequenceNeedsToBeReset) {                              │   │
│  │    await getHelpers(trx).sequence.resetAutoIncrementSequence(            │   │
│  │      this.collection, primaryKeyField                                     │   │
│  │    );                                                                     │   │
│  │  }                                                                        │   │
│  │                                                                           │   │
│  │  目的: 防止后续插入时主键冲突                                             │   │
│  │  适用场景: PostgreSQL, MySQL 等                                           │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### 1.3 关键代码位置索引

| 层级 | 功能 | 文件位置 |
|------|------|----------|
| **服务层** | 读取操作入口 | `api/src/services/items.ts:499` (`readByQuery`) |
| **服务层** | 创建操作入口 | `api/src/services/items.ts:127` (`createOne`) |
| **服务层** | 更新操作入口 | `api/src/services/items.ts:709` (`updateMany`) |
| **服务层** | 删除操作入口 | `api/src/services/items.ts:1070` (`deleteMany`) |
| **AST 构造** | 查询转 AST | `api/src/database/get-ast-from-query/get-ast-from-query.ts:23` |
| **字段解析** | 字段和关系解析 | `api/src/database/get-ast-from-query/lib/parse-fields.ts:32` |
| **权限处理** | AST 权限处理 | `api/src/permissions/modules/process-ast/process-ast.ts:19` |
| **查询构建** | AST 转 Knex 查询 | `api/src/database/run-ast/lib/get-db-query.ts:31` |
| **查询执行** | 执行 AST | `api/src/database/run-ast/run-ast.ts:20` |
| **事务管理** | 事务封装 | `api/src/utils/transaction.ts` |
| **数据库连接** | 连接管理 | `api/src/database/index.ts:35` (`getDatabase`) |

---

## 2. REST 与 GraphQL 接口的详细对比

### 2.1 请求入口与路由对比

| 维度 | REST API | GraphQL API | 设计取舍 |
|------|----------|-------------|----------|
| **路由数量** | 多路由，每个操作独立 | 单一入口 `/graphql` | REST: 符合 HTTP 语义，便于缓存和调试<br>GraphQL: 简化路由，所有操作通过查询语言表达 |
| **路由定义位置** | `api/src/controllers/items.ts` (主数据)<br>`api/src/controllers/*.ts` (其他资源) | `api/src/controllers/graphql.ts` | REST: 模块化，每个资源独立管理<br>GraphQL: 集中管理，便于扩展 |
| **HTTP 方法** | GET/POST/PATCH/DELETE 语义化 | GET/POST 两种 | REST: 利用 HTTP 方法表达操作意图<br>GraphQL: 方法用于区分查询方式，操作意图在查询中 |
| **系统数据路由** | `/users`、`/roles`、`/permissions` 等独立路由 | `/graphql/system` 统一入口 | REST: 系统数据与业务数据分离<br>GraphQL: 通过 scope 参数区分 |

**代码对比示例:**

```typescript
// REST - 多个独立路由
// api/src/controllers/items.ts
router.post('/:collection', collectionExists, asyncHandler(...), respond);
router.get('/:collection', collectionExists, readHandler, respond);
router.get('/:collection/:pk', collectionExists, asyncHandler(...), respond);
router.patch('/:collection', collectionExists, validateBatch('update'), asyncHandler(...), respond);
router.delete('/:collection/:pk', collectionExists, asyncHandler(...), respond);

// GraphQL - 单一入口
// api/src/controllers/graphql.ts
router.use('/system', parseGraphQL, asyncHandler(...), respond);
router.use('/', parseGraphQL, asyncHandler(...), respond);
```

### 2.2 参数解析机制对比

| 维度 | REST API | GraphQL API | 设计取舍 |
|------|----------|-------------|----------|
| **参数来源** | URL 路径 + 查询字符串 + 请求体 | 查询字符串 (GET) + 请求体 (POST) | REST: 利用 HTTP 标准机制<br>GraphQL: 统一封装在查询文档中 |
| **解析复杂度** | 简单，直接从 Express req 对象获取 | 复杂，需解析 GraphQL AST | REST: 无额外解析开销<br>GraphQL: 需额外解析步骤，但更灵活 |
| **类型验证** | 运行时验证 (sanitizeQuery) | 强类型验证 (GraphQL schema) | REST: 灵活但需要手动验证<br>GraphQL: 编译时即可发现错误 |
| **变量支持** | 无内置变量机制 | 完整的变量系统 (`$variable`) | REST: 需要手动处理参数替换<br>GraphQL: 内置变量，安全且复用性好 |

**代码对比示例:**

```typescript
// REST - 参数解析
// api/src/controllers/items.ts
router.get('/:collection/:pk', collectionExists, asyncHandler(async (req, res, next) => {
	// 从 URL 路径获取
	const collection = req.params['collection'];
	const pk = req.params['pk'];
	
	// 从查询字符串获取 (已通过中间件处理)
	const query = req.sanitizedQuery;  // 包含 fields, filter, sort, limit 等
	
	// 从请求体获取 (POST/PATCH)
	const body = req.body;
}));

// GraphQL - 参数解析
// api/src/middleware/graphql.ts
export const parseGraphQL: RequestHandler = asyncHandler(async (req, res, next) => {
	// GET 请求: 从查询字符串获取
	if (req.method === 'GET') {
		query = req.query['query'] as string;
		variables = parseJSON(req.query['variables'] as string);
		operationName = req.query['operationName'] as string;
	}
	// POST 请求: 从请求体获取
	else {
		query = req.body.query;
		variables = req.body.variables;
		operationName = req.body.operationName;
	}
	
	// 解析为 AST
	const document = parse(new Source(query), { maxTokens: ... });
	
	res.locals['graphqlParams'] = {
		document,
		query,
		variables,
		operationName,
		contextValue: { req, res, cache: new Map() },
	};
});
```

### 2.3 查询构造方式对比

| 维度 | REST API | GraphQL API | 设计取舍 |
|------|----------|-------------|----------|
| **查询格式** | Directus Query 对象 (JSON) | GraphQL 查询语言 (SDL) | REST: 与服务层直接兼容，无需转换<br>GraphQL: 更自然的查询语法，需转换 |
| **字段选择** | `?fields=id,name,author.*` | 嵌套字段选择 `{ id name author { id name } }` | REST: 字符串表达式，需要解析<br>GraphQL: 语法级支持，更直观 |
| **过滤条件** | `?filter={"status":{"_eq":"published"}}` | `filter: { status: { _eq: "published" } }` | REST: URL 编码的 JSON，可读性差<br>GraphQL: 内联在查询中，可读性好 |
| **嵌套关系** | `?deep={"author":{"_filter":{"active":{"_eq":true}}}}` | 嵌套查询 + 参数 `author(filter: { active: { _eq: true } }) { ... }` | REST: 复杂嵌套需多层 JSON<br>GraphQL: 语法级支持嵌套查询 |
| **聚合查询** | `?aggregate={"count":["id"]}` | `aggregated { count { id } }` | REST: 通用的聚合配置<br>GraphQL: 专门的查询类型 |

**代码对比示例:**

```typescript
// REST 查询构造 - 直接使用
// GET /articles?fields=id,title,author.*&filter={"status":{"_eq":"published"}}&sort=-date_created&limit=10

// 服务层直接接收
const result = await service.readByQuery(req.sanitizedQuery);
// req.sanitizedQuery = {
//   fields: ['id', 'title', 'author.*'],
//   filter: { status: { _eq: 'published' } },
//   sort: ['-date_created'],
//   limit: 10
// }

// GraphQL 查询构造 - 需要转换
// query {
//   articles(
//     filter: { status: { _eq: "published" } }
//     sort: ["-date_created"]
//     limit: 10
//   ) {
//     id
//     title
//     author {
//       id
//       name
//     }
//   }
// }

// 解析器中的转换
// api/src/services/graphql/schema/parse-query.ts
export async function getQuery(
	rawQuery: Query,
	schema: SchemaOverview,
	selections: readonly SelectionNode[],
	variableValues: GraphQLResolveInfo['variableValues'],
	accountability?: Accountability | null,
	collection?: string,
): Promise<Query> {
	// 1. 基础查询清理
	const query: Query = await sanitizeQuery(rawQuery, schema, accountability);
	
	// 2. 解析别名
	query.alias = parseAliases(selections);
	
	// 3. 解析字段选择 (递归处理嵌套)
	query.fields = await parseFields(selections, undefined, collection);
	
	// 4. 处理函数替换
	if (query.filter) query.filter = replaceFuncs(query.filter);
	query.deep = replaceFuncs(query.deep as any) as any;
	
	// 5. 处理 M2A 关系
	if (collection) {
		if (query.filter) {
			query.filter = filterReplaceM2A(query.filter, collection, schema, { aliasMap: query.alias });
		}
		query.deep = filterReplaceM2ADeep(query.deep, collection, schema, { aliasMap: query.alias });
	}
	
	// 6. 验证
	validateQuery(query);
	
	return query;
}
```

### 2.4 服务调用方式对比

| 维度 | REST API | GraphQL API | 设计取舍 |
|------|----------|-------------|----------|
| **服务实例化** | 控制器直接 `new ItemsService(...)` | 解析器通过 `getService(...)` 获取 | REST: 直接明了，控制器完全掌控<br>GraphQL: 解耦，便于测试和扩展 |
| **上下文传递** | `req.accountability`, `req.schema`, `this.knex` | `gql.accountability`, `gql.schema`, `gql.knex` | 两者一致，都是传递相同的上下文对象 |
| **方法调用** | 直接调用 `service.createOne/readByQuery/...` | 通过 `GraphQLService.read` 或直接调用 | REST: 一对一映射<br>GraphQL: 可能有额外封装 |
| **批量操作** | 控制器判断 `Array.isArray(req.body)` | 解析器判断字段名后缀 `_items`/`_item` | REST: HTTP 层面判断<br>GraphQL: 命名约定区分 |

**代码对比示例:**

```typescript
// REST - 服务调用
// api/src/controllers/items.ts
router.post('/:collection', collectionExists, asyncHandler(async (req, res, next) => {
	// 直接实例化服务
	const service = new ItemsService(req.collection, {
		accountability: req.accountability,
		schema: req.schema,
	});
	
	// 根据请求体类型选择方法
	let savedKeys: PrimaryKey[] = [];
	if (Array.isArray(req.body)) {
		const keys = await service.createMany(req.body);
		savedKeys.push(...keys);
	} else {
		const key = await service.createOne(req.body);
		savedKeys.push(key);
	}
	
	// 读取返回数据
	if (Array.isArray(req.body)) {
		const result = await service.readMany(savedKeys, req.sanitizedQuery);
		res.locals['payload'] = { data: result || null };
	} else {
		const result = await service.readOne(savedKeys[0]!, req.sanitizedQuery);
		res.locals['payload'] = { data: result || null };
	}
	
	return next();
}), respond);

// GraphQL - 服务调用
// api/src/services/graphql/resolvers/mutation.ts
export async function resolveMutation(
	gql: GraphQLService,
	args: Record<string, any>,
	info: GraphQLResolveInfo,
): Promise<Partial<Item> | boolean | undefined> {
	// 从字段名解析操作类型和集合
	const action = info.fieldName.split('_')[0] as 'create' | 'update' | 'delete';
	let collection = info.fieldName.substring(action.length + 1);
	
	// 命名约定判断操作模式
	const singleton = collection.endsWith('_batch') === false && 
	                   collection.endsWith('_items') === false &&
	                   collection.endsWith('_item') === false;
	const single = collection.endsWith('_items') === false && collection.endsWith('_batch') === false;
	const batchUpdate = action === 'update' && collection.endsWith('_batch');
	
	// 清理集合名后缀
	if (collection.endsWith('_batch')) collection = collection.slice(0, -6);
	if (collection.endsWith('_items')) collection = collection.slice(0, -6);
	if (collection.endsWith('_item')) collection = collection.slice(0, -5);
	
	// 通过 getService 获取服务实例
	const service = getService(collection, {
		knex: gql.knex,
		accountability: gql.accountability,
		schema: gql.schema,
	});
	
	// 根据操作模式调用相应方法
	if (single) {
		if (action === 'create') {
			const key = await service.createOne(args['data']);
			return hasQuery ? await service.readOne(key, query) : true;
		}
		// ... update, delete 类似
	} else {
		if (action === 'create') {
			const keys = await service.createMany(args['data']);
			return hasQuery ? await service.readMany(keys, query) : true;
		}
		// ... update, delete 类似
	}
}

// GraphQLService 中的封装
// api/src/services/graphql/index.ts
export class GraphQLService {
	// ...
	
	async read(collection: string, query: Query, id?: PrimaryKey): Promise<Partial<Item>> {
		const service = getService(collection, {
			knex: this.knex,
			accountability: this.accountability,
			schema: this.schema,
		});
		
		// 单例集合特殊处理
		if (this.schema.collections[collection]!.singleton)
			return await service.readSingleton(query, { stripNonRequested: false });
		
		// 单个项目查询
		if (id) return await service.readOne(id, query, { stripNonRequested: false });
		
		// 列表查询
		return await service.readByQuery(query, { stripNonRequested: false });
	}
}
```

### 2.5 响应格式对比

| 维度 | REST API | GraphQL API | 设计取舍 |
|------|----------|-------------|----------|
| **响应结构** | 固定 `{ data: ..., meta: ... }` | 灵活 `{ data: ..., errors: [...] }` | REST: 可预测，便于客户端处理<br>GraphQL: 符合规范，支持部分成功 |
| **元数据位置** | `meta` 顶层字段 | 无内置元数据，需通过查询获取 | REST: 便于分页、计数等<br>GraphQL: 更灵活但需要额外查询 |
| **错误处理** | HTTP 状态码 + `errors` 数组 | HTTP 200 + `errors` 数组 | REST: 利用 HTTP 语义，便于缓存判断<br>GraphQL: 支持部分成功，更细粒度错误 |
| **数据包装** | 始终 `data` 包装 | 直接是查询结果 | REST: 统一结构，便于扩展<br>GraphQL: 更简洁 |

**代码对比示例:**

```typescript
// REST 响应格式
// api/src/controllers/items.ts 中的 readHandler
const readHandler = asyncHandler(async (req, res, next) => {
	// ... 调用服务层 ...
	
	const result = await service.readByQuery(req.sanitizedQuery);
	const meta = await metaService.getMetaForQuery(req.collection, req.sanitizedQuery);
	
	// 固定格式
	res.locals['payload'] = {
		meta: meta,      // 包含 total_count, filter_count 等
		data: result,    // 实际数据
	};
	
	return next();
});

// 最终响应示例:
// {
//   "meta": {
//     "total_count": 100,
//     "filter_count": 25
//   },
//   "data": [
//     { "id": 1, "title": "Article 1" },
//     { "id": 2, "title": "Article 2" }
//   ]
// }

// GraphQL 响应格式
// api/src/services/graphql/index.ts
async execute({ document, variables, operationName, contextValue }: GraphQLParams) {
	// ... 执行查询 ...
	
	let result: ExecutionResult;
	try {
		result = await execute({
			schema,
			document,
			contextValue,
			variableValues: variables,
			operationName,
		});
	} catch (err: any) {
		throw new GraphQLExecutionError({ errors: [err.message] });
	}
	
	// 构建响应
	const formattedResult: FormattedExecutionResult = {};
	
	if (result['data']) formattedResult.data = result['data'];
	
	if (result['errors']) {
		formattedResult.errors = result['errors'].map((error) => 
			processError(this.accountability, error)
		);
	}
	
	if (result['extensions']) formattedResult.extensions = result['extensions'];
	
	return formattedResult;
}

// 最终响应示例:
// {
//   "data": {
//     "articles": [
//       { "id": 1, "title": "Article 1" },
//       { "id": 2, "title": "Article 2" }
//     ]
//   },
//   "errors": [  // 可选，部分成功时存在
//     {
//       "message": "Cannot query field \"unknown\" on type \"Article\".",
//       "locations": [{ "line": 3, "column": 5 }],
//       "path": ["articles", 0, "unknown"]
//     }
//   ]
// }
```

### 2.6 错误处理机制对比

| 维度 | REST API | GraphQL API | 设计取舍 |
|------|----------|-------------|----------|
| **HTTP 状态码** | 语义化使用 (400, 403, 404, 500 等) | 通常 200，仅在解析失败时非 200 | REST: 利用 HTTP 标准，便于代理/缓存处理<br>GraphQL: 传输层与应用层分离 |
| **错误格式** | `{ errors: [{ message, extensions: { code, ... } }] }` | `{ errors: [{ message, locations, path, extensions }] }` | 基本一致，GraphQL 额外包含位置和路径 |
| **错误分类** | 通过 `extensions.code` 区分 | 相同方式 + 内置位置信息 | REST: 简洁<br>GraphQL: 更便于调试 |
| **部分成功** | 不支持，全有或全无 | 支持，部分数据可返回 | REST: 简单一致<br>GraphQL: 更灵活，复杂场景有用 |

**错误代码对比:**

| 错误类型 | REST HTTP 状态码 | GraphQL extensions.code |
|----------|------------------|-------------------------|
| 未找到 | 404 | `NOT_FOUND` |
| 无权限 | 403 | `FORBIDDEN` |
| 未认证 | 401 | `UNAUTHORIZED` |
| 无效请求 | 400 | `INVALID_PAYLOAD` / `INVALID_QUERY` |
| 唯一约束 | 400 | `RECORD_NOT_UNIQUE` |
| 服务器错误 | 500 | `INTERNAL_SERVER_ERROR` |

### 2.7 嵌套关系处理对比

| 维度 | REST API | GraphQL API | 设计取舍 |
|------|----------|-------------|----------|
| **语法方式** | `fields=author.*` + `deep` 参数 | 嵌套字段选择 | REST: 字符串表达式<br>GraphQL: 语法级支持 |
| **嵌套过滤** | `?deep={"author":{"_filter":{"active":{"_eq":true}}}}` | `author(filter: { active: { _eq: true } }) { ... }` | REST: 复杂的 JSON 结构<br>GraphQL: 每个层级独立参数 |
| **嵌套排序** | `?deep={"author":{"_sort":"name"}}` | `author(sort: ["name"]) { ... }` | 同上 |
| **嵌套分页** | `?deep={"comments":{"_limit":10,"_page":1}}` | `comments(limit: 10, page: 1) { ... }` | 同上 |
| **多态关系 (A2O)** | `fields=sections.section_id:headings.title` | 内联片段 `... on Headings { title }` | REST: 特殊语法，不够直观<br>GraphQL: 类型系统原生支持 |

**代码对比示例:**

```typescript
// REST - 嵌套关系处理
// GET /articles?fields=id,title,author.*,comments.*&deep={
//   "author": { "_filter": { "active": { "_eq": true } } },
//   "comments": { "_limit": 10, "_sort": "-date_created" }
// }

// 服务层接收的查询结构:
// {
//   fields: ['id', 'title', 'author.*', 'comments.*'],
//   deep: {
//     author: { _filter: { active: { _eq: true } } },
//     comments: { _limit: 10, _sort: ['-date_created'] }
//   }
// }

// parseFields 中的处理:
// api/src/database/get-ast-from-query/lib/parse-fields.ts
// 关系字段检测
const isRelational =
	(!isFunctionCall || isRelationalFunctionCall) &&
	(name.includes('.') || isO2MAliasField);

// 构建关系结构
if (isRelational) {
	const parts = (isFunctionCall ? name : fieldKey).split('.');
	let rootField = parts[0]!;
	
	// 处理 A2O 集合限定符: sections.section_id:headings.title
	if (rootField.includes(':')) {
		const [key, scope] = rootField.split(':');
		rootField = key!;
		collectionScope = scope!;
	}
	
	// 构建关系结构
	if (rootField in relationalStructure === false) {
		if (collectionScope) {
			relationalStructure[rootField] = { [collectionScope]: [] };
		} else {
			relationalStructure[rootField] = [];
		}
	}
	
	// 处理子字段
	if (parts.length > 1) {
		const childKey = parts.slice(1).join('.');
		// 添加到关系结构中...
	}
}
```

### 2.8 设计取舍总结

#### 2.8.1 REST API 的设计优势与取舍

| 优势 | 取舍 |
|------|------|
| **符合 HTTP 语义** | 操作意图受限于 HTTP 方法 |
| **天然支持缓存** | POST/PATCH 无法有效缓存 |
| **简单直观** | 复杂查询需要多个请求或复杂参数 |
| **浏览器原生支持** | 需要手动处理嵌套关系 |
| **调试友好** | 响应格式固定，不够灵活 |

**REST API 的适用场景:**
- 简单的 CRUD 操作
- 需要利用 HTTP 缓存的场景
- 浏览器直接访问的场景
- 团队不熟悉 GraphQL 的情况

#### 2.8.2 GraphQL API 的设计优势与取舍

| 优势 | 取舍 |
|------|------|
| **精确获取数据** | 需要额外的查询解析开销 |
| **单一请求获取多资源** | 可能导致 N+1 问题（Directus 通过批量查询解决） |
| **强类型系统** | 学习曲线较陡 |
| **内置文档** | 需要维护 Schema |
| **灵活的嵌套查询** | 查询复杂度控制困难 |
| **支持部分成功** | 错误处理相对复杂 |

**GraphQL API 的适用场景:**
- 复杂的关系数据查询
- 需要精确控制返回字段的场景
- 前端频繁变更的场景
- 需要类型安全的场景

#### 2.8.3 关键设计决策分析

**决策 1: 统一服务层，接口层只做协议转换**

```
原因分析:
├── 避免重复代码
│   ├── 权限校验逻辑统一
│   ├── 业务逻辑统一
│   ├── 数据库交互统一
│   └── 事件钩子统一
│
├── 确保行为一致性
│   ├── REST 和 GraphQL 返回相同数据结构
│   ├── 权限校验规则一致
│   └── 验证逻辑一致
│
└── 便于维护和扩展
    ├── 新增功能只需修改服务层
    ├── 新增接口协议只需添加接口层
    └── 测试只需针对服务层
```

**决策 2: AST 作为中间表示**

```
原因分析:
├── 解耦查询格式与数据库执行
│   ├── REST 查询 → AST
│   ├── GraphQL 查询 → AST
│   └── AST → 数据库查询
│
├── 便于权限注入
│   ├── 权限条件直接注入 AST
│   ├── 不依赖原始查询格式
│   └── 灵活的字段级权限控制
│
└── 支持复杂的查询优化
    ├── 多关系排序/过滤的内部查询优化
    ├── case/when 权限条件的优化
    └── 批量查询优化
```

**决策 3: 不同的路由策略**

```
REST: 多路由，语义化
├── 优点:
│   ├── 符合 RESTful 设计原则
│   ├── 便于理解和调试
│   └── 天然支持 HTTP 缓存
│
└── 缺点:
    ├── 路由数量多
    ├── 复杂操作需要多个请求
    └── 嵌套关系需要特殊参数

GraphQL: 单一路由，查询语言
├── 优点:
│   ├── 单一入口，管理简单
│   ├── 灵活的查询能力
│   └── 类型系统支持
│
└── 缺点:
    ├── 失去 HTTP 语义
    ├── 需要额外的解析步骤
    └── 缓存策略复杂
```

**决策 4: 不同的错误处理策略**

```
REST: HTTP 状态码 + errors 数组
├── 优点:
│   ├── 利用 HTTP 标准语义
│   ├── 代理/缓存可以根据状态码处理
│   └── 简单直观
│
└── 缺点:
    ├── 状态码表达能力有限
    ├── 无法支持部分成功
    └── 错误详情需要额外字段

GraphQL: HTTP 200 + errors 数组
├── 优点:
│   ├── 支持部分成功
│   ├── 错误可以关联到具体字段
│   └── 包含位置信息便于调试
│
└── 缺点:
    ├── 无法利用 HTTP 缓存
    ├── 错误处理逻辑复杂
    └── 中间件难以识别错误
```

---

## 3. 总结与建议

### 3.1 架构设计的核心洞察

Directus 的数据读写架构体现了以下核心设计原则：

1. **关注点分离**
   - 接口层只负责协议转换
   - 服务层负责业务逻辑和权限校验
   - 数据库层负责数据存储和查询

2. **统一抽象**
   - AST 作为查询的中间表示
   - 统一的权限校验接口
   - 统一的事件钩子机制

3. **灵活性与一致性的平衡**
   - 支持多种接口协议（REST、GraphQL）
   - 确保所有接口的行为一致
   - 允许在协议层面做适当的特殊处理

### 3.2 两种接口的选择建议

**选择 REST API 的情况:**
- 项目简单，主要是 CRUD 操作
- 需要利用 HTTP 缓存机制
- 团队对 REST 更熟悉
- 需要与第三方系统集成（很多系统只支持 REST）

**选择 GraphQL API 的情况:**
- 数据关系复杂，需要多层嵌套查询
- 前端需要精确控制返回字段
- 需要类型安全和自动文档
- 团队熟悉 GraphQL 或愿意学习

**混合使用的情况:**
- 简单操作使用 REST（如列表查询、创建项目）
- 复杂查询使用 GraphQL（如多关系数据获取）
- 根据实际需求灵活选择

### 3.3 扩展开发建议

如果需要在 Directus 基础上进行扩展，建议：

1. **优先使用服务层 API**
   - 直接调用 `ItemsService` 等服务
   - 避免绕过服务层直接操作数据库
   - 确保权限校验和事件钩子正常触发

2. **使用事件钩子**
   - 使用 filter 钩子修改数据或查询
   - 使用 action 钩子执行副作用
   - 避免直接修改核心代码

3. **创建自定义端点**
   - 对于 REST，创建新的控制器
   - 对于 GraphQL，扩展 schema 和 resolver
   - 始终通过服务层进行数据操作

---

## 附录

### 附录 A: 关键文件索引

| 文件路径 | 功能描述 |
|----------|----------|
| `api/src/services/items.ts` | 核心数据服务，所有 CRUD 操作入口 |
| `api/src/database/get-ast-from-query/` | 查询转 AST 模块 |
| `api/src/database/run-ast/` | AST 执行模块 |
| `api/src/permissions/` | 权限校验模块 |
| `api/src/controllers/items.ts` | REST API 主控制器 |
| `api/src/controllers/graphql.ts` | GraphQL API 控制器 |
| `api/src/services/graphql/` | GraphQL 服务和解析器 |
| `api/src/database/index.ts` | 数据库连接管理 |

### 附录 B: 核心数据结构

**Directus Query 对象:**
```typescript
interface Query {
	fields?: string[];           // 字段选择
	filter?: Filter;             // 过滤条件
	sort?: string[];             // 排序
	limit?: number;              // 限制数量
	offset?: number;             // 偏移量
	page?: number;               // 页码
	search?: string;             // 搜索关键词
	aggregate?: Aggregate;       // 聚合函数
	group?: string[];            // 分组
	deep?: DeepQuery;            // 嵌套关系查询
	alias?: Record<string, string>; // 字段别名
}
```

**AST 节点类型:**
```typescript
interface AST {
	type: 'root' | 'a2o';
	name?: string;              // 集合名
	names?: string[];           // A2O 允许的集合名列表
	query: Query;               // 查询对象
	children: ASTChild[];       // 子节点（字段或关系）
	cases: Filter[];            // 权限条件
	whenCase?: number[];        // 字段级权限索引
}

type ASTChild = FieldNode | FunctionFieldNode | NestedCollectionNode;

interface FieldNode {
	type: 'field';
	name: string;
	fieldKey: string;
	whenCase: number[];
	alias?: boolean;
}

interface NestedCollectionNode {
	type: 'm2o' | 'o2m' | 'm2m' | 'a2o';
	name: string;
	fieldKey: string;
	query: Query;
	children: ASTChild[];
	cases: Filter[];
	whenCase: number[];
	// ... 关系相关字段
}
```

### 附录 C: 执行流程图

**读取操作完整流程:**
```
客户端请求
    ↓
接口层 (REST/GraphQL)
    ↓ 参数解析/查询转换
服务层 (ItemsService.readByQuery)
    ↓ filter 钩子
getAstFromQuery (构建 AST)
    ↓ 字段解析/关系处理
processAst (权限处理)
    ↓ 权限验证/条件注入
getDBQuery (构建 Knex 查询)
    ↓ 查询优化/条件应用
Knex 执行
    ↓ 数据库查询
PayloadService.processValues (类型转换)
    ↓ 数据格式化
runAst 嵌套关系处理
    ↓ 批量查询/数据合并
服务层 filter/action 钩子
    ↓
接口层响应格式化
    ↓
客户端响应
```

**写入操作完整流程:**
```
客户端请求
    ↓
接口层 (REST/GraphQL)
    ↓ 参数解析
服务层
    ↓ filter 钩子
    ↓ validateAccess (权限校验)
    ↓ processPayload (数据处理)
    ↓ 事务开始
    ↓ PayloadService 关系处理
    ↓ Knex 执行 (insert/update/delete)
    ↓ 用户完整性验证
    ↓ 问责制追踪 (Activity/Revisions)
    ↓ 事务提交/回滚
    ↓ 缓存清理
    ↓ action 钩子
    ↓
接口层响应格式化
    ↓
客户端响应
```
