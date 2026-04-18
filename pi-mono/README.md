# pi-mono 学习笔记

[pi-mono](https://github.com/badlogic/pi-mono) 是 Mario Zechner 开发的开源 AI 编程 Agent 框架，TypeScript 编写，npm workspace 组织的 monorepo。

## 项目定位

- **哲学**：默认极简（无子 agent、无 plan mode），通过扩展/技能/主题/提示模板自定义工作流
- **多运行模式**：Interactive (TUI) / Print / RPC / SDK
- **统一 LLM API**：支持 20+ 提供商，自动模型发现、跨模型迁移

## 笔记文件

- [architecture.md](./architecture.md)——整体架构分析：分层、包依赖、关键流程、入门阅读路径
- [notes/pi-ai.md](./notes/pi-ai.md)——`pi-ai` 深入：类型系统、provider 抽象、跨 provider 迁移、缓存/思考/工具的实现细节
- [notes/pi-agent-core.md](./notes/pi-agent-core.md)——`pi-agent-core` 深入：外层/内层循环、事件系统、工具三阶段执行、steering/follow-up 队列
- [notes/pi-tui.md](./notes/pi-tui.md)——`pi-tui` 深入：行级差分渲染、Synchronized Update、Unicode 宽度、Kitty 键盘协议、13 个内置组件
- [notes/pi-coding-agent-part1.md](./notes/pi-coding-agent-part1.md)——`pi-coding-agent` Part 1：主入口、4 种运行模式、6 个核心工具
- [notes/pi-coding-agent-part2.md](./notes/pi-coding-agent-part2.md)——`pi-coding-agent` Part 2：扩展系统、提示模板、Skills
- [notes/pi-coding-agent-part3.md](./notes/pi-coding-agent-part3.md)——`pi-coding-agent` Part 3：会话管理（JSONL 仅追加、树结构、压缩）+ Interactive UI + 主题

## 进度

- [x] pi-ai：LLM 统一层的设计与各 provider 适配实现
- [x] pi-agent-core：Agent 循环、事件流、工具执行
- [x] pi-tui：终端微分渲染原理
- [x] pi-coding-agent：
  - [x] Part 1：4 种运行模式、6 个核心工具
  - [x] Part 2：扩展系统（Extensions / Prompt Templates / Skills）
  - [x] Part 3：会话管理（持久化、分支、压缩）+ Interactive 模式 UI

**pi-mono 系列完结** 🎉
