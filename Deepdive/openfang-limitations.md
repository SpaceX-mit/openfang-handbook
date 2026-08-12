# What OpenFang Gets Wrong

> 回答 goal §68（要求至少 20 条）
> 共 24 条，每条含源码证据 + 影响 + Bianbu 启示

---

## 架构债

### L-01：kernel.rs 是 God Object

**证据**：单文件 9415 行，`OpenFangKernel` 约 60 个字段。其中约 40% 不是内核职责：
`browser_ctx`（Playwright）、`tts_engine`（语音合成）、`media_engine`、
`web_ctx`、`extension_health`、`skill_config_overrides`。

**影响**：
- TCB 膨胀到 14.4K 行，其中 kernel.rs 占 65%，无法形式化验证或第三方审计
- 任何子系统改动都要碰这个文件，合并冲突高发
- 单元测试困难（构造 60 字段的 kernel 需要大量 mock）

**Bianbu 启示**：`agentd` 必须拆成独立进程/服务。
把 `browser`/`tts`/`media` 这类执行资源放到独立的 `toold`，
`agentd` 只保留 registry + lifecycle + capability。

---

### L-02：MemorySubstrate 单一全局 Mutex

**证据**：
```rust
pub struct MemorySubstrate { conn: Arc<Mutex<Connection>>, ... }
// 6 个 store 全部 Arc::clone(&shared)
```

**影响**：开了 WAL（允许并发读）却在应用层用单 Mutex 串行化所有操作。
Agent 数量增加时这是首个瓶颈——知识图谱查询会阻塞其他 Agent 的 session 保存。

**Bianbu 启示**：`memoryd` 用连接池（读写分离），
或按 Agent 分库（`agent_<id>.db`）。

---

### L-03：三套并行的包机制且互不统一

**证据**：Agent（`agent.toml`）、Skill（`SKILL.toml` + 代码）、
Hand（`HAND.toml`，编译时内置）三套格式，无法互相引用。
`agent.toml` 里无法声明"我需要 Skill A"。

**影响**：用户要分发一个"带 3 个技能的 Agent"必须手动装 3 个技能再放 manifest，
无法一键分发。

**Bianbu 启示**：设计**单一** Agent Bundle 格式（见 to-bianbu.md §4），
Skill 作为 bundle 内的目录而非全局安装物。

---

### L-04：无 feature flags，无法裁剪

**证据**：`Cargo.toml` 只定义了 `http-memory` 一个 feature。
40+ channel、wasmtime、tauri、所有 LLM driver 全部无条件编译。

**影响**：
- 二进制 32MB，边缘设备无法裁剪
- RISC-V 移植时无法绕过 `ring`/`wasmtime` 等风险依赖
- 只用 Telegram 的用户也要编译 39 个用不到的 channel

**Bianbu 启示**：从第一天就做 feature 拆分。
`default = ["core"]`，channel/sandbox/driver 全部可选。

---

### L-05：kernel 可被嵌入，非特权进程

**证据**：`api/server.rs::build_router()` 的注释明说是为了让
`openfang-desktop` 能在同进程内嵌 kernel。

**影响**：这证明"Kernel"是逻辑概念而非特权概念。L2（Agent OS Kernel）
与 L1（Linux Kernel）之间无隔离边界，无法用 namespace/cgroup/seccomp 隔离 Agent。

**Bianbu 启示**：这恰恰是 Bianbu 的最大机会——
把 `agentd` 做成真正的系统服务（systemd unit + 独立 uid + cgroup + seccomp），
Agent 跑在独立进程/容器里。这是 OpenFang 做不到而 Bianbu 能做到的差异化。

---

## 安全缺陷

### L-06：Capability 声明但从不强制（最严重）

**证据**：`CapabilityManager::check()` 生产代码零调用
（仅 `capabilities.rs` 自身单元测试）。

**影响**：`FileWrite("~/safe/**")`、`NetConnect("*.example.com")`、
`MemoryWrite("agent:x/*")`、`ShellExec("git *")` 全部不生效。
唯一真实的能力强制是 `spawn_agent_checked` 的继承检查。

**Bianbu 启示**：`capabilityd` 必须是**执行路径上的强制点**，
不能是旁路的记账服务。设计时就要让"不经过 capabilityd 就无法调用工具"
在架构上成立（例如工具调用必须携带 capabilityd 签发的 token）。

---

### L-07：三个安全模块是死代码

**证据**：

| 模块 | 行数 | 单元测试 | 调用点 |
|------|------|---------|--------|
| `shell_bleed.rs` | 296 | 有 | 0 |
| `tool_policy.rs` | ~430 | 11 个 | 0 |
| `CapabilityManager::check()` | ~30 | 2 个 | 0 |

> 初稿把 WASM skill runtime 列为第四个。已更正：WASM **Skill** 未实现
> （`loader.rs` 返回 `NotAvailable`），但 WASM **Agent** 路径是活代码
> ——`kernel.rs:5623` 有生产调用，`wasm_agent_integration_test.rs`
> 有 fuel 耗尽断言。见 [openfang-security.md](openfang-security.md) §5.3。

**影响**：`tool_policy.rs` 死代码意味着 config.toml 的工具策略无效，
且子 Agent 可以调用 `cron_create`/`hand_activate`/`process_start`
（`SUBAGENT_DENY_ALWAYS` 名单不生效）。

**Bianbu 启示**：见 L-11（测试策略问题）。

---

### L-08：工具列表 fail-open

**证据**：
```rust
let tools_unrestricted = declared_tools.is_empty()
    || declared_tools.iter().any(|t| t == "*");
```

**影响**：忘写 `capabilities.tools` 的 manifest 得到全部 53 个内置工具，
含 `shell_exec`、`file_delete`、`agent_spawn`。**最高权限是默认值。**

**Bianbu 启示**：deny-all 必须是默认。空配置 = 零权限。

---

### L-09：boot 时自动加载未签名 manifest

**证据**：`kernel.rs:1472` 扫描 `~/.openfang/agents/*/agent.toml` 直接 `spawn_agent()`。
`manifest_signing.rs` 只 `hash_manifest()` 不验签。

**影响**：对该目录的文件写权限等于全 Agent 权限
（放一个 `tools = ["*"]` 的 manifest 即可）。

**Bianbu 启示**：Agent Bundle 必须验签，签名链接到 CA。

---

### L-10：OFP 明文传输

**证据**：`wire/peer.rs` 用裸 `TcpStream`，只有 HMAC 签名，无加密。

**影响**：局域网可窃听全部跨节点 Agent 通信内容。

**Bianbu 启示**：Agent IPC 用 mTLS 或 Unix socket + SO_PEERCRED。

---

### L-11：单元测试给了虚假安全感

**证据**：`tool_policy.rs` 有 11 个通过的单元测试，
`shell_bleed.rs` 有测试，`CapabilityManager::check()` 有 2 个测试。
**全部通过，CI 全绿，但功能未接线。**

**影响**：1744+ 测试的数字看起来很强，但至少 4 个安全模块的测试
只验证了函数逻辑，没验证集成。README 的 "16 security layers"
在文件层面成立，在生效层面不成立。

**Bianbu 启示**：安全功能必须有**集成测试/E2E 测试**，
断言"未授权操作确实被拒绝"，而不是"检查函数返回 Denied"。
CI 应有 dead-code 检测（`#[deny(dead_code)]` 或 cargo-udeps）。

---

### L-12：污点标签在检查点硬编码，无传播

**证据**：`tool_runner.rs:37/64` 中 `TaintedValue::new(cmd, labels, "llm_tool_call")`
的 labels 是当场 `HashSet::new()` + `insert()` 的字面量。
`merge_taint()` / `declassify()` 生产零调用。

**影响**：`check_sink()` 退化为恒真判断。web_fetch 返回的网页内容
进入 session 时不带任何标签，间接提示注入无法追踪。

**Bianbu 启示**：污点必须随数据结构传播。
`ToolResult` 应携带 `taint_labels` 字段，由产生数据的工具打标。

---

### L-13：LLM 输出无密钥扫描

**证据**：`agent_loop.rs` 返回 `AgentLoopResult.response` 前无任何正则扫描。

**影响**：LLM 可能在回复中打印 `sk-ant-xxx`，经 channel 发给最终用户或写入日志。

**Bianbu 启示**：输出侧扫描是必须的，不能只防输入。

---

### L-14：审计链验证失败不阻止启动

**证据**：
```rust
if let Err(e) = log.verify_integrity() {
    tracing::error!("Audit trail integrity check FAILED on boot: {e}");
}   // ← 没有 return / panic，继续启动
```

**影响**：被篡改的审计链会继续追加新记录，攻击痕迹被掩盖后系统照常运行。

**Bianbu 启示**：审计链断裂应进入安全降级模式（只读 / 拒绝新操作 / 告警）。

---

### L-15：exec_policy=Full 跳过全部污点检查

**证据**：
```rust
let is_full_exec = exec_policy.is_some_and(|p| p.mode == ExecSecurityMode::Full);
if !is_full_exec {
    if let Some(v) = check_taint_shell_exec(command) { /* 阻断 */ }
}
// 注释：Skip heuristic taint patterns for Full exec policy
//       (e.g. hand agents that need curl)
```

**影响**：为了让 Hand agent 能用 curl，整个污点检查被一刀切跳过，
包括元字符检测之外的所有启发式防护。粒度过粗。

**Bianbu 启示**：策略放宽应该是**逐项**的，不是全局开关。
"允许 curl" ≠ "允许任意注入模式"。

---

## 可靠性缺陷

### L-16：Cron、Trigger、Workflow 三者均不持久化

**证据**：`CronScheduler`、`TriggerEngine`、`WorkflowEngine` 都是纯内存结构。
13 张表中无 cron / trigger / workflow 表。

```bash
$ grep -in "sqlite\|rusqlite\|INSERT\|SELECT" kernel/src/workflow.rs
# 零结果
```
`WorkflowEngine` 用 `DashMap` 存 workflows 和 runs，runs 有 `max_runs` 容量上限。

**对照 —— 团队会做持久化，这三处就是没做**：

| | 持久化 | 重启后 |
|---|---|---|
| Cron 作业 | ❌ | 丢失 |
| Trigger 注册 | ❌ | 丢失 |
| Workflow 定义 + 运行记录 | ❌ | 丢失 |
| Task 队列 | ✅ `task_queue` + `idx_task_status_priority` | 保留 |
| Agent 定义 | ✅ `agents` 表 | 保留 |

**影响**：
1. **声称 24/7 自主运行的产品，daemon 重启后定时任务全部丢失** ——
   产品主张与实现的实质性矛盾
2. 用户通过 `POST /api/workflows` 创建的定义在重启后消失

**Bianbu 启示**：`scheduler` 的作业定义、编排层的 workflow 定义与运行记录
都必须持久化，重启后从存储重建。这是 systemd timer / cron 的基本属性。

---

### L-17：持久化错误静默丢弃

**证据**：8 处 `save_agent` 调用中 6 处用 `let _ =`
（kernel.rs:3398/3434/3473，routes.rs:5803/6108/9674）。

**影响**：磁盘满/DB 锁超时时，内存 DashMap 与 SQLite 分叉，
**无任何日志提示**。重启后状态静默回退。

**Bianbu 启示**：write-through 失败必须让调用方知道。
至少 `warn!`，关键路径应 propagate error。

---

### L-18：无 checkpoint，agent_loop 崩溃全丢

**证据**：`run_agent_loop` 是 50 次迭代的循环，无中间状态保存。

**影响**：一个跑到第 40 次迭代的长任务，进程崩溃后从头开始，
已消耗的 token 成本白费。

**Bianbu 启示**：长任务需要 checkpoint（类似 CRIU 或 LangGraph 的 checkpointer）。

---

### L-19：无跨表事务

**证据**：`spawn_agent` 写 `agents` 表 + 可能创建 `sessions` 行，
未见 `BEGIN`/`COMMIT` 包裹。

**影响**：中途崩溃留下孤立行（agent 无 session 或 session 无 agent）。

---

### L-20：Zombie Agent — children 列表只增不减

**证据**：`registry.add_child()` 只 `push`，无对应的 remove。
子 Agent 终止时父的 `children` 不清理。

**影响**：长期运行的父 Agent 的 `children` 无限增长（边际内存泄漏），
且无法区分活跃子 Agent 和已终止的。无 `wait()`/`SIGCHLD` 语义。

**Bianbu 启示**：需要进程回收语义。父 Agent 应能感知子 Agent 结束。

---

## 资源与扩展性

### L-21：无 RSS / 磁盘 / Agent 并发配额

**证据**：`ResourceQuota` 只有 `max_llm_tokens_per_hour` 和三档成本。
`scheduler.rs::check_quota()` 不检查内存。

**影响**：边缘设备（4GB RAM）OOM 风险。OOM killer 杀整个进程，
**所有 Agent 一起死**（无地址空间隔离）。

**Bianbu 启示**：cgroup v2 的 `memory.max` + `pids.max` 必须用上。
这是 Bianbu 相对 OpenFang 的天然优势。

---

### L-22：sessions 表无限增长且无索引

**证据**：13 张表中无清理机制；`sessions.agent_id` 无索引。

**影响**：`list_sessions(agent_id)` 全表扫描，随时间线性变慢。
磁盘占用无上限。

---

### L-23：Scheduler 无调度语义

**证据**：`AgentScheduler` 只有 quotas/usage/tasks 三个 DashMap，
无 priority/fairness/preemption/deadline（详见 kernel.md §5）。

**影响**：无法表达 Agent 优先级，无法保证公平性，过载时只能靠 quota 硬拒绝
而不能优雅降级。

**Bianbu 启示**：`scheduler` 是 Bianbu 最需要从零设计的部分，
OpenFang 在这里没有可借鉴的实现。

---

### L-24：无多租户隔离

**证据**：单 `OpenFangKernel` 实例，`AuthManager` 有 4 级角色但
所有 Agent 共享同一 registry / memory / 预算。

**影响**：用户 A 的 Admin 可以 kill 用户 B 的 Agent。
`memories` 表的 scope 是命名约定（`agent:{id}`），非强隔离。

---

## 汇总

| 类别 | 条数 | 最严重 |
|------|------|--------|
| 架构债 | 5 | L-01 God Object（TCB 膨胀） |
| 安全缺陷 | 10 | L-06 Capability 零强制 |
| 可靠性 | 5 | L-16 Cron/Trigger/Workflow 不持久化（矛盾产品主张） |
| 资源扩展 | 4 | L-21 无 RSS 配额（边缘 OOM） |
| **合计** | **24** | — |

**三条对 Bianbu 最有价值的教训**：

1. **L-06 + L-07**：能力系统必须在执行路径上，不能旁路。
   有测试 ≠ 生效，需要集成测试断言"拒绝确实发生"。
2. **L-05 + L-21**：Bianbu 的最大差异化机会是把 Agent 隔离做实
   （独立进程 + cgroup + seccomp），这是 OpenFang 架构上做不到的。
3. **L-16**：任何声称"长期运行"的调度状态必须持久化。
