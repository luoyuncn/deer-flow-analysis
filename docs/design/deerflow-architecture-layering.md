# DeerFlow 架构分层、边界与设计哲学

## 说明

这份文档基于当前仓库实现，对 DeerFlow 的架构分层、边界设计和整体设计哲学做一个集中总结。

从整体上看，DeerFlow 并不是传统的 MVC Web 应用，更适合把它理解成一个统一接入的 Agent 平台。它主要由四部分组成：

- 统一入口层
- 薄应用壳层
- 可复用的 Agent 运行时内核
- 一组可插拔的能力模块

项目自己对 2.0 的定位也很明确：它是一个 `super agent harness`，而不只是一个研究型应用。

## 系统拓扑

当前运行时主要分成四个面：

1. `nginx`
   作为统一入口。
   - `/api/langgraph/*` -> LangGraph Server
   - `/api/*` -> Gateway API
   - `/*` -> Next.js Frontend

2. `LangGraph Server`
   负责 Agent 执行、线程状态、流式响应，以及主运行时生命周期。

3. `Gateway API`
   负责非 Agent 运行面的 REST 能力，例如模型、技能、MCP 配置、上传、artifacts、memory 查看和线程本地文件清理。

4. `Frontend`
   负责产品交互层，并且有意识地把请求拆分到 LangGraph 和 Gateway 两个后端面。

这种拆法的价值在于：交互式 Agent 运行时和管理型 API 被明确分开，而不是堆在一个“大后端”里。

## 总体分层

### 1. 统一入口层

最外层是反向代理和路由层。

它的职责很克制：

- 提供一个稳定的统一入口
- 集中处理路由和跨域
- 让前端无需感知后端内部到底拆成几个服务

所以从产品角度看，用户面对的是一个系统；但从实现角度看，运行时其实是分层和分面的。

### 2. 薄应用壳层

后端的 `app/` 是典型的应用壳层。

它主要负责：

- 暴露 FastAPI 路由
- 对接 Slack、Telegram、Feishu 等 IM 渠道
- 把外部请求翻译成对 DeerFlow 运行时内核的调用

也就是说，`app/` 更偏“接入层”和“部署层”，而不是系统真正的核心领域层。

### 3. 运行时内核层

真正的核心在 `backend/packages/harness/deerflow/`。

这里包含的是 DeerFlow 作为一个可复用 Agent 平台所需要的基础能力：

- agent factory
- middleware orchestration
- sandbox abstraction
- tool assembly
- model factory
- MCP integration
- skills loading
- memory
- subagents
- embedded client

这层定义了 DeerFlow 的本质。Web UI 只是它的一种使用方式，不是它唯一的存在形式。

### 4. 能力模块层

在运行时内核内部，代码不是按传统业务实体切分的，而是按运行时能力切分：

- `agents`
- `sandbox`
- `tools`
- `models`
- `mcp`
- `skills`
- `memory`
- `subagents`
- `uploads`
- `config`

这说明它的架构思路是“平台优先”。系统首先考虑的是 Agent 需要哪些可组合能力，而不是先建一个业务对象模型。

### 5. 前端产品层

前端也做了类似的职责分离：

- `src/app`：页面和路由装配
- `src/components`：表现层和交互组件
- `src/core`：线程、上传、技能、模型、配置、API 集成等功能模块

这让前端在继续演化产品体验时，不至于把页面、组件和运行时逻辑全部混在一起。

## 最关键的边界

## 1. Harness 和 App 的边界

后端最硬的一条边界是：

- `deerflow.*` = 可复用的 harness
- `app.*` = 未发布的应用壳

依赖方向被明确限制为单向：

- `app` 可以 import `deerflow`
- `deerflow` 不能 import `app`

而且这不只是口头约定，仓库里有专门的测试来约束这件事。

这是整个项目最重要的边界，因为它保证了 DeerFlow 的运行时内核不会被接入层、FastAPI 路由或者 IM 渠道逻辑反向污染。

换句话说，团队不是把“能跑起来”当目标，而是有意识地维护“平台内核可独立存在”的结构。

## 2. 通用工厂和默认装配的边界

在 harness 内部还有一条很有价值的边界：

- `create_deerflow_agent()` 更像纯参数工厂
- `make_lead_agent()` 是面向产品默认行为的配置化装配入口

这意味着 DeerFlow 区分了两种东西：

- “如何构造一个 Agent”
- “产品默认如何运行这个 Agent”

这是一种成熟的设计，因为它避免了“默认实现即唯一实现”的架构塌缩。

## 3. Thread 级执行隔离边界

Thread 在 DeerFlow 里不是单纯的聊天历史概念，而是真实的执行隔离单元。

每个 thread 都有独立目录：

- `workspace`
- `uploads`
- `outputs`

Agent 看到的是 `/mnt/user-data/*` 下的虚拟路径，宿主机上再映射到 thread 对应的实际目录。

这条边界承担了三件事：

- 执行隔离
- 文件归属管理
- 路径安全控制

`Paths` 这一层把 thread id 校验、虚拟路径映射和 traversal 防护都集中起来，说明线程隔离是架构显式约束，而不是顺手实现出来的细节。

## 4. 前端通信边界

前端没有把所有请求都打到同一个后端接口面上，而是有意识地做了拆分：

- 会话执行和流式消息走 LangGraph SDK
- 模型、技能、上传等管理型能力走 Gateway REST API

这和后端本身的结构是一致的，也就是说前端不是在“掩盖”后端分层，而是在顺着后端的职责面去调用。

这会让系统在长期演进时更稳定，因为“聊天运行时”和“管理面能力”本来就是两种不同性质的流量。

## 核心执行模型

一次 DeerFlow 运行，本质上是四类东西被装配到一起：

1. 模型
2. 工具集合
3. 有顺序的中间件链
4. 系统提示词

这四者才是 DeerFlow 真正的组合模型。

其中最关键的是中间件链。

它不是一个附属实现细节，而是整个运行时行为的主骨架。典型职责包括：

- thread 目录初始化
- 上传文件注入
- sandbox 获取
- summarization
- todo 跟踪
- title 生成
- memory 更新
- 图像处理
- clarification 控制
- tool error 和 loop 控制

这说明 DeerFlow 的核心不是“一个超级 Agent 类”，而是“一个按顺序执行横切能力的 runtime pipeline”。

## 设计哲学

## 1. 平台优先，而不是单一功能优先

DeerFlow 2.0 的思路很明显：先做 Agent 运行平台，再做具体产品形态。

所以仓库里同时存在：

- 可发布的 harness
- embedded client
- 可配置 models
- 可配置 tools
- MCP 集成
- skills 机制
- 多种 sandbox provider

Web 前端是重要入口，但不是架构中心。

## 2. 配置驱动和反射装配，而不是硬编码

模型、工具、MCP、技能等大量能力都是配置驱动的。

这样做的直接收益是：

- 更容易切换 provider
- 更容易做工具组合
- 更容易按场景开关能力
- 更容易扩展而不改核心框架

当然代价也存在：动态性更强，意味着运行时的错误面会更复杂。所以代码里又配套做了较多配置解析、重载和错误提示优化。这是很典型的平台工程权衡。

## 3. 渐进加载，而不是一次性塞满上下文

DeerFlow 明显在避免“把所有能力一次性灌进模型上下文”。

技能是按需加载的，部分工具也支持 deferred loading。这背后的思路非常符合 LLM 系统工程：

- 控制上下文长度
- 只在需要时引入专门能力
- 提高系统的普适性和延展性

所以这里的“分层”不只是代码组织问题，也是上下文管理策略。

## 4. Thread 是执行单元，不只是会话单元

很多系统把 thread 只当成聊天记录容器。

DeerFlow 把 thread 同时当成：

- 存储隔离单元
- 执行环境绑定单元
- 上传归属单元
- artifact 归属单元
- sandbox 上下文单元

这是一个非常重要的架构选择，因为它让“对话 ID”变成了真实的 runtime boundary。

## 5. 安全是务实约束，不是口号

代码里可以明显看到一套务实的安全取向：

- thread id 校验
- 虚拟路径解析检查
- local sandbox 下默认限制 host bash
- 对高风险 artifact 内容类型强制下载

这说明团队很清楚自己在暴露的是高权限 Agent 能力，所以执行、文件和集成边界都被反复加固。

## 6. 进程分离优先于“大一统单体”便利

Gateway 和 LangGraph 是独立进程，但又通过共享配置文件和 mtime reload 机制协同。

这个选择很重要，因为它保证了：

- Gateway 可以持续保持“薄控制面”
- LangGraph 可以持续聚焦“执行面”
- 两者共享配置，但不直接耦死

代价是运行和部署稍复杂一些，但换来的是更清晰的系统模型。

## 哪些地方没有那么“纯”

这套架构总体很清晰，但它不是教条式的。

有两个点很值得注意：

1. 前端 `src/core` 不是完全纯粹的 domain 层。
   现在仍然有少量模块会依赖组件层类型。这说明前端更偏向“工程可交付”和“快速演进”，而不是严格学术意义上的分层洁癖。

2. 内部复用有时会跨过理想封装线。
   例如 embedded client 会复用一些运行时内部装配逻辑，而不是重新建立一整套彻底隔离的 public abstraction。

这两点我反而觉得是合理的。它说明团队知道哪些边界必须强守，哪些边界可以为复用效率和交付速度做务实让步。

## 最终总结

理解 DeerFlow 最好的方式是：

- `frontend` 是操作台
- `gateway` 是控制面
- `langgraph + deerflow harness` 是执行内核
- `sandbox`、`tools`、`skills`、`mcp`、`memory`、`subagents` 是能力层

所以这个项目本质上更像一个 Agent 运行环境，而不是一个普通 Web 应用。

它的架构始终围绕一个中心目标展开：

为 Agent 提供一个安全、可扩展、可组合，而且真正能执行工作的运行时。
