# 安全模型

## 分层安全架构

OpenFang 实现了4层安全防护：

```
1. 能力系统（Capability-based Security）— RBAC 最细粒度
2. Merkle 审计链（AuditLog）— 可验证的不可篡改日志
3. 污点追踪（Taint Tracking）— 数据流动安全
4. 工具执行审批（ApprovalManager）— 人工审核门控
```

---

## 1. 能力系统（capability.rs）

### 核心原则

Agent 只能执行被**显式授权**的操作。没有能力声明 = 操作被拒绝。

### Capability 完整枚举

```rust
pub enum Capability {
    // 文件系统
    FileRead(String),          // glob: "~/docs/**/*.txt"
    FileWrite(String),         // glob: "~/output/**"

    // 网络
    NetConnect(String),        // "api.openai.com:443" 或 "*.example.com"
    NetListen(u16),            // 端口号

    // 工具
    ToolInvoke(String),        // 工具名: "web_search"
    ToolAll,                   // ⚠️ 危险！允许所有工具

    // LLM
    LlmQuery(String),          // 模型 glob: "claude-*" 或 "gpt-4*"
    LlmMaxTokens(u64),         // token 预算上限

    // Agent 交互（多 Agent 场景）
    AgentSpawn,                // 可以创建新 Agent
    AgentMessage(String),      // 可以向 pattern 匹配的 Agent 发消息
    AgentKill(String),         // 可以终止 pattern 匹配的 Agent

    // 内存
    MemoryRead(String),        // scope 模式
    MemoryWrite(String),

    // Shell
    ShellExec(String),         // 命令 glob（高风险）
    EnvRead(String),           // 环境变量名 glob

    // OFP 网络
    OfpDiscover,               // 发现远程节点
    OfpConnect(String),        // 连接远程节点（地址 glob）
    OfpAdvertise,              // 广播本地服务

    // 经济（预留）
    EconSpend(f64),            // 最大花费 USD
    EconEarn,                  // 接收付款
    EconTransfer(String),      // 转账给 Agent
}
```

### Agent Manifest 配置

```toml
[agent]
name = "research-bot"
capabilities = [
    { type = "ToolInvoke", value = "web_search" },
    { type = "ToolInvoke", value = "file_write" },
    { type = "FileWrite", value = "~/output/**" },
    { type = "NetConnect", value = "*.googleapis.com:443" },
    { type = "LlmMaxTokens", value = 100000 },
]
```

### 能力检查流程

```rust
// CapabilityManager::check()
fn check(&self, agent_id: AgentId, required: &Capability) -> CapabilityCheck {
    // 从 AgentRegistry 获取 Agent 的已授权能力列表
    // 对每个已授权能力检查是否覆盖 required
    // 支持 glob 模式匹配（FileRead("~/**") 覆盖 FileRead("~/docs/x.txt")）
}
```

### 能力继承（spawn_agent_checked）

子 Agent 只能继承父 Agent 的能力子集，不能升级：

```rust
async fn spawn_agent_checked(
    manifest_toml: &str,
    parent_id: Option<&str>,
    parent_caps: &[Capability],  // 父 Agent 的能力
) -> Result<(String, String), String> {
    // 验证子 manifest 中的每个能力 ⊆ parent_caps
    // 违反则拒绝 spawn
}
```

---

## 2. Merkle 审计链（audit.rs）

### 原理

每条审计记录包含前一条记录的 SHA-256 哈希，形成不可篡改链：

```
Entry[0]: hash = SHA256(seq+ts+agent+action+detail+outcome+"0"×64)
Entry[1]: hash = SHA256(seq+ts+agent+action+detail+outcome+Entry[0].hash)
Entry[N]: hash = SHA256(seq+ts+agent+action+detail+outcome+Entry[N-1].hash)
```

任何记录被修改，其后所有记录的哈希都会失效。

### AuditEntry 结构

```rust
pub struct AuditEntry {
    pub seq: u64,           // 单调递增序号
    pub timestamp: String,  // ISO-8601
    pub agent_id: String,
    pub action: AuditAction,
    pub detail: String,     // 操作详情（工具名、文件路径等）
    pub outcome: String,    // "ok" / "denied" / 错误信息
    pub prev_hash: String,  // 前一条的 hash（genesis 用全零）
    pub hash: String,       // 本条的 hash
}
```

### 审计动作类型

```rust
pub enum AuditAction {
    ToolInvoke,         // 工具调用
    CapabilityCheck,    // 能力检查
    AgentSpawn,         // 创建 Agent
    AgentKill,          // 终止 Agent
    AgentMessage,       // Agent 消息
    MemoryAccess,       // 内存访问
    FileAccess,         // 文件访问
    NetworkAccess,      // 网络访问
    ShellExec,          // Shell 执行
    AuthAttempt,        // 认证尝试
    WireConnect,        // OFP 连接
    ConfigChange,       // 配置变更
}
```

### 持久化

SQLite `audit_entries` 表（schema V8）：
- Daemon 重启后日志保留
- 通过 `GET /api/audit` 查询
- 通过 `POST /api/audit` 追加（需认证）

---

## 3. 污点追踪（taint.rs，openfang-types）

### TaintLabel

```rust
pub enum TaintLabel {
    UserInput,        // 来自用户的不可信输入
    ExternalNetwork,  // 来自网络请求的数据
    FileSystem,       // 来自文件读取的数据
    Secret,           // 密钥/密码/token
    LlmOutput,        // LLM 输出（可能含幻觉）
}
```

### TaintSink 检查

在关键数据流动点检查污点：

**Shell 执行污点检查**（`tool_runner.rs::check_taint_shell_exec`）：
- Layer 1：检测 shell 元字符（`` ` ``、`$(`、`${`）
- Layer 2：检测注入 URL/base64 payload 的可疑模式（`curl`、`wget`、`| sh`、`eval`）

**网络请求污点检查**（`check_taint_net_fetch`）：
- 阻止含有 API key/token 的 URL（防数据外泄）
- 模式：`api_key=`、`token=`、`secret=`、`password=`

**`shell_bleed.rs`**：
- 检测 LLM 输出中隐藏的 shell 注入指令
- 防止 prompt injection → shell 执行链

---

## 4. 工具执行审批（ApprovalManager）

### 配置

```toml
[tool_policy]
require_approval = ["shell_exec", "file_delete", "agent_spawn"]
auto_approve_readonly = true   # 只读工具自动批准
```

### 审批流程

```
Agent 请求执行 tool_name
  │
  ├─ kernel.requires_approval(tool_name) → true
  │
  ├─ kernel.request_approval(agent_id, tool_name, action_summary)
  │    └─ 向 Dashboard 推送审批请求（WebSocket）
  │    └─ 等待用户点击批准/拒绝（超时自动拒绝）
  │
  ├─ 批准 → 执行工具
  └─ 拒绝/超时 → 返回 "Tool execution denied" 错误
```

---

## 5. CORS 安全

`server.rs` 中的 CORS 配置：

- **无认证**：限制为 localhost origins（127.0.0.1、localhost），拒绝跨域
- **有认证**：限制为 localhost + 配置的 origins
- **不使用** `CorsLayer::permissive()`（防跨站请求伪造）

---

## 6. Dashboard 认证

```toml
[auth]
enabled = true
password_hash = "$argon2id$v=19$m=65536,t=2,p=1$..."  # Argon2id
session_secret = "random-32-byte-hex-string"
```

- 密码哈希：**Argon2id**（唯一接受的格式）
- Session：cookie + secret（HMAC）
- 生成哈希：`openfang auth hash-password`

---

## 7. OFP 协议安全

- **HMAC-SHA256**：每条握手消息签名，防伪造
- **NonceTracker**：5分钟窗口内 nonce 去重，防重放攻击
- **ConstantTimeEq**：常量时间 HMAC 比较，防时序攻击
- **共享密钥**：OFP 节点间共享 `network.secret`

---

## 8. Manifest 签名

`manifest_signing.rs`：

```rust
pub fn hash_manifest(toml_content: &str) -> String {
    // SHA-256 of manifest content
}
```

在 `spawn_agent` 时计算 manifest 哈希并记录到 debug 日志（content integrity tracking）。

---

## 9. SSRF 防护（web_fetch.rs）

`web_fetch.rs` 实现 SSRF（Server-Side Request Forgery）防护：
- 阻止请求 `127.0.0.1`、`localhost`、`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`
- 阻止请求 `169.254.0.0/16`（AWS 元数据服务）
- 阻止请求 `::1`（IPv6 loopback）

---

## 快速安全配置参考

```toml
# ~/.openfang/config.toml

[api_key]
# API 认证密钥（非空则启用）
value = "your-secure-api-key"

[auth]
# Dashboard 认证
enabled = true
password_hash = "$argon2id$..."

[tool_policy]
# 需要人工审批的工具
require_approval = ["shell_exec"]

[network]
# OFP 节点认证密钥
secret = "very-long-random-string"

[sandbox]
# 是否启用 Docker 沙箱
use_docker = false
docker_image = "openfang-sandbox:latest"
```
