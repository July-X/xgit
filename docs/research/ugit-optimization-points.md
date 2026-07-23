# UGit 设计优化点分析：快速提交 与 稀疏检出

> 整理时间：2026-07-23
> 来源：
> - 用户提供的 2 张截图（`.reasonix/attachments/clipboard-20260723-222917...` 和 `...-223001...`）
> - UGit 官方手册页面：<https://docs.qq.com/doc/DTGF6RmVLa0RZUEJm>（SPA，匿名请求仅拿到外壳，正文需登录）
> - Git 官方 `git sparse-checkout` 文档：<https://git-scm.com/docs/git-sparse-checkout>
> 用途：补充 [`ugit-competitor-analysis.md`](./ugit-competitor-analysis.md) 与 [`ugit-usability-observations.md`](./ugit-usability-observations.md)，把"快速提交"和"检出子目录"两个优化点单独抽出来分析
> 状态：本文档只整理行为观察 + 已开源的 Git 原生能力，**不杜撰 UGit 实现细节**
## 1. 截图清单与本次能确认的事实

| # | 文件 | 尺寸 | 场景 | 截图内可见事实 |
|---|---|---|---|---|
| 1 | `clipboard-20260723-222917.157892-000010.png` | 1244×1148 | 设置 → 高级 | 选项清单（"使用变基更新"、"推送后自动解锁文件"、"忽略文件权限变化"、"冲突处理后自动提交"、"回滚合并提交"、"默认预览表格"、"把 .csv 当表格处理"、"双击切换分支"、"大小写不敏感"、"历史视图提交批次最大加载数"、"单文件默认查看详细历史"、"磁盘空间低于 X GB 提醒"、"克隆时默认克隆为美术仓库"、"LFS 并发传输数"、"工蜂 AI 提交信息"、"专业仓库待更新文件标记"、"开机自启动"等） |
| 2 | `clipboard-20260723-223001.476342-000011.png` | 851×574 | 添加平台 | 平台列表（工蜂合作 / 海外 / 社区 / 租户版 / CODING / Github / GitLab 私有化 / TAPD / Jira Cloud） |

> 像素尺寸由 `sips -g pixelWidth -g pixelHeight` 读取。截图 2 中可见的平台清单已在上表列出，便于与官方手册核对。这两张截图**都没有直接说明 "快速提交" / "检出子目录" 的实现路径**，下文以 Git 原生能力反推。

## 2. 快速提交（Quick Commit）

### 2.1 行为描述（UGit 文档原文）

UGit 文档表述（用户引用）：

> 快速提交：原生 Git 提交流程，如果远程有新的提交，Git 会强制要求先更新再提交，在一个大型项目中，提交流程会因为远程频繁变更而不停中断，影响工作效率。UGit 的快速提交，可以实现只要用户提交的文件其他人没修改，可以在不更新情况下直接完成提交，不会因远程频繁变更而中断提交流程，让大型团队协作更加流畅。

### 2.2 这一行为对应哪些 Git 原生能力

Git 推送被"非快进"（non-fast-forward）拒绝，本质是 `receive.denyNonFastForwards` 或 `receive.denyDeletes` 服务端策略，不是客户端能力。UGit 文档的"快速提交"行为有两条独立路径，需要区分。

#### 路径 A：服务端允许非快进推送

- 服务端需启用 `receive.denyNonFastForwards=false`（GitHub 默认禁，对此不通）
- GitLab / Gitea 默认也为禁
- 客户通常无权改服务端策略

#### 路径 B：客户端侧按文件做 push 安全性判断

- 用户提交前，UGit 会先 `fetch`（或比较 `HEAD..origin/<branch>` 的差异列表）
- 仅当本地变更路径集与远端新提交变更路径集**无交集**时，才执行"等价于 fetch + cherry-pick + push"组合
- 等价操作序列：

  ```
  git fetch origin <branch>
  # 如果用户提交的路径与远端新提交的路径不重叠
  git push origin <branch>
  ```

  这就避开了"remote has diverged，必须先 merge"的硬阻塞。
- 若检测到重叠，要么走 stash + merge / rebase，要么明确告知用户需要手动解决

这两条路径中：

- A 路径在 GitHub / GitLab / Gitea 公共实例下多数不可行
- B 路径在路径级冲突检测后，仍是 fast-forward push，对服务端策略友好

UGit 文档说"只要用户提交的文件其他人没修改"——这正是路径 B 的语言，与服务端特权无关。

### 2.3 xgit 借鉴方案（不杜撰）

#### 落点 0：原则（2026-07-23 用户拍板修订）

**工具可实现，用户知情的责任归用户**。本节同时继承上一版的硬红线思想，但承认完全禁用 force push 会让"rebased 远端 / 已 reflog 切回 commit"等可恢复场景断流，所以把"工具能力 vs 用户责任"按以下两层切分：

- **工具实现层**：
  - xgit 客户端**不**主动为"快速提交 / 路径级冲突检测"调用 `--force`；路径 B 命中无交集条件时仅做 fast-forward push
  - 用户显式要求 force push 时，工具**能**调用 `git push --force` / `--force-with-lease` / `+refspec`；默认走 `--force-with-lease` 而不是 `--force`（更安全）
  - 工具内部**不**静默重试 --force；不读远端错误响应后兜底升格
- **UI 责任层**（必须先达到才允许执行）：
  - 弹二级确认框，列出影响范围（即将被覆盖的远端 commit 数量、作者、最近时间）
  - 必须勾选"我已知悉，可能覆盖远端历史与他人的提交"
  - 必须输入分支名（防止误操作）或保留 5s 冷却期
  - 写入本地审计日志：`{时间、用户、仓库、本地 SHA、远端 SHA、要覆盖的 commit 数量、动作}`
  - 风险文本模板按操作类型提供（force push / 删除受保护分支 / 真的删除远程分支 / 改写 tag 等）
- **默认行为**：
  - "提交 + Push"按钮不调 --force
  - "快速提交并推送"按钮（路径 B 命中）不调 --force
  - 任何调用 --force 的入口都在"高级操作"折叠区，或右键菜单二级菜单；带显眼的危险图标

理由：完全硬禁迫使团队到 CLI 去做 force push，反而让用户失去 xgit 在风险展示、审计、撤销窗口上的优势。

#### 落点 1：路径级冲突检测，作为 push 前置门

Go 端实现要点（与 ADR-0001 / §5.1 Git CLI 路线一致）：

1. 提交按钮按下时，记录本次提交即将引入的 path 集合 P
2. `git fetch origin <branch>`，获取 `origin/<branch>`
3. `git diff --name-only origin/<branch>..HEAD`（即将 push 的本地变更）
4. `git diff --name-only HEAD..origin/<branch>`（远端未拉取的变更）
5. 若两组路径集合**无交集**，调用 `git push origin <branch>`（fast-forward，**不**带 `--force`）
6. 若有交集，回退到标准流程（提示用户 merge / rebase / stash / 或显式 force push）
7. push 中若出现 rejection / pre-receive hook 报错，**绝不**自动加 `--force` 重试，但 UI 提供"以强制推送重试"二级入口，进入 §"落点 2"的风控流程

这一实现不依赖任何服务端特权，可在 GitHub / GitLab / Gitea 等通用平台工作；它的"快进"意义在于减少在不必要场景下阻断用户，不削弱远端分支保护。

#### 落点 2：force push UI 风控（必须先达到才执行）

UI 触发路径：

- 右键分支 → "强制推送"（二级菜单）
- 推送失败后弹层 → "以 force-with-lease 重试"（明确按钮，带警告图标）

执行前必须做到：

1. 显示差异摘要：远端将被覆盖的 commit 数量、作者列表、最新 commit hash 时间
2. 二级确认框文本：

   ```
   ⚠ 强制推送风险

   远端分支：origin/<branch>
   即将覆盖 <N> 个他人提交（作者：张三 / 李四 …）
   你的本地 HEAD：a1b2c3d
   远端 HEAD：e9f8g7h

   可能导致他人工作丢失。建议先与协作者沟通后再继续。

   [取消]   [取消]   [□ 我已知悉，可能造成他人工作丢失]   [继续强制推送（强制 5s 后启用）]
   ```

3. 用户必须勾选二次确认框；执行按钮在用户勾选后再强制 5 秒冷却才能按下
4. 强制推送默认走 `git push --force-with-lease` 而不是 `--force`；UI 提供复选框让用户明确"确认远端无新人提交还要继续"
5. 写入审计日志：

   ```json
   { "ts": "...", "user": "...", "repo": "...", "branch": "...",
     "local_sha": "a1b2c3d", "remote_sha": "e9f8g7h",
     "overwritten_commits": N, "authors": [...],
     "force_mode": "--force-with-lease" }
   ```

6. push 后立即给出 reflog / `git reset --hard origin/<branch>@{1}` 回滚路径；不替用户自动回滚

#### 落点 3：POC 验证

- 在 L 档样本（参考 [ADR-0002](../adr/0002-poc-benchmark-plan.md) §6）下，分别测试有远程新提交时的两条路径，记录 wall-clock 与成功率
- 不能在未跑样本前在文档中写"已验证快速提交有效"
- POC 必须包含"路径 B 误判回退到 force push"的回归用例，确保 UI 弹出的是二级确认框而不是静默强制推送

- 在 L 档样本（参考 [ADR-0002](../adr/0002-poc-benchmark-plan.md) §6）下，分别测试有远程新提交时的两条路径，记录 wall-clock 与成功率
- 不能在未跑样本前在文档中写"已验证快速提交有效"

## 3. 检出子目录（Sparse Checkout）

### 3.1 行为描述

UGit 文档表述（用户引用）：

> 支持检出子目录：对于大型仓库，克隆完整仓库下来可能需要很长时间，有些时候，我们只需要下载一个或若干子目录即可进行工作，此时可以使用 UGit 克隆时，只勾选工作需要用到的目录进行克隆，这样可以快速完成，不用等待。

### 3.2 Git 原生能力

`git sparse-checkout` 自 Git 2.25 起转为推荐用法（自 Git 2.26 起 cone 模式为默认）。官方文档要点：

- `git sparse-checkout init --cone` 启用 cone 模式
- `git sparse-checkout set <DIR>` 通过目录列表定义工作区；cone 模式下自动转化为正向 + 反向 gitignore 模式
- 自 Git 2.34 起，sparse index 与 sparse-checkout 协同工作，可以让 `status` / `add` 等命令更快
- 真实减少落盘量的关键：与 partial clone（`--filter=blob:none` / `--filter=tree:0`）协同，避免一开始拉全量对象

执行序列：

```
git clone --filter=blob:none --sparse <url>
git sparse-checkout init --cone
git sparse-checkout set <DIR1> <DIR2>
```

### 3.3 UGit 大概做了什么

依据 Git 原生能力反推：

- 客户端在"添加仓库 / 克隆"流程中嵌入一个目录树选择器
- 选择完成后，客户端拼装 `git clone` + `git sparse-checkout set` 命令序列
- 不发明新协议，全靠 Git 已开源能力

### 3.4 xgit 借鉴方案

#### 落点 0：原则（与 §2.3 落点 0 一致）

- 稀疏检出过程**绝不**触发 `git push --force` 或破坏服务端 ref 的命令
- 任何"克隆后裁剪 origin/<branch>"动作都视为只读操作；如发现远端 ref 历史与本地 fetch 视图错位，提示用户走标准 fetch / repair
- 确实需要修复远端 ref 时，调用 §2.3 落点 2 的 UI 风控流程（用户知情 → 二次确认 → 5s 冷却 → 审计日志）
- "工具能实现、责任归用户"的红线与 §2.3 共享同一条原则，不重复声明

#### 落点 1：克隆流程嵌入目录选择

- 添加仓库 / 克隆新仓库时，给用户在远端目录树（`git ls-tree -r --full-tree origin/main`）上做多选
- 选择结果映射为 `git sparse-checkout set <dirs>` 命令行
- 提供"全部" / "稀疏"两种预设，分别走普通 clone + sparse-checkout vs `--filter=blob:none --sparse`

#### 落点 2：与 partial clone 协同（ADR-0001 §3.2）

- 新增仓库时，默认建议 `--filter=blob:none` + `--no-checkout` 或 `--filter=tree:0`，而不是无脑全量克隆
- LFS 大文件检出按需 `git lfs fetch --include=<path>`
- 给出当前仓库的实际占用与"如果克隆全部"对比，让用户有选择

#### 落点 3：UI 提示当前工作区是 sparse

- 标题栏明显标记"sparse"，避免用户误把工作区当成"全量仓库"
- 提供"补全更多目录"入口，背后是 `git sparse-checkout add <dir>`

#### 落点 4：POC 验证

- XL 档样本（300 GB+，参见 [ADR-0002 §6](../adr/0002-poc-benchmark-plan.md)）下，记录"全量克隆 vs sparse-checkout 单目录"的 wall-clock 与磁盘占用差距
- 不能在未跑样本前在文档中写"已验证 sparse 提速 N 倍"

## 4. 不杜撰 / 数据真实性边界

下列信息本文**不主张**，需后续 POC 或用户实测后确认：

- UGit 的具体实现代码：闭源，无法核对
- UGit 是否走了路径 A 还是路径 B：无法从截图与公开手册确认
- UGit 实际的"快速提交"对哪些路径规则生效：未提供
- UGit sparse-checkout 的目录选择 UI 对应的源码：未提供

如需数字，沿用 [ADR-0002](../adr/0002-poc-benchmark-plan.md) 与 [`git-cli-libgit2-performance.md`](./git-cli-libgit2-performance.md) 的方法学：同口径、可复现、原文出处。

## 5. 对 xgit 已建立文件的影响

不直接修改 ADR-0001 的 P0/P1 表，新增两条 Backlog 项并入调研文件：

| 项 | 来源 | 优先级 | 落地路径 |
|---|---|---|---|
| 路径级冲突检测 + 快速提交（仅 fast-forward） | 本文档 §2.3 + §2.3 落点 0 | 待 P1 评估 | 列入 [ADR-0001"待拆分 ADR"](../adr/0001-core-requirements.md) 第 4 项相关："客户端 Git 执行层与大仓库数据通道"，单独 ADR 立项 |
| 克隆流程嵌入 sparse-checkout + partial clone | 本文档 §3.4 + §3.4 落点 0 | 待 P0/P1 评估（与 §3.2 大仓库体验强相关） | 列入 [ADR-0001"待拆分 ADR"](../adr/0001-core-requirements.md) 第 4 项相关："稀疏检出与 partial clone 的默认策略" |
| **隐式 force push 工具层克制 + UI 风控** | 本文档 §2.3 落点 0 + §2.3 落点 2 + §3.4 落点 0 + 2026-07-23 用户拍板修订 | **工具层契约 + UI 责任**（与 P0 同级约束） | 进入 ADR-0001 §"技术路线"段；待拆分 ADR "客户端 Git 执行层" §1 必含：xgit 不主动 --force，但提供知情 --force UI 与审计日志 |

> **关于 P0 决策**：本文件**不**给这两项重新拍 P0 / P1，全部留在 Backlog。提升或下调需要真实用户验证 + ADR 修订流程，而不是调研阶段推断。

## 6. 相关引用

- [`docs/research/ugit-competitor-analysis.md`](./ugit-competitor-analysis.md)：功能矩阵与技术栈推断
- [`docs/research/ugit-usability-observations.md`](./ugit-usability-observations.md)：5 张截图的易用性观察
- [`docs/adr/0001-core-requirements.md`](../adr/0001-core-requirements.md)：顶层约束、技术路线、待拆分 ADR 清单
- [`docs/adr/0002-poc-benchmark-plan.md`](../adr/0002-poc-benchmark-plan.md)：L/XL/XXL POC 基准方案
- [`docs/research/git-cli-libgit2-performance.md`](./git-cli-libgit2-performance.md)：Git 执行层的公开数据审计
- Git 官方 sparse-checkout：<https://git-scm.com/docs/git-sparse-checkout>
- UGit 官方手册：<https://docs.qq.com/doc/DTGF6RmVLa0RZUEJm>
