# Harness Learning 08: Subagents 与 ACP

这一篇研究 DeerFlow 的结构性扩展能力。

很多系统会把“调用另一个 agent”当成普通技巧，但 DeerFlow 把它做成了 runtime 一等能力。

## 1. 这层解决什么问题

主 agent 并不适合亲自做所有事，尤其是：

- 可并行拆解的复杂任务
- 专门的 bash/代码探索任务
- 独立能力单元调用

DeerFlow 的方案是两层：

- `subagents`：DeerFlow runtime 内部的子代理能力
- `ACP`：外部/独立代理能力调用入口

## 2. Subagent 的基本结构

关键文件：

- [subagents/config.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/subagents/config.py)
- [subagents/executor.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/subagents/executor.py)
- `tools/builtins/task_tool.py`

### 2.1 配置层

`SubagentConfig` 描述一个子代理：

- 名称
- 描述
- system prompt
- 可用工具
- 禁用工具
- 模型策略
- 最大 turns
- timeout

这说明 subagent 不是随便 new 一个 agent，而是有独立 profile 的。

### 2.2 执行层

`SubagentExecutor` 会：

- 继承父 agent 的部分上下文
- 过滤工具
- 创建子 agent
- 在线程池中后台执行
- 维护任务结果状态

它不是普通函数调用，而更像受控的后台 agent job。

### 2.3 状态继承

子代理会继承这些关键上下文：

- `sandbox_state`
- `thread_data`
- `thread_id`
- 父模型名（可选）
- trace id

这说明 DeerFlow 想实现的是：

```text
资源上下文可共享，认知执行相对独立
```

## 3. 为什么需要 `SubagentLimitMiddleware`

如果模型一次性发起过多并行子任务，系统就可能失控。

DeerFlow 不是把限制写在提示词里赌模型听话，而是用 middleware 在运行时强制约束并发数量。

这再次体现了它的思想：

系统约束优先由 runtime 保证。

## 4. ACP 是什么

ACP 更接近“外部代理能力接入”。

它和内建 subagent 不完全是一回事：

- subagent 更像 harness 内部 delegation
- ACP 更像外部 agent/service capability

二者都扩展 agent 能力，但职责边界不同。

## 5. 设计思想

### 5.1 delegation 是 runtime capability，不是 prompt 技巧

这是 DeerFlow 和很多“会调用 task 工具”的系统最不同的地方之一。

### 5.2 共享资源，不共享一切

子代理会继承线程上下文和资源面，但不是简单共享整套 lead runtime。

### 5.3 并发控制属于治理层，不属于提示词层

通过 `SubagentLimitMiddleware` 做硬限制，是很平台化的选择。

## 6. 外部集成时怎么理解这层

如果你的宿主应用未来也要做：

- 任务拆解
- 多代理协作
- 领域专用 agent

那么 DeerFlow 这层很值得复用。

但如果当前目标只是先接一个单代理聊天系统，那么 subagent/ACP 可以晚一点再启用。

## 7. 学完这一篇你应该能回答

1. subagent 为什么不是普通工具封装
2. 父 agent 与子 agent 哪些上下文共享，哪些不共享
3. 为什么并发限制要放在 middleware 层
4. subagent 与 ACP 的职责边界是什么

## 最短总结

DeerFlow 把多代理协作做成了运行时能力：子代理拥有独立 profile 和执行流，但又能在父线程资源上下文中协同工作，这正是它比普通单 agent 框架更像“runtime kernel”的地方。
