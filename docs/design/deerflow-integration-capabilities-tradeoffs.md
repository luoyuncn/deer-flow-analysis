# DeerFlow 集成能力、限制与扩展性分析

## 说明

这份文档聚焦一个问题：

当业务系统把 DeerFlow 作为运行时内核集成进去之后，系统能力会得到哪些增强，又会受到哪些限制；这种 Agent 框架背后的设计模式是什么；以及未来新业务扩展时，哪些应该沉淀为 `skills`，哪些不应该。

这里讨论的前提是：

- 前端仍然使用业务方自己的前端
- 后端仍然使用业务方自己的 Python 服务
- DeerFlow 以 `create_deerflow_agent()` 或等价嵌入方式接入
- 现有业务服务、MCP、工具、消息存储继续保留

## 总体判断

把 DeerFlow 集成进现有业务系统后，得到的不是“一个更会聊天的 Agent”，而是一层统一的智能执行底座。

它最有价值的地方在于：

- 把会话、工具、MCP、skills、记忆、子任务编排放进同一套 runtime
- 让业务系统从“单点问答能力”升级为“可执行、可组合、可扩展的智能平台”

但它也会引入一套新的运行时约束：

- `thread_id`
- checkpointer
- tool contract
- middleware 顺序
- prompt / skills / runtime 的分层治理

因此，这种集成适合被理解为：

不是“把 DeerFlow 当整套产品替换业务系统”，而是“把 DeerFlow 当运行时内核嵌入业务系统”。

## 一、集成后业务系统能做到什么

### 1. 同一套内核支撑多个业务 Agent

接入 DeerFlow 后，系统不再只有一个固定的 ReAct Agent，而是可以围绕同一套 runtime 派生出多种 Agent 变体。

这些变体可以通过以下维度进行组合：

- 不同 system prompt
- 不同工具集合
- 不同 `tool_groups`
- 不同 `SOUL.md`
- 不同 skills 开关
- 不同模型配置
- 不同 memory / subagent 策略

这意味着未来新增一个业务 Agent，不需要重新造一个框架，而更像是基于同一内核做装配。

### 2. 会话从“消息历史”升级为“运行时状态”

传统简版 ReAct 往往主要依赖：

- 用户消息
- 历史消息
- 一段 system prompt

而 DeerFlow 把会话提升成运行时状态对象。除了消息之外，还可以承载：

- `title`
- `artifacts`
- `sandbox`
- `thread_data`
- `todos`
- `uploaded_files`
- `viewed_images`

这种设计的直接收益是：

- 更适合长任务和多步骤任务
- 更适合处理中间产物
- 更适合文件型会话
- 更适合“一个会话即一个工作空间”的交互模式

### 3. 业务系统可以统一编排跨系统能力

DeerFlow 的工具层可以把多种能力面统一收束到同一套调用界面中：

- 业务方自定义工具
- MCP tools
- DeerFlow builtin tools
- 未来其他外部能力源

这会使业务系统更容易构建这样的智能工作流：

- 客服 Agent 跨 CRM、ERP、工单系统
- 运营 Agent 跨 BI、广告平台、内容后台
- 内部 Copilot 跨知识库、代码仓、审批流

核心变化不是“多了几个工具”，而是“原来分散在多个系统里的操作面被统一调度起来了”。

### 4. 可以把业务 SOP 知识化、文件化

接入 `skills` 之后，业务流程、分析方法、排障套路和任务分解方式，不必都堆在一个巨大的 prompt 里。

它们可以被沉淀为：

- `SKILL.md`
- 参考文档
- 模板
- 最佳实践说明

这样带来的好处是：

- 更易维护
- 更容易审阅
- 更容易复用到多个 Agent
- 更适合让业务团队参与维护

### 5. 可以自然支持子任务拆解和并发执行

如果某些业务流程天然就是多阶段、多系统、多步骤的，DeerFlow 的 subagent 机制会比普通 ReAct 的“单轮思维链 + tool call”更有结构。

这意味着系统更容易做出这样的能力：

- 先查用户，再查订单，再查异常履约，再总结
- 并行查多个系统后汇总
- 让一个 agent 负责分析，另一个 agent 负责执行

从业务形态上看，这使 Agent 从“单线程问答器”更接近“轻量工作流编排器”。

### 6. 可以把长期记忆从聊天记录里分离出来

集成 DeerFlow 后，如果再结合你们现有的消息存储和外部记忆系统，例如 `mem0`，就可以形成三层分工：

- 消息表：原始对话、审计、搜索
- checkpointer：会话运行时状态
- memory：长期事实、偏好、背景知识

这会显著优于“所有上下文都塞进消息历史”的做法。

## 二、集成后会受到哪些限制

### 1. 必须接受 DeerFlow 的运行时契约

一旦接入 DeerFlow，业务系统就要接受一套新的抽象边界：

- 会话主键是 `thread_id`
- 多轮状态依赖 checkpointer
- Agent 能力由 tools / middleware / prompt 共同组成
- 不是所有上下文都应该放进 messages

这意味着你们的 agent 调用方式，不再只是一个简单的：

`chat(messages) -> answer`

而会变成：

`runtime(state, thread_id, tools, middleware, prompt, memory) -> result`

### 2. skills 不能替代真正的执行能力

skills 很强，但它们的本质仍然是：

- 工作流知识包
- 操作说明书
- 使用工具的策略模板

skills 并不是：

- 权限控制层
- 事务执行层
- 业务规则引擎
- 系统集成层

也就是说，skill 可以告诉模型“应该怎么做”，但不能替代真正去做这件事的工具、服务和业务逻辑。

### 3. 如果深度使用内建能力，会逐步依赖 DeerFlow 的配置体系

当系统开始大量使用 DeerFlow 默认的：

- tools
- MCP
- skills
- memory
- prompt 装配

那么业务系统就会逐步依赖它的这些约定：

- `config.yaml`
- `extensions_config.json`
- skills 目录结构
- agent 目录结构

这不是问题本身，但说明系统治理方式会从“纯代码显式装配”部分转向“代码 + 文件配置 + 目录资产”混合驱动。

### 4. 某些能力当前仍偏进程内

当前实现里，有些能力还更接近单进程友好，而不是天然分布式友好。

典型例子包括：

- subagent 的后台任务表和线程池
- memory update queue 的 debounce 计时器

这意味着：

- 单体部署或小规模多副本场景没有问题
- 大规模、多 worker、强一致后台任务体系下，需要你们自己做更强的分布式托管与观测

### 5. prompt、skills 和 runtime 会形成双层系统

接入 DeerFlow 后，团队不再只维护 Python 代码，还会同时维护：

- prompt
- skills
- `SOUL.md`
- 工具定义
- middleware 行为

这带来新的治理要求：

- 谁负责业务规则
- 谁负责技能文件
- 谁负责工具协议
- 谁负责提示词回归

如果这层治理没有建立起来，就会出现“代码和文本资产漂移”的问题。

## 三、这种 Agent 框架的设计模式是什么

DeerFlow 不是单一模式，而是几种架构模式叠加形成的 Agent runtime。

### 1. Ports-and-Adapters / Hexagonal Architecture

这是最核心的模式。

DeerFlow 的内核不直接依赖具体业务系统，而是通过一组边界对接宿主环境：

- 模型工厂
- 工具接口
- MCP adapters
- sandbox provider
- memory storage
- checkpointer

这意味着 DeerFlow 很适合作为“被嵌入的运行时”，而不是只能作为完整应用运行。

### 2. Factory Pattern

工厂模式在 DeerFlow 中非常明显：

- `create_deerflow_agent()`
- `create_chat_model()`
- `make_lead_agent()`

这使得“如何创建 Agent”与“Agent 运行时到底有哪些能力”被显式分开。

### 3. Middleware Pipeline / Chain of Responsibility

DeerFlow 的很多平台能力并不是写死在 agent 主逻辑里，而是通过 middleware 按顺序串联。

这让它天然适合承载：

- 安全治理
- 记忆排队
- 标题生成
- loop detection
- clarification
- sandbox 生命周期

这使 DeerFlow 更像一个“带治理层的 runtime”，而不是只有模型和工具的最小代理框架。

### 4. Strategy / Provider Pattern

很多底层能力都通过 provider 或 storage 接口抽象出来：

- sandbox provider
- memory storage
- checkpointer backend
- model provider

这使它对“已有基础设施的业务系统”非常友好，因为可以替换底层实现，而不一定要接受所有默认实现。

### 5. Registry / Plugin-Oriented Architecture

DeerFlow 的工具、MCP、skills 本质上都带有注册表与插件化特征：

- 工具通过 `use` 字段反射加载
- MCP 从扩展配置中发现
- skills 从目录中扫描

这让平台扩展能力很强，但也要求团队建立稳定的命名、版本和开关治理方式。

### 6. Stateful Workflow Runtime

DeerFlow 不是单轮 prompt 调用框架，而是带状态的工作流运行时。

这也是它比简单 ReAct 更适合复杂业务系统的关键原因：

- 它更擅长承载中间状态
- 更适合多步骤任务
- 更适合 tool-heavy 场景

## 四、未来扩展新业务是否方便

总体结论是：方便，但前提是业务扩展要落到正确的层上。

未来新增业务时，最重要的问题不是“能不能扩”，而是“这次扩展应该放在哪一层”。

### 适合扩展到 DeerFlow 的业务内容

以下内容通常非常适合复用 DeerFlow：

- 新的 Agent 变体
- 新的工具集
- 新的子任务拆解方式
- 新的 SOP / 分析模板
- 新的长期记忆策略
- 新的 MCP 能力接入

### 不适合放进 DeerFlow 核心的内容

以下内容不建议被 DeerFlow 替代：

- 业务真相源
- 鉴权与租户隔离
- 数据库事务
- 权限校验
- 核心业务规则引擎
- 核心审计链路

这些仍应留在业务系统自己的 service / domain 层。

## 五、未来业务都可以替换成 skills 吗

不能，也不应该。

这是集成 DeerFlow 时最容易产生误判的地方。

### skills 适合承载什么

skills 最适合承载的是：

- 业务 SOP
- 分析框架
- 判断策略
- 操作步骤
- 何时调用哪些工具的建议
- 特定任务的最佳实践

可以把它理解成：

`skill = 一个可维护的业务操作手册`

### skills 不适合承载什么

以下内容不适合 skills 化：

- 真正的数据写操作
- 跨系统事务
- 幂等控制
- 参数校验
- 权限判断
- 审计记录
- 核心业务约束

这些更应该落在：

- tool
- MCP
- middleware
- 业务 service / domain 层

### 最合理的分工方式

未来扩展业务时，建议用下面这张分层判断表：

| 扩展内容 | 最适合放置的位置 |
|---|---|
| 业务知识、SOP、分析套路 | `skills` |
| 查询、写入、执行动作 | `tool` 或 `MCP` |
| 鉴权、审计、限流、回退 | 业务后端或 middleware |
| 长期记忆策略 | `MemoryStorage` / 外部 memory 系统 |
| 身份、语气、角色行为 | system prompt / `SOUL.md` |
| 新 Agent 变体 | prompt + tool_groups + skills 组合 |

因此，正确的答案不是“未来都替换成 skills”，而是：

- 可执行业务能力尽量 tool 化 / MCP 化
- 可迁移业务知识尽量 skill 化
- 可治理的运行时行为尽量 middleware 化

## 六、这种集成最值钱的地方

如果这种集成做得好，长期最有价值的并不是“系统用了 DeerFlow”，而是业务方会形成一套真正可复用的智能资产层：

- 统一的 Agent runtime
- 统一的工具执行面
- 统一的 MCP 接入面
- 可维护的业务 skills 资产
- 可替换的长期记忆层
- 可复制的多 Agent 构建方式

这意味着未来新增一个业务 Agent，不再是从零造一个 Agent，而更像是：

- 选模型
- 选工具
- 挂 skills
- 选 memory 策略
- 配 persona
- 接入现有业务服务

这就是平台化的真正收益。

## 七、最终结论

这条集成路线非常适合以下类型的团队：

- 已有自己的前端
- 已有自己的后端
- 已有 MCP / tools / 业务服务
- 不想被完整 Agent 产品绑架
- 想把 Agent 能力沉淀成长期平台能力

但这条路线要避免两个误区：

1. 不要把 DeerFlow 当成“替代整个业务系统的一整套应用”
2. 不要把 skills 当成“所有未来业务扩展的万能容器”

更准确的定位应该是：

- DeerFlow 负责 runtime orchestration
- 业务系统继续负责业务真相、鉴权、审计和接口协议
- tools / MCP 承载执行能力
- skills 承载 SOP 和知识流程
- memory 承载长期认知层

如果按这个边界来做，DeerFlow 集成后最有可能形成的是一个“可持续扩展的 Agent 平台层”，而不是一个短期堆出来的智能功能点。
