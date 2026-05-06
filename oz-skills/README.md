# Oz Skills 阅读笔记

本文整理 [warpdotdev/oz-skills](https://github.com/warpdotdev/oz-skills) 相比之前 Warp 仓库内置 skills 的新增内容，并按旧的流程图笔记方式拆分到多个 Markdown 文件。

## 总体变化

`oz-skills` 从 Warp 内部仓库的 `.agents/skills` 走向一个公开可复用目录：每个 skill 都是 `.agents/skills/<name>/SKILL.md`，通过 YAML frontmatter 描述触发条件，再用 Markdown 描述工作流、检查项、脚本和参考资料。

相比之前整理过的 Warp skills，主要变化有三类：

| 类型 | 变化 | 代表 skills |
|---|---|---|
| 旧工作流通用化 | PR、CI、Issue dedupe 从 Warp 仓库约定抽象成通用 GitHub/`gh` 工作流 | `create-pull-request`, `ci-fix`, `github-issue-dedupe` |
| 新增领域工作流 | 覆盖数据分析、文档同步、MCP、调度、SEO/AEO、Web 质量、Terraform 等通用工程场景 | `analysis-artifacts`, `docs-update`, `mcp-builder`, `scheduler`, `seo-aeo-audit`, `web-accessibility-audit`, `web-performance-audit`, `webapp-testing`, `terraform-style-check` |
| 新增辅助资产 | 部分 skill 带 `references/`、`scripts/`、`examples/` 和 license，说明它们不只是提示词，而是可复制的工作流包 | `mcp-builder`, `seo-aeo-audit`, `web-accessibility-audit`, `webapp-testing` |

## 新增内容清单

| Skill | 相比旧 Warp skills 的增量 |
|---|---|
| `analysis-artifacts` | 新增 BigQuery 深度分析产物规范：分析目录、SQL、Python 可视化、README、TL;DR 和 query 反链。 |
| `dbt-model-index` | 新增 dbt 模型索引模板，用于把模糊数据问题映射到 BigQuery 表、grain、join key 和标准过滤。 |
| `docs-update` | 新增文档机器人：从近期代码提交判断是否需要更新用户文档，并支持 dry-run、执行模式和跨仓库文档。 |
| `mcp-builder` | 新增 MCP server 开发工作流，覆盖协议设计、TypeScript/Python SDK、工具设计、测试和评估问题集。 |
| `scheduler` | 新增本机提醒/轻量任务调度工作流，明确不用于云端 agent jobs 或 Oz-managed workflows。 |
| `seo-aeo-audit` | 新增 SEO 与 AEO/AI visibility 审计，包括 crawlability、结构化数据、移动端、AI 引用优化和脚本。 |
| `slack-qa-investigate` | 新增只读调查模式：回答仓库问题时强制查证、引用文件行号、禁止改文件和副作用命令。 |
| `terraform-style-check` | 新增 Terraform HCL 生成/审查规范，按 HashiCorp 风格组织文件、命名、安全和验证。 |
| `web-accessibility-audit` | 新增 WCAG 可访问性审计，覆盖自动测试、代码 grep、键盘/读屏人工测试和报告格式。 |
| `web-performance-audit` | 新增 Chrome DevTools MCP 性能审计，围绕 Core Web Vitals、网络链路和代码库优化。 |
| `webapp-testing` | 新增 Playwright 本地 Web 应用测试流程，强调 server helper、networkidle、先侦察再操作。 |
| `create-pull-request` | 旧 `create-pr` 的通用化：增加 gh 安装/认证、相关 skill 前置检查、标题生成、draft 决策和浏览器核验。 |
| `ci-fix` | 旧 CI 诊断/修错流程的通用化：直接定位 GitHub Actions 失败、抓日志、最小修复并推送 fix branch。 |
| `github-bug-report-triage` | 旧 issue triage 的 bug-only 版本：按 issue template 判断是否 actionable，并给出缺失信息反馈。 |
| `github-issue-dedupe` | 旧 dedupe 流程的通用化：多策略搜索、候选逐个比对，只在 90%+ 把握时评论。 |

## 流程图入口

[skills-flowcharts/README.md](skills-flowcharts/README.md)

分类文件：

- [代码仓库、GitHub 与文档工作流](skills-flowcharts/01-repo-github-and-docs.md)
- [数据分析与数仓索引](skills-flowcharts/02-data-analysis.md)
- [Web 测试、性能、可访问性与 SEO/AEO](skills-flowcharts/03-web-quality.md)
- [基础设施与 MCP 集成](skills-flowcharts/04-infra-and-mcp.md)
- [本机调度与自动化](skills-flowcharts/05-scheduling.md)

## 阅读建议

- 想看和旧 Warp PR/CI/Issue 流程的差异，先读 `01-repo-github-and-docs.md`。
- 想看 Oz skills 扩展到哪些新工程场景，读 `02` 到 `05`。
- 想快速查新增项，优先看上面的“新增内容清单”。
