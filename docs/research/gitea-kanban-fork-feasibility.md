# 调研 — 以 gitea-kanban 为 xgit 基础

> 调研时间: 2026-07-22
> 目的: 评估 `~/2026/code/gitea-kanban` (GitHub: July-X/gitea-kanban) 是否适合作为 xgit 的代码基础。

## 1. 结论

**可以作为基础,但不是"直接 fork + 加功能"**,而是"fork + 重塑方向"。

理由: gitea-kanban 与 xgit 在 **技术栈**、**桌面打包流水线**、**Git 核心能力** 上高度重叠,二开能省下大量搭基建的时间;但在 **产品定位** 上分歧明显 (gitea-kanban 是 Git Graph + Kanban,Gitea/GitHub API 绑定;xgit 是纯 Git GUI,游戏工程友好),需要在 fork 时明确删 / 改 / 加。

**最强可行性路径: 在本地把 `~/2026/code/gitea-kanban` 软链 / 复制成新仓库 `~/2026/code/xgit`,删掉 Kanban 与 Gitea/GitHub 适配器,保留 Git Graph 引擎、go-git / git CLI 集成、桌面打包流水线、自动更新签名系统。**

## 2. gitea-kanban 基本情况

| 项 | 值 | 来源 |
| --- | --- | --- |
| 仓库 | [July-X/gitea-kanban](https://github.com/July-X/gitea-kanban) |
| 协议 | Apache 2.0 | README 徽章 |
| 当前版本 | v0.8.20 | 最近 commit / CHANGELOG |
| 平台 | macOS / Windows | README |
| 包大小 | ~21-22 MB | README |
| 主分支 | master | git branch |
| 活跃 feature 分支 | feat/gitgraph-vscode-port、feat/gitgraph-vscode-recheck、feat/v0.8.0、feat/v0.5.0、feat/v0.4.0 | git branch -a |
| AGENTS.md | 存在 (36 KB) | 工作目录 |
| CLAUDE.md | 存在 (28 KB) | 工作目录 |
| 工作树 | `.worktrees/` 下有 worktree,说明用 worktree 协作 | ls -la |

## 3. 技术栈

### 3.1 后端

- **Go 1.26.0** + **Wails v2.12.0**
- 关键依赖 (from `go.mod`):
  - `github.com/go-git/go-git/v5 v5.13.2`
  - `github.com/wailsapp/wails/v2 v2.12.0`
  - `github.com/zalando/go-keyring v0.2.8` (系统钥匙串)
  - `github.com/google/uuid v1.6.0`
  - `golang.org/x/crypto v0.53.0` (ed25519 签名验证)
  - `github.com/sergi/go-diff v1.3.2` (行内 diff)

### 3.2 Git 操作路线 (重要决策)

`app_gitbinary.go` 注释里说:

> v2.0 拍板「默认内嵌 git 2.55.0」
>
> 流程:
>   1. OnStartup 调 gitbinary.Init → 释放嵌入二进制到 ${dataDir}/tools/git/
>   2. OnStartup 调 gitbinary.SetUserOverride(...) ...
>   3. 用户在 SettingsView 改路径: ...

也就是说 **gitea-kanban 选择的是"嵌入 git CLI 二进制 + 调 CLI",而不是 go-git 内嵌**。这与 rebased / git4idea 的路线一致。

但 `go.mod` 里又有 `go-git/v5`,所以是**两条路线并存**:核心 Graph / 深历史 deepen 用 go-git (流式),提交 / 推送 / 拉取等用 git CLI。**这是个值得抄的模式**。

### 3.3 前端

`frontend/package.json` + `vite.config.ts` 存在,推断是 **TypeScript + Vite**。

### 3.4 模块结构

```
gitea-kanban/
├── app/                       # 后端模块
│   ├── git/                   # Git 核心 (Graph / log / deepen)
│   ├── gitbinary/             # git CLI 嵌入与管理
│   ├── platform/              # 多平台适配层
│   │   ├── gitea/             # Gitea API (xgit 不需要)
│   │   └── github/            # GitHub API (xgit 不需要)
│   ├── updater/               # 自动更新 + ed25519 签名
│   ├── store/                 # 本地存储
│   ├── ipc/                   # Wails IPC
│   ├── logx/                  # 日志
│   └── config/                # 配置
├── frontend/                  # 前端 (TS + Vite)
├── app.go, app_*.go           # Wails App 主入口
├── go.mod, go.sum
└── .worktrees/                # Git worktree 协作
```

代码体量: 21 个 `.go` 文件,约 6,103 行 Go 代码。

## 4. 与 xgit 需求的重叠度

| 能力 | gitea-kanban 已有 | xgit 是否需要 |
| --- | --- | --- |
| Git Graph 渲染 (复刻 vscode-git-graph) | 有 (v0.6.0+ 自研 lane 算法) | 是,核心 |
| git CLI 嵌入 + 跨平台管理 | 有 (v0.4.0+ 嵌入 git 2.55.0) | 是,直接抄 |
| go-git 流式读取 | 有 (NoCheckout 轻量模式) | 是 |
| 本地存储 + 配置 | 有 | 是 |
| 自动更新 + ed25519 签名 | 有 (v0.8.0+) | 是,直接抄 |
| keychain 集成 | 有 (zalando/go-keyring) | 是,直接抄 |
| Gitea / GitHub API 适配 | 有 | 否,删 |
| Kanban 视图 | 有 | 否,删 |
| 提交签名验证 (GPG 9 种状态) | 有 | 否 (GPG 不是游戏工程关心) |
| 文件锁 / LFS 模板 / 引擎元数据 | 无 | 是,新写 |
| 工蜂锁等价物 | 无 | 是,新写 |
| 快速提交 (绕过更新) | 无 | 是,新写 |
| Unity / UE / Godot `.gitignore` 模板 | 无 | 是,新写 (可参考 JetBrains/idea-gitignore) |
| sparse checkout 友好 UI | 无 | 是,新写 |
| 本地 LFS Cache 加速 | 无 | 是,新写 |
| ed25519 自动更新 | 有 | 是,直接抄 |

## 5. 二开路径 (具体怎么 fork)

### 5.1 第一步:克隆 + 改名

```bash
cd ~/2026/code
git clone git@github.com:July-X/gitea-kanban.git xgit
cd xgit
git remote remove origin
git remote add origin git@github.com:July-X/xgit.git
```

注意: 这会保留 gitea-kanban 的全部 commit 历史。要不要清空是一个产品决策——保留有利于代码溯源 (能找到原始 commit 是谁写的),清空有利于"xgit 是一个全新项目"的品牌感。

### 5.2 第二步:删

| 删除项 | 来源 | 原因 |
| --- | --- | --- |
| `app/platform/gitea/` | 整个目录 | 不绑 Gitea |
| `app/platform/github/` | 整个目录 | 不绑 GitHub |
| `app_gitgraph_dto_e2e_test.go` 里的 Kanban 用例 | 测试文件 | Kanban 不做 |
| `app_issue.go`, `app_pull.go` | 文件 | PR / Issue 不在 v1 范围 |
| Kanban 视图相关的前端代码 | `frontend/` 内 | v1 不做 |
| Gitea / GitHub OAuth 流程 | 适配器内 | v1 不做 |

### 5.3 第三步:保留 + 改名

| 保留项 | 处理 |
| --- | --- |
| `app/git/`, `app/gitbinary/`, `app/updater/`, `app/logx/`, `app/config/`, `app/store/`, `app/ipc/` | 整体保留,模块名 `gitea-kanban/app/...` 批量改成 `xgit/app/...` |
| `app_gitgraph.go`, `app_gitbinary.go`, `app_clone.go`(推测) | 保留,做包名批量替换 |
| Wails 配置 (`wails.json`) | 改 productName / 标识 |
| 桌面打包脚本 | 改产物名 |
| AGENTS.md (36 KB) | **重点: 读一遍后选择性吸收**,不要整文件复制 |

### 5.4 第四步:加 (xgit 独有)

| 新增项 | 说明 |
| --- | --- |
| `app/gitlock/` | 自研文件锁 (不等同于 Git LFS Lock) |
| `app/gitlfs/` | LFS 模板 + 大文件分析 + 本地 Cache |
| `app/gitengine/` | Unity `.meta` / UE `.uasset` / Godot `.import` 检测 |
| `app/gitignore-templates/` | Unity / UE / Godot 模板 (参考 JetBrains/idea-gitignore) |
| `app/quickcommit/` | 快速提交 (绕过更新) |
| `app/sparse/` | sparse checkout 友好 UI |
| 前端新视图 | 锁管理 / LFS 配置 / 引擎模板选择 |

## 6. 风险与决策点

| 风险 | 严重度 | 缓解 |
| --- | --- | --- |
| gitea-kanban 老用户困惑 (突然少功能) | 中 | 明确两个项目各自维护,不发同一 release |
| commit 历史归属 (xgit 是否承认来自 gitea-kanban) | 中 | 保留 commit, README 加 "forked from July-X/gitea-kanban at v0.8.20" |
| AGENTS.md / CLAUDE.md 与 xgit 项目规则冲突 | 中 | 已写的 `xgit/AGENTS.md` 是独立文件,以它为准 |
| package 大小 (21 MB → xgit 删 Kanban 后可能更小) | 低 | 正向影响 |
| 二开方向跑偏 (忍不住又把 Kanban 加回来) | 高 | AGENTS.md §8 写明"反模式" |
| 跨平台支持 (gitea-kanban 没有 Linux) | 中 | xgit v1 可以只做 macOS + Windows,后续再加 Linux |

## 7. 进一步动作建议

1. **先读一遍 gitea-kanban 的 AGENTS.md** —— 它有 36 KB,可能包含很多有用的工程规范,可以借鉴到 xgit。
2. **先读一遍 `app/git/` 和 `app/gitbinary/`** —— 这两个是 xgit 最核心要保留的模块,确认能直接复用。
3. **画一份 `app/` 模块依赖图** —— 确认删除 Gitea/GitHub 适配器后,有哪些间接依赖需要清理。
4. **跑一遍 `wails build` 验证** —— 在 macOS 上确认当前 gitea-kanban 能干净构建,作为 xgit 二开的基线。
5. **决定 commit 历史策略** —— 保留 / 清空 / squash 到第一个 xgit commit?

## 8. 待补充

- gitea-kanban 的 Wails 配置细节 (`wails.json`)
- 前端实际技术栈 (Vue / React / Svelte ?)
- AGENTS.md / CLAUDE.md 的具体内容
- `.worktrees/` 的使用方式
- 实际克隆 + 删除 Kanban 后的代码量估算
