# Harness Learning 01: Checkpointer 与状态模型

这份笔记聚焦 DeerFlow harness 里最容易混淆、但也最值得先吃透的一组概念：

- `ThreadState`
- `thread_id`
- `checkpointer`
- thread filesystem
- memory

如果把这五者的边界看清，后面再读 middleware、client、subagent、sandbox，会顺很多。

## 关系总图

```mermaid
flowchart TD
    A[thread_id<br/>调用上下文里的线程键]
    B[ThreadState<br/>LangGraph 状态面]
    C[Checkpointer<br/>memory / sqlite / postgres]
    D[Thread Filesystem<br/>threads/thread_id/user-data/<br/>workspace / uploads / outputs]
    E[Memory<br/>memory.json / agent memory]

    A -->|configurable 中传入| C
    B -->|状态快照保存与恢复| C
    A -->|目录隔离键| D
    B -.运行中可能引用文件路径/产物.-> D
    B -.短期状态可被抽取/注入.-> E

    F[DeerFlowClient / LangGraph Server] -->|创建 agent 时注入 saver| C
    F -->|create_agent(..., state_schema=ThreadState)| B
```

## 一句话先记住

`ThreadState` 定义“要保存什么”，`thread_id` 定义“这是谁的状态”，`checkpointer` 负责“按这个键把状态存起来并取回来”，thread filesystem 负责“线程级文件隔离”，memory 负责“跨回合长期偏好与事实”。

## 1. `ThreadState`：被保存的状态到底是什么

代码位置：

- `backend/packages/harness/deerflow/agents/thread_state.py`

`ThreadState` 是 DeerFlow 在 LangGraph 里的状态总线。它继承 `AgentState`，在默认消息状态之外，又扩展了 DeerFlow 自己关心的运行时字段：

- `sandbox`
- `thread_data`
- `title`
- `artifacts`
- `todos`
- `uploaded_files`
- `viewed_images`

这里最关键的理解是：

`checkpointer` 保存的不是整个 Python 进程，也不是线程目录里的所有文件，而是 LangGraph 这张图当前维护的 state。

其中有两个 reducer 很值得注意：

- `merge_artifacts()`：让 `artifacts` 作为累积列表合并并去重
- `merge_viewed_images()`：让 `viewed_images` 支持增量合并，也支持用空字典清空

这说明 `ThreadState` 不是“一次请求临时返回的数据”，而是被设计成可演化、可延续的会话状态面。

### 你读这一层时要盯住什么

- 哪些字段属于 agent 可观察状态
- 哪些字段是会跨回合延续的
- reducer 是覆盖、合并、还是清空语义

### 它不等于什么

- 不等于 `thread_id`
- 不等于线程目录里的文件
- 不等于 `memory.json`

## 2. `provider.py`：sync checkpointer 怎么创建、复用、释放

代码位置：

- `backend/packages/harness/deerflow/agents/checkpointer/provider.py`

这个文件是同步版本的 checkpointer 工厂，核心做了三件事。

### 2.1 按 backend 创建 saver

支持三种 backend：

- `memory` -> `InMemorySaver`
- `sqlite` -> `SqliteSaver`
- `postgres` -> `PostgresSaver`

入口函数是 `_sync_checkpointer_cm(config)`。

这里有两个实现细节要记住：

- SQLite/Postgres 在返回前都会执行 `setup()`
- SQLite 相对路径会先经 `resolve_path()` 解析到 DeerFlow 的 `base_dir`

### 2.2 提供进程级单例

`get_checkpointer()` 会缓存一个全局 `_checkpointer`。

这代表：

- 第一次调用时创建 saver
- 后续调用复用同一个 saver
- 适合 embedded client 这类同步常驻进程

### 2.3 提供一次性上下文

`checkpointer_context()` 不走全局缓存，而是每次 `with` 新建一个 saver，用完即关。

这更适合：

- CLI
- 测试
- 需要确定性清理资源的短生命周期场景

### 你读这一层时要盯住什么

- `_sync_checkpointer_cm`
- `get_checkpointer`
- `checkpointer_context`
- `reset_checkpointer`

## 3. `async_provider.py`：为什么 LangGraph Server 用 async 版本

代码位置：

- `backend/packages/harness/deerflow/agents/checkpointer/async_provider.py`
- `backend/langgraph.json`

异步版本不是把 sync 代码简单改成 `async`，而是把 saver 生命周期绑定到 server 生命周期。

核心入口有两个：

- `_async_checkpointer(config)`：根据 backend 构建 async saver
- `make_checkpointer()`：对外暴露 async context manager

LangGraph Server 不是在 `make_lead_agent()` 里显式传入 checkpointer，而是在 `langgraph.json` 里配置：

```json
{
  "graphs": {
    "lead_agent": "deerflow.agents:make_lead_agent"
  },
  "checkpointer": {
    "path": "./packages/harness/deerflow/agents/checkpointer/async_provider.py:make_checkpointer"
  }
}
```

也就是说：

- `make_lead_agent()` 负责 agent 装配
- `langgraph.json` 负责 server 级 persistence 装配

### 这里最值得记住的点

- 未配置 `checkpointer` 时，async 版本也返回 `InMemorySaver`，不是 `None`
- async saver 更适合跟数据库连接池、server lifespan 一起管理

## 4. `client.py`：embedded client 怎么把 checkpointer 接进 agent

代码位置：

- `backend/packages/harness/deerflow/client.py`

`DeerFlowClient` 的 `_ensure_agent()` 是这层的关键入口。

逻辑顺序是：

1. 先准备 `create_agent(...)` 的参数
2. 如果 `self._checkpointer` 有显式传入，就优先使用它
3. 如果没有，就 fallback 到 `get_checkpointer()`
4. 最后把 `checkpointer` 注入 `create_agent(...)`

这说明 embedded 模式下的优先级是：

1. 显式传入的 saver
2. 配置驱动创建出来的 saver
3. 默认 `InMemorySaver`

### 这里最容易混淆的点

相同 `thread_id` 不等于自动保留上下文。

必须同时满足两件事：

- 调用时使用相同的 `thread_id`
- agent 实际挂上了可工作的 `checkpointer`

否则：

- 线程目录仍然可以按 `thread_id` 隔离
- uploads / outputs 仍然能落在对应线程目录
- 但 LangGraph 对话状态不会自动接续

## 5. 测试：作者真正承诺了哪些行为

代码位置：

- `backend/tests/test_checkpointer.py`
- `backend/tests/test_checkpointer_none_fix.py`

测试是最接近真实契约的地方。当前这组测试能反推出作者明确要求这些行为成立。

### 5.1 默认路径必须可运行

未配置 `checkpointer` 时：

- sync 返回 `InMemorySaver`
- async 返回 `InMemorySaver`

这说明默认体验是“先能跑起来”，而不是“没有配置就报错”。

### 5.2 sync 版本是单例语义

`get_checkpointer()` 多次调用返回同一个实例；`reset_checkpointer()` 后会重建。

### 5.3 client fallback 是明确设计

`DeerFlowClient(checkpointer=None)` 会回退使用 `get_checkpointer()`。

### 5.4 显式注入优先级最高

如果你手工给 client 传了 saver，它会覆盖配置路径上的 saver。

### 5.5 `None` 是被明确禁止的错误路径

专门有回归测试保证 context manager 返回的是可工作的 saver，而不是 `None`，因为 LangGraph 上层会直接调用 saver 的 `list()` 或 `alist()`。

## 五者边界表

| 概念 | 作用 | 是否默认跨回合 | 是否默认跨进程重启 | 主要代码位置 |
| --- | --- | --- | --- | --- |
| `ThreadState` | LangGraph 状态面 | 是，前提是有 saver | 取决于 saver backend | `agents/thread_state.py` |
| `thread_id` | 状态定位键、线程资源隔离键 | 只是键，不自己保存状态 | 只是键，不自己保存状态 | `client.py` / runtime config |
| `checkpointer` | 保存和恢复状态快照 | 是 | `memory` 否，`sqlite/postgres` 是 | `agents/checkpointer/` |
| thread filesystem | 线程文件隔离 | 是 | 通常是，只要底层文件还在 | `config/paths.py` |
| memory | 长期事实/偏好记忆 | 是 | 通常是 | `agents/memory/` |

## 最常见的三个误区

### 误区 1：有 `thread_id` 就一定能续聊

不对。`thread_id` 只是定位键，不负责保存状态。真正保存状态的是 `checkpointer`。

### 误区 2：checkpointer 会保存线程目录里的所有文件

不对。它保存的是 LangGraph state snapshot，不是整个文件系统。

### 误区 3：memory 和 checkpointer 是一回事

不对。

- checkpointer 偏“短中期会话状态”
- memory 偏“长期事实和偏好”

二者都能跨回合，但职责完全不同。

## 最短总结

你可以把这套模型压缩成一句话：

`ThreadState` 负责定义状态面，`thread_id` 负责给状态命名，`checkpointer` 负责持久化状态，thread filesystem 负责文件隔离，memory 负责长期记忆。

## 建议下一步

如果继续往下学，最自然的下一篇应该是：

1. middleware 如何读写 `ThreadState`
2. `thread_id` 如何在 sandbox / uploads / subagent 之间流动
3. `checkpointer` 与 LangGraph 实际 checkpoint record 的结构关系
