# POC 基准方案

> 整理时间：2026-07-22
> 关联：[ADR-0001](0001-core-requirements.md) 七条硬约束、[Git CLI / libgit2 / go-git 调研](../research/git-cli-libgit2-performance.md)
> 状态：方案已定，待首次执行

## 1. 目标

在投入正式实现前，用可复现样本回答三个问题：

1. Wails + Go 内嵌模式下，桥接、WebView、Go↔JS 协议是否会成为数百 GB 仓库交互的瓶颈
2. 系统 `git` CLI 与 libgit2（备选）在同机同仓同语义下，p50 / p95、CPU time、peak RSS 的真实差异
3. gitea-kanban 当前的 go-git 路径在 XL / XXL 规模下是否可用，或必须切到 Git CLI / libgit2

## 2. 样本仓库

| 档位 | 目标规模 | 用途 | 来源建议 |
| --- | --- | --- | --- |
| L | 30-50 GB、100k commits | 启动、首屏、status、搜索 | 脱敏的 Unity / Godot 工程，或 EpicGames/UnrealEngine 公开子集 |
| XL | 100-300 GB、大量 LFS | partial clone、LFS 状态、按需下载 | 工程内部美术/资产密集仓的脱敏副本 |
| XXL | 300 GB+、百万级文件或高 refs | sparse checkout、冷/热 status、commit graph 分页、取消 | 合成：用 `git-fast-export` + 脚本生成百万文件仓，或采购样本 |

如果短期内拿不到真实样本，先跑合成样本：用脚本在本地生成 1M / 5M / 10M 文件的目录，分别放进 Unity `.meta`、UE `.uasset`、Godot `.import` 模板。合成样本不能代表真实业务逻辑，但足以暴露扫描、遍历、虚拟滚动的上限。

## 3. 对比对象

| 路径 | 说明 |
| --- | --- |
| A. Git CLI（子进程） | `git` 2.55+，标准 `--porcelain=v2`、`-z`、`--filter=blob:none` 输出；CLI 批命令（如 `git cat-file --batch`）作为高频查询基线 |
| B. libgit2（进程内） | 需先解决 CGo 与三平台构建；仅在同口径条件成熟时进入测试 |
| C. go-git | gitea-kanban 当前路径，作为"改造工作量最小"的对照；不测它不具备的能力（partial clone、LFS） |

如果 B 在当前环境无法编译，不要伪造数据；记录"N/A：未构建 libgit2"并继续推进 A 与 C。

## 4. 指标与采集

每次运行采集：

- wall-clock：从命令发起到最后一个字节被前端消费
- CPU time：user + system（独立进程用 `time(1)`，进程内 API 用 `getrusage` / 平台等价）
- peak RSS：以操作系统报告的峰值工作集为准，注明采集方式（`ps`、`top`、`GetProcessMemoryInfo`、`/proc/<pid>/status`）
- 网络：总 fetch 字节、pack bytes、协议版本
- 缓存状态：冷（清 page cache）/ 热（第二次及以后运行），清缓存方法必须记录，无法可靠清理时只报告热缓存

统计方法：预热后至少 30 次，报告中位数与 p50 / p95。保留原始 JSON / CSV，Git CLI 额外保存 Trace2。

## 5. 测试场景

| 场景 | Git CLI 命令 | 进程内对应 |
| --- | --- | --- |
| 仓库打开 | `git rev-parse` / `git status --uno` | repository open + lightweight status |
| 工作区状态 | `git status --porcelain=v2 -z` | `git_status_list_new` / go-git `Worktree.Status()` |
| 历史首屏 | `git log -n 500 --format=... -z` | revwalk 500 commits |
| 全历史遍历 | `git rev-list --all` | 完整 revwalk |
| diff | `git diff --no-ext-diff --no-textconv` | tree / index / workdir diff，不读取大型 LFS 真实内容 |
| fetch | `git fetch --filter=blob:none` | 最接近的可用 fetch，不能等价 filter 时标记 N/A |
| checkout | sparse checkout cone | checkout 相同路径集合 |
| 连续交互 | 100 次 refs / HEAD / object 查询 | 批量 CLI 或持久 `cat-file --batch` 对进程内 API |

## 6. 通过门槛（暂定）

| 场景 | L | XL | XXL |
| --- | --- | --- | --- |
| 首屏 log（wall-clock） | < 1s | < 2s | < 3s |
| 文件名搜索 | < 200ms | < 500ms | < 1s |
| status（热缓存，0 / 1% 修改） | < 1s | < 3s | < 8s |
| 首字节延迟（连续交互） | < 30ms | < 80ms | < 150ms |
| 峰值内存（浏览态） | < 500 MB | < 800 MB | < 1.2 GB |

未达标时先调整命令、分页、缓存、虚拟列表，再考虑换引擎或换技术栈。

## 7. 执行顺序

1. 准备 L 档样本；先在纯 CLI 下跑通所有场景，建立可工作基线
2. 把同一套场景搬到 gitea-kanban 的 go-git 路径，记录可用与不可用
3. 如果 A / C 差异不可接受，再引入 libgit2
4. 通过 L 档后，按 XL → XXL 放大；任一级未达标时回到第 3 步调整

## 8. 当前阻塞

- [ ] libgit2 源码与 Go 绑定（git2go）在当前环境未编译
- [ ] XL / XXL 真实样本仓库未就绪，先跑合成样本
- [ ] 无真实数百 GB 游戏美术仓，合成样本不能代表业务逻辑

这些问题解决之前，不要杜撰数字。POC 文档只允许写入实际测得的数字。
