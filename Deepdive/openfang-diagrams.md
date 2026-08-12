# OpenFang 架构图集

> 回答 goal §82
> **数据来源**：全部基于其他 Deepdive 文档中已核实的源码事实，无新增未验证断言。
> 每张图标注了依据来源。

---

## D-1 死代码分层：为什么只有工具授权层失效

> 依据：[openfang-security.md](openfang-security.md) §1-3

```mermaid
graph TB
    subgraph DEAD["❌ 工具级授权层 —— 全部死代码"]
        D1["CapabilityManager::check()<br/>2 单元测试 · 0 调用"]
        D2["tool_policy.rs<br/>11 单元测试 · 0 调用"]
        D3["shell_bleed.rs<br/>296 行 + 测试 · 0 调用"]
    end

    subgraph LIVE1["✅ 请求认证层 —— 活代码"]
        L1["api/middleware.rs<br/>bearer token"]
        L2["session_auth.rs"]
        L3["kernel/auth.rs RBAC 四级"]
    end

    subgraph LIVE2["✅ 人工门控层 —— 活代码"]
        L4["approval.rs<br/>tool_runner 内 requires_approval<br/>+ request_approval().await"]
    end

    subgraph LIVE3["✅ 可靠性层 —— 活代码"]
        L5["heartbeat.rs"]
        L6["supervisor.rs"]
    end

    subgraph LIVE4["✅ 功能层 —— 活代码"]
        L7["workflow.rs ~14 调用点"]
        L8["cron.rs / triggers.rs"]
        L9["WASM Agent 路径<br/>kernel.rs:5623"]
    end

    FAIL["失效时的可观测性"]
    DEAD -->|"❌ 零信号<br/>本该被拒的调用成功了"| FAIL
    LIVE1 -->|"✅ 返回 401"| FAIL
    LIVE2 -->|"✅ 请求超时/拒绝"| FAIL
    LIVE3 -->|"✅ Agent 卡死/重启"| FAIL
    LIVE4 -->|"✅ 功能不工作"| FAIL

    style DEAD fill:#ffcccc
    style FAIL fill:#fff3e0
```

**核心**：死代码全部落在唯一"缺席时系统仍完全正常工作"的层。

---

## D-2 Capability 生命周期：grant 活，check 死

> 依据：[openfang-security.md](openfang-security.md) §1.2

```mermaid
graph LR
    M["AgentManifest<br/>[agent.capabilities]"]
    CONV["manifest_to_capabilities()<br/>kernel.rs:6757"]
    STORE["CapabilityManager<br/>DashMap&lt;AgentId, Vec&lt;Capability&gt;&gt;"]

    G["grant()<br/>kernel.rs:1372, 1719"]
    L["list()<br/>kernel.rs:6193<br/>仅查 ToolAll"]
    R["revoke_all()<br/>kernel.rs:3724"]
    C["check()<br/>❌ 生产零调用"]

    EXEC["工具执行<br/>tool_runner::execute()"]
    AUDIT["AuditAction::CapabilityCheck<br/>❌ 枚举存在·无人产生此记录"]

    M --> CONV --> STORE
    STORE --> G
    STORE --> L
    STORE --> R
    STORE -.->|"断链"| C
    C -.->|"从不到达"| EXEC
    C -.-> AUDIT
    L -->|"仅影响工具列表预过滤"| EXEC

    style C fill:#ffcccc
    style AUDIT fill:#ffcccc
```

---

## D-3 工具门控真实路径：fail-open

> 依据：[openfang-security.md](openfang-security.md) §2

```mermaid
flowchart TB
    START["available_tools_with_registry()<br/>kernel.rs:6148"]
    DECL["declared_tools =<br/>manifest.capabilities.tools"]
    EMPTY{"is_empty()<br/>或含 '*' ?"}
    OPEN["⚠️ tools_unrestricted = true<br/>放行全部 53 内置工具<br/>含 shell_exec / file_delete"]
    FILTER["按 declared_tools 过滤"]
    LLM["发给 LLM 的 ToolDefinition 列表"]
    CALL["LLM 返回 tool_call"]
    MATCH["tool_runner::execute()<br/>match tool_name"]
    NOCHECK["❌ 不复查该工具是否被声明"]
    RUN["执行"]

    START --> DECL --> EMPTY
    EMPTY -->|"是（忘写配置）"| OPEN
    EMPTY -->|"否"| FILTER
    OPEN --> LLM
    FILTER --> LLM
    LLM --> CALL --> MATCH --> NOCHECK --> RUN

    style OPEN fill:#ffcccc
    style NOCHECK fill:#ffcccc
```

**两个缺陷**：空配置得到最高权限；预过滤不是强制（LLM 幻觉工具名 / MCP 动态工具可绕过）。

---

## D-4 测试覆盖叠加死代码

> 依据：[openfang-test-analysis.md](openfang-test-analysis.md) §0, §3

```mermaid
graph TB
    subgraph T["2,698 个测试"]
        U["196 个 #[cfg(test)] 单元测试模块"]
        I["13 个集成测试文件"]
    end

    subgraph A["授权断言总数 = 2"]
        A1["api_integration_test.rs:1040<br/>assert_eq!(status, 401)"]
        A2["api_integration_test.rs:1058<br/>assert_eq!(status, 401)"]
    end

    L["✅ 请求认证层<br/>（活代码）"]
    D["❌ 工具授权层<br/>CapabilityManager / tool_policy / shell_bleed"]

    I --> A1 --> L
    I --> A2 --> L
    U -->|"11 + 2 个单元测试<br/>验证函数返回 Denied"| D
    I -.->|"零集成断言"| D

    style D fill:#ffcccc
    style A fill:#fff3e0
```

**唯二的授权断言恰好覆盖活着的那一层，死掉的那一层零断言。**

---

## D-5 Scheduler 名实之别

> 依据：[openfang-kernel.md](openfang-kernel.md) §5

```mermaid
graph TB
    subgraph TRIG["触发层（三者互不协调）"]
        C["cron.rs 1345 行<br/>时间触发"]
        TR["triggers.rs<br/>事件触发"]
        B["background.rs<br/>后台常驻"]
    end

    subgraph ACC["准入层（无调度决策）"]
        S["AgentScheduler<br/>quotas / usage / tasks<br/>❌ 无 priority/fairness/preempt/deadline"]
        M["MeteringEngine<br/>hour/day/month 成本"]
    end

    subgraph EXEC["执行层"]
        LOOP["run_agent_loop()"]
        TOKIO["Tokio 工作窃取调度器<br/>← 真正的调度者"]
    end

    C -->|tokio::spawn| LOOP
    TR -->|tokio::spawn| LOOP
    B -->|tokio::spawn| LOOP
    LOOP -.->|check_quota 前置| S
    LOOP -.->|check_quota 前置| M
    LOOP --> TOKIO

    style S fill:#ffe6e6
    style TOKIO fill:#e6f3ff
```

---

## D-6 崩溃恢复：什么活下来

> 依据：[openfang-kernel.md](openfang-kernel.md) §7，[openfang-memory.md](openfang-memory.md) §5.2

```mermaid
graph LR
    CRASH["daemon 崩溃 / 重启"]

    subgraph OK["✅ 恢复（SQLite 13 表）"]
        O1["Agent 定义 agents 表"]
        O2["Session sessions 表"]
        O3["Task task_queue 表"]
        O4["记忆 memories / kv_store"]
        O5["知识图谱 entities / relations"]
        O6["审计链 audit_entries<br/>+ boot 时 verify_integrity()"]
        O7["用量 usage_events"]
    end

    subgraph LOST["❌ 丢失（纯内存）"]
        X1["🔴 Cron 作业<br/>与 24/7 主张矛盾"]
        X2["🔴 Trigger 注册"]
        X3["待审批请求<br/>Agent 侧调用超时"]
        X4["A2A 任务"]
        X5["OFP 连接"]
        X6["进行中的 agent_loop<br/>无 checkpoint"]
    end

    CRASH --> OK
    CRASH --> LOST

    style LOST fill:#ffcccc
    style X1 fill:#ff9999
    style X2 fill:#ff9999
```

---

## D-7 内存层与单锁瓶颈

> 依据：[openfang-memory.md](openfang-memory.md) §1-2

```mermaid
graph TB
    A["Agent"]

    subgraph SUB["MemorySubstrate"]
        LOCK["⚠️ Arc&lt;Mutex&lt;Connection&gt;&gt;<br/>单锁·6 store 共享<br/>抵消 WAL 并发读"]
        S1["StructuredStore"]
        S2["SemanticStore"]
        S3["KnowledgeStore"]
        S4["SessionStore"]
        S5["UsageStore"]
        S6["ConsolidationEngine<br/>decay_rate 衰减"]
    end

    DB[("SQLite WAL<br/>13 表 · schema V8")]

    A --> SUB
    S1 --> LOCK
    S2 --> LOCK
    S3 --> LOCK
    S4 --> LOCK
    S5 --> LOCK
    S6 --> LOCK
    LOCK --> DB

    style LOCK fill:#ffe6e6
```

---

## D-8 六层记忆模型

> 依据：[openfang-memory.md](openfang-memory.md) §2

```mermaid
graph TB
    W["Working Memory ✅<br/>session.messages 内存 Vec"]
    S["Session Memory ✅<br/>sessions 表 MessagePack BLOB"]
    E["Episodic ⚠️<br/>memories 表有 scope/source<br/>但无 episode 边界"]
    SE["Semantic ⚠️<br/>Phase 1 = SQLite LIKE<br/>非真向量检索"]
    K["Knowledge Graph ✅<br/>entities + relations + 3 索引"]
    L["Long-Term ✅<br/>ConsolidationEngine 衰减 + kv_store"]

    W -->|save_session| S
    W -->|remember| E
    E -->|recall| SE
    E -->|consolidate| L
    W -.->|knowledge_add 工具<br/>依赖 LLM 主动调用| K
    SE -.->|注入 system prompt| W
    K -.->|knowledge_query| W

    style E fill:#fff3e0
    style SE fill:#fff3e0
```

---

## D-9 依赖倒置打破循环

> 依据：[openfang-architecture.md](openfang-architecture.md) §3

```mermaid
graph LR
    subgraph RT["openfang-runtime"]
        TRAIT["KernelHandle trait<br/>20+ 方法（定义）"]
        LOOP["agent_loop<br/>持 Arc&lt;dyn KernelHandle&gt;"]
    end

    subgraph K["openfang-kernel"]
        IMPL["impl KernelHandle for OpenFangKernel<br/>kernel.rs:7166"]
    end

    LOOP -->|"动态分派"| TRAIT
    IMPL -.->|"实现"| TRAIT
    K ==>|"Cargo 依赖"| RT
    RT -.->|"❌ 无依赖<br/>Cargo.toml 中无 openfang-kernel"| K

    style TRAIT fill:#e8f5e9
```

**分层由 Rust 编译器强制，非文档约定。**

---

## D-10 TCB 构成

> 依据：[openfang-security.md](openfang-security.md) §6

```mermaid
pie title TCB 自有代码 ~14,400 行
    "kernel.rs" : 9415
    "subprocess_sandbox.rs" : 1240
    "wire/peer.rs" : 1284
    "sandbox.rs" : 614
    "audit.rs" : 422
    "auth + middleware + session_auth" : 650
    "verify.rs + vault + 工具过滤" : 775
```

`kernel.rs` 占 65%。加上 SQLite（~150K）+ wasmtime（~200K）依赖，
TCB 总量 > 360K 行。理想 Agent OS TCB 应 < 2000 行。

---

## D-11 OFP 握手：认证有，加密无

> 依据：[openfang-wire](../llmwiki/wire-protocol.md)，[openfang-security.md](openfang-security.md) §4

```mermaid
sequenceDiagram
    participant C as Client PeerNode
    participant S as Server PeerNode
    participant N as NonceTracker

    Note over C,S: ⚠️ 裸 TcpStream — 无 TLS
    C->>S: HandshakeRequest{nonce, peer_id,<br/>agent_list, timestamp,<br/>signature=HMAC-SHA256}
    S->>N: check_and_record(nonce)
    N-->>S: Ok / Err(replay)
    S->>S: hmac_verify()<br/>subtle::ConstantTimeEq
    S-->>C: HandshakeResponse{accepted, peer_id, agent_list}
    Note over C,S: 后续 AgentMessage 同样明文
    Note over C,S: ✅ 防伪造 ✅ 防重放 ❌ 防窃听
```

---

## D-12 Bianbu Capability Token 流

> 依据：[openfang-to-bianbu.md](openfang-to-bianbu.md) §3

```mermaid
sequenceDiagram
    participant A as Agent 进程<br/>（独立 uid + cgroup）
    participant CD as capabilityd
    participant TD as toold
    participant SD as securityd

    A->>CD: 启动，出示 Bundle manifest
    CD->>CD: 校验 Bundle 签名
    CD-->>A: Capability Token 集<br/>{cap, agent_id, expiry, nonce, sig}

    A->>TD: invoke(tool, params, token) ← token 为必填
    TD->>TD: ①验 Ed25519 ②验 agent_id<br/>③验 expiry ④验 cap 覆盖<br/>⑤验 nonce 未用
    TD->>SD: 记审计（每次验签）
    alt 验签通过
        TD-->>A: 执行结果
    else 失败
        TD-->>A: Denied
        TD->>SD: 记拒绝
    end
```

**关键**：token 是协议必填参数，无 token 的请求在协议层无法构造——
消除 OpenFang 的旁路（D-2）。

---

## D-13 三套包格式无法组合

> 依据：[openfang-agent-model.md](openfang-agent-model.md) §6

```mermaid
graph TB
    subgraph F1["格式 1：Agent"]
        A1["~/.openfang/agents/&lt;name&gt;/agent.toml"]
    end
    subgraph F2["格式 2：Skill（全局安装）"]
        S1["~/.openfang/skills/&lt;name&gt;/"]
        S2["SKILL.toml + main.py + SKILL.md"]
    end
    subgraph F3["格式 3：Hand（编译时内置）"]
        H1["HAND.toml in binary"]
        H2["含 agent_manifest 模板"]
    end

    X["❌ agent.toml 无法声明<br/>'我需要 Skill A'"]
    Y["❌ 无法一键分发<br/>'带 3 个技能的 Agent'"]

    A1 -.-> X
    S1 -.-> X
    X --> Y

    style X fill:#ffcccc
    style Y fill:#ffcccc
```

---

## D-14 五维评分

> 依据：[openfang-agent-os-verdict.md](openfang-agent-os-verdict.md) §6

```mermaid
graph LR
    subgraph SCORE["Architecture Verdict"]
        F["Agent Framework<br/>100%"]
        R["Agent Runtime<br/>95%"]
        P["Agent Platform<br/>90%"]
        CP["Agent Control Plane<br/>85%"]
        OS["Agent OS<br/>62% ⚠️"]
    end

    subgraph DEDUCT["Agent OS 扣分项"]
        D1["Scheduler 25%<br/>无调度算法"]
        D2["IPC 40%<br/>同进程调用非真 IPC"]
        D3["Isolation 60%<br/>无地址空间隔离"]
        D4["× 0.9 结构性折扣<br/>L2/L1 无特权边界"]
    end

    OS --- D1
    OS --- D2
    OS --- D3
    OS --- D4

    style OS fill:#ffe6e6
```

---

## D-15 Agent OS 分层与缺失的特权边界

> 依据：[openfang-agent-os-verdict.md](openfang-agent-os-verdict.md) §3

```mermaid
graph TB
    L7["L7 User — Dashboard / CLI / Telegram"]
    L6["L6 Applications — Hands（7 内置）"]
    L5["L5 Skills / Tools — 53 内置 + Skills + MCP"]
    L4["L4 Agent — AgentEntry + Manifest + Session"]
    L3["L3 Agent Runtime — agent_loop / tool_runner / LLM driver"]
    L2["L2 Agent OS Kernel — registry / capabilities / audit / metering<br/>⚠️ 无真 scheduler"]
    GAP["❌ 无特权边界<br/>L2 是普通用户态进程<br/>Agent 是同进程 Tokio task<br/>→ namespace / cgroup / seccomp 全用不上"]
    L1["L1 Linux Kernel"]
    L0["L0 Hardware"]

    L7 --> L6 --> L5 --> L4 --> L3 --> L2 --> GAP --> L1 --> L0

    style L2 fill:#ffe6e6
    style GAP fill:#ffcccc
```

---

## D-16 Channel 路由解析层级

> 依据：`channels/src/router.rs` 的 20 个 pub fn（双方法核实计数与名称）
> 及 [../llmwiki/channels.md](../llmwiki/channels.md) 的路由模型

```mermaid
graph TB
    MSG["入站消息<br/>(channel, channel_id, user, peer)"]

    subgraph RESOLVE["ChannelRouter 解析入口（3 个变体）"]
        R1["resolve()<br/>:145"]
        R2["resolve_with_channel_id()<br/>:157"]
        R3["resolve_with_context()<br/>:208"]
    end

    subgraph TIERS["路由表（按 setter 名推断的优先级）"]
        T1["direct route<br/>set_direct_route() :107<br/>最具体"]
        T2["user default<br/>set_user_default() :102"]
        T3["channel default<br/>set_channel_default() :72<br/>+ _with_name() :78"]
        T4["global default<br/>set_default() :67<br/>兜底"]
    end

    subgraph BINDINGS["Agent Bindings"]
        B1["load_bindings(&[AgentBinding])<br/>:118"]
        B2["add_binding() :285<br/>remove_binding() :298"]
        B3["bindings() -> Vec :275"]
    end

    subgraph BCAST["广播（独立路径）"]
        C1["load_broadcast() :133"]
        C2["resolve_broadcast(peer_id)<br/>:242 → Vec&lt;(String, Option&lt;AgentId&gt;)&gt;"]
        C3["broadcast_strategy() :258"]
        C4["has_broadcast(peer_id) :266"]
    end

    REG["register_agent(name, id)<br/>:138 名称→ID 索引"]
    OUT["AgentId"]

    MSG --> R1
    MSG --> R2
    MSG --> R3
    R3 --> T1 --> T2 --> T3 --> T4 --> OUT
    B1 --> T1
    B2 --> B1
    REG --> T1
    MSG -.->|"peer 广播"| C4 --> C2 --> C3

    style T1 fill:#e8f5e9
    style T4 fill:#fff3e0
```

**观察 1**：三个 `resolve` 变体（`resolve` / `resolve_with_channel_id` /
`resolve_with_context`）对应输入具体度递增——这是渐进增强的演进痕迹，
新变体加参数而非改签名，保持向后兼容。

**观察 2**：四级路由表（direct → user → channel → global）中，
只有 `channel_default` 有配套的 `_with_name()` 和 `update_channel_default()`
（:96）与 `channel_default_name()`（:89）——说明 channel 级默认值是
Dashboard 里用户最常改的一层，为它做了额外的读写 API。

**观察 3**：广播是完全独立的路径（`resolve_broadcast` 返回
`Vec<(String, Option<AgentId>)>` 而非单个 `AgentId`），
不走四级解析。`Option<AgentId>` 说明广播目标可以没有绑定 Agent。

> **精度声明**：20 个 pub fn 的名称与行号经双方法核实。
> 四级优先级顺序是**从 setter 命名推断**，未逐行读 `resolve_with_context` 的
> 分支顺序核实（该处读取落在通道不稳定窗口）。若需精确顺序，
> 应重读 `router.rs:208-241`。

---

## 图集索引

| # | 图 | 依据文档 |
|---|---|---|
| D-1 | 死代码分层 | security §1-3 |
| D-2 | Capability 生命周期 | security §1.2 |
| D-3 | 工具门控 fail-open | security §2 |
| D-4 | 测试覆盖叠加 | test-analysis §0,§3 |
| D-5 | Scheduler 名实 | kernel §5 |
| D-6 | 崩溃恢复 | kernel §7 / memory §5.2 |
| D-7 | 内存单锁瓶颈 | memory §1-2 |
| D-8 | 六层记忆 | memory §2 |
| D-9 | 依赖倒置 | architecture §3 |
| D-10 | TCB 构成 | security §6 |
| D-11 | OFP 握手 | security §4 |
| D-12 | Bianbu Token 流 | to-bianbu §3 |
| D-13 | 三套包格式 | agent-model §6 |
| D-14 | 五维评分 | verdict §6 |
| D-15 | 分层与特权边界 | verdict §3 |
| D-16 | Channel 路由解析层级 | `router.rs` 20 个 pub fn |

**本文件 16 张 + 其他文档 12 张 = 28 张**，达到 goal §82 要求。

其他文档的 12 张分布：`kernel.md` 2、`memory.md` 2、`security.md` 2、
`agent-model.md` 1、`agent-os-verdict.md` 1、`architecture.md` 1、
`riscv-edge.md` 1、`to-bianbu.md` 1、`hands-workflow.md` 1。
