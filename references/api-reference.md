# API 参考 | OmniBox

> 基于最新官方文档同步：
> https://omnibox-doc.pages.dev/spider-development/api-reference

## 接口概览

爬虫源**按需实现** 5 个方法：

| 方法 | 说明 | params 主要字段 | 返回值 |
|---|---|---|---|
| `home` | 获取首页数据 | `{}` | 分类列表与推荐视频 |
| `category` | 获取分类数据 | `{ categoryId, page, filters? }` | 分页视频列表 |
| `detail` | 获取视频详情 | `{ videoId }` | 视频详情信息 |
| `search` | 搜索视频 | `{ keyword, page?, quick? }` | 搜索结果列表 |
| `play` | 获取播放地址 | `{ playId, flag? }` | 播放地址信息 |

推荐签名统一为：
```js
(params, context)
```

## 请求上下文 `context`

```ts
interface RequestContext {
  baseURL?: string;
  headers?: Record<string, string>;
  sourceId?: string;
  from?: string; // web(default) | tvbox | uz | catvod | emby
  prefetch?: boolean; // 预热调用；true 时脚本必须跳过写缓存等副作用
}
```

说明：
- `baseURL`：拼装绝对链接时可用
- `headers`：客户端请求头，可透传到第三方接口
- `sourceId`：当前爬虫源 ID；`getSourceCategoryData`、`getCache/setCache/deleteCache`、`processScraping`、`getScrapeMetadata` 都会从 context 读它，**缺失时这些方法直接抛错**
- `from`：调用端，未传时默认为 `web`
- `prefetch`：后端预热任务专用标记。为 `true` 时**不要写缓存、不要写任何用户可见状态**；`play` 等解析方法必须无副作用

调用端标识统一从 `context.from` 获取，不通过 `params` 传。旧的 `addPlayHistory` 已从 SDK 移除，播放记录由宿主自动写。

---

## `home`

作用：
- 返回首页分类与推荐列表

返回结构：
```ts
{
  class: Array<{ type_id: string; type_name: string }>;
  list: VodItem[];
  filters?: Record<string, any[]>;
  banner?: BannerItem[];
}
```

备注：
- `filters`：可选，按分类 ID 提供筛选器
- `banner`：可选，首页轮播数据
- 换句话说，`home()` 不只是 `class + list`，首页型源还应考虑是否需要 `filters / banner`

---

## `category`

入参：
- `categoryId: string`
- `page: number`
- `filters?: object`

返回：
```ts
{
  page: number;
  pagecount: number;
  total: number;
  list: VodItem[];
}
```

---

## `detail`

入参：
- `videoId: string`

返回：
```ts
{
  list: [
    {
      vod_id: string;
      vod_name: string;
      vod_pic: string;
      vod_content?: string;
      vod_director?: string;
      vod_actor?: string;
      vod_area?: string;
      vod_year?: string;
      vod_remarks?: string;
      vod_douban_score?: string;
      type_name?: string;
      vod_play_sources?: PlaySource[];
    }
  ]
}
```

### `PlaySource`
```ts
interface PlaySource {
  name: string;
  episodes: Episode[];
}
```

### `Episode`
```ts
interface Episode {
  name: string;
  playId: string;
  size?: number;
  episodeName?: string;
  episodeOverview?: string;
  episodeAirDate?: string;
  episodeStillPath?: string;
  episodeVoteAverage?: number;
  episodeRuntime?: number;
}
```

说明：
- TMDB 剧集信息可从刮削元数据里补充
- 常规详情页推荐返回 `vod_play_sources`

---

## `search`

入参：
- `keyword`
- `page?`
- `quick?`

返回：
```ts
{
  page: number;
  pagecount: number;
  total: number;
  list: VodItem[];
}
```

备注：
- `quick` 用于快速搜索场景
- UZ 端支持 `search: 1` 这类专用行为字段
- 如果有 `total`，建议与 `page / pagecount / list` 一起返回，别只返回分页号和列表

---

## `play`

入参：
- `playId`
- `flag?`

推荐返回格式：
```ts
{
  urls: Array<{ name: string; url: string }>;
  flag: string;
  header?: Record<string, string>;
  danmaku?: Array<{ name: string; url: string }>;
  parse?: number;
}
```

### `parse`
- `0`：直链（m3u8 / mp4 等）
- `1`：需要客户端嗅探

注意：
- `parse = 1` **仅 ok影视 app 有效**
- 其他客户端通常忽略该字段

### 兼容格式
也兼容：
- `url: string`
- `url: string[]`
- `url: { values, position }`

---

## 数据模型

### `VodItem`
```ts
{
  vod_id: string;
  link: string;      // 豆瓣详情链接，格式 https://movie.douban.com/subject/{vod_id}
  vod_name: string;
  vod_pic: string;
  type_id: string;
  type_name: string;
  vod_remarks?: string;
  vod_year?: string;
  vod_douban_score?: string;
  vod_subtitle?: string;
  vod_tag?: string;
  search?: number;
}
```

特殊字段：
- `link`：新版文档把它列为**必填**（豆瓣详情链接）。老脚本缺这个字段通常还能跑，但新写的源建议补齐，便于宿主做影视库关联/刮削匹配。
- `vod_tag: "folder"`：目录项，点击进入子目录而不是播放页
- `search: 1`：UZ 专用，点击时执行搜索

### `BannerItem`
```ts
{
  title: string;
  subtitle?: string;
  backgroundImage: string;
  genre?: string;
  actors?: string;
  description?: string;
}
```
