# Harness Learning 07: Memory、Title、Summarization

这一篇研究 DeerFlow 对“上下文长期化”和“对话可维护性”的处理方式。

这三块很容易和 `checkpointer` 混淆，但它们职责完全不同。

## 1. 这层解决什么问题

`checkpointer` 解决的是：

- 图状态快照保存与恢复

而这一篇的三块分别解决：

- `memory`：长期语义记忆
- `title`：会话标题生成
- `summarization`：上下文压缩与裁剪

## 2. 三者的边界

### `memory`

关注的是长期偏好、事实、用户背景。

### `title`

关注的是对话可读性和线程命名。

### `summarization`

关注的是上下文窗口治理。

这三者都可能跨回合，但只有 `memory` 真正偏长期语义存储。

## 3. Memory 系统怎么理解

关键文件：

- [updater.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/agents/memory/updater.py)
- `storage.py`
- `queue.py`

### 3.1 它不是原始聊天记录

DeerFlow memory 存的不是 message log，而是抽取后的语义结构。

这和“用户可见聊天历史”是两层。

### 3.2 它通过 updater 驱动

`MemoryUpdater` 会：

- 读取会话消息
- 通过模型提炼长期事实
- 做去重、清理和存储

这说明 memory 是一种“抽取型长期记忆”，不是简单 append log。

### 3.3 它还有存储抽象

通过 storage 层，外部应用可以替换底层存储。

这也是 DeerFlow 可集成性的关键。

## 4. Title 为什么做成 middleware

title 不是核心业务状态，但它属于会话运行时增强。

把它做成 middleware 有两个好处：

- 生命周期清晰
- 不污染主业务逻辑和工具逻辑

这也说明 DeerFlow 对“会话体验增强”采用的是 runtime augmentation，而不是业务字段硬编码。

## 5. Summarization 为什么存在

当对话很长时，仅靠 checkpointer 保存完整状态并不能解决上下文成本问题。

还需要主动做：

- 截断
- 摘要
- 保留重要片段

所以 summarization 是“上下文治理”，不是长期记忆。

## 6. 三者与 checkpointer 的关系

最容易混淆的就是这一点。

### checkpointer

保存 state snapshot

### memory

保存长期语义事实

### summarization

控制当前上下文规模

### title

增强线程可读性

你可以把它们理解为四层不同职责，不要互相替代。

## 7. 外部集成时怎么取舍

如果宿主系统已经有：

- 自己的消息表
- 自己的记忆系统
- 自己的线程命名策略

那么接入 DeerFlow 时可以有选择地：

- 保留 checkpointer
- 替换 memory storage
- 关闭或保留 summarization
- 决定是否沿用 title middleware

这也再次证明 DeerFlow 是可裁剪 runtime，不是只能整套照搬。

## 8. 学完这一篇你应该能回答

1. 为什么 memory 不是聊天记录
2. 为什么 summarization 不是长期记忆
3. 为什么 title 比较适合做 middleware
4. 这三块和 checkpointer 的职责边界是什么

## 最短总结

在 DeerFlow 里，memory、title、summarization 共同服务于“长期理解、可维护上下文和更好的线程体验”，但它们都不是 checkpointer 的别名，而是不同层次的运行时能力。
