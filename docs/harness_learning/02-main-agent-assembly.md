# Harness Learning 02: 主 Agent 装配链

这一篇研究 DeerFlow harness 最核心的一条主链：

```text
make_lead_agent
-> create_chat_model
-> get_available_tools
-> _build_middlewares
-> apply_prompt_template
-> create_agent(..., state_schema=ThreadState)
```

如果说 `checkpointer` 让你明白“状态如何保留”，那么这一篇会让你明白“一个 DeerFlow agent 到底是怎样被装出来的”。

## 1. 它解决什么问题

DeerFlow 不是写死一个固定 agent，而是在每次运行时，根据：

- 当前模型
- 当前 agent_name
- 当前是否 plan mode
- 当前是否启用 subagent
- 当前 prompt 片段
- 当前 tools 集合

把一个完整运行时现场装出来。

这意味着 DeerFlow 的核心不是某一段 prompt，而是“装配过程”。

## 2. 全局位置

主入口在：

- [agent.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/lead_agent/agent.py)

Server 模式下，它由 [langgraph.json](d:/dev/github/deer-flow-analysis/backend/langgraph.json) 注册成 graph entrypoint。

Embedded 模式下，`DeerFlowClient` 会复用几乎相同的装配方式。

## 3. 装配过程逐步拆开

### 3.1 读取运行时参数

`make_lead_agent(config)` 一开始先从 `config.configurable` 取运行时参数：

- `thinking_enabled`
- `reasoning_effort`
- `model_name`
- `is_plan_mode`
- `subagent_enabled`
- `max_concurrent_subagents`
- `agent_name`

这说明 DeerFlow 的很多行为不是靠全局常量决定，而是靠每次调用时的 runtime config 决定。

### 3.2 解析最终模型

`_resolve_model_name()` 和 agent config 一起决定最终模型名。

优先级大致是：

1. 请求级 `model_name`
2. agent 自定义配置里的 `model`
3. 全局默认模型

然后再检查：

- 模型是否存在
- 是否支持 thinking

所以 DeerFlow 的 model 选择不是 prompt 层逻辑，而是运行时解析逻辑。

### 3.3 创建工具集合

`get_available_tools()` 会按当前模型和运行时开关决定工具集合。

这里不是静态列表，而是动态装配：

- config tools
- builtin tools
- MCP tools
- ACP tools
- subagent tool
- 可选的 `view_image`

### 3.4 组装 middleware 链

`_build_middlewares()` 会按固定顺序拼接运行时治理链。

先从基础运行时 middleware 起步，再追加：

- summarization
- todo
- token usage
- title
- memory
- view image
- deferred tool filter
- subagent limit
- loop detection
- clarification

这里最重要的思想是：

DeerFlow 把很多“系统行为”放在 middleware，而不是 prompt。

### 3.5 组装 system prompt

`apply_prompt_template()` 会把这些上下文拼进去：

- agent role
- soul / agent identity
- memory context
- skills section
- deferred tools section
- subagent section
- working directory conventions

所以 prompt 不是单个文件，而是运行时拼接产物。

### 3.6 最终交给 LangChain/LangGraph

最后才进入真正的：

```python
create_agent(
    model=...,
    tools=...,
    middleware=...,
    system_prompt=...,
    state_schema=ThreadState,
)
```

这一步特别重要，因为它把 DeerFlow 的内核清楚暴露出来了：

```text
model + tools + middleware + system_prompt + state_schema
```

## 4. 这背后的设计思想

### 4.1 运行时优先，而不是静态配置优先

DeerFlow 的很多行为都允许被当前请求覆盖，这使它更像一个 runtime，而不是模板应用。

### 4.2 prompt 不是中心，装配才是中心

很多 agent 项目会把“智能性”几乎全压在 prompt 上。DeerFlow 不是这样。

它把：

- 状态
- 治理
- 能力
- 执行边界

都从 prompt 中剥离出来，放入独立运行时部件。

### 4.3 主 agent 是可变 profile，不是单一角色

通过 `agent_name`、tool groups、skills、memory、model 选择，DeerFlow 支持同一套 runtime 挂不同 agent profile。

## 5. 你读代码时应该盯住哪些点

建议顺序：

1. 看 `make_lead_agent()` 的入口参数
2. 看模型解析逻辑
3. 看 tools 装配调用
4. 看 middleware 装配调用
5. 看 prompt 模板调用
6. 看最终 `create_agent(...)`

你要一边读，一边问自己：

- 这一步是在决定“能力”还是“治理”还是“状态”
- 这一步是 compile-time 还是 runtime
- 这一步是否可以被宿主应用替换

## 6. 外部集成时这篇有什么价值

这条主装配链决定了外部系统可以在哪些层面介入：

- 换 model
- 换 tools
- 换 middleware
- 换 prompt 片段
- 换 state schema
- 换 checkpointer

也就是说，外部接入 DeerFlow 时，真正的控制点不在 UI，而在这条装配链。

## 7. 学完这一篇你应该能回答

1. `make_lead_agent()` 到底装配了哪些部件
2. DeerFlow 的核心为什么不是 prompt，而是 runtime assembly
3. 哪些能力来自 model，哪些来自 tools，哪些来自 middleware
4. 为什么同一个 harness 可以支撑 server 模式和 embedded 模式

## 最短总结

DeerFlow 主 agent 的本质不是一个预定义角色，而是一条在运行时把模型、工具、治理链、提示词和状态面拼成可执行 agent 的装配流水线。
