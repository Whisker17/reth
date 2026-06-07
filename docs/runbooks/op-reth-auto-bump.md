# op-reth 自动更新 Runbook

适用对象：DevOps / Release Engineering

目的：指导 DevOps 操作 GitHub Actions workflow，将 Mantle/reth 同步到上游 `ethereum-optimism/optimism` 的 `op-reth/v*` tag，复制上游锁定的 `paradigmxyz/reth` rev，运行验证，并创建需要人工 review 的 PR。

Workflow 文件：

```text
.github/workflows/update-op-reth.yml
```

默认 base branch：

```text
mantle-elysium
```

自动化分支：

```text
op-reth-auto-bump
```

本文命令示例使用的 GitHub 仓库：

```text
mantle-xyz/reth
```

如果生产仓库的 owner/name 不同，执行命令前请把本文所有 `--repo mantle-xyz/reth` 和相关 URL 替换成生产仓库标识。

当前上游 base tag 记录文件：

```text
.op-reth-base-tag
```

当前预期初始值：

```text
op-reth/v2.2.1
```

## Workflow 做什么

每次手动运行时，workflow 会执行以下流程：

1. 读取 `.op-reth-base-tag`，确定当前 Mantle/reth 基于哪个上游 `op-reth` tag。
2. 确定目标上游 tag：
   - 如果手动填写了 `target_tag`，使用该值；
   - 如果 `target_tag` 为空，自动检测 `ethereum-optimism/optimism` 最新稳定版 `op-reth/v*` tag。
3. 基于 `base_branch` 创建或更新 `op-reth-auto-bump` 分支。
4. 拉取上游旧 tag 和新 tag 对应的：
   - `rust/op-reth/**`
   - `rust/Cargo.toml`
5. 将上游 `rust/op-reth/**` 通过逐文件 three-way merge 应用到本地 `op-reth/`。
6. 从上游 `rust/Cargo.toml` 读取上游锁定的 `paradigmxyz/reth` rev，并复制到本地 `Cargo.toml`。
7. 如果 reth rev 发生变化，运行 `cargo update` 更新 `Cargo.lock`。
8. 在 PR body 中报告 Mantle fork 依赖状态，包括 `revm`、`revm-inspectors`、`alloy-evm`、`op-alloy` 相关依赖。
9. 如配置了 Claude Code，则尝试自动修复 merge conflict markers 或编译错误。
10. 运行轻量级验证。
11. 如果 `dry_run=false`，push `op-reth-auto-bump` 分支，并创建或更新 PR。
12. 如果配置了 `LARK_WEBHOOK_URL`，发送 Lark 通知。
13. 上传诊断 artifacts。

workflow 不会自动 merge。最终合并必须由人工 review 后完成。

## 安全模型

workflow 默认：

```text
dry_run=true
```

当 `dry_run=true` 时，workflow 会执行同步和验证逻辑，但跳过：

```text
git push
gh pr create
gh pr edit
```

以下场景必须先用 `dry_run=true`：

1. 第一次运行 workflow。
2. 第一次同步某个新的 `target_tag`。
3. workflow 文件刚改过。
4. secrets、runner、label 或权限刚调整过。

只有在确认要更新远端 `op-reth-auto-bump` 分支并创建或更新 PR 时，才使用：

```text
dry_run=false
```

如果同步后仍有冲突，workflow 会保留可供人工接手的状态：

- draft PR 分支中保留未解决冲突现场；
- 冲突报告写入 `docs/op-reth-auto-bump-conflicts.md`；
- workflow artifacts 中包含 conflict 列表和 `worktree.diff`。

## 所需权限

操作人需要具备：

1. GitHub 仓库 admin 或 maintainer 权限，用于配置 Actions secrets、variables 和 labels。
2. 触发 GitHub Actions workflow 的权限。
3. 查看 workflow logs 和下载 artifacts 的权限。
4. 如使用 bot 账号，需要创建或管理 Personal Access Token 的权限。
5. 可选：Lark Incoming Webhook 配置权限。
6. 可选：Claude sub2api proxy 或组织内 Claude API key 的访问权限。

## 必需 GitHub Secrets

在 GitHub 仓库中进入：

```text
Repository -> Settings -> Secrets and variables -> Actions -> Repository secrets
```

配置以下 secrets：

| Secret | 是否必需 | 用途 |
|--------|----------|------|
| `BOT_GITHUB_TOKEN` | 必需 | checkout、git push、创建 PR、更新 PR、编辑 label、GraphQL draft 转换 |
| `ANTHROPIC_API_KEY` | 推荐 | 允许 Claude Code 修复 conflict markers 和编译错误 |
| `ANTHROPIC_BASE_URL` | 推荐 | Claude sub2api proxy base URL |
| `LARK_WEBHOOK_URL` | 可选 | 发送 workflow 结果到 Lark |

### BOT_GITHUB_TOKEN 要求

推荐做法：

1. 使用 bot GitHub 账号创建 token。
2. 使用 classic PAT，并赋予：
   - `repo`
   - 如果 bot 还需要 push workflow 文件变更，再加 `workflow`
3. 在仓库 secrets 中保存为：

```text
BOT_GITHUB_TOKEN
```

该 token 必须能完成：

- checkout 私有仓库内容；
- push 到 `op-reth-auto-bump`；
- 创建 PR；
- 编辑 PR title 和 body；
- 添加和移除 labels；
- 通过 GitHub GraphQL 将已有 PR 转成 draft。

如果 workflow 在 checkout、push 或 PR 创建阶段失败，优先检查这个 token。

### 使用 CLI 配置 Secrets

在已经登录 `gh` 的机器上执行：

```bash
gh secret set BOT_GITHUB_TOKEN --repo mantle-xyz/reth
gh secret set ANTHROPIC_API_KEY --repo mantle-xyz/reth
gh secret set ANTHROPIC_BASE_URL --repo mantle-xyz/reth
gh secret set LARK_WEBHOOK_URL --repo mantle-xyz/reth
```

命令会交互式要求输入 secret 值。

## 可选 GitHub Variable：Runner 选择

workflow 会读取以下 repository variable：

```text
OP_RETH_AUTO_BUMP_RUNNER
```

如果不配置，默认使用：

```text
ubuntu-latest
```

MVP 阶段可以先用 `ubuntu-latest`。

生产阶段建议改成 larger runner、Depot runner 或 self-hosted runner label：

```bash
gh variable set OP_RETH_AUTO_BUMP_RUNNER \
  --repo mantle-xyz/reth \
  --body ubuntu-latest
```

未来可替换成类似：

```text
depot-ubuntu-32
self-hosted
self-hosted-linux-rust-32c
```

这里的值必须是仓库或组织中真实可用的 runner label。

## 必需 Labels

workflow 会使用以下 PR labels：

```text
A-dependencies
needs-manual-migration
```

创建命令：

```bash
gh label create A-dependencies \
  --repo mantle-xyz/reth \
  --color 0366d6 \
  --description "Dependency update"

gh label create needs-manual-migration \
  --repo mantle-xyz/reth \
  --color d73a4a \
  --description "Requires manual migration"
```

如果 label 已存在，创建命令可能失败。这种情况可以忽略，但需要在 GitHub UI 中确认 label 已存在。

## 仓库文件前置检查

第一次运行前，确认目标分支包含以下文件和目录：

```text
.github/workflows/update-op-reth.yml
.op-reth-base-tag
Cargo.toml
Cargo.lock
op-reth/
mantle-reth/
patches/
```

本地检查：

```bash
cd /Users/whisker/Work/src/networks/mantle/reth

test -f .github/workflows/update-op-reth.yml
test -f .op-reth-base-tag
cat .op-reth-base-tag
```

`.op-reth-base-tag` 的格式必须是：

```text
op-reth/v2.2.1
```

要求：

- 只能包含 tag 名；
- 不要加引号；
- 不要加 `refs/tags/` 前缀；
- 文件中不要有其它说明文字。

## Workflow Inputs

workflow 通过 `workflow_dispatch` 手动触发。

输入参数：

| Input | 是否必填 | 默认值 | 示例 | 含义 |
|-------|----------|--------|------|------|
| `base_branch` | 是 | `mantle-elysium` | `mantle-elysium` | PR base branch，也是创建 `op-reth-auto-bump` 的基准分支 |
| `target_tag` | 否 | 空 | `op-reth/v2.3.0` | 要同步到的上游 Optimism `op-reth` tag |
| `dry_run` | 否 | `true` | `true` / `false` | 是否跳过 push 和 PR 创建/更新 |

### target_tag 规则

有效格式：

```text
op-reth/vX.Y.Z
```

示例：

```text
op-reth/v2.3.0
op-reth/v2.4.1
```

无效示例：

```text
v2.3.0
refs/tags/op-reth/v2.3.0
op-reth/v2.3.0-rc.1
```

如果 `target_tag` 留空，workflow 会自动检测上游最新稳定版 `op-reth/v*` tag。

为了保证测试可复现，建议测试时始终填写 `target_tag`。

## 查询可用的上游 op-reth Tags

列出所有稳定版 `op-reth/v*` tag：

```bash
gh api repos/ethereum-optimism/optimism/git/refs/tags --paginate \
  --jq '[.[].ref | ltrimstr("refs/tags/")
        | select(startswith("op-reth/v"))
        | select(test("-rc|-dev|-pr") | not)]
        | sort_by(ltrimstr("op-reth/v") | split(".") | map(tonumber))
        | .[]'
```

只显示最新稳定 tag：

```bash
gh api repos/ethereum-optimism/optimism/git/refs/tags --paginate \
  --jq '[.[].ref | ltrimstr("refs/tags/")
        | select(startswith("op-reth/v"))
        | select(test("-rc|-dev|-pr") | not)]
        | sort_by(ltrimstr("op-reth/v") | split(".") | map(tonumber))
        | last'
```

验证某个 tag 是否存在：

```bash
git ls-remote --tags https://github.com/ethereum-optimism/optimism.git \
  refs/tags/op-reth/v2.3.0
```

如果命令输出 commit SHA，说明 tag 存在。若没有输出或命令非零退出，不要使用该 tag。

## 首次配置流程

### 1. 确认 Workflow 已经在 GitHub 上

workflow 文件必须已经 commit 并 push：

```text
.github/workflows/update-op-reth.yml
```

推荐检查：

```bash
gh workflow list --repo mantle-xyz/reth
```

预期能看到：

```text
Update op-reth
```

如果 GitHub Actions UI 中看不到该 workflow，按顺序检查：

1. workflow 文件是否已经 push 到 GitHub。
2. 如果 GitHub UI 不能从 `mantle-elysium` 显示该 workflow，确认 workflow 文件是否也存在于仓库 default branch。
3. 仓库是否启用了 GitHub Actions。
4. workflow YAML 是否有效。

### 2. 配置 Secrets

配置：

```text
BOT_GITHUB_TOKEN
ANTHROPIC_API_KEY
ANTHROPIC_BASE_URL
LARK_WEBHOOK_URL
```

最少必须配置 `BOT_GITHUB_TOKEN`，否则无法真实 push 和创建 PR。

即使是 `dry_run=true`，workflow 也会用 `BOT_GITHUB_TOKEN` 做 checkout 和 `gh api` 调用，因此第一次 dry-run 前也要配置。

### 3. 配置 Runner Variable

MVP 阶段：

```bash
gh variable set OP_RETH_AUTO_BUMP_RUNNER \
  --repo mantle-xyz/reth \
  --body ubuntu-latest
```

如果接受默认 `ubuntu-latest`，这一步可以跳过。

### 4. 配置 Labels

确认以下 labels 存在：

```text
A-dependencies
needs-manual-migration
```

### 5. 确认 Base Tag

执行：

```bash
gh api repos/mantle-xyz/reth/contents/.op-reth-base-tag \
  --jq '.content' | base64 --decode
```

预期输出：

```text
op-reth/v2.2.1
```

如果值不正确，先通过普通 PR 修正 `.op-reth-base-tag`，再运行 auto-bump workflow。

## 通过 GitHub UI 手动运行

第一次 dry-run 推荐走 GitHub UI。

1. 打开 GitHub 仓库：

```text
https://github.com/mantle-xyz/reth
```

2. 进入：

```text
Actions -> Update op-reth
```

3. 点击：

```text
Run workflow
```

4. 选择 workflow branch：

```text
mantle-elysium
```

这个 branch 只表示“使用哪个分支上的 workflow 文件运行”。

5. 填写 inputs：

```text
base_branch = mantle-elysium
target_tag  = op-reth/v2.3.0
dry_run     = true
```

6. 点击：

```text
Run workflow
```

7. 打开 run，观察日志。

dry-run 的预期行为：

- 不会 push 到 `op-reth-auto-bump`；
- 不会创建或更新 PR；
- 日志中会打印生成的 PR body；
- 如果生成了 artifacts，会上传 artifacts；
- 如果配置了 Lark webhook，会发送 dry-run 通知。

## 通过 GitHub CLI 手动运行

推荐的 dry-run 命令：

```bash
gh workflow run update-op-reth.yml \
  --repo mantle-xyz/reth \
  --ref mantle-elysium \
  -f base_branch=mantle-elysium \
  -f target_tag=op-reth/v2.3.0 \
  -f dry_run=true
```

观察运行状态：

```bash
gh run watch --repo mantle-xyz/reth
```

查看最近 runs：

```bash
gh run list \
  --repo mantle-xyz/reth \
  --workflow update-op-reth.yml \
  --limit 10
```

查看失败日志：

```bash
gh run view --repo mantle-xyz/reth --log-failed
```

查看指定 run：

```bash
gh run view <RUN_ID> --repo mantle-xyz/reth
```

下载 artifacts：

```bash
gh run download <RUN_ID> \
  --repo mantle-xyz/reth \
  --name op-reth-auto-bump-artifacts \
  --dir /tmp/op-reth-auto-bump-artifacts
```

## Dry-Run 验收 Checklist

第一次运行必须使用：

```text
dry_run=true
```

以下检查全部通过，才可以进入 `dry_run=false`：

1. Checkout 成功。
2. workflow 成功读取 `.op-reth-base-tag`。
3. workflow 成功校验 `target_tag`。
4. 上游 `ethereum-optimism/optimism` clone 成功。
5. sparse checkout 包含：
   - `rust/op-reth/**`
   - `rust/Cargo.toml`
6. `op-reth/` sync 步骤完成。
7. reth rev 提取成功；如果失败，错误信息清晰可定位。
8. fork dependency report 正常生成。
9. validation section 正常生成。
10. 日志中出现：

```text
dry_run=true; skipping push and PR creation
```

11. 远端分支没有被更新。
12. PR 没有被创建或更新。
13. 如果 workflow 运行到 artifact 生成阶段，artifacts 正常上传。

如果任一检查失败，不要运行 `dry_run=false`。

## 正式创建 PR

正式运行时使用：

```text
dry_run=false
```

CLI 示例：

```bash
gh workflow run update-op-reth.yml \
  --repo mantle-xyz/reth \
  --ref mantle-elysium \
  -f base_branch=mantle-elysium \
  -f target_tag=op-reth/v2.3.0 \
  -f dry_run=false
```

预期行为：

1. workflow 创建或更新本地分支 `op-reth-auto-bump`。
2. workflow push 远端分支：

```text
origin/op-reth-auto-bump
```

3. workflow 创建或更新指向以下 base branch 的 PR：

```text
mantle-elysium
```

4. 如果验证通过：
   - PR 为 ready for review；
   - label `A-dependencies` 存在；
   - 旧的失败 label 被移除。
5. 如果验证失败：
   - PR 为 draft；
   - labels `A-dependencies` 和 `needs-manual-migration` 存在；
   - PR body 包含失败细节；
   - 可以通过 conflict report 或 artifacts 接手处理。

## PR Review Checklist

每个自动生成的 PR 都要检查：

1. PR title 包含 old tag 和 new tag。
2. PR base branch 正确：

```text
mantle-elysium
```

3. PR head branch 是：

```text
op-reth-auto-bump
```

4. PR body 包含：
   - upstream Optimism compare link；
   - reth core rev summary；
   - 如果 rev 发生变化，包含 reth rev compare link；
   - Mantle fork dependency report；
   - validation results；
   - 如有异常，包含 workflow warnings；
   - 如有冲突，包含 conflict sections。
5. `.op-reth-base-tag` 已更新为 `target_tag`。
6. `Cargo.toml` 中所有 reth rev 一致。
7. 如果 reth rev 变化，`Cargo.lock` 已更新。
8. `op-reth/` 有同步变更。
9. `mantle-reth/` 的变更仅限迁移修复。
10. 没有无关文件变更。

## 验证级别

### MVP Workflow 验证

当前 workflow 运行轻量级检查：

```bash
cargo check --workspace
cargo test --workspace --no-run
cargo test \
  -p reth-optimism-node \
  -p reth-optimism-evm \
  -p reth-optimism-consensus \
  -p mantle-reth-cli \
  --lib --all-features
cargo test -p mantle-reth-integration-tests --no-run
```

这组检查是为了适配标准 GitHub runner。

### 生产目标验证

当 larger runner 可用后，应将以下完整 targeted test gate 纳入 workflow：

```bash
cargo test \
  -p mantle-reth-chainspec \
  -p mantle-reth-rpc-ext \
  -p mantle-reth-eth-api \
  -p mantle-reth-cli \
  -p reth-optimism-consensus \
  -p mantle-reth-integration-tests
```

如果 runner 资源允许，也可以增加：

```bash
cargo clippy --workspace --all-targets --all-features -- -D warnings
```

在 `ubuntu-latest` 上不要贸然启用这些重检查。先确认运行时间和稳定性。

## 结果处理

### 结果：没有更新

日志显示当前 tag 已等于 latest tag 或指定的 `target_tag`。

处理：

1. 确认 `.op-reth-base-tag` 等于请求的目标 tag。
2. 不会创建 PR。
3. 无需进一步操作。

### 结果：Dry Run 通过

处理：

1. 检查 logs 和 artifacts。
2. 确认没有远端分支或 PR 被修改。
3. 只有准备好创建或更新 PR 时，才用 `dry_run=false` 重新运行。

### 结果：Ready PR 已创建

处理：

1. 通知维护者 review PR。
2. 等待普通 PR CI。
3. 不要自动 merge。
4. PR merge 后，确认 `mantle-elysium` 上的 `.op-reth-base-tag` 已等于新 tag。

### 结果：Draft PR 已创建

处理：

1. 打开 PR body。
2. 查看 validation failure summary。
3. 如果有 `docs/op-reth-auto-bump-conflicts.md`，打开检查。
4. 如需要，下载 artifacts。
5. 分配给负责迁移的工程师。
6. 在冲突和验证失败解决前，保留 `needs-manual-migration`。

### 结果：Workflow 在创建 PR 前失败

处理：

1. 打开 workflow logs。
2. 确认失败是否发生在 branch push 之前。
3. 如果有 artifacts，下载 artifacts。
4. 修复配置或 workflow 问题。
5. 重新使用 `dry_run=true` 运行。

dry run 通过前，不要直接重试 `dry_run=false`。

## Artifacts

workflow 会上传：

```text
op-reth-auto-bump-artifacts
```

可能包含：

| 文件 | 含义 |
|------|------|
| `patch-conflicts.txt` | 包含 merge conflict markers 的文件列表 |
| `manual-conflicts-info.txt` | 已自动处理、但建议 review 的结构性变化 |
| `manual-conflicts-unresolved.txt` | 需要人工决策的结构性冲突 |
| `compat-report.md` | Mantle fork dependency report |
| `pr-body.md` | workflow 生成的 PR body |
| `worktree.diff` | 剩余 worktree 状态的 binary-safe diff |

下载命令：

```bash
gh run download <RUN_ID> \
  --repo mantle-xyz/reth \
  --name op-reth-auto-bump-artifacts \
  --dir /tmp/op-reth-auto-bump-artifacts
```

## 回滚和清理

### 取消错误的 Run

```bash
gh run cancel <RUN_ID> --repo mantle-xyz/reth
```

### 关闭错误的 Draft PR

只有在确认维护者不再需要该分支状态后，才关闭 PR：

```bash
gh pr close <PR_NUMBER> \
  --repo mantle-xyz/reth \
  --comment "Closing invalid op-reth auto-bump run. A corrected run will be created separately."
```

### 删除自动化分支

只有在 PR 已关闭且无人继续使用该分支时，才删除远端分支：

```bash
git push origin --delete op-reth-auto-bump
```

### 干净重跑

清理后先 dry-run：

```bash
gh workflow run update-op-reth.yml \
  --repo mantle-xyz/reth \
  --ref mantle-elysium \
  -f base_branch=mantle-elysium \
  -f target_tag=<TARGET_TAG> \
  -f dry_run=true
```

dry-run 验收通过后，才运行 `dry_run=false`。

## 故障排查

### 看不到 Run Workflow 按钮

可能原因：

1. workflow 文件还没 push。
2. workflow 文件不在 default branch。
3. 仓库没有启用 GitHub Actions。
4. YAML 文件无效。

检查：

```bash
gh workflow list --repo mantle-xyz/reth
ruby -e 'require "yaml"; YAML.load_file(".github/workflows/update-op-reth.yml")'
```

### Checkout 失败

优先检查：

```text
BOT_GITHUB_TOKEN
```

处理：

1. 确认 secret 存在。
2. 确认 token 有仓库读取权限。
3. 确认 token 没有过期。
4. 修复后用 `dry_run=true` 重跑。

### gh api 失败

可能原因：

1. `GH_TOKEN` 缺失或无效。
2. GitHub API rate limit。
3. 仓库名错误。
4. GitHub 事故或网络故障。

检查：

```bash
gh auth status
gh api repos/ethereum-optimism/optimism/git/refs/tags --paginate --jq 'length'
```

### target_tag 被拒绝

有效格式：

```text
op-reth/vX.Y.Z
```

验证 tag 是否存在：

```bash
git ls-remote --tags https://github.com/ethereum-optimism/optimism.git \
  refs/tags/<TARGET_TAG>
```

除非 workflow 明确改成允许 pre-release，否则不要使用 rc/dev/pr tag。

### 没有可更新版本

说明：

```text
.op-reth-base-tag == target_tag
```

或者，当 `target_tag` 为空时：

```text
.op-reth-base-tag == latest stable upstream op-reth tag
```

处理：

1. 确认 `.op-reth-base-tag`。
2. 确认请求的 `target_tag`。
3. 不会创建 PR。

### Push 失败

可能原因：

1. bot token 无 push 权限。
2. branch protection 阻止 bot push 到 `op-reth-auto-bump`。
3. 另一个 workflow run 刚更新过该分支。
4. `--force-with-lease` 因远端分支变化被拒绝。

处理：

1. 检查 branch protection rules。
2. 确认当前只有一个 `update-op-reth` workflow 正在运行或排队。
3. 等其它 run 完成后重跑。
4. 如有必要，经维护者确认后关闭旧 PR 并删除旧分支。

### PR 创建失败

可能原因：

1. bot token 缺少 PR 权限。
2. labels 不存在。
3. 已有 PR 的 base branch 不符合预期。
4. 前面的 branch push 已经失败。

检查：

```bash
gh pr list \
  --repo mantle-xyz/reth \
  --head op-reth-auto-bump \
  --state all

gh label list --repo mantle-xyz/reth | grep -E 'A-dependencies|needs-manual-migration'
```

修复 labels 或 token 权限后，再重新运行。

### Claude Code 没有运行

可能原因：

1. 未配置 `ANTHROPIC_API_KEY`。
2. 未配置 `ANTHROPIC_BASE_URL`。
3. `npm install -g @anthropic-ai/claude-code` 失败。
4. API proxy 不可用。

预期行为：

- 即使 Claude 不可用，workflow 仍可创建 draft PR；
- 冲突和编译错误会留给人工处理。

处理：

1. 检查 setup-node 和 install logs。
2. 检查 Claude API/proxy 可用性。
3. 修复后用 `dry_run=true` 重跑。

### cargo update 失败

预期行为：

- workflow 记录 `lock_update_failed=true`；
- PR body 中包含 warning；
- 如果 `Cargo.lock` 不一致，后续 `cargo check` 通常会失败。

处理：

1. 查看 cargo 输出。
2. 如果依赖解析需要人工介入，分配给 Rust 工程师。
3. 在 lockfile 和 validation 修好前，PR 保持 draft。

### 仍有 Merge Conflict Markers

预期行为：

- PR 是 draft；
- 存在 `needs-manual-migration` label；
- conflict report 已提交，或 artifacts 已上传。

处理：

1. 打开 conflict report：

```text
docs/op-reth-auto-bump-conflicts.md
```

2. 检查 marker conflict 文件列表。
3. 在 PR 分支中解决 conflict markers。
4. 本地或通过 PR CI 重新运行验证。
5. 标记 PR ready 前，删除 conflict report，除非维护者希望保留它作为审计上下文。

### 仍有 Manual Structural Conflicts

manual structural conflicts 分两类：

- `manual-conflicts-info.txt`：workflow 已自动处理，但建议 review 的结构性变化，例如上游删除了 Mantle 修改过的文件、本地删除但上游修改后已采用上游版本；
- `manual-conflicts-unresolved.txt`：需要人工决策的结构性冲突，例如二进制文件两边都发生变化，或 `git merge-file` 运行错误。

处理：

1. 先读取 `manual-conflicts-unresolved.txt` artifact 或 PR conflict report，逐文件决定保留本地版本、采用上游版本，还是手动迁移。
2. 再读取 `manual-conflicts-info.txt` artifact 或 PR body，确认自动处理的结构性变化没有误删有效 Mantle 逻辑。
3. 将修复 push 到 `op-reth-auto-bump`。
4. 重新运行验证。

## Merge 后流程

PR merge 到 `mantle-elysium` 后：

1. 确认 `.op-reth-base-tag` 等于已合并的上游 tag：

```bash
gh api repos/mantle-xyz/reth/contents/.op-reth-base-tag \
  --ref mantle-elysium \
  --jq '.content' | base64 --decode
```

2. 确认没有残留打开的 auto-bump PR：

```bash
gh pr list \
  --repo mantle-xyz/reth \
  --head op-reth-auto-bump \
  --state open
```

3. 确认 `mantle-elysium` 的正常 CI 是 green。
4. 如果 PR 曾经是 draft/failure 状态，并由人工修复后 merge，额外确认：
   - `needs-manual-migration` 已移除；
   - conflict report 已删除，或明确保留；
   - artifact-only 的冲突现场已经不再需要。

## 启用 Cron

满足以下条件前，不要启用定时运行：

1. 至少一次 `dry_run=true` 通过。
2. 至少一次 `dry_run=false` 成功创建正确 PR。
3. 至少一次 failure-mode run 生成可接手的 draft PR 或 artifact bundle。
4. bot token 权限稳定。
5. Lark 通知已验证。
6. runner 资源足够。
7. 维护者已经明确 review 责任归属。

稳定后建议 schedule：

```yaml
schedule:
  - cron: '0 3 * * 1'
```

含义：每周一 03:00 UTC 运行。

启用 cron 时，保留 `workflow_dispatch`，不要移除手动触发入口。

## Operator Quick Reference

指定 tag 的 dry-run：

```bash
gh workflow run update-op-reth.yml \
  --repo mantle-xyz/reth \
  --ref mantle-elysium \
  -f base_branch=mantle-elysium \
  -f target_tag=op-reth/v2.3.0 \
  -f dry_run=true
```

指定 tag 的真实 PR run：

```bash
gh workflow run update-op-reth.yml \
  --repo mantle-xyz/reth \
  --ref mantle-elysium \
  -f base_branch=mantle-elysium \
  -f target_tag=op-reth/v2.3.0 \
  -f dry_run=false
```

自动检测最新 tag，并 dry-run：

```bash
gh workflow run update-op-reth.yml \
  --repo mantle-xyz/reth \
  --ref mantle-elysium \
  -f base_branch=mantle-elysium \
  -f dry_run=true
```

观察 run：

```bash
gh run watch --repo mantle-xyz/reth
```

下载 artifacts：

```bash
gh run download <RUN_ID> \
  --repo mantle-xyz/reth \
  --name op-reth-auto-bump-artifacts \
  --dir /tmp/op-reth-auto-bump-artifacts
```

列出 auto-bump PR：

```bash
gh pr list \
  --repo mantle-xyz/reth \
  --head op-reth-auto-bump \
  --state all
```
