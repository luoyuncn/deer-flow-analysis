# Harness Learning 09: 外部应用接入模式

这一篇专门讲“外部应用如何接入 harness”。

这不是附属话题，而是理解 harness 价值的关键，因为 DeerFlow 明确把 harness 和 app 分层了。

## 1. 为什么这篇重要

如果你只会在 DeerFlow 官方项目里看 harness，那你看到的只是“一个应用怎么使用它”。

真正掌握 harness，必须知道：

- 它独立提供了哪些接入面
- 不同接入方式的边界与取舍是什么
- 宿主系统应该在哪一层和它对接

## 2. 三种接入模式

## 模式 A：LangGraph Server + Gateway

最接近官方运行方式。

### 特点

- 使用 `langgraph.json`
- graph entry 是 `make_lead_agent`
- checkpointer 由 async provider 注入
- Gateway 负责额外的 REST 管理能力

### 适合场景

- 先快速跑通整套 DeerFlow 能力
- 需要 LangGraph Studio / HTTP 服务形态
- 要复用前端、渠道适配等现成结构

### 代价

- 系统组件更多
- 外部宿主需要适配 HTTP/SSE 边界

## 模式 B：Embedded `DeerFlowClient`

这是最适合 Python 宿主后端快速内嵌的方式之一。

### 特点

- 不需要额外启动 LangGraph Server
- 直接在进程内创建 agent
- 暴露 `chat()` / `stream()` / MCP / skill / memory / uploads 等接口
- 返回格式尽量对齐 Gateway

### 适合场景

- 你的宿主本身就是 Python 服务
- 你希望减少部署复杂度
- 你想直接拿流式事件并做自定义 API 包装

### 代价

- 你要自己承担更多宿主侧生命周期管理

## 模式 C：低层 `create_deerflow_agent()`

这是最“内核级”的接入方式。

### 特点

- 直接用纯参数构建 agent
- 不强依赖 DeerFlow 的上层 app 形态
- 可以自己决定 prompt、middleware、tools、checkpointer

### 适合场景

- 宿主已经有自己的 API、会话、消息表、业务规则
- 你只想复用 DeerFlow runtime kernel 的关键能力
- 你需要强控制权

### 代价

- 需要你更懂 DeerFlow 的内部装配逻辑

## 3. 三种模式的本质差异

| 模式 | 控制权 | 集成成本 | 运行组件 | 最适合 |
| --- | --- | --- | --- | --- |
| Server + Gateway | 较低 | 较低 | 多 | 快速用全家桶 |
| Embedded Client | 中等 | 中等 | 少 | Python 宿主直连 |
| create_deerflow_agent | 最高 | 最高 | 最少 | 深度内嵌与自定义 |

## 4. 外部系统接入时最关键的五个映射

无论你用哪种模式，最终都要处理这五种映射：

### 4.1 会话 ID -> `thread_id`

宿主的 `conversation_id` 通常应该稳定映射到 DeerFlow 的 `thread_id`。

### 4.2 宿主消息历史 -> DeerFlow runtime state

不要把两者混成一个存储面。

### 4.3 宿主文件体系 -> DeerFlow thread filesystem

要决定是否让 DeerFlow 接管线程工作目录。

### 4.4 宿主能力 -> MCP / tool / ACP

要决定哪些能力走 MCP，哪些直接做工具，哪些交给 ACP。

### 4.5 宿主记忆体系 -> DeerFlow memory

要决定继续用 DeerFlow memory、替换 storage、还是完全由宿主管理。

## 5. 设计上的推荐理解

最推荐的思路是：

```text
把 DeerFlow harness 当成 runtime kernel
把你的宿主系统当成 protocol / business / product layer
```

这样最清晰：

- DeerFlow 负责 agent runtime
- 宿主负责业务边界、API 协议、审计、权限、产品逻辑

## 6. 学完这一篇你应该能回答

1. 三种接入模式的差异是什么
2. 为什么 `DeerFlowClient` 是很重要的嵌入入口
3. 为什么 `create_deerflow_agent()` 适合深度定制接入
4. 宿主系统和 harness 之间需要做好哪些映射

## 最短总结

DeerFlow harness 不是只能跑在官方服务后面，它本身提供了从全家桶接入到内核级嵌入的多层接入面，关键是宿主系统要想清楚状态、能力、文件、记忆和协议五种映射关系。
