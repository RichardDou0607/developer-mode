# 评审清单（review-checklist）

> 用途：两套清单——**A 提交前自审**（作者）、**B PR 评审**（评审者）。每条带默认判级：`[blocker]` 合并前必须修；`[major]` 应修或说明理由并留 issue；`[minor]` 建议修；`[nit]` 可选。级别随实际影响调整：可上调不可下调；任何 blocker 未清零就 Request changes。

## 0. 判级定义

| 级别 | 含义 | 典型例子 | 处理 |
| --- | --- | --- | --- |
| blocker | 正确性/安全/数据丢失/不可回滚等严重问题，或硬门禁失败 | 越权、注入、丢数据、测试造假、破坏性变更未版本化 | 必须修，阻断合并 |
| major | 明显缺陷或风险，影响正确性边缘/可维护性/性能/兼容性 | 无超时、吞异常、覆盖缺口、性能明显退化 | 本 PR 修，或说明理由 + 留 issue |
| minor | 局部可改进，不影响正确性 | 命名、重复代码、日志不完整 | 建议修，作者决定 |
| nit | 风格/措辞/个人偏好 | 空格、措辞 | 不阻断，可忽略 |

**评审原则**（Google Code Review Standard）：评审目标是「整体代码健康持续改善」，不要求完美；技术事实/数据优先于个人偏好；纯风格以仓库 style guide 为权威；纯偏好标 `nit`（对应 Google 的 Nit）。NIST/OWASP 类安全标准按风险裁剪，不必逐条打勾。

## A. 提交前自审（作者）

在 PR 描述里贴勾选结果；有未勾选项必须写明原因。

### A1 正确性与边界

- [ ] (blocker) 行为与需求/验收标准一致；边界（空/0/负数/超长/最大/重复/非法）有处理
- [ ] (major) 空值与缺省路径明确；返回值与异常路径都覆盖

### A2 错误处理与日志

- [ ] (blocker) 无空 catch/吞异常；错误带上下文与 cause
- [ ] (major) 日志分级合理、结构化、无密钥/PII；对外错误不泄露内部细节

### A3 并发与状态

- [ ] (blocker) 共享状态有同步/原子性保证；无竞态、死锁、丢失更新
- [ ] (major) 异步/定时/重试幂等；异常路径也释放资源

### A4 安全

- [ ] (blocker) 外部输入校验；SQL 参数化；命令不拼接；路径防穿越
- [ ] (blocker) 无硬编码密钥；权限最小；认证授权在服务端
- [ ] (major) 新增依赖/外部调用来源可信、无高危漏洞

### A5 性能与可靠性

- [ ] (major) 无 N+1/无界增长/无超时调用；关键路径有指标
- [ ] (minor) 复杂度与数据规模匹配，有上限保护

### A6 测试质量

- [ ] (blocker) 新行为/修复有测试；修 bug 的测试在改动前会失败
- [ ] (major) 覆盖边界与错误路径；断言具体；不依赖顺序/网络/真实时间
- [ ] (blocker) 无删除/skip/放宽断言/盲改快照；CI 门禁全绿
- [ ] (blocker) 门禁集合固定且全绿（build / unit / 关键集成 / typecheck / lint）；被 skip 的 CI job 不算通过

### A7 可读性与一致性

- [ ] (major) 命名/结构自解释，遵循仓库风格；无死代码/调试残留
- [ ] (minor) 无无关重构/格式化；diff 规模与改动匹配（~100 行合理，>1000 行通常过大）

### A8 兼容性与文档

- [ ] (blocker) 无未版本化的破坏性变更；数据迁移可回滚
- [ ] (major) 公开接口/配置/运维文档与 changelog 同步
- [ ] (minor) 关键取舍有 ADR 或注释说明
- [ ] (minor) CHANGELOG/发布说明已更新，版本号与变更类型（SemVer）匹配

## B. PR 评审（评审者）

### B0 流程

- 先读 PR 描述与测试计划，再看 diff；必要时本地跑测试或实际验证。
- 评审顺序（Google looking-for）：先整体设计（位置是否合适、是否与系统整合、是否现在做、是否过度设计），再功能/复杂度/测试/命名/注释/风格/一致性/文档，最后逐行。
- 逐条给出级别、影响与可操作建议；一条意见只讲一件事，避免纯否定。
- 结论：有 blocker → Request changes；只有 major/minor → 可 Approve，但写明哪些必须本 PR 修、哪些可另开 issue。
- 作者改完 push 后复核相关项；不重复已讨论内容，也不静默 dismiss 未解决问题。

### B1 正确性

- [ ] (blocker) 实现与 issue/需求一致；关键边界有测试或显式说明
- [ ] (major) 错误路径行为合理（不吞错、不泄露、可恢复）

### B2 设计

- [ ] (major) 复用既有能力；无重复机制；接口合理、依赖方向正确
- [ ] (minor) 模块边界清晰；没有为单场景过度抽象
- [ ] (major) 位置与时机合理：该放这里吗、是否与系统整合、是否在解决当前问题而非猜测的未来

### B3 安全

- [ ] (blocker) 输入验证、注入、权限、密钥、敏感数据无问题
- [ ] (major) 依赖/CI/构建链变更可信
- [ ] (blocker) 访问控制默认拒绝、对象归属校验；输入/注入/编码对照 OWASP A01/A03/A05 与适用 ASVS 条目

### B4 性能与可靠性

- [ ] (major) 无性能隐患（N+1/无界/阻塞）；超时与重试合理
- [ ] (minor) 观测性足够（指标/日志/告警）

### B5 测试

- [ ] (blocker) 测试覆盖改动行为与回归；无作弊（删/skip/放宽断言/伪造输出）
- [ ] (major) 测试风格与仓库一致；命名与断言清晰可维护

### B6 可读性

- [ ] (minor) 命名/注释/结构易读；无无关改动
- [ ] (nit) 风格偏好

### B7 兼容性与文档

- [ ] (blocker) 破坏性变更已版本化并有迁移/回滚
- [ ] (major) 文档/changelog 更新
- [ ] (major) 版本号与变更类型匹配（SemVer）、弃用流程与迁移说明齐全

### B8 证据

- [ ] (blocker) PR 正文有真实测试证据（命令 + 退出码 + 结果），未跑项如实标注
- [ ] (major) 安装/部署/回滚步骤可复现

### B9 依赖与供应链

- [ ] (major) 新增/升级依赖有理由与可信来源，lockfile 一致，许可与漏洞已评估（NIST SSDF PS/RV）

### B10 评审收尾

- [ ] (blocker) 每条意见已回复/解决；CI 全绿；可合并条件确认
- [ ] (major) 破坏性/数据迁移/权限变更已确认备份与回滚

## 1. 评审意见模板

```text
[blocker] src/foo.ts:42 — 用户输入被拼进 shell 命令，存在命令注入。
影响：任意命令执行。
建议：改用 execFile + argv 数组，并补一个注入用例测试。
```

规则：级别 + 文件:行 + 问题 + 影响 + 建议；同一条只讲一件事。

## 2. 快速判级表

| 影响类型 | 默认级别 |
| --- | --- |
| 数据丢失/损坏、越权、注入、密钥泄露 | blocker |
| 正确性缺陷、破坏兼容、硬门禁失败、测试作弊 | blocker |
| 无测试覆盖、无超时、明显性能退化、资源泄漏 | major |
| 可观测性不足、日志不规范、文档缺失 | major/minor |
| 命名/结构/注释/重复代码 | minor |
| 格式/措辞/个人偏好 | nit |

## 参考来源（2026-10-31 核验）

- Google Engineering Practices: The Standard of Code Review — https://google.github.io/eng-practices/review/reviewer/standard.html
- Google Engineering Practices: What to look for in a code review — https://google.github.io/eng-practices/review/reviewer/looking-for.html
- Google Engineering Practices: Small CLs — https://google.github.io/eng-practices/review/developer/small-cls.html
- Conventional Commits 1.0.0 — https://www.conventionalcommits.org/en/v1.0.0/
- GitHub Docs: Status checks（被 skip 的 job 报 Success）— https://docs.github.com/en/pull-requests/reference/status-checks
- OWASP Top 10:2025 / ASVS 5.0 — https://top10.owasp.org/2025/ 、https://asvs.owasp.org/ ；NIST SSDF — https://csrc.nist.gov/projects/ssdf
