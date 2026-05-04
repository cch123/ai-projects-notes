# 二、功能开关与发布推进

用于控制新功能从实验到稳定发布的生命周期。重点是区分编译期 feature、运行时 FeatureFlag、channel rollout 和后续清理。


### `add-feature-flag`

用途：给新功能增加 FeatureFlag，让代码可以按构建配置和运行时 channel 灰度启用。

```text
+------------------------------------------------------------+
| 触发：新功能需要门控、灰度、Dogfood/Preview rollout，或     |
|       需要在未稳定前避免默认暴露给所有用户。                |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 选择 feature 名称                                        |
| - Cargo feature: snake_case，例如 your_feature_name          |
| - FeatureFlag variant: CamelCase，例如 YourFeatureName       |
| - 名称应是高层产品能力，不应按单个 call site 命名。          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. app/Cargo.toml                                           |
| 在 [features] 下添加：                                      |
|   your_feature_name = []                                    |
| 注意：新 flag 默认不要加入 default。Stable rollout 才加。    |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. warp_core/src/features.rs                                |
| 在 FeatureFlag enum 中添加：                                |
|   YourFeatureName                                           |
| 可选：如果要 Dogfood 默认启用，加入 DOGFOOD_FLAGS。          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. app/src/lib.rs                                           |
| 在 enabled_features() 的 bridge 中添加：                    |
|   #[cfg(feature = "your_feature_name")]                     |
|   FeatureFlag::YourFeatureName                              |
| 作用：把 Cargo 编译期开关桥接到运行时 FeatureFlag 集合。     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：业务代码用 runtime gate 还是 #[cfg]?                   |
|                                                            |
| 优先 runtime gate，如果：                                   |
| - 代码在所有构建配置下都能编译。                            |
| - 只是控制行为是否启用。                                    |
| - 希望后续清理、测试、回滚更简单。                          |
|                                                            |
| 使用 #[cfg(feature = "...")]，如果：                        |
| - feature 关闭时相关依赖、类型或模块不存在。                 |
| - 某平台/目标下代码无法编译。                               |
| - 必须避免代码被编进二进制。                                |
+-----------------------------+------------------------------+
                              |
                 +------------+------------+
                 v                         v
+--------------------------------+  +--------------------------------+
| Runtime gate                   |  | Compile-time gate              |
| if FeatureFlag::X.is_enabled() |  | #[cfg(feature = "...")]        |
| { new behavior }               |  | fn / mod / use / branch        |
+---------------+----------------+  +---------------+----------------+
                |                                   |
                +-----------------+-----------------+
                                  |
                                  v
+------------------------------------------------------------+
| 判断：是否新增 keybinding?                                  |
|                                                            |
| 需要加 enabled predicate，如果：                            |
| - 新增 EditableBinding 或 FixedBinding。                     |
| - 绑定属于 gated feature。                                  |
| - feature 关闭时不应出现在 keyboard settings 中。            |
|                                                            |
| 不需要，如果：                                               |
| - keybinding 与该 feature 无关。                             |
| - 绑定在 feature 关闭时仍应存在且行为有效。                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 可选 rollout                                                 |
| - Dogfood: features.rs -> DOGFOOD_FLAGS += FeatureFlag::X    |
| - Stable: app/Cargo.toml -> default += "your_feature_name"   |
| - Preview/Stable promotion 细节走 promote-feature。          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 验证                                                        |
| - cargo run --features your_feature_name                    |
| - cargo fmt                                                 |
| - cargo clippy / targeted tests                             |
| - 搜索确认 cfg bridge、runtime checks、keybinding predicate  |
|   与期望一致。                                              |
+------------------------------------------------------------+
```

解释：原 skill：[add-feature-flag](https://github.com/warpdotdev/warp/blob/master/.agents/skills/add-feature-flag/SKILL.md)。

要点：
- 适用场景：新功能还不能默认暴露，但希望代码先合入主线、支持 Dogfood/Preview 验证，或需要保留快速回滚路径。
- 核心判断：默认使用 `FeatureFlag::X.is_enabled()` 做运行时 gate；只有依赖、平台或构建目标导致代码无法在 flag 关闭时编译，才使用 `#[cfg(feature = "...")]`。
- 关键产物：`app/Cargo.toml` 的 Cargo feature、`warp_core/src/features.rs` 的 enum variant、`app/src/lib.rs` 的 enabled bridge，以及业务调用点上的 gate。
- 容易踩的坑：把新 feature 直接加到 `default`、在多个 call site 滥用 `#[cfg]`、忘记给 gated keybinding 加 `.with_enabled(...)`，都会让 rollout 和后续清理变难。

### `promote-feature`

用途：把 gated FeatureFlag 推广到 Dogfood、Preview 或 Stable。

```text
+------------------------------------------------------------+
| 触发：已有 FeatureFlag 准备扩大启用范围。                    |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 确认当前状态                                              |
| - FeatureFlag variant 是否存在。                             |
| - Cargo feature 是否存在。                                   |
| - app/src/lib.rs enabled_features bridge 是否存在。           |
| - 当前是否在 DOGFOOD_FLAGS / PREVIEW_FLAGS / default。        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：目标 rollout 阶段是什么？                              |
|                                                            |
| Dogfood，如果：                                               |
| - 只想给 Dev/dogfood 使用。                                  |
| - 仍需要内部验证。                                           |
| - 不准备暴露给 Preview/Stable。                              |
|                                                            |
| Preview，如果：                                               |
| - dogfood 已足够稳定。                                       |
| - 想扩大到 Preview 用户。                                    |
| - Stable 仍需要更多信号。                                    |
|                                                            |
| Stable，如果：                                                |
| - 功能准备对所有用户默认启用。                               |
| - 需要保留快速回滚路径。                                     |
| - 不应再只依赖 runtime channel list。                        |
+-----------------------------+------------------------------+
            |                         |                         |
            v                         v                         v
+-----------------------+  +-----------------------+  +-------------------------+
| Dogfood path          |  | Preview path          |  | Stable path             |
| features.rs           |  | features.rs           |  | app/Cargo.toml          |
| DOGFOOD_FLAGS += X    |  | PREVIEW_FLAGS += X    |  | default += "feature"    |
|                       |  | DOGFOOD_FLAGS -= X    |  |                         |
+-----------+-----------+  +-----------+-----------+  +------------+------------+
            |                          |                           |
            |                          |                           v
            |                          |              +-------------------------+
            |                          |              | app/src/lib.rs          |
            |                          |              | enabled_features bridge |
            |                          |              | #[cfg(feature = "...")] |
            |                          |              | FeatureFlag::X          |
            |                          |              +------------+------------+
            |                          |                           |
            |                          |                           v
            |                          |              +-------------------------+
            |                          |              | features.rs             |
            |                          |              | remove X from           |
            |                          |              | DOGFOOD/PREVIEW arrays  |
            |                          |              +------------+------------+
            +--------------------------+---------------------------+
                                       |
                                       v
+------------------------------------------------------------+
| 判断：Stable 后是否删除 flag？                              |
|                                                            |
| 不要立即删除，如果：                                        |
| - 刚 promoted to Stable。                                   |
| - 还需要 1-2 release cycle 的回滚窗口。                      |
|                                                            |
| 后续删除条件：                                               |
| - 功能稳定。                                                |
| - 团队不再需要关闭该功能。                                  |
| - 使用 remove-feature-flag skill 单独清理。                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 验证                                                        |
| - cargo fmt                                                  |
| - cargo clippy --workspace --all-targets --all-features ... |
| - Stable path 检查 default + bridge + channel arrays 一致。  |
| - 创建后续 Linear issue：1-2 release cycle 后移除 flag。     |
+------------------------------------------------------------+
```

解释：原 skill：[promote-feature](https://github.com/warpdotdev/warp/blob/master/.agents/skills/promote-feature/SKILL.md)。

要点：
- 适用场景：已有 FeatureFlag 需要从局部验证推进到 Dogfood、Preview 或 Stable。
- 核心判断：Dogfood 只改 runtime channel list；Preview 要移出 Dogfood 因为 Preview 会被 Dogfood 包含；Stable 推荐加入 Cargo `default`，保留一行回滚路径。
- 关键产物：`DOGFOOD_FLAGS`/`PREVIEW_FLAGS`/`app/Cargo.toml default`/`enabled_features()` 的一致更新，以及 Stable 后的后续 cleanup issue。
- 容易踩的坑：Stable 后立刻删除 flag、只改 runtime array 不改 Cargo default、或忘记从较窄 channel list 中移除已经被更宽 channel 覆盖的 flag。

### `remove-feature-flag`

用途：功能已全量稳定后删除 FeatureFlag 和所有旧分支。

```text
+------------------------------------------------------------+
| 触发：FeatureFlag 已经稳定 rollout，需要清理技术债。         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 前置确认                                                     |
| 可以删除，如果全部满足：                                     |
| - feature 已在 app/Cargo.toml default 中启用。               |
| - 已在生产稳定足够时间。                                     |
| - 无计划再关闭或提供配置开关。                               |
| - 团队同意该功能永久保留。                                   |
|                                                            |
| 不应删除，如果：                                             |
| - 刚刚 promote 到 Stable。                                  |
| - 仍需要快速 rollback。                                      |
| - 功能仍在实验或争议中。                                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 删除 Cargo feature                                        |
| app/Cargo.toml                                               |
| - 从 default = [...] 删除 "your_feature_name"                |
| - 从 [features] 删除 your_feature_name = []                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 删除运行时 flag 定义                                      |
| warp_core/src/features.rs                                    |
| - FeatureFlag enum 删除 YourFeatureName                      |
| - DOGFOOD_FLAGS / PREVIEW_FLAGS / RELEASE_FLAGS 中删除引用   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 删除编译 bridge                                           |
| app/src/lib.rs                                               |
| - 删除 #[cfg(feature = "your_feature_name")]                 |
| - 删除 FeatureFlag::YourFeatureName                          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 全仓搜索所有残留                                          |
| - rg "YourFeatureName" app/ warp_core/                       |
| - rg "FeatureFlag::YourFeatureName" app/                     |
| - rg 'cfg\(feature = "your_feature_name"\)' app/             |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：runtime check 怎么清？                                 |
|                                                            |
| if FeatureFlag::X.is_enabled() { new } else { old }          |
|                                                            |
| 清理为 new，如果：                                           |
| - 功能已稳定全量。                                           |
| - old branch 只是关闭 flag 时的旧行为。                      |
|                                                            |
| 谨慎人工判断，如果：                                         |
| - else branch 不是旧行为，而是错误处理或配置 fallback。      |
| - 分支还承载非 flag 语义。                                  |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 判断：keybinding predicate 怎么清？                          |
|                                                            |
| 删除 .with_enabled(|| FeatureFlag::X.is_enabled())，如果：   |
| - keybinding 应随稳定功能永久存在。                          |
|                                                            |
| 保留/替换 predicate，如果：                                 |
| - 绑定仍受其他条件控制。                                     |
| - feature flag 只是多个 enabled 条件之一。                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 删除 dead code branches                                   |
| - 删除 flag disabled path。                                  |
| - 删除只为实验期存在的 helper/type/import。                  |
| - 保留实际仍被新行为使用的代码。                             |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 验证                                                        |
| - cargo fmt                                                  |
| - cargo clippy --workspace --all-targets --all-features ... |
| - cargo nextest run --no-fail-fast --workspace ...           |
| - cargo run                                                  |
| - rg 确认 feature/variant/cfg/check 无残留。                 |
+------------------------------------------------------------+
```

解释：原 skill：[remove-feature-flag](https://github.com/warpdotdev/warp/blob/master/.agents/skills/remove-feature-flag/SKILL.md)。

要点：
- 适用场景：功能已经全量启用并稳定，团队不再需要关闭它，也不需要把它作为用户配置项保留。
- 核心判断：删除 flag 的结果应是“新行为无条件存在”。所有 `FeatureFlag::X.is_enabled()` 分支都要判断哪边是稳定行为，不能机械删除。
- 关键产物：删除 Cargo feature/default entry、删除 enum variant 和 channel arrays 引用、删除 `enabled_features()` bridge、清理 runtime checks、keybinding predicate 和 dead branches。
- 容易踩的坑：只删 flag 定义却留下旧分支、把仍有非 flag 语义的 fallback 误删、或没有全仓搜索 cfg/runtime 残留。

[返回索引](README.md)
