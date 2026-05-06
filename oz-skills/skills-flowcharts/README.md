# Oz Skills 流程图索引

本文档把 [warpdotdev/oz-skills](https://github.com/warpdotdev/oz-skills) 的 15 个公开 Agent Skills 按工程工作流拆分。每个分类文件保留该分类下 skill 的 ASCII 流程图，并在解释中链接到原始 `SKILL.md`。

## 分类文件

| 分类 | 文件 | Skills |
|---|---|---|
| 代码仓库、GitHub 与文档工作流 | [01-repo-github-and-docs.md](01-repo-github-and-docs.md) | `create-pull-request`, `ci-fix`, `docs-update`, `github-bug-report-triage`, `github-issue-dedupe`, `slack-qa-investigate` |
| 数据分析与数仓索引 | [02-data-analysis.md](02-data-analysis.md) | `analysis-artifacts`, `dbt-model-index` |
| Web 测试、性能、可访问性与 SEO/AEO | [03-web-quality.md](03-web-quality.md) | `webapp-testing`, `web-performance-audit`, `web-accessibility-audit`, `seo-aeo-audit` |
| 基础设施与 MCP 集成 | [04-infra-and-mcp.md](04-infra-and-mcp.md) | `terraform-style-check`, `mcp-builder` |
| 本机调度与自动化 | [05-scheduling.md](05-scheduling.md) | `scheduler` |

## 和旧 Warp Skills 的关系

- `create-pull-request`、`ci-fix`、`github-issue-dedupe` 是旧 `create-pr`、`diagnose-ci-failures`/`fix-errors`、`dedupe-issue-local` 的通用化版本。
- `github-bug-report-triage` 和旧 `triage-issue-local` 相近，但范围收窄为 bug report 是否 actionable。
- 其他 skill 基本都是新增领域：数据分析、dbt、文档同步、MCP、调度、SEO/AEO、Web 质量、Terraform、只读问答。

## 阅读建议

- 准备发 PR 或修 CI：读 `01-repo-github-and-docs.md`。
- 做数据分析或写 BigQuery：读 `02-data-analysis.md`。
- 验证 Web 应用质量：读 `03-web-quality.md`。
- 写 Terraform 或 MCP server：读 `04-infra-and-mcp.md`。
- 做本机提醒或轻量任务：读 `05-scheduling.md`。
