# OmniBox Spider 实战经验沉淀

## 1. 最小改动原则
- 优先复制副本修改，不动源文件。
- 一次只解决一个明确目标，避免顺手重构。
- patch 前先确认锚点存在，避免错插到 search/catch 等错误位置。

## 2. 静态检查只是最低标准
- JavaScript 修改后必须执行 `node --check`。
- Python 修改后必须执行 `python3 -m py_compile`。
- 语法通过不等于运行正确，关键路径仍要额外核对。

## 3. 关键路径回归检查
改完后至少核对：
- `detail()` 能正常返回 `list: [vodDetail]`
- `vod_play_sources` 在有数据时不为空
- `play()` 能返回 `urls`
- 关键环境变量没有被误删（如 `SOURCE_NAMES_CONFIG`）

## 4. const / let 风险检查
- 可能被重新赋值的变量必须用 `let`，不要误用 `const`。
- 尤其注意：`playSources`、`vodName`、`vodPic` 这类后续可能会改写的变量。

## 5. 排序逻辑规则
- 排序只放在 `detail()` 最终返回前。
- 只改 `playSources` 顺序，不改 `episodes` 结构。
- `DRIVE_ORDER` 只用于详情页线路排序，不用于搜索结果排序。

## 6. 并发策略
- `play()` 可把 `getDriveVideoPlayInfo` 与元数据/弹幕链路并行。
- `detail()` 不要全量 `Promise.all` 打满，默认改为限并发（建议 4）。
- 元数据/弹幕失败不应阻塞播放主链路。

## 7. 交付说明模板
每次交付至少说明：
1. 修改的文件路径
2. 版本号变更（旧 -> 新）
3. 静态检查结果
4. 本次核心改动点

## 8. Web 播放链路优先保 URL 完整性
- 对 OmniBox 网页端（`context.from === "web"`）的播放链路，优先保证 URL 在 `detail -> play -> 前端` 全链路中完整透传。
- 如果播放页 / 直链自带 `&token`、`&time`、`sign` 等查询参数，不要裸传到 `playId`；优先先编码（如 `encodeURIComponent`），到 `play()` 再解码。
- 网页端很多时候不会可靠消费返回的 `header`；因此不要把 `Referer` / `UA` / `Origin` 当成 web 端成功播放的唯一前提。
- 若站点强依赖 header，优先考虑：
  1. 在 `play()` 内提前解出最终可播直链
  2. 或走代理层，而不是假设网页端一定会带你返回的 header

## 9. GitHub PR 正文排版
- 用 `gh pr create` / `gh pr edit` 传正文时，必须使用真实换行，不要把字面 `\n` 当成换行提交。
- 稳妥做法：优先写入临时 `.md` 文件，再用 `--body-file` 传给 GitHub CLI。
- PR 标题、正文、交付说明默认优先中文，除非上游仓库明确偏好英文。
- 如果是修 OmniBox 播放链路问题，PR 描述里要明确写清：问题出现在哪一段链路（如 `playId` 透传 / web header / 嗅探兜底），避免 reviewer 只看到“修播放”这种过于笼统的表述。

## 10. 新站分类 code 不要靠猜
- 新写 OmniBox 站点源时，首页 `class.type_id` 与分类请求里的 `topCode`，优先从站点前端真实菜单 / 路由 / hydration 数据里抠出真实 code，不要凭经验猜 `tv`、`anime` 这类通用名字。
- 真实站点经常使用自己的 code，例如这次乌云影视实际用了 `tv_series` 与 `animation`；如果误写成 `tv` / `anime`，分类接口会直接 500。
- 遇到分类报错时，先排查：`class.type_id` 是否与站点前端真实 code 一致，其次再看 filters 和分页参数。

## 11. filters 返回结构要贴 OmniBox 前端预期
- 当前 OmniBox 前端里，首页 `filters` 更稳妥的返回格式仍是：`分类ID -> 筛选项数组`。
- 每个筛选项对象优先按常见形状返回：`{ key, name, init, value: [{ name, value }] }`。
- 如果误返回成自定义对象嵌套（例如 `movie: { sort: [...] }` 这种 object-of-objects），前端可能在点击分类时直接报错。

## 12. 新增采集源后别忘了补更新注解
- 参考 OmniBox-Spider 现有采集源头注释，新源落仓时除 `@name` / `@version` 外，通常还应补齐：`@author`、`@description`、`@downloadURL`。
- 若该脚本准备走远程更新链路，`@downloadURL` 应直接指向仓库主分支可访问的原始文件地址。
- 这类注释补齐后，记得同步升级 `@version`，并重新做一次静态检查。

## 13. filters 结构必须贴前端实际字段名
- OmniBox 首页 `filters` 默认不要再返回旧习惯的 `{ n, v }` 子项；前端更稳妥的结构是：
  - `[{ key, name, init?, value: [{ name, value }] }]`
- 如果筛选项子项仍返回 `{ n, v }`，前端可能只渲染骨架屏、点击后不生效，表面看像“接口错了”，本质是字段名不匹配。
- 写新源或改旧源时，筛选结构至少检查：
  1. 顶层是 `分类ID -> 筛选项数组`
  2. 每个筛选项有 `key / name / value`
  3. `value` 内每项优先用 `name / value`
  4. 需要默认值时补 `init`

## 14. play 日志要把入参、页面地址、嗅探地址、出参打全
- 只记“播放失败”没用，排查 OmniBox 播放链路时，日志至少应覆盖：
  1. `play()` 原始入参
  2. 解析后的 `playId`
  3. 实际回读的播放页 URL
  4. 参与 `sniffVideo` 的 URL 与 headers
  5. `sniffVideo` 返回值
  6. 最终返回给宿主的出参（成功 / 回退）
- 对这类前端混淆站，如果短期还没解出直链，至少先把日志打全，方便后续继续抠真实播放链路。

## 15. 页面型站点首页解析不要误抓 SEO 区和轮播区
- 像 `dyrsok.com` 这种页面型站点，首页往往同时存在：
  1. `sr-only` / `webseo` 之类给搜索引擎看的链接区
  2. 轮播 `swiper` 推荐区
  3. 真正的列表卡片区
- 如果首页抓取条件写得太宽（例如只按 `a[href*="/dyrscom-"][title]` 抓），就很容易出现：
  - 标题对了，但图片抓成轮播图
  - 描述抓成整块轮播文案
  - 首页卡片信息串台
- 更稳妥的做法：优先锁定真实卡片容器（如带 `data-url/data-cover` 的卡片锚点），不要把 SEO 链接和轮播推荐混进普通列表。

## 16. 详情页线路必须逐线路抓，不能只看当前默认线路
- 页面型站点详情页经常同时展示：
  - 线路 tab（如 `originTabs`）
  - 当前线路对应的选集列表
- 只解析当前页面默认线路，会导致：
  - `vod_play_sources` 实际只有 1 条
  - 但备注或 tab 看起来有很多线路
  - 用户体感就是“线路不对 / 线路缺失”
- 正确做法：先解析所有线路入口，再逐个进入各 `?origin=...` 页面分别提取选集，最后按真实线路构造 `vod_play_sources`。

## 17. 播放链要返回“最终可播地址”，不是中转第一跳
- 对一些页面型站点，详情页里提取到的 `/api/m3u8?...` 可能只是第一跳，不是最终可播地址。
- 例如 `dyrsok.com` 的实际链路是：
  1. 页面内 `/api/m3u8?origin=...&url=...`
  2. 302 到 `box.dyrs.com.de/api/super?id=...`
  3. 主清单里再给出 `/api/m3u8?id=...&raw=1`
  4. `raw=1` 子清单才更接近最终可播 m3u8
- 经验：`play()` 里不要满足于“拿到第一跳就返回”，应继续探测到更稳定、更终态的清单地址。

## 18. URL 放进请求头前必须先编码，尤其是中文参数
- 如果 `playId.page` 里包含中文查询参数（如 `?origin=超级线路`），直接把原始 URL 放进请求头 `Referer`，Node/宿主可能报：
  - `Invalid character in header content ["Referer"]`
- 经验：任何会写进 header 的 URL，先做 `encodeURI(...)` 再使用；否则排查时会误以为是站点接口失效，实际是 header 非 ASCII 崩了。

## 19. 播放记录不要再由脚本写（`addPlayHistory` 已移除）
- 新版 SDK 删掉了 `OmniBox.addPlayHistory()`；旧脚本里所有相关代码（含 `getVideoMediaInfo()` 补 `totalDuration` 的写法）都应删除。
- 现在记录由宿主根据 `play()` 的返回值产生，所以想让"最近观看"正确显示，重点变成**把 `play()` 返回干净**：
  - `urls` 数组里给出可播地址；
  - `flag` 与 `detail()` 的线路名对应上；
  - `header` 带够（`Referer / Origin / User-Agent`）；
  - 分集 `playId` 里保留能反查片名/集名的轻量信息。
- 想在首页做"最近观看"入口，改读：`OmniBox.getSourceCategoryData("history", 1, 12)`（只读）。
- 如果用户反馈"记录有了但封面空/标题不准"，先查 `detail()` 返回的 `vod_name` / `vod_pic` 是否正确，而不是再去找写历史的接口。

## 20. 仓库 openclaw 分支是正式真源，工作区副本只是调试镜像
- 当一个脚本同时存在：
  - 仓库正式文件（如 `OmniBox-Spider/影视/采集/xxx.js`）
  - 工作区调试副本（如 `workspace/xxx.omnibox.js`）
- 一旦已经决定“后续以 openclaw 分支继续开发”，就必须明确：
  - `openclaw` 仓库文件 = 最新正式真源
  - 工作区副本 = 临时镜像 / 调试稿
- 否则很容易出现：
  - 先在工作区改了一轮
  - 又在仓库里改了一轮
  - 两边逻辑逐渐分叉
  - 最后连自己都分不清哪份才该提交 PR
- 经验：如果要保留工作区镜像，也应始终从仓库正式文件反向同步出去，而不是反过来用调试稿覆盖正式文件。

## 21. 磁力详情响应优先贴近传统 `vod_play_url` 思路，不要过度包装
- 这次木卡库 / 不太灵磁力调试里，一个关键教训是：用户给的参考爬虫（`粉妹自用磁力版.磁力 种子 电驴.php`、`四合一磁力.php`、`磁力.py`）真正值得对齐的是**详情接口磁力响应格式**，不是先入为主地改播放逻辑。
- 对磁力 / 电驴 / 种子类详情分集，优先保持：
  1. `episodes[].name` 用短标题（如 `第05集` / `第01-04集`）
  2. `episodes[].playId` 尽量直接放 magnet / ed2k / torrent 链接本身，或极轻量 payload
  3. 不要一上来就在 `playId` 里塞大段 JSON 元数据
- 原因：某些 OmniBox / TVBox 前端对详情选集按钮渲染很脆弱；`playId` 过重、字段过花时，可能出现：
  - 按钮空白
  - 详情能显示但点不动
  - 播放阶段取值异常
- 如果用户明确给了“可用参考爬虫”，先对齐参考爬虫的**详情分集表达方式**，再决定要不要在 `play()` 里额外增强。

## 22. 新建脚本落仓时，`@downloadURL` 和最终仓库路径必须一起收尾确认
- 新建 OmniBox-Spider 脚本时，不能只顾着功能通了就发 PR；落仓前至少再检查一轮：
  1. 最终目录是否正确（如 `影视/磁力`）
  2. 最终文件名是否符合用户要求（例如不带 `.omnibox`）
  3. 头注释是否已补 `@downloadURL`
  4. `@downloadURL` 是否指向 **main 分支最终文件路径**，而不是工作区调试名或旧路径
- 这一步应视为新建源交付清单的一部分，不要等用户提醒后再补。

## 23. 解析源配置优先做成“全局环境变量 + 默认补足 + 按地址去重”
- 如果多个解析型脚本都可能复用同一批解析接口，优先抽成全局环境变量（如 `PARSE_APIS`），不要每个脚本都单独造一套 `MOVIE360_PARSE_APIS`、`XXX_PARSE_APIS`。
- 更稳妥的合并策略是：
  1. 先读环境变量里的解析源
  2. 再补默认内置解析源
  3. 按“解析地址 URL”去重
  4. 若环境变量与默认项重复，优先保留环境变量项（包括其自定义名称）
- 这样既能保留默认可用性，又能让用户在全局层覆盖优先级与线路名称。

## 24. 解析接口返回 HTML 时，不要直接判失败；先嗅探，再回退原生地址交给播放器兜底
- 解析型接口不一定都返回 JSON；很多 `...?url=` / `...?jx=` 接口实际返回的是 HTML 播放页。
- 更稳的处理顺序应是：
  1. 若返回 JSON，且状态成功且有 `url`，按 JSON 结果处理
  2. 若不是 JSON，则对“解析页地址本身”调用 `sniffVideo`
  3. 若嗅探无结果，或 JSON 缺 `url` / 状态失败，则回退原生目标地址，并设置为交给播放器继续解析的模式（如 `parse=1`）
- 这样能避免日志里大量出现 `Unexpected token '<'` 后直接把整条线路判死，同时保留前端/播放器自己的兜底能力。

## 25. 字节系 CDN（douyinvod / byteimg / ixigua）不能带外站 Referer
- 现象：某条线路的剧集点了没反应或一直转圈，后端日志里是
  `代理播放响应 status_code=403`，而上游 URL 形如 `https://v11-cold.douyinvod.com/.../xxx.mp4`。
- 原因：脚本的 `play()` 通常统一返回一套 `PLAY_HEADERS`（带站点 `Referer` / `Origin`），
  但抖音/西瓜这类字节系 CDN 会拒绝带外站 Referer 的请求。
- 实测对照（同一 URL，同一 UA）：

  | 请求头 | 结果 |
  |---|---|
  | 站点 Referer + Origin | 403 |
  | 只有站点 Referer | 403 |
  | 不带 Referer | **200** |

- 正确做法：按**最终播放地址的 host** 决定头，而不是一套头走天下。

  ```js
  const CDN_STRIP_HEADER_RE = /(douyinvod|douyinpic|douyinstatic|byteimg|ibytedtos|ixigua|bytedance)/i;

  function buildPlayHeaders(targetUrl, base = PLAY_HEADERS) {
    const headers = { ...base };
    let host = "";
    try { host = new URL(targetUrl).hostname; } catch (_) {}
    if (host && CDN_STRIP_HEADER_RE.test(host)) {
      delete headers.Referer; delete headers.referer;
      delete headers.Origin;  delete headers.origin;
    }
    return headers;
  }
  ```

- 注意别一刀切：`*.gtimg.com`（腾讯）、普通 m3u8 站外源带站点 Referer 反而是 200，
  只有字节系域名需要剥掉。

## 26. 改完 `play` 脚本没效果，先怀疑后端的结果缓存
- 后端有一张 `spider_play_result_cache` 表，按 `source_id + flag + play_id_hash` 缓存 play 结果，
  TTL 约 **10 分钟**。另外 `spider_script_cache` 也会缓存脚本内容。
- 表现：脚本明明改了（容器里 `grep` 能看到新代码、直接 `node index.js` 跑也是新行为），
  但走 `/api/spider-source/:id/execute` 或前端播放时仍是旧结果。
- 处理顺序：
  1. 保存脚本（`PUT /api/spider-source/:id/script`）刷新脚本缓存
  2. 清播放缓存：`DELETE FROM spider_play_result_cache WHERE source_id='<源ID>';`
  3. 还不行就重启容器
- 调试时请**优先直接跑脚本**验证逻辑，再回头看后端链路，能把“脚本没改对”和“缓存没刷”区分开。

## 27. 把鱼壳（webhtv）Python 源移植到 OmniBox 的三个坑
- 鱼壳源继承 `base.spider.Spider`、handler 是同步的 `homeContent/detailContent/playerContent`，
  OmniBox 是 `from spider_runner import OmniBox, run` + `async def home(params, context)`，
  两者协议不通，必须重写，不能直接复制。
- **建源时 `type` 要写 `python`**（不是 `js`、也不是 `py`）：
  - `type=python` → `scriptPath = <id>/index.py`
  - `type=js` / `type=py` → 仍然是 `index.js`，会把 Python 代码交给 node 跑，
    报 `SyntaxError: Invalid or unexpected token`。
- 依赖能省则省：
  - `requests` 可以换成标准库 `urllib.request`，少一次 pip 安装
  - `pycryptodome` 可以内嵌一份纯 Python AES-CBC（约 120 行），
    省掉 pip 依赖；注意 S 盒要用 GF(2^8) 指数/对数表生成，
    手写 256 项常量很容易错一位导致全部解不出来
- `playId` 建议 `base64url(json{u,f,n})`，别把带 `?&=#` 的长 URL 直接塞进去。
- 日志必须包一层 try/except：缺 `sourceId` 时 `OmniBox.log` 会抛错，
  在 except 分支里再抛一次会把整条请求打挂。

## 28. 判定「播放地址是否绑定请求方 IP」，以及绑了之后怎么办
- 现象：脚本在服务器上跑得好好的，服务端自检清单 200、首片 206，但换个网络
  的客户端（TVBox、异地朋友）一放就 403，而且**是 403 不是超时**。
- 403 本身就有信息量：说明 DNS 解析、TCP、TLS 都通了，是 CDN 在**鉴权阶段**拒绝。
  这时不要再去查 DNS、证书、播放器内核，方向错了。

### 怎么确认是不是绑 IP
1. 拿一条刚签发的播放地址，在上面那台服务器上 `curl` → 通常 200。
2. 换一个出口 IP 请求**同一条地址**（另一台机器、第三方抓取服务、代理都行）：
   - 也是 200 → 不绑 IP，问题在别处（Referer/UA/播放器）
   - 403 → **基本可以确定绑了申请时的源 IP**
3. 加一个随机 query 参数（`&_=123456`）破掉 CDN 缓存再试：
   如果仍然 200，说明不是"缓存命中侥幸通过"，而是回源鉴权真的认这个 IP。

### 顺手把出口 IP 抓出来
这类 CDN 的 403 页面会打印诊断信息，例如：

```
Node_info: 4137-CACHE17
Client_ip: 122.192.198.135
Forbid_code: 000203
Hit-status: MISS
```

把 auth_key 的签名段改坏（比如 md5 换全 0）再请求一次，就能读到
`Client_ip` —— 那是「CDN 眼里的我们」，可以直接和家庭/机房公网 IP 对比，
一眼看出服务器是不是挂了代理、走了多拨或 VPN。

注意 `urllib` 的 `HTTPError` 分支要读够长度（403 页 ~350 字节），
只读几十字节会把 `Client_ip` 截掉，导致探测一直"失败"。

### 绑了 IP 之后能做什么
| 场景 | 结论 |
|---|---|
| 脚本服务器与播放设备**同公网出口**（都在家里） | 直连可用，检查是否被代理/VPN 改过出口 |
| 服务器在云上、播放设备在别处 | **直连物理不可行**，改语言、改 header、改 DNS 都无效 |
| 只想省服务器带宽 | 也没有中间方案：清单和 `.ts` 分片通常**都**绑 IP，「清单走服务器、分片直连」同样 403 |

真正的出路只有两条：把脚本部署到播放设备所在网络，或让那个网络里有一台
「地址签发器」（用它的 IP 调站点 API 拿地址）。否则只能走 proxy-play 代理，
代价是流量全部经过服务器。

### 另外两个细节
- 有的站点是显式的：分片参数里直接带 `whip=<请求方IP>`（一眼可见）；
  有的隐式的：IP 被 hash 进 `auth_key` 的签名段，看不出来，只能靠对比测试。
- 别指望伪造 `X-Forwarded-For` 骗过站点：多数站点前置 WAF 会直接 403。

## 29. 「封面不显示」八成是详情页指向了 tmdb 原图，不是选择器写错了

排查顺序反了会白改好几轮。正确的第一步是**把 home / search / detail 三处返回的
`vod_pic` 都打印出来对比**，而不是先怀疑 `img` 选择器。

真实案例（4KVM）：

| 位置 | 来源字段 | 实际地址 | 结果 |
|---|---|---|---|
| home / search | `<img data-src>` 竖版图 | `gimg0.baidu.com/gimg/...&src=4kvm.staticimgjs.org/...-300x450.jpg`（28KB） | 正常 |
| detail | `.video-player[data-poster]` | `gimg0.baidu.com/gimg/...&src=image.tmdb.org/t/p/original/xxx.jpg`（821KB） | **国内连不上** |

`image.tmdb.org` 在国内基本不可达，且 `original` 是原图。表现就是「列表有封面、
点进详情页一片空白」。

### 修法
1. **封面优先取站点自己的图**。CMS 站点的 `<meta property="og:image">` 通常就是站点
   压缩过的竖版海报，比播放器的 `data-poster` 小一个数量级，还免翻墙。
2. **刮削结果不要无条件覆盖封面**。`scrapeData.posterPath` 是 tmdb 的，直接盖上去
   等于把好不容易拿到的本地图换成墙外图。判定条件写成「站点没图 / 站点图来自 tmdb
   才换」，并且用 `w500` 而不是 `original`。
3. **外链图统一走国内代理**。百度图床可以直接当中转：
   ```
   https://gimg0.baidu.com/gimg/app=2001&n=0&g=0n&fmt=jpeg&src=<去掉scheme的地址>
   ```
   注意这个格式**没有 `?`**（`/gimg/app=2001&...`），补成 `/gimg?app=2001` 反而 404。
4. 给个开关：`K4VM_IMG=baidu`（默认，保留站点代理）/ `direct`（还原源站直连，
   海外环境更快）。

### 顺带一个选择器陷阱
`.absolute.bottom-0 span` 匹配的是「class 含 `absolute bottom-0` 的元素的**后代** span」，
如果那个 span **自己**就是 `absolute bottom-0`，选不中，于是掉到下一档兜底
`.year, .date, span`，把整段剧情简介当成 `vod_remarks` 抓了回来 —— 卡片角标被撑爆。
改成 `span.absolute.bottom-0`（匹配自身），再加一句「超过 20 字丢弃」，问题消失。

## 30. 播放地址由 wasm 签名时，别急着上嗅探——把站点的 wasm 拿过来自己跑

`sniffVideo` 每次要开无头浏览器、等页面把播放器拉起来，实测单集 **9~12 秒**。
但只要播放页是「前端算签名 → fetch 接口拿 m3u8」，这段逻辑就能搬到 Node 里，
耗时直接降到 **0.3~0.7 秒**。

### 怎么判断站点属于这一类
播放页 HTML 里搜不到 `m3u8`，但 `app.ultra.min.*.js` 这类文件里能看到：

```js
const s = await import('/static/wasm/nbmovie_wasm.js');
await s.default();
const url = s.build_play_url(dataid, secretKey, quality, userlink);
fetch(url);
```

关键词：`build_play_url`、`wasm`、`<link id="wasm-cfg" data-js=... data-bg=...>`。
有这些就说明地址是本地算出来的，不是服务端渲染的。

### Node 侧落地要点（OmniBox 跑 Node 20，都支持）
1. **wasm-bindgen 产物是 ESM，要存成 `.mjs` 才能 import**
   ```js
   fs.writeFileSync(path.join(dir, "mod.mjs"), jsText);
   const mod = await import(pathToFileURL(jsFile).href);   // CJS 里动态 import ESM 可以
   ```
2. **init 必须喂字节**：Node 没有 `file://` 的 fetch，直接 `init()` 会报
   `TypeError: fetch failed → not implemented... yet...`
   ```js
   await mod.default({ module_or_path: new Uint8Array(fs.readFileSync(wasmFile)) });
   ```
3. 文件缓存到 `os.tmpdir()`。首次要下 60~70KB，之后同进程/同容器直接复用。
   冷启动那次 ~2.4s，热的一次 0.5s。

### 两个必须踩的坑
- **签名参数里的 userlink 是每次渲染页面现生成的**。同一秒连抓两次都不一样；
  更坑的是**首页里它是 `'0'`**，拿去签出来的地址必然 `401 请提供访问令牌`。
  所以：detail 阶段解析 `nav[x-data]` 里的 `userlink:'...'` 存下来，
  过期或取到 `'0'` 就重新抓一次 `/play/xxx` 页面再签。
- **签出来的直链多半在抖音系 CDN**（`oss.douyinbit.com` / `v26-default.douyinvod.com`）。
  实测同一地址：无 Referer → `206 video/mp4`，带 `Referer: https://www.4kvm.net/` → `403`。
  这就是老问题 #25 的变种，返回 `header` 时必须按 host 把 Referer/Origin 剥掉。

### 结构上怎么留后路
直取失败（缺 dataid、wasm 拉不到、userlink 拿不到）一律 `return null`，
让 play 继续走原来的 iframe/video/嗅探链路。嗅探慢但不会坏，
所以直取只当加速层，不要当唯一路径。

## 31. 网页端有封面、TVBox 没封面：多半是 URL 里的 `&` 被转义成了 `\u0026`

这个坑很容易查错方向——图片 URL 明明 curl 是 200，网页端也显示得好好的，
就以为问题在客户端网络，反复换 UA、换 Referer、换 CDN 都没用。

### 真正的原因
OmniBox 后端是 Go，`json.Marshal` 默认开 `SetEscapeHTML(true)`，会把 JSON
字符串里的 `&` `<` `>` 转成 `\u0026` `\u003c` `\u003e` 再输出。

实测 TVBox 接口原文：

```json
"vod_pic":"https://gimg0.baidu.com/gimg/app=2001\u0026n=0\u0026g=0n\u0026fmt=jpeg\u0026src=..."
```

- 走标准 `JSON.parse` 的客户端会把 `\u0026` 还原成 `&`，图片正常；
- 但 **OmniBox 自己的网页端、以及部分 TVBox 内核，会把 `\u0026` 原样带进图片请求**
  → 请求不存在的路径 → 404 → 没有封面。

实测症状既可能是「网页有图、电视端空白」，也可能是**网页端和电视端都没图**
（4KVM 就是后者）。所以别看到网页端也没图就排除这个原因，直接去拉原始 JSON 看。

### 排查动作（一步到位）
别看 execute 接口，要看**客户端真正调的那个接口**的原始文本：

```bash
curl -s "http://<host>/api/tvbox/source/<id>?conv=2&token=<tk>&ac=detail&ids=<vid>" \
  | grep -o '\\u0026' | head
```

有 `\u0026` 就是这个问题。

### 解法
**让交给客户端的 URL 不带 `&`。**

- 封面：优先用源站的直链（`https://<cdn>/uploads/xxx.jpg`），
  别用 `?url=`、`?src=` 这类带参数的图片代理。
  就算代理是「国内更快」的百度图床，也只在网页端有意义，TVBox 端一律废掉。
- 播放地址：同理。`...?name=viptvmv.mp4`（单参数）没事，
  一旦出现 `?a=1&b=2`，TVBox 播放也会失败——遇到这种源要提前发现，
  必要时换签名方式或换线路。
- 实在只有带参地址可用时，给个环境变量开关，默认走无参版本。

### 顺带一个替代方案
`https://wsrv.nl/?url=<去掉scheme的地址>` 只有 `?` 没有 `&`，
同样能躲开转义，但它是境外服务，国内速度一般，只适合兜底。

### 还顺手学到
排查这类问题，用容器里自带的 Python playwright
（`/root/.cache/ms-playwright`）直接登录 OmniBox 截图，
比反复猜前端逻辑快得多——能一眼看出是「服务端没给图」还是「客户端没显示」。

### 补充：绕不开时优先保「域名可达」
如果只有带 `&` 的地址能用（比如百度图床），而源站是不带 `&` 的境外域名，
**优先选源站直连**。4KVM 的 `4kvm.staticimgjs.org` 其实是韩国首尔机房
（115.89.129.249 / MOACK），国内实测能访问，比「百度图床但 URL 被转义坏」更可靠。
顺带记两个试过不行的绕法：
- 把 `&` 编码成 `%26`：百度图床直接 **400**
- 改用 path 风格 `/gimg/app=2001/src/...`：**404**

## 32. 「能播，就是加载很久」：先量 sniffVideo 的真实耗时，别无条件 await

用户说「不能播放」和「要等很久才播」经常是同一条链路：**服务端嗅探超时失败 →
回退 `parse=1` 把页面交给客户端 webview → 客户端自己把播放器跑起来**。
这时候 `play()` 里那次 `await OmniBox.sniffVideo(...)` 是纯损耗，白等 10 秒。

### 定位动作（按顺序，别猜）
1. 用 `POST /api/spider-source/{id}/execute` 对 `play` 计时。
   实测 NO视频：`10.5s`，返回 `{"parse":1, "urls":[{"url": 详情页}]}` → 嗅探没命中。
2. 直接打后端嗅探接口确认是不是固定超时：
   `POST /api/spider/omnibox/video/sniff {"url":..., "headers":{...}}`
   （SDK 的 `API_BASE_URL` 默认是 `http://localhost:7023/api/spider/omnibox`，
   所以外部完整路径是 `/api/spider/omnibox/video/sniff`，不是 `/api/video/sniff`）。
   实测详情页 `9.1s` 返回 `嗅探失败: run video_sniffer: exit status 1`，iframe 页同样失败。
3. 用容器内 Python Playwright 打开真实页面、监听媒体请求，
   确认「服务端嗅探不到」到底是链路太深，还是压根需要用户交互。

### 多层 iframe + wasm 完整性校验的站点（NO视频 novipnoad.us 实例）
真实链路是三层，且**每一层都依赖浏览器环境**：

```
详情页 (window.playInfo = {vid, pkey})
  └─ iframe player.<host>/v1/?url=&pkey=&ref=   ← 响应 Set-Cookie: device=...
       └─ iframe /v1/player.php?id=&device=      ← 需要上面那个 cookie
            └─ enc-vod.<host>/ftn/<id>.js?ckey=&ref=&ip=&time=
                 → var videoUrl = JSON.decrypt("...密文...")
                 → 最终 mp4（ftn.qq.com）
```

- 外层 iframe 页内嵌一段**每次请求动态生成**的 wasm（约 539B），
  用浏览器指纹的 djb2 hash 解密出 `ckey`(40 hex) 和 `vkey`(JSON: ref/ip/time)；
  `vkey` 只在 `window.top !== window.self` 时写入 sessionStorage。
- 所以**把播放器 iframe 页单独作为顶层打开会直接不出流**（Playwright 实测
  连 `player.php` 都不请求），`parse=1` 的目标必须是**站点详情页**，
  由站内 `play.js` 自己去嵌 iframe。
- 详情页自身也不会自动播（DOM 加载完没有 iframe），需要客户端点一次播放按钮。
- `JSON.decrypt` 的混淆实现藏在 `/js/jquery.min.js` 尾部；
  密文每次随 ckey 变，但解出的 mp4 直链在同一集内是稳定的。
  纯 HTTP 直取要同时复现 wasm 指纹 + 混淆解密，成本远高于收益 → 不要硬啃。

### 可直接复用的规则
- ⚠️ **本条结论已被 #33 推翻，不要照抄**：「先证明 sniffVideo 能成功，
  若实测失败就直接删掉、只留 `parse=1`」会导致彻底无法播放。
  本地嗅探失败可能只是本地出口 IP 被站点拒绝，且 `parse=1` 并非直链等价物。
  正确做法见 #33。保留原文仅作为排查过程记录。
- 播放器页带 `top !== self` 之类判断时，`parse=1` 的目标选**站点详情页**，
  不要选剥出来的 iframe 地址（页面更轻但鉴权不成立）。
- `parse=1` 只对支持 webview 嗅探的端有效；TVBox 这类端会把它当视频地址，
  这点要在交付说明里写明，别笼统说「修好了播放」。
- 顺带优化详情：列表页已知 href，就把真实分类路径编进 `vod_id`
  （`154887-tv-korea`），`detail()` 一次请求命中；
  旧版纯数字 id 用 WordPress `/?p=<id>` 的 301 直达兜底
  （实测 301 → `/tv/korea/154887.html`），最后才回退分类轮询。
  详情页成功判定要加 `window.playInfo|multilink-btn` 之类的特征校验，
  否则 404 页会被当成成功、分集恒为 0。
- 实测收益：detail 3.2s → 1.1s，play 10.5s → 0.0s，韩剧 8 集/电影 1 集/旧 id 均正常。

## 33. 【推翻 #32】删掉 sniffVideo 前，必须确认「删掉后客户端仍能起播」

#32 实测 NO视频 的 `sniffVideo` 固定 9s 失败，于是 v1.2.0 把它删掉，
只回 `parse=1 + 详情页`，`play()` 从 10.5s 降到 0.0s。
用户反馈 **「彻底无法播放」** —— 优化把唯一能用的播放链砍了。

### 为什么判断错了
- 本地容器（colima）出口 IP 被站点判异常：点击分集后
  `enc-vod...js?ckey=...` 返回 404、播放器显示 `bg_404.png`，
  所以本地 sniffVideo 100% 失败。**本地失败 ≠ 用户 NAS 失败**。
- 站点详情页**不自动起播**：DOM 加载完 frames 只有详情页自身，
  必须点 `.multilink-btn` 才插入 `player.<host>/v1/` iframe。
  服务端 sniffVideo 是全新 headless context，不会去点按钮，
  对「多集剧」本来就抓不到流；但 `window.playInfo` 带 `vid` 的**电影页**
  可能自动起播，用户侧 sniff 成功拿到直链。
- `parse=1 + 详情页` 只有在客户端 webview 真的加载页面、并让用户点击时才成立；
  网页端 / TVBox 常把 URL 直接当视频地址 → 一个 HTML 页面当然播不了。
  所以「删嗅探只留 parse=1」不是等价替换。

### 正确做法
1. 先量清「慢」来自哪里：宿主 `video_sniffer.py` 是
   `page.goto(domcontentloaded) + asyncio.sleep(wait_after_load_ms=6000)`
   再加候选探测，约 9 秒是**宿主固定开销**。SDK 签名只有
   `sniffVideo(url, headers)`；后端实测忽略 `timeout_ms` / `wait_after_load_ms`
   （传 2000 仍耗时 8.96s），脚本侧无法缩短。
2. 不要因为本地嗅探失败就删主链路。保留三段式：
   `sniffVideo → 成功 parse=0 直链 → 失败 parse=1 页面回退`。
   只做无损优化（传 UA/Referer、补耗时与结果日志、detail 走 `?p=` 直达）。
3. 要给用户对比手段，就加环境变量开关（如 `NOVIDEO_SKIP_SNIFF=1`），
   **默认保持已验证可播的行为**，别默认切到未验证的新链路。
4. `vod_id` 这类宿主侧标识别顺手改格式。1.2.0 把纯数字改成
   `154887-tv-korea` 虽让 detail 变快，但给播放/历史链路引入额外变量；
   回退时一并还原纯数字，用 WordPress `/?p=<id>` 301 兜底
   （1.85s，仍快于逐分类试探的 3.2s），并兼容曾发出的带路径 id。
5. 改完 `play()` 要提醒清 `spider_play_result_cache`（约 10 分钟），
   否则旧结果被复用，看起来像「没修好」。playId 结构变化通常能让 key 自动失效。

### 回归实测（v1.2.1，本地 27023 beta）
category 1.45s / vod_id 纯数字；detail 纯数字 1.85s(5集)、
旧格式 id 1.06s(8集)、电影 1.47s(1集)；play 8.78s 后按预期回退 parse=1，
`urls/parse/header` 与 1.0.2 一致。

### 一句话
「嗅探慢」该优化的是等待，不是把嗅探删掉；
在拿到可替代的真实直链之前，`parse=1` 绝不是直链的等价物。

## 34. 同一站点「电影能播、剧集不能播」：自动起播 vs 需要点击，以及 wasm/ckey 完整逆向

NO视频(novipnoad.us) 用户反馈：电影《生化危机》能播，台剧/港剧全部失败。
日志里剧集 `play` 固定 8~9s 后 `嗅探失败: exit status 1` -> 回退 parse=1。

### 根因（Playwright 实测）
- **电影**详情页 `window.playInfo={vid:"ftn-...",pkey:"..."}`（文章正文内联脚本），
  `play.js` 加载后**自动**插入播放器 iframe → 服务端 sniffVideo 能抓到 mp4
  → 返回 `parse=0` 直链，用户端可播（本地实测 9.22s / parse=0 / njc-download.ftn.qq.com）。
- **剧集**详情页 `window.playInfo` 只有 `pkey`，DOM 加载完 frames 只有详情页自身、
  8 秒零媒体请求；必须点 `.multilink-btn[data-vid]` 才创建 iframe。
  宿主 `video_sniffer.py` 只 goto + sleep，**不会点击** → 剧集嗅探 100% 失败。
- 由此反推：该用户端 **只认 parse=0 真实直链**，parse=1 页面（无论详情页还是
  播放器页）都不生效。判断依据：1.2.0 只回 parse=1 时「彻底无法播放」，
  1.2.1 恢复嗅探后电影立刻可播。别再假设客户端有可用 webview。

### 完整播放链与可复现结论
```
详情页 (playInfo{vid,pkey})
  └─ iframe player.<host>/v1/?url=<vid>&pkey=<pkey>&ref=<站内路径>   [Set-Cookie: device]
       └─ iframe /v1/player.php?id=<vid>&device=...                     [postMessage AUTH ckey]
            └─ <script src=enc-vod.<host>/<prefix>/<id>.js?ckey=&ref=&ip=&time=>
                 → var videoUrl=JSON.decrypt("密文")  (或 {"code":403/404...})
                 → {"code":200,"quality":[{"url":"https://njc-download.ftn.qq.com/.../hd.mp4"}]}
```
1. `/v1/` 页内 wasm（539~544B）**每次请求动态生成**，seed 不是固定浏览器指纹值：
   页面里的 CSS 常量也随机。可从 iframe HTML 静态解析：
   `grid-template-columns:(\d+)px 1fr (\d+)px;width:(\d+)px` → 第二段列宽 = W-a-b；
   `?(\d{4,6}):0` 两处 → canvas/proto 常量（83927 / 61934）；
   `width:(\d+)px!important` 第一个 → CSS 检测值。
   `str = "<grid>|<canvas>|<proto>|<css>"`，`seed = djb2(str)`（JS 位运算得到有符号 i32）。
2. Node 里 `WebAssembly.Module(bytes)` + `Instance` 注入 `env.auth(cp,cl,vp,vl)`，
   `run(seed)` 后从 `exports.memory.buffer` 读出 `ckey`(40hex) 和
   `vkey`(`{"ref","ip","time"}`)。**不需要浏览器**。
3. 请求 enc-vod 时 `ckey` 必须 **`.toUpperCase()`**，小写返回 `{"code":403,"msg":"invalid request"}`；
   大写返回 200 + 密文。`ref/ip/time` 用 vkey 里的原值，且要与 iframe 同一次请求配套（时效性）。
4. `JSON.decrypt` 的混淆实现藏在 `/js/jquery.min.js?v=3.3.1_20260613` 尾部约 56KB，
   从 `Function(` 处（`new Function` 语法合法的唯一偏移）往后可在 Node/vm 里 eval 出
   `JSON.decrypt`，但**解密依赖浏览器运行时上下文**（补 location/sessionStorage.vkey/
   navigator/document.referrer/ckey 后仍是乱码，decodeURIComponent 抛 URIError）。
   纯 HTTP 直取因此未采用；不要为了省 9 秒把 56KB 混淆代码内联进爬虫脚本。
5. 把 `/v1/` 播放器 URL 作为 **iframe 子帧**加载（父页面任意 http 页）时全链路可通、
   `videoUrl` 为 code:200；但作为顶层打开、或放在 `data:text/html` 页里（静态/动态/srcdoc
   三种都试过）都 **hits 为空**。宿主 sniffVideo 只做 goto，无法注入 iframe → 包装页方案无效。
   给 sniff 传 Referer header 也救不回来。

### 可复用规则
- 同一站点「电影行、剧集不行」优先查 `window.playInfo` 是否带 vid、页面是否自动起播，
  别只盯 IP/风控。
- 服务端嗅探器不会点击任何 DOM；需要交互才能出流的站点，sniffVideo 必然失败。
- 用 `parse=0` 直链能否拿到，决定客户端能不能播；先用一次「能播」的内容实测
  play 返回的 parse 值，反推客户端能力，再决定要不要保留 parse=1 页面回退。
- 剧集白等 9 秒无意义，可改成每集独立的播放器 URL（主）+ 详情页（备）两条线路，
  并用环境变量（如 `NOVIDEO_MODE=player|page|sniff`）保留回退，默认别动已验证可播的电影链路。
- 彻底解决仍需复现 `JSON.decrypt` 的密钥派生（疑与浏览器环境指纹/页面上下文有关）。

### 补充（v1.3.1）：网页端只认直链，`data:` 包装被 ORB 拦截
- 宿主网页端播放器是 ArtPlayer + hls.js（`/app/static/assets/player-*.js`），
  全文无 `iframe`、无 `parse === 1` 分支，`urls[0].url` 直接进 video/hls。
  所以 parse=1 的页面 URL（详情页或播放器页）在网页端都不会出画面，
  只有 parse=0 的 m3u8/mp4 能播。判断「本端能不能播直链」先看这一点。
- 想用 `data:text/html` 页面当父容器嵌播放器 iframe：iframe 实际落在
  `chrome-error://chromewebdata`，被 Chrome ORB 拦截（opaque origin 的跨域 HTML），
  与 Referer 无关（`extra_http_headers` 也救不回来），浏览器安全策略无法绕过。
- 站点无可用 embed 播放页：`/embed/<id>.html`、`/play/<id>.html` 均 301 到详情页；
  `?embed=1` 是精简详情页（无 playInfo / video / iframe）；`?iframe=1` 只有分集按钮。
- 结论：剧集要拿直链，必须有一个 `www.novipnoad.us` / `player.novipnoad.us` 域下的
  http 父页面来嵌 `/v1/`（让 wasm 写 vkey），而宿主嗅探器只能 goto 一个 URL、
  不能注入或点击。在宿主支持「自动点击/执行 JS」之前，这类站点在网页端只能看
  单集/电影；多集剧要直链就得换源或让客户端用带嗅探 webview 的端。

## 35. MacCMS 站点：列表走 ajax/data、搜索被关、响应体被二次 JSON 编码

zzztool.com（枫叶4K影院，zp 模板）实测，可作为同类 MacCMS 站的排查模板。

### 可用入口
- 列表：`/index.php/ajax/data?mid=1&tid=<tid>&limit=30&page=<n>`
  返回 `{code,page,pagecount,total,list[]}`，字段很全（vod_id/vod_name/vod_pic/
  vod_remarks/vod_actor/vod_director/vod_blurb/type_id/type_id_1…）
  - `limit` 上限 30，传 50/100 会异常只回 10 条
  - `tid` 是父分类：1 电影 / 2 电视剧 / 3 综艺 / 4 动漫 / 5 短剧（条目里 `type_id_1` 即父类）
  - `by/order` 无效（固定按时间），`wd/key/keyword` 全部被忽略
- 详情：`/detail/<id>.html`；播放页 `/play/<id>-<sid>-<nid>.html`
- 播放页内联 `var player_aaaa={...,"url":"...","from":"...","encrypt":0}`

### 不可用入口（别浪费时间）
- `/type/*.html`、`/search/-------------.html?wd=`：返回「系统安全验证」图形验证码页；
  cookie / 手机 UA / POST / `X-Requested-With` / `?security=1` 都绕不过
- `/index.php/ajax/suggest?wd=` → `{"code":999,"msg":"联想搜索功能关闭中"}`
- `/api.php/provide/vod/` → 6 字节 `closed`
- 无 embed/自动起播页：`/embed/<id>.html`、`/play/<id>.html` 都 301 到详情页；
  `?embed=1` 是精简详情页；`?iframe=1` 只有分集按钮

### 播放线路
- `/static/js/playerconfig.js` 的 `MacPlayerConfig.player_list` 按 `from` 给 `ps/parse`：
  `ps=0` → `url` 就是直链（如 dyttm3u8 / 1080zyk / rym3u8）；
  `ps=1` → `parse` 是第三方解析页前缀（如 `https://xxx/player/?url=`），
  `url` 是 `co_xxx` / `JD-xxx` 短标识，拼 `${parse}${encodeURIComponent(url)}` 后
  用 sniffVideo 取直链（9~15s），失败回退 parse=1；`context.prefetch` 时跳过嗅探
- 解析 `player_list` 用 `indexOf(",MacPlayerConfig.")` 切片，比固定正则稳
  （后面紧跟 `downer_list` / `server_list`，拼写容易记错）

### 两个通用坑
1. **宿主会把 text/html 响应体再 JSON 编码一层**（首尾引号 + 内部 `\"` 转义）。
   `cheerio.load` 后 `h1`/`player_aaaa` 全部取不到，`$("title").text()` 会返回带
   `</title><meta ...` 的长串——看到这种「标题里混进标签」就优先怀疑这个。
   统一 `unwrapBody`：对象取 `.body/.data`，字符串首尾是 `"` 就 `JSON.parse` 一次。
2. **封面百度图床**：`https://gimg0.baidu.com/gimg/app=2001&...&src=image.zzztool.com/upload/...`
   要提取 `?src=` 还原成 `https://image.zzztool.com/...` 直链，避免 `&` 被转义裂图（同 #31）。

### 实测（v1.0.0，本地 27023）
home 2.04s/20 条；category 2.32s/30 条 pagecount=376；detail 1.11s（5 线路×30 集）；
play 直链 1.06s parse=0；解析线路 9~15s parse=0；search 首次 16s、缓存 0.13s。

### 补充：PC / 手机模板与验证码的正确处理方式
- 该站手机 UA 走 `template/z/`、PC 走 `template/zp/`，但两端搜索表单都指向
  `/search/-------------.html?wd=`；手机端并不会绕过验证码——
  用 Playwright 模拟真实 iPhone（mobile + touch + platform=iPhone）提交搜索，
  结果页仍是「系统安全验证」，手机 UA 直接 curl 也命中 6602 字节验证页。
- 用户说「手机上能搜」，通常是**之前手动输入过一次验证码，验证 cookie 还在**。
  验证流程：`POST /captcha.php?type=verify`，body `check=<code>`，
  成功返回 `{code:1}` 后 `location.reload()`。
- 因此正确做法不是换 UA，而是让脚本接受手动获取的 Cookie：
  `ZZZ_COOKIE=<浏览器里复制的整串 Cookie>`，`httpGet` 统一带上；
  search 先走站内真搜索，命中验证页/空结果时回退本地匹配。
  这样既保留了「没 cookie 也能用」的降级能力，又能在用户过验证后拿到准确结果。
- 给用户的操作路径：先用广告拦截屏蔽 `shenhe.js` 让 PC 页面能显示 →
  打开 `/type/<id>.html` 过一次验证码 → F12 复制 Cookie → 填 `ZZZ_COOKIE`。

### 补充：优先用搜索引擎蜘蛛 UA 绕过图形验证码（修正「搜索已关闭」的结论）
- 站点对普通 UA 的 `/type/*`、`/search/*` 会弹 4 位数字验证码（白底黑字、无干扰线），
  但 **Baiduspider / Googlebot / bingbot 的 UA 被直接放行**（Sogou 不在白名单）。
  实测蜘蛛 UA：`/type/2.html` → 200、46 条、可翻 313 页；
  `/search/-------------.html?wd=` → 真结果，容器 `.module-card-item`。
- 所以「搜索接口已关闭」是误判：接口没关，只是对普通 UA 加了验证码。
  **遇到 `/type`、`/search` 要验证码时，先试主流蜘蛛 UA**，
  比 OCR、暴力枚举、手工 cookie 都更稳、更快（实测搜索 <1s）。
- 验证码本身不适合脚本化：
  - `/captcha.php?type=code` 同一 PHPSESSID 连续请求也会换图，无法先识别再提交
  - 宿主 `OmniBox.request` 会把 PNG 按 UTF-8 解码（0x89 → U+FFFD），
    二进制已被破坏，脚本内拿不到原始像素，纯 JS OCR 走不通
  - 正确路径是让 `httpGet` 支持自定义 UA，搜索/分类用蜘蛛 UA，详情/播放仍用普通 UA

### 补充：CDN 带 Referer 反而 SSL 断连，播放 header 不要乱塞
- 用户「能看到封面但播不了」，但用 hls.js 在 headless Chromium 里实测是能播的
  （`readyState=4`、`videoWidth=1280/1920`、`currentTime` 正常增长），
  说明网络与流本身没问题，问题在返回给客户端的东西。
- 两处真实原因：
  1) `play()` 的 `header` 里强塞了 `Referer: <站点>`。实测 `svip.ryiplay18.com`
     带 Referer 直接 `SSL_ERROR_SYSCALL`（curl code=000），不带 Referer 返回 200；
     `vip.dytt-tvs.com` / `cdn.yzzy31-play.com` 有无 Referer 都正常。
     -> 播放 header 只保留 `User-Agent`；解析线路优先用 `sniffVideo` 返回的 header
  2) 详情页线路顺序里前两条是「蓝光2k / 至臻4k」（第三方解析 9~15s），
     用户点第一条容易以为卡死。-> 把「自营」直链线路排到前面（约 1s）
- 排查顺序（值得复用）：
  1. `curl` master m3u8 → 子 playlist，分别带/不带 Referer，看状态码与大小
  2. `curl -H 'Origin: ...'` 看 `access-control-allow-origin`（网页端 hls.js 必需）
  3. 用 hls.js + headless Chromium 真播一次，读 `readyState/videoWidth/currentTime`
  4. 最后才怀疑客户端；只看 `play()` 返回的 JSON 不足以判断「能不能播」

### 补充：m3u8 能下不代表能播——必须抽检首个分片
- 第三方解析线路会给到「m3u8 200 但分片 404」的残缺流。实例：枫叶4K影院
  《灵武大陆》至臻4k，m3u8 正常 32KB，但分片
  `https://nadou.inter.71edge.com/cdn/oss/bj/ifilm/*.png`（PNG 伪装的 TS）
  返回 `404 {"code":"NoSuchKey"}`；hls.js 表现为 `fragLoadError`、`readyState=0`。
  同线路《嫁金钗》分片 200 —— 属源站资源缺失，不是脚本或网络问题。
- 判定方法（低成本，约 1s）：
  1. 拉 m3u8，取第一条非 `#` 行
  2. 若是相对路径，用 `new URL(frag, m3u8Url)` 拼绝对（master playlist 的子列表
     也是相对路径，探子列表够判断）
  3. 发 `Range: bytes=0-1023` 请求，2xx/206 才算可用
- 失败要有兜底而不是直接报错：用详情页找**同 nid 的其它 sid**，
  逐个取 `player_aaaa`，对备选直链同样抽检，把可用线路放第一位返回，
  坏线路保留但标注「源缺失」，方便用户与排查。
- 另外：签名类 CDN（`X-Amz-Expires=300` 等）只有 5 分钟有效期，
  宿主 play 结果缓存约 10 分钟，后 5 分钟必失效——这类线路要么缩短缓存，
  要么让 playId 带时间片，否则会出现「刚点能播、过一会就不行」。

### 补充：线路名重复不是脚本 bug，但要给客户端区分开
- 《灵武大陆》详情页自己输出了 **两个同名标签「蓝光2k」**：
  `sid=8` 共 232 集（最后一集叫「正片」）、`sid=9` 共 220 集（命名 第001集…第221集）。
  脚本按 `mx-anthology-tab-label` 取线路名，客户端于是显示两条完全一样的线路。
  抽样其他剧集（73011/96244/96472/74303/74007）都是
  `蓝光2k / 至臻4k / 自营t / 自营y / 自营r` 各一条，只有集数特别多的剧会重复。
- 处理方式：透出前统计同名数量，**只对重名的线路追加集数**
  （`蓝光2k（232集）` / `蓝光2k（220集）`），唯一名保持原样。
  不要直接去重——两条线路集数不同（232 vs 220），合并会丢集。
- 顺带结论：线路名只是展示用，路由仍靠 `playId = videoId-sid-nid`，
  改名不会影响已收藏/历史里的播放。

### 补充：解析流「200 但不可播」的两种假货，以及 4K 卡在码率
枫叶4K影院《灵武大陆》两条蓝光2k 全挂，至臻4k 能播但极卡。逐层查下来：
1. **占位演示流**（最坑）：sid=9 解析出
   `https://yun.jiexicn.top/api/vod/demo/playlist.m3u8`，
   m3u8 200、分片也 200，但分片实际是 `console.lookbook.top/api/vod/demo/000.gif`
   经 `image.baidu.com/search/down?url=` 包了一层的 1px 动图。任何「只看状态码」的校验都会放行。
2. **auth_key 失效后 302 到失效页**：sid=8 解析出
   `cibn-edge-5g.1ljx.com/cloud/flv/...m3u8?auth_key=<epoch>.<rand>`，
   过期后 302 → `aibox.eu.org/110`（402 DEPLOYMENT_DISABLED）。
   带 Referer / 换 UA 都一样，**不是防盗链，是签名过期**。
3. **master playlist 陷阱**：站点直链多是 `index.m3u8`（97B、只有一行 variant），
   直接取「首个非 # 行」当分片去探，探到的还是 m3u8（必然 200）。
   必须先看 `#EXT-X-STREAM-INF`，跟到 `3000k/hls/mixed.m3u8` 这类真正分片列表。
4. **卡 = 码率，不是脚本**：至臻4k（tjtele PERSONCLOUD）单片 7.6~8MB / 5s ≈ 12Mbps，
   分片实测 8MB 下载 0.42s（18MB/s），链路本身很快；`x-obs-traffic-limit=409600`
   并不能删（签名覆盖 query，删掉直接 403）。带宽一般或 HLS 缓冲小的端就卡。

### 可复用规则
- 校验流是否可播：跟到真分片 → 分片数 ≥3 → 抽检首个分片要 2xx/206；
  **异常一律判不可用**（不要“失败放行”，那是把死链交给用户的元凶）。
- 维护占位黑名单：`/api/vod/demo/`、`image.baidu.com`、`console.lookbook.top`、
  `picsum.photos`；分片是 `.gif` 基本可以直接判假。
- 换线要并发：详情页拿 sid→线路名，同 nid 并发取多条 `player_aaaa` + 并发抽检，
  首次 ~7s；再把「失效 sid」和「好用 sid 顺序」各缓存 10~15 分钟，
  后续同剧换集从 19.5s 降到 7~9s。
- 高码率线路（分片 ≥2.5MB）自动附带同集 1080p 直链，让用户一键换线；
  主线路仍按用户点的来，别擅自降清晰度。
