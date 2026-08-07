# 架构总览

## 项目定位

OpenFang 是一个 **Agent 操作系统（Agent OS）**，不是聊天框架。核心差异：
- 传统框架：等你输入 → 返回答案
- OpenFang：Agent 自主运行，按计划触发，与其他 Agent 协作，通过多种渠道汇报结果

整个系统编译为**单一二进制**（~32MB），启动即运行。

---

## 14 个 Crate 一览

```
openfang-types       ← 共享数据类型（纯数据，无业务逻辑）
openfang-memory      ← 内存底层：SQLite + 语义检索 + 知识图谱
openfang-runtime     ← Agent 执行环境：循环、LLM 驱动、工具、沙箱
openfang-wire        ← OFP 点对点网络协议（TCP + HMAC-SHA256）
openfang-kernel      ← 核心内核：组装所有子系统
openfang-api         ← HTTP REST API + WebSocket 服务器（Axum）
openfang-channels    ← 40+ 消息渠道适配器（Telegram、Slack 等）
openfang-skills      ← 插件系统：Python/WASM/Node/Shell 技能
openfang-hands       ← Hands：预制自主能力包
openfang-extensions  ← 一键集成（25 个 MCP 服务模板 + 凭证保管库）
openfang-migrate     ← 从 OpenClaw 迁移工具
openfang-desktop     ← Tauri 桌面应用封装
openfang-cli         ← 交互式命令行（用户正在开发，勿修改）
xtask                ← 构建辅助任务
```

---

## Crate 依赖图

```
                    openfang-types
                    /    |    \
          memory  runtime  wire
            |       |       |
            +---kernel------+
                   |
              api + channels + skills + hands + extensions
                   |
                  cli / desktop
```

**关键规则：**
- `openfang-runtime` **不依赖** `openfang-kernel`（防止循环依赖）
- `openfang-kernel` 依赖 `openfang-runtime`，并实现其中定义的 `KernelHandle` trait
- `openfang-types` 是叶节点，不依赖任何内部 crate

---

## 依赖倒置：KernelHandle

Runtime 需要调用内核操作（spawn agent、send message 等），但不能依赖 kernel crate。解决方案：

```
openfang-runtime/src/kernel_handle.rs  ← 定义 trait
openfang-kernel/src/kernel.rs          ← impl KernelHandle for OpenFangKernel
```

agent_loop 持有 `Option<Arc<dyn KernelHandle>>`，由 kernel 注入。

---

## 启动流程

```
openfang start
  │
  ├─ load_config() → ~/.openfang/config.toml
  ├─ OpenFangKernel::new()
  │     ├─ AgentRegistry (SQLite)
  │     ├─ MemorySubstrate (SQLite)
  │     ├─ MeteringEngine
  │     ├─ AuditLog (Merkle chain)
  │     ├─ EventBus
  │     ├─ WorkflowEngine
  │     ├─ CronScheduler
  │     ├─ HandRegistry
  │     ├─ ExtensionRegistry
  │     └─ WasmSandbox
  │
  ├─ kernel.start_background_agents()
  │     ├─ MCP 连接建立
  │     ├─ OFP PeerNode 启动
  │     └─ 各 Agent 任务启动
  │
  └─ build_router(kernel) → Axum HTTP 服务器
        ├─ channel_bridge::start_channel_bridge()
        └─ 监听 0.0.0.0:4200
```

---

## 数据流：一次 Agent 消息

```
外部消息 / API 调用
  │
  ▼
POST /api/agents/{id}/message
  │
  ▼
routes.rs::send_message()
  │
  ▼
kernel.send_message(agent_id, text)  ← 加 per-agent 互斥锁防并发
  │
  ▼
run_agent_loop(manifest, session, driver, kernel_handle, ...)
  │
  ├─ 1. memory_recall() → 语义相关记忆注入 system prompt
  ├─ 2. prompt_builder → 构建 CompletionRequest
  ├─ 3. driver.complete() → LLM API 调用
  ├─ 4. loop_guard → 防止无限循环
  ├─ 5. tool_runner::execute() → 如有 tool_call
  │       ├─ 内置工具 (file_read、shell_exec...)
  │       ├─ 技能工具 (Python/Node/Shell)
  │       └─ MCP 工具
  ├─ 6. metering → 计费追踪
  ├─ 7. audit_log → Merkle 链记录
  └─ 8. session_store.save() → 持久化会话
```

---

## AppState（API 层状态）

```rust
pub struct AppState {
    pub kernel: Arc<OpenFangKernel>,
    pub started_at: Instant,
    pub peer_registry: Option<Arc<PeerRegistry>>,
    pub bridge_manager: Mutex<BridgeManager>,
    pub channels_config: RwLock<ChannelsConfig>,
    pub shutdown_notify: Arc<Notify>,
    pub clawhub_cache: DashMap<...>,
    pub provider_probe_cache: ProbeCache,
    pub budget_config: Arc<RwLock<BudgetConfig>>,
}
```

API handlers 通过 `State(state): State<Arc<AppState>>` 获取状态，再通过 `state.kernel` 访问内核。

---

## 关键设计原则

1. **单一二进制**：所有子系统编译进同一个可执行文件，无需外部依赖（除 SQLite，已打包）
2. **依赖倒置**：Runtime 定义接口，Kernel 实现，避免循环依赖
3. **并发安全**：
   - `DashMap` 用于高频并发读写（Agent 注册表、交付追踪）
   - `RwLock` 用于读多写少（技能注册表、模型目录）
   - `Mutex` 用于串行化关键操作（MCP 连接、per-agent 消息锁）
   - `Arc<Notify>` 用于异步事件通知（优雅关闭）
4. **不可变 boot 配置**：KernelConfig 启动后不变，热更新通过专用 override 字段实现
5. **Merkle 审计链**：所有安全事件形成可验证的哈希链，防篡改
