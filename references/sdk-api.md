# OmniBox SDK API 速查

> **基线（2026-11）**：以 OmniBox `latest` 镜像内的真实 SDK 源码为准，并与官方文档
> `https://omnibox-gamma.vercel.app/spider-development/sdk.html` 校准。
> 源码位置：`/app/static/sdk/omnibox_sdk.js`、`omnibox_sdk.py`、`omnibox_sdk.d.ts`、`omnibox_sdk.pyi`。
>
> ⚠️ 官方 `sdk` 页只列出常用子集，**不是完整清单**；本文件按真实源码列出全部公开方法，并标注哪些未被官方文档收录。

---

## 0. 已移除的方法（最容易踩的坑）

新版 SDK 删掉了下列方法。**旧脚本或旧文档里出现这些调用，一律视为过时写法，不要再写进新脚本。**

| 旧调用 | 新版本现状 | 正确做法 |
|---|---|---|
| `OmniBox.getEnv(name)` / `get_env(name)` | **已移除** | 直接用 `process.env.NAME` / `os.environ.get("NAME")`，执行器启动子进程时会注入 |
| `OmniBox.getAnalyzeSites()` / `get_analyze_sites()` | **已移除，且无替代** | 脚本侧不再读取解析站列表；需要解析能力就用 `sniffVideo()`，或返回 `parse: 1` 交给客户端嗅探 |
| `OmniBox.addPlayHistory(...)` / `add_play_history(...)` | **已移除** | 播放记录改由宿主根据 `play()` 返回值自动记录，脚本不要再写；要**读**历史用 `getSourceCategoryData("history", page, pageSize)` |
| `getFrom()` / `getSourceId()` 等全局 getter | **已移除** | 统一从 handler 第二个参数取：`context.from` / `context.sourceId` |

> 这条变更是"新版文档看起来变短了"的主要原因：SDK 页从 20 个方法收缩到 16 个，`getEnv`、`getAnalyzeSites`、`addPlayHistory` 全部下线。

---

## 1. 引入方式

### JavaScript

```javascript
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");

module.exports = { home, category, detail, search, play };
runner.run(module.exports);
```

### Python

```python
from spider_runner import OmniBox, run

if __name__ == "__main__":
    run({"home": home, "category": category, "detail": detail, "search": search, "play": play})
```

- handler 推荐签名统一为 `(params, context)`。
- 上下文由 Runner 通过 `setRequestContext()` 注入 SDK，脚本一般不需要自己调用。
- **Python 侧方法名已统一为小驼峰**（`getSourceCategoryData`、`getDriveVideoPlayInfo`），旧版 snake_case 别名（`get_source_category_data`）在当前 SDK 中已不存在。

---

## 2. 方法总览

| # | 方法 | 分组 | 官方文档 | 说明 |
|---|---|---|---|---|
| 1 | `request(url, options)` | 基础 | ✅ | HTTP 请求 |
| 2 | `log(level, message)` | 基础 | ✅ | 打日志 |
| 3 | `getSourceFavoriteTags()` | 源内数据 | ✅ | 收藏标签列表 |
| 4 | `getSourceCategoryData(categoryType, page, pageSize)` | 源内数据 | ✅ | 历史/收藏/追剧/标签分页 |
| 5 | `getDriveFileList(shareURL, pdirFid)` | 网盘 | ✅ | 目录文件列表 |
| 6 | `getDriveVideoPlayInfo(shareURL, fid, flag, path, getTranscodeUrls)` | 网盘 | ✅（参数不全） | 取可播地址 |
| 7 | `getDriveFileRawUrl(shareURL, fid)` | 网盘 | ❌ | 原始下载直链 |
| 8 | `getDriveShareInfo(shareURL)` | 网盘 | ✅ | 分享信息（stoken 等） |
| 9 | `getDriveInfoByShareURL(shareURL)` | 网盘 | ✅ | 盘类型 / 名称 / 图标 |
| 10 | `checkRapidTransfer(shareURL)` | 网盘 | ❌ | 是否可秒传 |
| 11 | `rapidTransferPlay(shareURL, fid, targetDrive, flag)` | 网盘 | ❌ | 秒传后播放 |
| 12 | `processScraping(videoId, keyword, resourceName, videoFiles)` | 刮削 | ✅ | 触发 TMDB 刮削 |
| 13 | `processDriveScraping(...)` | 刮削 | ❌ | `processScraping` 的兼容别名 |
| 14 | `getScrapeMetadata(videoId)` | 刮削 | ✅ | 读刮削结果 |
| 15 | `getDriveMetadata(shareURL)` | 刮削 | ❌ | `getScrapeMetadata` 的兼容别名 |
| 16 | `sniffVideo(url, headers)` | 播放 | ✅ | 页面嗅探真实地址 |
| 17 | `getVideoMediaInfo(playUrl, headers)` | 播放 | ✅ | ffprobe 媒体信息 |
| 18 | `getDanmakuByFileName(fileName)` | 播放 | ✅ | 按文件名匹配弹幕 |
| 19 | `getCache` / `setCache` / `deleteCache` | 缓存 | ✅ | 跨请求缓存 |
| 20 | `upsertMediaItems` / `listMediaItems` / `getMediaItem` | 影视库 | ❌ | 仅 JS SDK 提供 |
| 21 | `getCloudAuth(driveType)` | 网盘 | ❌ | 仅 Python SDK 提供 |
| — | `setRequestContext` / `getRequestContext` | 内部 | ❌ | Runner 注入用，脚本不要依赖 |

---

## 3. 基础能力

### `request(url, options)` ✅

- 入参：`url`；`options.method`（默认 `GET`）、`options.headers`、`options.body`（字符串或对象，对象自动 JSON 序列化）。
- 出参：`{ statusCode, headers, body }`，`body` 是**字符串**，需要自己 `JSON.parse`。
- 实现细节：默认 `User-Agent: OmniBox-SDK/1.0`；自动处理 `gzip / deflate / br`；`options.body` 存在时会强制 `Content-Type: application/json`（表单提交要自己覆盖该头）。

```javascript
const res = await OmniBox.request("https://api.example.com/list", {
  method: "POST",
  headers: { "Content-Type": "application/json", Referer: "https://example.com/" },
  body: { page: 1 },
});
if (res.statusCode !== 200) throw new Error(`HTTP ${res.statusCode}`);
const data = JSON.parse(res.body || "{}");
```

### `log(level, message)` ✅

- 级别建议只用 `info / warn / error`。
- 内部走 `/log` 接口，失败只打 `console.error`，不会中断脚本。
- 日志前缀建议 `[站名][method]`，例如 `[wbbb][detail]`。

### 环境变量

SDK 已不再提供 `getEnv`：

```javascript
const token = process.env.SITE_TOKEN || "";
```

```python
import os
token = os.environ.get("SITE_TOKEN", "")
```

---

## 4. 请求上下文 context

Runner 每次调用 handler 前会 `setRequestContext(context)`，并把同一个对象作为**第二个参数**传入。

```ts
interface RequestContext {
  baseURL?: string;                 // OmniBox 服务地址，用于拼绝对链接
  headers?: Record<string, string>; // 客户端透传请求头
  sourceId?: string;                // 当前爬虫源 ID
  from?: string;                    // web(默认) | tvbox | uz | catvod | emby
  prefetch?: boolean;               // 预热调用，true 时不要产生副作用
}
```

- `sourceId`：`getSourceFavoriteTags` / `getSourceCategoryData` / `getCache` / `setCache` / `deleteCache` / `processScraping` / `getScrapeMetadata` 都依赖它。**缺 sourceId 时这些方法会直接 `throw`（不是返回空）**，本地脱离 Runner 跑脚本时要留意。
- `prefetch === true`：后端预热任务。此时**不要写缓存、不要写播放进度、不要打签到/解锁类接口**；`play` 等解析方法本身必须无副作用。
- 端差异统一从这里分支，不要再指望 `params.from`。

---

## 5. 爬虫源数据

### `getSourceFavoriteTags()` ✅ → `Promise<string[]>`

当前源已配置的收藏标签名列表。

### `getSourceCategoryData(categoryType, page = 1, pageSize = 20)` ✅

| categoryType | 含义 |
|---|---|
| `history` | 当前源的播放历史 |
| `favorite` | 当前源的收藏 |
| `follow` | 全站追剧（与源无关） |
| 其它字符串 | 按收藏标签名筛选（先用 `getSourceFavoriteTags()` 取候选） |

返回 `{ list, total, pageCount }`；失败时 SDK 兜底 `{ list: [], total: 0, pageCount: 0 }`。

> 播放历史现在是**只读**的：脚本只能读（`history`），写历史由宿主完成。

---

## 6. 网盘能力

### `getDriveFileList(shareURL, pdirFid = "0")` ✅

返回 `{ files, total, has_more }`。

- `pdirFid = "0"` 表示根目录。
- 光雅盘（`guangyapan.com/s/`）会自动把 `"0" / "root" / 空` 归一化成 `""`。
- 递归展开目录时自己判断目录项与 `fid`（子目录 ID）。

### `getDriveVideoPlayInfo(shareURL, fid, flag = "", path = "", getTranscodeUrls = true)` ✅

返回 `{ url, header, danmaku }`。

| 参数 | 说明 |
|---|---|
| `flag` | 播放方式（服务端代理 / 本地代理 / 直连，三种方式都支持）；拿不准就传 `""` |
| `path` | 目标文件路径；**quark / uc** 传入后可按路径逐层定位，省掉整树遍历 |
| `getTranscodeUrls` | 默认 `true`；仅夸克：`false` 时不转存，只走分享直下 |

- 兼容旧调用：第 4 个参数传 `boolean` 会被当成 `getTranscodeUrls`。
- 失败兜底 `{ url: "", header: {}, danmaku: [] }`，**不会 throw**，所以必须自己判空。

### `getDriveFileRawUrl(shareURL, fid)` ❌未文档化

返回 `{ url }`，原始下载直链。适合"需要下载而不是在线播放"的链路。

### `getDriveShareInfo(shareURL)` ✅

返回分享信息（`stoken` / `drive` / `displayName` 等）。部分盘的播放前置需要它。

### `getDriveInfoByShareURL(shareURL)` ✅

返回 `{ driveType, displayName, iconPath, iconUrl }`。

- 识别口径与盘搜（PanSou）对齐，只认明确配置的分享域名，按域名边界匹配；
- **未识别 / 请求失败一律返回 `driveType: "others"`、`displayName: "未知网盘"`、`iconUrl: ""`**，不再兜底成某个具体网盘；
- `pikpak / magnet / ed2k` 没有对应驱动，同样落到 `others`。

> 判断"这条链接能不能走网盘播放"时应显式排除 `others`，不要用 `if (!driveType)`。

### `checkRapidTransfer(shareURL)` / `rapidTransferPlay(shareURL, fid, targetDrive, flag)` ❌未文档化

秒传加速链路：先 `check` 拿 `{ available, targetDrives, sourceDrive }`，`available` 为真再 `rapidTransferPlay`。属于可选优化，主链路不要依赖。

---

## 7. 刮削能力

### `processScraping(videoId, keyword, resourceName, videoFiles)` ✅

- `videoId` 直接传 handler 收到的那个即可：SDK 会**按 `|` 取第一段**，再和 `context.sourceId` 拼成 `spider_source_{sourceId}_{videoId}` 作为刮削唯一标识。
- `videoFiles` 每项建议至少带 `fid / file_id / file_name / name / format_type: "video"`。
- 归一化后 `videoId` 为空会 **throw**；调用方要 `try/catch`。
- 失败会重新抛出，不会静默吞掉。

### `getScrapeMetadata(videoId)` ✅

返回 `{ scrapeData, videoMappings }`；**失败 / 无数据时返回 `{ scrapeData: null, videoMappings: null }`，不抛错**。

- 匹配剧集时用 `videoMappings[].fileId`；网盘场景约定键是 `{shareURL}|{fileId}`，采集场景用你 `detail()` 里写进 `videoFiles` 的同一个键。
- 因为 `videoId` 会按 `|` 截断，`detail` / `play` 两侧**必须用同一个前缀**，否则映射对不上。

### `processDriveScraping` / `getDriveMetadata` ❌未文档化

`processScraping` / `getScrapeMetadata` 的向后兼容别名，签名把 `videoId` 换成 `shareURL`。旧脚本可以继续用，新脚本统一写 `processScraping` / `getScrapeMetadata`。

---

## 8. 播放辅助

### `sniffVideo(url, headers)` ✅ → `{ url, header }`

加载页面并拦截请求，用正则匹配真实视频地址（等价 TVBox `parse=1` 的行为）。需要带 Playwright 的完整运行环境，lite 镜像不可用。

### `getVideoMediaInfo(playUrl, headers)` ✅

ffprobe 原始结果，常见 `format.duration` / `format.size` / `streams`；m3u8 在 ffprobe 失败时会自动回退清单解析。失败返回 `{}`。

- 适合：详情页补时长/大小、`play()` 里做直链校验。
- 不适合：搜索、列表阶段批量调用（会显著拖慢首屏）。

### `getDanmakuByFileName(fileName)` ✅ → `Array<{name, url}>`

按文件名匹配弹幕，失败返回 `[]`。

---

## 9. 缓存能力

```javascript
await OmniBox.setCache("detail:abc123", { list: [] }, 3600);
const cached = await OmniBox.getCache("detail:abc123"); // 未命中 = null
const removed = await OmniBox.deleteCache("detail:abc123"); // 0 / 1
```

实现约束（来自源码，务必按这个来）：

- **key 最长 512 字节**，单条 value JSON 约 **512KB** 上限；
- 缓存按 `context.sourceId` 隔离；缺 `sourceId` 会直接 `throw`；
- `getCache` 未命中返回 `null`；写入过 JSON `null` 后同样返回 `null`，无法区分（必要时用哨兵对象）；
- `setCache` 省略 `exSeconds` 表示**永不过期**；
- `/api/spider/omnibox` 是公开路由：**不要缓存 Cookie / Token**，源 ID 泄露即等于该源缓存可被他人读写。

> 实战仍建议：长 key（分享链接、筛选 JSON）先做 `md5/sha1` 短哈希，统一写成 `前缀:短哈希`，并在统一缓存 helper 里做一次长度归一化。

---

## 10. 影视库（仅 JS SDK）❌未文档化

```javascript
await OmniBox.upsertMediaItems([{ sourceType: "cloud_share", sourceKey: "uuid", title: "片名" }]);
const page = await OmniBox.listMediaItems({ page: 1, pageSize: 20, keyword: "", sourceType: "cloud_share" });
const item = await OmniBox.getMediaItem("id");
```

- `upsertMediaItems(items)`：批量写入，必填 `sourceType / sourceKey / title`，返回成功条数。
- `listMediaItems({ page, pageSize, keyword, sourceType, genre })`：`sourceType` 取 `site | webdav | cloud_share`。
- `getMediaItem(id)`：按主键取单条。
- 与 `sourceId` 无关，定时任务类脚本也能调用；Python SDK 当前**没有**这三个方法。

---

## 11. 运行器协议（本地调试用）

`spider_runner` 的契约，自己写本地 harness 时按这个来：

- 输入（stdin，UTF-8 JSON）：`{ "method": "home", "params": {...}, "context": {...} }`
- 成功（stdout）：`{ "success": true, "data": <handler 返回值>, "error": null }`
- 失败（stderr）：`{ "success": false, "data": null, "error": "..." }`，并以退出码 `1` 结束
- 未知 method 也走失败分支并退出 `1`
- SDK 的 API 基址：`process.env.OMNIBOX_API_URL || http://localhost:7023/api/spider/omnibox`

> 含义：**handler 里 throw 等于整次调用失败**。能兜底的地方就兜底成空结构，别把异常甩给宿主。

---

## 12. 快速对照（写脚本时最常用的几条）

| 需求 | 用法 |
|---|---|
| 请求站点接口 | `request()` + 手动 `JSON.parse` |
| 读环境变量 | `process.env.X` / `os.environ.get("X")` |
| 区分端 | `context.from` |
| 预热调用跳过写操作 | `context.prefetch === true` 时直接返回或跳过写缓存 |
| 缓存详情/搜索 | `getCache` / `setCache`（key ≤ 512B） |
| 网盘展开 | `getDriveFileList` 递归 → `getDriveVideoPlayInfo` |
| 判盘类型 | `getDriveInfoByShareURL`，`others` 即不可播 |
| 刮削 + 回填 | `processScraping` → `getScrapeMetadata`（一次刮、全线路回填） |
| 播放页取真实地址 | 先 `sniffVideo`，失败再 `parse: 1` |
| 时长/大小 | `getVideoMediaInfo`（只在拿到直链之后） |
| 弹幕 | `getDanmakuByFileName(fileName)` |
| 读播放历史 | `getSourceCategoryData("history", 1, 20)` |
