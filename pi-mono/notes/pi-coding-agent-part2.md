# pi-coding-agent Part 2：扩展系统、提示模板、Skills

> 分析日期：2026-04-19
> 源码版本：0.67.68
> 源码路径：`packages/coding-agent/src/core/extensions/`（和 skills.ts / prompt-templates.ts）

**本篇范围**：扩展系统（Extensions API + 生命周期 + 钩子）、提示模板（Prompt Templates）、Skills 三大可插拔子系统，以及它们在会话启动时的组装顺序。

**不在本篇**：主入口 / 模式 / 工具见 Part 1；会话管理 / 交互 UI 见 Part 3。

---

## 1. 扩展系统

### 1.1 目录结构

```
packages/coding-agent/src/core/extensions/
├── types.ts      # 1472 行，API 全部接口定义
├── loader.ts     # 558 行，发现 + jiti 加载 + 虚拟模块绑定
├── runner.ts     # 1000+ 行，事件分发 + 生命周期 + 上下文绑定
├── index.ts
└── wrapper.ts
```

### 1.2 三个扩展来源

| 来源 | 位置 | 优先级 |
|------|------|-------|
| **项目本地** | `.pi/extensions/` | 最高（覆盖全局同名） |
| **全局** | `~/.pi/agent/extensions/` | 中 |
| **显式配置** | `settings.json` 中 `extensions` 数组 / CLI `--extension` / `-e` | 按配置顺序 |

发现规则（[loader.ts:474](packages/coding-agent/src/core/extensions/loader.ts)）：
- 直接文件：`*.ts` / `*.js`
- 子目录含 `index.ts` / `index.js`
- 子目录含 `package.json` 且有 `pi.extensions` 字段

### 1.3 jiti 加载 + 虚拟模块绑定（Bun 兼容的关键）

Pi 可编译成 Bun 单文件二进制。二进制里已经打包了 `@sinclair/typebox` / `pi-ai` / `pi-agent-core` / `pi-tui` 等，但扩展在运行时 `import` 它们。解决方案：

```typescript
const VIRTUAL_MODULES = {
  "@sinclair/typebox": _bundledTypebox,
  "@mariozechner/pi-agent-core": _bundledPiAgentCore,
  "@mariozechner/pi-tui": _bundledPiTui,
  "@mariozechner/pi-ai": _bundledPiAi,
  "@mariozechner/pi-ai/oauth": _bundledPiAiOauth,
  "@mariozechner/pi-coding-agent": _bundledPiCodingAgent,
}
```

- **Bun 二进制模式**：`jiti` 拦截 import，直接返回预加载模块（不查文件系统）
- **Node.js / 开发模式**：用 `alias` 映射到 `node_modules`

这让扩展作者用标准的 `import type { ExtensionAPI } from "@mariozechner/pi-coding-agent"` 就行，不管是二进制还是源码发行都能跑。

### 1.4 扩展生命周期

```
loadExtensions()
  ├─ 创建 ExtensionRuntime（所有方法先放 throwing stubs）
  ├─ 对每个扩展路径：
  │   ├─ jiti 动态 import → factory 函数
  │   ├─ 创建 Extension 对象
  │   ├─ 创建 ExtensionAPI（绑定 Extension + runtime + eventBus）
  │   └─ await factory(api)  // 扩展工厂函数执行
  └─ 返回 LoadExtensionsResult { extensions, errors, runtime }

new ExtensionRunner(extensions, runtime, ...)

runner.bindCore(actions, contextActions)
  // ← 关键：把真实实现注入到之前 throwing 的 stubs
  // 此刻所有 registerProvider / setModel 等方法立即可用
```

**为什么用 throwing stubs + bindCore 的延迟绑定模式？**

- 扩展加载时，AgentSession 还没完全就绪（模型、工具、TUI 都没起来）
- 但扩展在 factory 里可能就要调 `registerProvider()` 之类
- 方案：加载阶段把调用排队（写进 `pendingProviderRegistrations`），`bindCore` 时统一 flush
- 好处：扩展间无依赖顺序、循环依赖免疫、即插即用

### 1.5 ExtensionAPI（工厂函数签名）

```typescript
export default async function (pi: ExtensionAPI) {
  pi.on(eventType, handler)
  pi.registerTool(toolDefinition)
  pi.registerCommand(name, options)
  pi.registerShortcut(keyId, options)
  pi.registerFlag(name, options)
  pi.getFlag(name)
  pi.registerProvider(name, config)
  pi.unregisterProvider(name)
  pi.registerMessageRenderer(customType, renderer)
  pi.sendMessage(message)
  pi.sendUserMessage(content)
  pi.appendEntry(customType, data)
  pi.getActiveTools() / setActiveTools(names)
  pi.getAllTools()
  pi.setSessionName(name) / getSessionName()
  pi.setLabel(entryId, label)
  pi.setModel(model)
  pi.setThinkingLevel(level) / getThinkingLevel()
  pi.exec(cmd, args, opts)
  pi.events  // 扩展间通信的 EventBus
}
```

### 1.6 ExtensionContext（事件处理函数收到的上下文）

```typescript
interface ExtensionContext {
  ui: ExtensionUIContext      // UI 交互（下面详解）
  hasUI: boolean              // Print/SDK 模式时为 false
  cwd: string
  sessionManager: ReadonlySessionManager
  modelRegistry: ModelRegistry
  model: Model | undefined
  isIdle(): boolean           // agent 是否空闲
  signal: AbortSignal | undefined
  abort(): void
  hasPendingMessages(): boolean
  shutdown(): void
  getContextUsage(): { tokens, contextWindow, percent }
  compact(options?): void     // 手动触发压缩
  getSystemPrompt(): string
}
```

**ExtensionCommandContext**（命令处理程序专用，多出来的）：

```typescript
waitForIdle(): Promise<void>
newSession(options?): Promise<{ cancelled }>
fork(entryId): Promise<{ cancelled }>
navigateTree(targetId, options?): Promise<{ cancelled }>
switchSession(path): Promise<{ cancelled }>
reload(): Promise<void>
```

### 1.7 ExtensionUIContext（UI 能力，按模式有限支持）

| 类别 | 方法 | Interactive | Print | RPC | SDK |
|------|------|-------------|-------|-----|-----|
| 对话框 | `select` / `confirm` / `input` | ✅ | ❌ | ✅（异步化） | 靠用户实现 |
| 通知 | `notify(msg, type)` | ✅ | 写日志 | ✅ | - |
| 状态 | `setStatus` / `setWorkingMessage` | ✅ | ❌ | ✅（仅 setStatus） | - |
| 小部件 | `setWidget` / `setFooter` / `setHeader` | ✅ | ❌ | ❌ | ❌ |
| 编辑器 | `setEditorText` / `getEditorText` / `setEditorComponent` / `editor(title, prefill)` / `pasteToEditor` | ✅ | ❌ | `editor` 支持 | ❌ |
| 主题 | `theme` / `getAllThemes` / `setTheme` / `getToolsExpanded` | ✅ | ❌ | ❌ | ❌ |
| 自定义 | `custom<T>(factory, options)` 插入任意 TUI 组件 | ✅ | ❌ | ❌ | ❌ |

### 1.8 事件系统（订阅点全集）

| 类别 | 事件名 | 能返回（修改）的内容 |
|------|--------|---------------------|
| **资源** | `resources_discover` | `{ skillPaths?, promptPaths?, themePaths? }` 追加资源 |
| **会话** | `session_start` | - |
| | `session_before_switch` / `session_before_fork` / `session_before_compact` / `session_before_tree` | `{ cancel? }` 阻止操作 |
| | `session_compact` / `session_shutdown` / `session_tree` | - |
| **Agent** | `before_provider_request` | 替换整个 payload |
| | `before_agent_start` | `{ message?, systemPrompt? }` 修改输入 |
| | `agent_start` / `agent_end` / `turn_start` / `turn_end` | - |
| | `context` | `{ messages? }` 修改 LLM 上下文 |
| | `after_provider_response` | - |
| **消息** | `message_start` / `message_update` / `message_end` | - |
| **工具** | `tool_execution_start/update/end` | - |
| | `tool_call` | `{ block?, reason? }` 或**直接修改 `event.input`** |
| | `tool_result` | `{ content?, details?, isError? }` 覆盖结果 |
| **用户** | `input` | `{ action: "continue"\|"transform"\|"handled", text?, images? }` |
| | `user_bash` | `{ operations?, result? }` 劫持直接 bash |
| | `model_select` | - |

**突变式钩子的设计取舍**：`tool_call` 处理函数可以直接 `event.input.command = "safer-version"`，后续处理函数看到的就是改过的参数——高效但无 re-validation，信任链条断裂。

### 1.9 一个最小扩展

```typescript
// ~/.pi/agent/extensions/minimal.ts
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent"
import { Type } from "@sinclair/typebox"

export default async function (pi: ExtensionAPI) {
  // 订阅会话启动
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info")
  })

  // 拦截危险命令
  pi.on("tool_call", async (event, ctx) => {
    if (event.toolName === "bash" && event.input.command?.includes("rm -rf /")) {
      const ok = await ctx.ui.confirm("Dangerous", "Allow rm -rf /?")
      if (!ok) return { block: true, reason: "blocked by user" }
    }
  })

  // 注册自定义工具
  pi.registerTool({
    name: "weather",
    label: "Get Weather",
    description: "Get current weather",
    parameters: Type.Object({ location: Type.String() }),
    async execute(id, params, signal, onUpdate, ctx) {
      return {
        content: [{ type: "text", text: `Weather: sunny in ${params.location}` }],
        details: {},
      }
    },
  })

  // 注册 /greet 命令
  pi.registerCommand("greet", {
    description: "Greet the user",
    handler: async (args, ctx) => ctx.ui.notify(`Hello ${args || "World"}`, "info"),
  })

  // 扩展间通信
  pi.events.on("my-event", (data) => { /* ... */ })
}
```

---

## 2. 提示模板（Prompt Templates）

### 2.1 文件格式

**非常简单**——纯 Markdown + YAML frontmatter，**无 Handlebars**，只支持位置变量替换。

```markdown
---
description: Create a React component
argument-hint: <name> [features...]
---

Create a React component named $1 with features: $@
```

### 2.2 发现来源

| 来源 | 位置 |
|------|------|
| 全局 | `~/.pi/agent/prompts/*.md` |
| 项目 | `.pi/prompts/*.md` |
| 包 | `package.json` 中 `pi.prompts` 字段 |
| 设置 | `settings.json` `prompts` 数组 |
| CLI | `--prompt-template <path>` |

**非递归**——只扫描根目录的 `*.md`。

### 2.3 变量替换（[prompt-templates.ts:68-102](packages/coding-agent/src/core/prompt-templates.ts)）

| 语法 | 含义 |
|------|------|
| `$1`, `$2`, ... | 位置参数 |
| `$@` 或 `$ARGUMENTS` | 所有参数（空格连接） |
| `${@:N}` | 第 N 个起的所有（1-indexed） |
| `${@:N:L}` | 第 N 个起 L 个 |

### 2.4 扩展流程

```typescript
// agent-session.ts 的 prompt() 方法
if (expandPromptTemplates && text.startsWith("/")) {
  const template = findTemplate(name)
  if (template) text = substituteArgs(template.content, args)
}
```

用法：`/component Button "onClick handler" "disabled support"` → 模板内 `$1` = `Button`，`$@` = `onClick handler disabled support`。

**没有继承、没有 include、没有 Handlebars helpers**——就是简单字符串替换。需要更复杂的逻辑就写扩展。

---

## 3. Skills

### 3.1 Skill 是什么

Skill 是一个**目录**，包含 `SKILL.md` 和任意配套脚本/资源：

```
brave-search/
├── SKILL.md          # 说明 + 使用流程（Markdown + YAML frontmatter）
├── search.js
└── content.js
```

### 3.2 Frontmatter 字段

```yaml
---
name: brave-search                # 必需，必须与目录名一致
description: "Search the web..."  # 必需，最多 1024 字符
disable-model-invocation: false   # 可选，true 时从系统提示隐藏
---
```

### 3.3 名称校验规则（[skills.ts:92-116](packages/coding-agent/src/core/skills.ts)）

- 1–64 字符
- 仅小写字母、数字、连字符
- 不以连字符开头/结尾
- 无连续连字符
- 必须匹配父目录名

### 3.4 发现来源

- 全局：`~/.pi/agent/skills/` / `~/.agents/skills/`
- 项目：`.pi/skills/` / `.agents/skills/`（向上查找父目录）
- 包：`package.json` 的 `pi.skills` 字段或 `skills/` 目录
- 设置：`settings.json` `skills` 数组
- CLI：`--skill <path>`

**发现规则**（[skills.ts:172-279](packages/coding-agent/src/core/skills.ts)）：
- 目录含 `SKILL.md` → 作为 skill 根，不再向下递归
- 否则：扫根目录的 `.md` 直接子文件 + 递归子目录找 `SKILL.md`

### 3.5 Progressive Disclosure（关键设计）

Skills 的核心设计是**两级分离**：

1. **系统提示里只包含名称 + 描述 + 路径**（XML）：

```xml
The following skills provide specialized instructions for specific tasks.
Use the read tool to load a skill's file when the task matches its description.

<available_skills>
  <skill>
    <name>brave-search</name>
    <description>Search the web using Brave API</description>
    <location>/abs/path/to/SKILL.md</location>
  </skill>
</available_skills>
```

2. **Agent 按需用 `read` 工具读取完整 SKILL.md**，看到详细步骤后执行

**好处**：
- 系统提示只占少量 token
- Skills 可以任意详细而不污染初始上下文
- Agent 按实际需要拉取，无用的 skill 完全不消耗 token

**触发条件**：`formatSkillsForPrompt` 只在 `read` 工具可用时注入（[system-prompt.ts:162-164](packages/coding-agent/src/core/system-prompt.ts)）——没 `read` 也读不了 SKILL.md，注入没意义。

### 3.6 `/skill:name` 命令

默认启用（`enableSkillCommands: true`）。用户可以主动调用：

```
/skill:brave-search      # 载入并执行
/skill:brave-search "python security vulnerabilities"  # 带参数
```

`disable-model-invocation: true` 的 skill **只能**这样主动触发，不在系统提示里出现。

### 3.7 冲突处理

同名 skill（`skillMap`）：**先找到的获胜**，后来的记诊断警告并丢弃。

---

## 4. 运行时组装顺序

会话启动时：

```
启动 / /reload
  ↓
discoverAndLoadExtensions()
  ├─ 加载 .pi/extensions/ 本地
  ├─ 加载 ~/.pi/agent/extensions/ 全局
  ├─ 加载 settings.json / CLI 显式路径
  └─ 返回 { extensions, runtime }
  ↓
new ExtensionRunner(extensions, runtime, ...)
  ↓
runner.bindCore(actions, contextActions)       ← 真实实现注入
  ↓
emit "resources_discover"                       ← 扩展可返回额外 skill/template/theme 路径
  ↓
loadSkills(allPaths)                            ← 合并默认 + 扩展贡献的路径
  ↓
loadPromptTemplates(allPaths)
  ↓
buildSystemPrompt()
  ├─ 工具列表 + promptSnippet
  ├─ 工具 promptGuidelines
  ├─ 项目 context files (AGENTS.md / CLAUDE.md 等)
  ├─ <available_skills>（仅当有 read 工具）
  └─ 日期 + cwd
  ↓
会话就绪
```

### 4.1 冲突解决规则

| 资源 | 规则 |
|------|------|
| **工具名** | 内置 < SDK `customTools` < 扩展。扩展内**先注册者获胜**（first-wins） |
| **命令** | 扩展命令 > skill `/skill:name` > 模板 `/template-name` |
| **快捷键** | 扩展可覆盖可覆盖的内置；`restrictOverride=true` 的不能；多扩展同快捷键——**后注册者获胜**（last-wins）并记警告 |
| **Skills 同名** | 先找到者获胜（first-wins） |

**为什么工具是 first-wins、快捷键是 last-wins？**
- 工具：first-wins 防止意外覆盖（A 依赖内置 bash，B 扩展注册同名 bash 不会悄悄改变 A 的行为）
- 快捷键：last-wins 让用户的项目级扩展能覆盖全局扩展的默认绑定（项目后加载）

### 4.2 `/reload` 热重载

```typescript
await extensionRunner.emitSessionShutdown()  // 通知扩展
await resourceLoader.reload()                 // 重扫扩展/skills/templates/themes
sessionStartEvent.reason = "reload"
emit("resources_discover", { reason: "reload" })
buildSystemPrompt()                           // 重建
```

**保留的**：会话历史 / 当前流式状态
**重建的**：扩展（jiti 缓存清空） / skills / templates / themes / 系统提示

---

## 5. 推荐阅读顺序

| 顺序 | 文件 | 重点 |
|------|------|------|
| 1 | [extensions/types.ts](packages/coding-agent/src/core/extensions/types.ts) | 全部接口与事件签名；从这里建立 API 形状的 mental model |
| 2 | [extensions/loader.ts](packages/coding-agent/src/core/extensions/loader.ts) | jiti 虚拟模块、发现规则、ExtensionRuntime throwing stubs |
| 3 | [extensions/runner.ts](packages/coding-agent/src/core/extensions/runner.ts) | 事件分发、工具/命令/快捷键冲突解决、bindCore |
| 4 | [core/skills.ts](packages/coding-agent/src/core/skills.ts) | 发现规则、名称校验、XML 格式化 |
| 5 | [core/prompt-templates.ts](packages/coding-agent/src/core/prompt-templates.ts) | 变量替换、扩展流程 |
| 6 | [core/system-prompt.ts](packages/coding-agent/src/core/system-prompt.ts) | 三者如何组装进系统提示 |
| 7 | [core/agent-session.ts](packages/coding-agent/src/core/agent-session.ts) | 完整的会话启动编排 |

---

## 6. 设计亮点

| 亮点 | 说明 |
|------|------|
| **jiti + 虚拟模块** | Bun 二进制里 `import "@mariozechner/pi-ai"` 被 jiti 拦截返回预打包模块，开发模式降级到 `node_modules`——扩展作者无感知 |
| **ExtensionRuntime throwing stubs + bindCore** | 加载期间真实实现还没就绪就用 throwing stubs 占位，`bindCore()` 一次性注入——解决"扩展想调 API 但 API 还没ready"的鸡生蛋问题 |
| **pendingProviderRegistrations 队列** | 扩展在工厂里调 `registerProvider` 时，先进队列，`bindCore` 时 flush 到 ModelRegistry |
| **突变式钩子（tool_call 可改 event.input）** | 高效但无 re-validation——设计上接受信任换性能 |
| **Progressive Disclosure（Skills）** | 系统提示只带名称+描述+路径，Agent 用 `read` 按需拉取 SKILL.md；无 `read` 工具时不注入（没意义） |
| **Skills 目录含 SKILL.md 不再递归** | 允许复杂 skill（多文件、脚本）而不被误扫 |
| **工具 first-wins / 快捷键 last-wins** | 工具保守防意外覆盖，快捷键激进让用户配置生效——规则不同但各自合理 |
| **disable-model-invocation** | 某些 skill 只想让用户主动 `/skill:name` 调，不想让模型自己决策调用——给工具箱里加"非默认选项" |
| **模板零复杂度** | 不用 Handlebars，只有 `$1` / `$@` / `${@:N:L}`——复杂逻辑交给扩展，避免模板沦为迷你编程语言 |
| **EventBus 跨扩展通信** | 所有扩展共享一个 `pi.events: EventBus`，可互相发消息协作（而不是靠全局变量） |
| **resources_discover 事件** | 扩展可以贡献额外的 skill / template / theme 路径——扩展能让其他子系统"长出新资源" |
| **/reload 不丢历史** | 重载扩展和资源但保留消息历史——开发扩展时迭代很顺畅 |

---

## 7. 三大系统对比

| 维度 | Extensions | Prompt Templates | Skills |
|------|------------|------------------|--------|
| **形式** | TS/JS 代码（有完整 API） | Markdown + YAML | Markdown + YAML + 配套文件 |
| **能力** | 工具、命令、快捷键、UI、事件钩子 | 快速插入固定模板文本 | 提供复杂工作流说明 |
| **触发** | 事件驱动 / 手动注册命令 | `/template-name args` | 模型自主调用（read SKILL.md）或 `/skill:name` |
| **上下文占用** | 看扩展实现 | 展开后才进 LLM | 描述进系统提示，正文按需加载 |
| **适合场景** | 权限控制、UI 定制、第三方集成 | 常用 prompt 模板化 | 特定任务的工作流指南（带代码/资源） |
| **编写难度** | 高（写代码） | 极低（写 Markdown） | 低-中（写 Markdown，可选配套脚本） |

**选型经验**：
- 只是想复用一段 prompt 文本 → Template
- 想让 agent 学会一个带具体步骤的工作流 → Skill
- 想改变 agent 的行为、加工具、挂钩子 → Extension
