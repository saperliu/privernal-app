# NOTICE — privernal-app（春祈平台移动端）

本文件说明本应用的构建基座、第三方关系与许可。应用源码以
[MIT License](./LICENSE)（Copyright (c) 2026 saperliu）发布。

## 1. 构建基座

- 本应用基于 **uni-app x**（DCloud）框架开发（uvue + UTS），通过 HBuilderX / DCloud CLI 编译，
  Android/iOS 运行时由 DCloud 引擎与云打包服务承载。
- DCloud 的运行时与云打包服务不在本仓库分发，其使用遵循 DCloud 的服务条款与许可；
  开发者自行安装 HBuilderX 并遵守其账号与计费约定。
- 应用不内嵌任何第三方 npm 运行时依赖：网络、存储、加密（SHA512）均使用
  uni-app x / UTS 内置 API 自行实现。

## 2. 与平台其它组件的关系

- 后端：`privernal-platform`（Apache-2.0），本应用通过 HTTP + 登录 token 与其对接，
  与本应用无代码级分发关系。
- PC 前端：`privernal-platform-ui`（MIT），本应用以 web-view 承载其部分页面，
  同样无代码级分发关系。
- 三者相互独立，任何一方不得标注为其它方的自研组件。

## 3. 第三方素材

- 应用图标与品牌素材（logo-ribbon.svg 及其栅格产物）为本项目自绘，随 MIT 许可分发。
- 除上述基座外，无其它随源码分发的第三方组件；如后续引入，须在本文件登记。
