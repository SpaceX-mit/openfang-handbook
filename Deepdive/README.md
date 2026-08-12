# OpenFang Deep Dive — 交付说明

> 调研日期：2026-08-07 | 代码版本：v0.6.9 | 源文件：259 个 `.rs`
> 方法论：`SOURCE CODE > DOCUMENTATION > README > MARKETING`

---

## 覆盖度声明（先读这一节）

goal 文档要求 20 份文档、28 张 Mermaid、20 个 ADR、30 条 Lessons、20 个 Threat、20 个 Gap。
本轮完成的是**核心骨架 + 全部关键结论**，不是全部 100 节。诚实说明：

| goal 章节 | 状态 | 落在哪份文档 |
|-----------|------|-------------|
| 1 核心问题 / 98 Verdict | ✅ 完成 | `openfang-agent-os-verdict.md` |
| 3-5 仓库 / crate / 依赖图 | ✅ 完成 | `openfang-architecture.md` |
| 6-7 Kernel / Boot | ✅ 完成 | `openfang-kernel.md` |
| 8-10 Agent Process / Lifecycle / Manifest | ✅ 完成 | `openfang-agent-model.md` |
| 11-15 Capability / Security / Sandbox | ✅ 完成 | `openfang-security.md` |
| 16-19 Memory / KG / Persistence | ✅ 完成 | `openfang-memory.md` |
| 26 Scheduler | ✅ 完成（重要发现） | `openfang-kernel.md` §5 |
| 41-42 Resource / Scheduling | ✅ 完成 | `openfang-kernel.md` §5 |
| 54-55 Threat Model / Gap | ✅ 完成 | `openfang-security.md` §7-8 |
| 56-58 OS Layer / Primitive / Mapping | ✅ 完成 | `openfang-agent-os-verdict.md` |
| 66-67 Bianbu Mapping | ✅ 完成 | `openfang-to-bianbu.md` |
| 68 Limitations（≥20 条） | ✅ 完成 | `openfang-limitations.md` |
| 69-71 RISC-V / Edge / Offline | ✅ 完成 | `openfang-riscv-edge.md` |
| 76-77 Lessons / ADR | ✅ 完成 | `openfang-adr-lessons.md` |
| 20-23 Runtime / Loop / LLM | ⚠️ 已在 llmwiki 覆盖 | `../llmwiki/runtime.md`、`llm-drivers.md` |
| 24-25 Hands / Hand vs Agent | ✅ 完成 | `openfang-hands-workflow.md` §1-5 |
| 27 Workflow Engine | ⚠️ 部分（编排表达力未核实） | `openfang-hands-workflow.md` §6-7 |
| 29-33 OFP / A2A / MCP / Skill / Channel | ⚠️ 已在 llmwiki 覆盖 | `../llmwiki/wire-protocol.md` 等 |
| 34-37 API / CLI / Daemon | ⚠️ 部分（API 已覆盖） | `../llmwiki/api.md` |
| 59-64 竞品对比 | ❌ 未做 | 需要读 OpenHands/OpenClaw 源码，本仓库没有 |
| 84 ER 图 | ⚠️ 表结构已列，未画 ER | `openfang-memory.md` §3 |
| 86 Test 分析 | ✅ 完成 | `openfang-test-analysis.md` |
| 87 Benchmark | ❌ 未做 | 需实机运行，无硬件 |
| 88 Code Quality | ✅ 完成 | `openfang-code-quality.md` |
| 82 Mermaid 图集 | ⚠️ 27/28 | `openfang-diagrams.md`（15 张）+ 其他文档 12 张 |

**未做的部分为什么不做**：竞品对比（59-64）需要 OpenHands、OpenClaw、Hermes、ZeroClaw、LangGraph
的源码，这些不在当前仓库内，无法遵守"源码优先"原则；凭 README 对比会违反 goal §2 的第一原则。
Benchmark（87）需要实机运行 RISC-V 硬件。这两块需要额外输入才能做。

---

## 文档清单

| 文件 | 行数 | 内容 |
|------|------|------|
| [openfang-agent-os-verdict.md](openfang-agent-os-verdict.md) | — | **核心结论**：是不是 Agent OS，五维评分，OS primitive 映射 |
| [openfang-architecture.md](openfang-architecture.md) | — | 仓库结构、14 crate 逆向、依赖图、分层验证 |
| [openfang-kernel.md](openfang-kernel.md) | — | Kernel 逆向、boot 时序、**Scheduler 真相** |
| [openfang-agent-model.md](openfang-agent-model.md) | — | Agent-as-Process、生命周期状态机、Manifest 解剖 |
| [openfang-security.md](openfang-security.md) | — | Capability 强制链、16 层验证、WASM 沙箱、威胁模型、TCB |
| [openfang-memory.md](openfang-memory.md) | — | 四存储层、13 张表、崩溃恢复语义 |
| [openfang-limitations.md](openfang-limitations.md) | — | What OpenFang Gets Wrong（22 条） |
| [openfang-riscv-edge.md](openfang-riscv-edge.md) | — | RISC-V 可行性、边缘部署、离线模式 |
| [openfang-to-bianbu.md](openfang-to-bianbu.md) | — | Bianbu Agent OS 映射、MUST/SHOULD/NOT ADOPT |
| [openfang-adr-lessons.md](openfang-adr-lessons.md) | — | 20 ADR + 30 条 Architecture Lessons |
| [openfang-hands-workflow.md](openfang-hands-workflow.md) | — | Hands 激活时序、结构化构造、Workflow 引擎、五概念区分 |
| [openfang-test-analysis.md](openfang-test-analysis.md) | — | 2,698 测试中仅 2 个授权断言、覆盖分布倒挂 |
| [openfang-code-quality.md](openfang-code-quality.md) | — | unsafe/unwrap/panic 审计、CI 三个盲区 |
| [openfang-diagrams.md](openfang-diagrams.md) | — | 15 张 Mermaid 图集（全部基于已核实事实） |

---

## 三个最重要的发现（源码证据）

### 1. Scheduler 名不副实

`crates/openfang-kernel/src/scheduler.rs` 中的 `AgentScheduler` 只有三个 `DashMap`：
`quotas` / `usage` / `tasks`。**没有任何调度逻辑**——无 priority、无 fairness、无 preemption、
无 deadline、无 run queue。它是一个**配额记账器 + JoinHandle 持有者**。

真正的"何时执行"由三个独立组件决定：`cron.rs`（时间触发）、`triggers.rs`（事件触发）、
`background.rs`（后台执行）。三者互不协调，没有统一的调度决策点。

**这直接影响 Agent OS 判定**——传统 OS 的 scheduler 是内核最核心的子系统。

### 2. Supervisor 不监督

`crates/openfang-kernel/src/supervisor.rs` 的 `Supervisor` 只有 `watch::channel<bool>` 关机信号
+ 三个计数器（restart_count / panic_count / agent_restarts）。它**不主动探活**，
只被动记账。心跳检测在独立的 `heartbeat.rs`。

### 3. AgentRegistry 是纯内存的，持久化在上一层

`registry.rs` 里**没有一行 SQLite**——三个 `DashMap`（agents / name_index / tag_index）。
持久化是 write-through 到 `MemorySubstrate`：

```rust
// kernel.rs:1754, 3398, 3434, 3473 ...
let _ = self.memory.save_agent(&entry);   // ← 注意 let _
// kernel.rs:1261 boot 时恢复
match kernel.memory.load_all_agents() { ... }
```

**大部分 `save_agent` 调用用 `let _ =` 丢弃错误**（kernel.rs 3398/3434/3473，
routes.rs 5803/6108/9674）。磁盘满或 DB 锁定时，Agent 状态变更静默丢失，内存与 DB 分叉。

> 这条同时修正了 `llmwiki/kernel.md` 中"AgentRegistry 后端：SQLite"的表述——
> 不准确，registry 本身无 SQLite，持久化在 MemorySubstrate 层。

---

## 与 llmwiki/ 的关系

`llmwiki/` 是**功能导向**的开发者手册（17 份，回答"这个模块怎么用"）。
`Deepdive/` 是**架构判定导向**的研究报告（回答"这算不算 Agent OS，能不能借鉴到 Bianbu"）。

重叠部分（Runtime / Channels / Skills / Hands / OFP / A2A / MCP）不重写，直接引用 llmwiki。
