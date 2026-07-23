# Git CLI、libgit2 与 go-git 性能调研

> 整理时间：2026-07-22
> 目标：为 xgit 的数百 GB 游戏仓库 Git 执行层选型提供证据
> 结论状态（2026-07-23 修订）：xgit 的 Git 执行层已经 ADR 锁定为原生 `git` + `git-lfs` CLI 独占；本调研保留 go-git / libgit2 对照只为 fork 与运维参考，不作为候选实现

## 1. 对比对象与口径

这里把“内存 Git”理解为运行在 xgit 进程内的 Git library：

- Git CLI：xgit 启动独立 `git` 子进程，通过 stdin / stdout / stderr 通信
- libgit2：C library，直接在 xgit 进程内调用；Go 接入通常需要 CGo
- go-git：纯 Go 的进程内实现，是 gitea-kanban 当前使用的方案，和 libgit2 不是同一个实现

“进程内”只表示省掉子进程启动和文本协议，不代表仓库驻留内存，也不代表算法天然更快。三套实现都会读 `.git`、pack、index 和工作区；真正决定数百 GB 仓库表现的是对象格式支持、索引利用、扫描范围、缓存状态与调用方式。

本调研只把满足下列条件的数据视为可横向比较：

1. 同一物理机、同一文件系统和同一仓库 commit
2. 操作语义一致，例如都返回完整 status，而不是一边只计数、一边构造全部对象
3. 明确版本、冷 / 热缓存、线程数和运行次数
4. 分开记录 wall-clock、CPU time、peak RSS 和输出体积
5. 网络操作使用同一服务端、协议、认证方式和网络条件

## 2. 公开数据能证明什么

### 2.1 唯一接近同机、同仓、同操作的 status 数据

libgit2 issue [#4230](https://github.com/libgit2/libgit2/issues/4230) 在 2017 年报告了 gentoo/gentoo 仓库上的 status 对比：

| 实现 | 配置 | wall-clock | 报告的 RAM |
| --- | --- | ---: | ---: |
| Git 2.10.2 | untracked cache 关闭 | 0.257 s | 32 MB |
| Git 2.10.2 | untracked cache 开启 | 0.097 s | 38 MB |
| libgit2 `df4dfaa`，经 git2go 调用 | issue 中的测试程序 | 0.626 s | 154 MB |

这组数据只能作为历史案例，不能代表当前版本：

- Git 2.10.2 和 libgit2 `df4dfaa` 都是 2016-2017 年的版本
- issue 没有把“RAM”的采样方法说清楚，不能确认是 peak RSS、某时刻 RSS 还是其他口径
- Git 的短格式文本输出与 libgit2 的结构化 status list 不一定完全等价
- 没有冷 / 热缓存分组、标准差和现代 SSD 环境数据

因此，本文不把 `0.626 / 0.257` 或 `0.626 / 0.097` 写成当前 libgit2 对 Git CLI 的性能倍数。

### 2.2 旧 index-pack 微基准

libgit2 PR [#1228](https://github.com/libgit2/libgit2/pull/1228) 在 2013 年使用 git.git 的 44 MB pack 报告：

- libgit2 修改前：55 s
- libgit2 加入 delta base cache 后：10 s
- 同期 Git `index-pack`：8.5 s

原作者将其称为“unscientific testing”。硬件、运行次数和 RSS 均未给出，而且当时 Git 的 indexer 是多线程，libgit2 测试路径是单线程。它可以说明该 PR 把 libgit2 的一个性能缺陷从 55 s 降到 10 s，不能说明今天两者的 clone / fetch 性能差距。

### 2.3 Git 自身优化的真实数据

这些数据来自 GitHub 官方文章，数字可追溯，但比较对象是 Git 优化前后，不是 Git CLI 对 libgit2：

| Git 场景 | 公开结果 | 来源 | 能说明什么 |
| --- | ---: | --- | --- |
| `git cat-file --batch-check='%(objectname) %(objecttype)' --batch-all-objects --unordered`，torvalds/linux 副本 | 8.1 s → 4.3 s | [Git 2.34 highlights](https://github.blog/open-source/git/highlights-from-git-2-34/) | Git 复用 pack offset 后，全对象元数据扫描变快 |
| 多个 topic branch 对同一 base 计算 ahead / behind | >500 ms → 28 ms | [Git 2.41 highlights](https://github.blog/open-source/git/highlights-from-git-2-41/) | 单次 graph walk 比循环执行多个 `rev-list` 更有效 |
| GitHub 全站部署 on-disk reverse index 后，服务 fetch 的日峰值 CPU time | 约 10.8 s → 7 s | [Scaling monorepo maintenance](https://github.blog/2021-04-29-scaling-monorepo-maintenance/) | Git 的 pack reverse index 在 GitHub 服务端生产负载上降低 CPU |
| GitHub 部署 multi-pack bitmap + geometric repack 后，单仓平均 repack | 约 1 min → 15 s | 同上 | MIDX、bitmap 与增量维护策略能显著降低服务端维护成本 |

最后两项是 GitHub 服务端生产指标，不是桌面客户端延迟，也没有固定硬件和仓库规模，不能直接套到 xgit。它们的价值在于证明：大仓库性能不只是“省掉一次进程启动”，对象索引和维护策略会改变整体数量级。

## 3. 当前没有可信公开数字的部分

截至本次调研，没有找到一套现代、公开、可复现的 benchmark，同时满足以下条件：

- 当前 Git 与当前 libgit2
- 同机、同仓、同 commit、同缓存状态
- 覆盖 `status`、log / revwalk、diff、clone、fetch、checkout
- 同时报告 p50 / p95、CPU time 和 peak RSS
- 提供可直接运行的代码及原始结果

libgit2 曾有 checkout / merge benchmark PR [#3055](https://github.com/libgit2/libgit2/pull/3055)，但未合并，也没有可引用的稳定结果表。公开的 libgit2 benchmark 页面没有提供足以支撑本次选型的当前数字。

因此，以下说法目前都没有证据支持：

- “libgit2 因为在进程内，所以一定比 Git CLI 快”
- “Git CLI 的进程启动固定是 10 ms / 20 ms / 50 ms”
- “libgit2 在大仓库中一定比 Git CLI 多占几倍内存”
- “Git CLI 在所有 Git 操作上都比 libgit2 快”

## 4. 能力差异及其性能含义

### 4.1 Git CLI 与 libgit2

| 能力 | Git CLI | libgit2 当前状态 | 对 xgit 的含义 |
| --- | --- | --- | --- |
| 进程启动 | 每次调用有启动和 pipe 成本，当前没有可外推的统一毫秒值 | 进程内调用，没有子进程启动 | 高频、极短、细粒度查询可能更适合 library，但需要实测批处理后的 CLI |
| commit-graph | 支持读取、写入和维护 | 核心读取、revwalk 使用和 writer API 已支持；changed-path Bloom filter、generation number v2 等扩展仍有缺口 | 不能再把 libgit2 简化成“不支持 commit-graph” |
| multi-pack-index | 支持，能结合 bitmap 和 maintenance | 已支持 ODB 读取和 writer API | 基础 MIDX 不是二者的分界线，bitmap 等扩展仍需单独核验 |
| reachability bitmap | 支持，并用于大对象图可达性计算 | 本次未验证到官方内建读写或利用证据 | 数百 GB、超多对象的 revwalk / fetch negotiation 必须通过 POC 测量 |
| partial clone | 支持 `--filter=blob:none`、`tree:0`、promisor remote 与缺失对象按需获取 | 完整工作流仍未实现；官方 [#5564](https://github.com/libgit2/libgit2/issues/5564)、[#6880](https://github.com/libgit2/libgit2/issues/6880) 仍记录缺口 | 这是 xgit 数百 GB 仓库主路径的硬约束，当前明显偏向 Git CLI |
| fsmonitor | built-in fsmonitor 及相关 index 优化 | 本次未验证到官方 status 集成证据 | 百万文件 status 不能只比较进程启动；避免全量扫描更重要 |
| Git LFS | 通过 `git-lfs` 提供完整客户端工作流 | 没有端到端 LFS 客户端；可用通用 filter API 自行集成，见 [#6565](https://github.com/libgit2/libgit2/issues/6565) | xgit 仍需 `git-lfs` 或自己实现协议、传输、缓存和凭证 |
| 故障隔离 | Git 子进程崩溃或内存暴涨不会直接带崩 GUI 主进程 | 与 GUI 同进程；崩溃、内存泄漏和 CGo 边界影响整个客户端 | 对长时间运行的桌面 GUI，可靠性也是性能成本 |
| 数据交换 | 需要解析稳定的 `--porcelain` / `-z` 输出；输出过大时可流式读取 | 结构化 API，避免文本解析和重复序列化 | libgit2 更适合高频细粒度对象访问；CLI 应使用批命令，不能每条记录启动一次进程 |

libgit2 的 commit-graph system API：<https://libgit2.org/docs/reference/main/sys/commit_graph/index.html>

libgit2 的 MIDX system API：<https://libgit2.org/docs/reference/main/sys/midx/index.html>

### 4.2 go-git 单列

go-git 不能拿 libgit2 的能力和性能替它背书。其官方 [COMPATIBILITY.md](https://github.com/go-git/go-git/blob/main/COMPATIBILITY.md) 当前明确列出：

- pack protocol v2：不支持
- protocol `filter` capability：不支持
- multi-pack-index：不支持
- Git LFS：不支持
- `gc`、`fsck`、`prune`：不支持
- merge 仅支持 fast-forward，部分高级 Git 工作流缺失

其官方 `worktree_status_bench_test.go` 提供 2,000 / 5,000 tracked files 以及 1k / 5k / 20k ignored files 的 benchmark 代码，但仓库没有发布可引用的 ns/op 结果。源码还明确把 clean status 称为当前实现的 worst-case：没有文件变化时仍会不必要地计算每个文件的 hash。

源码：<https://github.com/go-git/go-git/blob/main/worktree_status_bench_test.go>

所以，现阶段没有真实数据支持“go-git 的内存遍历比 Git CLI 更适合数百 GB 仓库”。相反，它缺少 xgit P0 所需的 protocol filter、LFS 和 MIDX，不能承担 clone / fetch / status 主路径。

## 5. 对 xgit 的结论

### 5.1 当前选择（已与 ADR-0001 §"技术路线"对齐）

xgit 的 Git 执行层 = **原生系统 `git` + `git-lfs` CLI**。这是 ADR-0001 的硬约束，本调研文档不再把 libgit2 / `go-git` 作为候选方案。

- 数百 GB 仓库的主执行层以 Git CLI + `git-lfs` 为唯一实现路径
- 在 fork gitea-kanban 时，移除 `app/git/`、`app/platform/*/` 中对 `go-git` 的依赖，替换为 `os/exec` 调原生 CLI
- 本调研继续保留 go-git / libgit2 的能力对照，是为了让 fork 与运维有参照依据，不是要把它们引入 xgit 进程

### 5.2 xgit 必须补的同口径 benchmark

每套实现都在相同仓库副本上运行，固定 Git / libgit2 commit 和编译参数：

| 场景 | Git CLI | libgit2 对应操作 | 指标 |
| --- | --- | --- | --- |
| 仓库打开 | `git rev-parse` / `git status --porcelain=v2 -uno` | init + repository open + lightweight status | p50 / p95、CPU、peak RSS、启动成本 |
| 工作区状态 | `git status --porcelain=v2 -z` | `git_status_list_new` | 冷 / 热缓存、0 / 1% 文件修改、百万文件 |
| 历史首屏 | `git log -n 500 --format=... -z` | revwalk 500 commits | 首字节、完成时间、peak RSS、输出条数 |
| 全历史遍历 | `git rev-list --all` | 完整 revwalk | commit-graph 开 / 关、bitmap 可用 / 不可用 |
| diff | `git diff --no-ext-diff --no-textconv` | tree / index / workdir diff | 文本、小二进制、LFS pointer，不读取大型 LFS 内容 |
| fetch | `git fetch --filter=blob:none` | libgit2 可实现的最接近路径 | 网络字节、pack bytes、CPU、peak RSS；不能实现等价 filter 时标记 N/A，禁止硬比 |
| checkout | sparse checkout cone | libgit2 checkout + 相同路径集合 | 文件数、写入字节、耗时、取消响应 |
| 连续交互 | 100 次 refs / HEAD / object 查询 | 批量 CLI 或持久 `cat-file --batch` 对进程内 API | 总时间、每次延迟、GUI 主进程 RSS |

测试至少覆盖 `core-requirements.md` 的 L / XL / XXL 三档。每项预热后运行不少于 30 次，保留原始 JSON / CSV、Trace2、版本、硬件、文件系统和仓库统计。涉及 page cache 的冷测不能只用主观“刚重启”；要记录实际清缓存方法，无法可靠清理时只报告热缓存。

## 6. 最终判断

公开证据目前支持以下判断：

1. libgit2 的进程内 API 确实省掉子进程和文本协议，但没有现代同口径数据证明这能在数百 GB 仓库上转化为总体优势。
2. Git CLI 对大仓库索引、partial clone、fsmonitor、bitmap、maintenance 和 LFS 的覆盖更完整。GitHub 的公开数据说明这些底层优化本身可能带来显著收益，但不能当成 Git CLI 对 libgit2 的直接倍数。
3. libgit2 已支持核心 commit-graph 和 MIDX，不能用过时的“不支持”说法否决它；真正的硬缺口是完整 partial clone、端到端 LFS，以及尚未验证的 bitmap / fsmonitor 路径。
4. xgit 先用 Git CLI 做可工作的性能基线，再用同一套 L / XL / XXL benchmark 判断 libgit2 是否值得进入局部热路径。这比引用零散旧数据更可靠。
