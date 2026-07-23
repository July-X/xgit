# xgit 落地路线图

> 整理时间：2026-07-23
> 关联：[ADR-0001](./adr/0001-core-requirements.md)、[ADR-0002](./adr/0002-poc-benchmark-plan.md)、[AGENTS.md §3](../AGENTS.md)
> 蓝本：`~/2026/code/gitea-kanban`（VCS: github `July-X/gitea-kanban.git`；本地 `master` 最新提交 `6d8eea0 2026-07-22`）

## 1. 当前已经收敛到的硬约束（不要再讨论）

| 维度 | 锁定的决定 | 来源 |
|---|---|---|
| 桌面 GUI 技术栈 | **Wails v2 + Go** | [AGENTS.md §3](../AGENTS.md) / [ADR-0001 §"技术路线"](./adr/0001-core-requirements.md) |
| Git 执行层 | **原生系统 `git` + `git-lfs` CLI 独占**（go-git / libgit2 / 自实现 pack parser 都不引入） | AGENTS.md §3 / ADR-0001 |
| v1 平台范围 | **Gitea only**；不引入 `PlatformAdapter` 多实现抽象 | [docs/research/gitea-only-scope.md](./research/gitea-only-scope.md) |
| LFS 锁 | **方案 C**：可独立部署的 `xgit-server`（内网） | ADR-0001 §"LFS 锁方案" |
| Force push | **工具可实现 + UI 风控**：默认走 `--force-with-lease`；强制勾选 + 5s 冷却 + 审计日志；push 失败不静默升格 | [docs/research/ugit-optimization-points.md §2.3 / §3.4](./research/ugit-optimization-points.md) / ADR-0001 §"隐式 force push 工具层克制 + UI 风控" |
| 资源消耗 | 空闲 < 100 MB、单进程、CPU 空闲 = 0；浏览大仓库 < 500 MB | ADR-0001 §3.1 |

任何"是不是该改技术栈"的讨论都需要另起 ADR 修订；**本路线图不再讨论它们**。

## 2. 阶段总览（顺序与进入门槛）

> 阶段不是"在某周内完成"的里程碑，而是"必须先达到前一阶段的进入门槛，才能开启下一阶段"的依赖关系。

```
[阶段 0 准入]
        ↓  ✓ 文档基线建好（ADR-0001 / ADR-0002 / 本路线图）
[阶段 1 工程起点]
        ↓  ✓ 蓝本 fork + wails build 通过 + go-git 替换 CLI 子集跑通 dry-run
[阶段 2 Git 执行层 CLI 化]
        ↓  ✓ P0 第 1/2/3/5 条在 L 档样本上读起来门槛达标
[阶段 3 大仓库体验]
        ↓  ✓ XL / XXL 样本上 sparse-checkout + partial clone + commit graph 读出达标
[阶段 4 游戏工程特性]
        ↓  ✓ 三套引擎模板在真实工程上跑通；xgit-server 锁协议 L 档通过
[阶段 5 UI / UX 风控 + Gitea 端到端]
        ↓  ✓ force push 风控 + 路径 B + Gitea 完整 API 覆盖
[阶段 v1.0 候选]
```

## 3. 阶段 0（已完成）— 文档基线

确认项：

- [x] `docs/adr/0001-core-requirements.md`：七条硬约束 + 技术路线 + 平台范围 + force push 风控
- [x] `docs/adr/0002-poc-benchmark-plan.md`：L / XL / XXL 同口径 benchmark 方案
- [x] `docs/research/`：UGit、gitea-kanban fork、Git CLI / libgit2 / go-git 公开数据审计、Gitea-only 范围、易用性与优化点观察
- [x] AGENTS.md §3 / §5.3 / §7：与 ADR 同步
- [x] 本路线图

不在本阶段做的事：

- 任何代码或配置变更
- 任何 git commit / branch / tag

## 4. 阶段 1 — 工程起点：fork + 跑通 `wails build`

### 4.1 目标

把 gitea-kanban 的"Wails + Go + Vue 3"工程基建原样搬到 `/Users/zhongxingxing/2026/code/xgit`，并在 macOS 上 `wails build` / `wails dev` 跑通。**不引入新功能，不改产品形态**，目的是建立"可运行基线"。

### 4.2 进入阶段 1 的进入门槛

必须先确认：

1. `~/2026/code/gitea-kanban` 当前 `master` HEAD 在 2026-07-22 commit `6d8eea0`（更新时再核对）
2. `wails doctor` 在本机可跑；如不可跑，先解决 wails / Go / Node 工具链依赖（不在本阶段任务范围）
3. `wails build` 在 gitea-kanban 项目本身能产出 .app

### 4.3 fork / 搬移方式（多条路径，需用户选择）

| 路径 | 怎么做 | 优点 | 风险 |
|---|---|---|---|
| A. 软链 + 复制软链 | 把 `~/2026/code/gitea-kanban` 完整副本软链到 `~/2026/code/xgit`，保留 gitea-kanban 仓库元数据，再用 git remote 改写 | 起点和 gitea-kanban 完全同步 | 容易在 `cd` 操作里走错仓库；后续合流风险 |
| B. 全新 git init + tar 复制 | 在 `~/2026/code/xgit` 用 `git init` 起全新仓库，`rsync` 把 gitea-kanban 文件全量拷过来（排除 `.git/`） | 干净的工作树，git remote 仅指向 `https://github.com/July-X/xgit.git` | 需要"清理上游脚印"（CHANGELOG / CLAUDE.md 等私人痕迹），引入"启动 commit"工作 |
| C. 直接 `git remote add` + reset | 在 `~/2026/code/xgit`（已经是 git 仓库）里 `git remote add gitea-kanban ... && git fetch gitea-kanban`，再人工挑选 commit | 复用现 commit | 不直观，commit 历史混乱 |

**默认建议**：路径 B。新仓库是"白纸但已有 baseline 工作树"，先 `Initial commit from gitea-kanban base` 占位，再用 AGENTS.md / ADR / 本路线图覆盖或删去上游不需要的文件（CHANGELOG / CLAUDE.md / NOTICE / gitea-only 适配器）。

### 4.4 阶段 1 内的步骤

1. **fence 阶段**：先把 `docs/adr/`、`docs/research/`、AGENTS.md、本路线图打成一个基线 commit；这是后续一切 fork 工作的"我方文档"起点
2. **fork 阶段**：路径 B，rsync / cp 拉 gitea-kanban；改名 `app.go / app_*.go` 中的产品名 `gitea-kanban` → `xgit`；删 `app/platform/github/` 与 `app/platform/adapter.go`（Gitea only）
3. **保留 vs 删除清单**（路径 B 之后）：

   | 路径 | 处理 |
   |---|---|
   | `app/git/`（go-git 调用层） | **暂时保留**，在阶段 2 替换为 CLI；不要在本阶段提前改 |
   | `app/platform/gitea/` | **保留**，v1 主线 |
   | `app/platform/github/` | **删除** |
   | `app/platform/adapter.go` | **删除** |
   | `app/platform/httplog.go` / `platform.go` | 看是否仍被 `app/platform/gitea/` 使用，独立后仅留 `httplog.go` |
   | `app_gitclone.go` / `app_gitsync.go` / `app_gitbinary.go` | **暂时保留**；阶段 2 拆分为 CLI 调用 |
   | `CLAUDE.md` | 改名 `AGENTS.md` 后删除；AGENTS.md 本仓库已经存在 |
   | `CHANGELOG.md` `NOTICE` | 删除；后续按需重写 |
   | `frontend/` `src-go/` | 保留原样 |
   | `wails.json` `go.mod` `package.json` `pnpm-lock.yaml` | 保留原样；版本与改名 |
   | `tasks/` `tools/` `docs/`（在 gitea-kanban 内的） | 检查并挑选 |

4. **构建验证**：
   - `wails build` 在 macOS 上产 .app
   - `wails dev` 启动 30s 稳定存活
   - `cd frontend && pnpm typecheck` 无新增错误
   - `go build ./...` 通过
   - `go test ./app/...` 通过（gitea-kanban 仓库已有的测试套件是基线，**不应新加测试，也不应改测试结果**）

### 4.5 阶段 1 退出门槛

- 三平台（macOS / Windows / Linux）至少 macOS 能跑通 `wails build`
- 启动 GUI 能进入"空仓库"主页（v0.x 启动屏），不要求已实现 git 功能
- 现有 git 测试套件全部通过
- commit 历史只包含两类：
  1. "Initial commit from gitea-kanban base"
  2. 必要的删除 / 改名 / 文档覆盖 commit
- 阶段 1 的工作产出用一份 `M1-1-engine-start.md`（在 `docs/releases/`）做交付记录，对应 gitea-kanban 的"v0.1.x" 节奏

### 4.6 不在阶段 1 做的事（避免范围蔓延）

- 不实现任何新功能
- 不改 go-git 调用（阶段 2 才替换）
- 不接入 LFS 命令行（阶段 2）
- 不做 force push 风控（阶段 5）
- 不重写 `app/platform/gitea/`（后续阶段局部调整）
- 不建任何 platform adapter（v1 Gitea only）

## 5. 阶段 2 — Git 执行层 CLI 化

### 5.1 目标

把 gitea-kanban 当前的 `app/git/` 中所有 `go-git` 调用替换为 `os/exec` 调原生 `git` + `git-lfs` CLI，**等价外部行为**为第一目标，**性能优化**为第二目标。

### 5.2 进入门槛

阶段 1 退出门槛满足。

### 5.3 步骤

1. **替换边界**：
   - 入参 `git.PlainClone` / `git.Clone` 等改为 `exec.CommandContext(ctx, "git", ...)` + 流式解析
   - 取消模型用 `context.WithCancel`；子进程继承 `Cancel` 时必须等子进程退出
   - 调用前设定 `GIT_OPTIONAL_LOCKS=0` / `GIT_TERMINAL_PROMPT=0` / `GCM_INTERACTIVE=Never`，按 AGENTS.md §3
   - 长输出走 `--porcelain` / `-z` / `-0`，边读边解析，不 `io.ReadAll`
2. **LFS 子命令**：把 `git lfs track / pull / push / fetch / prune` 拆为独立包（暂命名 `app/gitlfs/`）；与 `app/git/` 区分
3. **保持 ABI**：Wails bindings 与 API 签名不变，前端代码不需要改
4. **测试对照表**：原 gitea-kanban `app/git/` 的测试至少保留接口，behaviour 取证；不要靠"测试现在还过"作性能或正确性结论
5. **不偷偷引入新行为**：保持现有 git 操作功能等价

### 5.4 退出门槛

- `go test ./app/...` 通过
- `wails dev` 现有 GUI 流程（clone / fetch / 同步等）行为与 gitea-kanban v0.7.21 表现一致（用 gitea-kanban 同一份 v0.7.21 路线做对照）
- L 档样本（参见 [ADR-0002 §2](./adr/0002-poc-benchmark-plan.md)）下：
  - 启动 ≤ 1s
  - 状态刷新（无改动） ≤ 1s
  - 历史首屏 ≤ 1s

### 5.5 不在阶段 2 做的事

- 不优化性能数字
- 不引入 sparse-checkout / partial clone 优化（阶段 3）
- 不实现引擎 `.gitattributes` 模板（阶段 4）
- 不做 `xgit-server` 锁协议（阶段 4）

## 6. 阶段 3 — 大仓库体验

### 6.1 目标

让 `wails dev` 在 XL / XXL（100 GB+ / 300 GB+）样本上能首屏可读、操作不假死，与 ADR-0001 §3.2/§3.5/§3.7 对齐。

### 6.2 进入门槛

阶段 2 退出 + L 档样本已准备。

### 6.3 步骤

1. **sparse-checkout + partial clone**：克隆向导嵌目录树多选；默认 `--filter=blob:none --sparse --no-checkout`；映射为 `git clone` + `git sparse-checkout init --cone` + `git sparse-checkout set <dir>`
2. **背景 fetch / 进度事件**：`git fetch` 走 sideband 解析；进度通过 `runtime.EventsEmit` 推前端；前端按 commit / pack bytes / ETA 分别显示
3. **磁盘占用可视化**：`du`/fs + `git count-objects -v` + LFS 缓存体积分别统计；UI 提示触发 `git maintenance run`
4. **commit graph + 文件名索引**：增量、按需、可关闭；按 ADR-0001 §3.7（已升 P0）

### 6.4 退出门槛

XL / XXL 样本上跑通 [ADR-0002 §6 门槛](./adr/0002-poc-benchmark-plan.md)：首屏 < 2-3s（XL-XXL）、status 热缓存 < 3-8s、peak RSS 符合 `idle < 100 MB / browsing < 500-800 MB`

### 6.5 不在阶段 3 做的事

- 不接 xgit-server（阶段 4）
- 不实现 force push 风控（阶段 5）
- 不做路径级冲突检测（阶段 5）

## 7. 阶段 4 — 游戏工程特性

### 7.1 目标

实现 ADR-0001 §3.3 / §3.4：LFS 模板、`.gitattributes` 引导、`xgit-server` 锁协议与客户端。

### 7.2 进入门槛

阶段 3 退出 + 至少一个 UE5 / Unity / Godot 真实工程（脱敏）样本。

### 7.3 步骤

1. **LFS 大文件检测**：clone 后扫描仓库，提示用户把 >X MB 的文件纳入 LFS
2. **三套引擎 `.gitattributes` 模板**：UE5 / Unity / Godot，按 [docs/research/ugit-competitor-analysis.md §3.3](./research/ugit-competitor-analysis.md) 列出的 glob 列表；用户确认后写入仓库根
3. **`xgit-server` 协议 v0.1**：单仓库锁状态、原子加锁、续租、释放、强解锁、审计；服务端权威
4. **锁 UI 集成**：在文件树 + LFS 文件视图里展示锁状态；客户端不允许把本地锁冒充全局锁；断线后只显示缓存
5. **与 Git LFS Lock API 兼容性评估**：[ADR-0001 §"LFS 锁方案"](./adr/0001-core-requirements.md) 留待拆分，本阶段出研究稿

### 7.4 退出门槛

- L 档真实游戏工程：LFS 检测 / `.gitattributes` 引导 / 文件锁 / 团队视图能跑通
- `xgit-server` 至少 5 个并发连接下稳定；服务端崩溃恢复不丢锁
- 不要求与 GitHub / GitLab 互操作

### 7.5 不在阶段 4 做的事

- 不接 GitHub / GitLab OAuth（ADR-0001 §"v1 平台范围"）
- 不引入 `PlatformAdapter` 抽象
- 不实现 GitHub-only 特性

## 8. 阶段 5 — UI / UX 与风控

### 8.1 目标

按 AGENTS.md §5.3 实现 force push 风控；按 [docs/research/ugit-optimization-points.md §2.3](./research/ugit-optimization-points.md) 实现路径级冲突检测 UI；按 Gitea API 推进 UI 流程补完。

### 8.2 进入门槛

阶段 4 退出 + Gitea 实例已确定（自部署或腾讯工蜂/腾讯云 Gitea）。

### 8.3 步骤

1. **force push 风控 UI**：二级确认 / 影响范围展示 / 5s 冷却 / 审计日志写盘 / push 后回滚路径提示
2. **路径级冲突检测 UI**："提交 + Push" 与"快速提交并推送"双按钮；fetch 后无路径冲突走 fast-forward push；有冲突走标准 merge / rebase
3. **多仓管理 UI**：参考 [docs/research/ugit-usability-observations.md §2.1](./research/ugit-usability-observations.md) 与 UGit 截图；URL 与 OAuth 主页放在设置 → 账户
4. **空提交拦截 / 提交信息模板 / LFS 大小提示**：与 gitea-kanban `app_prefs.go` 对接
5. **Gitea API 端到端**：OAuth 登录、PAT、项目、Issue / PR / 分支保护规则读取、LFS 仓库侧状态

### 8.4 退出门槛

- 全部 P0 在 L / XL / XXL 三档样本上跑通
- force push 三件 UI 模板（提示 / 校验 / 5s 冷却）独立可测
- Gitea 实例（自部署）上 OAuth / clone / fetch / push / PR 流程全跑通

### 8.5 不在阶段 5 做的事

- 接入 GitHub / GitLab（ADR-0001 §"v1 平台范围"：v1 Gitea only，新平台走独立 ADR）
- 引入 platform adapter
- 重写大仓库数据通道（与阶段 3 共测）

## 9. 阶段 v1.0 候选

只有当前面所有阶段退出门槛都达到、且 force push / LFS / Gitea 端到端至少 5 个真实用户跑过 1 个月以上无 P0 回归，才能进 v1.0 候选。本路线图不擅自起"v1.1 / v1.2"子版本。

## 10. 每个阶段的"通过 / 不通过"记录位置

- `docs/releases/M1-1-engine-start.md` —— 阶段 1
- `docs/releases/M2-1-git-cli.md` —— 阶段 2
- `docs/releases/M3-1-large-repo.md` —— 阶段 3
- `docs/releases/M4-1-game-engines.md` —— 阶段 4
- `docs/releases/M5-1-ui-wind-controls.md` —— 阶段 5

每个 release-doc 必须引用本路线图对应章节 + 当阶段的进入 / 退出门槛清单 + 实际跑过的命令原始输出。

## 11. 仍未启动的事项（按本路线图的依赖）

- L / XL / XXL 三档样本仓库的实际生成 / 采购（gitea-kanban 当前没有 dev-only fixture）
- libgit2 / go-git / Git CLI 现代同口径 benchmark（来自 [docs/research/git-cli-libgit2-performance.md §3](./research/git-cli-libgit2-performance.md)）；本路线图已经**不再依赖**这个 benchmark 做选型，但 POC 的"路径 B 是否达 L 档"仍需真实数据
- Windows WebView2 / WebKitGTK 在大仓库交互上的具体瓶颈（与 [docs/research/core-requirements.md §5.2](./research/core-requirements.md) 待补实测对应）

## 12. 任意阶段的"风险与回滚"

| 风险 | 缓解 |
|---|---|
| fork 后与上游 gitea-kanban 失去同步 | 不强求同步；路线图把 fork 当"起步模板"，不假设有同步窗口 |
| 阶段 2 替换 go-git 后 ABI 漂移 | 阶段 2 结束门槛里有 ABI 等价条款；任何 ABI 漂移记入 `M2-1-git-cli.md` 偏差清单 |
| L 档 / XL 档 / XXL 档样本未就绪 | 阶段 2 推迟开始；不要用合成数据算性能数字 |
| 用户拍板推翻本路线图的某条 | 走 ADR 修订；当前阶段不重做 |

## 13. 相关引用

- [ADR-0001](./adr/0001-core-requirements.md) — 顶层约束与技术路线
- [ADR-0002](./adr/0002-poc-benchmark-plan.md) — L / XL / XXL benchmark 方案
- [AGENTS.md](../AGENTS.md) — agent 工程契约
- [docs/research/gitea-only-scope.md](./research/gitea-only-scope.md) — v1 Gitea only 范围
- [docs/research/git-cli-libgit2-performance.md](./research/git-cli-libgit2-performance.md) — Git 执行层公开数据审计
- [docs/research/ugit-optimization-points.md](./research/ugit-optimization-points.md) — 路径 B 与 sparse-checkout 借鉴
- [docs/research/ugit-usability-observations.md](./research/ugit-usability-observations.md) — UI 借鉴清单
- [docs/research/gitea-kanban-fork-feasibility.md](./research/gitea-kanban-fork-feasibility.md) — 以 gitea-kanban 为基础的 fork 路径
