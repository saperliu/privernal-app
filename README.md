# privernal-app —— 春祈平台移动端（uni-app x）

[English](./README.en.md) | 简体中文

privernal-app 是春祈平台（Privernal Studio）的移动端，基于 **uni-app x**（uvue + UTS）开发，一套代码覆盖 **Web / Android / iOS / 微信小程序**（三端运行为硬约束，差异与打包矩阵见仓库文档）。与 PC 端、后端共用同一服务：PC 创建并授权的应用、新建的动态表单**自动同步到 APP，免发版更新**。消息中心聚合系统消息与工作流待办（待办跳 PC 审批页办理，令牌走一次性行权码握手、不经 URL）；动态表单在 APP 内**原生渲染**卡片列表 / 详情 / 表单（schema 实时拉取，暂不支持的组件类型自动回退 PC 页 webview 承载）；多组织账号免选择流程，密码 SHA512 摘要传输。开发需 HBuilderX 5.x（uni-app x 不支持纯 CLI 创建）。

## 功能特性

- **登录**：设置服务器域名（按域名接入对应企业服务器，本地持久化）；密码 SHA512 摘要传输；多组织账号自动选择（APP 端免 PC 的选组织流程）
- **消息 TAB**：系统消息（列表/未读角标/详情/单条已读/全部已读）+ 工作流（待办/已办/我发起计数，待办跳转 PC 审批页办理）
- **工作台 TAB**：已授权应用汇总网格（按当前用户授权实时过滤，onShow + 下拉刷新），点击图标进入应用
- **应用详情**：按应用实时拉取菜单树，**菜单目录作分组标题、菜单功能作图标宫格**（4 列彩色图标 + 名称），点击图标进入列表页
- **动态表单原生渲染**：`/builder/{code}` 菜单在 APP 内**原生渲染**移动端卡片列表/详情/表单（schema 实时拉取，PC 端建表单免发版同步）；暂不支持的原生组件类型自动回退 PC 页面（webview）
- **动态表单/功能页 webview 轨**：大屏/报表/系统管理等其余页面由 web-view 承载 PC 端页面（令牌免登 + 全屏内容模式），PC 端全部能力开箱即用
- **我的 TAB**：组织/部门/姓名展示、修改密码、退出登录

## 平台支持与跨端约束

**硬约束：全部代码必须保证 Web（PC）、App、小程序三端正常运行；APP 必须保持可打包 iOS 与 Android。** 编码细则见 [AGENTS.md](AGENTS.md) 硬约束 #5，三端差异矩阵见 [architecture.md](architecture.md) §8，打包矩阵见 [development.md](development.md) §6。

| 端 | uni-app x 支持版本 | 当前状态 |
| --- | --- | --- |
| Web（PC 浏览器） | 4.0+ | ✅ 全流程 E2E 已验证（2026-10-23） |
| App Android | 3.9+ | 代码就绪；真机/云打包待验证（TODO P1） |
| App iOS | 4.11+ | 代码就绪；云打包链路待验证（需开发者账号，TODO P1） |
| 微信小程序 | 4.41+ | 待验证（web-view 需业务域名白名单） |
| 支付宝小程序 | 5.31+ | 按需扩展 |
| HarmonyOS | 4.61+ | 按需扩展 |

## 快速开始

环境要求：**HBuilderX 5.x**（uni-app x 不支持纯 npm CLI 创建，必须使用 HBuilderX）。

```bash
CLI=~/Applications/HBuilderX.app/Contents/MacOS/cli

$CLI open                                            # 启动 HBuilderX（如未运行）
$CLI project open --path <本目录绝对路径>              # 导入项目（首次）
$CLI launch web --project <本目录绝对路径>             # 运行到浏览器，dev server 默认 http://localhost:5173
```

登录示例（本地联调）：域名 `http://127.0.0.1:8600`，账号 `admin / admin123`；高级设置中 PC 端地址填 PC 前端 dev 地址（如 `http://localhost:9527`），生产环境（同域部署）留空。

> ⚠️ `admin / admin123` 只是初始化种子账号的首登口令，仅适用于本地联调。**首次登录后必须立即修改口令**；
> 公网部署前还需确认已改（口令策略与必须处理的安全项见后端仓库 `SECURITY.md`）。

## 文档导航

| 文档 | 内容 |
| --- | --- |
| [AGENTS.md](AGENTS.md) | **AI 进入本项目必读**：技术硬约束、UTS 深坑、接口契约、命令、红线 |
| [project-overview.md](project-overview.md) | 项目整体说明：背景定位、用户、范围边界、部署形态 |
| [architecture.md](architecture.md) | 架构与数据流：三端协作、模块划分、核心链路时序、接口清单 |
| [DESIGN.md](DESIGN.md) | 视觉规则：色板、字号、间距、组件样式规范、uni-app x 样式约束 |
| [user-guide.md](user-guide.md) | 使用者手册：各功能操作说明与常见问题 |
| [development.md](development.md) | 开发方式：环境搭建、日常命令、回归清单、打包发布、故障排查 |
| [component-api.md](component-api.md) | 组件相关：工具模块 API、可复用 UI 片段、内置组件用法、web-view 契约 |
| [TODO.md](TODO.md) | 当前任务、优先级与开发进度 |

## 目录结构

```
privernal-app/
├── pages/
│   ├── login/login.uvue                      登录（域名/账号/密码/高级设置）
│   ├── message/message.uvue                  消息 TAB（系统消息 | 工作流）
│   ├── workbench/workbench.uvue              工作台 TAB（应用网格）
│   ├── mine/mine.uvue                        我的 TAB（信息/改密入口/退出）
│   ├── app-detail/app-detail.uvue            应用功能列表（菜单树扁平化）
│   ├── webview/webview.uvue                  web-view 容器（PC 页面承载）
│   └── change-password/change-password.uvue  修改密码
├── utils/
│   ├── http.uts                              请求封装 + 平台接口路径（统一 401 处理）
│   ├── store.uts                             本地存储（域名/PC地址/token/用户信息）
│   └── sha512.uts                            纯 UTS SHA-512（登录/改密摘要）
├── static/                                   tabBar 图标（81px PNG）
├── pages.json                                路由 + tabBar（消息/工作台/我的）
├── manifest.json                             应用清单（uni-app-x）
└── App.uvue / main.uts                       应用入口
```

## 当前状态

web 平台已完成全流程 E2E 验证（登录 → 工作台应用 → 动态表单 → 消息 → 我的 → 退出），验证记录与截图见 [development.md](development.md) 回归清单与 `../gui-test-screenshots/app-e2e/`；Android/iOS 真机验证待进行（见 [TODO.md](TODO.md)）。
