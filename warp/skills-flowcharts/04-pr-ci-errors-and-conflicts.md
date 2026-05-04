# 四、PR、CI、错误修复与冲突处理

用于把本地改动送入 review、诊断 CI、修复编译/测试问题，以及在 Git 操作中处理冲突。


### `create-pr`

用途：为当前分支创建或更新 Warp PR。

```text
+------------------------------------------------------------+
| 触发：用户要求开 PR、提交 review、准备 merge，或当前改动     |
|       已完成并需要进入代码审查。                            |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 确认当前分支和 base                                      |
| - git branch --show-current                                 |
| - 默认合入 origin/master 前先同步：                         |
|   git fetch origin                                          |
|   git merge origin/master                                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：merge master 是否产生冲突？                           |
|                                                            |
| 有冲突，如果：                                               |
| - git status 显示 unmerged paths。                           |
| - 文件含 <<<<<<< / ======= / >>>>>>>。                       |
|                                                            |
| 处理方式：使用 resolve-merge-conflicts skill。               |
|                                                            |
| 无冲突：继续 PR 准备。                                      |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：PR 是否包含代码改动？                                 |
|                                                            |
| 是代码改动，如果：                                          |
| - Rust/Swift/TS/UI/构建配置/测试文件发生变化。               |
| - 行为、编译结果或 runtime path 会改变。                     |
|                                                            |
| 文档-only，如果：                                           |
| - 仅 markdown、skills、说明文档变化。                        |
| - 不影响构建或 runtime。                                    |
+-----------------------------+------------------------------+
                 |                              |
                 v                              v
+--------------------------------+  +--------------------------------+
| 代码改动：运行相关检查         |  | 文档-only：可跳过 cargo checks |
| - ./script/presubmit           |  | 但仍要 review diff 和 PR body  |
| - 或 cargo fmt/clippy/tests    |  |                                |
+---------------+----------------+  +---------------+----------------+
                |                                   |
                +-----------------+-----------------+
                                  |
                                  v
+------------------------------------------------------------+
| 判断：测试覆盖是否足够？                                    |
|                                                            |
| 需要 regression test，如果：                                |
| - 修 bug，且能写出之前失败、现在通过的用例。                 |
|                                                            |
| 需要 unit test，如果：                                      |
| - 新增非平凡算法、数据结构、解析、搜索、布局逻辑。           |
|                                                            |
| 需要 layout validation，如果：                              |
| - 新增或改动 View 组件，至少应验证可 layout 不 panic。       |
|                                                            |
| 需要 integration test，如果：                               |
| - 改用户可见 flow。                                         |
| - 修 end-to-end regression。                                |
| - P0 use case：坏掉会导致 out-of-band release。              |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. Review 本地改动                                           |
| - git --no-pager log <base>..HEAD --oneline                 |
| - git --no-pager diff <base>...HEAD --stat                  |
| - git --no-pager diff <base>...HEAD                         |
| 目标：确认没有意外文件、描述能覆盖真实变化。                 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. Linear / 标题                                             |
| - 有 Linear issue 时，标题包含 [WARP-1234] 便于自动关联。    |
| - remote branch 使用 owner/name 前缀约定。                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 创建或更新 PR                                             |
| 判断现有 PR：gh pr view --json number,url                   |
|                                                            |
| 不存在：gh pr create --draft --title ... --body ...         |
| 已存在：gh pr edit --title ... --body ...                   |
| PR body 使用 .github/pull_request_template.md。              |
| commit/PR description 末尾加：                              |
| Co-Authored-By: Warp <agent@warp.dev>                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 后续                                                     |
| - 监控 CI。                                                 |
| - 失败时用 diagnose-ci-failures / fix-errors。              |
| - review comments 需要处理时进入对应 GitHub review flow。   |
+------------------------------------------------------------+
```

解释：原 skill：[create-pr](https://github.com/warpdotdev/warp/blob/master/.agents/skills/create-pr/SKILL.md)。

要点：
- 适用场景：本地改动准备进入 review、用户要求开 PR、更新现有 PR，或需要把当前分支提交给 CI。
- 核心判断：先确认分支基于最新 `origin/master`，再按改动类型决定验证范围；代码改动要跑 fmt/clippy/tests，文档-only 可以跳过 Rust presubmit。
- 关键产物：同步后的 feature branch、经过检查的 diff、包含 Linear ID 和 What/Why/How 的 PR 描述、draft PR，以及必要测试说明。
- 容易踩的坑：没 merge master 就开 PR、PR body 和实际 diff 不一致、bug fix 没 regression test、P0/user-visible flow 没考虑 integration test。

### `diagnose-ci-failures`

用途：诊断 PR 的 GitHub Actions / CI 失败并生成修复计划。

```text
+------------------------------------------------------------+
| 触发：用户要求查看 CI、诊断 PR checks、分析失败日志，或      |
|       PR 状态显示 failed。                                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 确认当前 branch                                           |
| git branch --show-current                                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 确认 PR 是否存在                                          |
| gh --no-pager pr view <branch> --json number,title,url,state |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：PR 存在吗？                                            |
|                                                            |
| 不存在：                                                     |
| - 告知用户当前 branch 没有 PR。                              |
| - 可建议使用 create-pr skill。                              |
| - 不继续拉 CI。                                             |
|                                                            |
| 存在：继续检查 statusCheckRollup。                          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 读取 CI 状态                                              |
| gh pr view <branch> --json statusCheckRollup                |
| 分类：                                                      |
| - completed + success                                       |
| - completed + failure                                       |
| - in progress / queued                                      |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：CI 是否仍在运行？                                      |
|                                                            |
| 仍在运行，如果：                                             |
| - 有 queued / in_progress checks。                           |
| - summary check 尚未完成。                                  |
|                                                            |
| 输出方式：                                                   |
| - 列出已失败 checks。                                       |
| - 列出仍在运行 checks。                                     |
| - 建议等完成后再做最终诊断。                                |
|                                                            |
| 已完成：继续提取所有失败 logs。                             |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 对每个 failed check 提取日志                              |
| gh run view <run-id> --log-failed                           |
| 重点摘取：                                                   |
| - file paths / line numbers                                 |
| - compiler errors                                           |
| - clippy lint names                                         |
| - test names / assertion failure                            |
| - stack trace                                               |
| - platform-specific context                                 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 错误分类                                                  |
| - Formatting: cargo fmt                                     |
| - Linting: cargo clippy                                     |
| - Compilation: type/import/signature/trait                   |
| - Test failure: failing test + reason                        |
| - Platform-specific: WASM/Linux/macOS/Windows                |
| - Build infra: dependency/cache/toolchain                    |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：本地失败是否应作为 CI 问题？                           |
|                                                            |
| 应作为问题，如果：                                           |
| - 同一测试/检查在 CI 失败。                                  |
| - 日志显示与本 PR 改动相关。                                |
|                                                            |
| 谨慎处理，如果：                                             |
| - 本地失败但 CI 通过。                                      |
| - 可能是本地环境或 flaky。                                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. 输出修复计划，不直接改代码                                |
| 计划包括：                                                   |
| - Problem Statement                                          |
| - Current State                                              |
| - Proposed Changes                                           |
| - Validation Steps                                           |
| 并引用 fix-errors skill 处理具体错误类型。                   |
+------------------------------------------------------------+
```

解释：原 skill：[diagnose-ci-failures](https://github.com/warpdotdev/warp/blob/master/.agents/skills/diagnose-ci-failures/SKILL.md)。

要点：
- 适用场景：PR CI failed、用户要求检查 checks、需要从 GitHub Actions 日志中提取失败原因。
- 核心判断：这个 skill 只诊断并产出计划，不直接改代码；如果 CI 仍在运行，要区分已失败 checks 和仍在 queued/in progress 的 checks。
- 关键产物：失败 check 列表、run log 摘要、错误分类，以及包含 Problem Statement、Current State、Proposed Changes、Validation Steps 的修复计划。
- 容易踩的坑：只看 summary check 不看真实 failed run、把本地环境失败当 CI 失败、或在未确认 root cause 前直接开始改代码。

### `fix-errors`

用途：修复编译、格式化、clippy、WASM、测试失败。

```text
+------------------------------------------------------------+
| 触发：cargo/check/presubmit/CI 报错，或用户要求修 build、    |
|       lint、test failure。                                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 选择检查范围                                              |
|                                                            |
| 全量 pre-PR：./script/presubmit                             |
| 只查 fmt：cargo fmt -- --check                              |
| 全 workspace clippy：                                       |
|   cargo clippy --workspace --exclude warp_completer ...     |
| warp_completer：cargo clippy -p warp_completer ...          |
| WASM：cargo clippy --target wasm32-unknown-unknown ...      |
| tests：cargo nextest run --no-fail-fast --workspace ...     |
| doc tests：cargo test --doc                                 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 读完整错误                                                |
| 必须确认：                                                   |
| - 第一个 root cause 是什么。                                 |
| - 后续错误是否由第一个错误级联。                            |
| - expected type 与 actual type。                            |
| - 失败是否只发生在 WASM 或特定 feature。                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：错误类型                                               |
|                                                            |
| unused import/const：删除无用项。                            |
| unknown import/type：rg 搜索正确模块路径。                   |
| type mismatch：按 expected 类型调整借用/转换/所有权。        |
| struct field change：更新所有初始化、解构、pattern。         |
| function signature change：更新所有 call sites。             |
| enum variant change：补 exhaustive match arm。               |
| trait impl mismatch：让 return type / bounds 满足 trait。    |
| fmt：运行 cargo fmt。                                       |
| clippy：修 lint，不用 allow 掩盖真实问题。                   |
| test failure：先复现单测，再定位行为差异。                  |
| WASM：检查 local_fs、dead_code、filesystem API。             |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：WASM 错误是否由 filesystem API 导致？                  |
|                                                            |
| 是，如果：                                                   |
| - wasm32 target 编译失败。                                   |
| - 错误路径涉及 std::fs、Path IO、local filesystem tests。    |
|                                                            |
| 修法：                                                       |
| - gate tests: #[cfg(feature = "local_fs")]                   |
| - gate code only when compile requires it。                  |
| - 对仅 local_fs 使用的类型加 cfg_attr allow(dead_code)。     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 小步修复                                                  |
| - 一次修一个错误类别。                                       |
| - 多个 call sites 用 rg 找全。                               |
| - 避免无关重构。                                             |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. targeted 验证                                             |
| - 单 crate：cargo nextest run -p <package>                   |
| - 单测：cargo nextest run -E 'test(<substring>)'             |
| - clippy：运行失败对应 package/target。                      |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 最终验证                                                  |
| - cargo fmt                                                  |
| - cargo clippy                                               |
| - relevant tests                                             |
| - PR 前必要时 ./script/presubmit                             |
+------------------------------------------------------------+
```

解释：原 skill：[fix-errors](https://github.com/warpdotdev/warp/blob/master/.agents/skills/fix-errors/SKILL.md)。

要点：
- 适用场景：编译错误、clippy、fmt、WASM build、nextest 或 presubmit 失败。
- 核心判断：先读完整错误并分类，再一类一类修。很多 Rust 错误是级联的，第一处类型、trait、import 或 signature 问题修掉后，后续错误可能自然消失。
- 关键产物：最小范围代码修复、针对失败类型的验证命令输出、最终 fmt/clippy/tests 通过。
- 容易踩的坑：盲目加 `allow`、只修表面报错、不检查所有 call sites、WASM 下忘记 `local_fs` gate 或错误地把 runtime gate 当成 compile gate。

### `resolve-merge-conflicts`

用途：用紧凑上下文解决 Git merge/rebase/cherry-pick/stash 冲突。

```text
+------------------------------------------------------------+
| 触发：merge、rebase、cherry-pick、stash pop 停在 conflict， |
|       或 git status 显示 unmerged paths。                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 先提取摘要，不直接读大文件                               |
| python3 .agents/skills/resolve-merge-conflicts/scripts/     |
|   extract_conflict_context.py                               |
| 输出：                                                       |
| - unresolved files                                           |
| - index stages                                               |
| - text hunks count                                           |
| - marker or index-only conflict hints                        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 逐个文件 drill down                                      |
| python3 .../extract_conflict_context.py --file path/to/file  |
| 输出：                                                       |
| - nearby context                                             |
| - ours/base/theirs sections                                  |
| - compact unified diff between ours and theirs               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：能否直接选择一边？                                    |
|                                                            |
| 用 ours，如果：                                              |
| - 当前分支版本完整保留目标改动。                             |
| - theirs 是旧实现或被当前改动替代。                          |
| - 没有需要从 theirs 合入的新行为。                           |
| 命令：git checkout --ours -- path/to/file                    |
|                                                            |
| 用 theirs，如果：                                            |
| - incoming 版本是正确新基线。                                |
| - ours 的改动已废弃或可丢弃。                                |
| 命令：git checkout --theirs -- path/to/file                  |
|                                                            |
| 手动合并，如果：                                             |
| - 两边都包含必须保留的行为。                                 |
| - 需要结合两个分支的 API/signature/logic。                   |
| - index-only conflict 不能用 marker 直接判断。               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 编辑解决                                                   |
| - 优先只编辑冲突 hunk。                                      |
| - 只有 compact context 不够时再读更多文件内容。              |
| - 删除所有 conflict markers。                                |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 复查 unresolved                                           |
| - python3 .../extract_conflict_context.py                    |
| - git diff --name-only --diff-filter=U                       |
| - rg "<<<<<<<|=======|>>>>>>>" resolved_files                |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：是否需要运行测试/构建？                               |
|                                                            |
| 需要，如果：                                                 |
| - 合并改动涉及代码、API、测试、构建配置。                    |
| - 手动整合了逻辑。                                          |
|                                                            |
| 可只做静态检查，如果：                                       |
| - 冲突只在纯文档且无生成产物。                               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 验证并 stage                                              |
| - targeted tests / fmt / clippy as relevant                  |
| - git add resolved files                                     |
| - git status 确认无 unmerged paths                           |
+------------------------------------------------------------+
```

解释：原 skill：[resolve-merge-conflicts](https://github.com/warpdotdev/warp/blob/master/.agents/skills/resolve-merge-conflicts/SKILL.md)。

要点：
- 适用场景：merge、rebase、cherry-pick、stash pop 出现 unmerged paths 或 conflict markers。
- 核心判断：先用脚本提取冲突摘要和单文件 hunk，不直接打开大文件；只有 compact context 不足以判断时才扩大阅读范围。
- 关键产物：逐文件解决后的内容、无 conflict markers、无 unmerged paths、相关测试/构建验证通过，并 stage resolved files。
- 容易踩的坑：一上来读完整文件导致误改非冲突区域、机械选择 ours/theirs 丢掉另一边必要行为、或解决后忘记搜索残留 markers。

[返回索引](README.md)
