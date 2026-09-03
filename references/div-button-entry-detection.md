---
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: 'ac77346c-1194-49d3-9ee1-0ae6803a0de3'
  PropagateID: 'ac77346c-1194-49d3-9ee1-0ae6803a0de3'
  ReservedCode1: '22e555d9-5db7-43df-8492-4d219c975154'
  ReservedCode2: '22e555d9-5db7-43df-8492-4d219c975154'
---

# div 模拟按钮与入口检测方案

## 背景

电信 H5 页面（wapcq.189.cn）均为 Vue 单页应用，按钮和入口用 `<div>` + `@click` 模拟，DOM 上无 `onclick` 属性、无 `role` 属性，传统 CSS 选择器（`button`、`nav a`、`.menu a`）完全匹配不到。

## 按钮检测（_check_button，规则2）

### 候选识别（evaluate JS）

三类候选，按优先级：

1. **标准按钮**：`button[type=submit]`、`button:not([type])`、`button[type=button]`、`input[type=submit]`、`input[type=button]` — 检查 `disabled` 状态
2. **div 模拟按钮**（evaluate 收集）：
   - **显式可点击**：`[onclick]`、`[role="button"]`、`[role="link"]`
   - **样式像按钮**：背景色非透明 + borderRadius > 2 + 尺寸 40~600×24~100
   - **icon-card 入口**：div 含 `img` 或 `.icon` 子元素 + `p/span` 短文字（≤15字），尺寸 40~800×30~500
3. **去重**：按 (text, round(cx/5), round(cy/5)) 去重，最多验证 3 个

### 点击验证流程

```
for each candidate:
  1. 确保页面稳定（wait_for_load_state + URL 偏离则 goto 回原始页面）
  2. 记录点击前状态：url_before, body_len_before
  3. 用 index 重新收集候选 → 取坐标 → scrollIntoView（window.scrollTo）
  4. page.mouse.click(x, y)
  5. 等 1500ms（等导航/响应完成）
  6. 判定响应：
     - URL 变化 → 有响应
     - evaluate 异常（context destroyed）→ 有响应（页面正在导航）
     - 弹窗/loading 出现 → 有响应
     - body.innerText.length 变化 > 30 → 有响应
     - 新增可见 input/overlay/mask → 有响应
     - 以上都不满足 → "点击无响应"（失效）
```

### 关键坑

- **不用 marker 属性定位**：Vue 重渲染会清除 `data-tib-marker`，改用 index（重新收集候选列表按 index 取坐标，querySelectorAll 顺序稳定）
- **clicked 标志**：点击后的 evaluate 异常 = 响应成功，点击前的异常 = 定位失败
- **视口外按钮**：故障快修页 icon-card 在 y=3646（视口 1080），必须先 `scrollIntoView` 再点击，否则 mouse.click 落空

## 入口检测（_check_entry，规则3）

### 候选识别（evaluate JS）

六类入口：

1. **标准 a 标签**：`nav a`、`.nav a`、`.menu a`、`header a`、`.banner a`、`.tab a`、`.footer a`、`.bottom-nav a`、`[data-entry]`
2. **导航栏/Tab/菜单**：class/id 匹配 `nav|tab|menu|bottom|entry|switch|item|bar|footer|category`
3. **Banner**：尺寸 200~1200×60~400 + 含 img/背景图，排除 span/p/h1~h6 纯文字标题
4. **浮窗**：position=fixed/absolute + 尺寸 40~600×≤300
5. **icon-card**：div 含 img/.icon + p/span 短文字，尺寸 40~800×30~500
6. **onclick/role**：带 onclick 或 role=button/link 的元素

### 点击验证（500ms 判定）

与按钮检测类似，但响应判定窗口为 500ms（用户需求），额外增加：

- **class 状态变化检测**：Tab 切换不改变内容长度，但 active class 会变（如 `am-tabs-default-bar-tab` → `am-tabs-default-bar-tab-active`）。点击前记录候选元素 className，点击后对比，变化则视为有响应。

### 关键坑

- **检查项间页面恢复**：按钮检测点击跳转后，入口检测会在错误页面执行。修复：`inspect` 循环中每项检查前检测 `page.url != page_url`，偏离则 `page.goto(page_url)` 导航回去。
- **Banner 误报**：纯文字标题 span（如"相关服务"）因尺寸和含图片被误判为 Banner。修复：排除 span/p/h1~h6 标签。
- **icon-card 尺寸限制**：故障快修页 icon-card 为 449×481，初始 height 上限 300 太小，需放宽到 500。

## Playwright Python API 注意

- `page.evaluate(expression, arg)` 只接受**一个** arg 参数，传多个值需打包成 dict：`page.evaluate("(pt) => {...}", {"cx": x, "cy": y})`
- `page.evaluate` 中 JS 抛异常会向上传播，需 try/except 捕获
- `page.mouse.click(x, y)` 的坐标是视口坐标，元素在视口外时点击落空，必须先滚动

## 验证页面

| 页面 | URL | button_status | entry_response |
|------|-----|:-:|:-:|
| 密码重置页 | wapcq.189.cn/page/ecsc_servicepassword/#/ | 1 | -1（无入口） |
| 故障快修页 | wapcq.189.cn/page/fast-fault-repair-ts/#/index | 1 | 1 |
| 宽带页 | wapcq.189.cn/page/ecsc-broadband/ | -1（无按钮） | -1（无入口） |
| 百度首页 | www.baidu.com/ | 1 | 1 |