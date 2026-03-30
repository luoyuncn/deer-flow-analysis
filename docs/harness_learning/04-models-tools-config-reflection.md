# Harness Learning 04: Models、Tools、Config、Reflection

这一篇研究 DeerFlow 的“能力装配层”。

如果前两篇让你看到了 runtime 骨架，那么这一篇会回答：

```text
能力到底是怎么进到这个 runtime 里的？
```

## 1. 这层解决什么问题

一个可复用 harness 不能把：

- 模型类
- 工具变量
- 扩展能力
- 配置路径

都写死在代码里。

DeerFlow 的方案是：

- 用 config 描述能力
- 用 reflection 解析能力
- 用 assembler 把能力挂进 agent

## 2. 四个关键部件

关键文件：

- [models/factory.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/models/factory.py)
- [tools/tools.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/tools/tools.py)
- [app_config.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/config/app_config.py)
- [resolvers.py](d:/dev/github/deer-flow-analysis/backend/packages/harness/deerflow/reflection/resolvers.py)

## 3. Config 是能力声明，不只是参数文件

`config.yaml` 在 DeerFlow 里不是普通配置，而更像 runtime 声明文件。

它声明了：

- 模型列表
- 工具列表
- sandbox provider
- skills 路径
- memory 配置
- checkpointer
- summarization
- guardrails

也就是说，config 不只是“值”，还是“系统构件的来源”。

## 4. Reflection 是平台化关键

`resolve_variable()` 和 `resolve_class()` 允许 DeerFlow 通过字符串路径加载真实对象：

- class path -> 模型类 / provider 类
- variable path -> 工具变量 / provider 实例

这一步是 DeerFlow 可扩展性的基础。

没有 reflection，很多能力就只能硬编码在代码里，harness 很快会退化成一个项目内专用实现。

### 它还做了什么额外工作

- 类型校验
- 缺依赖时给出安装提示

这说明 reflection 不是裸 `importlib`，而是被包装成了平台友好的能力解析层。

## 5. Model Factory：把配置模型变成真实 LLM

`create_chat_model(...)` 做的事情比表面上多。

它不仅仅是：

- 读配置
- 解析类
- 初始化对象

还负责：

- 默认模型解析
- `thinking_enabled` 行为切换
- `reasoning_effort` 兼容
- vision 能力信息配合
- tracing callback 注入

这说明 DeerFlow 对“模型”也不是简单当成 SDK 对象，而是当成 runtime capability surface。

## 6. Tool Assembler：能力注册表的总入口

`get_available_tools(...)` 是理解工具系统的关键入口。

它会按顺序组合：

1. config 中声明的普通工具
2. built-in tools
3. subagent tool（按运行时开关）
4. `view_image`（按模型能力）
5. MCP tools
6. ACP tool

这里的思想非常重要：

工具不是一个固定列表，而是一个在 runtime 中动态计算出来的能力面。

## 7. 为什么这一层值得从“思想”上记住

### 7.1 配置驱动，替代硬编码

很多系统会在代码里直接 import provider/tool/model。DeerFlow 刻意减少这种耦合。

### 7.2 运行时装配，替代编译期绑定

tool、model、extensions 都可以随着配置、请求参数、外部文件变化而变化。

### 7.3 能力面与治理面分离

模型和工具属于能力；
middleware 属于治理。

这两者被清晰拆开，是 DeerFlow 可维护性的关键之一。

## 8. 外部集成时这层意味着什么

宿主应用接 DeerFlow 时，最自然的扩展点就是这层：

- 你可以声明自己的模型
- 你可以声明自己的工具
- 你可以保留自己的配置体系，再映射到 DeerFlow config
- 你可以只复用 reflection + assembler，而保留上层协议不变

## 9. 学完这一篇你应该能回答

1. 为什么 config 在 DeerFlow 里像“runtime 构件声明”
2. reflection 为什么是平台化关键设施
3. `get_available_tools()` 为什么比“工具仓库”更重要
4. model factory 为什么不仅仅是实例化模型

## 最短总结

DeerFlow 通过 config + reflection + assembler，把模型与工具从硬编码依赖变成了运行时可装配能力面，这正是 harness 可复用性的核心来源之一。
