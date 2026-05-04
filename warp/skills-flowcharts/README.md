# Warp Skills 流程图索引

本文档把 Warp 本地 skills 按工程工作流拆成多个文件，避免单个 Markdown 过长。每个分类文件保留该分类下所有 skill 的详细 ASCII 流程图，并在解释中链接到 Warp 官方仓库里的原始 `SKILL.md`。

## 分类文件

| 分类 | 文件 | Skills |
|---|---|---|
| 规格驱动与实现规划 | [01-specs-and-implementation.md](01-specs-and-implementation.md) | `spec-driven-implementation`, `write-product-spec`, `write-tech-spec`, `implement-specs` |
| 功能开关与发布推进 | [02-feature-flags-and-rollout.md](02-feature-flags-and-rollout.md) | `add-feature-flag`, `promote-feature`, `remove-feature-flag` |
| 遥测与产品信号 | [03-telemetry.md](03-telemetry.md) | `add-telemetry` |
| PR、CI、错误修复与冲突处理 | [04-pr-ci-errors-and-conflicts.md](04-pr-ci-errors-and-conflicts.md) | `create-pr`, `diagnose-ci-failures`, `fix-errors`, `resolve-merge-conflicts` |
| 测试体系 | [05-testing.md](05-testing.md) | `rust-unit-tests`, `warp-integration-test` |
| 代码审查与 Issue 运营 | [06-review-and-issues.md](06-review-and-issues.md) | `review-pr`, `review-pr-local`, `triage-issue-local`, `dedupe-issue-local` |
| UI 规范与 Skill 维护 | [07-ui-and-skill-maintenance.md](07-ui-and-skill-maintenance.md) | `warp-ui-guidelines`, `update-skill` |

## 阅读建议

- 做较大功能：先读 `01-specs-and-implementation.md`，再按功能是否 gated 读 `02-feature-flags-and-rollout.md`。
- 准备 PR 或修 CI：读 `04-pr-ci-errors-and-conflicts.md`。
- 需要补验证：读 `05-testing.md`。
- 做 review、triage、dedupe：读 `06-review-and-issues.md`。
- 改 UI 或维护 skill：读 `07-ui-and-skill-maintenance.md`。
