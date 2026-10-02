# SafelyFork

为任意 GitHub 仓库创建一个**安全的 fork**：fork 原仓库后自动启用 Actions，并装上一个带安全守卫的定时"同步上游" workflow。

## 为什么需要它

GitHub fork 有个经典风险：上游仓库被删除、被清空、或被 force-push 重写历史后，普通的 `git fetch upstream && git merge` 会把这些破坏原样带进你的 fork。SafelyFork 的 workflow 在每次同步前做三道检查：

| 场景 | 行为 |
|---|---|
| 上游被删除 / 转私有 | fetch 失败，安全跳过 |
| 上游文件数骤减（疑似删库/清空） | 跳过同步，并开 issue 报警 |
| 上游重写历史（无共同祖先） | 跳过同步，并开 issue 报警 |
| 正常更新 | 先打 `safepoint/日期` tag，再 merge 推送 |

同步计划为每小时运行一次（上游第 17 分），也可在 Actions 页面手动触发（`workflow_dispatch`）。

## 安装

只需 [gh CLI](https://cli.github.com/)（已登录，需 `repo` + `workflow` scope）、`jq`、`base64`。克隆本仓库即可，无其他依赖：

```bash
gh auth login        # 确保已登录
./bin/safely-fork --help
```

可选：把 `bin` 加入 PATH，或 `ln -s "$PWD/bin/safely-fork" /usr/local/bin/safely-fork`。

## 使用

```bash
# 最简：fork octocat/Hello-World 并全自动配置
# fork 会复制全部分支（不是"Copy the main branch only"），并打上 safelyfork topic
./bin/safely-fork octocat/Hello-World

# fork 到组织，重命名，完成后触发一次测试运行
./bin/safely-fork owner/repo --org my-org --fork-name repo-backup --dispatch

# 先看看会做什么，不做任何修改
./bin/safely-fork owner/repo --dry-run
```

完整选项：

| 选项 | 说明 |
|---|---|
| `--org <org>` | fork 到指定组织（默认 fork 到当前账号） |
| `--fork-name <name>` | 重命名 fork（默认：`原仓库名-SafelyFork`） |
| `--workflow-file <path>` | 要推入的 workflow 文件（默认本仓库的 `safely-fork.yml`） |
| `--workflow-name <name>` | 在 fork 中保存的文件名（默认 `safely-fork.yml`） |
| `--dispatch` | 完成后立即触发一次 workflow 测试运行 |
| `--set-fork-sync-pat` | 交互式录入 PAT，保存为 fork 的 secret `FORK_SYNC_PAT` |
| `--dry-run` | 只打印将执行的操作，不做任何修改 |

工具幂等：重复运行会复用已有 fork、更新 workflow 文件而不是报错。

默认行为：fork 复制**全部分支**（显式传 `default_branch_only=false`，不受网页端"Copy the main branch only"默认勾选影响），并给 fork 添加 `safelyfork` topic（GitHub topic 会规范化为小写；已有的其他 topics 会保留）。

## 关于 FORK_SYNC_PAT

fork 中的 `GITHUB_TOKEN` 默认没有修改 `.github/workflows/*` 的权限。如果上游的合并会改动 workflow 文件，同步会明确失败并提示。解决方法是创建一个 classic PAT（勾选 `repo` + `workflow` scope），配置为 fork 的 secret `FORK_SYNC_PAT`：

```bash
./bin/safely-fork owner/repo --set-fork-sync-pat
# 或手动：fork 的 Settings → Secrets and variables → Actions → New repository secret
```

未配置时其余同步（不涉及 workflow 文件的合并）照常工作。

## 文件说明

- `safely-fork.yml` — 核心资产：推入每个 fork 的定时同步 workflow（含三道安全守卫）
- `bin/safely-fork` — 配套 CLI，负责 fork → 推入 workflow → 启用 Actions → 启用 workflow
