# Harness Learning 05: Sandbox 与线程资源模型

这一篇研究 DeerFlow 的执行边界。

如果你把 `checkpointer` 看成“状态持久化面”，那 sandbox 与 thread filesystem 就是“执行资源面”。

## 1. 它解决什么问题

agent 要想真正做事，不能只有 messages 和 prompt，还需要：

- 工作目录
- 上传文件目录
- 输出目录
- 命令执行环境
- 文件读写映射

DeerFlow 把这些问题组织成：

- 线程目录
- 虚拟路径
- sandbox provider
- middleware 生命周期

## 2. 三个关键层

关键文件：

- [paths.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/config/paths.py)
- [thread_data_middleware.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/middlewares/thread_data_middleware.py)
- [sandbox/middleware.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/sandbox/middleware.py)

## 3. 线程资源目录是什么样

`Paths` 定义了标准目录布局：

```text
{base_dir}/threads/{thread_id}/
  user-data/
    workspace/
    uploads/
    outputs/
  acp-workspace/
```

这几个目录各自承担不同职责：

- `workspace`：agent 工作目录
- `uploads`：用户上传
- `outputs`：最终产物
- `acp-workspace`：ACP 代理专属线程工作区

这说明 DeerFlow 不把文件资源散落在项目各处，而是统一以 `thread_id` 为隔离边界。

## 4. 虚拟路径系统

agent 在 sandbox 中看到的不是宿主真实路径，而是类似：

- `/mnt/user-data/workspace`
- `/mnt/user-data/uploads`
- `/mnt/user-data/outputs`
- `/mnt/acp-workspace`

这层映射的价值是：

- 给模型稳定、一致的工作路径抽象
- 屏蔽宿主真实目录结构
- 方便 local / container 两种后端切换

## 5. `ThreadDataMiddleware` 的作用

它在运行开始时负责两件事：

1. 找到 `thread_id`
2. 计算并注入线程目录路径到 `thread_data`

这让后续模块不需要各自重新理解线程目录结构。

这是一种很典型的 runtime normalization 设计。

## 6. `SandboxMiddleware` 的作用

`SandboxMiddleware` 负责把“线程”映射成“执行环境”。

它会：

- 通过 `thread_id` 向 provider 申请 sandbox
- 把 `sandbox_id` 写入 state
- 在合适的时机 release

这里最值得注意的是：

线程资源与执行环境虽然关联，但不是一回事：

- 线程目录是宿主侧资源
- sandbox 是执行环境抽象

## 7. 设计思想

### 7.1 资源面与状态面分离

`ThreadState` 是状态面；
thread filesystem / sandbox 是资源面。

这和 `checkpointer` 那篇形成呼应。

### 7.2 统一路径抽象，减少 prompt 和工具耦合

模型看到固定虚拟路径，工具与 provider 做真实路径解析。

### 7.3 线程是资源隔离单元

thread 不只是“聊天会话名”，还是：

- 文件空间隔离键
- sandbox 绑定键
- ACP workspace 绑定键

## 8. 外部集成时要特别注意什么

如果你把 DeerFlow 接进外部系统，必须明确这三层不要混：

- 你的业务 conversation/session id
- DeerFlow 的 `thread_id`
- 你的物理文件存储方案

最稳妥的方式通常是：

- conversation_id 和 thread_id 做 1:1 映射
- 让 DeerFlow 接管线程工作目录
- 宿主只负责更上层业务文件策略

## 9. 学完这一篇你应该能回答

1. thread filesystem 和 checkpointer 为什么是两层
2. 为什么需要虚拟路径，而不是直接暴露宿主路径
3. `ThreadDataMiddleware` 和 `SandboxMiddleware` 为什么必须配合
4. 为什么 `thread_id` 同时影响状态与资源，但不等于两者中的任意一个

## 最短总结

DeerFlow 把线程目录、虚拟路径和 sandbox provider 组织成了一套执行资源模型，使 agent 不仅能“思考”，还能在受控、可隔离的工作空间里稳定执行。
