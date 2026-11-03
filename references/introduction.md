# 爬虫开发介绍 | OmniBox

> 基于最新官方文档同步：
> https://omnibox-doc.pages.dev/spider-development/introduction

## 概览

OmniBox 支持通过自定义爬虫脚本扩展视频源。可以使用：
- **JavaScript**（Node.js 运行时）
- **Python**（Python 3 运行时）

爬虫源的核心价值：
- 自定义视频源
- 统一接口
- 后台可视化编辑与调试
- 实时查看日志与执行结果

## 支持语言

### JavaScript
- Node.js 运行时
- 支持 Node 标准库与 npm 包
- 使用 `require()`
- 支持 ES6+

### Python
- Python 3 运行时
- 支持标准库与第三方库
- 支持 async/await
- 自动处理中文编码

## 核心接口

爬虫源**按需实现**以下 5 个方法，并非必须全量实现：

| 方法 | 作用 | 推荐签名 |
|---|---|---|
| `home` | 首页分类与推荐 | `(params, context)` |
| `category` | 分类分页列表 | `(params, context)` |
| `detail` | 视频详情 | `(params, context)` |
| `search` | 搜索结果 | `(params, context)` |
| `play` | 播放地址 | `(params, context)` |

说明：
- **推送类脚本**：通常只需 `detail + play`
- **常规影视源**：建议实现全部 5 个

## 请求上下文 `context`

运行器在调用 handler 时会传入第二个参数 `context`，应统一从这里读取调用上下文，不要从全局读取。

| 字段 | 类型 | 说明 |
|---|---|---|
| `baseURL` | string | 当前请求基础 URL，可用于拼接绝对链接 |
| `headers` | object | 客户端请求头（UA、Cookie 等） |
| `sourceId` | string | 当前爬虫源 ID |
| `from` | string | 调用端：`web`（默认）/ `tvbox` / `uz` / `catvod` / `emby` |

示例：

### JavaScript
```js
async function home(params, context) {
  const from = context?.from || 'web';
  const baseURL = context?.baseURL || '';
  const headers = context?.headers || {};
  const sourceId = context?.sourceId || '';
}
```

### Python
```python
async def home(params, context):
    from_val = (context or {}).get("from", "web")
    base_url = (context or {}).get("baseURL", "")
    headers = (context or {}).get("headers", {})
    source_id = (context or {}).get("sourceId", "")
```

## 脚本注释属性

官方当前明确说明会被后端解析并实际使用的属性只有：
- `@version`
- `@downloadURL`
- `@indexs`
- `@push`
- `@dependencies`

并且必须位于脚本**前 50 行**内。

## SDK 注入方式

SDK 由运行器自动注入：
- JavaScript：通过 `NODE_PATH`
- Python：通过 `PYTHONPATH`

常见引入方式：

### JavaScript
```js
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");
```

### Python
```python
from spider_runner import OmniBox, run
```

## 最新官方文档结构

当前爬虫开发主文档已收拢为 5 个页面：
- `introduction`
- `getting-started`
- `script-annotation-attributes`
- `api-reference`
- `sdk`

后续同步官方文档时，优先按这 5 个页面校准。

另外，官方现在在 `getting-started / api-reference` 里也更明确写出了：
- `home()` 可选返回 `filters / banner`
- `category()` / `search()` 推荐返回 `total`
- `search: 1` 是 UZ 专用字段
- `vod_tag: "folder"` 是目录项

## 开发流程建议

1. 在后台创建爬虫源
2. 选择 JS 或 Python
3. 实现最小 `home/detail/play` 或目标必需方法
4. 用后台调试器测试
5. 补齐 `category/search`
6. 补日志、兜底返回、端兼容逻辑

## 本地技能配套参考

- 快速起步：`getting-started.md`
- 接口结构：`api-reference.md`
- 注释属性：`script-annotation-attributes.md`
- 统一 SDK 视图：`sdk-api.md`
- 经验沉淀：`lessons-learned.md`、`omnibox-spider-lessons.md`
