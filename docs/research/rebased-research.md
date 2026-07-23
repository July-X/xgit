# 调研 — DetachHead/rebased

> 调研时间: 2026-07-22
> 目的: 评估 [DetachHead/rebased](https://github.com/DetachHead/rebased) 是否适合作为 xgit 的代码基础。

## 1. 结论

**不建议以 rebased 为代码基础二开**。许可证不兼容、构建系统过重、本质是 IntelliJ 套壳而非自研 Git 客户端,与 xgit"自研 + 游戏工程友好"的定位不匹配。

可以借鉴 rebased 的**产品化思路**与**分发模式**,但不建议直接 fork。

## 2. 基本信息

| 字段 | 值 |
| --- | --- |
| 仓库 | [DetachHead/rebased](https://github.com/DetachHead/rebased) |
| 描述 | A git client based on the IntelliJ platform |
| Stars | 4,727 |
| Forks | 182 |
| 主语言 | Java (228 MB) + Kotlin (169 MB) + Python (40 MB) |
| 默认分支 | master |
| 仓库大小 | 5,283,492 KB (约 5.3 GB) |
| 首次提交 | 2026-01-09 |
| 最近 push | 2026-07-21 (活跃) |
| 许可证 | Other / NOASSERTION (实际继承自 IntelliJ Platform License) |

## 3. 实际定位 (重要)

rebased 的自述动机:

> Rebased is an open-source remake of the short-lived [jetbrains git client](https://youtrack.jetbrains.com/issue/IJPL-72504/Make-git-client-a-standalone-app#focus=Comments-27-12868395.0-0). It's basically just a JetBrains IDE with all the bundled plugins removed except the git integration, with some additional UI tweaks.

也就是说,**rebased 不是一个"自研的 Git 客户端",而是 JetBrains 官方那个"短期存在过的独立 Git 客户端"产品的社区重制版**。它做的事:

1. 拉取 [`JetBrains/intellij-community`](https://github.com/JetBrains/intellij-community)
2. 删掉除 git4idea 之外的所有插件
3. 加一些 UI 微调 (默认把 log 放主编辑器、可关闭 `.idea` 目录、多打包一些 TextMate bundles)
4. 用 Bazel 独立构建成桌面应用

代码上**主要贡献在构建脚本和品牌资源**,Git 操作核心仍来自 git4idea。

需求来源是 YouTrack 上的 [IJPL-72504](https://youtrack.jetbrains.com/issue/IJPL-72504/Make-git-client-a-standalone-app) —— 这是 YouTrack 上第 3 高票的公开 issue,多年来一直有人要求 JetBrains 官方把 Git 客户端独立出来。JetBrains 做过但很快关停,所以社区来重制。

致谢里提到 [`obiscr/intellij-community`](https://github.com/obiscr/intellij-community) —— 之前另一个类似的尝试,DetachHead cherry-pick 了部分 commit。

## 4. 分发模式

| 平台 | 分发方式 |
| --- | --- |
| Linux | AppImage,推荐 AppManager / Gear Lever 装到菜单 |
| Windows | `.exe` 安装包 / `winget install detachhead.rebased --source winget` |
| macOS | `brew install detachhead/tap/rebased` / `.dmg` |

## 5. 不适合做代码基础的理由

| 反对理由 | 严重度 | 说明 |
| --- | --- | --- |
| **许可证不兼容** | 高 | IntelliJ Platform License 限制性较强;xgit 是 Apache 2.0。继承 rebased 代码会引入 License 风险 |
| **构建系统过重** | 高 | Bazel + 5.3 GB monorepo,clone 一次要几十分钟,CI 跑一遍要几小时 |
| **本质是套壳** | 高 | Git 操作仍是 git4idea (Java 调用 Git CLI),没有"游戏工程友好"的可扩展点 |
| **缺乏差异化基础** | 中 | xgit 要做的是 Unity/UE/Godot 引擎元数据、文件锁、LFS 模板——rebased 不沾边 |
| **缺少分层架构** | 中 | 整个项目就是 IntelliJ Platform 本身,没有"我们自己的 core"层可以独立演化 |

## 6. 可以借鉴的点 (非代码层面)

1. **产品定位极其清晰** —— README 第一行就是 "A git client based on the IntelliJ platform."。一句话讲清楚它是什么、不做什么。
2. **需求来源是真实的痛点** —— YouTrack 第 3 高票 issue 驱动开发,不是凭空想象。xgit 调研 UGit 时也要找"游戏工程师在用的 Git 工具到底哪里不好用"的真实反馈。
3. **品牌资产齐备** —— rebased.svg、screenshot.png 在 community-resources 下,独立 IDE 套壳产品都需要这种品牌包装。xgit 后期也要补。
4. **多平台分发三件套** —— Linux AppImage / Windows winget / macOS brew tap,是开源桌面应用的标准动作。
5. **macOS Gatekeeper 应对** —— README 里专门一段教用户用 `xattr -rd com.apple.quarantine` 绕过未签名提示。xgit 没做 Apple Developer ID 签名时也会遇到,这段文案可以直接借鉴(注意是 gitea-kanban 也已经在用的话术)。
6. **AGENTS.md 已存在** —— rebased 仓库根目录有 `AGENTS.md` (5,232 bytes),说明这个项目也有 AI agent 协作规范。可以读一遍做参考。

## 7. 不建议借鉴的点

1. **5.3 GB monorepo** —— xgit 应该从一开始拆 monorepo,而不是继承 rebased 的大杂烩。
2. **AGENTS.md 的具体内容** —— 没有读过,不要盲抄,先读一遍再判断。
3. **JetBrains 整体技术栈** —— Java + Kotlin + Bazel 对游戏开发者桌面 GUI 来说门槛太高,xgit 的目标用户不全是 IntelliJ 用户。
4. **"我是 JetBrains IDE 的另一种皮肤"** —— 这种定位很难做出游戏工程的差异化。xgit 必须从第一天起就自研核心。

## 8. 待补充

- rebased 的 AGENTS.md 具体内容 —— 计划读一次
- rebased 与 obiscr/intellij-community 的关系 —— 历史 commit 链还需要细看
- rebased 在 Windows / Linux 下的实际稳定性 —— 没用过,只看 GitHub 评估
