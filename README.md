# dsh-developer-mode · 开发者模式（v0.2.1 安全/可移植性修正）

给 DeepSeek Harness 增加一个名为「开发者模式」的 Agent 预设。它不是又一个工具包，而是一套**可验证的工程化开发流程**：装进预设的 persona 硬门禁 + 随包携带的 `dev-workflow` 技能，要求 Agent 在写代码前先探索与复用、实现后先烟测再回归、提交前自查、推送前获得用户明确同意，并且只用真实执行过的命令输出作为“验证通过”的证据。

> **v0.2.1（2026-11-01）**：修正测试发现的 P1/P2 风险——① 把 Git/GitHub 完整规范拆到 `references/git-github.md`，让 skill 的**渲染结果**低于 DSH tool-result-pruner 的 8,192 字符阈值（旧版 8,256 会被裁剪掉 §4–§7）；② UI 验证改为“`layout_audit` 可用则必须用，不可用则说明并请求安装，禁止伪造 PASS”；③ 普通 push/PR 也要求用户明确同意；④ 密钥检查不再打印命中原文；⑤ 禁止裸 `gh auth switch`（双账号会自动切换）；⑥ 非 git 工作区先探测/询问；⑦ 0–8 门禁只用于代码/多步任务；⑧ skill 目录 fallback 改用 `baseUrl` 绝对路径；⑨ 增加 `engines.dsh` 版本门禁、LICENSE 与 `./cordis.patch.yml` export；⑩ 回归口径改为“相对基线不新增失败”。安装/卸载方式不变。

## 工作流概览

预设把一次开发任务组织为 8 个阶段，每阶段都有进入条件、必做项与退出证据（只读/琐碎任务可精简，见 SKILL 顶部“适用范围”）：

| 阶段 | 做什么 | 关键产出 |
| --- | --- | --- |
| 0 接入 | 复述目标/验收/边界；按规模建 todo；读 AGENTS.md、README、构建与测试脚本；**先探测是否 git 仓库** | 需求复述 + 仓库现状（含是否 git） |
| 1 探索与复用 | glob/grep 找同类实现与测试，读调用点；DSH 插件先 cordis_inspect_list/query | 复用清单（复用/扩展/新建 + 理由） |
| 2 计划 | 文件级改动点、验收标准、烟测+回归计划、风险与回滚 | 计划（大改先找用户确认） |
| 3 开发 | 小步、最小 diff、遵循仓库风格，每步跑 lint/typecheck/相关测试；git 仓库才开短分支 | 无遗留失败的增量改动 |
| 4 烟测 | 主流程 + 1 个错误路径；UI 用 layout_audit/assert（工具可用时）；插件用 plugin_manager 核对 | 「命令 + 关键输出」证据 |
| 5 回归 | 跑全量测试与检查、补新测试、对比基线（相对基线不新增失败） | 通过/失败/跳过数 |
| 6 Git | status/diff 自查、单逻辑单提交、Conventional Commits；完整规范见 `references/git-github.md` | 提交 hash + 规范 message |
| 7 GitHub | **用户明确同意后** gh auth status、push 分支、gh pr create、gh pr checks --watch --fail-fast | PR URL + CI 状态 |
| 8 交付 | 做了什么/为什么/如何验证/改动文件/风险后续，按 references/delivery-template.md 写 | 带真实命令与结果的最终回复 |

## 按需加载的 references（v0.2.1：5 个）

v0.2 把长规范做成**按需加载的 references**，避免把长规范塞进始终生效的 persona/SKILL；v0.2.1 把 Git/GitHub 也拆出去，控制 skill 渲染结果的体积：

| 文件 | 内容 | 何时读 |
| --- | --- | --- |
| `skills/dev-workflow/references/git-github.md` | 分支/Conventional Commits、提交前检查（含不打印密钥原文的检查）、PR 模板、gh 账号与 push 同意门禁、CI、无 git/无远端/无 gh 降级、Windows/pwsh 转换 | 阶段 6/7 |
| `skills/dev-workflow/references/engineering-standards.md` | 10 个维度（设计/编码/错误处理与日志/可观测/安全/性能与可靠性/兼容性与版本/测试与质量门禁/文档与 ADR/依赖供应链）的最小必做与反模式 | 阶段 2/3 写代码前 |
| `skills/dev-workflow/references/review-checklist.md` | 提交前自审 + PR 评审两套可勾选清单，按正确性/边界/错误处理/并发/安全/性能/测试质量/可读性/兼容性/文档分维度，含 blocker/major/minor/nit 判级 | 提交前与评审时 |
| `skills/dev-workflow/references/architecture-design.md` | 设计流程（需求→约束→≥2 候选→权衡→ADR→接口先行）、深模块、依赖方向、错误策略、迁移与回滚设计、ADR 模板与自检 | 新模块/改接口/迁移前 |
| `skills/dev-workflow/references/delivery-template.md` | 交付报告模板与证据规范（命令 + 退出码 + 关键输出）；PR 正文模板见 `git-github.md` §3 | 阶段 8 交付时 |

SKILL §5 给出 9 条可勾选的工程规范自检；细节按需读取 references，不自动加载、不占常驻上下文。

## 安装与被选中

安装（二选一）：

- 让 Agent 用 `plugin_manager` 的 `install_bundle` 动作安装本目录（`target` 传本目录绝对路径）；
- 或在 DSH Web「设置 → 插件」里安装本目录/本 bundle。

> 安装/卸载/升级会改 profile：先说明影响并获用户明确同意，必要时备份；安装后核对 `application` 与 `list_plugins` 行状态，失败时回滚。

安装后：

1. `plugin_manager` `list_bundles` 应能看到 `dsh-developer-mode`（v0.2.1）；
2. `list_plugins` 中 `preset-developer` 行应处于 enabled 状态（激活失败会带 diagnostic 留在 roster 上，但无法建会话）；
3. **新建会话** → 打开模式选择器 → 选中「开发者模式」；
4. 想让它成为所有新会话的默认模式：在 Settings 的模式/预设设置里把默认项设为「开发者模式」（已有会话不受影响）。

## 包含文件

| 文件 | 作用 |
| --- | --- |
| `package.json` | bundle 元数据；`dsh.bundle.patch` 指向 patch；`engines.dsh` 版本门禁；`exports`/`files` 声明 locale、icon、skills 与 LICENSE |
| `cordis.patch.yml` | 通过 `insert:` 插入 `@deepseek-ai/dsh-agent-preset` 行（preset id=`developer`，order=5）：persona 硬门禁 + 工具面 + 分组（planning/compaction/delegation）+ 技能根解析 |
| `skills/dev-workflow/SKILL.md` | `dev-workflow` 技能正文：阶段门禁、探索复用、测试策略、Git/GitHub 摘要、工程规范自检、DSH 插件专项、反模式 |
| `skills/dev-workflow/references/`（5 个文件） | 按需加载参考：`git-github.md`、`engineering-standards.md`、`review-checklist.md`、`architecture-design.md`、`delivery-template.md` |
| `locale/zh.json`、`locale/en.json` | bundle 展示标题与描述（中文/英文） |
| `icon.svg` | 原创 64×64 图标：终端提示符 + Git 分支，深色底浅色线条 |
| `LICENSE` | MIT |
| `README.md` | 本文件 |

## 验证清单

安装后可按顺序核对：

- [ ] `plugin_manager` `list_bundles` 出现 `dsh-developer-mode`，版本 `0.2.1`；
- [ ] `list_plugins` 中 `preset-developer` 行 enabled，`fiberPhase` 正常、无 diagnostic；
- [ ] 新建会话时模式选择器出现「开发者模式」，选中后新建成功；
- [ ] 新会话技能目录包含 `dev-workflow`（可以让 Agent 调用 `skill` 工具加载它）；
- [ ] `skills/dev-workflow/references/` 下 5 个文件存在且非空；
- [ ] 加载 `dev-workflow` 后能按相对路径读取 `references/git-github.md` 与 `references/review-checklist.md`；
- [ ] skill 渲染结果（含 `<skill_resources>` 前言与 base dir）不超过 pruner 阈值（默认 8,192）；可用 `scripts/prune_probe.mjs` 复跑；
- [ ] 新会话中 `tool-cordis`、`tool-plugin-manager` 等行按预期启用（后者受 profileContext 守卫）；
- [ ] 既有 preset（standard/ptc/minimal/cordis 等）在模式选择器中仍可选择、既有插件行状态未变；
- [ ]（可选）`python3` 解析 `cordis.patch.yml`：顶层为 `insert` 行、preset `id` 为 `developer`、`order` 为 `5`。

## 卸载

- 用 `plugin_manager` 的 `remove_bundle`，`target` 传 `dsh-developer-mode`；
- 或在 Web「设置 → 插件」里移除本 bundle。

移除后模式选择器不再出现「开发者模式」；已有会话保持创建时的预设（DSH 语义如此，不会随卸载切换）。

## 已知限制与前置条件

- **预设只影响新建会话**：已有会话与它们的子会话保持创建时的插件版本与组合，切换模式需要新建会话；
- **默认模式是用户设置**：安装不会自动改掉现有默认预设，需要在 Settings 里手动设为「开发者模式」；
- **UI 验证依赖可选 bundle**：`layout_audit/layout_assert/layout_snapshot/layout_diff` 来自独立的 `dsh-layout-guard`，不在本 preset 内。未安装时 persona/SKILL 要求 Agent 明确说明并请求安装，禁止伪造 PASS；本机已装时可用；
- **技能体积与 pruner**：DSH 本身对 skill 正文没有大小上限，但 preset 的 `tool-result-pruner` 默认在压缩时裁剪超过 8,192 字符的工具结果。SKILL 正文已压到渲染结果 ~7,840 字符（当前路径）；安装路径更长或后续扩写时，请用 `scripts/prune_probe.mjs` 复跑并保持余量。README/旧审计里的“8000/2000 上限”是项目自设预算，不是 DSH 契约；
- **技能根解析**：`skill-filesystem` 通过 `createRequire(baseUrl)` 从已安装包解析 `skills/` 目录；失败时用 `baseUrl` 拼出绝对路径，不再依赖 `DSH_PROFILE_DIR`（该变量只注入 Agent shell，Host 进程可能没有）。手工拷贝目录而不经包安装时仍可能找不到技能；
- **push/PR 需明确同意**：persona 与 SKILL 都要求普通 push、创建 PR 前获得用户同意；用户只要本地修改时跳过阶段 7；
- **gh 前置**：阶段 7 需要 `gh` 已安装并登录；多账号时不要裸跑 `gh auth switch`（双账号会自动切到另一个），需要切换时先确认账号。gh 缺失/未登录时输出本地 commit hash + 手工步骤，禁止伪造 PR；
- **非 git 工作区**：阶段 0 先探测；非 git 时询问用户是否初始化或跳过 Git/GitHub 阶段，禁止悄悄 `git init`；
- **Windows**：技能命令按 bash 写；Windows 上 `tool-pwsh` 生效，`head`/heredoc/`/tmp` 不可用，按 `references/git-github.md` 的 pwsh 转换或使用 Git Bash/WSL；
- **外部技能可选**：persona 提到的 `tdd`/`code-review`/`diagnosing-bugs`/`setup-pre-commit`/`pr` 不随本 bundle 分发；缺失时跳过并在交付说明，不要卡住；
- **plugin_manager 前置**：需要 profileContext 与 danger-full-access/审批；headless/SDK 或低权限会话不可用时只做静态核对并说明；
- **流程是约束不是保证**：persona 与技能提高遵循概率，不替代人工 review；高风险/破坏性操作仍要求用户确认；
- **references 按需加载**：5 个 reference 不会自动进入上下文，SKILL 要求按需读取；若模型跳过读取，规范只剩 SKILL 里的摘要；
- **规范是默认 baseline**：仓库既有规范（AGENTS.md、lint 配置、团队约定）优先，冲突时按仓库规范并在 PR 注明；
- **`active` 工具面**：本预设未包含 `tool-schedule` 与 `tool-plugin-market` 行（如需要可自行在 profile 层补充；README 已声明）；
- **重复安装**：HMR 下 `install_bundle` 对已安装包返回 `already-installed`，不会新增行；不要手工改 profile 或重复插入 bundle 行（会产生第二条 `preset-developer` Loader 行，第二条 preset 注册会被 registry 拒绝）。
