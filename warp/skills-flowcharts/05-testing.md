# 五、测试体系

用于选择合适测试层级并实现验证。单元测试覆盖局部逻辑，集成测试覆盖真实 Warp app 中的用户可见流程。


### `rust-unit-tests`

用途：为 Warp Rust 代码写、改、运行 crate-level unit tests。

```text
+------------------------------------------------------------+
| 触发：新增逻辑、修 bug、改行为、需要 regression/unit coverage |
|       或用户要求写/跑 Rust 单元测试。                       |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 判断测试粒度                                              |
| 好测试：                                                     |
| - 一个 case 聚焦一个函数或一个行为。                         |
| - 名称描述行为，例如 parses_utf8_sequence_when_valid。       |
| - 断言能解释失败差异。                                      |
|                                                            |
| 不好的测试：                                                 |
| - 只变化 constructor inputs 或 struct fields，行为不变。     |
| - 依赖全局状态或真实机器配置。                               |
| - 大而混杂，一次覆盖很多不相关路径。                         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 放置测试文件                                              |
| 优先单独文件：                                               |
| - ${filename}_tests.rs                                      |
| - mod_test.rs                                                |
|                                                            |
| 在对应源文件末尾接入：                                      |
| #[cfg(test)]                                                 |
| #[path = "filename_tests.rs"]                                |
| mod tests;                                                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：使用哪种 harness？                                    |
|                                                            |
| 普通同步逻辑：                                               |
| - #[test]                                                    |
| - assert_eq! / assert_ne! 优于 assert!                       |
|                                                            |
| async 逻辑：                                                 |
| - #[tokio::test]，仅当需要 runtime。                         |
|                                                            |
| UI/model 逻辑：                                              |
| - warpui::App::test                                          |
| - initialize_app_for_terminal_view                           |
| - add_window_with_terminal                                   |
|                                                            |
| filesystem-heavy 逻辑：                                      |
| - virtual_fs::{VirtualFS, Stub}                              |
|                                                            |
| feature flag 行为：                                          |
| - FeatureFlag::X.override_enabled(true) scoped guard         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：是否需要特殊注意？                                    |
|                                                            |
| TerminalModel locking：                                      |
| - 避免同一 call stack 多次 model.lock()。                    |
| - lock scope 越短越好。                                      |
|                                                            |
| global/external state：                                      |
| - 优先 mock / VirtualFS。                                    |
| - 必要时 serial_test #[serial]。                             |
|                                                            |
| panic 行为：                                                 |
| - 只有 panic 是 intended API 时才用 #[should_panic]。         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 编写断言                                                  |
| - 用 assert_eq! 提供清晰 diff。                              |
| - 枚举新增行为时，代码和测试都偏好 exhaustive coverage。     |
| - UI 数值断言可用 assert_lines_approx_eq!。                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 运行测试                                                  |
| - cargo nextest run --no-fail-fast --workspace --exclude ... |
| - cargo nextest run -p <crate_name>                          |
| - cargo nextest run -E 'test(<substring>)'                   |
| - cargo test --doc                                           |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 提交前检查                                                |
| - cargo fmt                                                  |
| - cargo clippy --workspace --all-targets --all-features ... |
| - 必要时 ./script/presubmit                                  |
+------------------------------------------------------------+
```

解释：原 skill：[rust-unit-tests](https://github.com/warpdotdev/warp/blob/master/.agents/skills/rust-unit-tests/SKILL.md)。

要点：
- 适用场景：新增非平凡逻辑、修复 bug、需要 regression coverage，或需要验证 crate-level 行为。
- 核心判断：测试应覆盖一个明确行为或代码路径；UI/model 测试优先用 `warpui::App::test` 和现有 helper，文件系统逻辑优先用 `VirtualFS`。
- 关键产物：`*_tests.rs` 或 `mod_test.rs`、源文件末尾的 `#[cfg(test)]` 接入、清晰断言、targeted nextest/doc test 结果。
- 容易踩的坑：写只变化构造参数但不覆盖新行为的测试、依赖真实 home 或全局状态、同一 call stack 多次 `model.lock()`，或用 `#[should_panic]` 表达非 API 语义。

### `warp-integration-test`

用途：编写、接入、运行 Warp 自定义集成测试。

```text
+------------------------------------------------------------+
| 触发：需要端到端验证 Warp UI/terminal 行为，新增或修复       |
|       crates/integration 下的 integration test。             |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 理解框架运行方式                                          |
| Rust test wrapper                                            |
|   -> run_integration_test("test_name")                       |
|   -> launches integration binary                             |
|   -> register_tests() finds Builder                          |
|   -> Builder creates hermetic HOME/temp dirs/rc files         |
|   -> TestDriver runs TestStep sequence                       |
|   -> assertions poll until success/timeout                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 选择测试位置                                              |
| actual test function: crates/integration/src/test/<area>.rs  |
| module hub: crates/integration/src/test.rs                   |
| runner registration: crates/integration/src/bin/integration.rs |
| nextest list:                                                |
| - tests/integration/ui_tests.rs                              |
| - tests/integration/shell_integration_tests.rs               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：放进哪个模块？                                        |
|                                                            |
| 现有模块，如果：                                             |
| - feature area 已有相近测试。                                |
| - 可复用同一组 helpers/assertions。                          |
|                                                            |
| 新模块，如果：                                               |
| - feature 不适合现有区域。                                   |
| - 独立模块能让测试组织更清楚。                               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：列入 ui_tests 还是 shell_integration_tests？           |
|                                                            |
| ui_tests，如果：                                             |
| - 主要验证 UI/app 行为。                                     |
| - 不需要每个 shell matrix。                                  |
|                                                            |
| shell_integration_tests，如果：                              |
| - 行为依赖 shell integration。                               |
| - 需要每个 shell 或特定 shell 集合验证。                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 写 pub fn test_name() -> Builder                          |
| Builder::new() 默认提供：                                    |
| - per-test root directory                                    |
| - isolated HOME                                              |
| - generated Bash/Zsh/Fish rc files                           |
| - file-backed user preferences                               |
| - default 2-minute timeout                                   |
| - optional real display support                              |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：需要 Builder setup 吗？                               |
|                                                            |
| with_setup，如果：                                           |
| - 需要 test dir 文件。                                       |
| - 需要设置 runtime env vars。                                |
| - 需要 fixture config。                                      |
|                                                            |
| with_user_defaults，如果：                                   |
| - 需要预置 Warp persisted preferences。                      |
|                                                            |
| set_should_run_test，如果：                                  |
| - 测试只在特定平台/shell/runtime capability 下有效。         |
|                                                            |
| with_real_display，如果：                                    |
| - 需要截图/视频/真实显示。                                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 写 TestStep sequence                                      |
| 推荐起步：                                                   |
| - wait_until_bootstrapped_single_pane_for_tab(0)             |
| - 如果 block_index 重要，clear_blocklist_to_remove_bootstrapped_blocks() |
| - 使用 execute_command_for_single_terminal_in_tab 等 helper  |
| - new_step_with_default_assertions("...")                    |
| - add_named_assertion(...)                                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：用 helper 还是 raw events？                            |
|                                                            |
| 用 helper，如果：                                            |
| - app/src/integration_testing/ 已有 terminal/blocklist/      |
|   command palette/navigation/settings/workflow helpers。      |
|                                                            |
| 用 raw with_event/with_event_fn，如果：                      |
| - 没有合适 helper。                                          |
| - 测试必须驱动低层鼠标/键盘位置。                            |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：sleep、retry、PreconditionFailed 怎么用？              |
|                                                            |
| 不用 sleep，如果可轮询 assertion 等待最终状态。              |
|                                                            |
| set_retries 仅用于真正 retryable 的 step，不能掩盖确定性 bug。 |
|                                                            |
| PreconditionFailed 用于环境进入无效状态，允许 outer harness  |
| 重跑整个测试。                                               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 接入所有注册点                                            |
| - module added to crates/integration/src/test.rs             |
| - pub use exported from crates/integration/src/test.rs       |
| - register_test!(test_name) in bin/integration.rs            |
| - listed in ui_tests.rs or shell_integration_tests.rs        |
| 默认应进入 CI；只有有明确原因才 #[ignore]。                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. 运行                                                      |
| authoring 快速路径：                                         |
|   cargo run -p integration --bin integration -- test_name     |
| nextest：                                                    |
|   cargo nextest run --no-fail-fast --workspace test_name      |
| real display：                                               |
|   WARPUI_USE_REAL_DISPLAY_IN_INTEGRATION_TESTS=1 ...         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 7. 验证完成标准                                              |
| - 测试函数、module、pub use、register、macro list 都已接入。 |
| - direct integration binary 通过。                           |
| - 如进自动化，nextest 通过。                                 |
| - assertions 验证用户可见行为。                              |
| - 不依赖真实 home、dotfiles、settings、机器状态。             |
| - screenshot/video artifact 如生成，必须实际检查。           |
+------------------------------------------------------------+
```

解释：原 skill：[warp-integration-test](https://github.com/warpdotdev/warp/blob/master/.agents/skills/warp-integration-test/SKILL.md)。

要点：
- 适用场景：需要验证真实 Warp app 中的 UI、terminal、shell integration 或用户可见端到端 flow。
- 核心判断：integration test 不是普通 unit test。它启动真实 app、隔离 HOME、生成 shell rc、驱动 TestStep，并通过 polling assertions 等待最终状态。
- 关键产物：`pub fn test_name() -> Builder`、`src/test.rs` module/pub use、`bin/integration.rs` 注册、`ui_tests.rs` 或 `shell_integration_tests.rs` 接入，以及 direct run 和 nextest 结果。
- 容易踩的坑：只写测试函数但忘记注册到 nextest macro list、用 sleep 代替 polling assertion、依赖个人 dotfiles/settings、或截图/视频 artifact 生成后不实际检查。

[返回索引](README.md)
