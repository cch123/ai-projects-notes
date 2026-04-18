# pi-agent-core：通用 Agent 运行时

> 分析日期：2026-04-19
> 源码版本：0.67.68
> 源码路径：`packages/agent/src/`
> 路径相对于 pi-mono 仓库根目录

**定位**：位于 `pi-ai`（LLM 层）和上层应用（`pi-coding-agent` / `pi-mom` / `pi-pods`）之间的通用 Agent 运行时。职责：Agent 循环、工具执行、事件流、消息状态管理。

---

## 1. 目录结构

包非常精简，仅 5 个 TS 文件：

```
packages/agent/src/
├── index.ts         # 8 行，公开导出
├── types.ts         # 350 行，类型系统（事件、消息、工具、配置）
├── agent.ts         # 544 行，Agent 类：有状态包装 + 生命周期
├── agent-loop.ts    # 640 行，无状态核心循环 + 工具执行（重点）
└── proxy.ts         # 341 行，代理服务器转发 StreamFn
```

依赖只有一个：`@mariozechner/pi-ai`。

---

## 2. 公共 API

[index.ts](packages/agent/src/index.ts) 导出 4 个模块的全部内容。关键 API：

**高级（有状态）**：
```typescript
new Agent(options)
  .prompt(input, images?)     // 开始新轮
  .continue()                 // 从当前转录继续
  .subscribe(listener)        // 订阅事件，返回取消函数
  .state                      // 只读 AgentState
```

**低级（无状态）**：
```typescript
runAgentLoop(prompts, context, config, emit, signal, streamFn?)
runAgentLoopContinue(context, config, emit, signal, streamFn?)
agentLoop(...)           // 返回 EventStream 而非 emit
agentLoopContinue(...)
```

**特殊**：
```typescript
streamProxy(baseUrl, ...)   // 通过 HTTP 代理转发 LLM 请求
```

---

## 3. Agent 循环（核心）

### 3.1 循环结构：外层 + 内层

这是 `runLoop()` 的精华（[agent-loop.ts:155-232](packages/agent/src/agent-loop.ts)）：

```typescript
while (true) {                                              // ← 外层：follow-up
  let hasMoreToolCalls = true
  while (hasMoreToolCalls || pendingMessages.length > 0) {  // ← 内层：工具 + steering
    await emit("turn_start")

    // 1. 注入 steering / follow-up 消息（若有）
    for msg of pendingMessages: emit(message_start/end), push to context
    pendingMessages = []

    // 2. 流式调用 LLM
    const message = await streamAssistantResponse(...)

    // 3. 停止条件
    if (stopReason === "error" | "aborted") {
      emit("turn_end"); emit("agent_end"); return
    }

    // 4. 执行工具调用
    const toolCalls = message.content.filter(c => c.type === "toolCall")
    hasMoreToolCalls = toolCalls.length > 0
    if (hasMoreToolCalls) {
      const toolResults = await executeToolCalls(...)
      for r of toolResults: push to context and newMessages
    }

    await emit("turn_end", { message, toolResults })

    // 5. 拉取 steering（在工具后、下一轮前）
    pendingMessages = await config.getSteeringMessages?.() ?? []
  }

  // 内层退出：无工具调用且无 steering
  const followUps = await config.getFollowUpMessages?.() ?? []
  if (followUps.length > 0) {
    pendingMessages = followUps
    continue  // ← 外层重入，内层重启
  }
  break
}
await emit("agent_end", newMessages)
```

**两级队列的意图**：
- **Steering**（内层）：UI 高优先级消息，工具执行后立刻注入，不让 agent 停下来
- **Follow-up**（外层）：只在 agent 将要停止时检查，低优先级后续任务

### 3.2 Agent 实例状态

```typescript
class Agent {
  _state: MutableAgentState {
    systemPrompt, model, thinkingLevel,
    tools: AgentTool[],           // getter 返回引用；setter 浅拷贝
    messages: AgentMessage[],
    isStreaming: boolean,
    streamingMessage?: AgentMessage,   // 正在流式的部分消息
    pendingToolCalls: Set<string>,     // 执行中工具 ID
    errorMessage?: string,
  }
  listeners: Set<Listener>
  steeringQueue / followUpQueue: PendingMessageQueue
  activeRun?: { promise, controller }  // 当前运行
}
```

### 3.3 LLM 调用（`streamAssistantResponse`）

[agent-loop.ts:238-331](packages/agent/src/agent-loop.ts)，关键步骤：

```typescript
// 1. 上下文级转换（AgentMessage → AgentMessage）
if (config.transformContext) {
  messages = await config.transformContext(messages, signal)
}

// 2. LLM 级转换（AgentMessage → pi-ai Message）
const llmMessages = await config.convertToLlm(messages)

// 3. 动态解析 API key（支持 OAuth 过期）
const apiKey = await config.getApiKey?.(provider) ?? config.apiKey

// 4. 调用 streamSimple（默认）或自定义 streamFn
const response = await streamFunction(config.model, llmContext, {...})

// 5. 消费事件，就地更新 partial message
for await (const event of response) {
  switch (event.type) {
    case "start":
      context.messages.push(partialMessage)
      emit("message_start", partialMessage)
    case "text_*" | "thinking_*" | "toolcall_*":
      context.messages[last] = event.partial  // ← 就地替换
      emit("message_update", { event, message })
    case "done" | "error":
      const final = await response.result()
      context.messages[last] = final
      emit("message_end", final)
      return final
  }
}
```

**关键技巧**：数组**就地索引替换**而非 `[...messages.slice(0,-1), new]`——每个 delta 都避免深拷贝，大幅降低 GC 压力。

### 3.4 工具执行

**模式选择**（[agent-loop.ts:336-351](packages/agent/src/agent-loop.ts)）：

```typescript
const hasSequentialTool = toolCalls.some(tc =>
  tools.find(t => t.name === tc.name)?.executionMode === "sequential"
)
if (config.toolExecution === "sequential" || hasSequentialTool) {
  return executeToolCallsSequential(...)
}
return executeToolCallsParallel(...)
```

单个工具可以用 `executionMode: "sequential"` 强制整批串行（比如 bash，防止并发破坏 shell 状态）。

**并行模式的精妙设计**：

```typescript
// 准备阶段：顺序 await（验证 + beforeToolCall 钩子）
for (const toolCall of toolCalls) {
  const prep = await prepareToolCall(...)
  if (prep.kind === "immediate") results.push(prep.result)
  else runnableCalls.push(prep)
}

// 执行阶段：并发
const running = runnableCalls.map(p => ({
  prepared: p,
  execution: executePreparedToolCall(p, signal, emit)  // 立即启动
}))

// 收集阶段：按源顺序 await
for (const r of running) {
  const executed = await r.execution
  results.push(await finalizeExecutedToolCall(r.prepared, executed, ...))
}
```

- 准备串行：确保 `beforeToolCall` 阻塞行为确定
- 执行并发：性能
- 结果按助手消息中的原始顺序回填：重放可确定

### 3.5 工具三阶段

```
prepareToolCall(toolCall)
  ├─ 查工具定义 → 找不到 = ImmediateOutcome(error)
  ├─ tool.prepareArguments?.() → 修正 API 变形
  ├─ validateToolArguments()（来自 pi-ai，TypeBox + AJV）
  ├─ config.beforeToolCall?.() → { block: true } 可拦截
  └─ 返回 PreparedToolCall
  
executePreparedToolCall(prepared)
  ├─ emit("tool_execution_start")
  ├─ try: tool.execute(id, args, signal, onUpdate)
  │   └─ onUpdate 回调 → emit("tool_execution_update") 可多次
  └─ catch: createErrorToolResult()
  
finalizeExecutedToolCall(prepared, executed)
  ├─ config.afterToolCall?.() → 可覆盖 content/details/isError
  ├─ emit("tool_execution_end")
  ├─ 构造 ToolResultMessage
  ├─ emit("message_start", toolResult)
  └─ emit("message_end", toolResult)
```

`afterToolCall` 是**字段级替换**（`override.content ?? result.content`），不是深合并。

### 3.6 错误处理层次

| 层级 | 位置 | 策略 |
|------|------|------|
| 工具异常 | `executePreparedToolCall` try/catch | 转 `createErrorToolResult`，Agent 下一轮看到 |
| 验证失败 | `prepareToolCall` | 转 `ImmediateOutcome(error)`，不调工具 |
| 钩子异常 | Before/After 内 try/catch | 转错误工具结果 |
| LLM 失败 | `stopReason` 编码在流事件 | 循环检测 `error`/`aborted` 后 `agent_end` |
| 运行时崩溃 | `runWithLifecycle` 外层 try | 构造 error message，push 到转录，emit `agent_end` |

---

## 4. 事件系统

### 4.1 完整事件类型（11 种）

```typescript
type AgentEvent =
  | { type: "agent_start" }
  | { type: "agent_end"; messages: AgentMessage[] }
  | { type: "turn_start" }
  | { type: "turn_end"; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_execution_start"; toolCallId; toolName; args }
  | { type: "tool_execution_update"; toolCallId; toolName; args; partialResult }
  | { type: "tool_execution_end"; toolCallId; toolName; result; isError }
```

### 4.2 顺序保证

```
agent_start
  turn_start
    message_start(user)
    message_end(user)
    message_start(assistant, partial)
      message_update × N (text_delta / thinking_delta / toolcall_delta)
    message_end(assistant, final)
    [每个 tool call:]
      tool_execution_start
        tool_execution_update × M (可选)
      tool_execution_end
      message_start(toolResult)
      message_end(toolResult)
  turn_end
  [若有更多轮 → 重复 turn_start...]
agent_end
```

**不变量**：
- `message_start` 一定先于 `message_end`
- 同一工具的 `execution_start` → `execution_end` 连续（但并发工具的事件可以交错）
- `tool_execution_update` 夹在 start/end 之间
- 工具结果的 `message_*` 在 `tool_execution_end` 之后

### 4.3 订阅机制

```typescript
subscribe(listener: (event, signal) => Promise<void> | void): () => void {
  this.listeners.add(listener)
  return () => this.listeners.delete(listener)
}
```

**监听器特性**：
- 返回取消函数
- 可异步（被 await）
- 按订阅顺序串行执行
- 接收活跃 AbortSignal（可观察但不拥有）

**状态同步**：`Agent.processEvents` 在派发事件给监听器**之前**先更新 `_state`，所以监听器读 `agent.state` 时已是最新。

---

## 5. 工具抽象

### 5.1 `AgentTool` 接口

```typescript
interface AgentTool<TParameters extends TSchema, TDetails = any>
  extends Tool<TParameters> {  // 继承自 pi-ai
  label: string
  prepareArguments?: (args: unknown) => Static<TParameters>
  execute: (
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: (partial: AgentToolResult<TDetails>) => void
  ) => Promise<AgentToolResult<TDetails>>
  executionMode?: "sequential" | "parallel"
}
```

`AgentToolResult` 包含 `content`（LLM 看得到）+ `details`（UI 用，不进 LLM）。

### 5.2 参数管线

```
toolCall.arguments (untrusted, from LLM)
  ↓ tool.prepareArguments?.()    (变形修正：如把字符串 "true" 转布尔)
  ↓ validateToolArguments()      (pi-ai 提供，TypeBox schema + AJV)
  ↓ validatedArgs (typed Static<TParameters>)
```

### 5.3 流式工具结果

工具可以通过 `onUpdate` 推送多次部分结果：

```typescript
execute: async (id, args, signal, onUpdate) => {
  for (const chunk of stream) {
    onUpdate({ content: [{type:"text", text: partial}], details: { progress: 50 } })
  }
  return { content: [...], details: {...} }  // 最终
}
```

每次 `onUpdate` → emit `tool_execution_update` 事件。最终 `return` → emit `tool_execution_end`。

---

## 6. 消息状态管理

### 6.1 副本语义

```typescript
get messages() { return messages }         // 返回引用
set messages(next) { messages = next.slice() }  // 浅拷贝输入
```

防止外部代码篡改内部数组，但读取便宜。

### 6.2 两级转换

```
AgentMessage[] (用户/助手/工具结果/自定义)
  ↓ transformContext?()  (可选)：修剪长对话、注入外部上下文
AgentMessage[] (转换后)
  ↓ convertToLlm()  (必须)：过滤/转换自定义消息
Message[] (pi-ai 认识的：user/assistant/toolResult)
```

**默认 `convertToLlm`**（[agent.ts:27-31](packages/agent/src/agent.ts)）：

```typescript
return messages.filter(m =>
  m.role === "user" || m.role === "assistant" || m.role === "toolResult"
)
```

### 6.3 自定义消息扩展（TypeScript 声明合并）

```typescript
declare module "@mariozechner/pi-agent-core" {
  interface CustomAgentMessages {
    artifact: { role: "artifact"; type: "code" | "doc"; content: string }
  }
}
// 此后 AgentMessage union 自动包含 artifact
```

应用要自行在 `convertToLlm` 里决定如何处理——丢掉 / 转成文本 / 嵌入助手消息。

---

## 7. 与 pi-ai 的集成

### 7.1 StreamFn 契约

```typescript
type StreamFn = typeof streamSimple  // 签名完全一致
```

默认用 `streamSimple`。可换成 `streamProxy`（HTTP 代理）或自定义实现，只要签名一致。

### 7.2 事件翻译

`AssistantMessageEvent`（pi-ai）→ `AgentEvent`（agent-core）映射：

| pi-ai | agent-core |
|-------|------------|
| `start` | `message_start` |
| `text_delta` / `thinking_delta` / `toolcall_delta` | `message_update` |
| `done` / `error` | `message_end` |

注意：agent-core 的 `message_update` payload 里**既包含**原始 `assistantMessageEvent`，**又包含**当前 `partial message`，UI 可按需用。

---

## 8. 推荐阅读顺序

| 顺序 | 文件 | 重点 |
|------|------|------|
| 1 | [types.ts](packages/agent/src/types.ts) | AgentEvent / AgentMessage / AgentTool / AgentLoopConfig 整套类型 |
| 2 | [agent.ts](packages/agent/src/agent.ts) | Agent 类：状态 / 订阅 / 生命周期；`runWithLifecycle` |
| 3 | [agent-loop.ts](packages/agent/src/agent-loop.ts) | **核心**：`runLoop` / `streamAssistantResponse` / 工具三阶段 |
| 4 | [proxy.ts](packages/agent/src/proxy.ts) | 可选：HTTP 代理下如何保持事件语义 |
| 5 | [index.ts](packages/agent/src/index.ts) | 公共 API 复核 |

---

## 9. 设计亮点

| 亮点 | 描述 |
|------|------|
| **就地消息更新** | 流式 delta 用 `messages[last] = partial`，避免每个 token 触发数组复制，GC 压力极低 |
| **外层 + 内层循环** | Steering（紧急）vs Follow-up（低优）两级队列分离；外层还隐含防死循环 |
| **并发执行 + 顺序结果** | 工具并发跑但结果按助手原始顺序回填，兼顾性能和确定性 |
| **串行的准备阶段** | 即使 parallel 模式，`beforeToolCall` 钩子也是串行的，保证拦截决策可预测 |
| **两级消息转换** | `transformContext`（AgentMessage 层）和 `convertToLlm`（LLM 协议层）分工清晰 |
| **声明合并扩展自定义消息** | 应用通过 TS `declare module` 扩展 `CustomAgentMessages`，不改 agent-core 类型 |
| **StreamFn 可插拔** | 整个 LLM 调用点抽象成一个函数类型，轻松替换为代理 / mock / 本地实现 |
| **状态先更新后派发** | `processEvents` 内先改 `_state`，再 await 监听器——监听器里读状态永远一致 |
| **区分 error vs aborted** | 两者都停循环，但独立的 `stopReason`，UI 和日志可区别对待 |

---

## 10. 自己构建一个最小 Agent

根据上面的分析，最小使用方式：

```typescript
import { Agent } from "@mariozechner/pi-agent-core"
import { getModel } from "@mariozechner/pi-ai"

const agent = new Agent({
  systemPrompt: "You are a helpful assistant.",
  model: getModel("anthropic", "claude-opus-4-7"),
  tools: [myTool],                         // AgentTool[]
  convertToLlm: defaultConvertToLlm,       // 或自定义
  apiKey: process.env.ANTHROPIC_API_KEY,
})

agent.subscribe((event) => {
  if (event.type === "message_update") {
    console.log(event.message.content)     // 流式显示
  }
})

await agent.prompt("What is 2+2?")
console.log(agent.state.messages)
```

复杂场景要扩展：
- 自定义消息 → `declare module` + `convertToLlm` 改写
- 权限/速率 → `beforeToolCall`
- 成本统计 → `afterToolCall` + 监听 `message_end`
- Steering 注入 → 实现 `getSteeringMessages`
- 代理服务器 → 构造时传 `streamFn: streamProxy(...)`
