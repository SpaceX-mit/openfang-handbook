# Kernel 深度解析

## 文件位置

```
crates/openfang-kernel/src/
├── kernel.rs        ← OpenFangKernel 主结构体 + KernelHandle 实现（9415行）
├── lib.rs           ← 模块导出
├── registry.rs      ← AgentRegistry（纯内存 DashMap×3，持久化在 MemorySubstrate 层）
├── scheduler.rs     ← AgentScheduler（触发自主 Agent 执行）
├── supervisor.rs    ← Supervisor（崩溃重启、心跳检测）
├── metering.rs      ← MeteringEngine（token/cost 计量）
├── audit.rs         ← （在 runtime crate）Merkle 链审计日志
├── event_bus.rs     ← EventBus（内部事件发布订阅）
├── workflow.rs      ← WorkflowEngine（多步骤编排）
├── cron.rs          ← CronScheduler（Agent 定时任务）
├── triggers.rs      ← TriggerEngine（事件驱动触发）
├── capabilities.rs  ← CapabilityManager（RBAC + 能力检查）
├── auth.rs          ← AuthManager（Dashboard RBAC）
├── approval.rs      ← ApprovalManager（工具执行审批）
├── auto_reply.rs    ← AutoReplyEngine（自动回复规则）
├── heartbeat.rs     ← 心跳检测
├── background.rs    ← BackgroundExecutor
├── config.rs        ← 配置加载
├── config_reload.rs ← 热重载
└── ...
```

---

## OpenFangKernel 结构体

Kernel 是整个系统的"主板"，约 60 个字段，每个都是一个子系统：

```rust
pub struct OpenFangKernel {
    // 配置
    pub config: KernelConfig,

    // 核心子系统
    pub registry: AgentRegistry,          // Agent 状态存储
    pub capabilities: CapabilityManager,  // 能力检查
    pub event_bus: EventBus,              // 内部事件总线
    pub scheduler: AgentScheduler,        // 触发调度
    pub memory: Arc<MemorySubstrate>,     // 统一内存底层
    pub supervisor: Supervisor,           // 进程监控
    pub workflows: WorkflowEngine,        // 工作流编排
    pub triggers: TriggerEngine,          // 事件触发
    pub background: BackgroundExecutor,   // 后台任务
    pub audit_log: Arc<AuditLog>,         // Merkle 审计链
    pub metering: Arc<MeteringEngine>,    // 成本计量

    // LLM 相关
    default_driver: Arc<dyn LlmDriver>,
    wasm_sandbox: WasmSandbox,
    pub model_catalog: RwLock<ModelCatalog>,

    // 技能/扩展
    pub skill_registry: RwLock<SkillRegistry>,
    pub skill_config_overrides: RwLock<Option<HashMap<...>>>,
    pub hand_registry: HandRegistry,
    pub extension_registry: RwLock<IntegrationRegistry>,
    pub extension_health: HealthMonitor,

    // 外部协议
    pub mcp_connections: Mutex<Vec<McpConnection>>,
    pub mcp_tools: Mutex<Vec<ToolDefinition>>,
    pub a2a_task_store: A2aTaskStore,
    pub a2a_external_agents: Mutex<Vec<(String, AgentCard)>>,

    // 媒体/工具
    pub web_ctx: WebToolsContext,
    pub browser_ctx: BrowserManager,
    pub media_engine: MediaEngine,
    pub tts_engine: TtsEngine,
    pub embedding_driver: Option<Arc<dyn EmbeddingDriver>>,

    // 安全
    pub auth: AuthManager,
    pub approval_manager: ApprovalManager,
    pub credential_resolver: Mutex<CredentialResolver>,
    pub pairing: PairingManager,

    // 网络
    pub peer_registry: OnceLock<PeerRegistry>,   // OFP
    pub peer_node: OnceLock<Arc<PeerNode>>,       // OFP

    // 渠道/消息
    pub channel_adapters: DashMap<String, Arc<dyn ChannelAdapter>>,
    pub delivery_tracker: DeliveryTracker,

    // 并发控制
    pub running_tasks: DashMap<AgentId, AbortHandle>,
    agent_msg_locks: DashMap<AgentId, Arc<Mutex<()>>>,

    // 其他
    pub cron_scheduler: CronScheduler,
    pub process_manager: Arc<ProcessManager>,
    pub hooks: HookRegistry,
    pub booted_at: Instant,
    self_handle: OnceLock<Weak<OpenFangKernel>>,  // 用于 trigger dispatch
}
```

### OnceLock 的使用

`peer_registry` 和 `peer_node` 用 `OnceLock` 是因为需要在 `Arc::new(kernel)` 之后才能初始化（它们需要 `Weak<OpenFangKernel>` 引用）。典型模式：

```rust
let kernel = Arc::new(OpenFangKernel::new(...));
kernel.peer_registry.get_or_init(|| PeerRegistry::new());
```

---

## Agent 生命周期

### 状态机

```
Pending ──spawn()──→ Running ──kill()──→ Terminated
                        │
                        ├──error──→ Crashed ──restart()──→ Running
                        │
                        └──pause()──→ Suspended ──resume()──→ Running
```

### AgentRegistry

- **本体是纯内存**：三个 `DashMap`（`agents` / `name_index` / `tag_index`），
  `registry.rs` 中**没有一行 SQLite**
- 持久化在上一层，write-through 到 `MemorySubstrate`：
  - 写：`kernel.memory.save_agent(&entry)` → `agents` 表（在 `memory.db`，不是独立的 `agents.db`）
  - 读：boot 时 `kernel.memory.load_all_agents()`（kernel.rs:1261）
  - ⚠️ 8 处 `save_agent` 调用中 6 处用 `let _ =` 丢弃错误，写穿失败会导致内存与 DB 静默分叉
- 每个 Agent 有：`AgentId`(UUID)、`AgentManifest`(TOML)、`AgentState`、`last_active`

> 深入分析见 [../Deepdive/openfang-kernel.md](../Deepdive/openfang-kernel.md) §7.3

### 心跳与崩溃检测

`heartbeat.rs` 中，Supervisor 定期检查各 Agent 的 `last_active` 时间戳。如果超过阈值（`heartbeat_interval_secs`）未更新，标记为 Crashed 并按 `max_restarts` 决定是否重启。

Agent loop 在长 LLM 调用之前会调用 `kernel.touch_agent(id)` 刷新时间戳，防止误判。

---

## KernelHandle 实现

Kernel 实现 `KernelHandle` trait（定义于 `openfang-runtime/src/kernel_handle.rs`），共 20+ 方法：

| 方法 | 功能 |
|------|------|
| `spawn_agent` | 从 TOML manifest 创建并启动新 Agent |
| `spawn_agent_checked` | spawn + 能力继承检查 |
| `send_to_agent` | 向其他 Agent 发消息（先 UUID 查找，失败则按名查） |
| `list_agents` | 返回所有运行中 Agent 列表 |
| `kill_agent` | 终止指定 Agent |
| `activate_agent` | 唤醒暂停/崩溃的 Agent |
| `memory_store` / `memory_recall` | 跨 Agent 共享内存 |
| `find_agents` | 按名字/标签/工具名查找 Agent |
| `task_post` / `task_claim` / `task_complete` | 任务队列 |
| `cron_create` / `cron_list` / `cron_cancel` | 定时任务管理 |
| `publish_event` | 发布自定义事件到 EventBus |
| `knowledge_add_entity` / `knowledge_add_relation` / `knowledge_query` | 知识图谱操作 |
| `requires_approval` / `request_approval` | 工具执行审批 |
| `send_channel_message` / `send_channel_media` | 向渠道发送消息 |
| `hand_list` / `hand_activate` / `hand_status` / `hand_deactivate` | Hands 管理 |
| `list_a2a_agents` / `get_a2a_agent_url` | A2A 外部 Agent 发现 |
| `touch_agent` | 刷新心跳时间戳 |

---

## 计量引擎（MeteringEngine）

- 路径：`crates/openfang-kernel/src/metering.rs`
- 追踪每个 Agent 的 token 消耗和 USD 成本
- 支持全局预算上限（`budget.global_limit_usd`）
- 支持 per-agent 预算
- API：`GET /api/budget`、`GET /api/budget/agents`、`GET /api/budget/agents/{id}`

---

## 事件总线（EventBus）

- 路径：`crates/openfang-kernel/src/event_bus.rs`
- 内部发布/订阅系统，用于 Agent 间事件通知
- 事件类型（定义于 `openfang-types/src/event.rs`）：
  - `AgentStarted`、`AgentStopped`、`AgentCrashed`
  - `MessageReceived`、`ToolInvoked`
  - `CronFired`、`TriggerFired`
  - 自定义事件（通过 `publish_event` 发布）

---

## 工作流引擎（WorkflowEngine）

- 路径：`crates/openfang-kernel/src/workflow.rs`（1385行）
- 支持多步骤、多 Agent 编排
- `WorkflowId`、`WorkflowRunId`、`StepAgent`
- API：`POST /api/workflows`、`POST /api/workflows/{id}/run`

---

## 触发引擎（TriggerEngine）

- 路径：`crates/openfang-kernel/src/triggers.rs`
- 基于事件触发 Agent 执行
- `TriggerId`、`TriggerPattern`（匹配事件类型和 payload 模式）
- API：`POST /api/triggers`、`GET /api/triggers`

---

## Cron 调度器

- 路径：`crates/openfang-kernel/src/cron.rs`（1345行）
- 使用 `cron` crate（`0.16`）解析表达式
- 每个 Agent 可注册多个 cron 作业
- 支持时区（chrono-tz）
- Agent 内通过 `cron_create` 工具创建，内核通过 `KernelHandle::cron_create` 委托
- Cron 结果通过 WebSocket 实时推送到 Dashboard

---

## StubDriver

当没有配置任何 LLM 提供商时，内核使用 `StubDriver` 代替真实驱动。它的 `complete()` 返回一个指导性错误，而不是 panic，确保 Dashboard 仍能启动。
