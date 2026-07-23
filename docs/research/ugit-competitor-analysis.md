# 竞品调研 — 腾讯 UGit

> 调研时间: 2026-07-22
> 目的: xgit 的直接竞品是腾讯自研的 UGit（闭源）。本调研用于借鉴产品形态、对照功能差距、定位 xgit 的差异化方向。

## 1. 资料来源

| 来源 | 类型 | 备注 |
| --- | --- | --- |
| `assets/ugit-about-dialog.png` | 关于对话框截图（用户提供） | 显示版本 5.510 X64 Community、所用的开源项目列表 |
| `assets/ugit-official-site.html` | 官网首页原始 HTML | 26 KB,2026-07-22 抓取 |
| https://ugit.qq.com/zh/ | 官网 | 标题: "UGit - 让每个人都可以轻松使用Git" |
| https://docs.qq.com/doc/DTGF6RmVLa0RZUEJm | 官方用户手册 | SPA,匿名请求只拿到 metadata,正文需登录 |

`docs.qq.com/doc/...` 是腾讯文档 SPA。匿名请求 opendoc API 只返回 `bodyData`(标题、创建者、设置等),不返回 `padHTML`。后续如果需要正文,走内部 SSO,或者请用户把内容截图 / 导出。

## 2. 基本信息

| 项 | 值 | 来源 |
| --- | --- | --- |
| 软件名 | UGit | 关于对话框 / 官网 |
| 开发商 | 腾讯 (Tencent) | 官网 footer |
| 协议 | 闭源,仅 Tencent 及合作伙伴可用 | 关于对话框 |
| 截图时的版本 | 5.510 X64 Community | 关于对话框 |
| 首次公开发版 | 2019-05-16 (1.0.0.257) | 官网版本日志 |
| 锁功能首发 | 2019-05-30 (1.0.0.297) | 官网版本日志 |
| 平台 | Windows (Intel / Apple Silicon)、macOS 10.15+ | 官网 |
| Slogan | "让每个人都可以轻松使用Git" | 官网 |
| 客户端定位 | "腾讯自研 Git 客户端" | 官网 |

## 3. 用到的开源项目 (关键技术栈信号)

UGit 关于对话框公开致谢的开源依赖:

- **GitHub Desktop** — Electron-based Git GUI,GitHub 官方维护
- **ElectronJS** — 跨平台桌面应用框架
- **Monaco Editor** — VS Code 同款代码编辑器 (Microsoft)
- **LuckySheet** — 国产开源 Web Excel 组件

技术栈推断:UGit 是 **Electron + TypeScript / JavaScript** 体系,UI 层使用与 VS Code 一致的代码编辑器组件,以及国产 Web Excel 方案。

## 4. 功能矩阵

### 4.1 大文件 / LFS 相关 (游戏工程最关心)

| 能力 | UGit | 备注 |
| --- | --- | --- |
| 内置 LFS 模板 | 有 | "腾讯众多大型项目 LFS 管理经验沉淀,尤其是游戏项目" |
| 大文件分析 | 有 | 帮助配置 Git LFS 规则 |
| 提交时大文件提示 | 有 | 按工蜂单文件大小限制提示 |
| LFS 缓存清理 | 有 | 单仓库 / 多仓库 |
| 本地 LFS Cache 加速 | 有 | "体验极致的下载速度" |
| 超大文件 (>4GB) 下载 | 有 | "无损下载" |
| 工蜂锁 (游戏项目二进制协作) | 有 | "解决了 Git LFS Lock 的稳定性和性能问题" |
| - 文件锁 | 有 | |
| - 目录锁 | 有 | |
| - 全分支锁 | 有 | |
| - 强制加锁工作流 | 有 | 必须先加锁才能提交 |
| - 推送后自动解锁 | 有 | |
| - 锁白名单 | 有 | 限定配置的目录只允许特定用户加解锁 |
| 检出子目录 (sparse checkout) | 有 | 大型仓库加速 |

### 4.2 提交流程

| 能力 | UGit | 备注 |
| --- | --- | --- |
| 快速提交 (绕过更新) | 有 | 提交的文件其他人没修改就不需要更新 |
| SVN 迁移 Git | 有 | |

### 4.3 平台集成 (深度绑定腾讯内部)

| 能力 | UGit | 备注 |
| --- | --- | --- |
| 工蜂 OAuth | 有 | |
| Github OAuth | 有 | |
| Coding.net OAuth | 有 | |
| 工蜂合并请求 (MR) | 有 | 客户端内评审 |
| 工蜂代码审查 | 有 | |
| 工蜂 Issue | 有 | |
| SSH 访问工蜂 (零配置) | 有 | |
| 蓝盾 (CIPD) 集成 | 有 | 截图显示 |
| TAPD 集成 | 有 | 截图显示 |

> 这块是 xgit 短期不会碰的——除非我们和某个具体代码托管平台达成合作。作为"未来可选集成"的清单记录。

### 4.4 版本控制可视化与协作

| 能力 | UGit | 备注 |
| --- | --- | --- |
| 极简操作 (类 SVN/P4) | 有 | "一键提交或更新" |
| Gitflow 可视化 | 有 | |
| 分支生命周期管理 | 有 | 批量清理无用分支 |
| 提交关联工蜂 Issue | 有 | |
| 多仓库管理 (Submodule 替代) | 有 | 批量克隆、更新、切分支 |
| 仓库分组 | 有 | 标签显示 |
| 变更集分组提交 | 有 | |
| 集成 CodeAction (远程代码审查) | 有 | 无需克隆 |
| 版本标记 (good / bad / star) | 有 | 便于版本回溯 |
| 分支规则管理 | 有 | 按规则一键锁定 |

### 4.5 性能与异构仓库

| 能力 | UGit | 备注 |
| --- | --- | --- |
| Git / SVN / P4 任两两同步 | 有 | Commit 维度单向/双向 |
| 仓库迁移 | 有 | |
| UE4 DDC 加速 | 有 | |
| Unity Cache 加速 | 有 | |
| 客户端钩子 (python / shell / batch) | 有 | 提交规范检查等 |
| 定时任务 | 有 | 锁分支、定时更新 (3 种策略) |
| Excel Diff & Merge | 有 | 单元格内容、公式;暂不支持样式 |

## 5. 对 xgit 的启示

### 5.1 必须做的能力 (差异化竞争基础)

1. **自己的文件锁机制**——UGit 解决了 Git LFS Lock 的稳定性和性能问题,这是游戏项目二进制协作的硬需求。xgit 不能再依赖 Git LFS Lock。
2. **快速提交**——大型团队痛点:不更新就能提交未冲突的文件。值得实现。
3. **大文件分析 + LFS 模板**——引导式配置,不是"用户必须自己写 `.gitattributes`"。
4. **检出子目录 (sparse checkout)**——几十 GB 的游戏仓库必备。
5. **本地 LFS Cache 加速**——远程 LFS 服务的中间层。
6. **超大文件下载**——单文件 >4GB 不出错。

### 5.2 短期不必做的

1. 工蜂 / 蓝盾 / TAPD 集成——腾讯内部平台绑定,xgit 面向通用游戏开发者。
2. SVN/P4 互操作——游戏团队 Git 化已是趋势,UGit 做这个吃的是迁移红利期,xgit 现在做优先级低。
3. Excel Diff & Merge——与游戏工程无关。

### 5.3 差异化方向 (xgit 能打而 UGit 不会打的)

| 方向 | 理由 |
| --- | --- |
| 引擎深度集成 (Unity `.meta` / UE `.uasset` / Godot `.import` 友好) | UGit 是"通用 Git 客户端"思路,没看到针对引擎元数据文件的特别处理 |
| 美术协作流 (PSD / Aseprite / Substance 预览) | UGit 没提 |
| 跨平台一致性 (macOS 优先体验) | UGit 主推 Windows |
| 真正开源 (Apache 2.0) | UGit 是闭源 + 仅内部可用 |
| 国际化 (多语言 UI) | UGit 只服务中文用户 |
| Git worktree / partial clone 等现代 Git 特性 | 官网未提及 |

### 5.4 技术栈参考

UGit 用 **Electron + Monaco Editor + LuckySheet**。xgit 在 AGENTS.md §3 已经列出 Electron / Tauri 候选。如果走 Electron 路线,电子表格能力可以用 **Luckysheet** 或其继任者 **Univer**;代码编辑器用 Monaco 几乎是必然。

## 6. 待补充

- UGit 在工蜂之外的部署情况 (用户量、活跃度、版本节奏)——公开渠道没数据
- UGit 是否原生支持 partial clone、commit graph、worktree 等新 Git 特性——官网未提及
- UGit 与 GitHub Desktop 的实际差异 (除了工蜂集成之外)—— 需要装一份对比
- 详细用户手册正文——见 §1 的说明
