# TODO.md —— 任务与进度

最后更新：2026-11-03。

## 当前状态

web 平台功能完整并全流程 E2E 通过；Android/iOS 真机未验证；一条后端修复已落码待重启。联调环境：APP dev 5173 / PC dev 9527 / 后端 8600（IDEA 进程）。

## 待办任务

### P0（阻塞项，先做）

| # | 事项 | 说明 | 验证方式 |
| --- | --- | --- | --- |
| 0 | 后端消息已读修复**重启验证** | `privernal-platform/.../SystemMessageServiceImpl.saveUserMessageRead` 的 ALL 分支已改"先查后更新"（原 insert 撞 `unique_user_message` 唯一键，报"数据编码已存在"）。代码已落，等 IDEA 重启生效 | 重启后 APP 消息 TAB 点开一条 ALL 群发未读消息 → 关闭弹窗 → 未读角标 -1；或 curl `PUT /app/system/systemMessage/user/read/{id}` 返回 code 200 且 readStatus 变 1 |

### P1（重要，本迭代内）

| # | 事项 | 说明 |
| --- | --- | --- |
| 2 | **Android 真机/模拟器验证 + 云打包** | `cli launch app-android` 全流程回归；Android release 包禁 http 明文，生产需 https 域名；云打包命令见 development.md §打包矩阵 |
| 3 | 应用图标 mdi 化 | 工作台应用图标目前是"首字+色块"；`system_app_info.icon` 存 mdi iconify 名（与 PC 一致），APP 端用字体图标或 SVG 渲染，配色沿用 DESIGN.md §1.4 |
| 4 | webview URL 组装健壮性 | `app-detail.uvue` 直接 `path + '?bare=1&...'`，菜单 path 自带 query 时会拼坏；改为判断已有 `?` 后用 `&` 合并（仅剩 webview 轨受影响） |
| 5 | ~~token 传递安全评估~~ **已解决（2026-11-03）** | 原风险：token 明文出现在 PC 页面 URL（浏览器历史/服务端日志可见面）。现改为**一次性行权码握手**：APP 侧 `utils/http.uts` 先 `POST /auth/web/handshake` 拿 60 秒单次码，URL 只带 `&ek=`；PC 侧 `router/guard/route.ts` 见 `ek` 才调 `POST /auth/web/exchange` 兑换并立即剥码重定向，token 不再进 URL。**注意旧版 APP 与新 PC 不可混搭**（旧版仍传 `?token=`，新接收端已不认） |
| 8 | **渲染器组件类型跟进机制** | 原生渲染器当前支持 6 类字段；设计器新增组件类型时需同步在 `renderer/schema.uts` 扩展（未跟进的类型当前自动回退 PC 页，不阻塞但有体验落差）。约定：新增组件必须同步登记移动端渲染方式 |
| 9 | 渲染器增强：列表高级操作 | 原生列表暂无批量删除/排序/列显隐；工作流表单"提交即发起"已随主路径，但请假审批记录/撤回等流程操作入口未做（详情页可后续加"发起/流程记录"） |
| 6 | **三端兼容验证：微信小程序端** | 三端约束的目标端之一。mp-weixin 编译+冒烟（登录/工作台/应用列表）；web-view 需 https+小程序后台"业务域名"白名单且个人主体不支持；request 合法域名生产需配置；本地 http 仅限开发者工具（manifest urlCheck=false 已设）。差异矩阵见 architecture.md §8 |
| 7 | **iOS 云打包链路验证** | "APP 可打包 iOS/Android"硬约束的 iOS 侧：Apple 开发者账号 + App ID（Bundle ID）+ p12 证书与描述文件（iOS 仅支持自有证书）；`cli pack --platform ios` 参数见 development.md §6；打出的 ipa 真机安装并跑回归清单 |

### P2（增强，排队）

| # | 事项 |
| --- | --- |
| 6 | 消息实时推送：后端 SocketIO（16001/webview /socket 通道）接入，新消息红点即时更新 |
| 7 | 消息未读总数服务端口径（当前角标=本地已加载列表未读数，翻页后不准） |
| 8 | 消息页下拉刷新（工作台已有，消息页未开 enablePullDownRefresh） |
| 9 | PC 列表页移动端适配优化（窄屏下表格横向滚动体验，与 PC 端联动） |
| 10 | HarmonyOS / 支付宝小程序端验证 |
| 11 | 启动图（splash）与 APP 桌面图标资产（当前为脚手架默认） |
| 12 | 多语言、深色模式（uni-app x darkmode） |
| 13 | 消息详情页独立成页（当前 uni.showModal 展示正文，长内容体验一般） |

## 已知问题（未修，知悉）

- 消息已读：依赖 P0 重启生效；生效前点已读不报错但状态不变（后端旧代码返回 500 被静默）。
- web 平台 web-view 是全屏 iframe（uni-app x 平台限制），App 端才可自由布局。
- IAB 自动化环境点击/截图管线停摆属测试环境伪象，不是应用 bug（绕过法见 development.md §自动化测试）。

## 进度日志约定

已完成的任务从本文件移除，仅保留未完成事项与已知问题；历史过程记录不随开源仓库分发。
