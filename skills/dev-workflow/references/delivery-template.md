# 交付报告模板（delivery-template）

> 用途：阶段 8 交付时按本模板写最终回复。PR 正文用 `references/git-github.md` §3 的模板（四段 + 摘要内关联 issue，面向评审者与 CI）；本模板面向用户与下一任维护者。**没有运行过的验证不得写成通过。**

## 1. 完整模板

```markdown
## 做了什么
- 目标：
- 结果：完成 / 部分完成（缺什么、为什么）

## 为什么
- 关键设计选择与复用决策（复用了什么、为什么新建；ADR 链接）

## 如何验证（真实执行）
| 检查 | 命令 | 结果 |
| --- | --- | --- |
| 烟测 | `<command>` | exit=0；主流程 OK；错误路径返回 ... |
| 回归 | `<command>` | 128 passed / 0 failed；基线 122 → 128，无删改 |
| 静态检查 | `<format/lint/typecheck>` | exit=0 |
| UI（如适用） | `layout_audit` | PASS（报告路径） |
| 安装/部署（如适用） | `plugin_manager list_plugins` | preset-developer enabled |
| CI（如适用） | `gh pr checks` | 全绿 / 链接 |

## 改动文件
- `path`：一句话作用（新增/修改/删除）

## 风险与后续
- 风险：
- 回滚：
- 后续建议：

## 复盘
- 失败最多的步骤 / 缺失的检查 / 写回 preset、skill 或测试的改进项

## 未做 / 未验证
- 如实列出原因与预期（缺服务/缺凭据/平台不符/需用户操作）
```

## 2. 证据规范

- 每条验证写「命令 + 退出码 + 关键输出」；不能只写「已验证 / 测试通过」。
- 输出过长只贴关键行（通过数、关键字段、错误码）；完整日志可按需另存 artifacts。
- UI 给 `layout_audit` 的 PASS/FAIL 与报告路径；安装给 `plugin_manager` 的 application/warnings 摘要。
- 无法运行或未覆盖的项必须写原因与影响，不许省略、含糊或事后补写。
- 交付前先按 `review-checklist.md` 的 A 段自审完毕，再写报告；报告里的命令与证据必须与实际执行一致。

## 3. 精简交付（小任务）

固定三行：**做了什么 / 验证命令与结果 / 风险或未验证项**。其余从简，但证据要求不降级，仍不得虚构验证。

## 4. 与 PR 正文的区别

| 场景 | 用哪个模板 | 受众 |
| --- | --- | --- |
| PR 正文 | `references/git-github.md` §3（摘要含关联 issue/变更/测试计划与结果/风险回滚） | 评审者 / CI |
| 最终交付回复 | 本文件完整模板 | 用户 / 下一任维护者 |

## 5. 反模式

- 只写「完成了」没有证据；把计划要做的事写成已完成。
- 隐瞒未跑项或把失败描述成「环境问题」而不给证据。
- 改动文件清单与实际 diff 不一致。
- 风险写「无」，但实际存在兼容/迁移/性能风险。

## 参考来源（2026-10-31 核验）

- Claude Code Best Practices：要求证据（运行了什么命令、输出是什么，而非「All tests pass」）— https://code.claude.com/docs/en/best-practices
- GitHub Docs: Status checks（被 skip 的 job 报 Success，不能充当通过）— https://docs.github.com/en/pull-requests/reference/status-checks
- Google Engineering Practices: Small CLs（自包含改动、含相关测试）— https://google.github.io/eng-practices/review/developer/small-cls.html
