# BewlyCat · Safari 分支

> 基于 [BewlyCat](https://github.com/keleus/BewlyCat)（其本身基于 [BewlyBewly](https://github.com/BewlyBewly/BewlyBewly)）的 **Safari 自用构建分支**。
> 上游不维护 Safari；本分支的 Safari 适配、构建与问题由本仓库自行维护。

<p align="center" style="margin-bottom: 0px !important;">
<img width="300" alt="BewlyCat icon" src="./assets/icon-512.png"><br/>
</p>

<p align="center">只需对您的 Bilibili 主页进行一些小更改即可。</p>

## ⚠️ 使用声明（务必阅读）

> [!CAUTION]
> **本分支仅供作者个人自用（Safari 构建），严禁任何他人将本分支的代码或构建产物打包、发布到 App Store 或任何其他应用商店 / 分发渠道。**
>
> 若他人无视本声明自行打包发布，属其个人行为，与本分支维护者无关；由此产生的全部责任（包括但不限于账号、审核、版权、法律及账号封禁等）由行为人自行承担。

- 上游代码遵循 MIT 协议；此声明是维护者的使用请求与责任划分，不改变上游许可证。
- 本分支与上游项目无隶属关系，上游不保证 Safari 兼容性，也不会处理 Safari 相关反馈。

## 本分支是什么

仅面向 Safari 的最小适配分支；相关规则按构建目标裁剪，不影响其他浏览器的构建产物。当前包含：

1. **DNR 规则适配**：Safari 的 `modifyHeaders` 头名白名单不含 `sec-ch-ua*`，`regexFilter` 也不支持分组 `()` 与析取 `|`；Safari 构建改用拆分后的无分组规则。
2. **`cookies.onChanged` 防御**：Safari 会以空参数派发该事件，避免解构抛错。
3. **深色模式评论区**：Safari 会把文档级 `:host` 规则应用到 shadow host，浅色 token 落到 `bili-comments`；Safari 构建在深色模式下显式下发深色 token。
4. **构建期开关**：`vite.config.ts` 注入 `$bewly-safari-build`，上述样式只在 Safari 包中生效。

功能集与上游一致；功能说明与更新日志请见上游仓库。

## 构建

环境：macOS 27+ / Safari 27+、Node.js 24、pnpm。

```sh
pnpm install --frozen-lockfile
pnpm build-safari
```

产物目录：`extension-safari/`。

## 安装（Safari 27，无需 Xcode）

1. Safari → 设置 → 高级 → 勾选「显示网页开发者功能」；
2. Safari → 设置 → 开发者 → 勾选「允许未签名扩展」；
3. 同页点「添加临时扩展…」，选择 `extension-safari/` 文件夹；
4. 设置 → 扩展 → 启用 BewlyCat，网站访问设为「允许」；
5. 打开 <https://www.bilibili.com/>。

注意：

- 临时扩展会在**退出 Safari 后或 24 小时后**被移除，需要重新添加；
- 「允许未签名扩展」每次启动 Safari 后都要重新勾选；
- 未签名扩展无法持久安装，这是 Safari 的限制，与构建方式无关。

## 已知限制

- 上游明确不维护 Safari，上游新增功能可能与本分支不兼容，需要本仓库自行跟进。
- Safari 的 DNR 支持有限（无 `tabIds` 条件、头名白名单、正则子集），依赖请求头改写的功能可能降级。
- 启动日志中偶见 `[WBI] Failed to initialize WBI keys`，属于预热时序噪音；首次需要签名时会懒加载补钥，不影响页面功能。

## 上游与鸣谢

- 上游：[keleus/BewlyCat](https://github.com/keleus/BewlyCat)
- 基础：[BewlyBewly](https://github.com/BewlyBewly/BewlyBewly)
- 模板：[vitesse-webext](https://github.com/antfu/vitesse-webext)
- 其它参考：[Bilibili-Evolved](https://github.com/the1812/Bilibili-Evolved)、[bilibili-API-collect](https://github.com/SocialSisterYi/bilibili-API-collect)、[PiliPlus](https://github.com/bggRGjQaUbCoE/PiliPlus)（完整列表见上游 README）
- 贡献者：见上游仓库贡献者列表

## 许可证

本分支与上游代码遵循 MIT 协议，版权归原作者及贡献者所有，详见 [LICENSE](./LICENSE)。使用前请阅读上游项目相关声明。
