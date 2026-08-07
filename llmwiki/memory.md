# 内存子系统（openfang-memory）

## 架构概览

```
openfang-memory/src/
├── substrate.rs    ← MemorySubstrate：统一入口（组合三个存储）
├── session.rs      ← SessionStore：会话/消息历史（SQLite + MessagePack）
├── structured.rs   ← StructuredStore：键值对存储（SQLite）
├── knowledge.rs    ← KnowledgeStore：知识图谱（SQLite）
├── semantic.rs     ← SemanticStore：语义/向量检索
├── consolidation.rs← 记忆整合（将旧消息摘要压缩）
├── usage.rs        ← 使用量追踪
├── migration.rs    ← 数据库 schema 迁移
└── http_client.rs  ← 可选：HTTP 远程内存后端
```

---

## MemorySubstrate（统一入口）

```rust
pub struct MemorySubstrate {
    session_store: SessionStore,
    structured_store: StructuredStore,
    knowledge_store: KnowledgeStore,
    semantic_store: SemanticStore,
    conn: Arc<Mutex<Connection>>,   // 共享 SQLite 连接
}
```

所有内存操作都通过 `MemorySubstrate` 完成，它组合三个专用存储。

---

## 三个存储层

### 1. SessionStore（会话历史）

**用途**：持久化 Agent 的对话消息历史

**存储格式**：
- 消息列表序列化为 **MessagePack**（`rmp-serde`），存为 BLOB
- 相比 JSON 节省 40-60% 空间

**核心操作**：

```rust
session_store.get_session(session_id)    // 加载会话
session_store.save_session(&session)     // 保存/更新会话
session_store.list_sessions(agent_id)    // 列出 Agent 的所有会话
session_store.delete_session(session_id) // 删除会话
```

**Session 结构**：

```rust
pub struct Session {
    pub id: SessionId,
    pub agent_id: AgentId,
    pub messages: Vec<Message>,
    pub context_window_tokens: u64,
    pub label: Option<String>,
}
```

**SQLite schema**：
```sql
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    agent_id TEXT NOT NULL,
    messages BLOB NOT NULL,        -- MessagePack
    context_window_tokens INTEGER DEFAULT 0,
    label TEXT,
    created_at TEXT,
    updated_at TEXT
)
```

---

### 2. StructuredStore（键值对存储）

**用途**：
- Agent 自定义键值数据（通过 `memory_store` / `memory_recall` 工具）
- 跨 Agent 共享数据
- Agent 私有状态

**核心操作**：

```rust
structured_store.set(scope, key, value)  // 存储
structured_store.get(scope, key)         // 检索
structured_store.list(scope)             // 列出 scope 下所有键
structured_store.delete(scope, key)      // 删除
```

**Scope 设计**：
- `agent:{id}` — Agent 私有数据
- `global` — 跨 Agent 共享
- 自定义 scope

---

### 3. KnowledgeStore（知识图谱）

**用途**：结构化知识存储，支持图模式查询

**SQLite schema**：
```sql
CREATE TABLE entities (
    id TEXT PRIMARY KEY,
    entity_type TEXT NOT NULL,
    name TEXT NOT NULL,
    properties TEXT,          -- JSON
    created_at TEXT,
    updated_at TEXT
);

CREATE TABLE relations (
    id TEXT PRIMARY KEY,
    source_id TEXT REFERENCES entities(id),
    target_id TEXT REFERENCES entities(id),
    relation TEXT NOT NULL,   -- JSON (RelationType)
    properties TEXT,          -- JSON
    created_at TEXT
);
```

**核心操作**：

```rust
knowledge_store.add_entity(entity)       // 添加实体（UPSERT）
knowledge_store.add_relation(relation)   // 添加关系
knowledge_store.query_pattern(pattern)   // 图模式查询
```

**GraphPattern 查询示例**：

```rust
// 查找所有 Person 类型实体
GraphPattern {
    entity_type: Some(EntityType::Person),
    relation: None,
    target_type: None,
}

// 查找 A 指向 B 的关系
GraphPattern {
    entity_type: Some(EntityType::Company),
    relation: Some(RelationType::WorksAt),
    target_type: Some(EntityType::Person),
}
```

---

### 4. SemanticStore（语义检索）

**两阶段实现**：
- **Phase 1（当前）**：SQLite LIKE 匹配（基于关键词）
- **Phase 2（规划中）**：Qdrant 向量数据库（真正的语义相似搜索）

**用于 Agent loop 的记忆注入**：在每次 LLM 调用前，semantic_store 检索与当前消息相关的历史记忆，注入 system prompt。

**嵌入驱动**（`embedding.rs`）：
```rust
pub trait EmbeddingDriver: Send + Sync {
    async fn embed(&self, text: &str) -> Result<Vec<f32>, EmbeddingError>;
}
```
当 `embedding_driver` 存在时，使用向量相似度；否则回退到文本匹配。

---

## 记忆整合（consolidation.rs）

**触发条件**：当 Agent 会话历史超过 `max_history_messages` 时

**流程**：
1. 取最旧的 N 条消息
2. 用 LLM 生成摘要
3. 将摘要写入 StructuredStore
4. 从 Session 中删除原始消息

---

## 数据库路径

```
~/.openfang/
├── agents.db      ← Agent 注册表
├── memory.db      ← MemorySubstrate（sessions + structured + knowledge）
├── audit.db       ← 审计日志（schema V8）
└── ...
```

所有 SQLite 操作通过共享的 `Arc<Mutex<Connection>>`，确保单写者安全。

---

## 会话 API

```
GET  /api/agents/{id}/session          ← 当前会话消息历史
GET  /api/agents/{id}/sessions         ← 所有历史会话列表
GET  /api/agents/{id}/sessions/{sid}   ← 特定会话详情
DELETE /api/agents/{id}/session        ← 清空当前会话

GET  /api/agents/{id}/kv               ← Agent KV 存储
GET  /api/agents/{id}/kv/{key}         ← 读取指定键
PUT  /api/agents/{id}/kv/{key}         ← 写入指定键
DELETE /api/agents/{id}/kv/{key}       ← 删除指定键
```

---

## 重要说明

- SQLite `bundled` feature：无需系统安装 SQLite，静态链接进二进制
- 消息使用 MessagePack 而非 JSON：减小存储体积，提升序列化速度
- `truncate_str()` 在 `openfang-types/src/lib.rs` 中：安全截断 UTF-8 字符串（不会在多字节字符中间截断，修复了生产中 em dash 引发的 panic）
