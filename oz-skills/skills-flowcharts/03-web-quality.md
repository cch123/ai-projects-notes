# 三、Web 测试、性能、可访问性与 SEO/AEO

这一组是 `oz-skills` 相比旧 Warp skills 最明显的新增面：从代码实现扩展到本地 Web app 验证、Chrome DevTools 性能审计、WCAG 可访问性和搜索/AI 可见性。


### `webapp-testing`

用途：用 Playwright 测试本地 Web 应用、截图、查浏览器日志和验证交互。

```text
+------------------------------------------------------------+
| 触发：用户要求验证 frontend、debug UI、截图、查 console 或    |
|       测试本地 web app。                                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. 选择测试路径                                              |
|                                                            |
| static HTML：                                                |
| - 直接读 HTML 找 selectors。                                 |
| - 如果信息不足，按 dynamic app 处理。                        |
|                                                            |
| dynamic webapp：                                             |
| - 判断 server 是否已运行。                                   |
| - 未运行先用 scripts/with_server.py --help。                 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. 管理 server                                               |
| - 优先用 skill 自带 with_server.py。                         |
| - 单 server 或多 server 都由 helper 管 lifecycle。            |
| - Playwright 脚本只写浏览器逻辑。                            |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. Reconnaissance then action                                |
| - page.goto(url)                                             |
| - page.wait_for_load_state('networkidle')                    |
| - screenshot / page.content / locator inspection             |
| - 根据实际 rendered DOM 选择 selectors。                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. 执行动作并验证                                            |
| - 点击、输入、导航、断言。                                   |
| - 捕获截图、console、network/DOM 证据。                      |
| - 记录失败时的状态，避免只报“没过”。                         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 常见坑                                                    |
| - dynamic app 没等 networkidle 就 inspect DOM。               |
| - 跳过侦察，直接猜 selector。                                |
| - 把 server 管理逻辑混入测试脚本导致重复复杂度。             |
+------------------------------------------------------------+
```

解释：原 skill：[webapp-testing](https://github.com/warpdotdev/oz-skills/blob/main/.agents/skills/webapp-testing/SKILL.md)。

要点：
- 旧 Warp skills 有 integration test，但没有独立的 Playwright 本地 Web app 测试工作流。
- 关键架构是把 server lifecycle 和 browser automation 分离：helper 管服务，脚本只表达用户行为。
- 容易踩的坑：没等 JS 执行完成就查 DOM、截图前页面还在加载、或用源码里的 selector 代替 rendered DOM 事实。

### `web-performance-audit`

用途：用 Chrome DevTools MCP 审计页面加载性能、Core Web Vitals、网络请求和代码库优化机会。

```text
+------------------------------------------------------------+
| 触发：用户要求 audit/profile/debug/optimize 页面性能、        |
|       Lighthouse 分数或 site speed。                         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 0. 验证 Chrome DevTools MCP 可用                             |
| - 尝试 navigate_page 或 performance_start_trace。             |
| - 不可用则停止，要求配置 chrome-devtools MCP。               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. Performance trace                                         |
| - navigate_page(url)                                         |
| - performance_start_trace(autoStop: true, reload: true)      |
| - 记录 cold load trace。                                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. Core Web Vitals 分析                                      |
| 指标阈值：                                                   |
| - LCP < 2.5s / < 4s / poor                                   |
| - INP < 200ms / < 500ms / poor                               |
| - CLS < 0.1 / < 0.25 / poor                                  |
| - FCP、TTFB、TBT、Speed Index 同步评估。                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. Network analysis                                          |
| - scripts / stylesheets / documents / fonts / images         |
| - render-blocking resources                                  |
| - late network chains                                        |
| - missing preloads                                           |
| - caching headers                                            |
| - oversized or uncompressed bundles                          |
| - unused preconnects，先验证 origin 是否真的无请求。          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. Accessibility snapshot                                    |
| - take_snapshot(verbose: true)                               |
| - 标记 ARIA、contrast、focus、accessible name 高层问题。      |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. Codebase analysis                                         |
| 如果有源码：                                                 |
| - 检测 bundler/framework。                                   |
| - 检查 tree-shaking、dead code、unused CSS/JS、polyfills。    |
| - 检查 compression/minification/source maps。                |
| 第三方站点无源码时跳过。                                     |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. 输出报告                                                  |
| - Core Web Vitals table                                      |
| - top issues by impact                                       |
| - specific recommendations                                   |
| - codebase findings                                          |
+------------------------------------------------------------+
```

解释：原 skill：[web-performance-audit](https://github.com/warpdotdev/oz-skills/blob/main/.agents/skills/web-performance-audit/SKILL.md)。

要点：
- 核心原则是“先量化再建议”：没有影响的数据点不要升级成优化任务。
- 它要求用 MCP trace、network request 和代码库证据验证建议，例如未用 preconnect 必须确认该 origin 没有请求。
- 容易踩的坑：泛泛建议“优化图片/压缩 JS”、把 0ms impact 当 blocker、或没有源码时仍然臆测 bundler 配置。

### `web-accessibility-audit`

用途：按 WCAG 2.0/2.1/2.2 审计 Web 应用可访问性，并输出可执行修复建议。

```text
+------------------------------------------------------------+
| 触发：用户要求 accessibility/a11y/WCAG audit，或提到 screen   |
|       reader、keyboard navigation 等问题。                   |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. Automated testing                                         |
| - 运行 axe / Lighthouse / eslint-plugin-jsx-a11y 等工具。    |
| - skill 自带 run-eslint.sh 和 grep patterns 可辅助。          |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. Manual code inspection                                    |
| 搜索常见问题：                                               |
| - img alt                                                    |
| - aria-label / role / id                                     |
| - onClick without keyboard handler                           |
| - label/input association                                    |
| - heading hierarchy                                          |
| - focus styles                                               |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. 按 WCAG 和影响分级                                        |
| POUR：Perceivable / Operable / Understandable / Robust       |
| Levels：A must pass，AA should pass，AAA nice to have         |
| 严重度：critical / serious / moderate                        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. Manual testing                                            |
| - keyboard-only navigation                                   |
| - focus order and focus visible                              |
| - screen reader smoke test                                   |
| - reduced motion                                             |
| - form error handling                                        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. 输出报告                                                  |
| - Summary                                                    |
| - Critical Issues                                            |
| - Serious / Moderate Issues                                  |
| - Testing Recommendations                                    |
| - Next Steps                                                 |
| 每个问题包含 WCAG 条款、影响、位置、修复建议。               |
+------------------------------------------------------------+
```

解释：原 skill：[web-accessibility-audit](https://github.com/warpdotdev/oz-skills/blob/main/.agents/skills/web-accessibility-audit/SKILL.md)。

要点：
- 新增的是完整 WCAG 审计框架，而不是简单 UI review：自动工具只能发现一部分，键盘和读屏仍需人工验证。
- 输出要把问题映射到 WCAG 条款和用户影响，否则修复优先级很难判断。
- 容易踩的坑：只跑自动工具就结束、滥用 ARIA 修补语义 HTML、隐藏 focus outline、或给 icon button 留空 accessible name。

### `seo-aeo-audit`

用途：优化搜索可见性、排名、结构化数据，以及面向 AI 引用/回答引擎的 AEO。

```text
+------------------------------------------------------------+
| 触发：用户要求 SEO、meta tags、structured data、search        |
|       ranking、AEO 或 AI visibility。                        |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 1. Technical SEO                                             |
| - robots.txt 不阻塞渲染资源。                                |
| - meta robots / canonical URLs。                             |
| - XML sitemap。                                              |
| - URL structure。                                            |
| - HTTPS/security。                                           |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 2. On-page SEO                                               |
| - title tags。                                               |
| - meta descriptions。                                        |
| - heading structure。                                        |
| - image alt/file naming。                                    |
| - internal linking。                                         |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 3. Structured data                                           |
| JSON-LD templates：                                          |
| - Organization                                               |
| - Article                                                    |
| - Product                                                    |
| - FAQ                                                       |
| - Breadcrumbs                                                |
| 使用 Rich Results / schema validation 检查。                 |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 4. Mobile / international SEO                                |
| - responsive design。                                        |
| - tap targets。                                              |
| - font sizes。                                               |
| - hreflang。                                                 |
| - html lang。                                                |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 5. Audit workflow and tools                                  |
| - Lighthouse SEO category。                                  |
| - PageSpeed Insights API。                                   |
| - Search Console export。                                    |
| - skill scripts 可选：lighthouse.sh/pagespeed.sh/search-console-export.mjs |
+-----------------------------+------------------------------+
                              |
                              v
+------------------------------------------------------------+
| 6. AEO / AI visibility                                       |
| - 内容写成清晰问题和直接答案。                               |
| - 强化 entity、authority、source attribution。                |
| - 增加 FAQ/HowTo/Article 等 schema。                         |
| - 监控 AI citation frequency/context/source。                |
+------------------------------------------------------------+
```

解释：原 skill：[seo-aeo-audit](https://github.com/warpdotdev/oz-skills/blob/main/.agents/skills/seo-aeo-audit/SKILL.md)。

要点：
- 它把传统 SEO 和 AEO 合在一起：不仅让搜索引擎索引，也让 AI answer engines 更容易抽取、引用和归因。
- 新增辅助资产包括 JSON-LD templates 和 Lighthouse/PageSpeed/Search Console 脚本。
- 容易踩的坑：只改 meta description 不看 crawlability、用 schema 标注页面不存在的内容、或把 AEO 写成堆关键词而不是清晰可引用答案。
