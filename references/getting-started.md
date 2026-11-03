# 快速开始 | OmniBox

> 基于最新官方文档同步：
> https://omnibox-doc.pages.dev/spider-development/getting-started

## 创建爬虫源

### 1. 进入管理后台
- 打开 OmniBox 管理后台
- 进入「爬虫源管理」页面
- 点击「新建爬虫源」

### 2. 填写基本信息
- **名称**：爬虫源显示名称
- **类型**：JavaScript 或 Python
- **描述**：可选

### 3. 创建脚本
系统支持两种类型：
- **JavaScript**：适合熟悉 Node.js 的开发者
- **Python**：适合熟悉 Python 的开发者

## JavaScript 最小示例

```javascript
// @name 示例爬虫源
// @version 1.0.0
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");

module.exports = { home, category, detail, search, play };
runner.run(module.exports);

async function home(params, context) {
  await OmniBox.log("info", "获取首页数据");
  return {
    class: [
      { type_id: "1", type_name: "电影" },
      { type_id: "2", type_name: "电视剧" },
    ],
    list: [
      {
        vod_id: "1",
        vod_name: "示例视频",
        vod_pic: "https://example.com/pic.jpg",
        type_id: "1",
        type_name: "电影",
        vod_remarks: "HD",
        vod_year: "2024",
        vod_douban_score: "8.5",
        vod_subtitle: "2024 / 中国大陆 / 剧情",
      },
    ],
    // filters: { "1": [{ key, name, init, value: [...] }] },
    // banner: [{ title, subtitle, backgroundImage, genre, actors, description }],
  };
}
```

## Python 最小示例

```python
# -*- coding: utf-8 -*-
# @name 示例爬虫源
# @version 1.0.0
from spider_runner import OmniBox, run

async def home(params, context):
    await OmniBox.log("info", "获取首页数据")
    return {
        "class": [
            {"type_id": "1", "type_name": "电影"},
            {"type_id": "2", "type_name": "电视剧"},
        ],
        "list": [
            {
                "vod_id": "1",
                "vod_name": "示例视频",
                "vod_pic": "https://example.com/pic.jpg",
                "type_id": "1",
                "type_name": "电影",
                "vod_remarks": "HD",
                "vod_year": "2024",
                "vod_douban_score": "8.5",
            }
        ],
    }

if __name__ == "__main__":
    run({"home": home})
```

## 五个核心 handler

| 方法 | 作用 | 常见返回 |
|---|---|---|
| `home` | 首页分类与推荐 | `{ class, list, banner?, filters? }` |
| `category` | 分类分页列表 | `{ page, pagecount, total, list }` |
| `detail` | 视频详情 | `{ list }` |
| `search` | 搜索 | `{ page, pagecount, total, list }` |
| `play` | 播放信息 | `{ urls, flag, header?, parse?, danmaku? }` |

## 重要提醒

- `context` 通过第二个参数获取，且不只是 `from`，还包括：
  - `baseURL`
  - `headers`
  - `sourceId`
- `home()` 现在可选返回 `filters` 与 `banner`
- `category()` / `search()` 返回里建议尽量补 `total`
- `play.parse = 1` 仅 **ok影视 app** 真正支持
- `vod_tag: "folder"` 表示目录项，不是播放项
- `search: 1` 是 UZ 专用字段，必要时再加

## 推荐开发顺序

1. 先做最小可运行脚本
2. 确认 `home` / `detail` / `play` 路通
3. 再补 `category` / `search`
4. 最后补过滤、banner、弹幕、刮削、历史等增强能力

## 本地技能建议联读

- `api-reference.md`
- `script-annotation-attributes.md`
- `sdk-api.md`
- `js-template.md` / `py-template.md`
