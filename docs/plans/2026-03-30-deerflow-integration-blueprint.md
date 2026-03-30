# DeerFlow Integration Blueprint Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 将现有 Python 后端中的简版 ReAct Agent 平滑迁移到 DeerFlow runtime，同时保留现有前端、业务服务、MCP 和消息存储体系。

**Architecture:** 采用“宿主应用 + DeerFlow runtime kernel”模式。前端和业务 API 不变，后端增加一层 `AgentFacade` 负责把现有请求协议、会话 ID、MCP/tool 注册和记忆系统适配到 `create_deerflow_agent()` 之上。短期内消息历史与 DeerFlow checkpointer 并存；中期把 memory 切到自定义 `MemoryStorage` 以接入 `mem0`。

**Tech Stack:** Python 3.12+, FastAPI/Flask/Django 任一宿主框架, DeerFlow Harness (`deerflow-harness`), LangGraph checkpointer, SQLite/Postgres, MCP, Mem0

---

### Task 1: 建立宿主应用接入骨架

**Files:**
- Create: `<your-backend>/app/agent/facade.py`
- Create: `<your-backend>/app/agent/runtime.py`
- Modify: `<your-backend>/app/api/chat.py`
- Reference: `backend/packages/harness/deerflow/agents/factory.py`

**Step 1: 定义 AgentFacade 接口**

```python
class AgentFacade:
    def stream(self, *, conversation_id: str, user_id: str, message: str):
        raise NotImplementedError
```

**Step 2: 在 runtime.py 中集中初始化 DeerFlow agent**

```python
agent = create_deerflow_agent(
    model=create_chat_model(name="your-model", thinking_enabled=True),
    tools=[],
    features=RuntimeFeatures(sandbox=False, memory=False, auto_title=False),
    checkpointer=your_checkpointer,
)
```

**Step 3: 在 facade.py 中把现有 conversation_id 映射为 DeerFlow thread_id**

```python
thread_id = conversation_id
```

**Step 4: 在 chat API 中调用 Facade，保持前端协议不变**

```python
return StreamingResponse(facade.stream(...), media_type="text/event-stream")
```

**Step 5: 运行现有聊天接口回归测试**

Run: `pytest <your-backend>/tests/api/test_chat.py -v`

### Task 2: 建立 Prompt 维护分层

**Files:**
- Create: `<your-backend>/app/agent/prompt_builder.py`
- Create: `<your-backend>/prompts/base_system.md`
- Create: `<your-backend>/prompts/domain_rules.md`
- Optional: `<your-backend>/agents/<agent-name>/SOUL.md`
- Reference: `backend/packages/harness/deerflow/agents/lead_agent/prompt.py`

**Step 1: 将 prompt 拆成三层**

```text
平台层: runtime/tool/sandbox 规则
领域层: 业务规则、术语、输出格式
身份层: persona / SOUL / 品牌语气
```

**Step 2: 在 prompt_builder.py 中统一拼装 system prompt**

```python
def build_system_prompt(*, agent_name: str | None, enable_skills: bool) -> str:
    return base_prompt + "\n\n" + domain_prompt
```

**Step 3: 不把用户态动态数据写死进固定 prompt 文件**

```python
session_context = f"<session_context>tenant={tenant_id}</session_context>"
```

**Step 4: 仅在需要 DeerFlow 默认技能段时，调用其 skills prompt helper**

```python
skills_section = get_skills_prompt_section() if enable_skills else ""
```

**Step 5: 回归验证关键 prompt 输出**

Run: `pytest <your-backend>/tests/agent/test_prompt_builder.py -v`

### Task 3: 打通会话与持久化

**Files:**
- Modify: `<your-backend>/app/agent/facade.py`
- Modify: `<your-backend>/app/db/models.py`
- Modify: `<your-backend>/app/services/conversation_service.py`
- Reference: `backend/packages/harness/deerflow/agents/checkpointer/provider.py`

**Step 1: 明确两个存储面**

```text
SQLite 消息表: 用户可见历史、审计、搜索
DeerFlow checkpointer: agent runtime state
```

**Step 2: conversation_id 与 thread_id 使用 1:1 映射**

```python
thread_id = conversation.id
```

**Step 3: 把 DeerFlow 调用统一带上 thread_id**

```python
result = agent.invoke(
    {"messages": [("user", message)]},
    config={"configurable": {"thread_id": thread_id}},
)
```

**Step 4: 将 agent 输出同步写回现有消息表**

```python
message_repo.append_assistant_message(conversation_id, content)
```

**Step 5: 若是多实例部署，优先切到 Postgres checkpointer**

Run: `pytest <your-backend>/tests/agent/test_session_mapping.py -v`

### Task 4: 接入现有 MCP 体系

**Files:**
- Create: `<your-backend>/config/extensions_config.json`
- Modify: `<your-backend>/app/agent/runtime.py`
- Reference: `backend/packages/harness/deerflow/mcp/tools.py`
- Reference: `backend/packages/harness/deerflow/config/extensions_config.py`

**Step 1: 用标准 MCP server 注册现有 MCP 能力**

```json
{
  "mcpServers": {
    "your_mcp": {
      "enabled": true,
      "type": "stdio",
      "command": "python",
      "args": ["-m", "your_mcp_server"]
    }
  }
}
```

**Step 2: 在宿主应用启动时初始化 MCP tools**

```python
tools = get_available_tools(include_mcp=True, subagent_enabled=False)
```

**Step 3: 若已有自己的 MCP registry，可在 DeerFlow 外层先管开关，再交给 DeerFlow 组装**

**Step 4: 为 HTTP/SSE MCP 单独设计 token 注入与环境变量策略**

**Step 5: 回归验证 MCP tool schema 与调用结果**

Run: `pytest <your-backend>/tests/agent/test_mcp_integration.py -v`

### Task 5: 建立 Skills 体系并与 MCP 协同

**Files:**
- Create: `<your-backend>/skills/public/<skill-name>/SKILL.md`
- Modify: `<your-backend>/config/extensions_config.json`
- Modify: `<your-backend>/app/agent/prompt_builder.py`
- Reference: `backend/packages/harness/deerflow/skills/loader.py`
- Reference: `backend/packages/harness/deerflow/agents/lead_agent/prompt.py`

**Step 1: 把 skills 当“工作流知识包”，不要当可执行插件**

**Step 2: 为每个关键业务域写一个 SKILL.md**

```markdown
---
name: order-analysis
description: 订单排障与履约分析
---
```

**Step 3: 在 skill 中显式说明何时调用哪些 MCP tools**

```markdown
当需要查询 ERP 状态时，优先使用 `erp__query_order`
```

**Step 4: 通过 extensions_config.json 开关 skill**

```json
{
  "skills": {
    "order-analysis": { "enabled": true }
  }
}
```

**Step 5: 回归验证 skill 与 MCP 协同**

Run: `pytest <your-backend>/tests/agent/test_skills_prompt.py -v`

### Task 6: 设计 Memory 双层架构

**Files:**
- Create: `<your-backend>/app/agent/memory/mem0_storage.py`
- Modify: `<your-backend>/config.yaml`
- Modify: `<your-backend>/app/agent/runtime.py`
- Reference: `backend/packages/harness/deerflow/agents/memory/storage.py`
- Reference: `backend/packages/harness/deerflow/config/memory_config.py`

**Step 1: 保留现有 SQLite 消息表，不把它直接当 DeerFlow memory**

**Step 2: 让 DeerFlow memory 只负责“长期语义记忆”**

```text
消息表: 原始对话
Mem0: 用户事实、偏好、长期上下文
```

**Step 3: 实现自定义 MemoryStorage，适配 mem0**

```python
class Mem0MemoryStorage(MemoryStorage):
    def load(self, agent_name=None): ...
    def reload(self, agent_name=None): ...
    def save(self, memory_data, agent_name=None): ...
```

**Step 4: 在 config.yaml 中切换 storage_class**

```yaml
memory:
  enabled: true
  storage_class: your_app.agent.memory.mem0_storage.Mem0MemoryStorage
```

**Step 5: 决定是否保留 DeerFlow 的 LLM memory updater**

```text
方案 A: 继续用 DeerFlow 的 updater，只替换存储层
方案 B: 关闭 DeerFlow memory，完全由 mem0 管理抽取与检索
```

### Task 7: 做最小可上线收口

**Files:**
- Modify: `<your-backend>/app/agent/facade.py`
- Modify: `<your-backend>/app/observability/agent_metrics.py`
- Modify: `<your-backend>/README.md`

**Step 1: 埋点 thread_id / model / tool_calls / latency / MCP failures**

**Step 2: 增加降级开关，允许回退到旧 ReAct agent**

**Step 3: 在灰度环境用真实业务对话回放**

**Step 4: 校验 prompt、session、MCP、skills、memory 五条链路**

**Step 5: 输出上线清单与回滚方案**

