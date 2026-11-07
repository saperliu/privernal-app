# architecture.md —— 架构与数据流

## 1. 总体架构

```
┌────────────────────────────────────────────────────────────┐
│                     privernal-app（uni-app x）                  │
│                                                              │
│  原生壳（.uvue 页面）                    web-view 容器页        │
│  ┌──────────────────────────┐   ┌───────────────────────┐   │
│  │ login / message /        │   │ webview.uvue          │   │
│  │ workbench / mine /       │   │  └─ iframe(PC 页面)   │   │
│  │ app-detail / change-pwd  │   └───────────┬───────────┘   │
│  └──────────┬───────────────┘               │               │
│             │ uni.request + Bearer          │ URL 携带       │
│             │ （utils/http.uts）            │ token/app/bare │
└─────────────┼───────────────────────────────┼───────────────┘
              ▼                               ▼
┌──────────────────────────┐   ┌─────────────────────────────────┐
│  privernal-platform 后端     │   │  privernal-platform-ui（PC 前端）    │
│  8600 单体 Spring Boot    │◄──┤  动态表单运行时 /builder/{code}    │
│  登录/权限/应用/菜单/消息/  │   │  审批办理 /flow/todo              │
│  流程(warm-flow)/动态数据  │   │  守卫注入：token→登录态 bare→全屏  │
└──────────────────────────┘   └─────────────────────────────────┘
        ▲
        │ PC 前端的动态数据接口（/dbs/open/dynamic/*）也走这里
```

三层职责：

1. **原生壳**：身份（登录/改密/登出）、导航（应用汇总与应用内菜单）、消息中心。只做这三件事，其余交给 PC 页面。
2. **web-view 承载层**：动态表单/大屏/审批等复杂页面 = PC 前端页面 + URL 参数注入登录态与上下文。
3. **后端**：唯一数据源，应用/菜单/表单定义/数据/消息全部实时拉取——这是"APP 免发版同步"的根基。

## 2. 模块划分（APP 端）

| 模块 | 文件 | 职责 |
| --- | --- | --- |
| 请求层 | `utils/http.uts` | 统一 request（Bearer 头/超时/双响应包裹解析）、401 全局处理、接口路径常量 |
| 存储层 | `utils/store.uts` | uni.storage 封装：域名/PC 地址/token/用户信息（UserInfo type） |
| 摘要 | `utils/sha512.uts` | 纯 UTS SHA-512（逐位算术，兼容 App 端无 crypto API） |
| 登录 | `pages/login/login.uvue` | 域名校验 → 登录 → getInfo 刷会话 → 落存储 → 进工作台；已登录直接进 |
| 工作台 | `pages/workbench/workbench.uvue` | 应用网格（授权过滤 + PLATFORM 角色过滤），onShow/下拉刷新 |
| 应用详情 | `pages/app-detail/app-detail.uvue` | 菜单树拉取 + 递归扁平化：目录作分组标题、叶子作功能图标宫格；**`/builder/{code}` 分发到原生渲染页**，其余功能拼 URL 进 webview（未显式配置 PC 地址时一次性提示实际地址） |
| 原生渲染器 | `renderer/schema.uts` + `renderer/data-service.uts` + `pages/dynform/dynform.uvue` | 动态表单原生渲染（详见 §3.4） |
| webview | `pages/webview/webview.uvue` | 解析 url/title 参数 → `<web-view :src>`；@error 弹模态（含地址与配置指引） |
| 消息 | `pages/message/message.uvue` | 系统消息（分页/已读/全读/角标）+ 工作流（计数卡 + 待办列表 + 跳转办理） |
| 我的 | `pages/mine/mine.uvue` | 缓存即时展示 + getInfo 刷新；改密入口；登出确认 |
| 改密 | `pages/change-password/change-password.uvue` | 客户端校验（非空/≥6 位/两次一致）→ PUT updatePwd（双 SHA512） |

页面注册在 `pages.json`（7 页 + tabBar 3 项）；入口 `main.uts`/`App.uvue` 为脚手架标准写法。

## 3. 核心数据流

### 3.1 登录与令牌

```
login.uvue
  1. 校验域名 http(s):// 前缀 → setBaseUrl / setPcUrl（持久化）
  2. POST /auth/login {loginName, password: sha512(明文), loginType: 1}
     └─ loginType=1（APP）：多组织账号直接发正式 token，不走 PC 的临时令牌+选组织流程
  3. data.token → setToken
  4. GET /back/system/user/getInfo（Bearer）
     └─ 必调：刷新服务端会话权限（不调则部分接口 403"没有权限"）
     └─ 取 userId/loginName/nickName/organizationName/departmentName/roleKeys → setUserInfo
  5. uni.switchTab → /pages/workbench/workbench
  （getInfo 失败不阻断进入，"我的"页会再刷一次）
```

令牌生命周期：所有请求 `Authorization: Bearer <token>`；任何响应 `code == 401` → 清 token/userInfo → `uni.reLaunch` 回登录页。登出走 `DELETE /auth/logout` 后本地清理。

### 3.2 工作台应用列表

```
workbench onShow / onPullDownRefresh
  → POST /back/system/appInfo/app/list（后端按用户角色授权过滤）
  → 前端二次过滤：code=PLATFORM 且 roleKeys 不含 SuperPlatformRole 则跳过
  → 映射 {id, code, appName, shortName, bgColor, fgColor}（配色见 DESIGN.md §1.4）
```

### 3.3 应用功能列表

```
app-detail onLoad(appId, appCode)
  → GET /back/system/menu/app/getRouters?id=&code=
  → data.routes：递归 walk
     ├─ 跳过 hidden==true 或 meta.hide==true 的节点（与 PC"隐藏菜单"语义一致）
     ├─ 目录（有 children）→ 作为分组标题，继续下钻
     └─ 叶子（有 path）→ 功能图标 {title, 首字, 截断名, path, 六色板配色}，归入当前分组
  → Map<分组, 功能项[]> 渲染：分组标题 + 4 列功能宫格
```

### 3.4 动态表单（原生渲染，2026-10-24 起生效）

```
app-detail 点击 /builder/{code} 菜单
  → uni.navigateTo /pages/dynform/dynform?code={表编码}&title&app
  → renderer/data-service: GET /back/dynamic/form/code/{code}（表单定义 metaData schema）
                          POST /back/dynamic/formColumn/list?formId（列配置 showList/showQuery）
  → renderer/schema: 组件树扁平化
     ├─ 支持的字段类型：input/textarea→文本、number→数字、date→日期picker、
     │               select/radio→选项picker、relation-attr→关联picker、user-select→用户picker
     ├─ 布局节点（form/row/col）穿透，子字段线性堆叠
     └─ 出现未支持类型（子表/部门选择等）→ allSupported=false → 回退页"打开 PC 版页面"（webview）
  → 列表态：**卡片主标题 = 数据标题（每行必有 title 字段）**；副字段 = 列配置 showList=1 按 sort 升序取前 4
     ├─ 列配置（PC 端"动态表单→列配置"弹窗的"列表是否显示/排序"）即移动卡片字段与顺序的配置入口，改完 APP 即时跟随、零发版
     ├─ 列配置中的系统列（created_name/created_time 等）显示中文名（schema.systemLabelOf）
     ├─ relation/user 字典预载（relationList/userList）→ 显示翻译后文本
     ├─ showQuery varchar 首列（按 sort）→ 顶部搜索（等值过滤，空值不上送）
     └─ 触底分页（pageNum/pageSize）
  → 详情态：GET /dbs/open/dynamic/{tableCode}/{id}（行内 snake/camel 双键，rowValue 兜底）
  → 表单态：按 schema 渲染输入控件（picker 选项来自 props.options/字典），保存
     POST /dbs/open/dynamic/data/add {…值, tableCode, id?}（新增不带 id；空值不上送）
  → 删除：DELETE /dbs/open/dynamic/{tableCode}/{id}（确认弹窗）
非 /builder 菜单（大屏/报表/系统管理等）仍走 §3.5 的 webview 链路，两轨并存。
```

要点：**表单 schema/列配置/数据全部实时拉后端——PC 端新建表单/改字段，APP 原生渲染即时跟随，免发版机制不变**；仅"新增设计器组件类型"需要渲染器跟进（未跟进类型自动回退 webview，不阻塞）。PC 端零改动。

### 3.4b webview 通用链路（非 /builder 菜单）

```
app-detail 点击其他菜单（大屏/报表/系统管理页等）
  1. URL = getPcUrl() + 菜单path + '?bare=1&app=' + appCode + '&token=' + getToken()
  2. navigateTo /pages/webview/webview?url=encodeURIComponent(URL)&title=encodeURIComponent(标题)
  3. web-view(iframe) 加载 PC 页面；PC 守卫注入 token（免登）+ bare=1（全屏）+ 按应用装载路由
```

两轨并存：动态表单走原生渲染轨，其余 PC 页面走 webview 轨；渲染器未支持的表单类型也回退 webview 轨。

### 3.5 消息

```
message onShow
  ├─ 系统消息：POST /app/system/systemMessage/user/list {pageNum, pageSize:15}
  │    ├─ 客户端过滤 messageType=WORKFLOW 的行（任务类归"任务消息"分段，本段只留通知/报警）
  │    ├─ 未读角标 = 本段未读数（只统计非 WORKFLOW 行），两段角标常显
  │    ├─ 点卡片：未读则 PUT /app/system/systemMessage/user/read/{id} → 本地置已读 → showModal 展示正文
  │    ├─ 全部已读：PUT readAll → 失败时降级逐条 read/{id}（后端 selectSystemMessageListNoRead
  │    │    的 SQL 尾逗号语法错已修，待 IDEA 重启生效；重启前走降级路径）
  │    └─ scrolltolower 触底 → 下一页 append
  └─ 任务消息（统一列表 + 三口径统计，全部真实接口）
       ├─ 统计卡：GET /back/flow/task/count（todo/done/start，Long→String 兜底，纯统计不切换）
       ├─ 统一列表 = 待办任务（POST /back/flow/task/todo/list，"待办"橙标签）
       │            + 审批/工单消息（POST user/list + messageType=WORKFLOW，标签按标题关键词 审批/超时/工单）
       │    按时间 ISO 串降序混排在同一个列表；his_task.flowName 空兜底 nodeName；ISO 时间格式化
       ├─ 点击直达业务表单（APP 原生渲染页，非 PC 页面）：
       │    任务条目 → 原生 dynform 页 /pages/dynform?code={formPath}（todo/list 行自带 formPath=表单编码）
       │    消息条目 → detailUrl=/builder/{tableCode} 时提取编码进原生 dynform（存量消息已批量修复
       │              detail_url，经 content 单据编号→instance→definition.form_path；后端新消息由
       │              FlowNoticeSender 直发该路径）；/flow/instance 等流程页仍由 PC 承载
       └─ "全部已读"接口失败时降级为逐条 PUT read/{id}（静默失败，不弹错误）
```

### 3.6 我的 / 改密 / 登出

```
mine onShow：缓存 getUserInfo() 即时渲染 → GET getInfo 静默刷新覆盖
改密：客户端校验 → PUT /back/system/user/manual/updatePwd {oldPassword: sha512, password: sha512}
      （两个都是摘要，与后端 matchesPassword(摘要) 口径对齐）→ toast → navigateBack
登出：showModal 确认 → DELETE /auth/logout（失败也继续）→ 清 token/user → reLaunch 登录页
```

## 4. 存储设计（uni.storage）

| 键 | 内容 | 写入方 |
| --- | --- | --- |
| `va_base_url` | 服务器域名（末尾无斜杠） | 登录页 |
| `va_pc_url` | PC 页面地址（空则取 base 的 origin） | 登录页高级设置 |
| `va_token` | 登录令牌 | 登录成功 / 401 清空 |
| `va_user` | UserInfo JSON（userId/loginName/nickName/organizationName/organizationId/departmentName/departmentId/roleKeys 逗号串） | 登录 getInfo / mine 刷新 |

注意：与 PC 端存储（`PRIVERNAL_token`）完全独立——webview 内 PC 页写自己的 localStorage，两套互不干扰。

## 5. 后端接口清单（APP 消费的）

| 方法 路径 | 用途 | 鉴权 |
| --- | --- | --- |
| POST `/auth/login` | 登录（loginType=1） | 免鉴权 |
| GET `/back/system/user/getInfo` | 用户信息 + 刷新会话权限 | Bearer |
| POST `/back/system/appInfo/app/list` | 应用列表（按用户授权） | Bearer |
| GET `/back/system/menu/app/getRouters?id=&code=` | 应用菜单树 | Bearer |
| POST `/app/system/systemMessage/user/list` | 用户消息分页 | Bearer |
| PUT `/app/system/systemMessage/user/read/{messageId}` | 单条已读 | Bearer（依赖后端 upsert 修复） |
| PUT `/app/system/systemMessage/user/readAll` | 全部已读 | Bearer |
| POST `/back/flow/task/todo/list` | 待办分页 | Bearer + `flow:task:list` |
| GET `/back/flow/task/count` | 待办/已办/我发起计数 | Bearer + `flow:task:list` |
| PUT `/back/system/user/manual/updatePwd` | 修改密码（双 SHA512） | Bearer |
| DELETE `/auth/logout` | 登出 | Bearer |

响应包裹：`{code, message, data}`；分页为 `{code, message, data:{total, list}}`。webview 内 PC 页面另走 PC 既有接口（动态数据 `/dbs/open/dynamic/*` 等），不在本清单。

## 6. PC 端耦合点（唯一）

`privernal-platform-ui/src/router/guard/route.ts` → `initRoute()` 顶部两段：

```ts
// ?token= 注入登录态；?bare=1 全屏内容模式（仅在嵌入态生效）
const appToken = typeof to.query.token === 'string' ? to.query.token : '';
if (appToken) { localStg.set('token', appToken); }
if (to.query.bare === '1') { useAppStore().fullContent = true; }
```

其余 PC 代码零改动。深链 `?app=` 按应用装载路由是 PC 守卫既有能力。

## 7. 设计决策记录（ADR 摘要）

| 决策 | 理由 | 代价/备选 |
| --- | --- | --- |
| 动态表单用 web-view 承载 PC 页面，而非 UTS 重写 | PC 运行时能力 1:1（字段类型/联动/校验/导入导出/列配置），且天然免发版同步；重写成本高且双维护 | 依赖 PC 前端可用；App 端 WebView 性能可接受（表单页非高频重绘）；备选"UTS 原生渲染表单"列为远期 |
| 壳侧只做 3 TAB + 登录 | 保持壳稳定，发版频率低 | — |
| 登录 loginType=1 | 复用 PC 登录接口，APP 端免选组织（后端对非 PC 端直接发正式 token） | 多组织用户默认组织，切换组织入口待规划 |
| 纯 UTS sha512 而非原生插件 | 零原生依赖，跨端一致 | 实现需对拍保证正确性（踩过 rotr64 坑，已修） |
| 401 全局清态回登录页 | 简单可靠 | webview 内 PC 页的 401 由 PC 自己的登录跳转处理，与壳互不影响 |
| token 不进 URL（一次性行权码握手） | APP 先 `POST /auth/web/handshake` 换 60 秒单次 `ek`，PC 兑换后立即剥码重定向；token 不进 URL/日志/历史 | 新旧 APP 与 PC 需同批发布（旧版传 `?token=`，新接收端不认） |

## 8. 平台矩阵与差异（三端硬约束）

项目约束：代码必须三端可运行（Web/App/小程序），APP 保持可打包 iOS/Android。各端差异与运行前提：

| 差异点 | Web（PC） | App（Android/iOS） | 小程序（微信，≥4.41） |
| --- | --- | --- | --- |
| 跨域 CORS | 存在（后端已开放，反射 Origin） | 不存在（原生 http） | 不存在 |
| web-view 承载 PC 页 | 全屏 iframe；跨域 iframe 内 DOM 外层不可读 | 全屏原生 WebView，可自由布局 | 全屏组件；**src 必须 https 且域名在小程序后台配置"业务域名"；个人主体小程序不支持 web-view** |
| http 明文 | 可用（联调） | debug 基座可；**release 禁止**（Android 明文策略 / iOS ATS） | **必须 https**，且配置 request 合法域名（生产） |
| 本地联调 | localhost 直接可 | 标准基座（debug）可 http | 开发者工具可（manifest urlCheck=false 已设）；**真机预览不可 http** |
| 令牌/存储 | uni.storage（`va_*` 键） | 同左 | 同左 |
| 登录接口 | 同一套 REST | 同一套 | 同一套 |

条件编译现状：仅 `App.uvue` 的 Android 返回键逻辑（`#ifdef APP-ANDROID || APP-HARMONY`）一处；其余代码三端通用。新增平台差异必须走 `#ifdef` 并给默认降级。

通用备忘（三端共有）：
- `uni.request` App 端无 Promise，统一回调封装；
- 后端 Long→String 序列化：含 19 位 id 的响应取值一律按 string 处理（分页 `total` 亦为字符串）。
