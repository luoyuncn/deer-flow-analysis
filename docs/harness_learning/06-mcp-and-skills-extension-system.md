# Harness Learning 06: MCP 与 Skills 扩展系统

这一篇研究 DeerFlow harness 最像“开放平台”的一面：扩展系统。

重点是两条线：

- MCP：把外部能力服务器接入为工具
- Skills：把工作流知识包注入为系统提示上下文

## 1. 这层解决什么问题

如果 harness 想成为可复用 runtime，就不能只依赖内建工具。

它必须能让外部能力以标准方式进入系统，并且让行为知识也能以标准方式进入系统。

DeerFlow 把两类扩展分开处理：

- MCP = 可执行能力扩展
- Skills = 工作流知识扩展

## 2. MCP：能力扩展

关键文件：

- [mcp/client.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/mcp/client.py)
- [mcp/tools.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/mcp/tools.py)
- [extensions_config.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/config/extensions_config.py)

### 2.1 配置模型

`extensions_config.json` 中的 `mcpServers` 描述 MCP server。

支持三类 transport：

- `stdio`
- `sse`
- `http`

说明 DeerFlow 的 MCP 不是单 transport 假设，而是一个统一接入层。

### 2.2 运行时装配

`build_servers_config()` 会把配置文件里的 server 描述转换成 `MultiServerMCPClient` 可理解的参数。

`get_mcp_tools()` 再负责：

- 初始化多 server MCP client
- 获取 tools
- 处理 OAuth/header 注入
- 把 async tool 包装成兼容同步调用的 wrapper

### 2.3 为什么这很重要

这意味着 DeerFlow 不需要知道外部能力的业务语义，只需要把它们变成标准工具面。

## 3. Skills：知识扩展

关键文件：

- [skills/loader.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/skills/loader.py)
- [prompt.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/lead_agent/prompt.py)

### 3.1 Skill 的本质

在 DeerFlow 里，skill 不是“代码插件”，而是“带元数据的工作流知识包”。

它通常是一个目录，里面至少有一个 `SKILL.md`。

### 3.2 Loader 做什么

`load_skills()` 会：

- 扫描 `public/` 和 `custom/`
- 找到 `SKILL.md`
- 解析 skill 元数据
- 从 `extensions_config.json` 读取是否启用

这说明 skill 的启停是运行时可配置的，不需要改代码。

### 3.3 Skill 如何生效

skill 不是直接被执行，而是被注入到 system prompt 的 skills section 中，变成模型行为约束与工作流程提示的一部分。

所以 skill 更像：

```text
可热加载的 prompt-level workflow package
```

## 4. MCP 与 Skills 的分工

这是最值得记住的边界。

### MCP 负责

- 能做什么
- 能调用哪些外部系统
- 能返回哪些结构化结果

### Skills 负责

- 什么时候做
- 先做什么后做什么
- 用哪些工具更合适
- 遇到什么情况该如何处理

也可以这样理解：

- MCP = 手和脚
- Skills = 经验与流程知识

## 5. `extensions_config.json` 为什么是独立文件

因为：

- MCP server 和 skill 开关更像“运行时扩展配置”
- 它们需要被 Gateway 或外部管理界面动态修改
- 不适合和大而稳定的 `config.yaml` 混在一起

这是一种很典型的“核心配置 / 扩展配置”分层。

## 6. 外部集成时这层意味着什么

你接 DeerFlow 时，最自然的扩展策略通常是：

- 把已有外部系统包装成 MCP server
- 把已有操作手册/业务 SOP 写成 skill

这样你就把：

- 可执行能力
- 业务工作流知识

分别放到了最合适的层。

## 7. 学完这一篇你应该能回答

1. 为什么 MCP 和 skill 必须分成两层
2. `extensions_config.json` 为什么值得独立存在
3. MCP tools 是怎样进入 agent 的
4. skills 为什么不是“工具插件”

## 最短总结

DeerFlow 的扩展系统把“外部能力接入”和“工作流知识注入”拆成了 MCP 与 Skills 两条线，从而让 harness 同时具备开放能力面和可演化行为面。
