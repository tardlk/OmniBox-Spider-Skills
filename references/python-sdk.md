# Python SDK | OmniBox

> 本文件是 Python 侧展开说明，按 `latest` 镜像内的真实 `omnibox_sdk.py` / `omnibox_sdk.pyi` 校准（2026-11）。
> 只要看"有哪些方法 / 签名 / 坑"，优先读 `sdk-api.md`。

---

## 0. 三个必知变更

1. **方法名已统一为小驼峰**：`getSourceCategoryData`、`getDriveVideoPlayInfo`；旧的 `get_source_category_data` 这类 snake_case 别名在当前 SDK 里已不存在。
2. `getEnv` / `get_env` 一类的环境变量读取方法**已移除** → 用 `os.environ.get("KEY", "")`。
3. `get_analyze_sites()`（解析站列表）与 `add_play_history()`（写播放记录）**已移除** → 解析用 `sniffVideo()` 或 `parse: 1`；历史读取用 `getSourceCategoryData("history", ...)`。

另外，旧版 Python SDK 里的 `aes_encrypt / aes_decrypt / md5 / base64_encode / base64_decode` 也已下线，改用标准库：

```python
import base64, hashlib
digest = hashlib.md5(raw.encode()).hexdigest()
b64 = base64.b64encode(raw.encode()).decode()
```

需要 AES 时走 `@dependencies: pycryptodome`，自己封装。

---

## 引入方式

```python
# -*- coding: utf-8 -*-
import os
from spider_runner import OmniBox, run


async def home(params, context): ...
async def category(params, context): ...
async def detail(params, context): ...
async def search(params, context): ...
async def play(params, context): ...


if __name__ == "__main__":
    run({
        "home": home,
        "category": category,
        "detail": detail,
        "search": search,
        "play": play,
    })
```

说明：
- Runner 从 stdin 读 `{method, params, context}`，UTF-8 解码已由 Runner 处理（Windows 也不会有中文乱码）；
- handler 统一签名 `(params, context)`，`params` / `context` 都是 `dict`；
- `context` 字段：`baseURL / headers / sourceId / from / prefetch`；
- 不需要自己写 `main` 和 JSON 解析。

---

## 1. `OmniBox.request(url, options=None)`

```python
async def request_json(url, options=None):
    options = options or {}
    headers = {
        "User-Agent": UA,
        "Referer": BASE_URL + "/",
        **(options.get("headers") or {}),
    }
    res = await OmniBox.request(url, {**options, "headers": headers})
    if res.get("statusCode") != 200:
        await OmniBox.log("error", f"[req] HTTP {res.get('statusCode')} {url}")
        return None
    try:
        return json.loads(res.get("body") or "{}")
    except Exception as e:
        await OmniBox.log("error", f"[req] JSON 解析失败 {url}: {e}")
        return None
```

返回 `{"statusCode", "headers", "body"}`，`body` 仍是字符串。

---

## 2. `OmniBox.log(level, message)`

```python
await OmniBox.log("info", f"[detail] video_id={video_id}")
await OmniBox.log("error", f"[play] 失败: {e}")
```

级别 `info / warn / error`。失败只打印，不会中断脚本。

---

## 3. 环境变量

```python
import os
BASE_URL = (os.environ.get("SITE_BASE_URL") or "https://example.com").rstrip("/")
TOKEN = os.environ.get("SITE_TOKEN", "")
```

---

## 4. `OmniBox.getSourceFavoriteTags()`

```python
tags = await OmniBox.getSourceFavoriteTags()  # list[str]
```

依赖 `context.sourceId`，缺失会抛异常。

---

## 5. `OmniBox.getSourceCategoryData(category_type, page=1, page_size=20)`

| category_type | 含义 |
|---|---|
| `"history"` | 当前源播放历史（只读） |
| `"favorite"` | 当前源收藏 |
| `"follow"` | 全站追剧 |
| 其它 | 收藏标签名 |

注意：**方法名是驼峰，参数名仍是下划线**（`category_type / page_size`）。

```python
async def home(params, context):
    data = await OmniBox.getSourceCategoryData("history", 1, 12)
    items = data.get("list") or []
    vod_list = [
        {
            "vod_id": it.get("vodId") or it.get("vod_id"),
            "vod_name": it.get("title") or it.get("vodName"),
            "vod_pic": it.get("pic") or "",
            "type_id": "history",
            "type_name": "最近观看",
        }
        for it in items
    ]
    return {"class": [{"type_id": "history", "type_name": "最近观看"}], "list": vod_list}
```

---

## 6. 网盘能力

### 6.1 `getDriveFileList(share_url, pdir_fid="0")`

```python
async def list_all_videos(share_url):
    videos = []

    async def walk(pdir_fid):
        data = await OmniBox.getDriveFileList(share_url, pdir_fid)
        for f in data.get("files") or []:
            if f.get("dir"):
                await walk(f.get("fid") or "")
            elif re.search(r"\.(mp4|mkv|m3u8|ts|avi)$", (f.get("file_name") or f.get("name") or ""), re.I):
                videos.append(f)

    await walk("0")
    return videos
```

### 6.2 `getDriveVideoPlayInfo(share_url, fid, flag="", path="", get_transcode_urls=True)`

```python
info = await OmniBox.getDriveVideoPlayInfo(share_url, fid, "", "剧集/第01集.mp4")
if not (info or {}).get("url"):
    raise RuntimeError("网盘未返回可播地址")
```

- `path` 对 quark / uc 有效，能省掉整树遍历；
- `get_transcode_urls=False` 仅夸克有意义（不转存，只分享直下）；
- 失败不抛异常，返回空 `url`。

### 6.3 `getDriveFileRawUrl(share_url, fid)`（未文档化）

```python
raw = await OmniBox.getDriveFileRawUrl(share_url, fid)
url = raw.get("url")
```

### 6.4 `getDriveShareInfo(share_url)` / `getDriveInfoByShareURL(share_url)`

```python
info = await OmniBox.getDriveInfoByShareURL(share_url)
if (info or {}).get("driveType") in (None, "", "others"):
    return None  # 未知网盘，跳过
```

未识别统一返回 `others` + `未知网盘`，别用 falsy 判断。

### 6.5 `getCloudAuth(drive_type)`（仅 Python，未文档化）

```python
cfg = await OmniBox.getCloudAuth("quark")
```

读取后端配置的网盘凭据（`/cloud-auth`）。需要走自有网盘账号链路时才会用到。

---

## 7. 刮削能力

### 7.1 `processScraping(video_id, keyword, resource_name, video_files)`

```python
candidates = [
    {
        "fid": ep["fid"],
        "file_id": ep["fid"],
        "file_name": ep["name"],
        "name": ep["name"],
        "format_type": "video",
    }
    for ep in episodes
]

try:
    await OmniBox.processScraping(video_id, vod_name, vod_name, candidates)
except Exception as e:
    await OmniBox.log("warn", f"[detail] 刮削失败: {e}")
```

- SDK 会把 `video_id` 按 `|` 取第一段，再拼 `context.sourceId`；
- `detail()` 与 `play()` 的 `video_id` 前缀必须一致；
- 归一化后为空会抛异常。

### 7.2 `getScrapeMetadata(video_id)`

```python
metadata = await OmniBox.getScrapeMetadata(video_id) or {}
scrape_data = metadata.get("scrapeData") or {}
mappings = metadata.get("videoMappings") or []
```

无数据返回 `{"scrapeData": None, "videoMappings": None}`，不抛异常。

### 7.3 `processDriveScraping` / `getDriveMetadata`（兼容别名）

把 `share_url` 当 `video_id` 传，等价于上面两个方法。新脚本直接用通用名。

---

## 8. 播放辅助

### 8.1 `sniffVideo(url, headers=None)` → `{"url", "header"}`

```python
sniffed = await OmniBox.sniffVideo(target_url, {"Referer": BASE_URL + "/"})
if (sniffed or {}).get("url"):
    return {"urls": [{"name": "播放", "url": sniffed["url"]}], "flag": flag, "header": sniffed.get("header") or {}, "parse": 0}
```

需要带 Playwright 的完整运行环境。

### 8.2 `getVideoMediaInfo(play_url, headers=None)`

```python
media_info = await OmniBox.getVideoMediaInfo(play_url, {"Referer": BASE_URL + "/"}) or {}
fmt = media_info.get("format") or {}
duration = int(float(fmt.get("duration") or 0))
```

失败返回 `{}`。只在拿到可播地址后调用。

### 8.3 `getDanmakuByFileName(file_name)` → `list[{name, url}]`

```python
danmaku = await OmniBox.getDanmakuByFileName(file_name)
```

失败返回 `[]`。

---

## 9. 缓存能力

```python
import hashlib

def short_key(prefix: str, raw: str) -> str:
    return f"{prefix}:{hashlib.md5(str(raw).encode()).hexdigest()}"


async def get_cached_json(key):
    try:
        return await OmniBox.getCache(key)
    except Exception:
        return None


async def set_cached_json(key, value, ttl=3600):
    try:
        await OmniBox.setCache(key, value, ttl)
    except Exception as e:
        await OmniBox.log("warn", f"[cache] 写入失败 {key}: {e}")
```

- **key ≤ 512 字节**，value 单条 JSON ≈ **512KB**；
- 按 `sourceId` 隔离，缺 `sourceId` 抛异常；
- 省略 `exSeconds` = 永不过期；
- 不要缓存 Cookie / Token。

---

## 10. 没有的部分

- 影视库三件套（`upsertMediaItems / listMediaItems / getMediaItem`）**仅 JS SDK 提供**，Python 侧没有对应方法。
- AES / md5 / base64 辅助方法已下线，用标准库或 `@dependencies` 装第三方包。

---

## 11. 通用开发建议

```python
EMPTY_LIST = lambda page=1: {"page": int(page or 1), "pagecount": 0, "total": 0, "list": []}
EMPTY_DETAIL = {"list": []}

def empty_play(flag="play"):
    return {"urls": [], "flag": flag}
```

- 每个 handler 先 `await OmniBox.log("info", ...)` 打入口参数；
- 统一 `request_json` / `fetch_html` 封装，别到处重复；
- `try/except` 兜底成空结构，**别让异常冒到 Runner**（Runner 收到异常会整次调用失败、退出码 1）；
- `play()` 保持无副作用；`context.get("prefetch")` 为真时跳过所有写操作。
