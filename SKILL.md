---
name: tele-inspect-ops
name_cn: tele-inspect巡检系统运维
description: 线上AI智能巡检系统（tele-inspect）的开发、部署与运维技能。覆盖项目架构（Next.js前端5000+FastAPI后端8000+PostgreSQL）、Windows Server部署脚本（win:*）、巡检执行逻辑与已知坑（load_time统计、Vue div按钮、异常定位记录、iframe内弹窗检测）、定时任务并发调度（并发2/超时180s/重试1次）、IM群机器人webhook告警。触发词：启动项目、巡检系统、部署到Windows、巡检超时、定时任务、巡检按钮、死链接、tele-inspect。
description_cn: 线上AI智能巡检系统（tele-inspect）开发部署与运维技能：架构、Windows启动、巡检逻辑与已知坑（load_time统计、Vue div按钮、异常定位、iframe弹窗检测）、定时并发调度、IM群机器人webhook告警。
AIGC:
  ContentProducer: '001191110102MAD55U9H0F10002'
  ContentPropagator: '001191110102MAD55U9H0F10002'
  Label: '1'
  ProduceID: '5bdb812f-fc14-4b3e-9f24-ec2a208a4c8e'
  PropagateID: '5bdb812f-fc14-4b3e-9f24-ec2a208a4c8e'
  ReservedCode1: '636dab4f-5e35-462a-b9a3-063fe1cfacb8'
  ReservedCode2: '636dab4f-5e35-462a-b9a3-063fe1cfacb8'
---

# tele-inspect 巡检系统运维

## 项目架构

| 组件 | 技术 | 端口 | 说明 |
|------|------|------|------|
| 前端 | Next.js（src/server.ts 自定义服务器） | 5000 | 页面展示、任务管理 |
| 后端 | Python FastAPI（backend/main.py） | 8000 | 巡检执行、APScheduler 定时调度 |
| 数据库 | PostgreSQL（inspection_db） | 5432 | init.sql 建表：inspection_tasks / inspection_results / inspection_reports |

告警通知：backend/notifier.py 通过 IM 群机器人 webhook（imtwo.zdxlz.com）推送巡检异常消息。

## 启动与部署

### 本地（macOS/Linux）
```bash
# 前端（开发模式）
bash ./scripts/dev.sh        # 或 npm run dev
# 后端
cd backend && python3 main.py
```

### Windows Server
PowerShell 脚本与 bash 脚本完全独立，通过 npm 的 `win:` 前缀调用：
```powershell
npm run win:build     # 安装依赖 + next build + tsup 打包
npm run win:start     # 生产模式启动前端（需先 build）
npm run win:backend   # 启动后端（自动检测虚拟环境与 Playwright）
npm run win:dev       # 开发模式（调试用）
npm run win:stop      # 停止前端(5000)+后端(8000)，可 -FrontendPort/-BackendPort 指定端口
```
所有脚本自动检测并释放占用端口。部署环境要求：Node 20+、Python 3.10+、PostgreSQL 15+、`playwright install chromium`。

### 资源评估（4核8G 云电脑）
- 运行时：Windows Server 桌面版占 ~2G + 前后端常驻 <1G + Playwright 巡检时临时 0.5G，8G 够用
- 构建阶段峰值 2-3G：建议在构建机生成 `.next/` + `dist/` 后连同代码拷到服务器，跳过最吃资源的构建步骤

### PostgreSQL 安装（Windows Server 从零部署）

服务器无 PostgreSQL、无 winget/choco/scoop 时，用 EDB 官方 zip 绿色版安装：

```powershell
# 1. 下载（约 300MB，curl 断点续传比 Invoke-WebRequest 更可靠）
$dir = ".temp\pg-setup"
curl.exe -L -C - --retry 3 --retry-delay 5 -o "$dir\pg.zip" `
  "https://get.enterprisedb.com/postgresql/postgresql-16.9-1-windows-x64-binaries.zip"

# 2. 解压——必须用 tar，不能用 Expand-Archive（后者对大 zip 易超时且可能不完整）
tar -xf "$dir\pg.zip" -C $dir
# 验证完整性：share\postgres.bki 必须存在，否则 initdb 会报错
Test-Path "$dir\pgsql\share\postgres.bki"

# 3. 安装到正式目录（⚠️ 不能用 C:\Program Files\，权限不足 initdb 会失败）
Copy-Item -Path "$dir\pgsql" -Destination "C:\PostgreSQL\16" -Recurse -Force
# ⚠️ Copy-Item 可能不完整，务必验证 share\postgres.bki 存在

# 4. 初始化数据目录
& "C:\PostgreSQL\16\bin\initdb.exe" -D "C:\PostgreSQL\16\data" -U postgres -E UTF8 --locale=C -A trust

# 5. 注册 Windows 服务（⚠️ 不能带 -U postgres，那是 Windows 账户名不是 PG 用户名，会报错 1057）
& "C:\PostgreSQL\16\bin\pg_ctl.exe" register -N "PostgreSQL16" -D "C:\PostgreSQL\16\data"
Set-Service -Name "PostgreSQL16" -StartupType Automatic
Start-Service -Name "PostgreSQL16"

# 6. 建库 + 导入建表脚本
& "C:\PostgreSQL\16\bin\psql.exe" -U postgres -h 127.0.0.1 -p 5432 -c "CREATE DATABASE inspection_db;"
& "C:\PostgreSQL\16\bin\psql.exe" -U postgres -h 127.0.0.1 -p 5432 -d inspection_db -f init.sql
```

**已知坑：**
- `C:\Program Files\` 下 initdb 报 `Permission denied`，必须用无空格路径如 `C:\PostgreSQL\16`
- `pg_ctl register -U postgres` 的 `-U` 指的是 Windows 服务运行账户，不是 PostgreSQL 超级用户；系统无 `postgres` Windows 账户时报错 1057，去掉 `-U` 默认用 LocalSystem 即可
- `Expand-Archive` 解压 300MB zip 易超时且可能不完整（缺 share 目录），改用 `tar -xf` 更快更可靠
- `Invoke-WebRequest` 下载大文件易超时，改用 `curl.exe -C -` 断点续传
- trust 认证（`-A trust`）无需密码，后端 `POSTGRES_PASSWORD` 留空即可

### 代码更新与重建（git pull 后）

项目代码可能以非 git 方式拷贝到服务器，需先关联远程再拉取：

```powershell
# 关联远程仓库（本地非 git 仓库时）
git init
git remote add origin git@github.com:Flokken/tele-inspect.git
git fetch origin
git checkout -b main --track origin/main -f
# -f 强制覆盖工作区文件，.gitignore 已排除 node_modules/.next/dist/venv/.temp 等运行文件

# 技能仓库同理
cd C:\Users\Administrator\.config\TeleAgent\skills\tele-inspect-ops
git init && git remote add origin git@github.com:Flokken/tele-inspect-ops-skill.git
git fetch origin && git checkout -b main --track origin/main -f
```

拉取后重建前端（⚠️ git pull 会覆盖 package.json，之前手动装的依赖会丢失）：
```powershell
pnpm install                    # 重新安装依赖
npx next build                  # 构建
npx tsup src/server.ts --format cjs --platform node --target node20 --outDir dist --no-splitting --no-minify
```

**⚠️ 代码更新后必须检查数据库表结构同步**（详见已知坑第 10 条）。

### Windows Server 进程管理

停止前后端服务时，**不要用 `taskkill /F /IM node.exe`** 杀所有 node 进程——TeleAgent 自身也运行 node 进程（Session 0 = Services），全杀会导致系统卡顿、PowerShell 命令超时数分钟。

正确做法：只杀 Console 会话（Session 1）中占用 5000/8000 端口的进程：
```powershell
# 用 tasklist 查 PID 和 Session
tasklist /FI "IMAGENAME eq node.exe" /FO CSV /NH
# 只杀 Console session 的进程，保留 Services session 的 TeleAgent 进程
$consoleNodes = @(7928, 1640, 160)  # 替换为实际 Console session PID
foreach ($procId in $consoleNodes) { Stop-Process -Id $procId -Force -ErrorAction SilentlyContinue }
```

如果 PowerShell 已经卡住（命令持续超时），等待 30-60 秒后通常会恢复，用 `echo "test"` 测试响应。

## 巡检核心逻辑（backend/inspector.py）

- 8 项规则定义在 backend/config.py 的 CHECK_ITEMS，阈值在 INSPECTION_CONFIG
- 关键阈值：页面加载超时 5000ms、接口超时 3000ms、严重超时 5000ms、空白阈值 0.1
- 巡检结果写入 inspection_results，异常时 send_alert 推送告警

| 规则ID | 名称 | 级别 | 说明 |
|--------|------|------|------|
| 1 | 页面空白 | 严重 | 检测页面是否为空白或加载失败 |
| 2 | 按钮失效 | 严重 | 检测核心按钮点击是否有响应 |
| 3 | 入口点击无响应 | 严重 | 检测入口点击是否触发跳转或交互 |
| 4 | 跳转错误 | 严重 | 检测跳转目标是否与预期一致 |
| 5 | 死链接 | 严重 | 检测页面链接是否返回错误状态码 |
| 6 | 页面加载超时 | 一般 | 首屏加载超过5秒判定为超时 |
| 7 | 接口响应慢 | 一般 | 接口响应超过3秒判定为慢 |
| 8 | 内容异常弹窗 | 严重 | 检测页面是否弹出加载失败/网络异常/未授权等错误弹窗 |

### 规则8：内容异常弹窗检测（_check_content_error）

检测页面加载后弹出"错误提示 + 确认/重试按钮"类异常弹窗。这类场景常见于商品详情页加载失败、网络异常、渠道未授权等，页面本身有内容（黑色背景+弹窗），不会被规则1（页面空白）检出。

**检测策略（三选一即判定异常）：**
1. 可见弹窗 + 弹窗内含错误关键词（加载失败/网络异常/未获授权/请重试…）
2. 可见弹窗 + 弹窗内含重试/确认类按钮（重试/刷新/我知道了/确定…）
3. 页面正文大面积覆盖错误关键词（无弹窗容器但整页提示加载失败）

**关键技术点：**
- **必须遍历所有 frame**：电信商城 PC 转 H5 模式（`#/pc2h5`）将商品页放在 iframe 内渲染，弹窗在 iframe 的 DOM 中而非主页面。只检测 `page` 上下文会漏判，必须遍历 `page.frames` 逐帧检测。
- **弹窗选择器需覆盖组件库 hash 类名**：Vue/React 组件库弹窗 class 是 hash 类名（如 `cardModal_card_modal_content__1RrTJ`、`baseModal_base_modal__3D0+4`），不含标准 `dialog/modal/popup` 关键词。选择器需同时匹配标准关键词和 `Modal/mask/overlay` 等子串。
- **错误关键词需含业务级异常**：除网络错误外，还需覆盖"未获授权""商品编码不存在""无渠道码""未上架"等业务异常文案。
- **动作关键词需含确认类按钮**：不仅检测"重试/刷新"，还需检测"我知道了/确定/知道了"等确认按钮（在错误弹窗上下文中视为异常确认）。
- **轮询等待异步渲染**：Vue/React SPA 弹窗可能延迟出现，每 500ms 轮询检测一次，最多等 4 秒。

**已验证的异常场景：**
- "当前销售渠道未获授权" + "我知道了"
- "商品编码不存在或还未上架" + "我知道了"
- "无渠道码！无法打开页面" + "我知道了"

### 新增巡检规则时的全栈改动清单

新增一条巡检规则需要同步修改以下文件：

| 文件 | 改动 |
|------|------|
| `backend/config.py` | CHECK_ITEMS 新增规则条目 |
| `backend/inspector.py` | InspectionResult 新增字段 + to_dict 输出 + `_check_xxx` 方法 + `_run_check` 映射注册 |
| `backend/database.py` | `insert_result` SQL 和 params 新增字段 |
| `init.sql` | inspection_results 表新增列定义 |
| `src/storage/database/shared/schema.ts` | Drizzle ORM schema 新增字段 |
| `src/app/tasks/new/page.tsx` | CHECK_ITEMS 数组新增选项 |
| `src/app/tasks/[id]/edit/page.tsx` | 同上 |
| `src/app/tasks/page.tsx` | CHECK_ITEM_LABELS 新增映射 |
| `src/app/results/page.tsx` | InspectionResult 接口新增字段 + 表头新增列 + 表格行新增 StatusBadge + colSpan 调整 |
| `src/app/api/results/route.ts` | ResultRow 接口新增字段（查询用 SELECT * 无需改 SQL） |
| 已有数据库 | `ALTER TABLE inspection_results ADD COLUMN IF NOT EXISTS xxx INTEGER DEFAULT -1;` |

## 已知坑与修复（务必遵守）

1. **load_time 只统计首次 page.goto 耗时**：页面被反爬拦截（非200）时会等待 5s 再重试，重试等待时间不得计入 load_time，否则同一站点加载时间在 2.6s~8s 波动、时超时不超时误报。修复方式：首次 goto 后立即记录 load_time，反爬重试逻辑不再覆盖该值。
2. **电信 H5 页面按钮/入口多为 div 模拟**：Vue 项目用 `<div>` + @click 模拟按钮和入口（无 onclick 属性、无 role），DOM 上看不出可点击特征。_check_button 和 _check_entry 已扩展为"特征识别 + 实际点击验证"：用 evaluate 收集候选（按钮样式/icon-card/onclick/role/Banner/浮窗/Tab），实际点击后检测 URL 变化/弹窗/内容变化/class 变化/新交互元素。详见 references/div-button-entry-detection.md。
3. **异常描述必须带定位信息**：按钮失效记录按钮文字（text_content 为空时回退取 value 属性，截断 50 字符）；死链接记录 `[链接文字] URL (状态码: xxx)`；跳转错误记录 `[链接文字] 链接格式异常: href`。业务人员靠这些信息定位问题。
4. **定时调度并发控制**：scheduled_inspection 用 `asyncio.Semaphore(2)` 限制最多 2 个并发，单任务超时 `TASK_TIMEOUT = 180`（3分钟），失败自动重试 1 次（MAX_RETRIES=1）。`/api/inspect/batch` 接口也走 scheduled_inspection（后台任务），不要回退到旧的 _run_batch_inspection 串行逻辑。
5. **Vue SPA 渲染时序坑**：`page.goto` 返回后 Vue 应用可能尚未渲染完成，此时 evaluate 收集候选按钮/入口会得到空列表（button_status/entry_response 返回 -1）。修复：候选为空时每 500ms 轮询重试收集，最多等 6 秒。不要用 `data-tib-marker` 临时属性标记元素做后续定位——Vue 重渲染会清除该属性导致"找不到元素"，改用 index 定位（重新收集候选列表按 index 取坐标，querySelectorAll 顺序稳定）。
6. **检查项间页面恢复**：按钮检测点击 div 按钮会触发 SPA 跳转（如 `#/index` → `#/ecsc-broadband`），后续入口检测会在错误页面执行导致误报。修复：`inspect` 的检查循环中每项检查前检测 `page.url != page_url`，偏离则 `page.goto(page_url)` 导航回去再执行。
7. **点击后导航异常处理**：点击 div 按钮触发跳转时 evaluate 会抛 "Execution context was destroyed"，不应判为"验证失败"而应视为"有响应"。用 `clicked` 标志区分点击前后异常：点击后的 evaluate 异常 = 响应成功（页面正在导航），点击前的异常 = 定位/滚动失败。
8. **Tab 切换检测需含 class 变化**：Tab 点击不改变页面内容长度（body.innerText.length 不变），但 active class 会变（如 `am-tabs-default-bar-tab-active`）。点击响应验证必须增加 class 状态对比，否则 Tab 切换被误判为"无响应"。
9. **内容异常弹窗可能在 iframe 内渲染**：电信商城 PC 转 H5 模式（`#/pc2h5`）将商品页放在 iframe 内，弹窗在 iframe DOM 中。规则8检测必须遍历 `page.frames` 逐帧执行检测脚本，只检测主页面会漏判。弹窗 class 可能是组件库 hash 类名（如 `cardModal_card_modal_content__xxx`），选择器需用子串匹配而非精确类名。
10. **代码更新后数据库表结构未同步导致巡检结果不入库**：git pull 拉取新代码后，`insert_result` 可能引用了数据库表里尚不存在的新列（如 `content_error`），导致每次插入都报 `column "xxx" does not exist`。而 `inspect_task` 接口的 except 块只 print 日志不返回错误给前端，前端显示"巡检完成"但实际未入库。**修复要点**：(a) 代码更新后立即对比 `init.sql` 与实际表结构，用 `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` 补齐缺失列；(b) `insert_result` 的 except 块必须返回 `success=False` 及错误信息给前端，不能静默吞掉异常；(c) 超时结果 dict 必须包含 `http_status: 0` 等所有 insert_result 需要的字段，否则 KeyError 也会被静默吞掉。
11. **废弃 mysql-client.ts 导致 TypeScript 编译失败**：`src/storage/database/mysql-client.ts` 是遗留文件（项目已改用 PostgreSQL），但 `next build` 的 TypeScript 检查会扫描所有 src 下文件。git pull 覆盖 package.json 后 mysql2 依赖丢失，导致编译报 `Cannot find module 'mysql2/promise'`。**修复方式**：确认该文件无任何引用后（grep `mysql-client|mysql2` 在 src 下无匹配），清空文件内容为 `export {};` 注释说明已废弃，不要安装 mysql2 依赖。
12. **page_title 超长导致 PostgreSQL 拒绝插入**：部分页面 title 可能超过 VARCHAR(256) 限制，`insert_result` 会报 `value too long for type character varying`。修复：inspector.py 中获取 page title 后截断到 256 字符：`result.page_title = title[:256] if title else None`。
13. **video.js 等视频播放器错误提示被规则8误判为异常弹窗**：部分商品页（如 FTTR 宣传页）使用 video.js 视频播放器，headless 浏览器无法播放视频时播放器自动弹出 `vjs-error-display vjs-modal-dialog` 错误提示（"此视频暂无法播放，请稍后再试"），class 含 `modal` 且内容含"请稍后再试"被规则8匹配为异常弹窗。人工访问时视频正常播放不会弹此错误，属 headless 环境特有问题。**修复方式**：(a) 规则8弹窗检测排除 video.js 等播放器错误元素（`vjs-error-display`、`vjs-modal-dialog`、`video-js`、`aliplayer`、`dplayer` 等 class）；(b) Chromium 启动参数添加 `--autoplay-policy=no-user-gesture-required` 和 `--use-fake-ui-for-media-stream` 尽量支持视频播放。
14. **PC 转 H5 路径被电信安全防护系统拦截（巡检盲区）**：`/page/newmall/index.html#/pc2h5?...` 路径会被电信安全防护系统拦截（405 Method Not Allowed 安全拦截页），页面整体白色无商品内容。但拦截页有文字内容（非空白，规则1不触发）、无弹窗（规则8不触发）、HTTP 200，现有 8 项规则均无法识别，属巡检盲区。而 `/scloud/account/orz/newmallgoods?...` 直连 H5 路径不受拦截、正常打开。**排查要点**：用户报告"页面空白"但巡检报正常时，检查任务 URL 路径是否为 `#/pc2h5` 格式，建议统一使用 `/scloud/account/orz/newmallgoods` 直连路径。

## IM 群机器人 webhook（向量微服务）

- 发送：`POST https://imtwo.zdxlz.com/im-external/v1/webhook/send?key=KEY`，JSON body，支持 text/image/file/news 四种类型
- 上传：`POST .../webhook/upload-attachment?key=KEY&type=TYPE`（1=图片，2=文件），multipart/form-data，≤30M
- 完整接口说明与 Python 调用示例见 references/im-webhook.md
- div 模拟按钮与入口检测方案（识别策略、点击验证、Vue SPA 坑）见 references/div-button-entry-detection.md

## 任务管理与巡检触发

### 电信商城 URL 渠道参数（fromid）

电信商城商品页（`zxkf.cq.189.cn`）的 URL 必须带 `fromid` 渠道参数才能正常访问，缺失时页面弹出"无渠道码！无法打开页面"异常弹窗（规则8 可检出）。

- `fromid=201`：标准/默认销售渠道，约 90% 任务使用
- 其他值（`389`、`20131351101`、`40602105011` 等）：特定活动或推广入口渠道
- 具体渠道名称映射属业务编码，需咨询商城/渠道侧业务人员

**排查要点**：用户手动复制链接测试报"无渠道码"异常但任务巡检正常时，检查用户链接是否缺少 `fromid` 参数。任务配置的 URL 应始终带正确的 `fromid`。

### 创建任务（前端 API，非后端）

后端无创建任务接口，任务通过前端 Next.js API 路由创建：

```bash
curl -X POST http://localhost:5000/api/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "page_name": "故障快修",
    "page_url": "https://wapcq.189.cn/page/fast-fault-repair-ts/#/index",
    "channel_type": "H5",
    "check_items": "1,2,3,4,5,6,7,8",
    "alert_receiver": "王",
    "robot_key": "你的机器人key"
  }'
```

- `task_id` 自动生成，格式 `T{YYYYMMDD}{random8}`（如 `T20260826ns8wemv7`）
- `check_items` 是逗号分隔的字符串，全选填 `"1,2,3,4,5,6,7,8"`
- 前端必须运行在 5000 端口

### 触发巡检

```bash
# 批量巡检（后台异步执行，走并发调度 scheduled_inspection）
curl -X POST http://localhost:8000/api/inspect/batch \
  -H "Content-Type: application/json" \
  -d '{"task_ids": [9,10,11]}'

# 单任务巡检（按数据库 task id）
curl -X POST http://localhost:8000/api/inspect/task/7

# 临时巡检（不依赖任务表，直接指定 URL）
curl -X POST http://localhost:8000/api/inspect \
  -H "Content-Type: application/json" \
  -d '{"task_id": 0, "page_url": "https://wapcq.189.cn/page/ecsc_servicepassword/#/", "check_items": ["1","2"]}'
```

### 查询巡检结果

```bash
# 最近巡检记录（含按钮/入口/跳转/链接/弹窗状态）
psql -U postgres -d inspection_db -c "
  SELECT r.id, t.page_name, r.load_time,
         r.button_status AS btn, r.entry_response AS entry,
         r.jump_status AS jump, r.link_status AS link,
         r.content_error AS popup,
         r.overall_result AS result, LEFT(r.error_desc, 100)
  FROM inspection_results r
  JOIN inspection_tasks t ON r.task_id = t.id
  ORDER BY r.check_time DESC LIMIT 10;"

# 查看所有任务
psql -U postgres -d inspection_db -c "
  SELECT id, task_id, page_name, check_items, status
  FROM inspection_tasks ORDER BY id;"
```

- `button_status` / `entry_response` / `jump_status` / `link_status` / `content_error`：-1=N/A, 0=异常, 1=正常
- `overall_result`：0=异常, 1=正常

### 前端"巡检总数"统计说明

前端结果页（`src/app/results/page.tsx`）显示的"巡检总数"等统计数字。**原版代码**中该值为 `results.length`，即前端 API 路由 `GET /api/results` 返回的数组长度，而该路由 SQL 硬编码 `LIMIT 200`，导致即使数据库有超过 200 条巡检结果，前端最多显示 200。**已修复**：results API 新增 `SELECT COUNT(*) FILTER(WHERE ...)` 全量统计查询（total/normal/abnormal），与列表分页查询分离；前端统计卡片改用 API 返回的 `stats` 字段。同时修复了 export API 的 `LIMIT 1000` 和 tasks API 的 `LIMIT 100`。

## 验证命令

```bash
# 后端健康检查（scheduler 应为 running）
curl -s http://localhost:8000/api/health

# 前端可访问（307 是 Next.js 正常重定向）
curl -s -o /dev/null -w "%{http_code}" http://localhost:5000

# Python 语法检查
python3 -c "import ast; ast.parse(open('backend/inspector.py').read()); print('OK')"

# 查看最近巡检记录
psql -U postgres -d inspection_db -c "SELECT id, task_id, page_url, check_time, load_time, overall_result, LEFT(error_desc, 80) FROM inspection_results ORDER BY check_time DESC LIMIT 5;"
```