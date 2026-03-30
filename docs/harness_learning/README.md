# Harness Learning Notes

这个目录用于持续沉淀 DeerFlow harness 学习资料。

## 学习路线

- [00-harness-overview-and-mastery-roadmap.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/00-harness-overview-and-mastery-roadmap.md)
  - 从总览视角理解 harness 的边界、模块布局、运行时主链、外部接入方式，以及从 0 到 1 的掌握路线

- [02-main-agent-assembly.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/02-main-agent-assembly.md)
  - 理解 `make_lead_agent()` 如何把 model、tools、middleware、prompt、state 组装成主 agent

- [03-middleware-governance.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/03-middleware-governance.md)
  - 理解 DeerFlow 为什么把系统级治理逻辑沉到 middleware 层，以及顺序为什么是隐含协议

- [04-models-tools-config-reflection.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/04-models-tools-config-reflection.md)
  - 理解 config、reflection、model factory、tool assembler 如何构成能力装配层

- [05-sandbox-and-thread-resource-model.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/05-sandbox-and-thread-resource-model.md)
  - 理解 thread filesystem、虚拟路径、sandbox provider 和执行资源隔离

- [06-mcp-and-skills-extension-system.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/06-mcp-and-skills-extension-system.md)
  - 理解 MCP 和 skills 两条扩展线分别如何注入能力与工作流知识

- [07-memory-title-and-summarization.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/07-memory-title-and-summarization.md)
  - 理解长期语义记忆、会话标题和上下文压缩与 checkpointer 的边界

- [08-subagents-and-acp.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/08-subagents-and-acp.md)
  - 理解 DeerFlow 如何把多代理协作与外部代理调用做成运行时能力

- [09-external-integration-modes.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/09-external-integration-modes.md)
  - 理解外部系统接入 harness 的三种主要模式与取舍

- [10-minimal-host-integration-blueprint.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/10-minimal-host-integration-blueprint.md)
  - 把前面所有概念收束成最小宿主集成蓝图

## 已完成专题

- [01-checkpointer-state-model.md](d:/dev/github/deer-flow-analysis/docs/harness_learning/01-checkpointer-state-model.md)
  - `ThreadState / thread_id / checkpointer / thread filesystem / memory` 五者关系

## 约定

- 后续所有 harness 学习资料继续保存在这个目录
- 文件按编号递增，便于形成系统化学习顺序
- 每篇优先回答三个问题：
  - 这个模块解决什么问题
  - 它在运行链里处于什么位置
  - 外部系统集成时应该如何理解它
