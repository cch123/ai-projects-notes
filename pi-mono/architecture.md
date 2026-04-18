# pi-mono 架构分析

> 分析时间：2026-04-18
> 分析版本：0.67.68
> 源码位置：`~/pi-mono`

## 1. 项目概述

**pi-mono** 是一个开源 AI 编程代理的 monorepo，作者 Mario Zechner。

- **语言**：TypeScript（ES2022），Node.js ≥ 20
- **构建**：npm workspace + 自定义构建顺序
- **代码质量**：Biome（lint/format）+ TypeScript strict + Vitest
- **版本策略**：Lockstep——所有包同步发布

### 核心理念

- 默认功能极简，不带子 agent / plan mode
- 通过 **扩展（Extensions）**、**技能（Skills）**、**主题**、**提示模板** 自定义行为
- 同一套 Agent 运行时服务多种交互模式（TUI、单次打印、RPC、SDK）

---

## 2. 单体仓库结构

```
pi-mono/
├── packages/
│   ├── ai/                 # LLM 统一 API（基础）
│   ├── agent/              # 通用 Agent 运行时（基础）
│   ├── tui/                # 终端 UI 框架（基础）
│   ├── coding-agent/       # 编程 Agent CLI（主应用）
│   ├── web-ui/             # Web 组件库
│   ├── mom/                # Slack Bot
│   └── pods/               # GPU Pod 管理 CLI
├── AGENTS.md               # Agent 规则与贡献流程
├── CONTRIBUTING.md
├── tsconfig.base.json
└── package.json            # workspace 定义
```

### 各包一览

| 包名 | 角色 | 内部依赖 |
|------|------|---------|
| `@mariozechner/pi-ai` | 统一 LLM API（20+ provider） | 无 |
| `@mariozechner/pi-tui` | 终端 UI，微分渲染、内置组件 | 无 |
| `@mariozechner/pi-agent-core` | 通用 Agent 运行时、事件流 | pi-ai |
| `@mariozechner/pi-coding-agent` | 编程 Agent CLI（主应用） | pi-ai、pi-agent-core、pi-tui |
| `@mariozechner/pi-web-ui` | Web 端对话组件库 | pi-ai、pi-tui |
| `@mariozechner/pi-mom` | Slack Bot | pi-ai、pi-agent-core、pi-coding-agent |
| `@mariozechner/pi`（pods）| vLLM GPU Pod 管理 CLI | pi-agent-core |

---

## 3. 分层依赖

```
┌─────────────────────────────────────────────────┐
│ 应用层                                            │
│  coding-agent │ mom │ web-ui │ pods             │
├─────────────────────────────────────────────────┤
│ Agent 运行时层                                    │
│            pi-agent-core                        │
├─────────────────────────────────────────────────┤
│ 基础层                                            │
│     pi-ai               pi-tui                  │
└─────────────────────────────────────────────────┘
```

- **pi-ai / pi-tui**：无上游依赖，纯基础设施
- **pi-agent-core**：依赖 pi-ai，为上层 Agent 提供循环与事件
- **pi-coding-agent**：最复杂，集齐 ai/agent/tui

---

## 4. 关键架构

### 4.1 多运行模式（pi-coding-agent）

入口：`packages/coding-agent/bin/pi` → `src/cli.ts` → `src/main.ts`，按参数分派到：

| 模式 | 入口 | 用途 |
|------|------|------|
| Interactive | `src/modes/interactive/interactive-mode.ts` | TUI，微分渲染，键盘/主题/扩展完整体验 |
| Print | `src/modes/print-mode.ts` | 单次执行，输出 md/text/json，无 TUI |
| RPC | `src/modes/rpc/` | 进程间通信，供 IDE 等宿主集成 |
| SDK | `src/core/sdk.ts` | 程序化嵌入，返回 `AgentSession` 对象 |

### 4.2 工具系统

`packages/coding-agent/src/core/tools/` 下 6 个核心工具，用 [TypeBox](https://github.com/sinclairzx81/typebox) 定义 schema：

| 工具 | 功能要点 |
|------|---------|
| read | 行范围、字节限制、截断 |
| write | 原子写入、自动建目录 |
| edit | 行替换/插入/删除，自动生成 diff |
| bash | 带超时和 cwd 隔离 |
| find | 按名称/类型检索文件 |
| grep | 内容正则检索 |

### 4.3 Agent 循环（pi-agent-core）

核心文件：`packages/agent/src/agent.ts`

```
prompt(input)
  └─ 循环：
     ├─ 组装 context
     ├─ 调用 LLM（stream）
     ├─ 收到 tool_call → beforeToolCall → 执行 → afterToolCall
     ├─ 工具结果回填 context
     └─ 直到 stop
  └─ 全程发出事件：
     agent_start / turn_start / message_update /
     tool_execution_start|update|end / turn_end / agent_end
```

**事件驱动**：UI、持久化、扩展都是事件订阅者，彼此解耦。

### 4.4 LLM 统一层（pi-ai）

三个接口：

- `streamSimple(model, context, tools, options)`——最小流式接口
- `stream(...)`——暴露 provider 特定选项
- `complete(...)`——非流式

**Provider 支持（19 个）**：
- 商业：OpenAI / Anthropic / Google / Mistral / xAI
- 云：Azure / AWS Bedrock / Google Vertex
- 兼容：Ollama / vLLM / LM Studio 等 OpenAI 协议
- 特殊：GitHub Copilot / Gemini CLI / OAuth 相关

**跨 provider 能力**：模型自动发现、token 计费、图像、thinking/reasoning、prompt caching、跨模型上下文迁移。

### 4.5 扩展机制（三层）

| 层 | 形式 | 能力 |
|----|------|------|
| Extensions | TS 代码 | 新增工具 / 拦截事件 / 自定义 UI / 斜杠命令 |
| Prompt Templates | Handlebars + YAML frontmatter | 动态角色、上下文注入、模板继承 |
| Skills | Markdown 文档块 | 代码示例、快捷命令、工作流步骤 |

扩展加载来源：本地 `.pi/agent/extensions` / `extensions.json` 远程包 / `AGENTS.md` 内联。

### 4.6 会话管理

`packages/coding-agent/src/core/session-manager.ts`（约 30KB）负责：

- YAML 持久化
- 消息条目类型：user / assistant / tool_call / tool_result / model_switch / thinking_level_change
- 分支、重新上下文化、智能压缩（旧消息摘要）

---

## 5. 入门阅读路径

| 顺序 | 文件 | 看什么 |
|------|------|--------|
| 1 | `README.md`（根） | 项目全貌 |
| 2 | `packages/ai/README.md` | LLM 抽象 |
| 3 | `packages/ai/src/types.ts`（前 100 行） | Model / Tool / StreamOptions |
| 4 | `packages/agent/README.md` | Agent 事件序列 |
| 5 | `packages/agent/src/agent.ts`（前 200 行） | Agent 循环 |
| 6 | `packages/coding-agent/README.md`（前 150 行） | 快速上手 |
| 7 | `packages/coding-agent/src/main.ts`（前 150 行） | 模式分派 |
| 8 | `packages/coding-agent/src/core/agent-session.ts`（前 300 行） | 会话状态 |
| 9 | `packages/coding-agent/src/core/extensions/types.ts`（前 200 行） | 扩展接口 |
| 10 | `packages/coding-agent/src/modes/interactive/interactive-mode.ts`（前 200 行） | TUI 渲染 |

---

## 6. 关键命令速查

```bash
npm run build   # 严格顺序: tui → ai → agent → coding-agent → mom → web-ui → pods
npm run dev     # 全包 watch
npm run check   # Biome + tsc
npm test        # 所有包 vitest
./pi-test.sh    # 从源码运行 pi

npm run release:patch   # lockstep 发布（所有包同步）
```

> `web-ui` 编译需要依赖包 `.d.ts` 文件——这是构建顺序的原因。

---

## 7. 代码质量规则（摘自 AGENTS.md）

1. 所有 import 必须在文件顶部，禁止内联 import
2. 禁用 `any`（除非确有必要）
3. 提交前必须通过 `npm run check`
4. 改工具必须跑相关测试
5. 并发 Agent 场景下用具体路径 `git add`，禁用 `git add -A`
