# ADR-0001 · xgit 核心功能要求与技术约束

> **状态**：已接受（2026-07-22；2026-07-23 多次修订：第 7 条 P0 / Gitea only / force push 风控 / Wails+Go / 原生 Git CLI / 路线图）
> **决策日期**：2026-07-22
> **决策者**：用户
> **取代**：原草案 `docs/research/core-requirements.md`（v0.1-v0.3）

## 背景

xgit 是一个面向游戏开发者的 Git 桌面 GUI 客户端，目标用户工程规模通常为 UE5 / Unity / Godot 量级（30-100+ GB）。立项时确立了七条硬约束作为后续所有技术选型和功能取舍的判据，但未经过正式 ADR 流程，导致约束力和引用机制不足。本次升格意在把"约束清单 + 决策记录 + 后续引用链"一并固化为后续所有 ADR 的顶层依赖。

## 上下文

- 项目目前处于白纸状态：仅 1 个 initial commit，未建立工程目录
- 桌面 GUI 技术栈在 AGENTS.md §3 中给出四个候选，但未拍板
- 本路线图：`docs/roadmap.md`（阶段 1-5 的进入 / 退出门槛；落 §12.5 阶段状态表 + §13.1 防跑偏检查清单，让 AI agent 在动任何代码 / ADR / release-doc / 调研文档前先读）
- LFS 锁方案给过五个候选，需结合后续 ADR 展开
- 已调研对象：UGit（闭源游戏专用）、gitea-kanban（开源 Wails+Go）、rebased（JetBrains 套壳）

## 决策

### 七条硬约束（按优先级）

| # | 约束 | 优先级 | 验证方式 |
| --- | --- | --- | --- |
| 1 | 系统资源消耗低 | **P0** | 空闲内存 < 100 MB、单进程、CPU 空闲 = 0 |
| 2 | 超大型游戏库体验 (UE5 引擎库量级) | **P0** | 能在 30-50 GB 仓库上流畅浏览 commit / 文件树 |
| 3 | LFS 友好 | **P0** | 初始化时引导配置 `.gitattributes`,内置引擎模板 |
| 4 | 超大文件优化 + 自研易用 LFS 锁 | **P0** | 单文件 >4GB 不出错,LFS 锁体验优于 Git LFS Lock |
| 5 | git 磁盘占用受控 | **P0** | 默认开启 gc / repack / prune,有可视化统计 |
| 6 | 自动 gc | P1 | 后台跑 `git maintenance`,空闲时调度 |
| 7 | 快速索引 + 方便搜索 | **P0** | commit graph + 文件名索引 + 全文搜索,首屏 < 2s |

P0 = 没做就不能发版；P1 = v1.0 必须有，v1.x 持续打磨。

**调整记录**：第 7 条原为 P1，v0.3 提升为 P0。原因是数百 GB 仓库的首屏加载和搜索体验直接决定工具是否可用，且 commit graph / 文件名索引 / 全文搜索是其他 P0 功能（如大仓库浏览、LFS 状态可视化）的前置依赖。

### 技术路线（已确认）

**以 gitea-kanban 的 Wails + Go 工程基建为基础**，但不直接沿用其现有 Git 数据路径。客户端需要为数百 GB 仓库重做 Git 执行层、任务调度、增量数据协议和性能验收。

- 系统 `git` CLI + `git-lfs` CLI 是 Git 执行层的**唯一实现路径**（2026-07-23 用户拍板；详见 §7 决策记录）。能力覆盖完整（partial clone、promisor object、fsmonitor、reachability bitmap、maintenance、LFS）；这里没有"Git CLI 比 libgit2 更快"的性能结论，是能力与平台范围驱动的硬约束
- 不引入 `go-git` / libgit2 / 自实现 pack parser；任何想用进程内 Git library 的方案必须另起 ADR 并改 §7 决策记录
- 数百 GB 仓库上的同口径 benchmark 方案见 [`docs/research/git-cli-libgit2-performance.md`](../research/git-cli-libgit2-performance.md) §5.2

### v1 平台范围：Gitea only（2026-07-23 用户拍板）

- **v1 阶段全面对接 Gitea 服务端**；功能稳定后再评估是否扩展到 GitHub / GitLab / Forgejo 等其他平台
- v1 不引入 `PlatformAdapter` 多实现抽象，避免在没有真实需求时引入可移植性成本
- fork 时保留 `app/platform/gitea/`，删除 `app/platform/github/`，未来新增平台必须另起 ADR
- 与 Gitea 无关的能力（Git 客户端执行层、`git sparse-checkout` + `--filter=blob:none`、LFS、`xgit-server` 锁、引擎模板）按既有 ADR 推进
- 范围重写详见 [`docs/research/gitea-only-scope.md`](../research/gitea-only-scope.md)

### 隐式 force push 工具层克制 + UI 风控（2026-07-23 用户拍板修订）

- 工具实现层契约：xgit 客户端**不**为"快速提交 / 路径级冲突检测"等"减少中断"型 UX 主动调用 `--force` / `--force-with-lease` / `+refspec`；用户**显式**选择 force push 时工具**能**调用，默认走 `--force-with-lease`
- UI 责任层契约：任何 force 入口必须先做二次确认 → 显示影响范围（被覆盖的远端 commit 数量、作者、最新 hash 时间）→ 强制勾选"我已知悉，可能覆盖远端历史与他人的提交" → 5s 冷却 → 写审计日志
- 默认行为：
  - "提交 + Push"、"快速提交并推送"按钮不调 --force
  - 任何调用 --force 的入口都在"高级操作"折叠区 / 右键菜单二级菜单；带显眼危险图标
  - push 失败不静默升格 force；UI 显式给出"以 force-with-lease 重试"二级入口，进入风控流程
- 与上面"Gitea only"和 P0 属于同级"工具层契约 + UI 责任"约束
- 详细论证见 [`docs/research/ugit-optimization-points.md`](../research/ugit-optimization-points.md) §2.3 / §3.4 "落点 0 / 落点 2"

### LFS 锁方案（已确认）

采用方案 C：中央锁服务。建设可独立部署到企业内网的 `xgit-server`，服务端作为跨用户锁状态的唯一权威来源。客户端只保留短时本地状态与待同步操作。

- 首版先保证 xgit 客户端与 `xgit-server` 之间的协议可靠
- 是否兼容 Git LFS Lock API，留作后续单独评估
- 部署形态必须支持单机内网服务，数据库与对象存储依赖保持可选或可替换

### 待拆分 ADR

本 ADR 仅确立顶层约束与基线路线，下列主题仍需单独 ADR 展开：

- `xgit-server` 锁协议、租约模型、高可用边界与 Git LFS Lock API 兼容性
- 客户端 Git 执行层：全部为原生 `git` + `git-lfs` CLI 调用（参见 ADR-0001 §"技术路线"），含任务调度、取消模型、输出解析与分批、UI 进度事件；包含"隐式 force push 工具层克制 + UI 风控"的硬约束。不再单独立"客户端 Git 执行层" ADR
- 引擎 `.gitattributes` 模板：Unity / UE / Godot 具体 glob 清单与版本维护策略
- 稀疏检出与 partial clone 的默认策略：cone mode、tree filter 与工作区/分离浏览
- 磁盘占用可视化：采样频率、UI 刷新节奏与自动 gc 触发策略
- 索引与搜索的具体实现：commit graph 生成时机、文件名索引存储、ripgrep 集成
- 平台扩展：v1 Gitea 链路稳定后，再单独 ADR 决定是否引入 GitHub / GitLab / Forgejo 等其他平台

## 理由

1. 七条硬约束覆盖了游戏工程 Git 工作流的全部关键场景，P0/P1 分级确保资源消耗、大仓库体验、LFS、磁盘、索引五个维度同时达标
2. gitea-kanban 的 Wails + Go 基建可直接复用，省 3-6 个月工程时间；其 21 MB 构建产物符合低资源消耗目标
3. 系统 Git CLI 目前在大仓库索引、partial clone、fsmonitor、bitmap、maintenance 和 LFS 上覆盖最完整，适合做可工作的首个基线
4. 中央锁服务方案解决了本地锁无法在网络分区时保证一致性的问题，符合游戏二进制资产不可合并的协作模型

## 后果

- 后续所有架构决策、技术选型、功能取舍都要回到本 ADR 对齐
- 任何影响本 ADR 七条硬约束的变更需走 ADR 修订流程
- 以 gitea-kanban 为基础意味着需要持续跟进上游或维护 fork，接受其工程约定（Vue 3、Pinia、Wails 配置）
- libgit2 的进程内优势（省掉子进程启动和文本协议）暂时不写入主架构；xgit 的 Git 执行层已锁定为原生 `git` CLI，不引入 libgit2
- 当前环境缺少 libgit2、Go/C 编译链和数百 GB 样本仓库，L/XL/XXL 同口径 benchmark 仍待启动；即使跑通也不影响"Git 执行层 = 原生 Git CLI"的 ADR 锁定

## 相关

- [`docs/research/git-cli-libgit2-performance.md`](../research/git-cli-libgit2-performance.md)：Git CLI、libgit2 与 go-git 的公开数据审计、能力差异和 benchmark 方案
- [`docs/research/ugit-competitor-analysis.md`](../research/ugit-competitor-analysis.md)：UGit 产品形态与"必须做能力"清单
- [`docs/research/ugit-usability-observations.md`](../research/ugit-usability-observations.md)：从 5 张截图整理的 UGit 易用性观察与借鉴清单
- [`docs/research/ugit-optimization-points.md`](../research/ugit-optimization-points.md)："快速提交"与"稀疏检出"两个优化点单独分析、对 Git 原生能力的反推、xgit 借鉴方案
- [`docs/research/gitea-only-scope.md`](../research/gitea-only-scope.md)：v1 平台范围 Gitea only 的范围重写、可行/不可行项、与 fork 路径的关系
- [`docs/research/rebased-research.md`](../research/rebased-research.md)：JetBrains 套壳路线的反例
- [`docs/research/gitea-kanban-fork-feasibility.md`](../research/gitea-kanban-fork-feasibility.md)：以 gitea-kanban 为基础的 fork 路径
- [`docs/roadmap.md`](../roadmap.md)：阶段 1-5 的进入 / 退出门槛与 release-doc 命名约定

## 决策记录

| 日期 | 变更 | 原因 |
| --- | --- | --- |
| 2026-07-22 | 第 7 条从 P1 提升为 P0 | 首屏加载与搜索体验是其他 P0 功能的前置依赖 |
| 2026-07-22 | 本文升格为 ADR-0001 | 原草案约束力不足，需建立正式引用链 |
| 2026-07-23 | 追加"v1 平台范围：Gitea only" | 用户拍板 v1 全面对接 Gitea，功能稳定后再扩展；避免引入 `PlatformAdapter` 多实现抽象 |
| 2026-07-23 | 追加"隐式 force push 工具层克制 + UI 风控" | 用户修订原"硬禁"约束为"工具可实现 + UI 必须做二次确认 / 5s 冷却 / 审计日志"；默认走 `--force-with-lease`；保留"xgit 不主动为减少中断型 UX 调用 --force" |
| 2026-07-23 | 锁定 Git 执行层 = 原生 `git` + `git-lfs` CLI 独占 | 用户明确不引入 `go-git` / libgit2 / 自实现 pack parser；fork gitea-kanban 时需替换 `app/git/` 中的 go-git 调用 |
