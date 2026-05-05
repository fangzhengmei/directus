# Directus 多语言字段跨层一致性实现机制

本文档详细描述 Directus 多语言字段在 Admin 语言切换器、API 请求参数、服务端降级链上的一致性实现，以及缺失翻译在前端展示和 API 响应中的处理方式。

---

## 一、多语言字段架构概述

### 1.1 数据模型设计

Directus 的多语言字段本质上是一个**特殊的 M2M（多对多）关系**，通过以下三个表实现：

| 表名 | 作用 | 关键字段 |
|------|------|----------|
| `{主表}` | 存储原始数据 | `id` (主键) |
| `{主表}_translations` | 翻译关联表（中间表） | `id`, `{主表}_id`, `{语言表}_code`, `{翻译字段1}`, `{翻译字段2}`, ... |
| `languages` | 语言配置表 | `code` (主键), `name`, `direction` |

### 1.2 关系配置示例

以 `articles` 表的翻译字段为例，关系配置如下：

```typescript
// 主表字段定义
{
  field: 'translations',
  type: 'alias',
  special: ['translations'],  // 关键标记
  meta: {
    interface: 'translations',
    options: {
      languageField: 'name',
      languageDirectionField: 'direction'
    }
  }
}

// 关系定义
// 关系1: articles_translations -> articles
{
  collection: 'articles_translations',
  field: 'articles_id',
  related_collection: 'articles',
  meta: {
    one_field: 'translations',
    junction_field: 'languages_code'  // 指向语言表的外键
  }
}

// 关系2: articles_translations -> languages
{
  collection: 'articles_translations',
  field: 'languages_code',
  related_collection: 'languages',
  meta: {
    one_field: null,
    junction_field: 'articles_id'  // 指向主表的外键
  }
}
```

---

## 二、Admin 语言切换器实现

### 2.1 语言偏好管理

**文件位置**: `app/src/stores/user.ts`

```typescript
// 语言优先级链
const language = computed(() => {
  const user = unref(currentUser);

  // 1. 优先使用用户个人设置的语言
  if (user && 'language' in user && user.language !== null) {
    return user.language;
  }

  // 2. 回退到项目默认语言
  if (serverStore.info?.project?.default_language) {
    return serverStore.info.project.default_language;
  }

  // 3. 最终回退到 'en-US'
  return 'en-US';
});

// 监听语言变化并切换
watch(language, (newLang, oldLang) => {
  if (newLang && newLang !== oldLang) {
    setLanguage(newLang);
  }
});
```

### 2.2 语言切换核心逻辑

**文件位置**: `app/src/lang/set-language.ts`

```typescript
export async function setLanguage(lang: Language): Promise<boolean> {
  const collectionsStore = useCollectionsStore();
  const fieldsStore = useFieldsStore();
  const translationsStore = useTranslationsStore();

  // 1. 检查语言是否可用
  if (Object.keys(availableLanguages).includes(lang) === false) {
    console.warn(`"${lang}" is not an available language in the Directus app.`);
  } else {
    // 2. 动态加载语言包（如未加载）
    if (loadedLanguages.includes(lang) === false) {
      try {
        const { default: translations } = await import(`./translations/${lang}.yaml`);
        i18n.global.mergeLocaleMessage(lang, translations);
        loadedLanguages.push(lang);
      } catch (err: any) {
        console.warn(err);
      }
    }

    // 3. 切换 Vue I18n 语言
    i18n.global.locale.value = lang;
    (document.querySelector('html') as HTMLElement).setAttribute('lang', lang);
  }

  // 4. 加载数据翻译（用户自定义翻译）
  try {
    await translationsStore.loadTranslations(lang);
    
    // 5. 触发集合和字段的翻译
    collectionsStore.translateCollections();
    fieldsStore.translateFields();

    // 6. 加载日期本地化
    await loadDateFNSLocale(lang);
  } catch {
    console.error('Failed loading translations');
  }

  return true;
}
```

### 2.3 用户自定义翻译存储

**文件位置**: `app/src/stores/translations.ts`

```typescript
export const useTranslationsStore = defineStore('translations', () => {
  const translations = ref<Translation[]>([]);
  const lang = ref<string>('en-US');

  // 加载当前语言的自定义翻译
  const loadTranslations = async (newLang = unref(lang)) => {
    try {
      // 从 API 获取翻译数据
      translations.value = await fetchAll(`/translations`, {
        params: {
          fields: ['language', 'key', 'value'],
          filter: {
            language: { _eq: newLang },  // 按当前语言过滤
          },
        },
      });
      lang.value = newLang;
    } catch {
      // 无权限访问翻译表时静默失败
    }
  };

  // 翻译变更时合并到 Vue I18n
  watch(translations, (newTranslations) => {
    const localeMessages = newTranslations?.reduce(
      (result: Record<string, string>, { key, value }) => {
        result[key] = getLiteralInterpolatedTranslation(value, true);
        return result;
      },
      {} as Record<string, string>,
    );

    if (localeMessages) {
      i18n.global.mergeLocaleMessage(unref(lang), localeMessages);
    }
  });

  return { loading, translations, loadTranslations, create };
});
```

### 2.4 翻译字段界面组件

**文件位置**: `app/src/interfaces/translations/translations.vue`

翻译字段界面具有以下特性：

1. **语言选择器**: 显示所有可用语言及其翻译进度
2. **分屏视图**: 支持同时编辑两种语言的翻译
3. **进度指示**: 显示每种语言已完成翻译的字段百分比

```typescript
// 语言选项构建
const languageOptions = computed(() => {
  const langField = relationInfo.value?.junctionField.field;

  if (!langField) return [];

  const writableFields = fields.value.filter(
    (field) => field.type !== 'alias' && field.meta?.hidden === false && field.meta.readonly === false,
  );

  const totalFields = writableFields.length;

  return languages.value.map((language) => {
    const info = relationInfo.value;
    if (!info) return language;

    const langCode = language[info.relatedPrimaryKeyField.field];
    const edits = getItemWithLang(displayItems.value, langCode);
    const filledFields = writableFields.filter((field) => !isNil((edits ?? {})[field.field])).length;

    return {
      text: language[props.languageField ?? relationInfo.value.relatedPrimaryKeyField.field],
      direction: props.languageDirectionField ? language[props.languageDirectionField] : undefined,
      value: langCode,
      edited: edits?.$type !== undefined,
      progress: Math.round((filledFields / totalFields) * 100),  // 翻译进度
      max: totalFields,
      current: filledFields,
    };
  });
});

// 默认语言选择逻辑
const defaultLang = computed(() => {
  const pkField = relationInfo.value?.relatedPrimaryKeyField.field ?? '';

  // 优先使用用户语言或配置的默认语言
  const userLocale = props.userLanguage ? locale.value : props.defaultLanguage;
  const firstDefault = getDefaultLang(userLocale);
  const firstLang = firstDefault?.[pkField] as string;

  function getDefaultLang(defaultLocale: string | null) {
    // 查找匹配语言，否则返回第一个语言
    return languages.value.find((lang) => lang[pkField] === defaultLocale) || languages.value[0];
  }
});
```

---

## 三、API 请求参数与服务端处理

### 3.1 翻译字段的 API 访问方式

翻译字段通过标准的 M2M 关系 API 访问，主要使用以下参数：

| 参数 | 作用 | 示例 |
|------|------|------|
| `fields` | 选择要返回的字段，支持嵌套字段 | `fields=id,translations.*,translations.languages_code.code` |
| `deep` | 控制嵌套关系的查询行为（过滤、排序等） | `deep={ "translations": { "_filter": { ... } } }` |
| `filter` | 主级过滤条件 | `filter={ "id": { "_eq": 1 } }` |

### 3.2 前端查询构建逻辑

**文件位置**: `app/src/composables/use-relation-multiple.ts`

翻译字段作为 M2M 关系，其查询构建逻辑如下：

```typescript
// M2M 关系的字段请求
case 'm2m':
  targetCollection = relation.value.junctionCollection.collection;
  fields.add(relation.value.junctionPrimaryKeyField.field);
  // 请求语言关联信息
  fields.add(`${relation.value.junctionField.field}.${relation.value.relatedPrimaryKeyField.field}`);
  break;

// 过滤条件构建
const filter: Filter = {
  _and: [{ [reverseJunctionField]: itemId.value === null ? { _null: true } : itemId.value } as Filter],
};

if (previewQuery.value.filter) {
  filter._and.push(previewQuery.value.filter);
}

// 发送请求
const response = await sdk.request<Item[]>(
  requestEndpoint(getEndpoint(targetCollection), {
    params: {
      search: previewQuery.value.search,
      fields: Array.from(fields),
      filter,
      page: previewQuery.value.page,
      limit: previewQuery.value.limit,
      sort: previewQuery.value.sort,
    },
  }),
);
```

### 3.3 服务端查询解析

**文件位置**: `api/src/database/get-ast-from-query/get-ast-from-query.ts`

```typescript
export async function getAstFromQuery(options: GetAstFromQueryOptions, context: GetAstFromQueryContext): Promise<AST> {
  options.query = cloneDeep(options.query);

  const ast: AST = {
    type: 'root',
    name: options.collection,
    query: options.query,
    children: [],
    cases: [],
  };

  let fields = ['*'];
  if (options.query.fields) {
    fields = options.query.fields;
  }

  const deep = options.query.deep || {};

  // 从 query 中移除 fields 和 deep，避免重复处理
  delete options.query.fields;
  delete options.query.deep;

  // 解析字段（包括关系字段）
  ast.children = await parseFields(
    {
      parentCollection: options.collection,
      fields,
      query: options.query,
      deep,
      accountability: options.accountability,
    },
    context,
  );

  return ast;
}
```

### 3.4 关系字段解析

**文件位置**: `api/src/database/get-ast-from-query/lib/parse-fields.ts`

```typescript
// 检测是否为关系字段
const isRelational =
  (!isFunctionCall || isRelationalFunctionCall) &&
  (name.includes('.') ||
    // 检测 O2M 关系（如翻译字段）
    !!context.schema.relations.find(
      (relation) => relation.related_collection === options.parentCollection && relation.meta?.one_field === name,
    ));

if (isRelational) {
  // 构建关系查询结构
  const parts = (isFunctionCall ? name : fieldKey).split('.');
  let rootField = parts[0]!;

  if (rootField in relationalStructure === false) {
    if (collectionScope) {
      relationalStructure[rootField] = { [collectionScope]: [] };
    } else {
      relationalStructure[rootField] = [];
    }
  }

  // 处理嵌套字段
  if (parts.length > 1) {
    const childKey = parts.slice(1).join('.');
    if (collectionScope) {
      (relationalStructure[rootField] as CollectionScope)[collectionScope]!.push(childKey);
    } else {
      (relationalStructure[rootField] as string[]).push(childKey);
    }
  }
}
```

### 3.5 Deep 参数解析

**文件位置**: `api/src/database/get-ast-from-query/utils/get-deep-query.ts`

```typescript
/**
 * 从对象中提取深层查询参数
 * 提取以 _ 开头的键值对，用于嵌套关系的查询控制
 */
export function getDeepQuery(object: Record<string, any>): Record<string, any> {
  const result: Record<string, any> = {};

  for (const [key, value] of Object.entries(object)) {
    // 只提取以 _ 开头的键（如 _filter, _sort, _limit 等）
    if (key.startsWith('_')) {
      const actualKey = key.substring(1);
      result[actualKey] = value;
    }
  }

  return result;
}
```

---

## 四、服务端降级链与一致性保证

### 4.1 翻译字段的特殊标记

**文件位置**: `api/src/utils/generate-translations.ts`

翻译字段通过 `special: ['translations']` 标记，这是服务端识别多语言字段的关键：

```typescript
// 构建翻译字段配置
function buildTranslationsAliasField(
  languagesFields: Record<string, unknown>,
): Partial<Field> & { field: string; type: Type | null } {
  return {
    field: 'translations',
    type: 'alias',
    meta: {
      interface: 'translations',
      special: ['translations'],  // 关键标记
      options: {
        languageField: languagesFields['name'] ? 'name' : null,
        languageDirectionField: languagesFields['direction'] ? 'direction' : null,
      },
    } as unknown as FieldMeta,
  };
}
```

### 4.2 语言配置表的默认创建

当首次为集合生成翻译字段时，会自动创建 `languages` 表并填充默认语言：

```typescript
// 种子语言数据
if (seedLanguages) {
  const itemsService = new ItemsService(languagesCollection, getServiceOptions(currentSchema));

  await itemsService.createMany(
    [
      { code: 'en-US', name: 'English', direction: 'ltr' },
      { code: 'ar-SA', name: 'Arabic', direction: 'rtl' },
      { code: 'de-DE', name: 'German', direction: 'ltr' },
      { code: 'fr-FR', name: 'French', direction: 'ltr' },
      { code: 'ru-RU', name: 'Russian', direction: 'ltr' },
      { code: 'es-ES', name: 'Spanish', direction: 'ltr' },
      { code: 'it-IT', name: 'Italian', direction: 'ltr' },
      { code: 'pt-BR', name: 'Portuguese', direction: 'ltr' },
    ],
    mutationOptions,
  );
}
```

### 4.3 跨层一致性保证机制

Directus 通过以下机制保证各层语言一致性：

| 层级 | 一致性保证 | 实现方式 |
|------|-----------|----------|
| **Admin UI** | 语言偏好持久化 | 用户设置存储在 `directus_users.language` 字段 |
| **API 请求** | 无内置语言参数 | API 不提供全局 `lang` 参数，由客户端控制 |
| **数据层** | 翻译表独立存储 | 每种语言的翻译存储为独立记录 |

**重要设计决策**: Directus API 不提供全局的 `?lang=xx` 查询参数来过滤翻译。翻译数据始终作为完整的 M2M 关系返回，语言过滤和选择完全由客户端（Admin 或自定义前端）负责。

这种设计的优势：
1. **灵活性**: 客户端可以同时获取多种语言的翻译
2. **一致性**: API 行为统一，不依赖请求上下文
3. **可预测性**: 响应结构始终一致，便于缓存和处理

---

## 五、缺失翻译处理策略

### 5.1 前端展示层处理

**文件位置**: `app/src/displays/translations/index.ts`

翻译显示组件实现了完整的语言回退机制：

```typescript
export default defineDisplay({
  id: 'translations',
  name: '$t:displays.translations.translations',
  description: '$t:displays.translations.description',
  icon: 'translate',
  component: DisplayTranslations,
  handler: (values, options, { collection, field }) => {
    if (!field || !collection || !Array.isArray(values)) return values;

    // 1. 获取关系信息
    const relations = relationsStore.getRelationsForField(collection, field.field);
    const junction = relations.find(
      (relation) => relation.related_collection === collection && relation.meta?.one_field === field.field,
    );
    if (!junction) return values;

    const relation = relations.find(
      (relation) => relation.collection === junction.collection && relation.field === junction.meta?.junction_field,
    );
    if (!relatedCollection) return values;

    // 2. 查找匹配语言的翻译
    const value =
      values.find((translatedItem: Record<string, any>) => {
        const lang = translatedItem[relation!.field][relatedPrimaryKeyField.field];

        // 无语言时回退到第一条
        if (!lang) return true;

        // 优先使用用户语言（如果启用）
        if (options.userLanguage) {
          return lang === i18n.global.locale.value;
        }

        // 否则使用配置的默认语言
        return lang === options.defaultLanguage;
      }) ?? values[0];  // 最终回退到第一条翻译

    // 3. 渲染模板
    const fieldKeys = getFieldsFromTemplate(options.template);
    // ... 渲染逻辑

    return renderPlainStringTemplate(options.template, stringValues);
  },
  // ...
});
```

### 5.2 前端回退链总结

前端翻译显示的完整回退链：

```
┌─────────────────────────────────────────────────────────────┐
│                    翻译显示回退链                              │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 检查 options.userLanguage 是否启用                       │
│     ├── 启用 → 查找语言 === i18n.global.locale.value 的翻译  │
│     │                        │                               │
│     │              找到? ──────→ 返回该翻译                   │
│     │                        │                               │
│     │                  否 ────┐                              │
│     │                         │                              │
│  2. 检查 options.defaultLanguage 是否配置                    │
│     ├── 配置 → 查找语言 === options.defaultLanguage 的翻译   │
│     │                        │                               │
│     │              找到? ──────→ 返回该翻译                   │
│     │                        │                               │
│     │                  否 ────┐                              │
│     │                         │                              │
│  3. 最终回退                                                  │
│     └── 返回 values[0]（第一条翻译）                          │
│                                                              │
│  注意：如果 values 为空数组，则返回 undefined                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 翻译字段编辑界面处理

**文件位置**: `app/src/interfaces/translations/translation-form.vue`

编辑界面对于缺失翻译的处理：

```typescript
// 获取指定语言的翻译项
const item = computed(() => {
  const item = getItemWithLang(displayItems, lang.value);
  if (item === undefined) return undefined;

  const itemEdits = getItemEdits(item);

  if (isEmpty(itemEdits) && item.$type === 'deleted') return item;

  return itemEdits;
});

// 启用翻译（创建空翻译项）
function onEnableTranslation(lang?: string, item?: DisplayItem, itemInitial?: DisplayItem) {
  if (!isEmpty(item) || !isEmpty(itemInitial)) return;
  updateValue(item, lang);  // 创建新的空翻译记录
}

// 获取指定语言的项
function getItemWithLang<T extends Record<string, any>>(items: T[], lang: string | undefined) {
  const langField = relationInfo.value?.junctionField.field;
  const relatedPKField = relationInfo.value?.relatedPrimaryKeyField.field;
  if (!langField || !relatedPKField || !lang) return;

  return items.find((item) => item?.[langField]?.[relatedPKField] === lang);
}
```

### 5.4 API 响应层处理

**重要**: Directus API 对缺失翻译**不做任何特殊处理**。翻译数据作为标准的 M2M 关系返回，响应结构完全取决于：

1. **请求的 `fields` 参数** - 决定返回哪些字段
2. **请求的 `deep` 参数** - 决定嵌套关系的查询行为
3. **数据库中实际存在的记录** - 缺失的语言翻译不会返回

#### 典型 API 响应示例

请求获取文章及其翻译：

```http
GET /items/articles?fields=id,title,translations.*,translations.languages_code.code
```

响应（假设只有英文和中文翻译）：

```json
{
  "data": [
    {
      "id": 1,
      "title": "Draft",  // 主表原始值（可能被翻译覆盖或作为回退）
      "translations": [
        {
          "id": 101,
          "articles_id": 1,
          "languages_code": {
            "code": "en-US",
            "name": "English"
          },
          "title": "Hello World",
          "content": "This is the English content"
        },
        {
          "id": 102,
          "articles_id": 1,
          "languages_code": {
            "code": "zh-CN",
            "name": "Chinese"
          },
          "title": "你好世界",
          "content": "这是中文内容"
        }
      ]
    }
  ]
}
```

注意：
- 法语（`fr-FR`）翻译不存在，所以不会出现在 `translations` 数组中
- 客户端需要根据自身语言偏好和回退逻辑来选择显示哪条翻译

### 5.5 前后端缺失翻译处理对比

| 场景 | 前端（Admin）处理 | API 响应处理 |
|------|------------------|--------------|
| **显示时** | 按用户语言→默认语言→第一条的顺序回退 | 返回所有存在的翻译，无回退 |
| **编辑时** | 显示空表单，允许创建新翻译 | 需要显式 POST/PATCH 创建记录 |
| **未找到** | 使用第一条翻译（或显示空值） | 该语言翻译项不在数组中 |

---

## 六、关键实现文件索引

### 6.1 前端文件

| 文件路径 | 功能描述 |
|----------|----------|
| `app/src/lang/set-language.ts` | 语言切换核心逻辑 |
| `app/src/stores/user.ts` | 用户语言偏好管理 |
| `app/src/stores/translations.ts` | 用户自定义翻译存储 |
| `app/src/interfaces/translations/translations.vue` | 翻译字段主界面 |
| `app/src/interfaces/translations/translation-form.vue` | 翻译表单组件 |
| `app/src/interfaces/translations/language-select.vue` | 语言选择器组件 |
| `app/src/displays/translations/index.ts` | 翻译显示处理器（含回退逻辑） |
| `app/src/composables/use-relation-multiple.ts` | 多关系查询逻辑 |
| `app/src/composables/use-relation-m2m.ts` | M2M 关系信息解析 |

### 6.2 后端文件

| 文件路径 | 功能描述 |
|----------|----------|
| `api/src/utils/generate-translations.ts` | 翻译字段生成逻辑 |
| `api/src/services/translations.ts` | 翻译表操作服务 |
| `api/src/controllers/translations.ts` | 翻译 API 路由 |
| `api/src/database/get-ast-from-query/get-ast-from-query.ts` | 查询 AST 生成 |
| `api/src/database/get-ast-from-query/lib/parse-fields.ts` | 字段解析（含关系检测） |
| `api/src/database/get-ast-from-query/utils/get-deep-query.ts` | Deep 参数提取 |
| `api/src/utils/sanitize-query.ts` | 查询参数清洗 |

---

## 七、最佳实践与常见问题

### 7.1 前端开发最佳实践

1. **使用 Display 组件处理翻译显示**
   ```typescript
   // 利用 translations display 的回退逻辑
   const display = useExtension('display', 'translations');
   const displayedValue = display.value?.handler(translations, options, context);
   ```

2. **手动实现语言回退**
   ```typescript
   function getTranslationForLang(translations: any[], targetLang: string, defaultLang?: string) {
     // 1. 尝试目标语言
     let translation = translations.find(t => t.languages_code?.code === targetLang);
     
     // 2. 回退到默认语言
     if (!translation && defaultLang) {
       translation = translations.find(t => t.languages_code?.code === defaultLang);
     }
     
     // 3. 回退到第一条
     if (!translation && translations.length > 0) {
       translation = translations[0];
     }
     
     return translation;
   }
   ```

### 7.2 常见问题

**Q: 为什么 API 不提供 `?lang=` 参数来过滤翻译？**

A: 这是设计决策。Directus 认为：
1. 翻译数据是完整的关系数据，应该完整返回
2. 客户端应该根据自身需求选择语言
3. 这种设计更灵活，支持同时使用多种语言的场景

**Q: 如何在自定义前端中实现类似 Admin 的语言回退？**

A: 参考 `app/src/displays/translations/index.ts` 的实现，关键点：
1. 获取当前用户语言（从用户设置或浏览器）
2. 按优先级查找翻译：用户语言 → 配置的默认语言 → 第一条
3. 处理空值情况

**Q: 主表中的原始字段（如 `title`）和翻译表中的字段（如 `translations.title`）是什么关系？**

A: 在 Directus 的翻译字段设计中：
- 主表的原始字段**通常不再使用**，或仅作为迁移/备份
- 实际使用的是翻译表中的对应字段
- 可以通过 `generate-translations` 工具将现有字段迁移到翻译表

---

## 八、总结

Directus 多语言字段的实现遵循以下核心原则：

1. **关系优先**: 翻译字段本质上是特殊标记的 M2M 关系
2. **客户端负责语言选择**: API 不做语言过滤，完全由客户端控制
3. **多层回退机制**: Admin 实现了完整的语言回退链
4. **数据完整性**: 每种语言的翻译独立存储，互不干扰

这种设计虽然增加了客户端的复杂度，但带来了更高的灵活性和一致性，使 Directus 能够适应各种多语言场景的需求。
