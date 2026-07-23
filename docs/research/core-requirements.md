# xgit 核心功能要求与技术约束

> 整理时间：2026-07-22
> 来源：用户对 xgit 立项的根本要求
> 用途：所有架构决策、技术选型、功能取舍都要回到这里对齐
> 状态：已升格为 ADR-0001（2026-07-22）。本文件保留为历史调研草稿，不再作为决策权威来源。

## 1. 一句话定位

为大型游戏仓库（UE5 / Unity / Godot 量级）设计的低资源消耗 Git 桌面 GUI，LFS 友好，自带易用的文件锁机制。

## 2. 七条硬约束

| # | 约束 | 优先级 | 验证方式 |
| --- | --- | --- | --- |
| 1 | 系统资源消耗低 | P0 | 空闲内存 < 100 MB、单进程、CPU 空闲 = 0 |
| 2 | 超大型游戏库体验 (UE5 引擎库量级) | P0 | 能在 30-50 GB 仓库上流畅浏览 commit / 文件树 |
| 3 | LFS 友好 | P0 | 初始化时引导配置 `.gitattributes`,内置引擎模板 |
| 4 | 超大文件优化 + 自研易用 LFS 锁 | P0 | 单文件 >4GB 不出错,LFS 锁体验优于 Git LFS Lock |
| 5 | git 磁盘占用受控 | P0 | 默认开启 gc / repack / prune,有可视化统计 |
| 6 | 自动 gc | P1 | 后台跑 `git maintenance`,空闲时调度 |
| 7 | 快速索引 + 方便搜索 | **P0** | commit graph + 文件名索引 + 全文搜索,首屏 < 2s |

P0 = 没做就不能发版;P1 = v1.0 必须有,v1.x 持续打磨。

**调整说明:** 第 7 条(快速索引 + 方便搜索)原为 P1,现已提升为 P0。原因是数百 GB 仓库的首屏加载和搜索体验直接决定工具是否可用,且 commit graph / 文件名索引 / 全文搜索是其他 P0 功能(如大仓库浏览、LFS 状态可视化)的前置依赖。

## 3. 逐条技术展开

### 3.1 系统资源消耗低 (P0)

| 维度 | 硬指标 | 备注 |
| --- | --- | --- |
| 空闲内存 | < 100 MB | 后台无操作时 |
| 浏览大仓库时内存 | < 500 MB | 30-50 GB 仓库 + 100k commits |
| 空闲 CPU | 0% | 不允许后台偷偷跑 |
| 进程数 | 单主进程 + 按需派生 git 子进程 | git 子进程跑完即退出 |
| 磁盘 (索引文件) | < 仓库 `.git` 大小的 10% | 索引必须能关 |

**反例参考:**
- GitHub Desktop: 500 MB+ 内存,Electron 体系
- VS Code + GitLens + Git Graph: 1.5 GB+ 内存,vscode-git-graph 大仓库能开 10+ git 子进程
- SourceTree: Mac 启动内存 800 MB+

**架构含义:**
- 倾向 **Wails / Tauri** 而不是 Electron (Wails 包 21 MB / 内存 100 MB 级;Electron 包 100 MB+ / 内存 500 MB+)
- 索引必须可关闭、按需构建、增量更新
- commit graph 不能一次性加载到内存,必须分片 / 虚拟滚动

### 3.2 超大型游戏库体验 (P0)

**目标场景:** EpicGames/UnrealEngine 这种 UE5 引擎库 (30-50 GB,不含 LFS;含 LFS 可能上 100 GB)。

| 能力 | 必需 | 说明 |
| --- | --- | --- |
| partial clone | 是 | `git clone --filter=blob:none` 起步 |
| partial clone + tree filter | 是 | `--filter=tree:0` 进一步省空间 |
| sparse checkout | 是 | 检出处不必下载整个工作区 |
| commit graph 读取 | 是 | 用 `.git/objects/info/commit-graph` 二进制索引,不解析 pack |
| multi-pack-index 读取 | 是 | 大仓库会拆分多 pack,必须读 midx |
| protocol v2 | 是 | 默认开启,服务端支持的话 clone / fetch 更快 |
| background fetch | 是 | 后台慢慢拉,不阻塞 UI |
| 增量加载 + 虚拟滚动 | 是 | commit graph 不能一次性渲染 100k 节点 |
| 进度可见 | 是 | clone / fetch / checkout 不能假死 |
| 工作区 / .git 分离浏览 | 是 | sparse checkout 下要知道哪些目录在工作区 |

**Git 维护特性 (必须后台开启):**
- `core.preloadIndex = true` — 预加载索引,加速 diff
- `core.fsmonitor = true` — 文件系统监视,加速 status
- `fetch.writeCommitGraph = true` — fetch 后自动写 commit graph
- `maintenance.auto = true` (Git 2.47+) — 后台自动维护

### 3.3 LFS 友好 (P0)

| 能力 | 必需 | 说明 |
| --- | --- | --- |
| 检测大文件 | 是 | clone 后扫描,提示用户纳入 LFS |
| `.gitattributes` 模板 | 是 | Unity / UE / Godot 三套内置模板 |
| 引导式配置 | 是 | 不让用户手写 `.gitattributes` |
| LFS 状态可视化 | 是 | 哪些文件在 LFS / 本地有 / 远程无 |
| LFS 缓存清理 | 是 | `git lfs prune` |
| LFS Cache 加速 | 是 | 本地代理层,加速 fetch |
| 超大文件下载 | 是 | 单文件 >4GB 不出错 |

**模板来源:** 抄 [JetBrains/idea-gitignore](https://github.com/JetBrains/idea-gitignore) 的 `.gitignore` 分类方式,补 `.gitattributes` 模板。

**关键引擎 LFS 经验 (从 UGit 调研借鉴):**
- UE5: `*.uasset`、`*.umap`、`*.fbx`、`*.png` (大纹理) 进 LFS
- Unity: `*.asset` 部分、`*.fbx`、`*.png`、`*.wav`、`*.unity` (场景) 进 LFS
- Godot: `*.gd` 不进 (文本)、`*.import` 不进、`*.png`/`*.wav` 进 LFS

### 3.4 超大文件优化 + 自研易用 LFS 锁 (P0)

**超大文件:**
- `git lfs pull --include="..."` — 按需下载
- 流式 diff,不全量加载
- 显示 LFS pointer 占位 vs 真实内容切换

**LFS 锁的设计难点 (UGit 自述):**
> "工蜂锁是针对游戏项目中存在大量二进制文件协作场景而设计的锁方案,解决了 Git LFS Lock 的稳定性和性能问题。"

Git LFS Lock 的已知问题:
- LFS 服务器实现参差不齐 (有的不支持锁 API)
- 锁的元数据不在 Git 中,需要单独的服务
- 锁冲突解决依赖服务端,断网时无法工作
- 工作区级锁 (worktree-level) 不存在

**xgit 锁方案：中央锁服务（已确认）**

- 采用方案 C，建设可独立部署的 `xgit-server`，面向企业内网和离线网络环境
- 客户端只保留短时本地状态与待同步操作，服务端是跨用户锁状态的唯一权威来源
- 服务端负责原子加锁、续租、释放、强制解锁、审计记录和权限校验
- 通过项目 / 仓库级租户边界隔离锁数据。客户端断线后只能查看缓存状态，不能把本地锁当成全局锁
- 首版先保证 xgit 客户端与 `xgit-server` 之间的协议可靠。是否兼容 Git LFS Lock API，留作后续单独评估
- 部署形态必须支持单机内网服务，数据库与对象存储依赖保持可选或可替换

**为什么不选本地锁为主：** 本地立即生效并不能证明其他协作者看到的是同一把锁，网络分区时尤其容易产生两个持有者。游戏二进制资产不可合并，必须由服务端统一裁决。

`xgit-server` 的协议、租约模型、高可用边界和 Git LFS Lock API 兼容性另写 ADR 展开。

### 3.5 git 磁盘占用受控 (P0)

| 能力 | 必需 | 说明 |
| --- | --- | --- |
| 占用统计 | 是 | 工作区 + `.git` + LFS 缓存分别统计 |
| gc 手动触发 | 是 | UI 按钮 + 进度 |
| gc 自动触发 | 是 | 见 §3.6 |
| LFS prune | 是 | 单独 UI 入口 |
| 悬空对象清理 | 是 | `git gc --prune=now` 显式操作 |
| reflog 清理 | 是 | `git reflog expire` |
| repack 优化 | 是 | `git repack -ad` |

**统计粒度:**
- 工作区文件总大小 (按类型 / 目录)
- `.git/objects` 大小
- LFS 缓存大小 (`git lfs env` 查)
- pack / loose objects 各自大小
- 引用大小 (refs + packed-refs)

**不替用户做:** 不要默认 `git gc --prune=now` —— 会丢悬空对象,影响恢复。

### 3.6 自动 gc (P1)

**触发策略:**

| 触发条件 | 频率 | 操作 |
| --- | --- | --- |
| 后台空闲 5 分钟 | 每次空闲 | `git maintenance run --auto` |
| 关闭应用前 | 每次 | `git maintenance run` |
| fetch / push 后 | 每次 | `git commit-graph write --reachable` |
| 累计 1000 个新对象 | 阈值 | `git gc --auto` |

**平台调度:**
- macOS: `launchd` plist
- Windows: Task Scheduler / Wails 后台 timer
- Linux: `systemd` timer / 简单 cron

**用户可配置:**
- 关闭自动 gc (高级用户)
- 调整触发阈值
- 限制运行时段 (比如工作时间不跑)

### 3.7 快速索引 + 方便搜索 (P0)

| 索引项 | 数据源 | 增量更新 |
| --- | --- | --- |
| commit graph | `.git/objects/info/commit-graph` | fetch 后自动 |
| 文件名索引 | `git ls-files --cache` | status 后增量 |
| 分支 / 标签 | `.git/refs/*` + packed-refs | 文件 watch |
| 提交消息全文 | 解析 commit 对象 | 增量 |
| 文件内容 (grep) | `git grep` / ripgrep 后端 | 不建索引,直接 ripgrep |

**搜索能力:**
- commit message 模糊搜索
- 提交 hash 精确搜索
- 作者 / 提交者搜索
- 文件路径模糊搜索
- 文件内容 grep (ripgrep 后端,增量)
- 按分支 / 标签 / 时间范围过滤

**性能指标:**
- 10k commits 仓库首屏 < 1s
- 100k commits 仓库首屏 < 3s
- 文件名搜索 < 200ms
- 文件内容 grep 与 ripgrep 持平

## 4. 对调研对象的对照评估 (按七条硬约束)

| 约束 | rebased | UGit | gitea-kanban | xgit 目标 |
| --- | --- | --- | --- | --- |
| 1. 资源消耗低 | C (IntelliJ 套壳) | B- (Electron + 不少功能) | B (Wails,21 MB) | A |
| 2. 大型游戏库 | C (依赖 IntelliJ 性能) | A+ (游戏专用) | B (有 sparse checkout 实验) | A+ |
| 3. LFS 友好 | C (通用 Git 客户端) | A+ (内置 LFS 模板) | C (无 LFS UI) | A |
| 4. 超大文件 + LFS 锁 | C | A (工蜂锁闭源,无法抄) | C (无) | A (自研锁) |
| 5. 磁盘占用可控 | C | B (有 UI) | C (无) | A |
| 6. 自动 gc | C | C | C | A |
| 7. 快速索引 + 搜索 | C (依赖 IDEA 索引) | C | B (有 Graph + 搜索) | A |

**评级说明:** A = 满足 / B = 部分满足 / C = 不满足或没体现。

**关键发现:**
- **UGit 是最接近 xgit 目标的开源/半开源参考** —— 它在大文件 / LFS / 锁这三项上是 A+。但闭源 + 仅 Tencent 内部可用,**实现细节抄不到**。可以借鉴的是产品形态 (有 / 没有某个功能) 而不是代码。
- **gitea-kanban 是工程起点** —— Wails + Go + go-git + keychain + ed25519 + 多平台打包这套工程基建可以直接复用。但它在游戏工程特性上是 C,要补 7 条里的 5 条。
- **rebased 在这七条上几乎全是 C** —— 它解决的是另一个问题 ("JetBrains 用户想要独立 Git 客户端"),与 xgit 不在同一象限。

## 5. 推论:技术选型建议更新

回到 AGENTS.md §3 的选型清单。考虑到 §3 的七条硬约束:

| 候选 | 资源 | 大仓库 | LFS | 锁 | 磁盘 | 自动 gc | 索引 | 综合 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **Wails + Go (gitea-kanban 现成)** | A | A | A | A | A | A | A | 7/7 |
| Tauri + Rust | A+ | A | A | A | A | A | A | 7/7 (但学习曲线高) |
| Electron + TypeScript (UGit 路线) | C | B | A | A | B | B | B | 4/7 |
| Flutter Desktop | B | B | B | B | B | B | B | 2/7 |
| 原生 (Swift + WinUI) | A+ | A+ | A | A | A | A | A | 7/7 (但双倍维护成本) |

**技术路线（已确认）：** **以 gitea-kanban 的 Wails + Go 工程基建为基础**，但不直接沿用其现有 Git 数据路径。客户端需要为数百 GB 仓库重做 Git 执行层、任务调度、增量数据协议和性能验收。理由：
1. 7 条硬约束都覆盖
2. 已有的工程基建可以直接复用 (Wails 配置、go-git、git CLI 嵌入、keychain、自动更新签名)
3. 用户已经在维护,二开路径清晰
4. Go + Wails 包体积 21 MB,符合"低资源消耗"
5. 需要新增 LFS、`xgit-server` 锁客户端、引擎模板和大仓库数据通道，工作量要通过基准测试重新估算

### 5.1 “内嵌模式”在数百 GB 仓库下的性能结论

这里的内嵌模式指 Wails 将 Go 后端与系统 WebView 放在同一个桌面应用内,不是把整个 Git 仓库或 Git 实现装进 WebView。仓库体积本身不会直接压垮 Wails;风险来自扫描范围、Git 对象读取方式、Go 与 JavaScript 之间的传输量和前端渲染数量。

**结论：** Wails + Go 可以保留，但只承担 UI shell 和任务编排。数百 GB 仓库的首个性能基线采用系统 `git` CLI 和 `git-lfs` CLI（已锁定为 Git 执行层的唯一路径），Go 负责启动、取消、解析和分批发送结果。这里是能力驱动与 Git 执行层硬约束的方案，不是"Git CLI 已被证明普遍快于 libgit2"的性能结论。详见 [`git-cli-libgit2-performance.md`](./git-cli-libgit2-performance.md)。`go-git` / libgit2 / 自实现 pack parser 在本 ADR 锁定下不进入主路径；fork 时需把 `app/git/` 中的 go-git 调用替换为原生 CLI。

| 层次 | 方案 | 数百 GB 仓库约束 |
| --- | --- | --- |
| WebView | 虚拟列表 + 按需详情 | 不渲染完整 commit graph / 文件树,只保留视口和小量缓存 |
| Wails bridge | 小请求、分页响应、任务事件 | 单批序列化后不超过 256 KB;禁止一次返回全量 commits / files / diff |
| Go 调度层 | 有界 worker + 可取消任务 | 同仓库写操作串行;读任务去重;关闭页面后立即取消无消费者任务 |
| Git 执行层 | 系统 `git` CLI 为主 | 复用 commit-graph、multi-pack-index、partial clone、sparse checkout、fsmonitor 和原生 pack 优化 |
| LFS 执行层 | `git-lfs` CLI | 二进制内容不经过 Wails bridge;只传元数据、进度和错误 |
| xgit 索引 | 按需、增量、可删除 | 不复制 Git 对象库;索引预算仍受 §3.1 的 10% 上限约束 |

**关键数据路径:**

1. Go 后端执行 `git status --porcelain=v2 -z`、`git log`、`git for-each-ref`、`git ls-tree` 等机器可解析命令。
2. stdout / stderr 边读边解析,按条数和字节双阈值分批;前端通过任务 ID 接收进度和结果。
3. 前端只保留当前页和相邻页,commit graph、文件树和搜索结果全部虚拟化。
4. LFS 文件、diff 原文和媒体预览不通过绑定方法返回大字节数组;文件访问采用受控本地 URL 或按范围读取,并单独验证 Windows WebView2 行为。
5. partial clone 下主动批量预取下一步需要的对象,避免浏览历史或展开目录时逐对象触发远端 fetch。

**资源与并发红线:**

- 禁止全仓库常驻文件 watcher;优先使用 built-in fsmonitor,不支持时退回显式刷新与节流扫描
- 同一仓库默认最多 2 个重读任务、1 个写任务;clone / fetch / checkout / LFS 操作纳入统一队列
- 列表请求必须有 cursor / limit,详情请求必须按用户动作触发
- 后端解析器必须流式工作,不能用 `CombinedOutput` 或 `io.ReadAll` 接住无限输出
- 读操作按需设置 `GIT_OPTIONAL_LOCKS=0`;写操作不得设置,避免破坏 Git 必需的锁语义
- 不预设 `GOMEMLIMIT`、worker 数量等固定值,应根据设备内存和 benchmark 调整

### 5.2 性能拍板前的基准门槛

目前的结论只支持进入 POC (Proof of Concept，概念验证)，**还不能证明几百 GB 仓库已经达标**。必须准备至少三档脱敏或可复现仓库：

| 档位 | 仓库特征 | 必测场景 |
| --- | --- | --- |
| L | 30-50 GB、100k commits | 启动、首屏 log、status、分支切换、搜索 |
| XL | 100-300 GB、大量 LFS 对象 | partial clone、LFS 状态、按需下载、磁盘统计 |
| XXL | 300 GB+、百万级文件或高 refs | sparse checkout、冷 / 热 status、commit graph 分页、取消任务、内存峰值 |

验收时要分别记录冷缓存与热缓存，以及 macOS / Windows / Linux 的 p50 / p95。P0 基线沿用 §3.1；另外要求首屏只加载 200-500 条记录，取消操作后 1 秒内停止 UI 更新，持续滚动时 UI 主线程不能出现明显长任务。XL / XXL 未达标时，先调整 Git 命令、分页和缓存，不要急着更换 Wails。

### 5.3 调研依据与待验证项

- Wails v2.13 官方文档说明绑定方法以 Promise 返回,Go / JavaScript 数据需要类型转换;因此 bridge 应传小 DTO (Data Transfer Object,数据传输对象),不应承载全量仓库数据: <https://wails.io/docs/howdoesitwork/>
- Git 官方 partial clone 文档明确以 100+ GiB 仓库为目标,同时指出逐个动态获取缺失对象会产生显著开销;因此 xgit 需要批量预取与 promisor remote 策略: <https://git-scm.com/docs/partial-clone>
- gitea-kanban 的 21 MB 构建产物能证明 Wails 分发体积较小,不能证明它在数百 GB 仓库上的 Git 数据路径已经达标
- 待补实测：Wails bridge 不同批大小的吞吐和 UI 长任务、Windows WebView2 的大文件 Range 响应，以及 [`git-cli-libgit2-performance.md`](./git-cli-libgit2-performance.md) 规定的 Git CLI / libgit2 / go-git 同口径基准

**Tauri** 仍然是强候选,但要从零搭基建,工程时间多 3-6 个月。
**Electron** 不推荐 (UGit 选了 Electron 是因为它起步早、团队熟;xgit 现在没这个包袱)。

## 7. 与现有调研文档的关系

> **已升格**: 本文已于 2026-07-22 升格为 `docs/adr/0001-core-requirements.md`。后续所有架构决策应引用该 ADR。

- `git-cli-libgit2-performance.md`：Git CLI、libgit2 与 go-git 的公开数据审计、能力差异和 L / XL / XXL benchmark 方案
- `ugit-competitor-analysis.md`: 提供"必须做的能力"清单 (§5.1 必抄 6 项) + 差异化方向
- `rebased-research.md`: 反例参考 —— 不要走 IntelliJ 套壳路线
- `gitea-kanban-fork-feasibility.md`: 配合 §5 的"以 gitea-kanban 为基础"做具体 fork 路径
- `roadmap.md`: 后续阶段 0-5 的进入 / 退出门槛（阶段 1 起按此推进）

## 6. 决策状态

> **已升格**: 本文已于 2026-07-22 升格为 `docs/adr/0001-core-requirements.md`。后续所有架构决策应引用该 ADR。

- 已确认：§2 第 7 条"快速索引 + 方便搜索"从 P1 提升为 P0
- 已确认：§3.4 采用方案 C，建设可由用户自主部署到内网的 `xgit-server`
- 已确认：§5 以 gitea-kanban 的 Wails + Go 工程基建为基础，但数百 GB 仓库能力必须通过 §5.2 的 POC 基准后才能进入正式实现
- 待执行：按 §5.2 方案启动 POC 基准
- 待拆分 ADR：`xgit-server` 锁协议与部署边界、客户端 Git 执行层与大仓库数据通道
