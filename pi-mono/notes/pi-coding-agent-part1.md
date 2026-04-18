# pi-coding-agent Part 1：主入口、运行模式、核心工具

> 分析日期：2026-04-19
> 源码版本：0.67.68
> 源码路径：`packages/coding-agent/src/`
> 路径相对于 pi-mono 仓库根目录

**本篇范围**：CLI 入口链、参数解析、4 种运行模式（Interactive / Print / RPC / SDK）、6 个核心工具（read / bash / edit / write / find / grep）及其与 pi-agent-core 的集成。

**不在本篇**：扩展系统、提示模板、Skills、会话管理内部、Interactive 模式的 TUI 渲染细节——放到 Part 2 / Part 3。

---

## 1. 目录速览

```
packages/coding-agent/src/
├── cli.ts                    # 18 行 shebang 入口
├── main.ts                   # 737 行，参数解析、模式分派
├── cli/
│   ├── args.ts              # 手写线性扫描解析器
│   ├── file-processor.ts    # @file 参数预处理
│   ├── initial-message.ts   # stdin/prompt 初始消息
│   ├── list-models.ts       # --list-models
│   └── session-picker.ts
├── modes/
│   ├── print-mode.ts        # 167 行，文本/JSONL 输出
│   ├── rpc/rpc-mode.ts      # 733 行，JSON-RPC 协议
│   └── interactive/
│       └── interactive-mode.ts  # 主 TUI 应用
├── core/
│   ├── sdk.ts               # createAgentSession 公开 API
│   ├── agent-session.ts     # 101KB，核心会话对象
│   ├── agent-session-runtime.ts
│   ├── agent-session-services.ts
│   ├── tools/               # 6 工具 + 辅助
│   │   ├── read.ts
│   │   ├── bash.ts
│   │   ├── edit.ts
│   │   ├── edit-diff.ts     # 模糊匹配 + diff 算法
│   │   ├── write.ts
│   │   ├── find.ts
│   │   ├── grep.ts
│   │   ├── ls.ts
│   │   ├── truncate.ts
│   │   ├── path-utils.ts
│   │   └── file-mutation-queue.ts
│   ├── extensions/          # Part 2 范围
│   ├── session-manager.ts   # Part 3 范围
│   └── ...
└── utils/
```

---

## 2. CLI 入口链

### 2.1 `bin/pi` → `cli.ts`（18 行）

```typescript
#!/usr/bin/env node
process.title = "pi";                         // ps 显示用
process.env.PI_CODING_AGENT = "true";
process.emitWarning = (() => {}) as any;      // 静音 deprecation 警告

import { EnvHttpProxyAgent, setGlobalDispatcher } from "undici";
setGlobalDispatcher(new EnvHttpProxyAgent()); // 尊重 HTTP(S)_PROXY env

import { main } from "./main.js";
main(process.argv.slice(2));
```

### 2.2 `main.ts` 流程

```
main(args)
  ├─ 1. 迁移 + 快速路径检查（--help / --version / --offline / --export）
  ├─ 2. parseArgs(args)              → Args 对象
  ├─ 3. resolveAppMode(parsed, isTTY) → "rpc" | "json" | "print" | "interactive"
  ├─ 4. SessionManager 创建           → 新建 / 继续 / 恢复 / 分支
  ├─ 5. createAgentSessionRuntime()   → 模型/工具/诊断 汇总
  └─ 6. 按模式分派：
       ├─ "rpc"         → runRpcMode(runtime)
       ├─ "interactive" → new InteractiveMode(runtime, opts).run()
       └─ "print"/"json"→ runPrintMode(runtime, { mode, ... })
```

### 2.3 参数解析（args.ts）

**不用 commander/yargs，纯手写**。好处是零依赖、完全控制未知 flag 收集：

```typescript
for (let i = 0; i < args.length; i++) {
  const arg = args[i]
  if (arg === "--help") { ... }
  else if (arg === "--mode" && i + 1 < args.length) { ... }
  ...
  else if (arg.startsWith("--")) {
    // 未知 flag 收集，留给扩展消费
    const [key, value] = arg.slice(2).split("=")
    result.unknownFlags.set(key, value ?? "")
  }
}
```

**Args 接口主要字段**（按类别）：

| 类别 | 字段 | 说明 |
|------|------|------|
| 模型 | `provider` / `model` / `apiKey` / `thinking` / `models` | 模型指定与循环候选 |
| 会话 | `continue` / `resume` / `session` / `fork` / `sessionDir` / `noSession` | 会话定位与分支 |
| 工具 | `tools` / `noTools` | 工具白名单 |
| 提示 | `systemPrompt` / `appendSystemPrompt` | 系统提示覆盖与追加 |
| 模式 | `mode` / `print` / `export` | 输出模式 |
| 资源 | `extensions` / `skills` / `promptTemplates` / `themes` / `noContextFiles` | 扩展、Skills、模板、主题 |
| 消息 | `messages[]` / `fileArgs[]` | 位置参数和 `@file` 前缀 |
| 扩展 | `unknownFlags: Map` | 未知 flag 交给扩展 |

### 2.4 模式选择

```typescript
function resolveAppMode(parsed: Args, stdinIsTTY: boolean): AppMode {
  if (parsed.mode === "rpc") return "rpc"
  if (parsed.mode === "json") return "json"
  if (parsed.print || !stdinIsTTY) return "print"   // stdin 被 pipe 自动切
  return "interactive"
}
```

**精巧点**：stdin 不是 TTY（被 pipe）时自动切 print，`echo "..." | pi` 开箱即用。

---

## 3. 四种运行模式

### 3.1 Interactive（TUI，主角）

**入口**：[modes/interactive/interactive-mode.ts](packages/coding-agent/src/modes/interactive/interactive-mode.ts)

- **输入**：CLI 初始消息（可选） + TUI 编辑器交互输入 + 键盘事件
- **输出**：pi-tui 渲染的完整 UI（代码高亮、diff、流式更新、消息树）
- **特有**：
  - 会话自动保存（间隔 + 退出保存）
  - `Ctrl+P` 模型循环
  - `/slash` 命令（Skills / 扩展）
  - 工具确认交互
  - 消息树分支导航

### 3.2 Print（脚本友好）

**入口**：[modes/print-mode.ts](packages/coding-agent/src/modes/print-mode.ts)（167 行）

**两个子格式**：

| `--mode text` | `--mode json` |
|---------------|---------------|
| 只输出最后一条 assistant 消息的文本 | 所有事件以 JSONL 流输出 |
| 适合脚本提取答案、管道处理 | 适合编程集成、实时处理 |

JSON 输出格式（每行一个事件）：

```
{"type":"session_header","sessionId":"...","model":{...}}
{"type":"message_start","role":"user",...}
{"type":"content_block_start","contentBlockIndex":0,"contentBlock":{"type":"text"}}
{"type":"content_block_delta","delta":{"type":"text_delta","text":"..."}}
{"type":"message_stop",...}
```

**关键点**：
- 信号处理（SIGTERM/SIGHUP → 优雅关闭）
- 追踪分离的子进程（bash 工具孤儿清理）
- 原始 stdout 保护（扩展不会污染管道输出）

### 3.3 RPC（进程集成）

**入口**：[modes/rpc/rpc-mode.ts](packages/coding-agent/src/modes/rpc/rpc-mode.ts)（733 行）

**协议**：自定义 JSON-RPC（非标准 2.0，概念类似）。stdin 读命令 JSONL，stdout 写事件 + 响应 JSONL。

**命令分类**（详见 [rpc-types.ts](packages/coding-agent/src/modes/rpc/rpc-types.ts)）：

| 类别 | 命令 |
|------|------|
| 提示 | `prompt` / `steer` / `follow_up` / `abort` / `new_session` |
| 查询 | `get_state` / `get_messages` / `get_session_stats` / `get_available_models` / `get_commands` / `get_fork_messages` / `get_last_assistant_text` |
| 模型 | `set_model` / `cycle_model` / `set_thinking_level` / `cycle_thinking_level` |
| 队列 | `set_steering_mode` / `set_follow_up_mode` |
| 压缩 | `compact` / `set_auto_compaction` / `set_auto_retry` / `abort_retry` |
| Bash | `bash` / `abort_bash` |
| 会话 | `switch_session` / `fork` / `set_session_name` / `export_html` |

**响应格式**：

```typescript
interface RpcResponse {
  id?: string
  type: "response"
  command: RpcCommand["type"]
  success: boolean
  data?: object | null
  error?: string
}
```

**扩展 UI 异步 RPC 化**（精妙设计）：

```typescript
// 扩展调用 ext.select(title, options) 时，RPC 发出：
output({ type: "extension_ui_request", id: uuid, method: "select", ... })

// pendingExtensionRequests: Map<id, { resolve, reject }>
// 客户端处理完 UI 交互后回应：
{ type: "extension_ui_response", id, value }
// RPC 收到就 resolve 对应 Promise
```

支持的扩展 UI 方法：`select` / `confirm` / `input` / `notify` / `setStatus` / `editor`。不支持 TUI 专有方法（`setWorkingMessage`、自定义 footer 等）。

### 3.4 SDK（编程嵌入）

**入口**：[core/sdk.ts](packages/coding-agent/src/core/sdk.ts)

```typescript
const { session, extensionsResult, modelFallbackMessage } = await createAgentSession({
  cwd?, agentDir?, authStorage?, modelRegistry?,
  model?, thinkingLevel?, scopedModels?,
  tools?, customTools?,                    // 覆盖或扩展工具
  resourceLoader?, sessionManager?,
  settingsManager?, sessionStartEvent?,
})
```

返回的 `AgentSession` 对象主要 API：

```typescript
session.prompt(message, { images? })
session.steer(message, images?)            // 改变当前生成
session.followUp(message, images?)         // 继续思路
session.abort()

session.setModel(model) / cycleModel()
session.setThinkingLevel(level) / cycleThinkingLevel()

session.compact(instructions?)             // 压缩历史
session.exportToHtml(outputPath)
session.navigateTree(targetId, options)    // 消息树导航
session.subscribe(callback)                // 事件订阅
```

**四种模式的对比**：

| 维度 | Interactive | Print | RPC | SDK |
|------|-------------|-------|-----|-----|
| 场景 | 日常开发 | 脚本/管道 | IDE / 第三方宿主集成 | 库嵌入 |
| 输入 | TUI 键盘 | stdin / args | JSONL stdin | 程序调用 |
| 输出 | TUI 渲染 | 文本 / JSONL | JSONL 事件+响应 | AgentSession 对象 |
| 会话保存 | 自动 | 可选 | 客户端控制 | 可选 |
| 扩展 UI | 原生支持 | 不支持 | 异步 RPC 化 | 自定义 ResourceLoader |

---

## 4. 六个核心工具

所有工具实现 `ToolDefinition` 接口（扩展系统定义），运行时包装成 pi-agent-core 的 `AgentTool`：

```typescript
interface ToolDefinition<Schema, Details, RenderState> {
  name: string
  label: string
  description: string
  promptSnippet: string          // LLM 看到的使用建议
  promptGuidelines?: string[]
  parameters: TSchema            // TypeBox

  execute(toolCallId, input, signal?, onUpdate?, ctx?)
    : Promise<{ content: Content[]; details?: Details }>

  renderCall?(args, theme, ctx): Component       // TUI 渲染
  renderResult?(result, options, theme, ctx): Component
  prepareArguments?(input: unknown): Schema
}
```

### 4.1 Read（[read.ts](packages/coding-agent/src/core/tools/read.ts)，270 行）

**参数**：`{ path; offset?; limit? }`

**流程**：
1. 路径解析（相对 → cwd）+ 可读性检查
2. 检测 MIME 类型 → 分支
3. **图片路径**：Base64 编码 + 可选缩放（默认 2000×2000）+ 返回图片 content
4. **文本路径**：
   - 应用 `offset` / `limit`
   - `truncateHead()` 裁剪：200 行 或 256 KB（先到为准）
   - 输出续读提示（`Use offset=N to continue`）

**边界**：首行 > 256 KB → 拒绝，建议用 bash `head`。

**无流式**（同步）。

### 4.2 Bash（[bash.ts](packages/coding-agent/src/core/tools/bash.ts)，451 行）

**参数**：`{ command; timeout? }`

**执行**：
1. 在 cwd spawn detached shell 进程
2. 双层缓冲管理输出：
   ```
   stdout/stderr 流
     ↓
   Rolling tail 缓冲（内存最近 512 KB）
     ↓ 超过 256 KB 阈值
   临时文件 /tmp/pi-bash-*.log
     ↓
   onUpdate() 流式回调
     ↓ 最终
   truncateTail() → 最后 200 行 或 256 KB
   ```
3. 超时 / 中止：`setTimeout` + `AbortSignal` → 杀进程树

**关键特性**：
- **唯一支持 `onUpdate` 流式的工具**（其他都是原子执行）
- `trackDetachedChildPid` 避免孤儿进程
- `spawnHook` 让扩展注入环境/前置命令
- 完整输出保留在临时文件，模型只看截断版

### 4.3 Edit（[edit.ts](packages/coding-agent/src/core/tools/edit.ts) + [edit-diff.ts](packages/coding-agent/src/core/tools/edit-diff.ts)）

**参数**：

```typescript
{
  path: string
  edits: Array<{ oldText: string; newText: string }>  // 支持批量
}
```

**算法**（核心在 `edit-diff.ts`）：

```
1. 检测行尾（\r\n vs \n）+ 处理 BOM
2. 规范化到 LF 统一处理
3. 对每个 edit 调用 fuzzyFindText(content, oldText)：
   ├─ 先尝试精确 indexOf(oldText)
   └─ 失败 → normalizeForFuzzyMatch 后再试：
       - NFKC Unicode 规范化
       - 去尾部空格
       - 智能引号 " " → ASCII " "
       - em-dash / en-dash → 连字符
       - 特殊空格 → 普通空格
4. 歧义检测：
   ├─ 多处匹配 → 拒绝，要求更多上下文
   ├─ 无匹配 → 拒绝，提示语法
   └─ oldText === newText → 拒绝
5. 重叠检测：
   - 按匹配位置排序
   - 检查相邻 edit 是否重叠
6. 逆序应用（保持偏移稳定）
7. 恢复原行尾和 BOM
8. 生成 unified diff（diff 库 diffLines，4 行上下文）
```

**并发安全**：`FileMutationQueue` 对同文件序列化，防止交叉 edit/write 竞态。

**事务性**：任一 edit 失败 → 全部回滚。

### 4.4 Write（[write.ts](packages/coding-agent/src/core/tools/write.ts)，286 行）

**参数**：`{ path; content }`

**流程**：`mkdir -p` 父目录 → 直接覆写。进入 `FileMutationQueue`。

**渲染亮点**：增量代码高亮缓存——前 50 行完整高亮，之后单行增量，流式 update 也能显示实时进度。

### 4.5 Find（[find.ts](packages/coding-agent/src/core/tools/find.ts)，375 行）

**参数**：`{ pattern; path?; limit? }`

**实现**：内置 `fd` 工具（按需下载）：

```
fd --glob --color=never --hidden --no-require-git --max-results=1000
```

- 路径含 `/` → `--full-path`，否则匹配 basename
- 忽略 `node_modules` / `.git`
- POSIX 路径规范化
- 截断 256 KB / 1000 结果

### 4.6 Grep（[grep.ts](packages/coding-agent/src/core/tools/grep.ts)，389 行）

**参数**：`{ pattern; path?; glob?; ignoreCase?; literal?; context?; limit? }`

**实现**：`ripgrep --json --line-number` 流式解析：

```
rg --json --line-number --color=never --hidden --glob <glob>
  ↓ 逐行 JSON 解析
  ↓ 记录 (file, line, match)
  ↓ 达到 limit → kill 进程
  ↓ 需要上下文时读取文件（带 LRU 缓存）
  ↓ 格式化：file:line: match 或 file-line- context
```

**截断**：单行 > 100 字符标记截断，防止模型看到超长行。

### 4.7 工具特性对比

| 工具 | 流式 | 并发保护 | 安全机制 | 外部依赖 |
|------|------|---------|---------|---------|
| read | ❌ | - | 路径验证、大小限制 | - |
| bash | ✅ | - | 超时、进程树追踪 | shell |
| edit | ❌ | FileMutationQueue | 模糊匹配歧义拒绝、事务 | - |
| write | ❌ | FileMutationQueue | 路径验证 | - |
| find | ❌ | - | 结果数限制 | fd（自动下载） |
| grep | ❌ | - | 行长限制、结果数限制 | ripgrep |

---

## 5. 与 pi-agent-core 的集成

### 5.1 工具集组装（[tools/index.ts](packages/coding-agent/src/core/tools/index.ts)）

```typescript
export const codingTools = [readTool, bashTool, editTool, writeTool]
export const readOnlyTools = [readTool, grepTool, findTool, lsTool]

// 带 cwd 的工厂
export function createCodingTools(cwd, options?): Tool[] {
  return [
    createReadTool(cwd, options?.read),
    createBashTool(cwd, options?.bash),
    createEditTool(cwd),
    createWriteTool(cwd),
  ]
}
```

### 5.2 包装成 AgentTool

```typescript
export function wrapToolDefinition(def: ToolDefinition): AgentTool {
  return {
    name: def.name,
    description: def.description,
    parameters: def.parameters,
    execute: def.execute,     // 直接委托
  }
}
```

`ToolDefinition` 比 `AgentTool` 多出的字段（`renderCall` / `renderResult` / `promptSnippet` 等）是 UI 层和提示装配用的，对 agent 核心循环透明。

### 5.3 交给 AgentSession

在 [sdk.ts](packages/coding-agent/src/core/sdk.ts) 里：

```typescript
let tools = options.tools ?? codingTools
if (options.customTools) { /* 扩展系统合并 */ }

const session = await createAgentSessionFromServices({ tools, customTools, ... })
```

---

## 6. 推荐阅读顺序（Part 1 范围）

| 顺序 | 文件 | 重点 |
|------|------|------|
| 1 | [cli/args.ts](packages/coding-agent/src/cli/args.ts) | CLI 表面一览 |
| 2 | [main.ts](packages/coding-agent/src/main.ts) | 分派流程与模式选择 |
| 3 | [modes/print-mode.ts](packages/coding-agent/src/modes/print-mode.ts) | 最简单的模式，理解事件流 |
| 4 | [modes/rpc/rpc-mode.ts](packages/coding-agent/src/modes/rpc/rpc-mode.ts) | RPC 协议 + 扩展 UI 异步化 |
| 5 | [core/tools/edit-diff.ts](packages/coding-agent/src/core/tools/edit-diff.ts) | 模糊匹配 + 重叠检测算法 |
| 6 | [core/tools/bash.ts](packages/coding-agent/src/core/tools/bash.ts) | 双层缓冲 + 流式 onUpdate |
| 7 | [core/sdk.ts](packages/coding-agent/src/core/sdk.ts) | 公开 API 全貌 |

---

## 7. 设计亮点

| 亮点 | 说明 |
|------|------|
| **stdin 非 TTY 自动切 print 模式** | `echo "..." \| pi` 开箱即用，不用 `-p` |
| **手写 arg 解析收集未知 flag** | 扩展可以读 `unknownFlags: Map`，不用改主解析器 |
| **Edit 两级匹配（精确 → 模糊）** | NFKC + 智能引号 + em-dash 归一化，容忍 LLM 小偏差 |
| **Edit 批量 + 事务 + 逆序应用** | 按匹配位置逆序写入，偏移稳定；任一失败全回滚 |
| **FileMutationQueue 文件级序列化** | 同文件 edit/write 串行，跨文件并发，避免竞态 |
| **Bash 双层缓冲** | 内存 512 KB 滚动尾 + 临时文件全量，防 OOM 同时保留完整输出 |
| **Bash 是唯一支持 onUpdate 的工具** | 流式推送滚动尾，UI 可实时看到输出 |
| **RPC 扩展 UI 异步 RPC 化** | `pendingRequests: Map<id, {resolve, reject}>`，扩展的 `select/confirm/input` 全转成 request/response 对 |
| **fd / ripgrep 按需自动下载** | 不依赖系统预装，保证工具能跑 |
| **Grep 达到 limit 主动 kill rg** | 超大仓库也能快速返回 |
| **跨项目会话分支询问** | 恢复别的项目会话会提示 fork，不污染当前 cwd |
| **Ctrl+P 模型循环受 --models 约束** | 启动时限定候选集合，循环只在集合内转 |

---

## 8. 几个脑洞题（自测理解）

- 为什么 Edit 不用 AST 替换，而用字符串（带模糊）匹配？
  - 答：语言无关、实现简单、LLM 输出就是文本上下文。模糊匹配兜底字符串小偏差。
- 为什么 Bash 是唯一流式工具？
  - 答：只有它的执行时间可能很长且输出有增量价值（编译/测试）。其他工具（read/edit/write/find/grep）要么快要么原子。
- Print JSON 和 RPC 事件有什么区别？
  - 答：Print 是单向事件流（pi → 客户端），RPC 是双向（客户端能发命令，pi 会回 response 和事件）。
- 扩展在 Print 模式为什么不能用 `select` 等 UI 方法？
  - 答：Print 模式没有双向通道，扩展只能走单向事件或直接读文件。RPC 才把 UI 方法通过请求/响应 id 异步化。
