# AGENTS.md —— AI 进入本项目必读

本文件是 AI 助手接手 privernal-app 时的第一份读物。先读完本文件再动代码；细节展开见文末文档索引。

## 项目一句话

uni-app x 移动工作台 APP：原生壳（登录 + 消息/工作台/我的 3 个 TAB）+ 动态表单**原生渲染器**（`renderer/` + `pages/dynform`）+ web-view 承载其余 PC 页面，与 PC 端共用后端 privernal-platform（单体，默认 8600）。核心卖点：应用/菜单/表单全部实时拉接口，PC 端改动无需 APP 发版。

## 硬约束（违反即翻车）

1. **uni-app x 不是普通 Vue 项目**：
   - 页面文件是 `.uvue`，逻辑语言是 **UTS**（不是 TS/JS，编译到 App 端是 Kotlin/Swift）；
   - **不支持 npm CLI 创建项目**（官方明示），必须 HBuilderX；项目无 package.json、无 node_modules，**禁止引入 npm 依赖**；
   - 样式是受限 CSS 子集（见 DESIGN.md §uni-app x 样式约束）。
2. **本机工具链**：HBuilderX 装在 `~/Applications/HBuilderX.app`（5.26.2026091802）。cli 路径 `~/Applications/HBuilderX.app/Contents/MacOS/cli`。`lsp lint` **不要传 `--platform`**（会报 "Failed to read platform settings"，是 cli 的坑不是项目问题）；lint 前项目须已 `project open` 导入。
3. **后端由用户 IDEA 调试进程启动，不要替用户重启/杀掉 8600**。判断"改了没生效"先比进程启动时间与源码 mtime。
4. **不要动 `utils/sha512.uts`**，除非同时用 Python 镜像对拍（见下文"SHA-512"）。登录/改密都靠它。
5. **三端兼容 + 双端打包是硬约束**：任何代码改动必须保证 Web（PC）、App（Android/iOS）、小程序（微信 ≥4.41）三端可编译运行，且 APP 保持可云打包 iOS/Android：
   - 只用 uni-app x 跨端 API；平台差异逻辑用条件编译 `#ifdef`（范例：App.uvue 的 Android 返回键 `#ifdef APP-ANDROID || APP-HARMONY`）；
   - **禁止访问 window / document / localStorage / navigator 等浏览器专属对象**——存储/网络/导航一律走 uni API（当前代码零浏览器对象依赖，保持住）；
   - 样式按 DESIGN.md §7 受限子集书写（App 端最严口径，三端天然一致）；
   - web-view 是动态表单承载的跨端基石：小程序端要求 **https + 小程序后台"业务域名"白名单**，且**个人主体小程序不支持 web-view**（三端差异矩阵见 architecture.md §8）；
   - 发版前跑三端回归：development.md §4 清单（web）+ #17 小程序冒烟；打包按 development.md §6 打包矩阵。

## UTS 语言深坑清单（本项目实测踩过）

| 坑 | 正确姿势 |
| --- | --- |
| `decodeURIComponent()` 返回 `string \| null` | 调用后必须判空再赋值 |
| `!= null` 对 `any` 类型**不收窄**，复合 `&&` 条件也不收窄 | `query?.x as string \| null` 显式转型后再判空；需要时用非空断言 `!` |
| `UTSArray` 在 web 平台是未知名 | 类型标注一律写 `Array<any>` |
| `uni.request` 的 `method` 参数是 `RequestMethod` 枚举 | 字符串变量要 `as RequestMethod` |
| 对象字面量不能 `as` 成自定义 type | 声明时标注：`const r: W64 = { hi: a, lo: b }` |
| `uni.request` 在 App 端**不支持 Promise** | 统一走回调封装（`utils/http.uts` 的 `http()`），不要 async/await 化 |
| Map 可用（`new Map<string, T>()`，`forEach((v,k)=>)`）；模板字符串可用，但项目惯例用 `+` 拼接 | 保持一致 |
| `JSON.parse<T>(s)` 类型不符返回 null | 平台响应一律 `UTSJSONObject` + `getString/getNumber/getJSON/getArray` 取值 |

**SHA-512**：App 端无 crypto API，`utils/sha512.uts` 是逐位算术纯 UTS 实现（64 位字拆 hi/lo 两个 number）。历史 bug：`rotr64` 必须分解为 `shr64(n) ∪ shl64(64-n)`（当初写成 shl(n) 导致登录"密码错误"）。改动它必须先用 Python 逐行镜像实现，与 `hashlib.sha512` 对拍公开标准向量（如 `'abc'`、空串边界、63/64/127 字节长串、中文）全 MATCH 后再改。**禁止把真实口令写入文档、用例或注释**。

## 接口契约速查

响应包裹两种：
- **CommonResult**：`{ code, message, data }`，`code == 200` 成功；
- **TableResult**（分页）：`{ code, message, data: { total, list } }`。注意后端 Long→String 序列化，`total` 可能是字符串数字，`getNumber('total')` 有返回 null 的隐患（见 TODO.md #1）。

登录后**必须调一次** `GET /back/system/user/getInfo` 刷新会话权限，否则部分接口报"没有权限"。

| 方法 路径 | 用途 | 备注 |
| --- | --- | --- |
| POST `/auth/login` | 登录 | body `{loginName, password: <SHA512 摘要，禁止传明文>, loginType: 1}`；`data.token`；loginType=1（APP）多组织账号直接获得正式 token，不走 PC 选组织流程 |
| GET `/back/system/user/getInfo` | 用户信息 | 组织/部门/姓名/roleKeys；登录后必调 |
| POST `/back/system/appInfo/app/list` | 应用列表（按用户授权） | 无参数；PLATFORM 应用仅 SuperPlatformRole 可见 |
| GET `/back/system/menu/app/getRouters?id=&code=` | 应用菜单树 | `data.routes`；过滤 `hidden`/`meta.hide`，children 递归 |
| GET `/back/dynamic/form/code/{code}` | 动态表单定义 | `data.metaData.schemas` 组件树；渲染器契约见 `renderer/schema.uts` |
| POST `/back/dynamic/formColumn/list` | 列配置 | **必须传 formId**（传 tableCode 返回全部表单的列）；showList/showQuery 是字符串 "0"/"1" |
| POST `/dbs/open/dynamic/data/list` | 动态数据分页 | body `{tableCode, pageNum, pageSize, <字段等值过滤>}`，空值不上送 |
| GET `/dbs/open/dynamic/{tableCode}/{id}` | 动态数据详情 | **行内 snake/camel 双键返回**，取值必须走 `schema.rowValue` 兜底 |
| POST `/dbs/open/dynamic/data/add` | 动态数据保存 | body `{…表单值, tableCode, id?}`；新增不带 id；空值不写 |
| DELETE `/dbs/open/dynamic/{tableCode}/{id}` | 动态数据删除 | |
| POST `/dbs/open/dynamic/data/relationList` | 关联字段选项 | body `{tableCode:被关联表, fromTableCode, displayField}` |
| POST `/back/system/user/list` | 用户字典 | user-select 字段翻译/选项（pageNum/pageSize） |
| POST `/app/system/systemMessage/user/list` | 用户消息分页 | body `{pageNum, pageSize}` |
| PUT `/app/system/systemMessage/user/read/{messageId}` | 单条已读 | ALL 消息已读依赖后端 upsert 修复（待重启，见 TODO.md #0） |
| PUT `/app/system/systemMessage/user/readAll` | 全部已读 | |
| POST `/back/flow/task/todo/list` | 待办分页 | 权限 flow:task:list |
| GET `/back/flow/task/count` | 待办/已办/我发起计数 | `data.todo/done/start`（同 total 字符串隐患） |
| PUT `/back/system/user/manual/updatePwd` | 改密 | body `{oldPassword, password}`，**两个都传 SHA512 摘要** |
| DELETE `/auth/logout` | 登出 | 免鉴权白名单内 |

封装入口全在 `utils/http.uts`（`http()` / `tableList()` / `tableTotal()` / `api*` 常量）。任何接口 `code == 401` 会自动清登录态并 `reLaunch` 到登录页。

## web-view 桥接契约（与 PC 前端）

APP 打开 PC 页面的 URL 规则：`getPcUrl() + 菜单path + ?bare=1&app=<应用code>&token=<token>`，经 `encodeURIComponent` 一次传入 webview 页。

| 参数 | 消费方（PC `src/router/guard/route.ts` initRoute 顶部） | 作用 |
| --- | --- | --- |
| `token` | 写入 `PRIVERNAL_token`（PC localStorage） | 免二次登录 |
| `bare=1` | `appStore.fullContent = true` | 隐藏 PC 侧栏/头部/页签 |
| `app` | 守卫深链参数 → 按应用装载路由 | 菜单权限收口 |

**PC 端唯一允许的耦合点就是上面那个守卫文件的两段代码，别挪、别删。** 改桥接契约要两端同步改。

## 常用命令

```bash
CLI=~/Applications/HBuilderX.app/Contents/MacOS/cli
$CLI open                                              # 启动 HBuilderX
$CLI project open --path <abs>                         # 导入项目（lint/运行前必须）
$CLI launch web --project <abs>                        # 运行到浏览器（后台常驻，dev 5173）
$CLI logcat web --project <abs>                        # 查看 web 运行日志
$CLI lsp lint --file <file> --project <abs>            # 无头诊断（不带 --platform！）
$CLI launch app-android --project <abs>                # 运行到 Android 基座
$CLI launch app-ios --project <abs> --iosTarget simulator   # 运行到 iOS 模拟器（需 Mac+Xcode）
$CLI launch mp-weixin --project <abs>                  # 运行到微信开发者工具（uni-app x ≥4.41；cli 对小程序命令标注"仅适用 uni-app 项目"，以实际编译为准，必要时用 HBuilderX 面板运行）
```

改完代码 HMR 自动热更；改 `.uts` 工具文件后建议整页 reload 再验（与前端 HMR 同理）。

## 测试方法与环境坑

- 主力验证：web 平台 + 浏览器自动化。**IAB 自动化环境的点击/截图管线可能整体停摆**（事件探针为空、cua/playwright click 全超时、截图报 surface preparation timed out），但 fill/evaluate/DOM 快照正常——绕过法：`evaluate` 里对目标元素 `el.click()`（应用自身 handler 正常执行）；tabbar 点 `.uni-tabbar__item`；uni.showModal 确认键是 `.uni-modal-dialog__action-text` 文本 OK/Cancel；uni-app x 按钮是 `uni-button`。截图尽早做，别留到最后。
- uni-app x web 端 placeholder 不是 input 的属性，locator 用 `getByRole('textbox')` 按序取。
- web 平台 web-view 是全屏 iframe；跨域 iframe 内 DOM 外层读不到（frameLocator 的 evaluate 也拿不回对象），验证 PC 页内部状态要同源直达 URL 开新标签。
- 回归清单见 [development.md](development.md) §E2E 回归清单，改完必跑。

## 红线与惯例

- **凭据红线**：token/密码等凭据只允许进入本地存储与请求头，**不得写入日志、URL、文档、示例、测试用例或任何外发内容**；文档与用例一律用占位符（如 `<SHA512 摘要>`）。跳转 PC 页一律走一次性行权码握手（`ek` 兑换即剥码），token 不落 URL。
- **外发确认**：凡对外发送操作——上传文件/附件、云打包提交、发布发行、调用外部服务——必须逐次征得用户确认后执行，不得批量授权或静默重试。
- 文案全中文；样式遵循 DESIGN.md（色板/字号/间距用既有刻度，别自造）；
- 新增页面走 pages.json + `pages/<name>/<name>.uvue`，新增接口只加在 `utils/http.uts`；
- 不引入 npm 依赖、不用 scoped 样式（页面 style 即页面级）、不写 TS 语法（装饰器/枚举/泛型默认值等 UTS 不支持的）；
- 后端联调端口：APP dev 5173、PC dev 9527、后端 8600、nginx 7700；
- 文档其余部分：[project-overview.md](project-overview.md) / [architecture.md](architecture.md) / [DESIGN.md](DESIGN.md) / [development.md](development.md) / [component-api.md](component-api.md) / [user-guide.md](user-guide.md) / [TODO.md](TODO.md)。
