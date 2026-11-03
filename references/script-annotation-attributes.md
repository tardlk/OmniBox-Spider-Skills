# 脚本注释属性 | OmniBox

> 基于最新官方文档同步：
> https://omnibox-doc.pages.dev/spider-development/script-annotation-attributes

## 重要前提

脚本注释属性必须写在脚本**前 50 行**内。

- JavaScript：行首 `//`
- Python：行首 `#`

官方当前明确说明：**只有下列属性会被后端解析并实际使用**。

## 1. `@version`

作用：
- 版本号
- 用于后台「检查更新」与「从远端更新」时的版本比较

示例：

### JavaScript
```js
// @version: 1.0.0
// @version 1.0.0
```

### Python
```python
# @version: 1.0.0
# @version 1.0.0
```

---

## 2. `@downloadURL`

作用：
- 远程脚本地址
- 用于后台从远端拉取脚本并比较 `@version`

未配置时：
- 无法使用检查更新 / 从远端更新

示例：

### JavaScript
```js
// @downloadURL: https://example.com/script.js
// @downloadURL https://example.com/script.js
```

### Python
```python
# @downloadURL: https://example.com/script.py
# @downloadURL https://example.com/script.py
```

---

## 3. `@indexs`

作用：
- 是否参与聚合搜索

取值：
- `1`：参与
- `0`：不参与

说明：
- 属性名与数字之间要有空格
- 未声明时视为不参与

示例：
```js
// @indexs 1
// @indexs 0
```

---

## 4. `@push`

作用：
- 是否为推送型爬虫

取值：
- `1`：推送型
- `0`：常规爬虫

说明：
- 推送型爬虫通常只需 `detail` 与 `play`
- 未声明时视为常规爬虫

示例：
```js
// @push 1
// @push 0
```

---

## 5. `@dependencies`

作用：
- 声明外部依赖（npm / pip）
- 保存脚本时，系统会尝试自动安装缺失依赖

格式：
- 单行
- 逗号分隔
- 支持冒号或空格两种写法

示例：

### JavaScript
```js
// @dependencies: axios,cheerio
// @dependencies axios,cheerio
```

### Python
```python
# @dependencies: requests,beautifulsoup4
# @dependencies requests,beautifulsoup4
```

---

## 实战建议

- 不要凭经验假设还有别的注释属性会被后端识别；同步官方文档时，以这 5 个为准。
- 远程分发脚本时，通常至少配：
  - `@version`
  - `@downloadURL`
- 参与聚合搜索的源：
  - `@indexs 1`
- 推送源：
  - `@push 1`
