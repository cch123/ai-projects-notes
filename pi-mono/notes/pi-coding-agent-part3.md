# pi-coding-agent Part 3：会话管理 + Interactive UI

> 分析日期：2026-04-19
> 源码版本：0.67.68
> 源码路径：`packages/coding-agent/src/core/session-manager.ts` 及 `modes/interactive/`

**本篇范围**：会话的持久化格式、仅追加树结构、分支/压缩机制；Interactive 模式的 TUI 架构、消息渲染、工具可视化、斜杠命令、主题系统。

**至此 pi-mono 系列完结**。前情回顾：[Part 1（主入口、模式、工具）](./pi-coding-agent-part1.md) / [Part 2（扩展、模板、Skills）](./pi-coding-agent-part2.md)。

---

## 1. 会话管理

### 1.1 SessionManager 概览

[session-manager.ts](packages/coding-agent/src/core/session-manager.ts)（1425 行）核心职责：

| 模块 | 方法 |
|------|------|
| **文件持久化** | `setSessionFile`, `newSession`, `_rewriteFile`, `_persist` |
| **树遍历** | `getLeafId`, `getBranch`, `getTree`, `getEntries` |
| **分支操作** | `branch`, `resetLeaf`, `branchWithSummary`, `createBranchedSession` |
| **条目追加** | `appendMessage`, `appendCompaction`, `appendCustomEntry`, `appendLabelChange` |
| **会话工厂** | `create`, `open`, `continueRecent`, `inMemory`, `forkFrom`, `list` |

### 1.2 持久化：JSONL + 仅追加

**位置**：`~/.pi/agent/sessions/<encoded-cwd>/<timestamp>_<sessionId>.jsonl`

**格式**：每行一个 JSON 对象，**不可修改、不可删除**，只能追加。

```jsonl
{"type":"session","version":3,"id":"uuid","timestamp":"...","cwd":"/path","parentSession":null}
{"type":"message","id":"m1","parentId":null,"message":{...}}
{"type":"message","id":"m2","parentId":"m1","message":{...}}
{"type":"model_change","id":"c1","parentId":"m2","provider":"anthropic","modelId":"claude-opus"}
{"type":"thinking_level_change","id":"t1","parentId":"c1","thinkingLevel":"high"}
{"type":"compaction","id":"k1","parentId":"t1","summary":"...","firstKeptEntryId":"m2","tokensBefore":45000}
{"type":"branch_summary","id":"b1","parentId":"k1","fromId":"m42","summary":"abandoned exploration"}
{"type":"label","id":"l1","parentId":"b1","targetId":"m2","label":"before-refactor"}
```

**版本迁移**：`migrateV1ToV2` / `migrateV2ToV3` 处理格式升级。

### 1.3 九种条目类型

| 类型 | 用途 | 写入 LLM？ |
|------|------|-----------|
| `message` | 用户/助手消息 | ✅ 直接 |
| `thinking_level_change` | 思考级别切换 | ❌ |
| `model_change` | 模型切换 | ❌ |
| `compaction` | 压缩边界（保留 firstKeptEntryId 之后） | ✅ 作为 `compactionSummary` 消息 |
| `branch_summary` | 分支点摘要 | ✅ 作为 `branchSummary` 消息 |
| `custom` | 扩展私有数据（重载时恢复状态用） | ❌ |
| `custom_message` | 扩展注入的 user 消息 | ✅ 转为 user |
| `label` | 用户书签（标记某条目方便跳转） | ❌ 仅 UI |
| `session_info` | 会话名称等元数据 | ❌ |

### 1.4 非线性树结构

每个条目有 `parentId`，`leafId` 指向当前分支末端：

```
     root(m1)
       │
      m2 ──── label: "before-refactor"
       │
       ├── m3 ── m4 ── m5（旧分支，leafId 曾经在这里）
       │
       └── m6 ── m7（当前 leafId，从 m2 分支出来）
```

**`branch(entryId)`**：移动 `leafId` 到该条目，之后的 prompt 会从这个位置继续，产生新分支。

**`buildSessionContext(leafId)`**（[session-manager.ts:315-422](packages/coding-agent/src/core/session-manager.ts)）：
```
1. 从 leafId 向 root 遍历，收集路径
2. 如果路径上有 compaction 条目，截断到 firstKeptEntryId
3. 条目 → Message 转换：
   - message          → 直接包含
   - custom_message   → 转为 user 消息
   - compaction       → 转为 user 消息（"Previous summary: ..."）
   - branch_summary   → 转为 user 消息（"You took a branch: ..."）
   - 其他 change/label → 忽略
4. 返回 SessionContext { messages, thinkingLevel, model }
```

### 1.5 压缩（Compaction）

**触发**：
- **手动**：`/compact` 命令
- **阈值**：上下文占比超过设定 → `_checkCompaction()` 自动触发
- **溢出恢复**：LLM 返回 context overflow 错误 → 自动压缩 + `agent.continue()` 重试（用户无感知）

**算法**（`core/compaction/index.ts`）：
1. `prepareCompaction` 校验消息数 ≥ 最小阈值
2. 调 LLM 生成摘要（默认 prompt 可被 `customInstructions` 追加或 `replaceInstructions` 完全替换）
3. 追加 `compaction` 条目：
   ```typescript
   {
     type: "compaction",
     summary: "...",
     firstKeptEntryId: "m20",  // 之前的消息在 LLM 视角被"遗忘"
     tokensBefore: 45000,
     fromHook: false,          // 是否扩展钩子生成
   }
   ```
4. 之后 `buildSessionContext` 自动只回放 summary + firstKeptEntryId 之后的消息

**扩展钩子 `session_before_compact`** 可：
- `{ cancel: true }` 取消压缩
- `{ summary: "..." }` 提供自定义摘要
- `{ customInstructions: "..." }` 调整生成风格
- `{ replaceInstructions: "..." }` 完全替换默认提示

**中止**：`AbortController` 支持用户 Esc 取消压缩。

### 1.6 会话入口语义

| 标志 | 方法 | 行为 |
|------|------|------|
| `--continue` / `-c` | `SessionManager.continueRecent()` | 在当前 cwd 对应目录找**最近修改**的会话打开 |
| `--resume` / `-r` | 同上 | `--continue` 的现代别名 |
| `--session <path>` | `SessionManager.open(path)` | 打开指定路径的会话文件 |
| `--fork <path>` | `SessionManager.forkFrom(src, cwd)` | 复制历史到新 sessionId，更新 cwd（跨项目迁移） |
| `--new-session` | `SessionManager.create(cwd)` | 空白新会话 |

**跨项目 fork**：从 cwd A 的会话 fork 到 cwd B 时，历史完整复制但 `cwd` 字段更新；会话文件放在 B 对应的目录。

---

## 2. Interactive 模式 UI

### 2.1 目录结构

```
modes/interactive/
├── interactive-mode.ts            # 5105 行，TUI 主编排
├── components/                    # 40+ 组件
│   ├── assistant-message.ts       # 助手消息渲染
│   ├── user-message.ts
│   ├── tool-execution.ts          # 工具调用/结果可视化
│   ├── bash-execution.ts          # ! 前缀直接 bash
│   ├── custom-message.ts          # 扩展消息卡片
│   ├── compaction-summary-message.ts
│   ├── branch-summary-message.ts
│   ├── footer.ts                  # 页脚（模型、思考级别、上下文 %、成本）
│   ├── model-selector.ts
│   ├── session-selector.ts
│   ├── tree-selector.ts           # 树导航
│   ├── extension-selector.ts      # 扩展注入的对话框
│   └── ...
└── theme/
    └── theme.ts                   # 1142 行，主题系统
```

### 2.2 InteractiveMode 主编排

构造时组装 TUI 容器树：

```
TUI 根
├── headerContainer      # Logo / 快捷键提示 / CHANGELOG
├── chatContainer        # 消息历史（滚动）
├── pendingMessagesContainer  # steering / follow-up 队列预览
├── statusContainer      # 临时状态消息（2 秒后消失）
├── widgetContainerAbove # 扩展小部件（编辑器上方）
├── editorContainer      # 输入编辑器
├── widgetContainerBelow # 扩展小部件（编辑器下方）
└── footer               # 底部状态行
```

**核心状态**：
- `streamingComponent` / `streamingMessage`——当前正在流式的消息组件
- `pendingTools: Map<toolCallId, ToolExecutionComponent>`
- `toolOutputExpanded`（全局折叠偏好） / `hideThinkingBlock`
- `compactionQueuedMessages`——压缩期间缓冲的用户消息

**事件订阅流程**：
```typescript
session.subscribe((event, signal) => {
  switch (event.type) {
    case "message_start":   创建对应组件，加入 chatContainer
    case "message_update":  调用 streamingComponent.updateContent(partial)
    case "message_end":     finalize 组件
    case "tool_execution_start":  创建 ToolExecutionComponent，记入 pendingTools
    case "tool_execution_update": toolComp.updateProgress(partial)
    case "tool_execution_end":    toolComp.finalize(result)
    // ...
  }
})
```

### 2.3 消息渲染（按类型）

**AssistantMessage** ([assistant-message.ts](packages/coding-agent/src/modes/interactive/components/assistant-message.ts))：
```typescript
updateContent(msg: AssistantMessage) {
  // 按内容块顺序：
  //   1. thinking 块（除非 hideThinkingBlock）→ 斜体 + thinkingText 颜色
  //   2. text 块 → 用 Markdown 组件渲染
  //   3. 若 stopReason === "error" → 红色错误行
  //   4. 若 stopReason === "aborted" → 红色中止提示
}
```

起始输出 OSC 133 终端标记 `\x1b]133;A\x07`，让 iTerm2 / VS Code 终端能识别"prompt 开始"位置，支持跳转到上一条、快速复制输出。

**其他消息类型**：
| 类型 | 组件 | 呈现 |
|------|------|------|
| UserMessage | user-message.ts | 带边框的蓝色块 |
| ToolResult | 内嵌在 ToolExecution 下方 | 结果渲染（见下节） |
| Compaction | compaction-summary-message.ts | 折叠卡片 + 原始 token 统计 |
| BranchSummary | branch-summary-message.ts | 折叠卡片 + "you took this path" 说明 |
| Custom | custom-message.ts | 扩展注册 `renderer` 决定样式 |

**流式更新**：组件在构造时一次加入容器，`updateContent(partial)` 内部调 pi-tui 的 `invalidate()` 触发局部重绘——不重建整个历史树。

### 2.4 工具调用可视化

工具定义可选两个渲染回调：

```typescript
interface ToolDefinition {
  renderCall?(args, theme, ctx): Component       // 调用参数展示
  renderResult?(result, options, theme, ctx): Component  // 结果展示
}
```

**内置工具**：

| 工具 | renderCall | renderResult |
|------|-----------|--------------|
| **bash** | 命令文本（代码块样式） | 截断的输出（带"X 行被截断"提示，可展开） |
| **read** | 文件路径 | 语法高亮代码块（按扩展名） |
| **edit** | 文件路径 + 编辑摘要 | Unified diff（红删绿增，带行号） |
| **write** | 文件路径 | "✓ Created/Overwrote" + 增量高亮 |
| **find** | pattern + path | 结果列表（截断标记） |
| **grep** | pattern + flags | `path:line: content` 格式 |

**无自定义渲染器**时，默认显示 `toolName: (executing...)` → 结果 JSON。

### 2.5 输入编辑器

基于 pi-tui 的 `Editor`，加了：

- **斜杠命令自动补全**：`CombinedAutocompleteProvider` 聚合内置 + 扩展命令 + Skills 命令 + 模板命令
- **模型名补全**：`/model op` → 列可用模型
- **文件路径补全**：`@<path>`，用 `fd` 工具查找
- **图片粘贴**：Ctrl+Shift+V / 剪贴板图片自动附加
- **`!` 前缀直接 bash**：`!ls` 触发 `BashExecution` 组件（不进 LLM 上下文），`!!ls` 进上下文

### 2.6 斜杠命令

**内置**（`core/slash-commands.ts`）：

| 命令 | 作用 |
|------|------|
| `/help` | 帮助与快捷键 |
| `/new` | 新建会话 |
| `/session` | 会话信息 |
| `/compact [instructions]` | 手动压缩 |
| `/fork` | 从当前条目 fork 新会话 |
| `/model` | 打开模型选择器 |
| `/thinking` | 切换思考级别 |
| `/theme` | 切换主题 |
| `/reload` | 重载扩展/技能/模板/主题 |

**调度**（[interactive-mode.ts:1089-1114](packages/coding-agent/src/modes/interactive/interactive-mode.ts)）：

```
用户输入 "/cmd args"
  ↓
_tryExecuteExtensionCommand(cmd, args)
  ├─ 扩展命令？→ 执行扩展 handler
  ├─ /skill:name？→ 执行 skill
  ├─ 内置命令？→ 调 TUI/AgentSession 方法
  └─ Prompt Template？→ 展开变量后作为 prompt 提交
```

### 2.7 状态行 / 页脚 / 小部件

**Footer**（[footer.ts](packages/coding-agent/src/modes/interactive/components/footer.ts)）：
- 左：当前模型 `anthropic/claude-opus-4-7`
- 中：思考级别 `medium`
- 右：上下文 `42%` / 累计成本 `$0.15`

**Status**（临时消息）：
- 如"分支已创建"/"已切换到模型 X"
- 2 秒自动消失

**Widgets**（扩展注入）：
- 位置：`widgetContainerAbove` 或 `widgetContainerBelow`（编辑器两侧）
- 生命周期：`ext.ui.setWidget(key, factory, options)` / 返回 dispose

**Header**：
- 默认：Logo + 快捷键提示 + CHANGELOG 展开
- 扩展可 `ext.ui.setHeader(factory)` 替换

### 2.8 主题系统

**Theme 类**（[theme.ts:342-441](packages/coding-agent/src/modes/interactive/theme/theme.ts)）：

```typescript
class Theme {
  fg(color: ThemeColor, text: string): string   // 前景色
  bg(color: ThemeBg, text: string): string      // 背景色
  bold(text) / italic(text)
  getFgAnsi(color): string                      // 原始 ANSI
  getThinkingBorderColor(level): (s) => string
}
```

**Truecolor vs 256 自动检测**：
- 现代终端（iTerm2 / VS Code / Ghostty / tmux 24-bit）→ Truecolor（`\x1b[38;2;R;G;Bm`）
- Apple Terminal / screen / 受限环境 → 256 色降级（`\x1b[38;5;Nm`）
- 自动通过 `detectColorMode()` 探测

**54 种命名颜色**，分 6 组：
- 核心 UI（11）
- 背景（11）
- Markdown（10）
- Diff（3）
- 语法高亮（9）
- 思考级别（6）
- Bash 模式（1）

**主题加载**：
- 内置：`resources/themes/dark.json` / `light.json`
- 全局自定义：`~/.pi/themes/<name>.json`
- 扩展贡献：`setRegisteredThemes()` 注入

**热重载**：监听主题文件变化，防抖 100ms 后校验并重新加载。

**Proxy-based 全局主题**（精妙）：
- `theme` 是 `Proxy`，指向 `globalThis` 存储的符号
- 原因：tsx + jiti 的多加载器场景下，不同模块 import 的 `theme` 可能指向不同实例
- 效果：全局唯一，切换时所有引用立即看到新主题

**HTML 导出**：主题转 CSS 变量，`ansi256ToHex()` 处理降级颜色转换。

---

## 3. 推荐阅读顺序

| 顺序 | 文件 | 重点 |
|------|------|------|
| 1 | [core/session-manager.ts](packages/coding-agent/src/core/session-manager.ts) | 基础结构、树遍历、JSONL 持久化 |
| 2 | [core/agent-session.ts:112-300](packages/coding-agent/src/core/agent-session.ts) | 事件订阅、自动持久化钩子 |
| 3 | [core/agent-session.ts:1595-2011](packages/coding-agent/src/core/agent-session.ts) | 压缩逻辑、溢出自动恢复 |
| 4 | [core/messages.ts](packages/coding-agent/src/core/messages.ts) | 条目 → LLM 消息转换 |
| 5 | [modes/interactive/interactive-mode.ts:187-600](packages/coding-agent/src/modes/interactive/interactive-mode.ts) | TUI 初始化、事件订阅 |
| 6 | [modes/interactive/components/assistant-message.ts](packages/coding-agent/src/modes/interactive/components/assistant-message.ts) | 消息渲染示例 |
| 7 | [modes/interactive/theme/theme.ts:342-650](packages/coding-agent/src/modes/interactive/theme/theme.ts) | 颜色抽象、Truecolor 降级、Proxy 主题 |
| 8 | [modes/interactive/components/footer.ts](packages/coding-agent/src/modes/interactive/components/footer.ts) | 状态行组装 |

---

## 4. 设计亮点

| 亮点 | 说明 |
|------|------|
| **JSONL 仅追加** | 天然支持分支、无合并冲突、可完整恢复历史；大小通过压缩控制 |
| **leafId + 父指针树** | 分支和导航不改历史，用户可回到任意条目开启新分支 |
| **压缩作为条目而非重写** | `compaction` 也是一条记录，记录 `firstKeptEntryId` 和 `summary`；原始消息仍在文件里 |
| **buildSessionContext 遇 compaction 自动截断** | 上下文构建时过滤老消息，LLM 看不到但磁盘保留 |
| **溢出自动恢复** | LLM 返回 overflow 错误 → 删除错误消息 → 压缩 → `agent.continue()`，对用户透明 |
| **branch_summary 记录被放弃路径** | 树导航时自动生成"你选了另一条路"的摘要，防止上下文空降 |
| **Label 作为独立条目** | 标签支持后期添加，且是不可变的——可安全引用 |
| **cross-cwd fork** | fork 会话可以搬家到新 cwd，历史复制+cwd 更新 |
| **OSC 133 终端集成标记** | assistant 消息头部发 `\x1b]133;A\x07`，iTerm2/VS Code 能识别 prompt 边界 |
| **流式更新增量重绘** | `updateContent(partial)` + `invalidate()`，不重建历史树 |
| **Proxy 全局主题** | 用 `Proxy` 指向 `globalThis` 符号存储，解决 tsx/jiti 多加载器下模块隔离问题 |
| **Truecolor 自动降级** | 现代终端 24-bit 直出，老终端自动 256 色映射 |
| **扩展工具的 renderCall/renderResult** | 工具自带 UI 逻辑而非硬编码在 InteractiveMode；TUI 只负责容器和布局 |
| **主题文件防抖热重载** | 改主题 JSON 文件 100ms 防抖后自动重新加载 |
| **`!` 前缀原生 bash** | 不进 LLM 上下文的快速执行通道，`!!` 才进上下文 |
| **compactionQueuedMessages 缓冲** | 压缩进行时用户还能输入；结束后统一 flush |

---

## 5. 会话和 UI 的协作全景图

```
[用户键入]
    ↓
Editor → autocomplete / 斜杠命令解析
    ↓
InteractiveMode.prompt(text)
    ↓
AgentSession.prompt(text)
  ├─ expandPromptTemplates()   [见 Part 2]
  ├─ sessionManager.appendMessage(userMsg)
  │    └─ _persist() → 文件追加一行
  ├─ 构建 Context（buildSessionContext + 扩展 transformContext）
  ├─ pi-agent-core.runAgentLoop()
  │    ├─ LLM 流式响应
  │    ├─ 工具执行（beforeToolCall → execute → afterToolCall）
  │    └─ 每个 message_end 触发事件
  └─ 事件流回到 InteractiveMode
       ├─ message_start  → new AssistantMessageComponent
       ├─ message_update → component.updateContent(partial)
       ├─ message_end    → component.finalize, sessionManager.appendMessage
       └─ tool_execution_* → ToolExecutionComponent 更新

[自动压缩触发（可选）]
    ↓
_checkCompaction() 检测到占比超阈值
    ↓
AgentSession.compact()
  ├─ session_before_compact 钩子（扩展可拦截）
  ├─ 生成摘要（调用 LLM）
  ├─ sessionManager.appendCompaction(entry)
  └─ emit compaction_end 事件
```

---

## 6. pi-mono 系列总结

至此 [pi-mono](https://github.com/badlogic/pi-mono) 四个核心包 + coding-agent 三部分全部完成：

| 篇目 | 覆盖 |
|------|------|
| [architecture.md](./architecture.md) | 整体 monorepo 架构、包依赖、分层 |
| [pi-ai.md](./notes/pi-ai.md) | 20+ LLM provider 统一层、跨 provider 迁移 |
| [pi-agent-core.md](./notes/pi-agent-core.md) | 外层/内层循环、工具三阶段、事件系统 |
| [pi-tui.md](./notes/pi-tui.md) | 行级差分渲染、Synchronized Update、Kitty 协议 |
| [pi-coding-agent-part1.md](./notes/pi-coding-agent-part1.md) | CLI、4 种运行模式、6 核心工具 |
| [pi-coding-agent-part2.md](./notes/pi-coding-agent-part2.md) | 扩展系统、提示模板、Skills |
| [pi-coding-agent-part3.md](./notes/pi-coding-agent-part3.md) | 会话树 / 压缩 / Interactive UI（本篇） |

**整个 pi 的架构神韵**：

1. **分层干净，每层有自己的抽象**——LLM（pi-ai） ← Agent 循环（pi-agent-core） ← UI（pi-tui） ← 应用（pi-coding-agent）
2. **事件驱动的解耦**——所有跨层通信都是事件流，UI / 持久化 / 扩展是平等的事件订阅者
3. **仅追加 + 树结构**——会话不可变历史支持无痛分支
4. **可插拔扩展的三个层次**——Extensions（代码）、Skills（说明书）、Prompt Templates（文本宏）
5. **细节重在健壮**——Unicode 宽度、跨 provider 工具 ID 规范化、stream 部分消息的就地更新、Proxy 主题、Termux 特例……每个都值得细品

这套架构适合照抄去做自己的 agent 产品——尤其是**会话树 + 仅追加 JSONL + 压缩边界**的组合，是非常优雅的持久化方案。
