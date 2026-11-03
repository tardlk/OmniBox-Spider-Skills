# JavaScript SDK | OmniBox

> 本文件是 JS 侧展开说明，按 `latest` 镜像内的真实 `omnibox_sdk.js` 校准（2026-11）。
> 只要看"有哪些方法 / 签名 / 坑"，优先读 `sdk-api.md`。

---

## 0. 先记住三条变更

1. `OmniBox.getEnv()` **已移除** → 用 `process.env.KEY`。
2. `OmniBox.getAnalyzeSites()` **已移除且无替代** → 需要嗅探用 `sniffVideo()`，或返回 `parse: 1`。
3. `OmniBox.addPlayHistory()` **已移除** → 播放记录由宿主按 `play()` 返回值自动写；要读历史用 `getSourceCategoryData("history", ...)`。

旧脚本里还留着这三类调用时，先删掉再谈其他逻辑。

---

## 引入方式

```javascript
const OmniBox = require("omnibox_sdk");
const runner = require("spider_runner");

module.exports = { home, category, detail, search, play };
runner.run(module.exports);
```

### 说明
- SDK 由运行器通过 `NODE_PATH` 注入，本地直接 `node script.js` 会找不到模块；本地调试要么走 Runner，要么加 fallback（见 SKILL.md 第 15 节）。
- handler 推荐统一签名：`(params, context)`。
- `context.from` 表示调用端：`web / tvbox / uz / catvod / emby`。
- 不需要自己读写 stdin/stdout，`runner.run()` 已经处理。

---

## 1. `OmniBox.request(url, options)`

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `url` | `string` | 是 | 请求地址 |
| `options.method` | `string` | 否 | 默认 `GET` |
| `options.headers` | `object` | 否 | 请求头 |
| `options.body` | `string \| object` | 否 | 对象会自动 JSON 序列化，并强制 `Content-Type: application/json` |

返回 `{ statusCode, headers, body }`，`body` 是字符串。

```javascript
async function requestJson(url, options = {}) {
  const res = await OmniBox.request(url, {
    ...options,
    headers: {
      "User-Agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)",
      Referer: BASE_URL + "/",
      ...(options.headers || {}),
    },
  });
  if (res.statusCode !== 200) {
    await OmniBox.log("error", `[req] HTTP ${res.statusCode} ${url}`);
    return null;
  }
  try {
    return JSON.parse(res.body || "{}");
  } catch (e) {
    await OmniBox.log("error", `[req] JSON 解析失败 ${url}: ${e.message}`);
    return null;
  }
}
```

要点：
- `body` 不会自动解析，必须自己 `JSON.parse`；
- 要提交表单（`x-www-form-urlencoded`）时，显式覆盖 `Content-Type`；
- 建议统一封装 `requestJson / requestHtml`，别在每个 handler 里重复判状态码。

---

## 2. `OmniBox.log(level, message)`

级别 `info` / `warn` / `error`。内部走 `/log` 接口，失败只 `console.error`，不会中断脚本。

```javascript
await OmniBox.log("info", `[detail] videoId=${videoId}`);
await OmniBox.log("warn", `[play] 未命中刮削映射，走原始分集名`);
await OmniBox.log("error", `[play] 取播失败: ${e.message}`);
```

建议：入口参数、关键请求 URL、返回条数、失败原因四项必打。

---

## 3. 环境变量（已不再有 `getEnv`）

```javascript
const BASE_URL = (process.env.SITE_BASE_URL || "https://example.com").replace(/\/+$/, "");
const TOKEN = process.env.SITE_TOKEN || "";
```

命名优先复用仓库既有变量（`SITE_API`、`DANMU_API`、`DRIVE_ORDER` 等），新增变量要在交付说明里解释原因。

---

## 4. `OmniBox.getSourceFavoriteTags()`

```javascript
const tags = await OmniBox.getSourceFavoriteTags(); // string[]
```

用于"按收藏标签分区展示"。依赖 `context.sourceId`，缺失会 throw。

---

## 5. `OmniBox.getSourceCategoryData(categoryType, page = 1, pageSize = 20)`

| categoryType | 含义 |
|---|---|
| `history` | 当前源播放历史（**只读**） |
| `favorite` | 当前源收藏 |
| `follow` | 全站追剧，与源无关 |
| 其它 | 收藏标签名 |

```javascript
async function home(params, context) {
  const history = await OmniBox.getSourceCategoryData("history", 1, 12);
  const list = (history.list || []).map((it) => ({
    vod_id: it.vodId || it.vod_id,
    vod_name: it.title || it.vodName,
    vod_pic: it.pic || it.cover || "",
    type_id: "history",
    type_name: "最近观看",
  }));
  return { class: [{ type_id: "history", type_name: "最近观看" }], list };
}
```

返回 `{ list, total, pageCount }`，异常时 SDK 兜底空结构。

---

## 6. 网盘能力

### 6.1 `getDriveFileList(shareURL, pdirFid = "0")`

```javascript
async function listAllVideos(shareURL) {
  const out = [];
  const walk = async (pdirFid) => {
    const pageData = await OmniBox.getDriveFileList(shareURL, pdirFid);
    for (const f of pageData.files || []) {
      if (f.dir) await walk(f.fid);
      else if (/\.(mp4|mkv|m3u8|ts|avi)$/i.test(f.file_name || f.name || "")) out.push(f);
    }
  };
  await walk("0");
  return out;
}
```

- 返回 `{ files, total, has_more }`；
- 目录用子 `fid` 递归，别把目录当视频；
- 递归前建议先上缓存（同一分享重复展开很贵）。

### 6.2 `getDriveVideoPlayInfo(shareURL, fid, flag = "", path = "", getTranscodeUrls = true)`

```javascript
const info = await OmniBox.getDriveVideoPlayInfo(shareURL, fid, "", "剧集/第01集.mp4");
if (!info.url) throw new Error("网盘未返回可播地址");
```

- `path`：quark / uc 支持按路径定位，能少走整树遍历，**推荐在已知路径时传**；
- `getTranscodeUrls`：仅夸克有意义，`false` = 不转存、只分享直下；
- 兼容旧写法：第 4 个参数传 boolean 会被当作 `getTranscodeUrls`；
- 失败不抛错，返回空 `url`，必须自己判空。

### 6.3 `getDriveFileRawUrl(shareURL, fid)`（未文档化）

```javascript
const { url } = await OmniBox.getDriveFileRawUrl(shareURL, fid);
```

原始下载直链，适合"下载而非在线播放"。

### 6.4 `getDriveShareInfo(shareURL)` / `getDriveInfoByShareURL(shareURL)`

```javascript
const info = await OmniBox.getDriveInfoByShareURL(shareURL);
if (info.driveType === "others") {
  // 未知网盘，没有图标也没有播放链路，直接跳过
  return null;
}
```

- `getDriveInfoByShareURL` 未识别/失败统一返回 `others` + `未知网盘`，**不要用 `!driveType` 判断**；
- `getDriveShareInfo` 拿 `stoken` 等前置信息。

### 6.5 `checkRapidTransfer` / `rapidTransferPlay`（未文档化）

秒传加速，可选优化：`available` 为真再 `rapidTransferPlay(shareURL, fid, targetDrive, flag)`。主链路不要依赖。

---

## 7. 刮削能力

### 7.1 `processScraping(videoId, keyword, resourceName, videoFiles)`

```javascript
const scrapeCandidates = episodes.map((ep) => ({
  fid: ep.fid,
  file_id: ep.fid,
  file_name: ep.name,
  name: ep.name,
  format_type: "video",
}));

try {
  await OmniBox.processScraping(videoId, vodName, vodName, scrapeCandidates);
} catch (e) {
  await OmniBox.log("warn", `[detail] 刮削失败: ${e.message}`);
}
```

关键点：
- `videoId` 会被 SDK 按 `|` 截断取第一段，再拼 `context.sourceId` 生成 `spider_source_{sourceId}_{videoId}`；
- 因此 `detail()` 和 `play()` 传的 `videoId` **前缀必须一致**；
- 归一化后为空会 throw，必须 try/catch；
- 一次刮削、之后所有线路回填，别按线路重复刮。

### 7.2 `getScrapeMetadata(videoId)`

```javascript
const { scrapeData, videoMappings } = await OmniBox.getScrapeMetadata(videoId);
```

- 无数据返回 `{ scrapeData: null, videoMappings: null }`，不抛错；
- 匹配键：网盘用 `{shareURL}|{fileId}`，采集用你 `detail()` 里写进 `videoFiles` 的同一个键；
- 拿 `scrapeData` 回填 `vod_name / vod_pic / vod_content / vod_actor / vod_director / vod_area`；
- 拿 `videoMappings[].episodeName` 改分集名。

### 7.3 `processDriveScraping` / `getDriveMetadata`（兼容别名）

等价于把 `shareURL` 当作 `videoId` 传。旧脚本可留着，新脚本统一写 `processScraping` / `getScrapeMetadata`。

---

## 8. 播放辅助

### 8.1 `sniffVideo(url, headers)` → `{ url, header }`

```javascript
const sniffed = await OmniBox.sniffVideo(targetUrl, {
  Referer: BASE_URL + "/",
  "User-Agent": UA,
});
if (sniffed && sniffed.url) return { urls: [{ name: "播放", url: sniffed.url }], flag, header: sniffed.header, parse: 0 };
```

- 需要带 Playwright 的完整环境（lite 镜像没有）；
- 推荐三段式：直链 `parse:0` → 网页 URL 先 `sniffVideo` → 仍失败再 `parse: 1`。

### 8.2 `getVideoMediaInfo(playUrl, headers)`

```javascript
const mediaInfo = await OmniBox.getVideoMediaInfo(playUrl, { Referer: BASE_URL + "/" });
const duration = Number(mediaInfo?.format?.duration || 0);
```

- 只在**已拿到可播地址之后**调用；不要放在搜索/列表阶段；
- 失败返回 `{}`，字段一律做兜底。

### 8.3 `getDanmakuByFileName(fileName)`

```javascript
const danmaku = await OmniBox.getDanmakuByFileName(fileName); // [{ name, url }]
```

失败返回 `[]`，直接塞进 `play()` 的 `danmaku` 字段即可。

---

## 9. 缓存能力

```javascript
async function getCachedJson(key) {
  try {
    const v = await OmniBox.getCache(key);
    return v == null ? null : v;
  } catch (e) {
    return null;
  }
}

async function setCachedJson(key, value, ttl = 3600) {
  try {
    await OmniBox.setCache(key, value, ttl);
  } catch (e) {
    await OmniBox.log("warn", `[cache] 写入失败 ${key}: ${e.message}`);
  }
}
```

约束：
- **key ≤ 512 字节**，value 单条 JSON ≈ **512KB** 上限；
- 按 `context.sourceId` 隔离，缺 `sourceId` 会 throw；
- 未命中返回 `null`，写入过 `null` 也返回 `null`；
- 省略 `exSeconds` = 永不过期；
- 公开路由，**不要缓存 Cookie / Token**。

长 key 一定要哈希：

```javascript
const crypto = require("crypto");
const shortKey = (prefix, raw) =>
  `${prefix}:${crypto.createHash("md5").update(String(raw)).digest("hex")}`;
```

---

## 10. 影视库（仅 JS，未文档化）

```javascript
await OmniBox.upsertMediaItems([
  { sourceType: "cloud_share", sourceKey: shareURL, title: vodName, tmdbType: "tv", tmdbId: 12345 },
]);
const { list, total } = await OmniBox.listMediaItems({ page: 1, pageSize: 20, sourceType: "cloud_share" });
const item = await OmniBox.getMediaItem("some-id");
```

- `sourceType`：`site | webdav | cloud_share`；
- 必填 `sourceType / sourceKey / title`；
- 与 `sourceId` 无关，定时任务也能写。

---

## 11. 通用开发建议

### 统一请求封装
```javascript
const BASE_URL = (process.env.SITE_BASE_URL || "").replace(/\/+$/, "");
const UA = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0 Safari/537.36";

async function fetchHtml(path) {
  const res = await OmniBox.request(path.startsWith("http") ? path : BASE_URL + path, {
    headers: { "User-Agent": UA, Referer: BASE_URL + "/" },
  });
  return res.statusCode === 200 ? res.body : "";
}
```

### 推荐开发习惯
- 每个 handler 先打入口参数日志；
- 每个外部请求后打"拿到多少条"；
- `try/catch` 兜底成空结构（`{ page: 1, pagecount: 0, total: 0, list: [] }` / `{ list: [] }` / `{ urls: [], flag }`）；
- `play()` 保持无副作用（不写缓存、不写历史），`prefetch` 为 `true` 时任何写操作都跳过。

### 典型空返回
```javascript
const emptyList = (page = 1) => ({ page: Number(page) || 1, pagecount: 0, total: 0, list: [] });
const emptyDetail = { list: [] };
const emptyPlay = (flag = "play") => ({ urls: [], flag });
```
