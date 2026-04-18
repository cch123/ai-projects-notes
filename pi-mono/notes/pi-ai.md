# pi-ai：统一 LLM API 层

> 分析日期：2026-04-19
> 源码版本：0.67.68
> 上游：[badlogic/pi-mono](https://github.com/badlogic/pi-mono)
> 本文档中的路径均相对于 pi-mono 仓库根目录

**定位**：把 20+ 个 LLM 提供商（Anthropic / OpenAI / Google / Mistral / Bedrock / Vertex / Ollama 等）抽象成一套统一的流式 API、消息格式、工具协议、事件模型。上层的 `pi-agent-core` / `pi-coding-agent` 都通过它调 LLM。

---

## 1. 目录结构

```
packages/ai/src/
├── types.ts                    # 413 行，所有核心类型集中地
├── stream.ts                   # 60 行，三个公共 API 入口
├── models.ts                   # 83 行，模型注册表查询 + 成本计算
├── models.generated.ts         # 自动生成：所有 provider × model 的元数据
├── api-registry.ts             # 99 行，provider 注册中心
├── index.ts / cli.ts
│
├── providers/                  # 15 个 provider，共 ~9000 行
│   ├── anthropic.ts            # 954 行，最完整的参考实现
│   ├── openai-completions.ts   # 894 行
│   ├── openai-responses.ts / openai-responses-shared.ts
│   ├── azure-openai-responses.ts
│   ├── google.ts / google-shared.ts / google-gemini-cli.ts / google-vertex.ts
│   ├── amazon-bedrock.ts
│   ├── mistral.ts
│   ├── faux.ts                 # 测试/Mock provider
│   ├── register-builtins.ts    # 433 行，惰性注册所有内置 provider
│   ├── transform-messages.ts   # 161 行，跨 provider 上下文迁移
│   └── simple-options.ts       # 48 行，推理预算自动调参
│
└── utils/
    ├── event-stream.ts         # AsyncIterable 事件流基建
    ├── json-parse.ts           # 流式 JSON 增量解析
    ├── overflow.ts             # 上下文溢出的跨 provider 检测
    ├── validation.ts           # 工具参数校验
    ├── oauth/                  # OAuth 流程
    └── ...
```

---

## 2. 核心类型（types.ts）

### Model：模型元数据

```typescript
interface Model<TApi extends Api> {
  id: string;                // "claude-opus-4-7"
  name: string;              // 显示名
  api: TApi;                 // "anthropic-messages" | "openai-completions" | ...
  provider: Provider;        // "anthropic" | "openai" | ...
  baseUrl: string;
  reasoning: boolean;        // 是否支持 thinking/reasoning
  input: ("text" | "image")[];
  cost: { input; output; cacheRead; cacheWrite };  // $ / 1M tokens
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: OpenAICompletionsCompat | OpenAIResponsesCompat;  // OpenAI 兼容端点的能力开关
}
```

### Context / Message：统一消息容器

```typescript
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}

// Message 是三种角色的 union：
UserMessage        { role: "user",       content: string | (TextContent | ImageContent)[] }
AssistantMessage   { role: "assistant",  content: (TextContent | ThinkingContent | ToolCall)[], usage, stopReason, ... }
ToolResultMessage  { role: "toolResult", toolCallId, toolName, content, isError }
```

### ContentBlock：4 种内容块

| 类型 | 字段 |
|------|------|
| `TextContent` | `{ type: "text", text }` |
| `ThinkingContent` | `{ type: "thinking", thinking, thinkingSignature?, redacted? }` |
| `ImageContent` | `{ type: "image", data (base64), mimeType }` |
| `ToolCall` | `{ type: "toolCall", id, name, arguments }` |

### Tool：TypeBox Schema

```typescript
interface Tool<T extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: T;  // TypeBox JSON Schema，运行时可用 AJV 校验
}
```

### Usage：实时成本

```typescript
interface Usage {
  input; output; cacheRead; cacheWrite; totalTokens;
  cost: { input; output; cacheRead; cacheWrite; total };  // 每条消息算一次
}
```

### AssistantMessageEvent：流事件协议

生命周期（所有 provider 必须遵守）：

```
start
  ├─ text_start → text_delta* → text_end
  ├─ thinking_start → thinking_delta* → thinking_end
  ├─ toolcall_start → toolcall_delta* → toolcall_end
  └─ ...
done | error
```

---

## 3. 三个公共 API（stream.ts）

```typescript
// 推荐：自动处理 reasoning/temperature/budget
streamSimple(model, context, options?: SimpleStreamOptions): AssistantMessageEventStream

// 高级：暴露 provider 特定选项（AnthropicOptions 等）
stream(model, context, options?: ProviderStreamOptions): AssistantMessageEventStream

// 非流式：内部调用 stream() 后 await result()
complete(model, context, options?): Promise<AssistantMessage>
```

调用链：

```
streamSimple/stream
  → api-registry 按 model.api 路由
  → provider.streamSimple() 或 provider.stream()
  → 内部 async loop 推送事件到 AssistantMessageEventStream
  → done/error 终结流
```

---

## 4. Provider 抽象（api-registry.ts）

```typescript
interface ApiProvider<TApi extends Api> {
  api: TApi;
  stream: StreamFunction;         // 完整选项
  streamSimple: StreamFunction;   // 简化选项
}

// 注册表：Map<api-id, ApiProvider>
// 支持运行时注册/注销，给插件留口子
```

### Anthropic 实现流程（providers/anthropic.ts 的例子）

```
streamAnthropic(model, context, options)
  │
  ├─ 1. createClient()                    # L556-639
  │     • 判断 OAuth token / API key / Copilot
  │     • 添加 beta header：
  │       - interleaved-thinking-2025-05-14
  │       - fine-grained-tool-streaming-2025-05-14
  │
  ├─ 2. transformMessages()               # 跨 provider 迁移（下节详述）
  │
  ├─ 3. convertMessages()                 # L641-909
  │     • TextContent → Anthropic text block（带 cache_control）
  │     • ImageContent → base64 + media_type（不支持时自动过滤）
  │     • ToolCall → tool_use block（normalizeToolCallId）
  │     • ToolResult → tool_result block
  │     • ThinkingContent → thinking / redacted_thinking（重放签名）
  │
  ├─ 4. buildParams()
  │     • systemPrompt + cache_control
  │     • thinking 参数：
  │       - Opus 4.6+：adaptive thinking + output_config.effort
  │       - 其他：enabled + budget_tokens
  │     • temperature（thinking 启用时静默忽略）
  │
  ├─ 5. SSE 事件处理                       # L285-440
  │     message_start        → input_tokens, cache_tokens
  │     content_block_start  → 新建 text/thinking/tool_use 块
  │     content_block_delta  →
  │       - text_delta：文本累积
  │       - thinking_delta：思考累积（带 signature）
  │       - input_json_delta：流式 JSON（parseStreamingJson 增量解析）
  │     content_block_stop   → 推送 *_end 事件
  │     message_delta        → 更新 stopReason, usage
  │
  └─ 6. 错误规范化                         # L443-462
        • AbortError → stopReason = "aborted"
        • APIError → stopReason = "error" + errorMessage
        • stopReason 映射：end_turn→stop, max_tokens→length, tool_use→toolUse
```

---

## 5. 跨 provider 上下文迁移（transform-messages.ts）

pi-ai 支持会话中途切换模型，这个文件处理三种棘手问题：

### 5.1 工具 ID 规范化

OpenAI 生成 ~450 字符的 ID（`call_abc123|xyz|...`），Anthropic 要求 `^[a-zA-Z0-9_-]{1,64}$`。

```typescript
normalizeToolCallId(id) = id.replace(/[^a-zA-Z0-9_-]/g, "_").slice(0, 64)
// 构建映射表，让后续 tool_result 能对上号
```

### 5.2 思考块过滤

```typescript
if (block.redacted) {
  // OpenAI 的加密 reasoning 只对同模型有效
  return isSameModel ? block : []
}
if (block.thinkingSignature && isSameModel) {
  return block  // 签名需原样传回（用于重放）
}
if (!block.thinking.trim()) return []  // 空块删除
return { type: "text", text: block.thinking }  // 跨模型降级为文本
```

### 5.3 孤立工具调用合成

两遍算法：第一遍转换消息，第二遍给没有 `tool_result` 的 `tool_call` 自动插入合成结果：

```typescript
{ role: "toolResult", toolCallId, toolName,
  content: [{ type: "text", text: "No result provided" }],
  isError: true }
```

否则 Anthropic 会直接拒绝请求。

---

## 6. 关键特性实现

### 6.1 模型自动发现

`models.generated.ts` 由脚本（`scripts/generate-models.ts`）从 provider API 自动抓取生成，包含 100+ 模型 × 全部元数据。`models.ts` 提供类型安全的查询 API：

```typescript
getModel<"anthropic", "claude-opus-4-7">(...)  // 编译期类型检查
getProviders() / getModels(provider)
calculateCost(model, usage)
supportsXhigh(model)
```

### 6.2 流式 JSON 重建（utils/json-parse.ts）

工具参数是增量 JSON（`{"path":"/foo`→`{"path":"/foo","co`→...），需要能中途解析：

```
parseStreamingJson(partial):
  1. JSON.parse() 尝试完整解析
  2. 失败 → partial-json 库（补全末尾 }])）
  3. 再失败 → 返回 {}（安全降级）
```

每收到 delta 就调一次，UI 可以实时预览参数。

### 6.3 上下文溢出统一检测（utils/overflow.ts）

20+ 个正则模式兜底所有 provider 的溢出错误消息：

```typescript
OVERFLOW_PATTERNS = [
  /prompt is too long/i,                    // Anthropic
  /exceeds the context window/i,            // OpenAI
  /input token count.*exceeds/i,            // Google
  /maximum prompt length is \d+/i,          // xAI
  /reduce the length of the messages/i,     // Groq
  // ... 共 20+
]

isContextOverflow(error):
  模式匹配 || usage.input > model.contextWindow  // z.ai 静默截断兜底
```

### 6.4 思考块重放签名

OpenAI/Google 的 reasoning 是加密的，多轮对话必须原样传回：

```typescript
// 接收时
{ type: "thinking", thinking: "[Reasoning redacted]",
  thinkingSignature: event.content_block.data, redacted: true }

// 重放时
if (block.redacted) {
  blocks.push({ type: "redacted_thinking", data: block.thinkingSignature })
  continue  // 跳过文本，直接传签名
}
```

### 6.5 Prompt Caching

Anthropic 的缓存控制需挂在**具体内容块**上（全局设置无效）。策略是在最后一条用户消息的最后一个内容块加 `cache_control`，形成缓存边界，之前的全部可复用：

```typescript
// anthropic.ts L884-906
if (cacheControl && params.length > 0) {
  const lastMsg = params[params.length - 1]
  if (lastMsg.role === "user") {
    const lastBlock = [...].at(-1)
    lastBlock.cache_control = { type: "ephemeral", ttl: "1h" }  // long + api.anthropic.com
  }
}
```

### 6.6 惰性加载（register-builtins.ts）

各 provider SDK 加起来几十 MB，全部 eager import 启动极慢。方案：

```typescript
let anthropicModulePromise: Promise | undefined

function loadAnthropic() {
  anthropicModulePromise ||= import("./anthropic.js")
  return anthropicModulePromise
}

// createLazyStream 返回一个立刻可用的空 EventStream，
// 背后 loadModule().then(forwardEvents)
```

首次调用时才加载，不用的 provider 永不加载。

### 6.7 GitHub Copilot 工具名规范化

Copilot 要求工具名首字母大写（"Read" 而非 "read"），硬编码了一份 Claude Code 工具名清单，走 OAuth 时自动在出入口转换：

```typescript
const ccToolLookup = new Map(claudeCodeTools.map(t => [t.toLowerCase(), t]))
toClaudeCodeName("read") → "Read"
fromClaudeCodeName("Bash", context.tools) → 按 tools 数组查回真实名
```

---

## 7. 有意思的设计

| 亮点 | 位置 | 价值 |
|------|------|------|
| **stopReason 枚举收敛** | 每个 provider 的 `mapStopReason()` | 上层不用关心 provider 差异，未知原因直接 `throw`——强制更新而不是静默 |
| **OpenAICompletionsCompat 能力位** | `types.ts` L265-294 | 一套 OpenAI 兼容代码能同时服务 vLLM/Ollama/OpenRouter/z.ai/Qwen 等十几家，只靠 `compat` flag 区分 |
| **合成 tool_result** | `transform-messages.ts` L98-119 | 历史缺结果也能继续对话，不会被 API 拒 |
| **temperature + thinking 互斥的静默处理** | `anthropic.ts` L682-685 | 客户端可以无脑传参，provider 决定生效 |
| **AssistantMessageEventStream** | `utils/event-stream.ts` | 同时实现 AsyncIterable 和 `.result(): Promise`，一份流两种消费方式 |

---

## 8. 推荐阅读顺序

| 顺序 | 文件 | 重点 |
|------|------|------|
| 1 | `types.ts` | 统一数据模型，全局类型字典 |
| 2 | `stream.ts` | 三个 API 入口的关系 |
| 3 | `api-registry.ts` | provider 注册机制 |
| 4 | `utils/event-stream.ts` | 流的底层实现 |
| 5 | `providers/anthropic.ts` | 最完整的 provider 参考，走一遍完整流程 |
| 6 | `providers/transform-messages.ts` | 跨 provider 兼容性的精华 |
| 7 | `models.ts` | 模型发现与成本 |
| 8 | `providers/register-builtins.ts` | 惰性加载模式 |

---

## 9. 自己动手扩展一个 provider

根据以上分析，添加新 provider 需要做：

1. 选 API 类型（`anthropic-messages` / `openai-completions` / 自定义）
2. 在 `providers/` 下新建 `myprovider.ts`，实现 `stream()` 和 `streamSimple()`
3. 导出函数，`stopReason` 严格映射到统一枚举
4. 在 `register-builtins.ts` 加一条 lazy loader
5. 在 `models.generated.ts`（或生成脚本）补充模型元数据
6. 如是 OpenAI 兼容端点，通过 `compat` 字段声明差异点，复用 `openai-completions.ts`

走 OpenAI 兼容路线是最经济的——只需填 compat flags 就能接入。
