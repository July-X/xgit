# 第一阶段：Gitea-only 范围调研

> 整理时间：2026-07-23
> 约束来源：用户 2026-07-23 拍板 —— **"第一阶段全面对接 Gitea 服务端，功能稳定后才会对接 GitHub / GitLab 等其他服务"**
> 关联：[ADR-0001](../adr/0001-core-requirements.md) 七条硬约束、[ADR-0002](../adr/0002-poc-benchmark-plan.md) POC 方案、[ugit-optimization-points.md](./ugit-optimization-points.md)、[gitea-kanban-fork-feasibility.md](./gitea-kanban-fork-feasibility.md)
> 状态：本轮结论已挂上 ADR-0001 顶层；后续 v1 任何"对接 GitHub/GitLab" 必须先走 ADR 修订流程

## 1. 一句话定位（v1 阶段）

> xgit 是一款面向游戏开发者的低资源 Git 桌面 GUI，**第一阶段只与 Gitea 服务端 API 配套**，等 Gitea 链路端到端稳定后，再走 ADR 决定是否扩展 GitHub / GitLab 等其他平台。

## 2. 在范围内 vs 不在范围内

### 2.1 v1 必做（约束：Gitea 优先）

- Gitea API：clone / fetch / push / pull / PR（合并请求）/ issue / 用户与组织 / Webhook 接收
- Gitea OAuth / PAT 认证，本地 keychain 存 token（与 ADR-0001 §3.5 锁配合）
- Gitea 仓库的 LFS 仓库侧覆盖；xgit-server 锁仅承担二进制协作
- 引擎 `.gitattributes` 模板（Unity / UE / Godot）—— 与服务端无关
- 客户端 Git 执行层（CLI / libgit2 / go-git 分工）：与 Gitea 无关
- 稀疏检出 + partial clone：Git 客户端原生能力，与服务端无关

### 2.2 v1 明确不做

- **GitHub API**：任何 GitHub-only 特性（PR Review 行内评论、Checks、GitHub Actions 集成等）—— 等 Gitea 稳定后再评估
- **GitLab / Forgejo / Bitbucket 等**：同上
- **同时支持多种平台**的 platform adapter 抽象层
- **只**为兼容多平台而引入的可移植性抽象（如多 token 存储、多 API 模型）
- UGit 截图中提到的工蜂 / TAPD / Jira Cloud / CODING 等内网集成 —— 不在范围
- **非快进推送**（UGit "快速提交"路径 A）：需要服务端 `receive.denyNonFastForwards=false`，跨平台不可控，**不做**

### 2.3 与"工蜂锁"无关

`xgit-server` 锁服务是**独立部署**的服务端，与 Gitea 不在同一进程、不在同一栈。客户端锁协议不依赖 Gitea API。该条不受本约束影响。

## 3. 对前三轮调研结论的重写

### 3.1 快速提交 — 仅路径 B 可行，已收敛

[ugit-optimization-points.md §2.2](./ugit-optimization-points.md) 已分析两条路径：

| 路径 | 是否在范围内 | 原因 |
|---|---|---|
| A：服务端允许 non-FF push | **不做** | 需要 Gitea 实例层 `receive.denyNonFastForwards=false`，违背 v1 "对接通用 Gitea 实例"的目标 |
| B：客户端路径级冲突检测（用原生 `git` CLI 实现） | **候选** | 纯客户端能力，对任意 Gitea 实例均有效；走 L 档 POC 后再决定 P1 与否 |

UGit 文档说的"只要用户提交的文件其他人没修改"对应路径 B，与 Gitea 特权无关。

### 3.2 隐式 force push 工具层克制 + UI 风控（2026-07-23 用户拍板修订）

- xgit 客户端**不**为"快速提交 / 路径级冲突检测"等"减少中断"型 UX 主动调用 `--force`
- 当用户**显式**选择 force push 时，工具**能**调用 `git push --force` / `--force-with-lease` / `+refspec`；默认走 `--force-with-lease`
- 任何 force 入口必须在 UI 上：
  - 显示将被覆盖的远端 commit 数量、作者列表、最新 commit hash 时间
  - 用户必须勾选二次确认框 + 执行按钮强制 5s 冷却
  - 写入本地审计日志
- 禁止静默重试 `--force`：push 失败时不读 raw response 自动升格 force；UI 提供"以 force-with-lease 重试"二级入口，进入风控流程
- 详细论证与风控 UI 模板见 [ugit-optimization-points.md §2.3 / §3.4 落点 0、落点 2](./ugit-optimization-points.md)

理由：硬禁会让"rebased 远端 / 已 reflog 切回 commit"等可恢复场景断流。工具应具备能力 + UI 把风险透明 + 用户作选择，留给用户知情决策的余地。

### 3.2 稀疏检出 — Git 客户端原生能力，与 Gitea 无关

[ugit-optimization-points.md §3](./ugit-optimization-points.md) 已分析：

- `git sparse-checkout` 起自 Git 2.25，cone 模式为默认自 Git 2.26
- 与服务端无关，任何 Gitea 实例都支持
- v1 阶段**可以落地**：克隆向导嵌目录树多选 + 默认建议 `--filter=blob:none --sparse`
- 过程不触发任何 `git push --force`，遵守 §3.2 的硬红线

不需要额外 ADR 即可推进（属于 ADR-0001 §3.2"超大型游戏库体验 P0"已经覆盖的范围）。

### 3.3 gitea-kanban fork — 路径要精简

[gitea-kanban-fork-feasibility.md](./gitea-kanban-fork-feasibility.md) 既给出 fork 路径又写了"删 Gitea/GitHub 适配器"。在 Gitea-only 约束下：

- **保留** `app/platform/gitea/` 整个目录
- **删除** `app/platform/github/` 与未来 `app/platform/gitlab/` 之类
- **不做** `PlatformAdapter` 多实现接口；如果未来要做，**走 ADR**
- v1 阶段不需要在 fork 时引入"可扩展多平台"的抽象成本

这把工程量进一步压缩到：

- 客户端 Git 能力（替换 go-git 写操作路径到 git CLI）
- Gitea 完整对接
- LFS + `xgit-server` 锁
- 引擎模板 + 稀疏检出 UI + 大仓库性能验收

### 3.4 系统资源消耗 — 不变

ADR-0001 §3.1 已经把"空闲 < 100 MB、浏览大仓库 < 500 MB"列为 P0，与是否对接 GitHub / GitLab 无关。

## 4. 与 ADR 流程的关系

本轮拍板的"第一阶段 Gitea-only"，需要进入顶层 ADR 才能让所有后续 PR 引用。本轮已：

- 在 [ADR-0001](../adr/0001-core-requirements.md) §"技术路线"部分追加"v1 平台范围：Gitea only"
- 在 [ADR-0001](../adr/0001-core-requirements.md) §"待拆分 ADR"清单中，明确"平台扩展（GitHub / GitLab / Forgejo 等）"作为独立 ADR 项，需 v1 Gitea 链路稳定后另起一份 ADR 拍板
- 其他调研文档保留向后兼容的措辞（"未来扩展"用 ADR 决定），避免在调研阶段先扩范围

## 5. 与性能调研的关系

[git-cli-libgit2-performance.md](./git-cli-libgit2-performance.md) 引用了 GitHub Blog 的两篇生产案例：

- Git 2.34 highlights（cat-file 8.1s→4.3s）
- Scaling monorepo maintenance（reverse index 10.8s→7s、repack 1min→15s）

这些是 Git 上游能力证明，与平台无关。即使 v1 不接 GitHub，这些数字仍可用作背景；引用处保留，只是不再隐含"我们要对接 GitHub"。

## 6. 暂不展开的话题

下列内容与"Gitea-only 范围"相关，但本轮不展开，需要时另起文档：

- Gitea API 速率限制与重试策略
- Gitea Webhook 与本地通知的整合
- Gitea 大文件 / LFS 服务端的兼容矩阵（Gitea 原生 LFS、MinIO、S3 兼容后端）
- `xgit-server` 与 Gitea 的部署共存形态

均进 ADR-0001 "待拆分 ADR" 清单的对应项，不在本轮开始。

## 7. 数据真实性边界

下列信息本文**不主张**：

- Gitea 实例层的默认策略：本文仅以"通用 Gitea 实例 + 不可改服务端"作为假设前提
- 任何性能提升倍数：路径 B 的"路径级冲突检测"提速比例需要 L 档 POC 实测
- UGit 实现细节：本节不复述，仅引用 [ugit-optimization-points.md](./ugit-optimization-points.md) 已有结论

## 8. 相关引用

- [ADR-0001](../adr/0001-core-requirements.md)：顶层约束
- [ADR-0002](../adr/0002-poc-benchmark-plan.md)：POC 基准方案
- [ugit-optimization-points.md](./ugit-optimization-points.md)：快速提交 / 稀疏检出的实现路径分析
- [ugit-competitor-analysis.md](./ugit-competitor-analysis.md)：UGit 功能矩阵与技术栈推断
- [gitea-kanban-fork-feasibility.md](./gitea-kanban-fork-feasibility.md)：fork 路径评估
- [git-cli-libgit2-performance.md](./git-cli-libgit2-performance.md)：Git 执行层公开数据
