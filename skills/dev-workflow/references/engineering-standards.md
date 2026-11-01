# 工程规范 baseline（engineering-standards）

> 用途：`dev-workflow` 技能的按需参考。阶段 2（计划）与阶段 3（开发）在写代码前读对应维度；交付前对照 `review-checklist.md`。本文件给「最小必做」与「反模式」，是默认 baseline；仓库已有规范（AGENTS.md、lint 配置、团队约定）优先，冲突时按仓库规范执行并在 PR 注明差异。

## 0. 使用方式

- 「最小必做」= 交付底线；未满足要在 PR/交付报告说明理由与后续处理。
- 不要求一次全做到：改动涉及哪一维读哪一节；不确定是否适用时按保守解释执行。
- 工具能强制的交给工具（formatter / lint / typecheck / CI），规范只写工具查不到的部分。
- 风险裁剪：一次性脚本/小型内部工具可按「不适用」裁剪，但要在计划或 PR 写明理由（NIST SSDF 明确它是可裁剪的出发点，不是逐项照做的清单）。

## 1. 设计

**最小必做**

- 写代码前说清模块边界与数据流：谁调用谁、输入输出、失败如何表示；跨模块改动先定接口再实现。
- 新增模块单一职责、接口清晰；公共 API 尽量小，内部实现可替换。
- 复用优先：先找既有能力（见 SKILL §1），不为一个场景造第二套机制。
- 设计决策落 ADR（小改动写进计划/PR），记录背景、备选、取舍与后果。
- 引入新依赖/新机制时能说清「不这样做会怎样」。
- 先写可测量的质量目标（性能/可靠性/安全/可维护性）；接口契约写明输入、输出、错误、副作用、并发假设与兼容承诺（SemVer 要求先声明公共 API）。

**反模式**

- 先写代码再补设计，接口在实现中反复变形。
- 万能工具类、巨型模块、万能基类；靠继承复用行为。
- 循环依赖、双向依赖、跨层直接调用（UI 直接读 DB、业务直接拼 SQL）。
- 为「以后可能用到」提前抽象。

## 2. 编码

**最小必做**

- 命名自解释：变量/函数说明「是什么/做什么」；布尔用 is/has/should；单位与量纲进名字（`timeoutMs`、`sizeBytes`）。
- 函数短小、参数少、早返回减少嵌套；嵌套 >3 层就提取；单文件行数遵循仓库习惯。
- 遵循仓库的格式化与 lint 配置并在提交前运行；不夹带无关格式化。
- 注释解释「为什么」而非「是什么」；复杂逻辑写清约束与边界条件。
- 不留死代码、调试输出、无主 TODO（保留的必须带 owner 与 issue）。

**反模式**

- 魔法数字/魔法字符串硬编码；复制粘贴重复代码。
- 无意义缩写、拼音命名、`data`/`info`/`temp` 之类空泛名。
- 注释掉的代码、`console.log`/`print`/`debugger` 残留。
- 重命名/重构与功能改动混在同一提交。

## 3. 错误处理与日志

**最小必做**

- 错误显式处理：能恢复就恢复，不能则上抛或转成用户可读错误；禁止空 catch / 吞异常。
- 错误信息带上下文：操作、关键参数（脱敏）、错误码/原因链；保留 cause/stack。
- 区分错误类型：用户输入错误（可提示、不重试）、依赖失败（可重试/降级）、程序缺陷（fail fast 并上报）。
- 日志分级（error/warn/info/debug），生产默认 info 以上；结构化字段（时间、级别、请求/追踪 id、耗时）。
- 日志与错误信息不含密钥、token、密码、身份证等敏感信息；必要时脱敏。
- 对外错误响应不暴露内部实现/堆栈；对外错误码稳定可依赖。
- 日志写 stdout 事件流、由运行环境收集与轮转；应用不自己管理日志文件（12-Factor Logs）。
- 安全事件（登录失败、权限拒绝、校验失败、配置变更）单独记录并带可追踪 ID，便于审计（OWASP Top 10 A09）。

**反模式**

- `catch (e) {}` / `except: pass`；catch 后返回默认值掩盖问题。
- 错误信息只有 "error"/"failed"，没有上下文与具体对象。
- 用日志代替指标/告警；在循环里刷大量 info 日志。
- 把异常/用户输入拼进 SQL、HTML、命令（注入风险）。

## 4. 可观测性

**最小必做**

- 关键路径有指标（请求量/耗时/错误率）与日志；跨服务/跨步骤有 trace 或 correlation id 贯穿。
- 新功能上线前定义「怎么知道它坏了」：至少一个可查指标或日志 + 告警阈值。
- 外部依赖调用埋点：耗时、失败率、重试次数、超时次数。
- 日志/指标标签基数可控（不把 user id、原始 URL、请求参数当标签）。
- 日志/指标/追踪三件套：traceId 贯穿请求，关键路径有指标与告警（OWASP A09：安全日志与告警失败是独立风险）。

**反模式**

- 只有用户报障才发现问题；没有错误率/延迟观测。
- 打点过多、维度爆炸、成本失控。
- 日志格式随意、无法聚合查询。

## 5. 安全

**最小必做**

- 所有外部输入（HTTP/CLI/文件/环境/第三方回调）都校验：类型、长度、范围、格式；校验在边界层完成。
- 最小权限：命令/文件/网络/数据库权限按需授予；不把管理员凭据放在默认路径。
- 密钥不写进代码/仓库/日志；用环境变量或密钥管理；已泄露的必须轮换。
- 密钥扫描/检查不得把命中原文写入日志、终端或模型上下文；只输出命中位置/数量，命中后人工核对并轮换（`gitleaks`/`detect-secrets` 或 `grep -Eiq` 只取退出码）。
- 防注入：SQL 参数化、命令用 argv 数组而非 shell 拼接、HTML 转义、路径防穿越、反序列化不信任输入。
- 认证/授权在服务端做；前端隐藏按钮不等于权限控制。
- 依赖与镜像来源可信、无已知高危漏洞（见 §10）。
- 访问控制默认拒绝：每个受保护资源都在服务端校验，对象 ID 校验归属（OWASP A01）；配置缺必需项时 fail fast（12-Factor Config）。
- 不自己发明加密/哈希；传输用 TLS，敏感数据存储加密（OWASP A04 Cryptographic Failures）。

**反模式**

- 「内网所以不校验」；用户输入直接拼进命令/查询/路径。
- 把 token 放 URL/日志/错误信息里；硬编码默认密码。
- 关闭 TLS/证书校验；给自己留后门接口。
- 把权限判断放在前端或只做 UI 隐藏。

## 6. 性能与可靠性

**最小必做**

- 复杂度与数据规模：注意循环内 IO/查询（N+1）、无界集合/缓存/队列增长；大列表分页或流式处理。
- IO 设超时与上限；网络/数据库调用设置 timeout；重试有限次、退避、只重试幂等操作。
- 资源释放：文件/连接/锁在 finally 或等价机制中关闭；并发用有界池。
- 缓存说明失效策略；缓存不得改变正确性语义（可旁路）。
- 高影响改动给出量级估算或基准数据（复杂度、QPS、延迟）。
- 可靠性：评估单点故障、降级路径与失败隔离（一个租户/请求不拖垮全局）。
- 先测量再优化：性能结论必须带命令、样本数与环境；优雅关闭（收到 SIGTERM 停止接新请求、完成在途、释放资源）；发布可回滚且 release 不可变（12-Factor Build/release/run）。

**反模式**

- 请求处理中同步跑长时间任务；无超时的 await/网络调用。
- 无界缓存、无界并发、无限重试、重试无退避。
- 用 sleep 等异步完成；用「应该很快」当性能论证。
- 无数据支撑的过早优化，牺牲可读性。

## 7. 兼容性与版本

**最小必做**

- 公开 API/CLI/配置/数据格式/事件 schema 的变更视为兼容性变更：尽量只加不改、不删不换语义。
- 破坏性变更必须版本化（SemVer major）、写迁移指南、给弃用窗口（deprecation + 警告）。
- 数据迁移用可回滚方案（expand-contract，见 `architecture-design.md` §5），不一步删旧。
- 配置/序列化格式有版本字段或向后兼容解析（未知字段忽略、缺省值明确）。
- 发布顺序与回滚顺序写清（先发代码还是先迁数据）。
- 已发布版本（tag/release）不可变，修问题发新版本；废弃先发 minor 弃用并至少保留一个 minor，再在 major 移除（SemVer 规则）。
- CHANGELOG 按 Keep a Changelog 更新：`Unreleased` 累积，按 Added/Changed/Deprecated/Removed/Fixed/Security 分组，日期 ISO 8601，破坏性/弃用/移除必写。

**反模式**

- 悄悄改字段含义/类型；直接删接口/枚举值/配置项。
- 不可逆数据迁移、无备份的批量 update。
- 用「内部使用」当理由不做兼容。
- 把 breaking change 混进 patch 版本发布。

## 8. 测试与质量门禁

**最小必做**

- 新行为/修 bug 有对应测试；覆盖正常路径、边界（空/最大/重复/非法）与错误路径。
- 单测快而独立：不依赖网络/真实时间/共享状态；用 fake、clock、in-memory 实现。
- 集成测试覆盖序列化/DB/HTTP/队列等边界。
- 质量门禁全绿：format → lint → typecheck → unit → integration → build；CI 上不 skip。
- 测试名称描述行为，断言具体；避免只断言「不抛错」。
- 修 bug 先写复现测试（红 → 绿 → 重构）。
- 门禁集合固定：build + unit + 关键集成 + typecheck + lint，PR 前全绿；被 skip 的检查不算通过（GitHub status checks 语义）。

**反模式**

- 只测 happy path；为覆盖率写无断言测试。
- 测试依赖执行顺序/共享全局状态；用 sleep 等异步。
- 删测试/放宽断言/skip 换绿；改快照不审查差异。
- 只跑本地单测，不跑 CI 同款命令。

## 9. 文档与 ADR

**最小必做**

- 公开接口/配置/环境变量/运维步骤有文档或注释；README 的安装/运行/测试命令保持可用。
- 关键取舍与架构变化有 ADR（模板见 `architecture-design.md` §6）；ADR 放仓库既有位置或 `docs/adr/`，编号递增、状态明确。
- 用户/运维可见的变更更新 changelog 或迁移说明。
- 文档与代码同一次改动更新，不留跑不通的旧命令。

**反模式**

- 只改代码不改文档；README 命令早已过期。
- 注释复述代码；过期注释误导。
- 决策只留在聊天记录里，不落盘。

## 10. 依赖与供应链

**最小必做**

- 新增依赖先评估：必要性、维护状态、license、体积、安全记录；能用标准库/既有依赖就不新增。
- 锁文件提交并保持一致；CI 用 lockfile 安装（`npm ci` / `pnpm --frozen-lockfile`）。
- 依赖审计：高危漏洞必须处理或记录接受理由；升级单独提交并跑全量测试。
- 版本范围保守（精确或受限范围），不盲目追 `latest`；生产构建可复现。
- 关注构建/发布链路安全（CI 权限、action 固定版本、产物校验），不执行不可信脚本。
- 供应链四件事：来源可信 + 锁定版本 + 记录理由 + 漏洞响应（NIST SSDF PS/RV 组）；CI 跑 SCA，高危阻断；构建产物记录 provenance/checksum（SSDF PS.3.2）。

**反模式**

- 为一个小功能引入大依赖；依赖废弃/来源不明的包。
- 删 lockfile 重装；依赖升级与功能改动混提交。
- 忽略 audit 高危；使用未 pin 的第三方 action。
- 从不可信来源下载并执行脚本或二进制。

## 11. 最小交付自检卡

- [ ] 设计/接口变更已说明，必要时有 ADR
- [ ] 命名/结构遵循仓库风格，无死代码/调试残留
- [ ] 错误显式处理、日志分级且无敏感信息
- [ ] 外部输入校验、权限最小、密钥安全
- [ ] 超时/重试/资源释放/复杂度有考虑
- [ ] 兼容性评估与迁移/回滚方案
- [ ] 测试覆盖正常/边界/错误路径，门禁全绿
- [ ] 文档/ADR/changelog 同步更新
- [ ] 依赖新增经评估、lockfile 一致、audit 通过
- [ ] 配置从环境注入、缺必需配置 fail fast；密钥不进仓库/日志
- [ ] CHANGELOG/发布说明已更新，版本号与变更类型匹配

## 参考来源（2026-10-31 核验）

- Google Engineering Practices：Small CLs / Code Review Standard / What to look for / Style Guides — https://google.github.io/eng-practices/
- Semantic Versioning 2.0.0 — https://semver.org/ ；Keep a Changelog 1.1.0 — https://keepachangelog.com/en/1.1.0/
- The Twelve-Factor App（Config / Dependencies / Build-release-run / Logs）— https://12factor.net/
- OWASP Top 10:2025 — https://top10.owasp.org/2025/ ；OWASP ASVS 5.0 — https://asvs.owasp.org/ ；OWASP Logging Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html ；Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- NIST SSDF（SP 800-218）— https://csrc.nist.gov/projects/ssdf
- ADR — https://adr.github.io/ ；ADR 模板 — https://adr.github.io/adr-templates/
- 说明：Google SRE Book 与 ISO/IEC 25010 本次未能核验，性能/可靠性相关条目为工程建议。
