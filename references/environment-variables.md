# OmniBox 环境变量复用指南

本文档来自对 `/root/.openclaw/workspace/OmniBox-Spider` 仓库的实际扫描结果。

目标：
- 新写脚本时优先复用已有环境变量命名
- 降低配置碎片化
- 保持不同脚本之间的配置习惯一致

---

## 总规则

### 1. 优先复用，不要随意新造变量名
当已有语义相同或接近的环境变量时，优先沿用现有命名。

例如：
- 采集站 API 基础地址 → 优先 `SITE_API`
- 弹幕接口地址 → 优先 `DANMU_API`
- 网盘优先级顺序 → 优先 `DRIVE_ORDER`
- PanCheck 相关配置 → 优先 `PANCHECK_*`

### 2. 新变量命名风格
若确实需要新增，遵循：
- 全大写
- 下划线分隔
- 尽量语义明确
- 优先“能力/域名 + 功能”风格

例如：
- `KANQIU_HOST`
- `TVBYB_HOST`
- `REBANG_HOST`

### 3. 读取方式
- JavaScript：优先 `process.env.KEY`
- Python：优先 `os.environ.get("KEY")`

### 4. 文档要求
当脚本使用环境变量时，必须在交付说明里写清楚：
- 变量名
- 作用
- 是否必填
- 示例值（如适合）

### 5. 先确认是否真的需要新环境变量
遇到“播放历史 / 收藏 / 追剧 / 收藏标签页”这类需求时，先确认是否可直接使用新版 SDK：
- `getSourceFavoriteTags()`
- `getSourceCategoryData("history" | "favorite" | "follow" | 标签名, page, pageSize)`

这类能力优先走 SDK，不要下意识新造本地存储环境变量或额外接口配置。

---

## 已扫描出的环境变量索引

### 通用采集/API 类

#### `SITE_API`
**作用**：普通采集站 / API 站的基础地址。
**已出现位置**：
- `影视/采集/电影天堂.js`
- `模板/JavaScript/采集站模板.js`

**建议**：
- 只要是“对接一个普通 JSON API / MacCMS API / 资源站 API”，优先使用 `SITE_API`

---

### 弹幕类

#### `DANMU_API`
**作用**：弹幕接口地址。
**已出现位置**：多个动漫与影视采集脚本。

**建议**：
- 新脚本只要需要独立弹幕服务接口，优先复用 `DANMU_API`

---

### 网盘优先级 / 网盘站配置

#### `DRIVE_ORDER`
**作用**：控制网盘线路优先级顺序。
**已出现位置**：
- `影视/网盘/聚盘搜索.js`

**建议**：
- 需要多网盘线路排序时优先用它

#### `DRIVE_TYPE_CONFIG`
**作用**：控制网盘类型配置。
**已出现位置**：多个网盘脚本（木偶、盘搜、至臻、闪电等）

**建议**：
- 多网盘站点或统一网盘类型适配时优先复用

#### `SOURCE_NAMES_CONFIG`
**作用**：控制源名称映射/配置。
**已出现位置**：多个网盘脚本与部分采集脚本

**建议**：
- 需要做来源名称配置化时优先复用

---

### PanCheck / 盘搜体系

#### `PANCHECK_API`
**作用**：PanCheck 服务地址。

#### `PANCHECK_ENABLED`
**作用**：是否启用 PanCheck。

#### `PANCHECK_PLATFORMS`
**作用**：指定参与校验的平台。

#### `PANSOU_API`
**作用**：盘搜 API 地址。

#### `PANSOU_CHANNELS`
**作用**：盘搜渠道配置。

#### `PANSOU_CLOUD_TYPES`
**作用**：盘搜云盘类型配置。

#### `PANSOU_FILTER`
**作用**：盘搜过滤条件。

#### `PANSOU_PLUGINS`
**作用**：盘搜插件配置。

**建议**：
- 只要你写的是盘搜/聚盘/网盘聚合类脚本，优先沿用 `PANCHECK_*` 与 `PANSOU_*` 体系，不要重新发明变量名

---

### 站点 Host 类变量

这些变量一般用于“站点地址可配置”：

- `KANQIU_HOST`
- `LETU_HOST`
- `REBANG_HOST`
- `TVBYB_HOST`
- `SJ_MUSIC_HOST`

以及一组网盘站点地址变量：
- `WEB_SITE_DUODUO`
- `WEB_SITE_ERXIAO`
- `WEB_SITE_HUBAN`
- `WEB_SITE_LABI`
- `WEB_SITE_MUOU`
- `WEB_SITE_OUGE`
- `WEB_SITE_SHANDIAN`
- `WEB_SITE_WOGG`
- `WEB_SITE_XIAOBAN`
- `WEB_SITE_ZHIZHEN`

**建议**：
- 如果某个脚本明确绑定单站点，可用 `XXX_HOST`
- 如果是同类脚本家族（特别是网盘站群），尽量延续已有前缀体系

---

### 小雅 / AList / TVBox 类

#### `ALIST_TVBOX_TOKEN`
**作用**：AList TVBox 访问令牌

#### `XIAOYA_BASE_URL`
**作用**：小雅基础地址

#### `XIAOYA_CLASS_JSON`
**作用**：小雅分类配置 JSON

#### `XIAOYA_ENABLE_PROXY`
**作用**：是否启用代理

#### `XIAOYA_PROXY_URL`
**作用**：代理地址

#### `XIAOYA_TOKEN`
**作用**：小雅访问令牌

**建议**：
- 写小雅 / AList / TVBox 相关脚本时优先沿用这套命名

---

### Emby 类

#### `EMBY_HOST`
#### `EMBY_TOKEN`
#### `EMBY_USER_ID`
#### `EMBY_DEVICE_ID`
#### `EMBY_CLIENT_VERSION`
#### `EMBY_PAGE_SIZE`

**作用**：Emby 服务连接与客户端参数配置。

**建议**：
- 任何 Emby 相关脚本都应优先复用这套变量命名

---

### Bili / 站点专项变量

#### `BILI_COOKIE`
**作用**：B站相关 Cookie

#### `IQIYIZY_BLOCKED_MAIN`
**作用**：爱奇艺资源站主站过滤/屏蔽配置

#### `CATEGORY_BLOCKLIST`
**作用**：分类黑名单

#### `EXCLUDE_CLASS_NAMES`
**作用**：分类名排除列表

#### `MAX_PAN_VALID_ROUTES`
**作用**：限制网盘有效线路数

**建议**：
- 若功能语义一致，继续沿用这些现有变量名

---

## 开发时的优先复用规则（强制）

写新 OmniBox 脚本时：

1. 如果是普通采集站 API：
   - 优先 `SITE_API`

2. 如果要接弹幕：
   - 优先 `DANMU_API`

3. 如果是网盘聚合 / 多线路排序：
   - 优先 `DRIVE_ORDER`
   - 优先 `DRIVE_TYPE_CONFIG`
   - 优先 `SOURCE_NAMES_CONFIG`

4. 如果要做 PanCheck / 盘搜：
   - 优先 `PANCHECK_*`
   - 优先 `PANSOU_*`

5. 如果是 Emby：
   - 优先 `EMBY_*`

6. 如果是小雅 / AList / TVBox：
   - 优先 `XIAOYA_*`
   - 优先 `ALIST_TVBOX_TOKEN`

7. 如果只是站点基础地址可配置：
   - 优先用已有 `XXX_HOST` 风格

---

## 交付要求

以后交付 OmniBox 脚本时，默认在代码外补一段：

- 使用到的环境变量
- 每个变量的作用
- 是否可选
- 默认值 / 示例值

如果新增变量，还要说明为什么不能复用现有变量名。
