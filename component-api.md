# component-api.md —— 组件相关

本项目**没有独立抽取的自定义组件目录**（uni-app x 支持 `components/` easycom，但当前页面少，复用通过"工具模块 + 模式片段"实现）。本文档覆盖：工具模块 API 参考、可复用 UI 片段库、内置组件用法要点、web-view 契约、扩展指引。

## 1. 工具模块 API 参考

### 1.1 `utils/http.uts` —— 请求层

```ts
type HttpDone = (ok: boolean, res: UTSJSONObject | null, msg: string) => void

function http(method: string, path: string, data: UTSJSONObject | null, done: HttpDone): void
function tableList(res: UTSJSONObject | null): Array<any> | null   // TableResult → data.list
function tableTotal(res: UTSJSONObject | null): number             // TableResult → data.total（⚠️ 字符串隐患见 TODO #1）
```

- `method`：'GET' | 'POST' | 'PUT' | 'DELETE'（内部 `as RequestMethod`）。
- 自动带 `Authorization: Bearer <token>`（无 token 时省略）；`Content-Type: application/json`；超时 20s。
- 成功判定：HTTP 200 且 `code == 200`；否则 `done(false, res, message)`。
- **401 全局处理**：清 token/userInfo → `reLaunch` 登录页，并回调 `done(false, …, '登录已过期…')`。
- 网络失败：`done(false, null, err.errMsg)`。

接口常量/函数：

| 成员 | 说明 |
| --- | --- |
| `apiLogin(loginName, passwordSha512, done)` | POST /auth/login，body 内置 `loginType: 1` |
| `API_GET_INFO` | GET 用户信息（登录后必调） |
| `API_APP_LIST` | POST 应用列表 |
| `apiUrlAppRouters(appId, appCode)` | GET 应用菜单树（拼 query） |
| `API_USER_MESSAGE_LIST` | POST 用户消息分页 |
| `apiUrlMessageRead(messageId)` | PUT 单条已读（拼 path） |
| `API_MESSAGE_READ_ALL` | PUT 全部已读 |
| `API_FLOW_TODO_LIST` / `API_FLOW_TASK_COUNT` | 流程待办/计数 |
| `API_UPDATE_PWD` / `API_LOGOUT` | 改密/登出 |

**新增接口**：在文件尾部加常量（无参路径）或 `api*()` 函数（带参拼装），页面只 import 使用。

### 1.2 `utils/store.uts` —— 存储层

```ts
type UserInfo = {
  userId: string, loginName: string, nickName: string,
  organizationName: string, organizationId: string,
  departmentName: string, departmentId: string,
  roleKeys: string   // 逗号拼接，如 "SuperRole,SuperPlatformRole"
}

function getBaseUrl(): string        // 服务器域名（已去尾斜杠）
function setBaseUrl(url: string)
function getPcUrl(): string          // PC 页面地址；未设置时 = originOf(getBaseUrl())
function setPcUrl(url: string)
function originOf(url: string): string  // 取 http(s)://host:port
function getToken() / setToken(t) / clearToken()
function getUserInfo(): UserInfo | null / setUserInfo(u) / clearUserInfo()
function isLoggedIn(): boolean       // token 非空
```

存储键：`va_base_url` / `va_pc_url` / `va_token` / `va_user`（详见 architecture.md §4）。

### 1.3 `utils/sha512.uts`

```ts
function sha512(input: string): string   // 128 位小写十六进制
```

登录与改密共用口径。**修改前必读 AGENTS.md "SHA-512" 段**（Python 镜像对拍流程）。

### 1.4 `renderer/schema.uts` —— 动态表单 schema 解析（原生渲染器）

```ts
type FieldKind = 'text' | 'number' | 'date' | 'select' | 'relation' | 'user'
type RenderField = { field, label, kind, rawType, options: FieldOption[], relationCode, relationDisplay }
type FormSchemaInfo = { formId, tableName, tableCode, isWorkflow, fields, listFields, queryField, allSupported, unsupportedTypes }

function parseFormDetail(res: UTSJSONObject): FormSchemaInfo | null  // 表单定义 → 渲染字段（不认识类型进 unsupportedTypes）
function mergeColumns(info: FormSchemaInfo, res: UTSJSONObject)      // 列配置 → listFields（showList=1 按 sort 升序，PC 端列配置即移动卡片配置）/queryField（showQuery varchar 按 sort）
function rowValue(row: UTSJSONObject, field: string): string         // 行取值 snake 优先 camel 兜底（详情接口双键）
function displayValue(field, raw, dictLabel): string                 // 选项映射/日期截断/字典翻译
function systemLabelOf(code): string                                 // 系统列中文名（title→数据标题/created_name→创建人等）
function systemValueOf(raw): string                                  // 系统值整理（时间戳截到分钟）
function anyToString(v: any): string
```

### 1.5 `renderer/data-service.uts` —— 动态数据服务

```ts
fetchFormByCode(code, done)                 // GET /back/dynamic/form/code/{code}
fetchColumns(formId, done)                  // POST /back/dynamic/formColumn/list（必须按 formId 过滤，否则返回全部表单的列）
fetchRows(tableCode, pageNum, pageSize, queryField, queryValue, done)  // 列表分页+等值过滤（空值不上送）
fetchRow(tableCode, id, done)               // 详情
saveRow(tableCode, editId, values: Map<string,string>, done)  // 保存（新增无 id；空值不写）
deleteRow(tableCode, id, done)
fetchRelationOptions(relationCode, fromTableCode, displayField, done)  // 关联字段选项
fetchUserOptions(done)                      // 用户字典（user-select）
```

页面消费方：`pages/dynform/dynform.uvue`（三态：卡片列表/详情/表单；不受支持类型回退 webview）。

## 2. 可复用 UI 片段库

以下片段是全项目复用的"事实组件"：复制 markup + 对应样式类即可（样式已在各页面内，遵循 DESIGN.md 同名规范）。

### 2.1 表单项 form-item（登录/改密页）

```html
<view class="form-item">
  <text class="label">标签</text>
  <input class="input" v-model="val" placeholder="提示" placeholder-style="color:#c0c6d2" />
  <text class="tip">辅助说明</text>   <!-- 可选 -->
</view>
```

### 2.2 信息行 row（我的页）

```html
<view class="group">
  <view class="row"><text class="row-label">组织</text><text class="row-value">系统企业</text></view>
  <view class="row row-click" @click="go">     <!-- 可点击行：末行加 row-click 去分隔线 -->
    <text class="row-label">修改密码</text><text class="row-arrow">›</text>
  </view>
</view>
```

### 2.3 分组功能列表 section（应用详情页）

```html
<view class="section" v-for="s in sections" :key="s.title">
  <text class="section-title">{{s.title}}</text>
  <view class="menu-card">
    <view class="menu-item" v-for="it in s.items" :key="it.path" @click="openMenu(it)">
      <view class="menu-dot" :style="'background-color:' + accentColor"></view>
      <text class="menu-title">{{it.title}}</text><text class="menu-arrow">›</text>
    </view>
  </view>
</view>
```

### 2.4 分段控制器 seg（消息页）

```html
<view class="seg">
  <view class="seg-item" :class="seg=='sys' ? 'seg-active' : ''" @click="switchSeg('sys')">
    <text class="seg-text" :class="seg=='sys' ? 'seg-text-active' : ''">系统消息</text>
    <view class="badge" v-if="n>0"><text class="badge-text">{{n}}</text></view>
  </view>
  <!-- 第二段同理 -->
</view>
```

### 2.5 卡片列表 + 未读点（消息页）

```html
<view class="msg-card" v-for="m in messages" :key="m.id" @click="open(m)">
  <view class="msg-head"><text class="msg-title">{{m.mainTitle}}</text><view class="dot" v-if="!m.read"></view></view>
  <text class="msg-sub" v-if="m.subTitle!=''">{{m.subTitle}}</text>
  <view class="msg-foot"><text class="msg-type">{{m.typeText}}</text><text class="msg-sender">{{m.createdName}}</text></view>
</view>
```

### 2.6 空态 / 加载 / 计数卡

```html
<view class="empty" v-if="!loading && list.length==0"><text class="empty-text">暂无数据</text></view>
<view class="loading" v-if="loading"><text class="loading-text">加载中…</text></view>
<view class="count-card"><text class="count-num">{{n}}</text><text class="count-label">待办任务</text></view>
```

### 2.7 功能宫格 tile（应用详情页）

```html
<view class="section" v-for="s in sections" :key="s.title">
  <text class="section-title">{{s.title}}</text>
  <view class="grid-card">
    <view class="tile" v-for="it in s.items" :key="it.path" @click="open(it)">
      <view class="tile-icon" :style="'background-color:' + it.bgColor">
        <text class="tile-icon-text" :style="'color:' + it.fgColor">{{it.short}}</text>
      </view>
      <text class="tile-name">{{it.name}}</text>
    </view>
  </view>
</view>
```

要点：菜单目录（有 children）→ 分组标题；叶子功能 → 4 列 tile（宽 25%）；图标块 48px/圆角 12px 六色循环（色板见 DESIGN.md §6）；名称超 5 字**代码侧**截断加"…"（跨端 text 省略号表现不一，不要用 CSS lines/ellipsis）。参考实现 `pages/app-detail/app-detail.uvue`。

**若某片段被第 3 处使用，考虑真正抽组件**：建 `components/<name>/<name>.uvue`，uni-app x easycom 自动注册（`<name>` 直接可用），props 用 type 标注、事件 `@` 回调——注意 UTS 组件 props 类型必须显式（`@Prop` 语法在 uni-app x 中为普通 props 声明），样式仍受 DESIGN.md §7 约束。

## 3. 内置组件用法要点（本项目实测）

| 组件 | 要点 |
| --- | --- |
| `view` | 默认 flex column；横向必须显式 `flex-direction: row`；文字不能直接放 view 里（App 端不渲染） |
| `text` | 文字容器；字号/颜色/对齐设在此；`{{}}` 插值可用 |
| `input` | 密码用 `:password="true"`（非 type）；v-model 双向；`@confirm` 监听键盘确认；placeholder 色用 `placeholder-style`（web 端 placeholder 不落 input 属性，自动化定位用 role） |
| `button` | **本项目禁用**：默认 `::after` 伪元素描边跨端无法去除（见 DESIGN.md §7.9）。按钮一律 `view + text` 自绘：view 类管 `height/border-radius/background-color + align-items/justify-content: center`，text 类管 `font-size/color`；加载态文字切换（"保存中…"） |
| `scroll-view` | 列表区标配：`scroll-y` + `flex:1`；触底加载 `@scrolltolower`；页面级下拉刷新走 pages.json `enablePullDownRefresh` + `onPullDownRefresh` + `uni.stopPullDownRefresh()` |
| `web-view` | `:src` 全屏承载网页；`@error` 失败提示；**web 平台渲染为全屏 iframe**（App 端可自由布局）；src 变更需 v-bind 响应式 |
| `image` | 目前仅 tabBar 图标经 pages.json 引用；组件内未用 |
| `uni.showToast` | 本项目统一 `icon: 'none'`（success 图标各端表现不一） |
| `uni.showModal` | 确认框/富文本详情；标题+content；确认/取消文案系统默认（OK/Cancel） |
| `uni.setNavigationBarTitle` | 页面标题动态化（app-detail/webview 均用） |
| 导航 | tab 页 `uni.switchTab`；普通页 `uni.navigateTo` / `uni.navigateBack`；重置 `uni.reLaunch`（登录态变化） |

## 4. web-view URL 契约

### 4.1 容器页参数（APP 内部）

`/pages/webview/webview?url=<encodeURIComponent(目标URL)>&title=<encodeURIComponent(标题)>`
容器内 `decodeURIComponent` 一次后赋给 `<web-view :src>`。

### 4.2 目标 URL 参数（PC 守卫消费）

| 参数 | 必带 | PC 端行为 |
| --- | --- | --- |
| `token` | ✅ | 写入 `PRIVERNAL_token`（免二次登录） |
| `bare=1` | ✅ | `appStore.fullContent=true`（隐藏侧栏/头部/页签） |
| `app` | 建议 | 深链按应用装载路由（菜单权限收口） |

### 4.3 注意事项

- 目标 URL = `getPcUrl() + 菜单path`；菜单 path 自带 query 时需 `&` 合并（当前实现直接拼 `?`，见 TODO P1 #4）。
- PC 端消费点唯一：`privernal-platform-ui/src/router/guard/route.ts` 的 `initRoute()` 顶部两段（详见 architecture.md §6），两端契约变更必须同步。
- 跨域 iframe 内页面 DOM 外层不可读（调试见 development.md §5）。
- **跨端（硬约束）**：web-view 三端均可全屏承载，但小程序端 src 必须 https 且域名在小程序后台配置"业务域名"白名单（个人主体不支持 web-view）；差异矩阵见 architecture.md §8。

## 5. pages.json 速览

| 块 | 本项目取值 |
| --- | --- |
| `pages` | 7 页；第一项 `pages/login/login` 为启动页；workbench 开 `enablePullDownRefresh` |
| `tabBar` | 3 项（消息/工作台/我的），color `#9098a8`、selectedColor `#2f6bff`、图标 static/tab-*.png（81px） |
| `globalStyle` | 导航白底黑字、标题"春祈平台"、页面背景 `#f4f6fa` |

**新增页面步骤**：① `pages/<name>/<name>.uvue`；② pages.json `pages` 数组登记（含导航标题；tab 页需同步 tabBar）；③ 接口需求在 utils/http.uts 加；④ 样式按 DESIGN.md；⑤ 跑 development.md 回归清单相关项。全程遵守三端硬约束（AGENTS.md #5）：只用跨端 API、平台差异走 `#ifdef`、不访问浏览器专属对象。
