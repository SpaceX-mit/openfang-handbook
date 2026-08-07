# OFP 点对点协议（openfang-wire）

## 概述

OpenFang Wire Protocol (OFP) 实现跨机器的 Agent 发现、认证和通信，基于 **TCP + JSON-RPC framing + HMAC-SHA256 认证**。

```
crates/openfang-wire/src/
├── lib.rs       ← 公开接口
├── peer.rs      ← PeerNode（TCP 服务端+客户端，1284行）
├── registry.rs  ← PeerRegistry（已知节点跟踪）
└── message.rs   ← WireMessage 协议消息
```

---

## 架构

```
本地 OpenFangKernel
  ├─ PeerNode（TCP 监听器）
  │    ├─ 接受来自其他 OpenFang 节点的连接
  │    └─ 连接到配置的已知节点
  │
  └─ PeerRegistry（节点 + 远程 Agent 注册表）
       ├─ PeerEntry（连接信息、状态）
       └─ RemoteAgent（远程 Agent 信息）
```

---

## 握手流程

```
Client → Server: HandshakeRequest {
    nonce,        // UUID v4，防重放
    peer_id,      // 本地节点 UUID
    agent_list,   // 本地 Agent 列表
    timestamp,    // Unix 时间戳（5分钟窗口校验）
    signature     // HMAC-SHA256(secret, nonce+peer_id+timestamp)
}

Server → Client: HandshakeResponse {
    accepted: true,
    peer_id,       // 服务端节点 UUID
    agent_list,    // 服务端 Agent 列表
}
```

**安全机制**：
- `NonceTracker`：5分钟时间窗口内的 nonce 去重，防重放攻击
- `hmac_verify()`：使用 `subtle::ConstantTimeEq` 常量时间比较，防时序攻击
- 双方共享同一个 `secret`（在 config.toml 配置）

---

## 消息协议

### WireMessage 枚举

```rust
pub enum WireMessage {
    // 握手
    HandshakeRequest { nonce, peer_id, agent_list, timestamp, signature },
    HandshakeResponse { accepted, peer_id, agent_list },

    // Agent 操作
    AgentMessage { target_agent_id, message, request_id },
    AgentMessageResponse { request_id, response, error },

    // 发现
    AgentListRequest { request_id },
    AgentListResponse { request_id, agents },

    // Ping/Pong
    Ping { id },
    Pong { id },

    // 关闭
    Goodbye,
}
```

### 传输格式

```
[4 bytes: message length (big-endian u32)] [N bytes: JSON]
```

每条消息前加4字节长度头，实现消息边界切分。

---

## PeerNode

```rust
pub struct PeerNode {
    config: PeerConfig,
    registry: Arc<PeerRegistry>,
    handle: Arc<dyn PeerHandle>,    // kernel 实现此 trait 响应远程请求
    nonce_tracker: NonceTracker,
}
```

### PeerConfig

```toml
[network]
enabled = true
listen_addr = "0.0.0.0:5678"
secret = "shared-secret-between-nodes"

[[network.peers]]
address = "192.168.1.100:5678"
name = "remote-node-1"
```

### PeerHandle trait

```rust
#[async_trait]
pub trait PeerHandle: Send + Sync {
    // 接收来自远程的 Agent 消息
    async fn handle_remote_message(
        &self,
        target_agent_id: &str,
        message: &str,
    ) -> Result<String, String>;

    // 返回本地 Agent 列表
    fn local_agents(&self) -> Vec<RemoteAgent>;
}
```

OpenFangKernel 实现此 trait，将远程消息路由到本地 Agent。

---

## PeerRegistry

```rust
pub struct PeerRegistry {
    peers: DashMap<String, PeerEntry>,        // peer_id → PeerEntry
    remote_agents: DashMap<String, RemoteAgent>, // agent_id → RemoteAgent
}

pub struct PeerEntry {
    pub peer_id: String,
    pub address: SocketAddr,
    pub state: PeerState,
    pub agent_count: usize,
    pub connected_at: Instant,
}

pub enum PeerState {
    Connecting,
    Connected,
    Disconnected,
    Failed,
}

pub struct RemoteAgent {
    pub id: String,
    pub name: String,
    pub peer_id: String,    // 所在节点
    pub description: String,
    pub tags: Vec<String>,
}
```

---

## API 端点

```
GET  /api/network/status     ← 本地节点状态（peer_id、地址、连接数）
GET  /api/peers               ← 已连接节点列表
POST /api/peers/connect       ← 主动连接新节点
POST /api/peers/{id}/disconnect ← 断开连接
GET  /api/peers/{id}/agents   ← 远程节点的 Agent 列表
```

---

## AppState 中的注意事项

```rust
// 注意：PeerRegistry 在 kernel 上是 OnceLock<PeerRegistry>
// 但在 AppState 上是 Option<Arc<PeerRegistry>>
// 因此在 server.rs 中需要包装：
peer_registry: kernel.peer_registry.get().map(|r| Arc::new(r.clone()))
```

这是 CLAUDE.md 中明确列出的"常见陷阱"之一。

---

## 典型使用场景

1. **多机部署**：多个 OpenFang 实例通过 OFP 互联，Agent 可以跨机发消息
2. **Agent 发现**：`agent_find` 工具自动在本地+远程节点搜索 Agent
3. **负载分布**：将任务分发给不同机器上的专用 Agent
4. **高可用**：Agent 故障时跨节点转移

---

## 与 A2A 的区别

| 特性 | OFP | A2A |
|------|-----|-----|
| 目标 | OpenFang 节点互联 | 任意 Agent 框架互操作 |
| 传输 | TCP（长连接） | HTTP（无状态） |
| 认证 | HMAC-SHA256 共享密钥 | A2A Agent Card URL 验证 |
| 发现 | 配置已知节点 | /.well-known/agent.json |
| 延迟 | 低（TCP 保持连接） | 高（每次请求建立连接） |
| 互操作性 | 仅 OpenFang | 任意 A2A 实现 |
