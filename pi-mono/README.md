# pi-mono 学习笔记

[pi-mono](https://github.com/badlogic/pi-mono) 是 Mario Zechner 开发的开源 AI 编程 Agent 框架，TypeScript 编写，npm workspace 组织的 monorepo。

## 项目定位

- **哲学**：默认极简（无子 agent、无 plan mode），通过扩展/技能/主题/提示模板自定义工作流
- **多运行模式**：Interactive (TUI) / Print / RPC / SDK
- **统一 LLM API**：支持 20+ 提供商，自动模型发现、跨模型迁移

## 笔记文件

- [architecture.md](./architecture.md)——整体架构分析：分层、包依赖、关键流程、入门阅读路径

## 后续计划

- [ ] pi-ai：LLM 统一层的设计与各 provider 适配实现
- [ ] pi-agent-core：Agent 循环、事件流、工具执行
- [ ] pi-tui：终端微分渲染原理
- [ ] pi-coding-agent：
  - [ ] 4 种运行模式的实现差异
  - [ ] 6 个核心工具的设计（read/write/edit/bash/find/grep）
  - [ ] 扩展系统（Extensions / Prompt Templates / Skills）
  - [ ] 会话管理、压缩、分支
