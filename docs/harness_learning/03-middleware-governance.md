# Harness Learning 03: Middleware 治理骨架

这一篇研究 DeerFlow 里最有“平台味”的一层：middleware。

如果你想真正理解 DeerFlow 的思想，而不是只看 prompt 和 tools，这一层必须吃透。

## 1. 它解决什么问题

在 agent runtime 里，有很多行为不是“业务任务本身”，但又必须被系统稳定处理：

- 线程目录初始化
- 上传文件注入
- sandbox 生命周期
- tool error 转换
- clarification 中断
- title 生成
- memory 写入队列
- 上下文压缩
- 循环检测
- 子代理并发限制

这些逻辑如果都写到 prompt 里，会变得不可控；如果都塞进工具里，又会耦合得很厉害。

DeerFlow 的答案是：

```text
把系统级治理逻辑放进 middleware
```

## 2. 全局位置

关键文件：

- [agent.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/lead_agent/agent.py)
- [tool_error_handling_middleware.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/middlewares/tool_error_handling_middleware.py)
- [thread_data_middleware.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py)
- [clarification_middleware.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/middlewares/clarification_middleware.py)

## 3. DeerFlow 的 middleware 不是装饰品

主链里，middleware 顺序是隐含协议。

在 [agent.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/lead_agent/agent.py) 里，作者明确写了顺序注释：

- `ThreadDataMiddleware` 要先于 `SandboxMiddleware`
- `UploadsMiddleware` 需要 thread data
- `SummarizationMiddleware` 要足够靠前
- `TitleMiddleware` 在首轮交换后生成标题
- `MemoryMiddleware` 在 title 后
- `ClarificationMiddleware` 必须最后

这说明 middleware 顺序不是“谁先谁后都差不多”，而是行为语义本身的一部分。

## 4. 三层结构来理解 middleware

### 4.1 基础运行时 middleware

这是共享基底。

`build_lead_runtime_middlewares()` 最终来自 `_build_runtime_middlewares(...)`，其中包含：

- `ThreadDataMiddleware`
- `UploadsMiddleware`（lead only）
- `SandboxMiddleware`
- `DanglingToolCallMiddleware`（lead only）
- Guardrail middleware（如果配置了）
- `SandboxAuditMiddleware`
- `ToolErrorHandlingMiddleware`

这是 DeerFlow 的 runtime substrate。

### 4.2 lead agent 专属治理 middleware

在基础链之上，lead agent 还会追加：

- `SummarizationMiddleware`
- `TodoMiddleware`
- `TokenUsageMiddleware`
- `TitleMiddleware`
- `MemoryMiddleware`
- `ViewImageMiddleware`
- `DeferredToolFilterMiddleware`
- `SubagentLimitMiddleware`
- `LoopDetectionMiddleware`
- `ClarificationMiddleware`

### 4.3 subagent 会复用一部分，但不是全部

subagent 通过 `build_subagent_runtime_middlewares()` 复用基础运行时链，但不会照搬 lead 的整套治理链。

这说明 DeerFlow 把“共享 runtime”与“角色专属行为”分开了。

## 5. 重点 middleware 怎么理解

### 5.1 `ThreadDataMiddleware`

职责很清楚：

- 从 runtime context 或 configurable 里拿到 `thread_id`
- 计算线程目录相关路径
- 注入 `thread_data` 到 state

它的重要意义在于：

- 把线程资源路径标准化
- 为 uploads、sandbox、artifacts 提供统一线程上下文

### 5.2 `ToolErrorHandlingMiddleware`

这是非常关键的治理点。

它把工具异常转成 `ToolMessage`，而不是直接炸掉整个运行。

设计思想是：

```text
让 agent 在工具失败时还有机会利用错误上下文继续推理
```

这很像一个健壮 runtime，而不是脆弱脚本。

### 5.3 `ClarificationMiddleware`

当模型调用 `ask_clarification` 时，它不是像普通工具那样执行，而是被拦截并转成：

- 一个格式化过的 `ToolMessage`
- 一个 `Command(goto=END)` 中断当前执行

这里的思想非常值得记住：

clarification 不是普通工具调用，而是 runtime control flow。

### 5.4 `MemoryMiddleware`

它不是 memory 存储本身，而是把会话内容送进 memory 更新流程的治理层入口。

### 5.5 `LoopDetectionMiddleware`

它的价值不在“多高级”，而在于说明 DeerFlow 把 agent 的失控风险当成 runtime 治理问题。

## 6. 设计思想总结

### 6.1 系统级行为从 prompt 中剥离

middleware 让很多本应由系统保证的行为，不再依赖模型“乖乖遵守提示词”。

### 6.2 中断、恢复、错误降级都属于 runtime 语义

尤其是 clarification 和 tool error，这两个最能说明 DeerFlow 的中间件不是普通包装器。

### 6.3 middleware 顺序本身就是协议

改顺序很可能导致：

- thread data 不可用
- sandbox 无法初始化
- clarification 不再能正确中断
- memory/title 行为错位

## 7. 外部集成时怎么用这层

如果你是外部宿主应用，一般有三种介入方式：

- 直接复用默认 middleware 链
- 在 `create_deerflow_agent()` 里通过 `features` 控制
- 通过 `extra_middleware` 插入你自己的治理逻辑

这意味着 DeerFlow 很适合做平台型内核，因为它允许宿主在“治理层”扩展，而不是只能改 prompt。

## 8. 学完这一篇你应该能回答

1. DeerFlow 为什么把很多逻辑放进 middleware
2. 基础 runtime middleware 和 lead-only middleware 的区别是什么
3. 为什么 clarification 其实是控制流问题
4. 为什么 middleware 顺序是隐含协议

## 最短总结

在 DeerFlow 里，middleware 不是“可选增强”，而是把线程资源、错误处理、澄清中断、上下文治理和执行安全组织成运行时协议的治理骨架。
