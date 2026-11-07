# DESIGN.md —— 视觉规则

本文件定义 privernal-app 的全部视觉规范。写新页面/改样式必须使用这里的色值、字号、间距刻度，不要自造。整体风格：**浅色、低信息密度、圆角卡片、单一主色蓝**。

## 1. 色板

### 1.1 语义色

| 用途 | 色值 | 使用处 |
| --- | --- | --- |
| 主色 | `#2f6bff` | 登录/保存按钮、选中态文字、链接文字、头像底、菜单圆点默认色、tabBar 选中色 |
| 主色浅底 | `#eef4ff` | 应用图标默认底色、消息类型标签底、消息分段选中底 |
| 危险色 | `#f53f3f` | 退出登录文字、未读圆点/角标 |
| 页面背景 | `#f4f6fa` | 所有页面根背景（pages.json backgroundColor 同值） |
| 卡片背景 | `#ffffff` | 卡片/分组容器/tabBar/分段栏 |
| 输入框背景 | `#fafbfd` | input 底色 |

### 1.2 文本层级（由深到浅，按信息层级选用，禁止跳级）

| 层级 | 色值 | 用途 |
| --- | --- | --- |
| 标题 | `#1f2733` | 页面大标题、卡片主文字、菜单项、行值 |
| 正文 | `#4b5563` | 表单 label、tabBar 未选中文字、seg 未选中文字 |
| 次要 | `#6b7484` | 消息副标题 |
| 辅助 | `#7a8496` | 分组标题（section-title） |
| 弱化 | `#9098a8` | 提示副文案、计数 label、tabBar 图标色、品牌副标题 |
| 占位/提示 | `#a2aab8` | input placeholder（placeholder-style）、空态文字、加载文字 |
| 脚注 | `#b3bac6` | 页脚说明、服务器信息 |

### 1.3 边框与分隔

| 用途 | 色值 |
| --- | --- |
| 输入框描边 | `#e2e6ef` |
| 列表行分隔线 | `#f0f2f7`（0.5px，用 border-bottom-width/color/style **分写**） |

### 1.4 应用图标配色（工作台网格，按应用 code 映射）

| 应用 code | 前景色（图标字） | 底色 | 语义 |
| --- | --- | --- | --- |
| `campus` | `#ea580c` | `#fff7ed` | 校园管理（橙） |
| `SYSTEM` | `#0e7490` | `#ecfeff` | 系统管理（青） |
| `PLATFORM` | `#7c3aed` | `#f5f3ff` | 平台管理（紫，仅平台管理员可见） |
| 其他/新建应用 | `#2f6bff` | `#eef4ff` | 默认（蓝） |

映射函数在 `pages/workbench/workbench.uvue` 的 `colorOf()`；应用详情页菜单圆点色在 `pages/app-detail/app-detail.uvue` 的 onLoad 里按同一映射设 `accentColor`。**新增语义配色时两处同步改。**

## 2. 字号与字重

| 字号 | 用途 |
| --- | --- |
| 26px / 700 | 登录页品牌标题 |
| 24px / 700 | 应用图标首字 |
| 22px / 700 | 工作台问候语、计数数字 |
| 20px / 700 | 我的页昵称 |
| 18px / 700 | 应用图标字（app-icon-text 实际为 24，此处为列表箭头字号 18/常规） |
| 16px / 400 | 主按钮文字 |
| 15px / 400（卡片主文字 600） | 输入框、菜单项、行文字、消息标题 |
| 14px / 400（选中 600） | 表单 label、seg 文字、空态文字 |
| 13px / 400 | 分组标题、操作链接、加载文字 |
| 12px / 400 | 提示 tip、副标题、消息发送人 |
| 11px / 400 | 角标文字、消息类型标签、服务器信息脚注 |

## 3. 间距与圆角刻度

- **页面水平边距**：16px（分组卡片）/ 20px（登录卡片、头部文字）
- **卡片内边距**：14–20px；列表行垂直内边距 14–15px
- **分区间距**：分组卡片 `margin-top: 14px`；表单项间距 18px；label 与输入框 8px
- **圆角刻度**：输入框 8px → 主按钮 10px → 卡片 12px → 登录卡片 14px → 应用图标 14px → 圆点/角标（全圆）→ 头像 30px（60 直径）

## 4. 页面结构模板

```
page（flex:1 + 页面背景色）
├── 头部区（可选：大标题 + 副文案，padding 18-20px）
├── 滚动区 scroll-view scroll-y（flex:1）
│   ├── 分组 section：section-title（13px #7a8496）+ menu-card（白底圆角）
│   └── 底部留白 .bottom-space（height:30px）
└── 空态 .empty / 加载 .loading（居中，14px/13px #a2aab8，padding 80-100px）
```

- TAB 页根节点用 `view`；需要滚动的列表区一律 `scroll-view scroll-y` + `flex:1`。
- 横向排列必须显式 `flex-direction: row`（uni-app x 默认 column）。
- 所有可点击容器：点击目标用容器类（`.app`/`.menu-item`/`.row-click`/`.seg-item`），不要只把内层 text 设为可点。

## 5. 组件样式规范

| 组件 | 规范 |
| --- | --- |
| 主按钮 | **自绘 view+text**（禁用 `<button>` 组件，原因见 §7.9）：高 46px、圆角 10px、背景主色、白字 16px，flex 居中；加载态文字切换（如"保存中…"） |
| 次要按钮（退出登录/删除/取消） | **自绘 view+text**：白底圆角 10px，文字用语义色（危险 `#f53f3f`/正文 `#4b5563`），其余同主按钮 |
| 审批操作按钮（任务处理态） | 等分一行：高 46px、圆角 12px、间距 10px（两端 11px 对齐页边距）、文字 15px 加粗；**语义分级配色**——通过类主色实底白字 `#2f6bff/#fff`、驳回类浅红底红字 `#fdecec/#f53f3f`、其他操作浅灰底深字 `#f4f6fa/#4b5563`；**配置了显式 hex 色的按钮**实底白字；按钮集合按流程节点按钮配置（数据链见 architecture.md §3.5） |
| 输入框 | 高 44px、圆角 8px、描边 `#e2e6ef`、底 `#fafbfd`、字号 15px、水平 padding 12px；placeholder 色用 `placeholder-style="color:#c0c6d2"` |
| 密码输入 | `:password="true"` 属性（不用 type="password"） |
| 分组卡片 | 白底、圆角 12px、可含多行；行间用 0.5px 分隔线（末行去掉） |
| 信息行 row | 左 label 定宽 64px（#4b5563）+ 右值 flex:1 右对齐（#1f2733）；可点击行右侧 `›`（18px `#c0c6d2`） |
| 分段控制器 seg | 白底容器、选中项圆角 8px 底 `#eef4ff`、文字 14px 选中主色加粗 |
| 功能宫格 tile | 应用详情页：分组标题（13px `#7a8496`）+ 白底圆角 12px 容器内 4 列（tile 宽 25%）；图标块 48×48 圆角 12px（色板循环，见 §6）、首字 20px/700；名称 12px `#4b5563` 居中、超 5 字代码侧截断加"…" |
| 角标 badge | 高 18px、全圆角、底 `#f53f3f`、白字 11px、水平 padding 6px；未读圆点 8px 全圆 |
| 消息卡片 | 白底圆角 12px、padding 14px 16px、间距 `margin-top:10px`；标题 15px/600 + 未读点 |
| 空态/加载 | 居中文本，见页面结构模板；列表底用"没有更多了"（13px #a2aab8） |
| 反馈 | 轻提示 `uni.showToast({icon:'none'})`；确认/详情 `uni.showModal`（中文标题+正文，取消键默认 Cancel/OK 由系统渲染） |
| tabBar | 文字 `#9098a8`/选中 `#2f6bff`、白底、borderStyle black；图标 81px PNG（static/tab-*.png） |

## 6. 图标规范

- **tabBar 图标**：81×81 PNG，常态灰 `#9098a8`、选中主色 `#2f6bff`（现由 PIL 脚本生成，几何圆润风格；替换时保持尺寸与双色规范）。
- **应用/功能图标**：统一"首字 + 色块"方案，不依赖图片资源。工作台应用图标 56px 圆角 14px（配色见 §1.4）；应用详情功能宫格 48px 圆角 12px，色板六色循环：蓝 `#2f6bff/#eef4ff`、青 `#0e7490/#ecfeff`、橙 `#ea580c/#fff7ed`、紫 `#7c3aed/#f5f3ff`、绿 `#16a34a/#f0fdf4`、粉 `#db2777/#fdf2f8`（前景色/底色成对）。
- **菜单/功能项**：不用线性图标，用首字色块保持列表轻量。
- 规划中：应用/菜单图标接入 mdi iconify 语义图标（见 TODO.md P1），落地后沿用 §1.4 与上述色板映射。

## 7. uni-app x 样式约束（实测，写样式前必读）

1. **view 默认 display:flex、flex-direction: column**——横向布局必须显式 `flex-direction: row`；居中用 `align-items / justify-content`。
2. **文字必须放在 `<text>` 内**（App 端裸文本不渲染）；文字样式（字号/颜色/对齐）设在 text 上。
3. **选择器只用单一 class**（`.foo`），不用标签/后代/伪类选择器；样式写在页面 `<style>`（页面级隔离，不用 scoped）。
4. **border 用分写属性**：`border-bottom-width / border-bottom-color / border-bottom-style`（简写在 App 端表现不稳）。
5. **长度单位统一 px**；百分比宽度可用（如工作台网格 `width: 33.3%`）。
6. `text-align: center` 用于 text；块级居中靠父容器 `align-items: center`。
7. 安全区：内容页无需处理（默认导航条），全屏 web-view 由系统组件托管。
8. **跨端一致性**：样式只使用三端安全属性（Web/App/小程序表现一致），不用 web 专属能力（`:hover` 伪类、复杂选择器、滤镜、web 动画曲线等）；一切以 App 端最严口径为准，web/小程序自然兼容。
9. **禁用 `<button>` 组件**：uni-app x 的 button 带默认 `::after` 伪元素描边（1px 灰线，web 端实测确认），样式不支持伪元素选择器、App 端原生默认样式同样去不掉——**全部按钮一律 `view + text` 自绘**（view 管背景/圆角/高度/flex 居中，text 管字号/颜色），加载态用文字切换。
