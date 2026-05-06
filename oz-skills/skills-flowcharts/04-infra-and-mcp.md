# 四、基础设施与 MCP 集成

这一组新增了两类偏平台工程的 skills：Terraform HCL 风格/安全规范，以及 MCP server 从设计到评估的完整开发流程。


### `terraform-style-check`

用途：生成、审查和维护符合 HashiCorp 官方风格的 Terraform HCL。

```text
+------------------------------------------------------------+
| 触发：用户要求写、review 或生成 Terraform 配置。              |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 按依赖顺序生成                                            |
| - provider config and version constraints                    |
| - data sources before dependent resources                    |
| - resources in dependency order                              |
| - outputs for key attributes                                 |
| - variables for configurable values                          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 文件组织                                                  |
| - terraform.tf：Terraform/provider versions                  |
| - providers.tf：provider configs                             |
| - main.tf：primary resources/data sources                    |
| - variables.tf：inputs，alphabetical                         |
| - outputs.tf：outputs，alphabetical                          |
| - locals.tf：local values                                    |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. HCL 风格                                                  |
| - terraform fmt alignment                                    |
| - logical block organization                                 |
| - consistent naming                                          |
| - variables/outputs 有 description、type、sensitive/validation |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 动态资源选择                                              |
| - 多实例优先 for_each，使用稳定 key。                        |
| - 条件创建可用 count。                                       |
| - 避免 count index 导致资源地址漂移。                        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 安全与版本                                                |
| - least privilege。                                          |
| - 加密、private access、public access block。                 |
| - pin Terraform/provider versions。                          |
| - provider alias 用于多区域/多账号。                         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. 验证                                                      |
| - terraform fmt                                              |
| - terraform validate                                         |
| - tflint / checkov / tfsec 等按项目可用性运行。              |
| - review checklist 覆盖 naming、安全、outputs、version。     |
+------------------------------------------------------------+
```

解释：原 skill：[terraform-style-check](https://github.com/warpdotdev/oz-skills/blob/main/.agents/skills/terraform-style-check/SKILL.md)。

要点：
- 这是新增的 IaC 风格规范，不只是格式化：它把文件边界、变量/输出语义、安全默认值和动态资源策略都固化下来。
- `for_each` 优先于 `count` 是重要架构取舍，因为稳定 key 能减少资源地址漂移和误删风险。
- 容易踩的坑：变量无 type/description、outputs 泄露敏感值、provider 版本不 pin、或为了多实例滥用 `count.index`。

### `mcp-builder`

用途：构建高质量 MCP server，让 LLM 通过设计良好的 tools 访问外部服务。

```text
+------------------------------------------------------------+
| 触发：用户要求创建 MCP server，集成 API/service，使用 Python  |
|       FastMCP 或 Node/TypeScript MCP SDK。                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| Phase 1: Deep research and planning                         |
| - 理解 MCP 现代设计：server boundaries、tool quality。        |
| - 读取 MCP best practices。                                  |
| - TypeScript 推荐；Python 可选。                             |
| - 研究目标 API docs、auth、data models。                     |
| - 规划 endpoints/tools 覆盖范围。                            |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| Phase 2: Implementation                                      |
| - 建项目结构。                                               |
| - 实现 core infrastructure：auth、client、config、errors。    |
| - 实现 tools：命名清晰、输入 schema、输出 LLM-friendly。      |
| - remote 优先 Streamable HTTP/stateless JSON；local 可 stdio。|
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| Phase 3: Review and test                                     |
| - code quality：类型、lint、错误处理、日志。                  |
| - build/test。                                               |
| - 用真实或 mock API 验证 tools 能完成任务。                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| Phase 4: Evaluations                                         |
| - 创建 10 个现实复杂问题。                                   |
| - 测试 LLM 是否能用 MCP server 完成任务。                    |
| - 每个 evaluation 包含 prompt、expected behavior 和 scoring。 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| Reference loading                                            |
| - mcp_best_practices.md 先读。                               |
| - node_mcp_server.md / python_mcp_server.md 在实现阶段读取。 |
| - evaluation.md 在评估阶段读取。                             |
+------------------------------------------------------------+
```

解释：原 skill：[mcp-builder](https://github.com/warpdotdev/oz-skills/blob/main/.agents/skills/mcp-builder/SKILL.md)。

要点：
- 新增的不只是 MCP 代码模板，而是“工具是否真的让 LLM 完成现实任务”的质量标准。
- 它明确推荐 TypeScript 和 remote Streamable HTTP/stateless JSON，原因是类型/lint 生态和部署扩展性更好。
- 容易踩的坑：把 REST endpoint 一比一暴露成低层工具、输出不适合 LLM 使用、没处理 auth/error、或没有 evaluation 就宣称 server 可用。
