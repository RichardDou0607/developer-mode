---
name: dev-workflow
description: 开发者工作流：做功能开发、修 bug、重构、补测试，或要 Git 提交/推送/创建 GitHub PR 时使用。给出 0–8 阶段门禁、探索复用、烟测与回归证据规范、软件工程规范自检（设计/编码/错误处理/安全/性能/兼容性/文档/评审）、Conventional Commits、gh PR 流程与 DSH 插件开发专项。
---

# 开发者工作流（dev-workflow）

把「先弄懂 → 再复用 → 小步写 → 烟测 → 回归 → Git → GitHub → 交付」跑成带证据的流程。persona 里的阶段门禁是硬约束；本文件给命令、清单与模板。

> **适用范围**：0–8 门禁用于代码改动或多步任务；只读审计、单行修改、纯问答可只建 1 条 todo 或跳过 todo，不要为琐碎任务制造 9 条空待办。plan mode 下不要用 `todo_write` 跟踪计划（计划属于 `exit_plan_mode`）。

## 0. 阶段门禁表

| 阶段 | 进入条件 | 必做 | 退出证据（缺了不继续） |
| --- | --- | --- | --- |
| 0 接入 | 收到任务 | 复述目标/非目标/完成定义；按规模建 todo（代码改动 0–8；只读/琐碎可单条或跳过）；读 AGENTS.md、README、构建测试脚本，先探测 git 仓库；查 gh 账号与不可逆风险 | 任务卡 + todo + 仓库现状（含是否 git） |
| 1 探索复用 | 阶段 0 完成 | glob/grep 找同类实现与测试，读调用点；DSH 先 cordis_inspect | 复用清单（复用/扩展/新建 + 理由） |
| 2 计划 | 复用结论成立 | 文件级改动点、验收标准、工程规范适用项、烟测+回归计划、风险回滚、拆自包含小步 | 计划确认（或明确无需确认） |
| 3 开发 | 计划确认 | 短分支（git 仓库；非 git 先按阶段 0 处理）；小步最小 diff；改逻辑配测试；接口契约/错误/日志/安全自检；每步 lint/typecheck/相关测试 | 每步检查通过，无遗留失败 |
| 4 烟测 | 实现可运行 | 主流程 + 1 个错误路径；UI 用 layout_audit/assert（工具可用时）；插件用 plugin_manager 核对 | 「命令 + 退出码 + 关键输出」 |
| 5 回归 | 烟测通过 | 全量测试与检查；补新测试；对照基线（相对基线不新增失败） | 通过/失败/跳过数 + 基线差异 |
| 6 Git | 回归通过 | status/diff/--check 自查；单逻辑单提交；Conventional Commits（完整规范见 `references/git-github.md`） | 提交 hash + 提交信息 |
| 7 GitHub | 用户明确同意推送 | gh auth status；push 分支；gh pr create；等 CI（完整规范见 `references/git-github.md`） | PR URL + checks 状态 |
| 8 交付 | 上述完成 | 按交付模板汇报；复盘一条流程改进 | 最终回复含命令与结果 |

任务卡 = 目标 / 非目标 / 完成定义 / 仓库与分支 / 不可逆风险 / 待确认项。阶段 6/7 的命令、PR 模板、CI 与降级见 `references/git-github.md`；非 git 仓库先按阶段 0 探测。阶段命令速查（`<...>` 换实际值；Windows 用 pwsh 时见 `references/git-github.md` 的转换说明）：

```bash
pwd; git rev-parse --show-toplevel 2>/dev/null || echo 'NOT-A-GIT-REPO'; git status --short --branch; gh auth status   # 0 接入
rg -n "<keyword>" -g '!node_modules' -g '!dist' | head -40   # 1 探索
git switch -c feat/<scope>-<slug>; <related-test>; echo "exit=$?"   # 3 开发（仅 git 仓库）
git add -p; git diff --cached --check; git diff --cached --stat   # 6 提交
```

## 1. 探索与复用（写实现前必须完成）

检索顺序：

1. `glob` 定位候选：`glob("**/*<keyword>*")`、`glob("src/**/*.ts")`，先看目录布局与命名。
2. `grep` 找符号与用法：`grep("<SymbolName>", include="*.ts")`，找定义、导出面、调用点、测试。
3. 读入口与注册点：package.json scripts/exports、index/main、路由表、DI 注册、客户端入口。
4. 读测试反推契约：测试是行为最快的快照，先看同名测试与 fixture；再读调用点确认参数来源、错误处理、副作用与并发约定。

DSH 专项（改 Harness 之前）：

- `cordis_inspect_list` 找到 Host/Client provider，再用 `cordis_inspect_query` 查 Config schema、Service 方法签名、Slot 树与 props、Tool schema、主题 token；不要猜 API。
- 读目标包 README 与 `lib/types/*.d.ts`；Desktop 里包在 `app.asar`，shell 打不开，用文件读取工具读。
- 已有 bundle 的 `cordis.patch.yml` 是最真实的用法样例；改既有行前先读它的 patch 上下文。

产出「复用清单」：

| 候选 | 位置 | 结论 | 理由 |
| --- | --- | --- | --- |
| `foo/bar.ts` 的工具函数 | 工具函数 | 直接复用 | 签名匹配、无副作用 |
| `FooService` | Service | 扩展现有 | 加一个可选参数即可 |

结论只有三选一：直接复用 / 扩展现有 / 新建并给出为什么现有方案不可用。

## 2. 测试策略

**烟测（smoke）**：证明主链路真的通，成本低（目标 <5 分钟），开发后立即跑。

- 最小集合：构建/安装 + 启动/加载 + 1~3 条关键路径 + 1 个错误路径。
- 证据格式：`[Smoke] cmd=... exit=0；关键路径 <结果>`，不能只写“已验证”。
- UI 改动：`layout_audit`（配 `layout_assert` 断言几何/物理不变量），以真实 Chromium 实测 PASS/FAIL 为准；若工具不可用（未安装 `dsh-layout-guard`），明确说明并请求安装，禁止凭源码想象布局或伪造 PASS。
- DSH 插件：真实安装后核对 `plugin_manager` 的 application/warnings、`list_plugins` 行状态与 slot 渲染。

**回归（regression）**：证明没弄坏别处，提交前必跑。

- 全量命令按仓库实际：`npm test`、`pnpm -r test`、`cargo test`、`go test ./...`，另跑 typecheck/lint/build 与关键 e2e。
- 报告「通过 / 失败 / 跳过」数量，并与改动前基线对比：**相对基线不新增失败；测试数不减少，除非删除的是过期测试并在 commit/PR 说明原因、关联 issue**。基线对比可用临时 worktree 在旧提交上复跑（见上方命令）。
- 为本次改动补回归测试：覆盖修复的 bug 或新增行为；能先红后绿就先红后绿。

测试金字塔：大量单测 → 适量集成测 → 少量端到端烟测；并发/时序类用明确条件等待，不用固定 sleep。

TDD 适用边界：新功能、可稳定描述的行为、bug 复现，先写失败测试（修 bug 先写复现测试）；探索性原型、一次性脚本、纯视觉微调不强行 TDD，改用烟测/截图证据。

失败处理：先复现并定位根因，再改最小处；本次引入就修，测试过期要单独提交说明。**flake**：同一测试重跑 1 次，连续两次结果不同才算 flake，把证据写进风险段，不静默跳过。确实无法运行（缺服务/缺凭据/平台不符）如实报告原因与预期。

**反作弊硬规则**：禁止为“变绿”删除测试、`.skip`/`.only`、注释或放宽断言、盲改快照/基线、`git commit --no-verify`、临时关 lint/typecheck、改 CI 让检查不跑；过期测试按上一条单独说明。CI 里被 skip 的 job 会报 Success，不能拿 skip 冒充通过。

## 3. Git 规范（摘要，详见 `references/git-github.md`）

- 分支：`feat/<slug>`、`fix/<slug>`、`test/<slug>`、`docs/<slug>`、`refactor/<slug>`；从默认分支切出、短生命周期，不直接提交 main/master。
- Conventional Commits：`<type>(<scope>): <subject>`，≤72 字符、祈使句、不加句号；完整 type 表与示例见 reference。
- 提交前：`git status --short` 只含本次文件；`git diff` / `git diff --stat` / `git diff --cached --check` 自查；一个逻辑改动一个提交。
- **密钥检查不得打印命中原文**：用 `gitleaks`/`detect-secrets`，或只用 `grep -Eiq` 返回退出码并输出“命中/未命中”（示例见 reference）。命中后人工核对并轮换，不要把值粘进上下文。
- `git stash`、`reset --hard`、`clean -fd`、`rebase`、`branch -D`、`push --force` 可能丢改动/改写历史：先说明影响与回滚，等用户明确确认。

## 4. GitHub 规范（摘要，详见 `references/git-github.md`）

- `gh auth status` 确认账号；**不要裸跑 `gh auth switch`**（两个账号时会自动切到另一个）。需要切换时先向用户确认具体账号，再用 `gh auth switch --hostname <host> --user <account>`。
- **普通 push 与创建 PR 也必须先获得用户明确同意**；用户只要本地修改时跳过本阶段。
- 推送后 `gh pr create`（完整四段模板见 reference）；等 CI 用 `gh pr checks --watch --fail-fast`（退出码 8 = 仍有 pending，不是失败）。
- 无远端/无权限/无网络/gh 未安装/未登录：输出本地 commit hash 与说明，禁止伪造 PR 或 CI 状态。
- 未经用户明确同意不做：force-push、直接推 main/master、`gh pr merge`（含 `--admin`）、删分支、发 Release、改仓库设置/权限/密钥。

## 5. 工程规范

阶段 3/5 按规范写与自检，references 按需读取：`references/engineering-standards.md`（baseline）、`references/architecture-design.md`（架构/ADR）、`references/review-checklist.md`（提交与评审）、`references/delivery-template.md`（交付报告）；Git/GitHub 见 `references/git-github.md`。进入阶段 6 前逐项勾选：

- [ ] 设计/接口变更已说明，必要时有 ADR；复用优先
- [ ] 命名与结构自解释、遵循仓库风格；无死代码/调试残留
- [ ] 错误显式处理（无空 catch），错误带上下文；日志分级、无密钥/PII
- [ ] 边界与并发：空值/超长/超时/重试/幂等/资源释放都有处理
- [ ] 安全：输入校验、最小权限、密钥安全、防注入；新增依赖经评估
- [ ] 性能与可靠性：无 N+1/无界增长/无超时调用；关键路径可观测
- [ ] 兼容性：公开 API/配置/格式变更已版本化，迁移与回滚可执行
- [ ] 测试：正常/边界/错误路径覆盖；不删不 skip；质量门禁全绿
- [ ] 文档与 ADR/changelog 同步；评审按 review-checklist 自审且 blocker 清零

## 6. 交付报告

按 `references/delivery-template.md` 写交付报告（做了什么/为什么/如何验证/改动文件/风险回滚/复盘/未做项）。没运行过的验证不得写成通过。

## 7. DSH 插件开发专项

- bundle 结构：`package.json` + `cordis.patch.yml`（可加 `locale/*.json`、`icon.svg`、`skills/`）。`dsh.bundle.patch` 指向 patch；preset 是 patch 里 `insert:` 的一条 `@deepseek-ai/dsh-agent-preset` 行。
- 子行解析：预设内子插件 `name` 是包说明符，按安装锚点优先、profile 兜底解析；包内资源用 `process.getBuiltinModule('node:module').createRequire(baseUrl).resolve('<pkg>/package.json')` 定位，禁止写死本机绝对路径。
- 安装：`plugin_manager` `action: install_bundle`，`target` 传 bundle 目录绝对路径；先说明影响、获用户明确同意并备份 → 安装 → 核对 → 必要时回滚；不要手工 `pnpm add` 或改 profile 文件（profile 属高风险）。
- 安装结果判读：看 `application` 字段（applied / warnings / nextStep），有 nextStep 说明需重启或后续动作；`list_plugins` 确认 preset 行 enabled、fiberPhase 正常（激活失败会带 diagnostic 留住 roster 但无法建会话）。
- 写代码前：`cordis_inspect_list` → `cordis_inspect_query` 查 Config/Service/Slots/Tool schema；读包 `lib/types`；改 UI 先看 client slot 契约。Inspect 验证相当于插件的集成测试：Service/Event/Config/Slot/Tool 都要查到真实注册结果。
- UI 验证：若 `layout_audit` 可用（需单独安装 `dsh-layout-guard`）则用它（可加 `layout_assert` 断言不溢出、零重叠、包含在容器内），以真实渲染结果为准；不可用时说明“未安装布局验证工具”并请求安装，禁止伪造 PASS。几何回归可用 `layout_snapshot` / `layout_diff`。
- 仓库前提：workspace 根可能不是 git 仓库，阶段 0 要显式声明；需要 Git 时在用户指定目录初始化，不悄悄 `git init` 整个 workspace。
- 环境降级：`plugin_manager` 需要 profileContext 与 danger-full-access/审批；headless/低权限不可用时只做静态核对并说明。`gh` 未安装/未登录时输出本地 commit hash 与手工 PR 步骤，禁止伪造 PR。
- Windows：命令按 bash 写；Windows 上 `tool-pwsh` 生效，按 `references/git-github.md` 的 pwsh 转换或使用 Git Bash/WSL。
- 回归：安装前后对比 `list_plugins`/`list_bundles`，确认既有 preset（standard/ptc/minimal/cordis 等）行仍在、既有插件 enabled 状态未变；已有会话保持原 plugin revision，验证必须新建会话。

## 8. 反模式清单

- 不探索直接写：重复实现已有函数，造出一套并行机制。
- 越过证据下结论：没跑命令就说“已验证/应该没问题”。
- 掩盖失败：try/catch 吞错、注释掉断言、skip/删测试、放宽阈值、`--no-verify` 或关检查。
- 大提交：功能 + 重构 + 格式化混在一个 commit。
- 破坏性操作不确认：force-push、reset --hard、rebase 公共分支、`gh pr merge --admin`、改仓库设置；普通 push/PR 未获同意也属于越权远端写入。
- 密钥/大文件/生成物入库（已推送的密钥未轮换）；密钥检查把命中原文打印进上下文。
- 多账号时裸跑 `gh auth switch`，导致 push/PR 用错账号。
- 只跑单测不跑烟测（流程不通），或只跑烟测不跑回归（弄坏别处）。
- 顺手重构无关代码扩大 diff；把“计划要做的”写成“已完成的”。
