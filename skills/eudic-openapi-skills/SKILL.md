---
name: eudic-openapi-skills
description: 调用欧路词典、法语助手、德语助手和西语助手 OpenAPI，查询和管理用户自己的生词本、语料、例句与笔记。适用于查看生词本、收藏单词、筛选最近语料和保存笔记等请求，不用于通用词典查词。
---

# 欧路词典/法语助手/德语助手/西语助手 (Eudic OpenAPI)

通过 HTTPS 直接调用欧路 OpenAPI，管理用户自己的生词本、语料、例句和笔记。需要能发送 HTTP 请求的执行环境，无需配置 MCP。

## 何时使用 (触发条件)

当用户提出以下类型的请求时，应使用本技能：

- "查看我的生词本"
- "添加单词到生词本"
- "查询我的生词本中某个单词的详情"
- "查询我的语料"
- "按最近N天或是否已收藏筛选语料"
- "批量查询语料单词详情"
- "添加笔记"
- "查看笔记"
- "删除生词本/单词/笔记"

## 环境配置

### 获取 API Token

1. 由用户访问[获取授权](https://my.eudic.net/OpenAPI/Authorization)，复制页面提供的完整授权信息（已经包含 `NIS ` 前缀）。
2. 从执行环境的 `EUDIC_AUTHORIZATION` 变量读取完整值，并原样放入 `Authorization` Header，不要重复添加 `NIS `。
3. 缺少授权时提示用户配置；不要把授权值写入 skill、项目文件、日志或回复。配置需对实际执行 HTTP 请求的进程可见。

下方示例使用 Bash 和 curl，`-i` 同时输出响应头和响应体，需先读取状态码再解析 JSON；Windows PowerShell 请用 `curl.exe`，将授权变量写成 `$env:EUDIC_AUTHORIZATION`、分类变量写成 `$env:EUDIC_CATEGORY_ID`，并使用 PowerShell 的续行语法或单行命令。

### 支持的语言与应用名称

| 语言 | language 参数 | 应用名称 |
|------|--------------|----------|
| 英语 | en | 欧路词典 |
| 法语 | fr | 法语助手 |
| 德语 | de | 德语助手 |
| 西班牙语 | es | 西语助手 |

GET 请求通过 query 参数传入 `language`；下方 POST、PATCH、DELETE 请求通过 JSON body 传入。语言根据用户指定的应用确定，未明确时先询问，不跨语言猜测数据。

## 操作流程

- 第一次先用指定语言获取生词本分类，验证授权并取得真实 ID。没有找到目标分类时列出结果供用户选择，不猜测 ID，也不把 `0` 当作固定分类 ID。
- 下方 `生词本ID` 均需替换为查询得到的真实 ID；GET 示例中的 `EUDIC_CATEGORY_ID` 也需先设置为该 ID。只有用户未指定分类的新增单词请求，才可省略 `category_ids` 使用默认分组。
- 查询单词详情仅表示读取用户生词本或语料中的记录；404 或空列表不代表词典不存在该词。
- 只执行用户要求的操作。重命名分类、删除分类、删除单词与删除笔记是不同操作；添加笔记会保存或覆盖已有内容，不会自动收藏单词。
- 列表的 `page` 默认 `0`、最大 `50`，`page_size` 默认 `100`、范围 `1` 到 `100`。负页码按 `0` 处理，不存在 `-1` 获取全部的模式。仅按任务需要翻页，空页停止；到达上限时说明结果可能不完整。
- 批量增删单词每次 `1` 到 `1000` 个，优先使用批量接口。不要为了查每个单词而逐条重复请求。
- 读取 HTTP 状态码后再解析响应；列表通常在 `data` 数组中，单个生词本单词详情直接返回对象。204 没有响应体，不要强行解析 JSON。
- 401 时停止并提示检查授权；403 或 429 时停止请求并报告限流，遵循服务返回的等待时间，不自动循环重试。写入结果不确定时先查询核实，不盲目重复写入。

## 示例约定

以下请求只有在用户明确提出对应需求后才执行。`en` 是英语示例，使用其他应用时需替换所有对应的 `language` 值。

---

## 核心功能与用法

### 1. 获取所有生词本

```bash
# language: en=欧路词典, fr=法语助手, de=德语助手, es=西语助手
curl -sS -i "https://api.frdic.com/api/open/v1/studylist/category?language=en" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}"
```

**参数**: 
- `language`: en/fr/de/es (必填)

**返回**: `{data: [...], message: ""}`；`data` 为生词本列表 (id, language, name, add_time)。

---

### 2. 添加新生词本

```bash
curl -sS -i -X POST "https://api.frdic.com/api/open/v1/studylist/category" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "name": "新单词本"}'
```

**参数**:
- `language`: en/fr/de/es (必填)
- `name`: 生词本名称 (必填)

---

### 3. 重命名生词本

```bash
curl -sS -i -X PATCH "https://api.frdic.com/api/open/v1/studylist/category" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"id": "生词本ID", "language": "en", "name": "新名称"}'
```

**参数**:
- `id`: 生词本ID (必填)
- `language`: en/fr/de/es (必填)
- `name`: 新名称 (必填)

---

### 4. 删除生词本

```bash
curl -sS -i -X DELETE "https://api.frdic.com/api/open/v1/studylist/category" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"id": "生词本ID", "language": "en"}'
```

---

### 5. 获取生词本单词

```bash
curl -sS -i "https://api.frdic.com/api/open/v1/studylist/words?language=en&category_id=${EUDIC_CATEGORY_ID}&page=0&page_size=100" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}"
```

**参数**:
- `language`: en/fr/de/es (必填)
- `category_id`: 查询得到的真实生词本ID (必填)
- `page`: 页码 (可选，默认0，最大50)
- `page_size`: 每页数量 (可选，默认100，范围1到100)

**返回**: `{data: [...], message: ""}`；`data` 为单词列表 (word, phon, exp, add_time, star, context_line)，部分字段可省略。

---

### 6. 批量添加单词

```bash
curl -sS -i -X POST "https://api.frdic.com/api/open/v1/studylist/words" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "category_id": "生词本ID", "words": ["apple", "banana", "orange"]}'
```

**参数**:
- `language`: en/fr/de/es (必填)
- `category_id`: 查询得到的真实生词本ID (必填)
- `words`: 单词数组 (必填)

---

### 7. 删除单词

```bash
curl -sS -i -X DELETE "https://api.frdic.com/api/open/v1/studylist/words" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "category_id": "生词本ID", "words": ["apple"]}'
```

---

### 8. 新增单个单词

```bash
curl -sS -i -X POST "https://api.frdic.com/api/open/v1/studylist/word" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "word": "hello", "star": 2, "context_line": "Hello, how are you?"}'
```

**参数**:
- `language`: en/fr/de/es (必填)
- `word`: 单词 (必填)
- `star`: 星级0到5 (可选，默认1)
- `context_line`: 语境例句 (可选)
- `category_ids`: 分组ID列表 (可选)

---

### 9. 查询单词

```bash
curl -sS -i "https://api.frdic.com/api/open/v1/studylist/word?language=en&word=hello" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}"
```

**参数**:
- `language`: en/fr/de/es (必填)
- `word`: 单词 (必填)

**返回**: 直接返回单词对象 (word, exp, add_time, star, category_ids, context_line)，部分字段可省略；不在生词本时返回404。

---

### 10. 获取笔记列表

```bash
curl -sS -i "https://api.frdic.com/api/open/v1/studylist/notes?language=en&page=0&page_size=100" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}"
```

**参数**:
- `language`: en/fr/de/es (必填)
- `page`: 页码 (可选，默认0，最大50)
- `page_size`: 每页数量 (可选，默认100，范围1到100)

**返回**: `{data: [...], message: ""}`；`data` 为笔记列表 (word, note, language, add_time)。

---

### 11. 获取例句列表

```bash
curl -sS -i "https://api.frdic.com/api/open/v1/studylist/sentences?language=en&page=0&page_size=100" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}"
```

**参数**:
- `language`: en/fr/de/es (必填)
- `page`: 页码 (可选，默认0，最大50；负数按0处理)
- `page_size`: 每页数量 (可选，默认100，范围1到100)

**返回**: `{data: [...], message: ""}`；`data` 为例句列表 (sentence, translation, note, add_time)。

---

### 12. 查询用户语料列表

```bash
curl -sS -i "https://api.frdic.com/api/open/v1/studylist/vocab_entries?language=en&recent_days=30&is_favorited=true&page=0&page_size=20" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}"
```

**参数**:
- `language`: en/fr/de/es (必填)
- `recent_days`: 最近N天 (可选，小于等于0表示不筛选)
- `is_favorited`: 是否已收藏到生词本 (可选，`true` 只看已收藏，`false` 只看未收藏)
- `page`: 页码 (可选，默认0，最大50；负数按0处理)
- `page_size`: 每页数量 (可选，默认100，范围1到100)

**返回**: `{data: [...], message: ""}`；`data` 为语料列表 (word, exp, add_time, rating, contexts)，`contexts` 中包含 src、trans、word_in_context。

---

### 13. 批量查询用户语料单词详情

```bash
curl -sS -i -X POST "https://api.frdic.com/api/open/v1/studylist/vocab_words" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "words": ["action", "amplify"]}'
```

**参数**:
- `language`: en/fr/de/es (必填)
- `words`: 单词数组 (必填，会自动忽略空白并去重)

**返回**: `{data: [...], message: ""}`；`data` 为用户语料中的单词详情列表 (word, exp, contexts)，`contexts` 中包含 src、trans、word_in_context。

---

### 14. 获取单个单词笔记

```bash
curl -sS -i "https://api.frdic.com/api/open/v1/studylist/note?language=en&word=hello" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}"
```

**参数**:
- `language`: en/fr/de/es (必填)
- `word`: 单词 (必填)

**返回**: 存在时返回 `{data: {word, note, language, add_time}, message: ""}`；笔记不存在时返回404，响应体可能为空。
---

### 15. 新增或更新笔记

```bash
curl -sS -i -X POST "https://api.frdic.com/api/open/v1/studylist/note" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "word": "hello", "note": "这是笔记内容"}'
```

**参数**:
- `language`: en/fr/de/es (必填)
- `word`: 单词 (必填)
- `note`: 笔记内容 (必填)

---

### 16. 删除笔记

```bash
curl -sS -i -X DELETE "https://api.frdic.com/api/open/v1/studylist/note" \
  -H "Authorization: ${EUDIC_AUTHORIZATION}" \
  -H "Content-Type: application/json" \
  -d '{"language": "en", "word": "hello"}'
```

**参数**:
- `language`: en/fr/de/es (必填)
- `word`: 单词 (必填)

**返回**: 删除笔记成功时返回201和提示信息；删除分类或批量删除单词成功时返回204。

---

## 响应码

| 响应码 | 含义 |
|--------|------|
| 200 | 查询成功（GET 或批量查询语料的 POST） |
| 201 | 创建/修改成功 (POST/PATCH) |
| 204 | 删除分类或批量删除单词成功，无响应体；删除笔记成功为201 |
| 400 | 参数错误 |
| 401 | 授权认证失败 |
| 403 / 429 | 请求受限，停止请求并按服务提示等待 |
| 404 | 生词本单词或笔记不存在，响应体可能为空 |
| 500 | 服务异常；写入结果不确定时先查询核实 |

## 流量限制

| 周期 | 限制次数 | 封停时间 |
|------|----------|----------|
| 1分钟 | 30次 | 1小时 |
| 30分钟 | 500次 | 24小时 |


接口详细说明：[生词本与语料 API](https://my.eudic.net/OpenAPI/Doc_Api_Study)、[笔记 API](https://my.eudic.net/OpenAPI/Doc_Api_Note)。
