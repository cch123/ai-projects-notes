# 六、代码审查与 Issue 运营

用于 review PR、叠加 Warp 本地审查经验，以及对公开 issue 做 triage 和 dedupe。


### `review-pr`

用途：基于 `pr_diff.txt` 和 `pr_description.txt` 审查 PR，并写出 `review.json`。

```text
+------------------------------------------------------------+
| 触发：工作流要求 review 当前 PR，并提供本地 artifacts：      |
| - pr_diff.txt                                                |
| - pr_description.txt                                         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 读取输入                                                  |
| - pr_description.txt：理解 PR 目的、范围、测试说明。         |
| - pr_diff.txt：只审查 PR 改动。                              |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 理解 diff line annotation                                |
| - [OLD:n]       -> side LEFT, line n                         |
| - [NEW:n]       -> side RIGHT, line n                        |
| - [OLD:n,NEW:m] -> unchanged context, use RIGHT line m        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：问题是否应 inline comment？                            |
|                                                            |
| inline comment，如果：                                       |
| - 问题位于 PR changed lines。                                |
| - 能给出具体文件和行号。                                     |
| - 行范围 <= 10 lines。                                       |
|                                                            |
| summary-only，如果：                                         |
| - 问题在 untouched code。                                    |
| - 是整体风险、测试缺口或架构疑问。                           |
| - 无法合法定位到 changed line。                              |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：severity label                                         |
|                                                            |
| CRITICAL，如果：                                             |
| - crash、security issue、data loss。                         |
|                                                            |
| IMPORTANT，如果：                                            |
| - logic bug、edge case、missing error handling。             |
|                                                            |
| SUGGESTION，如果：                                           |
| - 有价值改进，但不一定阻塞。                                |
|                                                            |
| NIT，如果：                                                  |
| - 纯清理，且必须包含 suggestion block。                      |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：是否建议加测试？                                      |
|                                                            |
| 建议加测试，如果：                                           |
| - 覆盖 distinct code path 或 edge case。                     |
| - 是 bug fix 且可写 regression。                             |
| - user-visible/P0 flow 缺少验证。                            |
|                                                            |
| 不建议，如果：                                               |
| - 只是 constructor inputs / struct fields 组合变化。         |
| - 现有测试已覆盖同一行为。                                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 写 review.json                                            |
| summary:                                                     |
| - Overview                                                   |
| - Concerns                                                   |
| - Found: X critical, Y important, Z suggestions              |
| - Approve / Approve with nits / Request changes              |
|                                                            |
| comments:                                                    |
| - path relative repo root                                    |
| - line                                                       |
| - side LEFT/RIGHT                                            |
| - optional start_line                                        |
| - body starts with required label                            |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 最终检查                                                  |
| - jq validate review.json                                    |
| - 确认 line/side 对应 annotated diff。                       |
| - 不运行 gh pr review / gh pr comment / gh api 发布评论。    |
+------------------------------------------------------------+
```

解释：原 skill：[review-pr](https://github.com/warpdotdev/warp/blob/master/.agents/skills/review-pr/SKILL.md)。

要点：
- 适用场景：自动 review 工作流提供 `pr_diff.txt` 和 `pr_description.txt`，需要生成机器可发布的 `review.json`。
- 核心判断：只对 PR changed lines 发 inline comment；涉及 untouched code 的风险只能放 summary。severity 要按 crash/security/data loss、logic/error handling、suggestion、nit 分级。
- 关键产物：合法 JSON、带 Overview/Concerns/Verdict 的 summary、line/side 正确的 comments，以及 `jq` 验证通过的 `review.json`。
- 容易踩的坑：直接发 GitHub review、评论未改动行、line side 用错、nit 没有 suggestion block、或建议只增加重复输入组合而不覆盖新行为的测试。

### `review-pr-local`

用途：给 `review-pr` 增加 Warp 仓库本地审查规则。

```text
+------------------------------------------------------------+
| 触发：core review-pr skill 在 warp-external / Warp 仓库运行。|
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 先遵守 core review-pr                                    |
| 本地 skill 不重写：                                         |
| - output schema                                              |
| - severity labels                                            |
| - safety rules                                               |
| - evidence rules                                             |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 叠加 Warp Rust 风格检查                                  |
| 标记问题，如果：                                             |
| - 有不必要类型标注。                                        |
| - 用长 path qualifier 代替合理 import。                      |
| - context 参数不叫 ctx 或没有放最后。                        |
| - 用 _param 保留未用参数，而不是删除。                       |
| - format macro 没用 inline format arguments。                |
| - enum 可穷尽却用了 wildcard `_` match arm。                 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. FeatureFlag 审查                                          |
| 标记问题，如果：                                             |
| - 新/改 feature flag 主要控制行为，却用 #[cfg] 到处切。      |
| - runtime FeatureFlag::X.is_enabled() 足以满足需求。         |
|                                                            |
| 不标记，如果：                                               |
| - 代码在 feature 关闭时无法编译。                            |
| - 平台或依赖要求 compile-time gate。                         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. TerminalModel locking 审查                                |
| 标记问题，如果：                                             |
| - call stack 可能已持有 model lock。                         |
| - 新代码再次 model.lock()，形成嵌套/重复锁风险。             |
| - 锁作用域过大。                                            |
| 建议：传递已锁引用，缩短 lock scope。                        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. WarpUI 审查                                               |
| 标记问题，如果：                                             |
| - render 或 event handling 中 inline MouseStateHandle::default() |
| - 本应构造时创建 handle，却每次渲染新建。                    |
|                                                            |
| UI 验证建议仅在有具体风险时提出：                           |
| - 用户可见行为变化。                                        |
| - 布局/视觉状态需要截图或 layout validation。                |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. 用户文案审查                                              |
| 标记问题，如果：                                             |
| - 插值后句子读起来不自然。                                  |
| - 拼接片段大小写或语法不匹配。                              |
| - link text 是裸 URL、click here 等低信息文本。              |
| - 产品术语在 UI、comments、workflow、errors 中不一致。       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 7. 降级和可观测性审查                                        |
| 标记问题，如果：                                             |
| - optional URL/session/workflow/metadata 缺失时渲染空内容。  |
| - user-visible error 泄露过多内部细节。                      |
|                                                            |
| 不建议删除，如果：                                           |
| - session link、workflow URL、diagnostic context 对 debug 有用。 |
| - 错误路径需要保留结构化日志给维护者排查。                  |
+------------------------------------------------------------+
```

解释：原 skill：[review-pr-local](https://github.com/warpdotdev/warp/blob/master/.agents/skills/review-pr-local/SKILL.md)。

要点：
- 适用场景：在 Warp 仓库执行 PR review，需要叠加 repo-specific 的常见风险和风格规则。
- 核心判断：本地规则不改变 core review schema，只提供额外审查维度：Rust 约定、FeatureFlag 使用、TerminalModel locking、WarpUI handle 生命周期、用户文案和错误诊断信息。
- 关键产物：更贴近 Warp 代码库的 review comments，尤其是能提前发现锁、UI 状态、feature rollout 和 observability 的回归。
- 容易踩的坑：把 V0 的非阻塞健壮性建议写成 blocking concern，或建议删除 session/workflow diagnostic links，降低后续排障能力。

### `triage-issue-local`

用途：为 `warp-external` issue triage 提供 Warp 本地规则。

```text
+------------------------------------------------------------+
| 触发：core triage-issue skill 在 warp-external 仓库运行。    |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 先读 issue 证据                                           |
| - title/body/comments                                        |
| - screenshots/videos                                         |
| - logs/stack traces/command output                           |
| - existing labels                                            |
| - repo docs / feature set                                    |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 隐私和公开 issue 安全                                     |
| 不要在公开 issue 中要求：                                    |
| - secrets/tokens                                             |
| - private workspace names                                    |
| - private repository names                                   |
| - account identifiers                                        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：是否 billing / appeal / account-status？               |
|                                                            |
| 是，如果涉及：                                               |
| - pricing/plans/subscriptions/payments/refunds/invoices      |
| - AI quotas/charges                                          |
| - suspensions/bans/takedowns/abuse decisions                 |
|                                                            |
| 处理：                                                       |
| - 不作为普通 bug/feature triage。                            |
| - 引导到 Warp support channel。                              |
| - 视情况加 area:billing 或 area:auth。                       |
|                                                            |
| 否：继续技术 triage。                                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 区分观察事实和用户猜测                                    |
| 事实：用户看到什么、做了什么、实际/期望行为。                |
| 猜测：Rust module、server behavior、feature flag、产品意图。 |
| triage 以事实为主要证据。                                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 判断 surface / labels                                     |
| - area:shell-terminal: output/block/shell integration/prompt |
| - area:terminal-input: input/cursor/key handling/IME         |
| - area:window-tabs-panes: window/tab/pane/focus/layout       |
| - area:editor-notebooks: editor/notebook/markdown/LSP        |
| - area:agent: agent conversations/local-cloud/prompts/AI UI |
| - area:code-review: diff/review comments/PR agent flows      |
| - area:mcp: MCP server/tool/OAuth/resource discovery         |
| - area:settings-keybindings: preferences/shortcuts           |
| - area:warp-drive: sync/share/workflows/notebooks            |
| - area:performance:*: CPU/memory/GPU/startup/render latency  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：是否已有功能/设置可满足？                              |
|                                                            |
| 是，如果 docs 或 repo feature set 显示已有 workflow。         |
| 处理：推荐现有功能，不当作 bug/feature gap。                 |
|                                                            |
| 否：继续判断复现和路由。                                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：是否需要 follow-up question？                         |
|                                                            |
| 可以问，必须满足：                                           |
| - 答案会改变 label、owner 或 reproduction confidence。       |
| - 现有 evidence 无法推断。                                  |
| - 最多 2 个问题。                                           |
|                                                            |
| 不该问，如果：                                               |
| - 答案能从截图/日志/issue body 推断。                        |
| - 问题只是泛泛索取信息。                                     |
| - 会要求公开隐私数据。                                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. 常见 follow-up 模式                                       |
| - 视觉问题无媒体：要 screenshot/short recording。            |
| - terminal 环境敏感：Warp version/channel、OS、shell、fresh session。 |
| - SSH/tmux：local/remote OS、shell、tmux、minimal workflow。 |
| - Agent/MCP：workflow、local/cloud、session link、tool/server、redacted error。 |
| - performance：scale、运行时长、触发动作、logs/profile。     |
| - keyboard/input：layout、custom bindings、IME、focused surface。 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 7. owner inference                                            |
| - 优先 .github/STAKEHOLDERS。                                |
| - 无 path-level match 时，用 label/surface 推断 owner。       |
+------------------------------------------------------------+
```

解释：原 skill：[triage-issue-local](https://github.com/warpdotdev/warp/blob/master/.agents/skills/triage-issue-local/SKILL.md)。

要点：
- 适用场景：对 `warp-external` 公开 issue 做分类、补信息、打 label、推断 owner。
- 核心判断：以用户观察到的 Warp 行为为准，而不是用户对 Rust 模块、server、feature flag 或产品意图的猜测；公开 issue 中不能索取 secrets、账号标识或私有 workspace/repo 信息。
- 关键产物：正确 area/os/repro/type labels、最多两个高价值 follow-up questions、必要时 support redirect，以及基于 `.github/STAKEHOLDERS` 的 owner inference。
- 容易踩的坑：问太多可以从证据推断的问题、把 billing/appeal 当普通 bug 处理、或忽略截图/日志中已经给出的关键信息。

### `dedupe-issue-local`

用途：为 `warp-external` issue 判重提供 Warp 仓库的本地规则。

```text
+------------------------------------------------------------+
| 触发：core dedupe-issue skill 在 warp-external 仓库中运行， |
|       需要判断 issue 是否和已有 issue 重复。                |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 先执行 core dedupe 规则                                  |
| 本地 skill 不重写：                                         |
| - duplicate detection algorithm                             |
| - similarity thresholds                                     |
| - output contract                                           |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. Warp 本地 title normalization                            |
| 比较标题前先弱化/移除低信号前缀：                           |
| - Bug:, Feature:, Request:                                  |
| - [Bug], [Feature]                                          |
| - Warp:                                                     |
| - [macOS], [Linux], [Windows]                               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：版本、OS、shell 是否阻止判重？                        |
|                                                            |
| 不阻止，如果：                                               |
| - 核心症状相同。                                            |
| - 复现路径相同或高度重叠。                                  |
| - version/os/shell 只是环境差异。                           |
|                                                            |
| 阻止或降低置信度，如果：                                    |
| - 问题明显只发生在不同 surface 或完全不同 workflow。         |
| - 环境差异解释了不同 root cause。                           |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：是否只是关键词相同？                                  |
|                                                            |
| 不能判重，如果只共享这些宽泛词：                            |
| - agent / terminal / MCP / settings / search / sync          |
|                                                            |
| 可以继续判重，如果：                                        |
| - 实际失败行为相同。                                        |
| - 期望能力或 broken capability 相同。                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 按 surface 深入比较                                          |
|                                                            |
| Terminal issue 比较：                                       |
| - shell/session context                                     |
| - command output behavior                                   |
| - prompt rendering                                          |
| - input behavior                                            |
| - remote/tmux involvement                                   |
|                                                            |
| Agent/MCP issue 比较：                                      |
| - trigger path                                              |
| - local vs cloud execution                                  |
| - MCP server/tool                                           |
| - visible error                                             |
| - expected workflow                                         |
|                                                            |
| UI/rendering issue 比较：                                   |
| - affected surface                                          |
| - visible symptom                                           |
| - screenshot/recording similarity                           |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 输出判重结论                                                 |
| - duplicate：同一 surface、同一核心症状、同一复现路径。      |
| - not duplicate：只是词相似，行为或 workflow 不同。          |
| - uncertain：证据不足，交给 core skill 的 follow-up 机制。   |
+------------------------------------------------------------+
```

解释：原 skill：[dedupe-issue-local](https://github.com/warpdotdev/warp/blob/master/.agents/skills/dedupe-issue-local/SKILL.md)。

要点：
- 适用场景：在 `warp-external` 公开 issue 中判断新报告是否和已有报告重复。
- 核心判断：判重依据是同一用户可观察失败和复现路径，而不是标题中共享 `agent`、`terminal`、`settings` 这类宽泛词。
- 关键产物：对 core dedupe 结果的 Warp 本地修正，包括标题归一化、surface-specific 比较和 duplicate/not duplicate/uncertain 的结论。
- 容易踩的坑：把不同 surface 的问题因为关键词相同而合并，或把 OS/version/shell 差异当成绝对 blocker，而不是辅助证据。

[返回索引](README.md)
