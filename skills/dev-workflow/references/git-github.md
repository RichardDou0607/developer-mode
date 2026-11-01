# Git 与 GitHub 规范（git-github）

> 用途：阶段 6/7 的完整命令、分支/提交规范、PR 模板、CI 处理与安全门禁。SKILL §3 只保留摘要；进入阶段 6/7 前读本文件。
> 适用：git 仓库。非 git 工作区先按 SKILL 阶段 0 的结论处理（询问是否初始化，或跳过 Git/GitHub 阶段）。

## 1. Git 规范

分支：`feat/<slug>`、`fix/<slug>`、`test/<slug>`、`docs/<slug>`、`refactor/<slug>`；从默认分支切出、短生命周期，不直接提交 main/master。

Conventional Commits：`<type>(<scope>): <subject>`，≤72 字符、祈使句、不加句号；破坏性变更用 `!` 或 `BREAKING CHANGE:` 脚注。

| type | 用途 | 示例 |
| --- | --- | --- |
| feat | 新功能 | `feat(preset): add developer mode` |
| fix | 修 bug | `fix(skill): resolve skill dir on win32` |
| test | 测试 | `test(patch): assert preset row order` |
| docs | 文档 | `docs(readme): document uninstall` |
| refactor | 重构不改行为 | `refactor(tools): extract schema helper` |
| chore | 杂务/依赖 | `chore(deps): bump dsh to 0.2.1-rc.5` |

## 2. 提交前检查

- `git status --short` 只含本次文件，无陌生改动；用 `git add -p` 或明确路径 stage，避免 `git add -A` 夹带无关文件；
- `git diff` 自查逻辑与格式，`git diff --stat` 看规模是否合理；
- `git diff --cached --check` 无空白错误/冲突标记；>1MB 大文件人工判断；
- **密钥检查不得打印命中原文**。优先用 `gitleaks` / `detect-secrets`；没有时用下面只返回退出码、不输出内容的检查：
  ```bash
  if git diff --cached -U0 | grep -Eiq 'api[_-]?key|token|secret|password' \
     || git diff -U0 | grep -Eiq 'api[_-]?key|token|secret|password'; then
    echo 'WARN: diff 命中疑似密钥模式；请人工核对并轮换。不要打印/粘贴原文。'
  else
    echo 'OK: 未命中常见密钥模式（不代表绝对安全）'
  fi
  ```
  命中后只在私密环境查看具体文件/行号；已推送的密钥删掉不够，必须轮换。
- 一个逻辑改动一个提交，message 符合规范。

命令序列：

```bash
git status --short
git diff
git add -p              # 或 git add <明确路径>，避免夹带无关文件
git diff --cached --check
git diff --cached --stat
git commit -m "feat(scope): ..."
git log --oneline -5
```

`git stash`、`git reset --hard`、`git clean -fd`、`git rebase`、`git branch -D`、`git push --force` 可能丢改动或改写历史：执行前先输出「将执行什么 / 影响什么 / 如何回滚」，等用户明确确认；不为“让状态干净”而回滚用户未提交的改动。

## 3. GitHub 规范

### 3.1 账号与同意门禁

1. `gh auth status` 确认账号。**不要裸跑 `gh auth switch`**：本机有两个账号时它会自动切到另一个账号，导致 push/PR 开错账号。需要切换时先向用户确认具体账号，再用：
   ```bash
   gh auth switch --hostname github.com --user <account>
   ```
   切换会改变全局 active 账号，影响用户在其它仓库的工作；非必要不切换。
2. **普通 push 与创建 PR 都必须先获得用户明确同意**。用户只要求本地修改（“先别提交/别推”）时，跳过本阶段的远端写入。
3. 未经用户明确同意不做：force-push（仅本人分支且获准才用 `--force-with-lease`）、直接推 main/master、`gh pr merge`（含 `--admin`）、删分支、发 Release、改仓库设置/权限/密钥。

### 3.2 推送与 PR

```bash
git push -u origin <branch>
```

创建 PR（需要早期反馈可加 `--draft`；摘要里用 `Closes #123` 关联 issue）：

```bash
cat > /tmp/pr-body.md <<'EOF'
## 摘要
一句话说明改了什么、为什么；Closes #123

## 变更
- 文件/模块：要点（含刻意不做的非目标）
- 安全与兼容性影响：...

## 测试计划与结果
- [x] `pnpm test` — 42 passed
- [x] `layout_audit` — PASS（证据：.layout-guard/reports/<id>.json；未安装时写明）
- [ ] 未跑：真实安装验证（原因：需 Full access）

## 风险与回滚
- 风险：...
- 回滚：revert 本 PR / 卸载 bundle
EOF
gh pr create --title "feat(scope): ..." --body-file /tmp/pr-body.md
```

PR 正文是“四段 + 摘要内关联 issue”；不与最终交付报告（`references/delivery-template.md`）混用。

### 3.3 等 CI 与评审

- `gh pr checks --watch --fail-fast`（**退出码 8 = 仍有 pending，不是失败**；`--required` 只看必需检查）；失败用 `gh run view <id> --log-failed` 读日志，修复后追加提交。
- review：逐条回复（已改/不改+理由），改完 push 同一分支并 resolve thread。
- 不把被 skip 的 CI job 当成通过。

### 3.4 降级（无 git / 无远端 / 无权限 / 无 gh）

- **非 git 工作区**：阶段 0 先 `git rev-parse --show-toplevel` 探测；明确声明“当前不是 git 仓库”，询问用户是否在指定目录初始化，或跳过 Git/GitHub 阶段。禁止悄悄 `git init` 整个 workspace。
- **无远端/无权限/无网络/gh 未安装/gh 未登录**：说明具体原因，输出本地 commit hash 与交付说明；禁止伪造 PR 链接或 CI 状态。若只是 gh 缺失，可给出可复制的手工 PR 步骤，由用户自行执行。

## 4. Windows（pwsh）注意

本技能的命令按 bash 写。Windows 上 `tool-bash` 被禁用、`tool-pwsh` 启用，`head`、heredoc、`/tmp` 等不可用；按下面方式转换，或明确使用 Git Bash/WSL：

```powershell
git status --short
git diff --cached --check
git diff --stat
# PR 正文用 here-string 写临时文件
@'
## 摘要
...
'@ | Set-Content -Encoding utf8 pr-body.md
gh pr create --title "feat(scope): ..." --body-file pr-body.md
if ($LASTEXITCODE -ne 0) { Write-Error "gh pr create failed: $LASTEXITCODE" }
```

## 参考来源

- Conventional Commits 1.0.0 — https://www.conventionalcommits.org/en/v1.0.0/
- GitHub Docs: Status checks — https://docs.github.com/en/pull-requests/reference/status-checks
- GitHub Docs: GitHub Flow — https://docs.github.com/en/get-started/using-github/github-flow
- GitHub CLI manual: `gh pr checks` — https://cli.github.com/manual/gh_pr_checks
- GitHub CLI manual: `gh auth switch` — https://cli.github.com/manual/gh_auth_switch
- Google Engineering Practices: Small CLs — https://google.github.io/eng-practices/review/developer/small-cls.html
