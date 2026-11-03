# DS Template Notes

## 何时使用本参考

当用户明确要的是：
- ds源
- drpy 源
- drpy-node 规则
- `var rule = {}`

而不是 OmniBox 五方法 spider 时，优先读这份参考。

## 目标区分

### OmniBox spider
输出形态：
```js
module.exports = { home, category, detail, search, play };
runner.run(module.exports);
```

### ds / drpy-node 规则
输出形态：
```js
var rule = { ... };
```

## ds 生成要点

### 必填核心字段
至少保证：
- `类型`
- `title`
- `host`
- `url`
- `searchUrl`（如果支持搜索）

常见附加字段：
- `searchable`
- `quickSearch`
- `filterable`
- `headers`
- `timeout`
- `class_name`
- `class_url`
- `play_parse`
- `lazy`
- `double`
- `limit`
- `推荐`
- `一级`
- `二级`
- `搜索`

### 规则字符串语法
传统服务端渲染站优先用字符串规则：
```js
"列表选择器;标题;图片;描述;链接"
```

常见语法：
- `&&`：嵌套
- `||`：备用
- `:eq(n)`：取索引
- `Text` / `Html` / `href` / `src` / `data-*`

### 复杂站点
如果站点是：
- 前后端分离
- 返回 JSON
- 有加密接口
- 需要自定义发包

则可以在 `推荐 / 一级 / 二级 / 搜索 / lazy` 中使用：
```js
async function () { ... }
```

### drpy-node 常用全局函数
- `request(url, options)`
- `post(url, options)`
- `pdfa(html, rule)`
- `pdfh(html, rule)`
- `pd(html, rule)`
- `setItem / getItem / clearItem`
- `urljoin`
- `md5`
- `MOBILE_UA / UC_UA / PC_UA`

## 实战建议

1. 先判断站点更适合 DOM 规则还是 API 规则。
2. 能返回 JSON 时，优先 API；结构更稳定。
3. 站点简单时优先字符串规则，代码更短更稳。
4. 站点复杂时才用 `async function`，不要一上来就写大函数。
5. 最终输出必须是 JS 代码，不是 JSON 说明文。

## 外部参考
- `/root/.openclaw/workspace/_tmp_claw_ds/template/ds_template.js`
- `/root/.openclaw/workspace/_tmp_claw_ds/references/drpy_api_reference.md`
