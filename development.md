# development.md —— 开发方式、命令与回归清单

## 1. 环境准备

### 1.1 HBuilderX（必需）

uni-app x **不支持 npm CLI 创建项目**，必须 HBuilderX。本机已装 5.26.2026091802 于 `~/Applications/HBuilderX.app`。重装/升级方法（版本号从官方 release 接口拿）：

```bash
curl -s https://download1.dcloud.net.cn/hbuilderx/release.json   # 取 displayVersion / version
# 拼直链（M 系列 Mac 用 arm64 包）：
# https://download1.dcloud.net.cn/download/HBuilderX.<version>.arm64.dmg
hdiutil attach HBuilderX.*.dmg && cp -R /Volumes/HBuilderX/HBuilderX.app ~/Applications/ && hdiutil detach /Volumes/HBuilderX
```

CLI 入口：`CLI=~/Applications/HBuilderX.app/Contents/MacOS/cli`；首次运行 `$CLI open` 启动 HBuilderX 常驻。

### 1.2 联调环境依赖

| 服务 | 地址 | 说明 |
| --- | --- | --- |
| 后端 privernal-platform | 8600 | **用户 IDEA 调试进程启动，勿动**；探活 `curl http://127.0.0.1:8600/auth/login` |
| 本机 nginx | 7700 | PC 前端请求链路（`/test/<svc>/` 剥前缀转 8600），502 = 后端没起 |
| PC 前端 privernal-platform-ui | 9527 | vite dev（用户维护）；APP webview 需要它在线 |
| APP dev server | 5173 | 由 `launch web` 拉起 |

## 2. 日常命令

| 命令 | 用途 |
| --- | --- |
| `$CLI project open --path <abs>` | 导入项目（lint/运行前必须；换机器首次） |
| `$CLI launch web --project <abs>` | 运行到浏览器（后台常驻，dev 5173；首次自动下载编译插件） |
| `$CLI logcat web --project <abs>` | 查看 web 运行日志（HMR/报错） |
| `$CLI lsp lint --file <file> --project <abs>` | 单文件无头诊断；**不要传 `--platform`**（cli 坑：报 "Failed to read platform settings"）；项目未导入时同样报错 |
| `$CLI launch app-android --project <abs>` | 运行到 Android 标准基座（需设备/模拟器） |
| `$CLI pack --project <abs> --platform android --android.packagename com.privernal.app --android.androidpacktype 3 ...` | Android 云打包（3=云端证书） |

修改源码后 HMR 自动热更；**改 `utils/*.uts` 后建议整页 reload**（模块级状态与 HMR 的老问题同源）。

## 3. 代码规范

- 页面：`pages/<name>/<name>.uvue`，`<script setup lang="uts">`；样式 `<style>`（页面级，不用 scoped）；选择器只用单 class。
- 类型：响应取值一律 `UTSJSONObject` 的 `getString/getNumber/getJSON/getArray`；自定义 type 用 `{ } as` 之外的声明式标注（见 AGENTS.md 深坑表）。
- 接口：只加在 `utils/http.uts`（路径常量或 `api*()` 函数），页面不裸拼 URL（webview URL 组装除外）。
- 文案：全中文；样式色值/字号/间距按 DESIGN.md 刻度。
- 分页口径：pageNum 从 1 起；TableResult 解析用 `tableList()`/`tableTotal()`。
- **跨端（硬约束）**：所有改动必须保持 Web/App/小程序三端可编译运行——只用跨端 API、平台差异走 `#ifdef`、禁止浏览器专属对象（window/document/localStorage 等）；细则见 AGENTS.md 硬约束 #5、差异矩阵见 architecture.md §8。

## 4. E2E 回归清单（改版必跑）

环境：后端 8600 + PC 9527 + APP 5173 全部在线。账号 `admin/admin123`，域名 `http://127.0.0.1:8600`，PC 地址 `http://localhost:9527`。

| # | 步骤 | 预期 |
| --- | --- | --- |
| 1 | 清存储打开 5173 | 登录页：品牌区、域名/账号/密码输入框、高级设置、登录按钮 |
| 2 | 域名留空直接登录 | toast 提示"请填写正确的服务器域名" |
| 3 | 填域名+admin/admin123 登录 | 跳工作台；问候"你好，罗小黑"；应用网格含**平台管理/系统管理/校园管理**；底部 3 TAB |
| 4 | 工作台下拉刷新 | 应用列表重新加载 |
| 5 | 点「校园管理」 | 标题变校园管理；功能以**图标宫格**按"校园管理"分组展示（4 列彩色首字图标，9 项，长名截断） |
| 6 | 点「班级管理」 | webview 打开 PC builder 页；**免登录**；表格显示 2 条班级数据；侧栏/头部不可见（bare） |
| 7 | 返回两次 → 点「系统管理」 | 分组菜单树（用户/角色/日志/定时任务/动态表单/AI 搭建等） |
| 8 | 点「用户管理」 | PC 用户管理页：部门树 + 用户表格 |
| 8a | 未配置 PC 地址时点任一非 /builder 功能 | toast 提示"PC 页面默认取服务器同域：…打不开时请在登录页高级设置配置"；webview 打开后端地址会显示错误页（本地联调需先在登录页高级设置配置 PC 地址，见回归步骤 3 的前置条件） |
| 8b | 点「班级管理」 | **原生渲染页**：卡片列表（共 N 条 + ＋新增 + 5 列卡片字段，user-select 翻译为昵称非裸 id） |
| 8c | 原生列表：＋新增 → 填文本/选年级/班主任/状态 picker → 保存 | 回列表且新卡片在顶部（保存落库） |
| 8d | 原生列表：点新卡片 → 详情 7 字段 → 编辑改一列保存 → 详情复核 → 删除确认 | 详情/更新/删除全链路生效 |
| 8e | 点「学生管理」 | 原生卡片：所在班级显示班级名（relationList 翻译）、日期截断 yyyy-MM-dd、性别/状态选项映射；18 条 + 触底分页 |
| 8f | 点「#48 联动验收屏」（datav 菜单） | 仍走 webview 打开 PC 大屏页（两轨并存不受影响） |
| 9 | 切「消息」TAB | 默认系统消息分段；空态或消息列表；有未读时顶部角标 |
| 10 | 切「工作流」分段 | 三张计数卡（当前账号为 0 或实际值）+ 待办列表/空态 |
| 11 | 切回「系统消息」（有未读时点开一条） | 弹出正文详情；关闭后未读数 -1（依赖后端已读修复已重启） |
| 12 | 切「我的」TAB | 头像/罗小黑/账号 admin；组织=系统企业、部门、姓名；修改密码、退出登录、底部服务器地址 |
| 13 | 点「修改密码」→ 原密码 admin123，新密码不一致两个 | 提交提示"两次输入的新密码不一致"（不触后端） |
| 14 | 返回 → 点「退出登录」→ 确认 | 回登录页，**域名回填保留** |
| 15 | 重登 | 直接进工作台（域名记忆生效） |
| 16 | 已登录状态直接刷新 5173 | 免登录直达工作台 |
| 17 | （发版前）小程序端冒烟：HBuilderX 运行到微信开发者工具 | 项目编译通过；登录/工作台/应用列表功能正常；web-view 打开 PC 页需生产 https 域名+业务域名白名单（本地 http 无法验证该项，见 architecture.md §8）；App 端按 development.md §6 打包矩阵验证 Android/iOS |

证据截图目录：`../gui-test-screenshots/app-e2e/`（含工作台、动态表单两张基线）。

## 5. 自动化测试方法与环境坑

主力方式：web 平台 + 浏览器自动化（IAB/CDP 均可）。

- **IAB 输入/截图管线停摆（环境伪象，非应用 bug）**：页面收不到任何 click/touch 事件（探针为空）、cua/playwright click 全超时、截图报 "surface preparation timed out"；但 `fill`/`evaluate`/DOM 快照正常。**绕过：evaluate 里 `el.click()`**（应用 handler 正常执行）。
- 元素定位：uni-app x web 端 placeholder 不落 input 属性 → `getByRole('textbox')` 按序；文本点击向上找容器类（`.app`/`.menu-item`/`.seg-item`）；tabbar 点 `.uni-tabbar__item`；`uni.showModal` 按钮文本节点 `.uni-modal-dialog__action-text`；按钮是 `uni-button`。
- 跨域 iframe（web-view）：外层读不到内部 DOM；验证 PC 页状态用同源直达 URL 开新标签（例：`http://localhost:9527/builder/demo_class_info?bare=1&app=campus&token=...`，读 `#app.__vue_app__.$pinia` 的 app-store.fullContent / aside 宽度）。
- 截图要趁早（管线可能中途停摆），关键页面先截后操作。
- vite（PC 前端）长跑后可能漏更文件——验证 PC 守卫代码是否生效：`curl -s http://localhost:9527/src/router/guard/route.ts | grep appToken`。

## 6. 打包发布

### 6.1 打包矩阵（APP 必须保持可打包 iOS/Android）

| 平台 | 方式 | 前置条件 | 关键约束 |
| --- | --- | --- | --- |
| Web（PC） | HBuilderX 发行→网站；日常自测 `launch web` | — | 后端 CORS 已开放；部署域与后端同域时登录页 PC 地址留空 |
| Android | `$CLI pack --project <abs> --platform android --android.packagename com.privernal.app --android.androidpacktype 3` | DCloud 账号（云打包） | **release 禁 http 明文 → 生产必须 https**；证书类型 0 自有/1 公共/2 老版/3 云端 |
| iOS | `$CLI pack --project <abs> --platform ios --ios.bundle <AppID> --ios.certfile <p12> --ios.certpassword <pwd> --ios.profile <mobileprovision>` | Apple 开发者账号、App ID（Bundle ID）、p12 证书+描述文件——**iOS 仅支持自有证书** | ATS 默认要求 https；真机验证走 `launch app-ios --iosTarget simulator`（需 Mac+Xcode）；上架 App Store 经 App Store Connect |
| 微信小程序 | HBuilderX 面板 运行→小程序模拟器（或 `cli launch mp-weixin`）；发行=微信开发者工具上传 | 微信开发者工具；小程序 AppID 填入 manifest.json 的 `mp-weixin.appid`（脚手架已含该节点且 `urlCheck:false`） | **web-view 必须 https + 小程序后台"业务域名"白名单，个人主体小程序不支持 web-view**；request 合法域名生产需配置 |

支付宝小程序（5.31+）/HarmonyOS（4.61+）按需扩展，方式同理。注意：cli 的小程序命令帮助标注"仅适用于 uni-app 项目"，uni-app x 的 mp-weixin 支持自 HBuilderX 4.41 起——以实际编译为准，必要时用 HBuilderX 面板操作。

### 6.2 热更新（wgt）

- 壳资源变更（页面/样式/JS）可走 `$CLI publish app --type wgt` 热更；涉及 uni-app x 原生层/SDK 版本变更必须整包。

## 7. 故障排查速查

| 症状 | 原因 | 处理 |
| --- | --- | --- |
| 登录报"用户不存在/密码错误"（确认密码无误） | sha512 实现被改动 | Python 镜像对拍（见 AGENTS.md），回滚 rotr64 相关改动 |
| 登录报"没有权限"（登录成功后的接口） | 未调 getInfo 刷会话 | 检查登录链路第 4 步是否执行 |
| 接口 401 循环回登录页 | token 过期/后端重启清会话 | 正常行为；若刚登录即 401 查系统时间 |
| launch web 卡在"正在下载编译工具" | 首次下载编译插件 | 等待（视网络几分钟）；失败重跑 |
| lint 报 "Failed to read platform settings" | 项目未导入 or 传了 --platform | `$CLI project open`；去掉 --platform |
| webview 打开白屏 | PC 前端未起（9527）/PC 地址配置错 | 探活 9527；核对高级设置 |
| webview 页面要登录 | token 参数丢失 | 检查 app-detail URL 拼装与 PC 守卫注入代码在位 |
| 消息已读报"数据编码已存在" | 后端旧代码（修复未重启） | IDEA 重启后端（TODO #0） |
| HMR 后页面状态异常 | utils 模块热更不完整 | 整页 reload |
