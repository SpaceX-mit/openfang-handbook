# REST API 完整参考

## 服务器配置

| 项 | 值 |
|-------|-------|
| 默认监听地址 | `0.0.0.0:4200` |
| 框架 | Axum 0.8 |
| 压缩 | gzip + brotli (tower-http) |
| CORS | localhost origins（无认证时）+ 配置 origins（有认证时） |
| WebSocket | 支持（`/ws`） |

---

## Agent 管理

### `POST /api/agents` — 创建 Agent

**请求体**：
```json
{
  "manifest": "<TOML manifest 字符串>"
}
```

**返回**：
```json
{
  "id": "uuid",
  "name": "agent-name"
}
```

### `GET /api/agents` — 列出所有 Agent

**返回**：Agent 列表（含 id、name、state、model、description、tags、tools）

### `GET /api/agents/{id}` — 获取单个 Agent 详情

**返回**：完整 Agent 信息（含 manifest、session、最近消息）

### `POST /api/agents/{id}/message` — 发送消息给 Agent

**请求体**：
```json
{
  "message": "Hello",
  "attachments": [{"path": "file.pdf"}],  // 可选
  "stream": false                          // 可选，默认 false
}
```

**返回**：
```json
{
  "response": "Agent 回复",
  "usage": {"input_tokens": 100, "output_tokens": 50, ...}
}
```

### `POST /api/agents/{id}/message/stream` — 流式消息（SSE）

SSE 流，每个事件格式：
```
data: {"type": "text_delta", "text": "..."}
data: {"type": "tool_use_start", "id": "...", "name": "..."}
data: {"type": "content_complete", "stop_reason": "end_turn", "usage": {...}}
```

### `DELETE /api/agents/{id}` — 删除 Agent

终止并卸载 Agent。

### `POST /api/agents/{id}/kill` — 强制终止 Agent

立即停止 Agent 执行。

### `POST /api/agents/{id}/restart` — 重启 Agent

终止后重新启动。

### `PUT /api/agents/{id}/mode` — 设置 Agent 模式

**请求体**：
```json
{
  "mode": "autonomous" | "interactive"
}
```

---

## 会话管理

### `GET /api/agents/{id}/session` — 获取当前会话

返回消息历史 + token 统计。

### `DELETE /api/agents/{id}/session` — 清空会话

清除当前会话的所有消息（开启新对话）。

### `GET /api/agents/{id}/sessions` — 列出所有历史会话

### `GET /api/agents/{id}/sessions/{session_id}` — 获取指定会话

---

## KV 存储

### `GET /api/agents/{id}/kv` — 列出所有键

### `GET /api/agents/{id}/kv/{key}` — 读取键值

### `PUT /api/agents/{id}/kv/{key}` — 写入键值

**请求体**：
```json
{
  "value": "any JSON value"
}
```

### `DELETE /api/agents/{id}/kv/{key}` — 删除键

---

## 预算/计量

### `GET /api/budget` — 全局预算状态

```json
{
  "total_usd": 123.45,
  "limit_usd": 500.0,
  "agents": 15,
  "last_reset": "2026-08-07T00:00:00Z"
}
```

### `PUT /api/budget` — 更新预算配置

**请求体**：
```json
{
  "global_limit_usd": 1000.0,
  "per_agent_limit_usd": 50.0
}
```

### `GET /api/budget/agents` — 各 Agent 花费排行

返回按花费降序的 Agent 列表。

### `GET /api/budget/agents/{id}` — 单个 Agent 预算详情

---

## 工作流

### `POST /api/workflows` — 创建工作流

**请求体**：
```json
{
  "name": "multi-step-task",
  "description": "...",
  "steps": [...]
}
```

### `GET /api/workflows` — 列出所有工作流

### `GET /api/workflows/{id}` — 获取工作流详情

### `PUT /api/workflows/{id}` — 更新工作流

### `DELETE /api/workflows/{id}` — 删除工作流

### `POST /api/workflows/{id}/run` — 执行工作流

**请求体**：
```json
{
  "input": {...}
}
```

### `GET /api/workflows/{id}/runs` — 获取执行历史

---

## 触发器

### `POST /api/triggers` — 创建事件触发器

**请求体**：
```json
{
  "agent_id": "uuid",
  "pattern": {
    "event_type": "MessageReceived",
    "payload_filter": {...}
  },
  "action": {...}
}
```

### `GET /api/triggers` — 列出所有触发器

### `DELETE /api/triggers/{id}` — 删除触发器

---

## 技能（Skills）

### `GET /api/skills` — 列出已安装技能

### `POST /api/skills/install` — 安装技能

**请求体**：
```json
{
  "source": "clawhub",
  "identifier": "skill-slug",
  "version": "latest"
}
```

### `DELETE /api/skills/{id}` — 卸载技能

### `POST /api/skills/reload` — 重新加载技能注册表

---

## ClawHub 市场

### `GET /api/clawhub/search?q=keyword` — 搜索技能

### `GET /api/clawhub/browse` — 浏览热门技能

### `GET /api/clawhub/skill/{slug}` — 获取技能详情

### `GET /api/clawhub/skill/{slug}/code` — 获取技能源码

### `POST /api/clawhub/install/{slug}` — 从 ClawHub 安装

---

## Hands（自主能力包）

### `GET /api/hands` — 列出所有可用 Hands

### `GET /api/hands/active` — 列出激活的 Hand 实例

### `GET /api/hands/{id}` — 获取 Hand 详情

### `POST /api/hands/{id}/check-deps` — 检查依赖项

### `POST /api/hands/{id}/install-deps` — 安装依赖

### `POST /api/hands/{id}/activate` — 激活 Hand

**请求体**：
```json
{
  "config": {
    "setting_key": "value"
  }
}
```

### `POST /api/hands/instances/{instance_id}/pause` — 暂停 Hand

### `POST /api/hands/instances/{instance_id}/resume` — 恢复 Hand

### `POST /api/hands/instances/{instance_id}/deactivate` — 停用 Hand

### `GET /api/hands/instances/{instance_id}/settings` — 获取 Hand 配置

### `PUT /api/hands/instances/{instance_id}/settings` — 更新 Hand 配置

### `GET /api/hands/stats` — Hands 统计信息

---

## 渠道（Channels）

### `GET /api/channels` — 列出所有渠道及配置状态

### `PUT /api/channels/{name}` — 配置渠道

**请求体**：根据渠道不同而不同，例如 Telegram：
```json
{
  "enabled": true,
  "bot_token": "123:ABC",
  "allowed_users": ["123456"],
  "agent_id": "uuid"
}
```

### `DELETE /api/channels/{name}` — 删除渠道配置

### `POST /api/channels/{name}/test` — 测试渠道连接

### `POST /api/channels/reload` — 重新加载所有渠道

---

## WhatsApp 网关

### `POST /api/whatsapp/qr/start` — 启动 QR 扫码流程

### `GET /api/whatsapp/qr/status` — 获取 QR 码状态

---

## 扩展/集成

### `GET /api/integrations` — 列出所有集成

### `GET /api/integrations/{name}` — 获取集成详情

### `POST /api/integrations/{name}/install` — 安装集成

### `DELETE /api/integrations/{name}` — 卸载集成

### `POST /api/integrations/{name}/configure` — 配置集成

### `GET /api/integrations/{name}/health` — 集成健康检查

---

## MCP 服务器

### `GET /api/mcp/servers` — 列出所有 MCP 服务器

### `POST /api/mcp/servers` — 添加 MCP 服务器

### `DELETE /api/mcp/servers/{id}` — 删除 MCP 服务器

### `GET /api/mcp/tools` — 列出所有 MCP 工具

---

## A2A 协议

### `GET /api/a2a/agents` — 列出已发现的外部 A2A Agent

### `POST /api/a2a/discover` — 发现外部 A2A Agent

**请求体**：
```json
{
  "url": "https://example.com"
}
```

### `POST /api/a2a/send` — 向外部 Agent 发送任务

**请求体**：
```json
{
  "agent_url": "https://example.com",
  "message": "Do something",
  "session_id": "optional-session-id"
}
```

### `GET /api/a2a/tasks/{id}/status` — 查询外部任务状态

---

## OFP 网络

### `GET /api/network/status` — OFP 网络状态

返回本地节点信息（peer_id、监听地址、连接数）。

### `GET /api/peers` — 列出已连接的 OFP 对等节点

### `POST /api/peers/connect` — 连接到远程 OFP 节点

**请求体**：
```json
{
  "address": "192.168.1.100:5678",
  "secret": "shared-secret"
}
```

---

## 审计日志

### `POST /api/audit` — 追加审计条目

**请求体**：
```json
{
  "agent_id": "uuid",
  "action": "ToolInvoke",
  "detail": "file_read",
  "outcome": "ok"
}
```

### `GET /api/audit?limit=100` — 查询审计日志

---

## 系统

### `GET /api/health` — 健康检查

返回 `{"status": "ok"}`

### `GET /api/health/detail` — 详细健康信息

包含各子系统状态、内存使用、运行时长等。

### `GET /api/metrics` — Prometheus 格式指标

### `GET /api/version` — 版本信息

```json
{
  "version": "0.6.9",
  "git_commit": "abc1234",
  "build_time": "..."
}
```

### `GET /api/status` — 系统状态

运行时长、Agent 数量、活跃会话数等。

### `POST /api/shutdown` — 优雅关闭

需要 API key（如果启用认证）。

---

## 认证

如果 `config.toml` 中 `api_key` 非空，所有 API 请求需携带：

```
Authorization: Bearer <api_key>
```

或：

```
X-API-Key: <api_key>
```

Dashboard 认证使用 Argon2id 哈希密码 + session cookie。

---

## WebSocket（/ws）

- 实时推送 Agent 消息、Cron 结果、系统事件
- 支持订阅特定 Agent 的事件流
- 自动重连机制（客户端）
