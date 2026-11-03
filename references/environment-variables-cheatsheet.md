# OmniBox 环境变量速查表（带注释版）

这是一份面向**实际开发**的速查表，基于 `/root/.openclaw/workspace/OmniBox-Spider` 仓库扫描结果整理。

目标：
- 写新脚本时快速决定该用哪个环境变量名
- 统一命名习惯
- 尽量复用已有变量，避免配置碎片化

---

# 一、使用原则

## 1. 优先复用已有变量名
如果你要实现的功能和现有变量语义一致，就直接复用，不要重新起名。

例如：
- 站点 API 地址 → `SITE_API`
- 弹幕接口地址 → `DANMU_API`
- 网盘优先级顺序 → `DRIVE_ORDER`
- PanCheck 配置 → `PANCHECK_*`

## 2. 新增变量时的命名规则
如果现有变量实在不合适，再新增，规则是：
- 全大写
- 下划线分隔
- 名字直接表达用途
- 优先沿用仓库既有风格（如 `XXX_HOST`、`PANCHECK_*`、`EMBY_*`）

## 3. 交付脚本时必须说明
以后写 OmniBox 脚本时，凡是用到环境变量，都应该在说明里写：
- 变量名
- 作用
- 是否必填
- 示例值（如有必要）

---

# 二、最常用变量（优先级最高）

## `SITE_API`
### 作用
普通采集站 / API 站的基础接口地址。

### 适用场景
- MacCMS API
- JSON 采集接口
- 普通资源站 API
- 影视采集模板类脚本

### 何时优先复用
只要你的脚本核心是“请求某个统一 API 地址”，就优先用它。

### 示例
```js
const SITE_API = process.env.SITE_API || "";
```
```py
SITE_API = os.environ.get("SITE_API", "")
```

### 备注
这是最应该优先复用的通用变量之一。

---

## `DANMU_API`
### 作用
弹幕接口地址。

### 适用场景
- 通过外部弹幕服务匹配番剧/影视弹幕
- `play()` 中返回 `danmaku`

### 何时优先复用
只要你的脚本要对接独立弹幕服务，就优先用 `DANMU_API`。

### 备注
不要为同类能力重新造名字，比如 `DM_API`、`DANMAKU_URL` 这类都不如直接复用统一名。

---

# 三、网盘 / 多线路 / 聚合相关

## `DRIVE_ORDER`
### 作用
控制网盘线路优先级顺序。

### 适用场景
- 多网盘结果排序
- detail 里多线路聚合
- 盘搜 / 聚盘类脚本

### 何时优先复用
当你要决定“夸克优先还是百度优先”“哪个线路先展示”时，优先用它。

### 示例用途
- 控制 `vod_play_sources` 的线路顺序
- 在搜索结果中按盘源优先级排序

---

## `DRIVE_TYPE_CONFIG`
### 作用
定义网盘类型配置。

### 适用场景
- 同一脚本支持多个网盘平台
- 需要按类型分流处理时

### 何时优先复用
当你需要配置“支持哪些盘、怎么识别这些盘、每类盘怎么处理”时，优先复用。

---

## `SOURCE_NAMES_CONFIG`
### 作用
定义来源名称映射或显示名配置。

### 适用场景
- 统一源名称展示
- 多站点 / 多盘源的别名整理

### 何时优先复用
当你需要“把内部源名映射成用户能看懂的名字”时优先用它。

---

## `MAX_PAN_VALID_ROUTES`
### 作用
限制网盘有效线路数量。

### 适用场景
- 搜索结果过多
- detail 线路太多需要裁剪

### 何时优先复用
当你需要控制“最多保留几条有效网盘线路”时优先用它。

---

# 四、PanCheck / 盘搜体系

这组变量建议作为一个家族整体复用。

## `PANCHECK_API`
### 作用
PanCheck 服务地址。

### 场景
- 批量校验网盘链接是否失效

---

## `PANCHECK_ENABLED`
### 作用
是否启用 PanCheck。

### 场景
- 允许脚本开关式启用链接校验

### 推荐值
- `1`：启用
- `0`：关闭

---

## `PANCHECK_PLATFORMS`
### 作用
指定参与校验的平台。

### 场景
- 只校验某几类网盘，比如夸克、百度、迅雷

### 示例值
```text
quark,baidu,xunlei
```

---

## `PANSOU_API`
### 作用
盘搜 API 地址。

### 场景
- 调用盘搜服务获取资源结果

---

## `PANSOU_CHANNELS`
### 作用
盘搜渠道配置。

### 场景
- 控制搜索来源渠道

---

## `PANSOU_CLOUD_TYPES`
### 作用
盘搜云盘类型配置。

### 场景
- 控制只搜哪些网盘类型

---

## `PANSOU_FILTER`
### 作用
盘搜过滤条件。

### 场景
- 屏蔽无关资源
- 过滤广告资源、低质量资源等

---

## `PANSOU_PLUGINS`
### 作用
盘搜插件配置。

### 场景
- 启用或控制某些盘搜扩展逻辑

---

## 这组变量什么时候必须优先复用？
当你写的是：
- 盘搜脚本
- 聚盘搜索
- 网盘资源搜索聚合
- PanCheck 校验逻辑

就**不要重新发明一套命名**，直接优先复用 `PANCHECK_*` + `PANSOU_*`。

---

# 五、Emby 体系变量

这一组专门用于 Emby 脚本，应该整体复用。

## `EMBY_HOST`
### 作用
Emby 服务器地址。

## `EMBY_TOKEN`
### 作用
Emby 访问令牌。

## `EMBY_USER_ID`
### 作用
Emby 用户 ID。

## `EMBY_DEVICE_ID`
### 作用
设备 ID，用于模拟客户端请求。

## `EMBY_CLIENT_VERSION`
### 作用
客户端版本号。

## `EMBY_PAGE_SIZE`
### 作用
分页大小配置。

### 什么时候优先复用
只要是 Emby 相关脚本，统一优先用 `EMBY_*`，不要写出另一套 `MEDIA_HOST`、`SERVER_TOKEN` 之类的平行命名。

---

# 六、小雅 / AList / TVBox 体系

## `ALIST_TVBOX_TOKEN`
### 作用
AList / TVBox 场景下的访问令牌。

## `XIAOYA_BASE_URL`
### 作用
小雅服务基础地址。

## `XIAOYA_CLASS_JSON`
### 作用
小雅分类配置 JSON。

## `XIAOYA_ENABLE_PROXY`
### 作用
是否启用代理。

## `XIAOYA_PROXY_URL`
### 作用
代理地址。

## `XIAOYA_TOKEN`
### 作用
小雅访问令牌。

### 什么时候优先复用
只要脚本涉及：
- 小雅
- AList
- TVBox 联动

都优先沿用这组变量。

---

# 七、站点地址类变量（Host 风格）

这组变量说明：仓库里已经存在“按站点拆分 host 变量”的风格。

## 已有示例
- `KANQIU_HOST`
- `LETU_HOST`
- `REBANG_HOST`
- `TVBYB_HOST`
- `SJ_MUSIC_HOST`

以及一组：
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

### 适用场景
- 脚本绑定某个具体站点
- 希望站点地址可配置
- 站点经常换域名

### 命名建议
如果你要新增站点地址变量：
- 单站点 → 优先 `XXX_HOST`
- 同一家族站群 → 优先延续 `WEB_SITE_XXX`

### 示例
新写“某某影院”脚本时，优先考虑：
- `MOUMOU_HOST`
而不是：
- `BASE_URL`
- `SITE_URL`
- `HOST_URL`
这种过于泛化、难复用的名字

---

# 八、专项变量

## `BILI_COOKIE`
### 作用
B站相关 Cookie。

### 场景
- 需要登录态
- 需要访问带权限或频控内容

---

## `CATEGORY_BLOCKLIST`
### 作用
分类黑名单。

### 场景
- 屏蔽不想展示的分类

---

## `EXCLUDE_CLASS_NAMES`
### 作用
排除某些分类名。

### 场景
- 清洗站点原始分类
- 屏蔽广告类 / 无意义分类

---

## `IQIYIZY_BLOCKED_MAIN`
### 作用
爱奇艺资源站的主站过滤/屏蔽配置。

### 场景
- 对某个专项脚本做特定策略控制

---

# 九、开发时怎么选变量名

## 场景 1：我在写普通采集站
优先看：
- `SITE_API`
- `DANMU_API`
- `CATEGORY_BLOCKLIST`
- `EXCLUDE_CLASS_NAMES`

## 场景 2：我在写网盘聚合 / 盘搜
优先看：
- `DRIVE_ORDER`
- `DRIVE_TYPE_CONFIG`
- `SOURCE_NAMES_CONFIG`
- `PANCHECK_*`
- `PANSOU_*`
- `MAX_PAN_VALID_ROUTES`

## 场景 3：我在写 Emby
直接优先：
- `EMBY_*`

## 场景 4：我在写小雅 / AList / TVBox
直接优先：
- `XIAOYA_*`
- `ALIST_TVBOX_TOKEN`

## 场景 5：我在写单站点脚本，站点地址经常变
优先：
- `XXX_HOST`
- 或延续现有家族前缀风格

---

# 十、以后写新脚本时的强制规则

1. **先查这份速查表**
2. 有现成变量名就直接复用
3. 没有时再新增
4. 新增变量必须：
   - 名称清晰
   - 与现有命名体系兼容
   - 在交付说明中写清原因

---

# 十一、推荐交付说明模板

以后交付脚本时，建议附上：

```md
## 环境变量
- SITE_API：站点 API 地址，必填
- DANMU_API：弹幕服务地址，可选
- DRIVE_ORDER：网盘线路优先级，可选
```

如果有新增变量：

```md
- NEW_VAR：作用说明（新增原因：现有变量无法准确表达此配置）
```

---

# 十二、结论

最值得记住的几个高优先级变量：
- `SITE_API`
- `DANMU_API`
- `DRIVE_ORDER`
- `DRIVE_TYPE_CONFIG`
- `SOURCE_NAMES_CONFIG`
- `PANCHECK_*`
- `PANSOU_*`
- `EMBY_*`
- `XIAOYA_*`

开发时优先复用这些，后续脚本会更统一，也更方便维护。