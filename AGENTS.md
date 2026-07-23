# AGENTS.md — xgit

> 给在这个仓库里协作的 AI Agent（Codex / Claude Code / 其他）阅读的工程协作契约。
> 修改本文档前请先想清楚：这是给后续所有 agent 的约束，不是个人备忘。

---

## 1. 项目身份

**xgit** 是一个面向**游戏开发者**的 Git 桌面 GUI 客户端。

GitHub: `https://github.com/July-X/xgit.git`
本地: `/Users/zhongxingxing/2026/code/xgit`
协议: Apache License 2.0
默认分支: `main`

### 1.1 "对游戏开发者友好"意味着什么

游戏工程的 Git 工作流和普通业务代码不一样，agent 在做任何技术决策前必须把下面这些场景纳入考量：

| 游戏工程特征                                             | 对工具的影响                                          |
| -------------------------------------------------------- | ----------------------------------------------------- |
| 大量二进制资产（贴图、模型、音频、动画）                 | 必须原生支持 Git LFS，不能让用户手动`git lfs track` |
| 单个仓库动辄几十 GB                                      | 必须流式读写，不能一次性把对象图加载进内存            |
| Unity`.meta` / UE `.uasset` / Godot `.import` 文件 | 改动检测必须包含元数据文件，不能只看源文件            |
| 引擎工程目录约定严格                                     | 内置 Unity / UE / Godot 模板的`.gitignore`          |
| 跨时区协作（美术 / 程序 / 策划）                         | UI 文案要考虑非英语用户，默认双语                     |
| 打包构建产物（`Library/`、`Saved/`、`Binaries/`）  | 默认`.gitignore` 要覆盖这些                         |

如果一个设计决策**只对 Web/后端项目合理，但对游戏项目是负担**，优先选择后者。

---

## 2. 仓库现状

```
.git/
.gitignore        # Node.js 全家桶模板（来源未确认）
LICENSE           # Apache 2.0
README.md         # 仅标题 + 一行描述
```

- **提交数**: 1（`1238bcd Initial commit`）
- **分支**: 仅 `main`
- **代码**: 无
- **技术栈**: **Wails + Go**（2026-07-23 用户拍板；详见 §7 决策记录与 [ADR-0001](docs/adr/0001-core-requirements.md)）

**这意味着**: 仓库处于"白纸状态"。Agent 在这里的首要任务是**建立约定**，而不是套用别处的模板。盲目 `npm init -y` 然后开写是大忌。

---

## 3. 技术栈（Wails + Go，已拍板）

桌面 GUI 技术栈已于 2026-07-23 由用户拍板锁定为 **Wails + Go**，agent 无权自行改换；任何技术栈调整必须另起 ADR 并改 §7 决策记录。

- 桌面框架：Wails v2（系统 WebView：macOS WebKit / Windows WebView2 / Linux WebKit2GTK；不嵌 Chromium）
- 后端语言：Go
- 数据交换：Wails bridge（JSON / Promise）；单批 ≤ 256 KB；事件流式而非一次性返回
- Git 执行层：**原生系统 `git` + `git-lfs` CLI 独占**（2026-07-23 用户拍板）。`go-git` / libgit2 / 自实现 pack parser 一律不引入。任何想用进程内 Git library 的方案必须另起 ADR 并改 §7 决策记录。子进程控制要求：`GIT_OPTIONAL_LOCKS=0` 用于只读批扫描，`GIT_TERMINAL_PROMPT=0`、`GCM_INTERACTIVE=Never` 始终开启，长任务用 `context.WithCancel` 可取消
- 数百 GB 仓库能力：以 gitea-kanban 的 Wails + Go 工程基建为蓝本，重做 Git 执行层、任务调度、增量数据协议；fork 时需要**移除** `app/git/` 与 `app/platform/*/` 中对 `go-git` 的调用，全部替换为 `os/exec` 调原生 CLI
- v1 仅对接 Gitea：见 [ADR-0001 §"v1 平台范围"](docs/adr/0001-core-requirements.md)；本节不再展开托管平台策略

**未选中的候选与原因（已记录到历史，不再讨论）**：

| 候选 | 主要落选原因 |
| --- | --- |
| Electron + TypeScript | 内存占用高，包体积大，与 §3.1 "空闲 < 100 MB" 目标冲突 |
| Tauri + Rust + 前端 | 无显著劣势，但 Wails + Go 在游戏团队 Git 操作 + 工程基建复用上更合适 |
| Flutter Desktop | Git 需走 dart 生态或 FFI，活跃度一般 |
| 原生（Swift + WinUI） | 双平台维护成本翻倍，不符 v1 投入 |

本节后续如有补充，必须引用 ADR-0001 / ADR-0002 的对应章节；不要把单方面的平台限制、技术栈补充或 UI 规则塞进这里。

---

## 4. 工程哲学（继承自全局 SOUL.md）

1. **正确性 > 可维护性 > 可靠性 > 自动化 > 可观测性 > 可扩展性 > 性能**
   不要反过来。不要为了"未来可能用到"提前抽象。
2. **生产思维**: 写下的代码默认是要进生产、被多人维护、有 CI/CD、有日志监控的。
3. **增量改动**: 优先小步重构、保留向后兼容、不重写能用的东西。
4. **先理解，再实现**: 改任何文件前先读它、查它被谁引用、确认改动半径。
5. **不确定就问**: 模糊或有争议的判断，停下问用户，不许猜。

---

## 5. 仓库内工作约定

### 5.1 搜索与阅读

- **文本搜索**: 一律用 `rg`（ripgrep）。`grep` 仅在 `rg` 不可用时降级。
- **代码阅读**: 优先用 Codegraph（`mcp__codegraph__*`）。它已经索引过的代码不要重复 Read。
- **不要做的事**: 全仓库 `cat`、盲目 `find . -name`、在主对话里粘贴长文件。

### 5.2 分支与提交

- 分支前缀: `codex/<short-topic>`（例: `codex/lfs-track-ui`、`codex/gitignore-templates`）
- 提交信息用中文或中英混写，描述**为什么**做这个改动，不只是**做了什么**
- 单次提交只做一件事。混着改的是 review 灾难。
- 不要 `git push --force`，不要 `git reset --hard` 不带说明。

### 5.3 危险操作

以下操作**必须先和用户确认**才执行：

- 删除任何目录或文件（包括 `node_modules` 之外的）
- 强制推送、reset --hard、rebase 已推送分支
- 修改 LICENSE / CI / 安全相关配置
- 任何会影响远端的操作

**危险操作的 UI 设计原则**（2026-07-23 沉淀）：

- 工具**能**实现危险操作，但 UI 必须让用户清晰理解风险和代价；工具不替用户做决定
- 不可逆操作（强制推送、删除远程分支、reset --hard、rebase 已推送分支等）的 UI 要求：
  - 醒目警告图标 / 文案，明确告知影响范围（"将覆盖远端历史 / 将丢失未推送的本地变更"等）
  - 必须显示具体可量化的代价（被覆盖的远端 commit 数量、作者、被删除的本地 commit 数量等）
  - 用户必须勾选二次确认框（"我已知悉 / 我承担风险"）；执行按钮可强制 5 秒冷却
  - 默认走更安全的等价路径（如默认 `--force-with-lease` 而非 `--force`）
  - 写入本地审计日志（时间、用户、仓库、本地 SHA、远端 SHA、动作、变更范围）
  - push 后立即给出回滚路径（如 `git reset --hard origin/<branch>@{1}`），不替用户自动回滚
- 默认入口不暴露；带 `--force` 的 push、删除远程分支等动作放在"高级操作"折叠区或右键二级菜单
- 实现细节与回滚建议均进 [ADR-0001 §"隐式 force push 工具层克制 + UI 风控"](docs/adr/0001-core-requirements.md)

### 5.4 UI 设计

- 任何用户可见的界面改动，**先调用 `$design-taste-frontend` skill**，再动手
- 需要配色/字体参考时叠加 `$ui-ux-pro-max`
- 不许直接产出"AI 风"界面（紫蓝渐变、3D 卡片、千篇一律的圆角阴影）

### 5.5 文档输出

- 所有交付给用户的 Markdown 必须先过 `$humanizer-zh` 去 AI 痕迹
- 交付物里不许出现工具过程痕迹（"humanizer 质检"、"规则对照表"、五维评分这种）
- 首次出现的英文缩写要在括号里加中文全称（POC 概念验证、SDK 软件开发工具包、LFS Large File Storage 等）

---

## 6. 文件组织建议

仓库目前是空的，下面是建议的目录结构（**仅建议，不是强制**，落地前确认）：

```
xgit/
├── AGENTS.md
├── README.md
├── LICENSE
├── .gitignore
├── .editorconfig
├── package.json / Cargo.toml / pubspec.yaml    # 视技术选型
├── docs/                  # 设计文档、ADR、用户手册
│   ├── adr/               # Architecture Decision Records
│   └── roadmap.md
├── src/                   # 源代码
│   ├── main/              # 主进程 / 后端逻辑
│   ├── renderer/          # 渲染进程 / UI
│   └── shared/            # 主-渲共享类型
├── tests/
├── scripts/               # 构建、发布、工具脚本
└── .github/
    └── workflows/         # CI/CD
```

新建目录前确认不会和已有约定冲突。

---

## 7. 决策记录

任何会影响后续 agent 行为的决策，落地后追加到本节。格式：

```
### YYYY-MM-DD — 决策标题
- 背景: 一句话说不清就别做这个决策
- 决定: 选了什么
- 依据: 为什么
- 代价: 放弃了什么、后续要注意什么
```

### 2026-07-23 — 技术栈拍板为 Wails + Go

- 背景: AGENTS.md §3 原列了四个候选（Electron / Tauri / Flutter / 原生），默认建议 Tauri；但项目既定核心要求是"低资源消耗 + 数百 GB 游戏仓库 + Gitea-only 对接"，需要靠单一决策收敛方向
- 决定: 桌面 GUI = **Wails v2**；后端语言 = **Go**；数据交换 = Wails bridge 单批 ≤ 256 KB；Git 执行基线 = 系统 `git` + `git-lfs` CLI，`go-git` 仅作窄场景
- 依据: 见 [ADR-0001 §"技术路线"](docs/adr/0001-core-requirements.md)；数百 GB 仓库基准见 [ADR-0002](docs/adr/0002-poc-benchmark-plan.md)。Wails 的系统 WebView（不嵌 Chromium）天然规避 Electron 的内存与包体积问题，对应 §3.1"空闲 < 100 MB"
- 代价: §3 不再列四个候选对比；任何技术栈调整必须另起 ADR 并改 §7 决策记录；不再兼容 Flutter / 原生 / Tauri 路线

### 2026-07-23 — v1 平台范围：Gitea only

- 背景: UGit、TAPD、Jira 等内网集成与 GitHub Actions 等多平台适配都不在 xgit 当前资源投入范围内
- 决定: v1 **只对接 Gitea 服务端**；不引入 `PlatformAdapter` 多实现抽象；fork 时保留 `app/platform/gitea/`、删除 `app/platform/github/`
- 依据: 见 [ADR-0001 §"v1 平台范围"](docs/adr/0001-core-requirements.md)、[docs/research/gitea-only-scope.md](docs/research/gitea-only-scope.md)
- 代价: GitHub / GitLab / Forgejo 等平台扩展延后到 v1 Gitea 链路稳定后再评估；走独立 ADR 拍板

### 2026-07-23 — Git 执行层固定为原生 Git CLI

- 背景: AGENTS.md §3 与 ADR-0001 §"技术路线" 原本在 Git 执行层留有三种候选（git CLI / go-git / libgit2），前序调研（`docs/research/git-cli-libgit2-performance.md`）显示公开数据不足以证明三者相对优劣
- 决定: **Git 执行层 = 原生系统 `git` + `git-lfs` CLI 独占**；不引入 `go-git` / libgit2 / 自实现 pack parser
- 依据: 调研文档结论（"暂未拉到同口径现代 benchmark"）不应反向解读为"三种实现都可行"。fork gitea-kanban 时会保留 Wails + Go 基建，但 `app/git/`、`app/platform/*/` 中对 `go-git` 的依赖必须**移除**并替换为 `os/exec`；fork 路径见 [`docs/research/gitea-kanban-fork-feasibility.md`](docs/research/gitea-kanban-fork-feasibility.md)
- 代价: 不再有"进程内 API 提速"或 "libgit2 POC 替代" 的可逆空间；任何想恢复该路径必须另起 ADR 并改本节。若实测出原生 CLI 在 L 档样本上无法满足首屏 / 搜索门槛，需要重新评估**是不是引入更多 CLI 任务编排优化**而不是换 library

### 2026-07-23 — 隐式 force push：工具层克制 + UI 风控

- 背景: UGit 的"快速提交"分两条路径（服务端允许 non-FF / 客户端路径级冲突检测）；Hard 禁 force push 会让"rebased 远端 / 已 reflog 切回 commit"等可恢复场景断流
- 决定:
  - xgit 客户端**不**为"快速提交 / 路径级冲突检测"等"减少中断"型 UX 主动调用 `--force` / `--force-with-lease` / `+refspec`
  - 用户**显式**选择 force push 时，工具**能**调用，默认走 `--force-with-lease`
  - 任何 force 入口必须二次确认 → 显示影响范围 → 5s 冷却 → 写审计日志
- 依据: 见 [ADR-0001 §"隐式 force push 工具层克制 + UI 风控"](docs/adr/0001-core-requirements.md)、[docs/research/ugit-optimization-points.md](docs/research/ugit-optimization-points.md) §2.3 / §3.4 落点 0 / 落点 2
- 代价: 路径级冲突检测 / "快速提交"这类 UX 决策需要 ADR + POC 实测，不能直接在调研结论上拍 P1；force 路径 UI 必须独立成组件




---

## 8. 反模式（Agent 不要做的事）

- ❌ 看到空仓库就 `npm create vite@latest` 然后开整
- ❌ 在 README 里写"待完善"、"TODO"、"Coming soon"占位
- ❌ 复制别处项目的 `CONTRIBUTING.md` 当自己的
- ❌ 把 SOUL.md 的内容整段抄到本文件，只引用的部分写引用
- ❌ 提交时夹带与改动无关的文件（编辑器配置、临时笔记）
- ❌ 在 commit message 里写"AI generated"、"Co-authored-by: Codex"
- ❌ 用 emoji 当 commit message 的前缀装饰

---

## 9. 反馈与维护

本文档由人工维护。Agent 如果发现某条规则在实际协作中反复成为障碍，可以**指出**但**不要直接改**——交给人来判断是否调整。

修改本文档需要走 PR，并在 §7 决策记录里留一笔。
