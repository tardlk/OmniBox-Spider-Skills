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
