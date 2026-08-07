# A2A 协议（Agent-to-Agent）

## 概述

Google A2A 是一个开放的跨框架 Agent 互操作协议。OpenFang 实现了完整的 A2A 规范，既可作为 A2A 服务端（暴露本地 Agent），也可作为客户端（调用外部 A2A Agent）。

**文件**：`crates/openfang-runtime/src/a2a.rs`

---

## 核心类型

### AgentCard（/.well-known/agent.json）

```json
{
  "name": "research-bot",
  "description": "Autonomous research agent",
  "url": "http://myserver.com",
  "version": "0.6.9",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "stateTransitionHistory": true
  },
  "skills": [
    {
      "id": "web-research",
      "name": "Web Research",
      "description": "Search and summarize web content",
      "tags": ["research", "web"],
      "examples": ["Research the latest news about AI"]
    }
  ],
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["text/plain"]
}
```

**暴露路径**：`GET /.well-known/agent.json`（由 API 服务器自动提供）

---

### A2aTask（任务单元）

```rust
pub struct A2aTask {
    pub id: String,               // UUID
    pub session_id: Option<String>,
    pub status: A2aTaskStatus,
    pub messages: Vec<A2aMessage>,
    pub artifacts: Vec<A2aArtifact>,
}
```

### A2aTaskStatus

```rust
pub enum A2aTaskStatus {
    Submitted,    // 已提交，等待处理
    Working,      // 处理中
    InputRequired,// 需要更多输入
    Completed,    // 完成
    Cancelled,    // 已取消
    Failed,       // 失败
}
```

A2A 规范允许两种状态格式，OpenFang 用 `A2aTaskStatusWrapper` 同时支持：
- 字符串形式：`"completed"`
- 对象形式：`{"state": "completed", "message": null}`

---

### A2aMessage

```rust
pub struct A2aMessage {
    pub role: String,    // "user" or "agent"
    pub parts: Vec<A2aPart>,
}

pub enum A2aPart {
    Text { text: String },
    File { mime_type: String, data: Option<String>, url: Option<String> },
    Data { data: Value },
}
```

---

### A2aArtifact（任务产物）

```rust
pub struct A2aArtifact {
    pub name: Option<String>,
    pub description: Option<String>,
    pub parts: Vec<A2aPart>,
    pub index: u32,
    pub append: bool,
    pub last_chunk: bool,
}
```

---

## A2A 服务端（暴露 OpenFang Agent）

OpenFang 自动将配置的 Agent 暴露为 A2A 服务：

```
GET  /.well-known/agent.json          ← Agent Card
POST /a2a/tasks/send                  ← 提交新任务
POST /a2a/tasks/{id}                  ← 向任务追加消息
GET  /a2a/tasks/{id}                  ← 查询任务状态
POST /a2a/tasks/{id}/cancel           ← 取消任务
```

**任务映射**：A2A task → OpenFang Agent 消息调用，结果回填到 A2aTask。

---

## A2A 客户端（调用外部 Agent）

### 发现外部 Agent

```bash
POST /api/a2a/discover
{
  "url": "https://external-agent.example.com"
}
```

OpenFang 会自动获取 `/.well-known/agent.json`，验证格式，并存储到 `a2a_external_agents`。

### 发送任务

```bash
POST /api/a2a/send
{
  "agent_url": "https://external-agent.example.com",
  "message": "Research quantum computing trends",
  "session_id": "optional-for-continuity"
}
```

返回：
```json
{
  "task_id": "uuid",
  "status": "submitted"
}
```

### 轮询状态

```bash
GET /api/a2a/tasks/{task_id}/status
```

```json
{
  "task_id": "uuid",
  "status": "completed",
  "response": "Agent 回复内容"
}
```

---

## A2aTaskStore（任务状态管理）

```rust
pub struct A2aTaskStore {
    tasks: DashMap<String, A2aTask>,
}
```

内存中存储，不持久化。Daemon 重启后任务状态丢失（符合 A2A 规范的无状态语义）。

---

## Agent 工具集成

在 Agent 执行时，通过 `KernelHandle` 可以直接使用 A2A：

```
// Agent 可以使用的工具
agent_find → 自动搜索已发现的 A2A 外部 Agent
agent_send → 向 A2A 外部 Agent 发送任务（通过 agent URL）
```

`kernel.list_a2a_agents()` 返回 `Vec<(name, url)>`，`kernel.get_a2a_agent_url(name)` 按名查 URL。
