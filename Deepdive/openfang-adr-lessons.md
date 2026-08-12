# Architecture Decision Records & Lessons

> 回答 goal §76-77（要求 ≥20 ADR、≥30 Lessons）
> ADR 是**逆向重建**的——从源码推断当初的决策，非官方文档

---

## Part 1: 20 个 ADR

### ADR-001：Runtime 定义 trait，Kernel 实现（依赖倒置）

- **Problem**：runtime 需调内核（spawn/send/list），但 kernel 依赖 runtime，会成循环依赖
- **Decision**：`KernelHandle` trait 定义在 `openfang-runtime/src/kernel_handle.rs`，
  `impl KernelHandle for OpenFangKernel` 在 kernel 侧
- **Reason**：Rust 拒绝循环 crate 依赖，必须打破
- **Trade-off**：runtime 持有 `Arc<dyn KernelHandle>` 有动态分派开销（可忽略）；
  换来编译期强制的分层
- **Evidence**：`openfang-runtime/Cargo.toml` 无 openfang-kernel；kernel.rs:7166 `impl`
- **Bianbu Impact**：✅ 采纳。服务边界应由构建系统强制，不靠文档

---

### ADR-002：Agent 是 Tokio task，不是进程

- **Problem**：Agent 需并发执行，隔离 vs 开销权衡
- **Decision**：`DashMap<AgentId, AbortHandle>`，所有 Agent 同进程
- **Reason**：进程/容器每 Agent 启动开销大；单二进制产品主张
- **Trade-off**：**放弃了地址空间隔离**——一崩全崩，无法用 cgroup/seccomp
- **Evidence**：`kernel.rs` `running_tasks: DashMap<AgentId, AbortHandle>`
- **Bianbu Impact**：❌ 不采纳。这是 OpenFang 最大的架构限制（L-05/L-21）

---

### ADR-003：per-agent 消息互斥锁

- **Problem**：同 Agent 并发消息会损坏 session（工具调用配对错乱）
- **Decision**：`agent_msg_locks: DashMap<AgentId, Arc<tokio::sync::Mutex<()>>>`
- **Reason**：代码注释明说是 Telegram 快速语音消息导致的真实故障
- **Trade-off**：同 Agent 串行（吞吐降低），不同 Agent 仍并行
- **Evidence**：kernel.rs 字段注释
- **Bianbu Impact**：✅ 采纳。session 有状态，必须串行化写入

---

### ADR-004：SQLite + WAL 作为唯一存储

- **Problem**：需持久化但要保持单二进制
- **Decision**：`rusqlite` bundled + `PRAGMA journal_mode=WAL`
- **Reason**：无外部依赖，静态链接，跨平台
- **Trade-off**：单机；无水平扩展
- **Evidence**：`substrate.rs::open()`
- **Bianbu Impact**：✅ 采纳（边缘场景 SQLite 正确），但换连接池

---

### ADR-005：单一 `Arc<Mutex<Connection>>` 共享给 6 个 store

- **Problem**：多 store 需访问同一 DB
- **Decision**：`Arc::clone(&shared)` 传给每个 store
- **Reason**：实现简单，避免连接池复杂度
- **Trade-off**：**抵消了 WAL 的并发读优势**，成为扩展瓶颈
- **Evidence**：`substrate.rs` 构造函数
- **Bianbu Impact**：❌ 不采纳。用 r2d2 连接池或按 Agent 分库（L-02）

---

### ADR-006：session 消息用 MessagePack 而非 JSON

- **Problem**：session 是最大的表，JSON 体积和解析开销高
- **Decision**：`rmp_serde` 序列化为 BLOB
- **Reason**：体积小 40-60%，解析快
- **Trade-off**：不可用 sqlite3 CLI 直接读/修；调试困难
- **Evidence**：`session.rs::get_session()` `rmp_serde::from_slice`
- **Bianbu Impact**：🟡 视情况。可读性对运维有价值，可考虑 JSON + 压缩

---

### ADR-007：write-through 缓存（内存 DashMap + SQLite）

- **Problem**：registry 需要快速查询又要持久化
- **Decision**：`AgentRegistry` 纯内存 DashMap，`memory.save_agent()` 写穿
- **Reason**：查询走内存（零 IO），持久化异步
- **Trade-off**：**6/8 处调用用 `let _ =` 丢弃错误**，写穿失败静默分叉
- **Evidence**：kernel.rs:3398/3434/3473，routes.rs:5803/6108/9674
- **Bianbu Impact**：🟡 模式可用，但**必须处理错误**（L-17）

---

### ADR-008：WASM 双计量（fuel + epoch）

- **Problem**：WASM 需同时限 CPU 和墙钟；fuel 不能防 IO 等待
- **Decision**：`consume_fuel(true)` + `epoch_interruption(true)` + watchdog 线程
- **Reason**：fuel 确定性（可复现计费），epoch 兜底真实超时
- **Trade-off**：watchdog 每次执行起一个 OS 线程
- **Evidence**：`sandbox.rs:111-190`
- **Bianbu Impact**：✅ 采纳思路。fuel 的确定性是 cgroup 无法提供的

---

### ADR-009：Capability 在 manifest 声明为结构体而非 Vec

- **Problem**：TOML 中写 `Vec<Capability>` 的 tagged enum 很丑
- **Decision**：`AgentCapabilities { tools, network, shell, memory_read, ... }`
  结构体，`manifest_to_capabilities()` 展开为 `Vec<Capability>`
- **Reason**：TOML 可读性
- **Trade-off**：两套表示需同步；转换逻辑（kernel.rs:6757）复杂（含 profile 展开）
- **Evidence**：`agent.rs:585` `pub tools: Vec<String>`
- **Bianbu Impact**：✅ 采纳。声明式配置的可读性重要

---

### ADR-010：工具控制用"预过滤列表"而非运行时检查

- **Problem**：如何限制 Agent 可用工具
- **Decision**：`available_tools_with_registry()` 只把声明的工具发给 LLM
- **Reason**：实现简单；假设 LLM 不调用没见过的工具
- **Trade-off**：**不是强制**——LLM 幻觉工具名、MCP 动态工具都绕过；
  且 `tools` 为空时 fail-open
- **Evidence**：kernel.rs:6148-6202，代码注释 "This is the primary mechanism"
- **Bianbu Impact**：❌ 不采纳。必须是执行路径上的强制（L-06/L-08）

---

### ADR-011：Merkle 哈希链审计

- **Problem**：审计日志可被有 DB 权限者篡改
- **Decision**：每条含前一条 hash，`verify_integrity()` 全链重算
- **Reason**：防篡改可检测，实现仅 422 行
- **Trade-off**：只能检测不能阻止；且**失败时只 error 不阻止启动**
- **Evidence**：`audit.rs::compute_entry_hash` / `verify_integrity`
- **Bianbu Impact**：✅ 采纳。但失败必须进降级模式（L-14）

---

### ADR-012：OFP 用 HMAC 而非 TLS

- **Problem**：节点间需认证
- **Decision**：HMAC-SHA256 共享密钥 + NonceTracker 防重放 + ConstantTimeEq
- **Reason**：实现简单，无证书管理负担
- **Trade-off**：**只认证不加密**——局域网可窃听全部内容
- **Evidence**：`wire/peer.rs::hmac_sign/hmac_verify`；用裸 TcpStream
- **Bianbu Impact**：🟡 防重放设计采纳，但必须加 TLS（L-10）

---

### ADR-013：OpenAI 兼容驱动的 base_url 可配置

- **Problem**：如何支持本地/自建 LLM
- **Decision**：`providers.openai_compat.base_url` 配置项
- **Reason**：一个配置项打开 Ollama/vLLM/LocalAI/llama.cpp 全生态
- **Trade-off**：无
- **Evidence**：`drivers/openai.rs`
- **Bianbu Impact**：✅ 必须采纳。对离线/边缘是决定性设计

---

### ADR-014：调度委托给 Tokio，自己只做准入

- **Problem**：需要决定何时执行 Agent
- **Decision**：cron/trigger 触发 → `tokio::spawn`；`AgentScheduler` 只 check_quota
- **Reason**：不重写调度器
- **Trade-off**：**放弃调度语义**——无优先级/公平性/抢占/deadline
- **Evidence**：`scheduler.rs` 只有三个 DashMap，无调度方法
- **Bianbu Impact**：❌ 不采纳。`scheduler` 需从零设计（L-23）

---

### ADR-015：Cron/Trigger 不持久化

- **Problem**：定时任务状态存哪
- **Decision**：纯内存（`CronScheduler` / `TriggerEngine`）
- **Reason**：（推断）实现简单
- **Trade-off**：**daemon 重启后 24/7 任务全丢**，与产品主张矛盾
- **Evidence**：13 张表中无 cron/trigger 表
- **Bianbu Impact**：❌ 不采纳。调度作业必须持久化（L-16）

---

### ADR-016：安全模块只写单元测试，不写集成测试

- **Problem**：如何验证安全功能
- **Decision**：为 `tool_policy`/`shell_bleed`/`CapabilityManager` 写单元测试
- **Reason**：（推断）单元测试快、易写
- **Trade-off**：**四个模块测试全绿但从未接线**——测试验证了函数逻辑，
  没验证"未授权操作确实被拒绝"
- **Evidence**：`tool_policy.rs` 11 个测试通过 + 零调用点
- **Bianbu Impact**：❌ 不采纳。安全功能必须 E2E 测试 + CI dead-code 检测

---

### ADR-017：boot 时扫描文件系统自动 spawn Agent

- **Problem**：用户手放的 agent 目录不出现在 `GET /api/agents`（issue #1140）
- **Decision**：boot 时扫 `~/.openfang/agents/*/agent.toml`，未注册的自动 spawn
- **Reason**：改善用户体验
- **Trade-off**：**两个真相来源（DB + FS）需对账**；且不验签 →
  文件写权限 = 全 Agent 权限
- **Evidence**：kernel.rs:1466-1530
- **Bianbu Impact**：🟡 便利性可保留，但**必须验签**（L-09）

---

### ADR-018：manifest 只算 hash 不验签

- **Problem**：manifest 完整性
- **Decision**：`hash_manifest()` 算 SHA-256 记 debug 日志
- **Reason**：（推断）Ed25519 依赖已引入但签名分发链未建
- **Trade-off**：hash 只能检测意外损坏，不能防恶意篡改（无 CA 链）
- **Evidence**：`manifest_signing.rs`；kernel.rs:7843 附近只 hash
- **Bianbu Impact**：❌ 不采纳。Bundle 必须真验签

---

### ADR-019：kernel 可嵌入（为 Desktop app）

- **Problem**：Desktop app 需要 kernel 但不想起子进程
- **Decision**：抽取 `build_router(kernel, addr)`，与 daemon 生命周期解耦
- **Reason**：避免 IPC 开销，简化 Desktop 打包
- **Trade-off**：证明 kernel 是库不是特权服务；无法做 OS 级隔离
- **Evidence**：`api/server.rs::build_router` 的 doc 注释
- **Bianbu Impact**：❌ 不采纳。`agentd` 应是独立系统服务

---

### ADR-020：无 feature flags，全量编译

- **Problem**：如何管理 40+ channel、10 driver、wasmtime 等可选组件
- **Decision**：全部无条件编译（只有 `http-memory` 一个 feature）
- **Reason**：（推断）简化构建矩阵，"单二进制包含一切"的产品主张
- **Trade-off**：32MB 无法裁剪；RISC-V 移植无法绕过风险依赖
- **Evidence**：`Cargo.toml` 无 `[features]` 段（除 http-memory）
- **Bianbu Impact**：❌ 不采纳。边缘场景必须可裁剪（L-04）

---

## Part 2: 30 条 Architecture Lessons

格式：Lesson / Evidence / Why Important / Bianbu Recommendation

---

**L1. trait 定义在被依赖侧可打破循环依赖**
- Evidence：`KernelHandle` 在 runtime，`impl` 在 kernel
- Why：Rust 编译器把分层从约定变成强制
- Bianbu：服务接口定义应放在被调用方，由构建系统强制

**L2. 有测试通过 ≠ 功能生效**
- Evidence：4 个安全模块（tool_policy 11 测试、shell_bleed、
  CapabilityManager::check、WASM runtime）全绿但零调用
- Why：这是本次调研最重要的方法论发现
- Bianbu：CI 加 dead-code 检测；安全功能强制 E2E 测试

**L3. 授权检查必须在执行路径上，不能旁路**
- Evidence：`CapabilityManager` 存在 kernel 里但没人调 `check()`
- Why：可选的检查等于没有检查
- Bianbu：capabilityd 签发的 token 设为 toold 接口的必填参数

**L4. 安全默认值必须是 deny**
- Evidence：`declared_tools.is_empty() → 全部 53 工具可用`
- Why：忘配置的人得到最高权限，与最小权限原则相反
- Bianbu：空配置 = 零权限

**L5. 单一全局锁会抵消底层并发能力**
- Evidence：WAL 允许并发读，但 `Arc<Mutex<Connection>>` 串行化了
- Why：架构层的错误无法被底层优化弥补
- Bianbu：memoryd 用连接池

**L6. God Object 会让 TCB 膨胀一个数量级**
- Evidence：kernel.rs 9415 行占 TCB 的 65%
- Why：14.4K 行的 TCB 无法形式化验证或第三方审计
- Bianbu：agentd 保持在 2000 行以内

**L7. 长期运行系统的调度状态必须持久化**
- Evidence：Cron/Trigger 纯内存，重启即丢
- Why：与"24/7 自主运行"的产品主张直接矛盾
- Bianbu：scheduler 作业定义入库

**L8. write-through 失败必须让调用方知道**
- Evidence：6/8 处 `let _ = save_agent()`
- Why：静默分叉比崩溃更难排查
- Bianbu：至少 warn，关键路径 propagate

**L9. 确定性资源计量对计费是必要的**
- Evidence：WASM fuel 与系统负载无关，cgroup CPU 配额受调度影响
- Why：可复现的计量才能公平计费
- Bianbu：sandboxd 保留 fuel 语义

**L10. 一个可配置的 base_url 能打开整个生态**
- Evidence：`providers.openai_compat.base_url` → Ollama/vLLM/LocalAI 全支持
- Why：最小改动换最大兼容性
- Bianbu：aihald 必须有

**L11. 认证不等于加密**
- Evidence：OFP 有 HMAC 防伪造，但明文传输可被窃听
- Why：两个正交的安全属性容易被混淆
- Bianbu：Agent IPC 需 mTLS 或 Unix socket

**L12. 污点必须随数据结构传播，不能在检查点构造**
- Evidence：`TaintedValue::new(cmd, 硬编码 labels, ...)`，
  `merge_taint`/`declassify` 零调用
- Why：不传播的污点等于模式匹配，防不住间接注入
- Bianbu：ToolResult 携带 taint_labels 字段

**L13. 输出侧扫描和输入侧同等重要**
- Evidence：无 LLM 输出密钥扫描
- Why：LLM 可能把密钥打印在回复里发给用户
- Bianbu：securityd 做双向扫描

**L14. 审计链断裂应触发降级，不是记日志继续**
- Evidence：`verify_integrity()` 失败只 `tracing::error!`
- Why：攻击痕迹被掩盖后系统照常运行
- Bianbu：进只读/拒绝新操作模式

**L15. 三级降级链比单一失败点更健壮**
- Evidence：`context_overflow.rs::RecoveryStage` 截断→压缩→新 session
- Why：LLM 上下文溢出是高频问题，硬失败体验极差
- Bianbu：contextd 直接采纳

**L16. 会话是有状态的，必须串行化写入**
- Evidence：`agent_msg_locks` per-agent 互斥（注释提到 Telegram 语音故障）
- Why：并发写会损坏工具调用配对
- Bianbu：memoryd 提供 per-session 写锁

**L17. 两个真相来源需要对账机制**
- Evidence：boot 时既 `load_all_agents()` 又扫 FS 自动 spawn
- Why：DB 与文件系统不一致时行为不确定
- Bianbu：单一真相来源，或明确的优先级+冲突解决

**L18. 便利性功能可能是权限提升路径**
- Evidence：boot 自动加载 `~/.openfang/agents/*/agent.toml` 且不验签
- Why：文件写权限被放大为全 Agent 权限
- Bianbu：便利性保留但加验签

**L19. hash 不是签名**
- Evidence：`manifest_signing.rs` 只 `hash_manifest()`
- Why：hash 防意外损坏，签名防恶意篡改，两回事
- Bianbu：Bundle 用 Ed25519 + CA 链

**L20. 权限只能收窄的继承是正确的**
- Evidence：`validate_capability_inheritance(parent, child)` 强制子 ⊆ 父
- Why：比 Unix fork（继承全部 uid 权限）更安全
- Bianbu：token 派生同样强制收窄

**L21. 同进程放弃了全部 OS 隔离机制**
- Evidence：所有 Agent 是 Tokio task
- Why：cgroup/seccomp/namespace/独立 uid 四层全部用不上
- Bianbu：这是最大的差异化机会

**L22. 无 RSS 配额在边缘设备是致命的**
- Evidence：`ResourceQuota` 只有 token/cost
- Why：OOM killer 杀整进程 → 所有 Agent 一起死
- Bianbu：cgroup v2 memory.max 必须用上

**L23. 无 feature flags 使裁剪和移植都不可能**
- Evidence：`Cargo.toml` 只有 http-memory 一个 feature
- Why：既无法减小体积，也无法绕过 RISC-V 风险依赖
- Bianbu：第一天就做 feature 拆分

**L24. 记忆需要衰减机制**
- Evidence：`ConsolidationEngine::new(shared, decay_rate)`
- Why：只增不减的记忆库长期检索质量下降
- Bianbu：memoryd 保留衰减 + 整合

**L25. 知识图谱写入需要来源可信度**
- Evidence：`add_entity` 无 source/confidence 列
- Why：被注入劫持的 Agent 可投毒，之后无法区分可信数据
- Bianbu：entity/relation 加 source + confidence

**L26. 自动抽取比依赖 LLM 主动调用更可靠**
- Evidence：知识图谱只能靠 LLM 记得调 `knowledge_add`
- Why：LLM 会忘，导致图谱有洞
- Bianbu：后台 consolidation 任务自动抽取

**L27. 策略放宽应逐项进行，不能一刀切**
- Evidence：`exec_policy=Full` 跳过全部污点检查（为了让 Hand 用 curl）
- Why："允许 curl" ≠ "允许任意注入模式"
- Bianbu：策略维度正交，独立开关

**L28. 静默完成检测能捕获部分注入**
- Evidence：`phantom_action_detected` 检测"声称已发送但未调工具"
- Why：思路正确（行为一致性校验），但条件过窄（仅 iteration==0）
- Bianbu：扩展为"声称的动作 vs 实际调用的工具"匹配

**L29. 三套包格式无法组合分发**
- Evidence：Agent/Skill/Hand 三套 TOML，`agent.toml` 无法声明依赖 Skill
- Why：用户无法一键分发"带技能的 Agent"
- Bianbu：单一 Bundle，Skill 内联

**L30. 进程回收语义不能省**
- Evidence：`registry.add_child()` 只 push 无 remove
- Why：无 `wait()`/SIGCHLD，父 Agent 不知子何时结束，children 无限增长
- Bianbu：agentd 提供子 Agent 生命周期通知

---

## 汇总

| 类别 | ADR | Lessons |
|------|-----|---------|
| ✅ 值得采纳 | 001, 003, 004(改), 008, 009, 011(改), 013 | L1, L9, L10, L15, L16, L20, L24 |
| 🟡 需修改 | 006, 007, 012, 017 | L5, L8, L12, L14, L27, L28 |
| ❌ 不要采纳 | 002, 005, 010, 014, 015, 016, 018, 019, 020 | L2, L3, L4, L6, L7, L11, L13, L17-19, L21-23, L25, L26, L29, L30 |

**❌ 占 9/20 ADR** —— 这不是说 OpenFang 差，而是说 Bianbu 的目标
（真隔离、边缘、可裁剪、多租户）与 OpenFang 的目标
（单二进制、开箱即用、桌面友好）本质不同，很多决策不可移植。
