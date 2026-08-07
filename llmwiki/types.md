# 核心类型系统（openfang-types）

## 定位

`openfang-types` 是整个项目的"数据层"——只定义结构体、枚举和 trait，**不包含业务逻辑**。所有其他 crate 都依赖它。

```
crates/openfang-types/src/
├── agent.rs          ← Agent 身份、Manifest、状态（1350行）
├── config.rs         ← KernelConfig 及所有子配置（4701行）
├── message.rs        ← LLM 消息格式（ContentBlock、Role）
├── tool.rs           ← ToolDefinition、ToolCall、ToolResult
├── memory.rs         ← Memory trait + 知识图谱类型
├── capability.rs     ← Capability 枚举（RBAC）
├── event.rs          ← 系统事件类型
├── error.rs          ← OpenFangError 主错误枚举
├── scheduler.rs      ← 调度相关类型（1056行）
├── manifest_signing.rs← Manifest 完整性哈希
├── taint.rs          ← 污点标签追踪
├── approval.rs       ← 工具执行审批类型
├── comms.rs          ← 通信协议类型
├── commands.rs       ← 命令类型
├── media.rs          ← 媒体类型
├── model_catalog.rs  ← 模型目录类型
├── tool_compat.rs    ← 工具名规范化
├── serde_compat.rs   ← 序列化兼容助手
└── webhook.rs        ← Webhook 类型
```

---

## Agent 核心类型

### AgentId

```rust
pub struct AgentId(pub Uuid);
// UUID v4，在 SQLite 中存为字符串
```

### AgentManifest（TOML 配置）

```toml
[agent]
name = "research-bot"
description = "Autonomous research agent"
model = "claude-sonnet-4-20250514"
system_prompt = "You are a research assistant..."
tags = ["research", "autonomous"]
tools = ["web_search", "file_write"]

[agent.autonomous]
max_iterations = 30
max_restarts = 5
heartbeat_interval_secs = 60

[agent.routing]
simple_model = "claude-haiku-4-5-20251001"
complex_model = "claude-sonnet-4-20250514"
```

关键字段（`openfang-types/src/agent.rs`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | `String` | 唯一名称 |
| `model` | `Option<String>` | 模型（None 使用内核默认） |
| `system_prompt` | `String` | 系统提示词 |
| `tools` | `Vec<String>` | 允许的工具列表 |
| `capabilities` | `Vec<Capability>` | 显式授权能力 |
| `tags` | `Vec<String>` | 标签（用于发现） |
| `max_history_messages` | `usize` | 最大消息历史（默认 200） |
| `autonomous` | `Option<AutonomousConfig>` | 自主运行配置 |
| `routing` | `Option<ModelRoutingConfig>` | 模型路由配置 |
| `fallback_model` | `Option<FallbackModel>` | 备用模型 |
| `workspace_dir` | `Option<PathBuf>` | 工作目录 |
| `mcp_servers` | `Vec<McpServerRef>` | MCP 服务器引用 |

### AgentState 枚举

```rust
pub enum AgentState {
    Pending,     // 已创建，未启动
    Running,     // 正常运行
    Suspended,   // 已暂停
    Crashed,     // 崩溃
    Terminated,  // 已终止（不可恢复）
}
```

### AutonomousConfig

```rust
pub struct AutonomousConfig {
    pub quiet_hours: Option<String>,     // cron 表达式
    pub max_iterations: u32,             // 每次调用最大迭代
    pub max_restarts: u32,               // 最大重启次数
    pub heartbeat_interval_secs: u64,    // 心跳间隔
}
```

### ModelRoutingConfig

```rust
pub struct ModelRoutingConfig {
    pub simple_model: String,    // tokens < simple_threshold
    pub medium_model: String,    // 中间范围
    pub complex_model: String,   // tokens > complex_threshold
    pub simple_threshold: u32,   // 默认 100
    pub complex_threshold: u32,  // 默认 500
}
```

---

## 消息类型（message.rs）

### Role

```rust
pub enum Role {
    User,
    Assistant,
    System,    // 仅用于 prompt 构建，不存储
    Tool,      // 工具结果消息
}
```

### ContentBlock

```rust
pub enum ContentBlock {
    Text { text: String },
    Image { source: ImageSource, media_type: String },
    ToolUse { id: String, name: String, input: Value },
    ToolResult { tool_use_id: String, content: Vec<ContentBlock>, is_error: bool },
    Thinking { thinking: String },              // Claude extended thinking
    RedactedThinking { data: String },          // 不透明 thinking
    Document { ... },
    Video { ... },
}
```

### TokenUsage

```rust
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cache_read_tokens: u32,    // Anthropic prompt caching
    pub cache_write_tokens: u32,
}
```

---

## 工具类型（tool.rs）

### ToolDefinition

```rust
pub struct ToolDefinition {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,   // JSON Schema
}
```

### ToolCall

```rust
pub struct ToolCall {
    pub id: String,      // 工具调用 ID（由 LLM 生成）
    pub name: String,    // 工具名
    pub input: Value,    // JSON 输入
}
```

### ToolResult

```rust
pub struct ToolResult {
    pub tool_call_id: String,
    pub content: String,       // 字符串结果
    pub is_error: bool,
}
```

---

## 能力类型（capability.rs）

```rust
pub enum Capability {
    // 文件系统
    FileRead(String),        // glob 模式，如 "~/**/*.md"
    FileWrite(String),

    // 网络
    NetConnect(String),      // 如 "api.openai.com:443"
    NetListen(u16),

    // 工具
    ToolInvoke(String),      // 工具名
    ToolAll,                 // 危险！允许所有工具

    // LLM
    LlmQuery(String),        // 模型模式
    LlmMaxTokens(u64),

    // Agent 交互
    AgentSpawn,
    AgentMessage(String),    // Agent 名模式
    AgentKill(String),

    // 内存
    MemoryRead(String),      // 范围模式
    MemoryWrite(String),

    // Shell
    ShellExec(String),       // 命令模式
    EnvRead(String),

    // OFP 网络
    OfpDiscover,
    OfpConnect(String),
    OfpAdvertise,

    // 经济
    EconSpend(f64),          // 最大花费（USD）
    EconEarn,
    EconTransfer(String),
}
```

---

## 错误类型（error.rs）

```rust
pub enum OpenFangError {
    NotFound(String),
    AlreadyExists(String),
    InvalidInput(String),
    CapabilityDenied(String),
    Memory(String),
    Serialization(String),
    Io(String),
    Internal(String),
    LlmError(String),
    ToolError(String),
    AgentError(String),
    NetworkError(String),
    AuthError(String),
}
```

---

## 内存 trait（memory.rs）

```rust
pub trait Memory {
    fn store(&self, memory: MemoryEntry) -> OpenFangResult<()>;
    fn recall(&self, filter: MemoryFilter) -> OpenFangResult<Vec<MemoryEntry>>;
    fn forget(&self, id: &str) -> OpenFangResult<()>;
    fn knowledge_add_entity(&self, entity: Entity) -> OpenFangResult<String>;
    fn knowledge_add_relation(&self, relation: Relation) -> OpenFangResult<String>;
    fn knowledge_query(&self, pattern: GraphPattern) -> OpenFangResult<Vec<GraphMatch>>;
}
```

### 知识图谱类型

```rust
pub struct Entity {
    pub id: String,
    pub entity_type: EntityType,
    pub name: String,
    pub properties: HashMap<String, Value>,
}

pub struct Relation {
    pub id: String,
    pub source_id: String,
    pub target_id: String,
    pub relation: RelationType,
    pub properties: HashMap<String, Value>,
}
```

---

## 工具名规范化（tool_compat.rs）

`normalize_tool_name(name)` — 将 LLM 返回的工具名（可能含有前缀、空格、大小写变体）标准化。确保 `web_search`、`WebSearch`、`web-search` 都能匹配到同一个工具。

---

## 污点类型（taint.rs）

```rust
pub enum TaintLabel {
    UserInput,        // 来自用户输入
    ExternalNetwork,  // 来自网络请求
    FileSystem,       // 来自文件读取
    Secret,           // 含有密钥/密码
    LlmOutput,        // 来自 LLM 输出
}

pub struct TaintedValue {
    pub value: String,
    pub labels: HashSet<TaintLabel>,
    pub source: String,   // 来源描述
}
```

`TaintSink` 定义什么值不能流入什么目标（如 Secret 不能流入 NetFetch URL）。
