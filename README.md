# xgit

对游戏开发者友好的 Git GUI 客户端。

## 这是什么

游戏工程的 Git 仓库通常 30-100+ GB，里面是贴图、模型、音频、LFS 对象，再加上引擎自己的元数据。现有工具要么在大仓库上卡顿，要么资源占用离谱。

xgit 想做一款开发机常驻、轻量、对游戏工程师友好的 Git 客户端——git lfs 自动引导、引擎元数据跟着源文件一起走、稀疏检出 / partial clone 开箱即用、空闲时内存占用 < 100 MB。

## 灵感来源

产品层主要参考 [UGit](https://ugit.qq.com/zh/)；工程层从 [gitea-kanban](https://github.com/July-X/gitea-kanban) 借走 Wails + Go 的桌面端基建。架构、决策与约束均在 `docs/adr/` 与 `docs/research/` 里，README 不重复。

## 快速开始

项目当前处于"白纸 + 文档基线 + 路线图"阶段，代码主体按 [`docs/roadmap.md`](./docs/roadmap.md) 阶段 1 起逐步推进。

- **下载 release 包** —— 待 roadmap 阶段 5 完成后发布
- **自行 clone 并构建** —— 待 roadmap 阶段 1 跑通 `wails build` 后补
- **常见问题** —— 待 AGENTS.md / 各 release-doc 沉淀后摘录

## 参与贡献

- **Issue / PR** —— 仓库入口在 [github.com/July-X/xgit](https://github.com/July-X/xgit)，具体流程等 roadmap 阶段 1 把基线跑通后写在 `CONTRIBUTING.md`（占位文件即可）
- **设计决策讨论** —— 走 [`docs/adr/`](./docs/adr/) 的修订流程；新增约束或技术路线调整必须先起 ADR
- **调研 / 易用性问题 / 灵感** —— 写到 [`docs/research/`](./docs/research/)，由工程维护者审阅后并入决策或下版本规划

## 协议

[Apache License 2.0](./LICENSE)。
