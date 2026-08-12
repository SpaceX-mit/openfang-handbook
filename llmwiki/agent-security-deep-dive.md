# Agent 安全深度调研：OpenFang 实现 vs AgentOS 行业要求

> 调研日期：2026-08-07 | 代码版本：v0.6.9  
> 参考标准：OWASP LLM Top 10 (2025)、NIST AI RMF、MITRE ATLAS、Google/Microsoft Agent Security Research

---

## 研究范围

本文档逐一审查 OpenFang 的安全实现代码，对比 Agent OS 行业安全基线，找出已达成项和 Gap。

**审查的源文件**：
- `crates/openfang-types/src/capability.rs` — 能力模型
- `crates/openfang-types/src/taint.rs` — 污点追踪
- `crates/openfang-kernel/src/capabilities.rs` — 能力管理器
- `crates/openfang-kernel/src/approval.rs` — 执行审批
- `crates/openfang-kernel/src/auth.rs` — RBAC 认证
- `crates/openfang-runtime/src/audit.rs` — Merkle 审计链
- `crates/openfang-runtime/src/tool_policy.rs` — 工具策略
- `crates/openfang-runtime/src/shell_bleed.rs` — Shell 注入防护
- `crates/openfang-runtime/src/sandbox.rs` — WASM 沙箱
- `crates/openfang-runtime/src/web_fetch.rs` — SSRF 防护
- `crates/openfang-wire/src/peer.rs` — 线协议 HMAC 认证
- `SECURITY.md` — 官方安全策略

---

## 一、AgentOS 行业安全基线

综合 OWASP LLM Top 10 (2025)、NIST AI RMF 和 Agent 特定安全研究，Agent OS 应满足的安全要求分为 8 大类：

### S1. 访问控制与最小权限
- Agent 只能访问显式授权的资源（Capability-based）
- 多用户分级（RBAC）
- 子 Agent 权限不超过父 Agent（权限非升级）
- 运行时权限撤销

### S2. 提示注入防护（OWASP LLM01）
- 检测并阻断直接提示注入（用户输入中的指令）
- 检测并阻断间接提示注入（来自工具结果、网页内容）
- 防止 Agent 被外部内容劫持执行恶意操作

### S3. 工具执行控制（OWASP LLM08 Excessive Agency）
- 工具调用白名单/黑名单
- 危险工具需人工审批（human-in-the-loop）
- 子 Agent 工具限制（深度感知）
- 工具执行超时

### S4. 数据流安全（OWASP LLM06 Sensitive Info Disclosure）
- 防止 API 密钥等 Secret 泄露到 LLM 上下文
- 防止 Secret 通过 URL 参数外泄
- Shell 命令注入防护
- 进程间数据流污染追踪

### S5. 运行时隔离
- 代码执行沙箱（WASM / 子进程 / 容器）
- Agent 间内存隔离
- 文件系统访问限制（路径穿越防护）
- 网络访问限制（SSRF 防护）

### S6. 审计与可观测性
- 不可篡改的操作日志
- 完整的 Agent 动作追踪
- 日志持久化（daemon 重启后保留）
- 完整性验证接口

### S7. 网络与协议安全
- 节点间通信认证
- 防重放攻击
- TLS 加密
- 速率限制

### S8. 供应链与插件安全（OWASP LLM05）
- 技能/插件完整性校验
- 恶意 Prompt Injection 扫描
- 技能沙箱隔离
- 凭证安全存储

---

## 二、OpenFang 安全实现深度审查

### S1. 访问控制与最小权限

#### 1.1 Capability 模型（capability.rs + capabilities.rs）

**实现细节**：
```rust
// 16 种能力类型，覆盖文件、网络、工具、Agent、内存、Shell、OFP、经济
pub enum Capability {
    FileRead(String), FileWrite(String),        // glob 模式
    NetConnect(String), NetListen(u16),
    ToolInvoke(String), ToolAll,                // ToolAll = 危险标记
    LlmQuery(String), LlmMaxTokens(u64),
    AgentSpawn, AgentMessage(String), AgentKill(String),
    MemoryRead(String), MemoryWrite(String),
    ShellExec(String), EnvRead(String),
    OfpDiscover, OfpConnect(String), OfpAdvertise,
    EconSpend(f64), EconEarn, EconTransfer(String),
}
```

`CapabilityManager` 用 `DashMap<AgentId, Vec<Capability>>` 存储每个 Agent 的授权列表，`check()` 方法遍历列表做 glob 匹配。有完整的单元测试（grant/check/no_grants 三个测试用例）。

**亮点**：
- 粒度极细：`FileRead("~/docs/**")` 和 `FileRead("/etc/**")` 是两个不同能力
- `ToolAll` 有显式危险注释，提醒使用者
- 支持 glob 模式匹配（`api.openai.com:443` 和 `*.example.com:*` 都合法）

#### 1.2 RBAC 多用户认证（auth.rs）

```rust
pub enum UserRole { Viewer = 0, User = 1, Admin = 2, Owner = 3 }

// 动作 → 最低角色要求
Action::ChatWithAgent    → User
Action::ViewConfig       → User
Action::ViewUsage        → Admin
Action::SpawnAgent       → Admin
Action::KillAgent        → Admin
Action::InstallSkill     → Admin
Action::ModifyConfig     → Owner
Action::ManageUsers      → Owner
```

Channel 绑定索引（`channel_type:platform_id` → `UserId`）实现跨渠道统一身份。

#### 1.3 权限继承（spawn_agent_checked）

```rust
async fn spawn_agent_checked(
    manifest_toml: &str,
    parent_id: Option<&str>,
    parent_caps: &[Capability],  // 父 Agent 已有能力
) -> Result<(String, String), String> {
    // 子 manifest 中的每个能力必须 ⊆ parent_caps，否则拒绝
}
```

子 Agent 无法通过 spawn 获取父 Agent 没有的能力（防权限升级）。

---

### S2. 提示注入防护

#### 2.1 污点追踪（taint.rs）— 完整实现

```rust
pub enum TaintLabel {
    ExternalNetwork,  // 来自网络请求
    UserInput,        // 来自用户输入
    Pii,              // 个人身份信息
    Secret,           // API 密钥/密码
    UntrustedAgent,   // 来自不信任 Agent
}
```

**三个 TaintSink 定义**：
```rust
TaintSink::shell_exec()    // 阻断 ExternalNetwork + UntrustedAgent + UserInput
TaintSink::net_fetch()     // 阻断 Secret + Pii
TaintSink::agent_message() // 阻断 Secret
```

**`declassify()` 方法**：显式去除污点标签，是唯一合法的"清洗"路径，审计可追踪。

有 4 个单元测试：注入阻断、外泄阻断、干净数据通过、去污后通过。

#### 2.2 幽灵动作检测（agent_loop.rs）

```rust
fn phantom_action_detected(text: &str) -> bool {
    // 检测 LLM 声称"已发送/已发布"但未实际调用工具的情况
    // 触发后重新进入循环强制执行真实工具调用
}
```

#### 2.3 Shell Bleed 检测（shell_bleed.rs）

扫描 LLM 要执行的脚本文件中是否存在敏感环境变量引用（如 `$OPENAI_API_KEY`）：
- 白名单：`PATH`、`HOME` 等 30+ 安全变量
- 检测 `*.py`、`*.sh`、`*.js` 等脚本中的 `$SECRET_VAR` 模式
- 扫描上限：100KB，防止超大脚本

**注意**：目前仅产生 Warning，不阻断执行（⚠️ 见 Gap 分析）

---

### S3. 工具执行控制

#### 3.1 工具策略引擎（tool_policy.rs）— 完整实现

**设计原则**：deny-wins（任何 deny 优先于所有 allow）

```
优先级：agent 规则 > global 规则 > 隐式允许
```

关键特性：
- **Glob 模式**：`shell_*` 匹配所有 shell 工具
- **命名组**：`@web_tools` 引用工具组，批量控制
- **深度感知**：子 Agent（depth > 0）自动禁用 `cron_create`、`hand_activate`、`process_start` 等
- **叶节点限制**：最大深度前一层禁用 `agent_spawn`、`agent_kill`

```rust
// depth > 0 始终禁用
const SUBAGENT_DENY_ALWAYS: &[&str] = &[
    "cron_create", "cron_cancel", "schedule_create", "schedule_delete",
    "hand_activate", "hand_deactivate", "process_start",
];
// 叶节点额外禁用
const SUBAGENT_DENY_LEAF: &[&str] = &["agent_spawn", "agent_kill"];
```

有 11 个单元测试，覆盖 deny-wins、agent 覆盖 global、组展开、深度限制等场景。

#### 3.2 人工审批系统（approval.rs）

- `ApprovalManager` 持有 `DashMap<Uuid, PendingRequest>` 等待审批请求
- 每个 Agent 最多 5 个并发待审批请求（防 DoS）
- 使用 `tokio::sync::oneshot::channel` 实现异步阻塞等待
- 超时自动拒绝（`ApprovalDecision::TimedOut`）
- 最近100条审批记录保存在 `VecDeque` 中（dashboard 显示）
- 支持热更新策略（`update_policy()`）

**可配置的触发工具列表**（config.toml）：
```toml
[tool_policy]
require_approval = ["shell_exec", "file_delete", "agent_spawn"]
```

#### 3.3 工具超时控制（agent_loop.rs）

```rust
TOOL_TIMEOUT_SECS = 120      // 普通工具默认超时
AGENT_TOOL_TIMEOUT_SECS = 600 // 跨 Agent 调用超时
// 均可通过环境变量覆盖，设为 0 则无限制
```

---

### S4. 数据流安全

#### 4.1 SSRF 防护（web_fetch.rs）

阻断向以下地址的请求：
- `127.0.0.1`/`localhost`（本地回环）
- `10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`（私有 IP）
- `169.254.0.0/16`（AWS EC2 元数据服务）
- `::1`（IPv6 回环）

#### 4.2 URL 污点检查（tool_runner.rs）

```rust
fn check_taint_net_fetch(url: &str) -> Option<String> {
    // 阻断含以下参数的 URL（防 Secret 外泄）
    let exfil_patterns = [
        "api_key=", "apikey=", "token=",
        "secret=", "password=", "Authorization:",
    ];
}
```

#### 4.3 Shell 元字符检测（subprocess_sandbox.rs）

```rust
fn contains_shell_metacharacters(command: &str) -> Option<String> {
    // 阻断: ` $ { } ; | & > < ! \ 等元字符
    // 防止 Shell 命令注入
}
```

#### 4.4 Secret 零化（全局）

```rust
// 所有 API 密钥字段使用 Zeroizing<String>
// 变量离开作用域时自动内存清零，防止内存转储泄露
use zeroize::Zeroizing;
pub api_key: Zeroizing<String>
```

#### 4.5 安全路径解析（文件操作）

`safe_resolve_path()` / `safe_resolve_parent()` 在所有文件工具中使用，防止路径穿越攻击（`../../../etc/passwd`）。

---

### S5. 运行时隔离

#### 5.1 WASM 沙箱（sandbox.rs）

使用 `wasmtime` 运行 WASM 技能模块，双重计量：
- **Fuel 限制**：CPU 使用量上限（防止无限循环）
- **Epoch 中断**：Watchdog 线程定时检查，强制超时

#### 5.2 子进程沙箱（subprocess_sandbox.rs，1240行）

Python/Shell/Node 技能在子进程中运行：
- `env_clear()`：清空所有环境变量，只传入显式允许的变量
- 受限 PATH：只包含必要的二进制目录
- `contains_shell_metacharacters()` 前置检查

#### 5.3 Docker 沙箱（docker_sandbox.rs）

可选的 Docker 容器隔离（`[sandbox] use_docker = true`），提供最强隔离级别。

#### 5.4 Agent 间内存隔离

- Structured Store 使用 `scope` 分隔（`agent:{id}` vs `global`）
- 能力检查 `MemoryRead(scope)` / `MemoryWrite(scope)` 控制跨 Agent 内存访问

---

### S6. 审计与可观测性

#### 6.1 Merkle 哈希链审计日志（audit.rs）— 完整实现

每条记录包含前一条的 SHA-256 哈希，构成不可篡改链：
```
Entry[N].hash = SHA256(seq + timestamp + agent_id + action + detail + outcome + Entry[N-1].hash)
```

**12 种审计动作类型**：
`ToolInvoke`, `CapabilityCheck`, `AgentSpawn`, `AgentKill`, `AgentMessage`,
`MemoryAccess`, `FileAccess`, `NetworkAccess`, `ShellExec`, `AuthAttempt`,
`WireConnect`, `ConfigChange`

**完整性验证**：`verify_integrity()` 重新计算所有哈希，检测任何篡改。

**持久化**：SQLite `audit_entries` 表（schema V8），daemon 重启后完整保留，并在启动时自动验证链完整性。

有 4 个单元测试：完整性、篡改检测、tip 变化、DB 持久化。

---

### S7. 网络与协议安全

#### 7.1 OFP 协议 HMAC-SHA256（peer.rs）

```rust
fn hmac_sign(secret: &str, data: &[u8]) -> String { ... }
fn hmac_verify(secret: &str, data: &[u8], signature: &str) -> bool {
    // 使用 subtle::ConstantTimeEq，防止时序攻击
    subtle::ConstantTimeEq::ct_eq(expected.as_bytes(), sig.as_bytes()).into()
}
```

#### 7.2 NonceTracker 防重放（peer.rs）

```rust
pub struct NonceTracker {
    seen: Arc<DashMap<String, Instant>>,
    window: Duration,  // 5 分钟滑动窗口
}
// 每次握手验证 nonce 唯一性，拒绝重放的握手请求
```

#### 7.3 GCRA 速率限制（rate_limiter.rs）

使用 `governor` crate，基于代价感知的令牌桶算法：
- 按 IP 地址限速
- 防止暴力扫描和 DoS 攻击

#### 7.4 HTTP 安全头（server.rs）

```
Content-Security-Policy
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security
```

---

### S8. 供应链与插件安全

#### 8.1 Ed25519 Manifest 签名（manifest_signing.rs）

```rust
pub fn hash_manifest(toml_content: &str) -> String {
    // SHA-256 内容哈希，用于完整性追踪
}
```

使用 `ed25519-dalek` 对 Agent manifest 签名，防止 manifest 被篡改。

#### 8.2 技能 Prompt Injection 扫描（verify.rs）

安装 Skill 时扫描其内容中的 override 指令和数据外泄模式：
- 检测 "Ignore previous instructions" 类提示注入
- 检测向外部发送数据的可疑模式

#### 8.3 凭证保管库（vault.rs）

- **AES-256-GCM** 加密存储
- 密钥来自 `OPENFANG_VAULT_KEY` 环境变量
- `VaultLocked` 错误类型：钥匙未设置时提供明确错误
- 优先级链：Vault > dotenv > 环境变量

---

## 三、达成 vs Gap 完整对比表

| # | 安全要求 | 标准来源 | OpenFang 实现 | 状态 | Gap 描述 |
|---|---------|---------|--------------|------|---------|
| S1.1 | Agent 最小权限（Capability-based） | NIST AI RMF | 16 种 Capability 类型，glob 匹配，DashMap 存储 | ✅ 已达成 | — |
| S1.2 | 多用户 RBAC | NIST AI RMF | Viewer/User/Admin/Owner 四级，动作级最低角色检查 | ✅ 已达成 | — |
| S1.3 | 子 Agent 权限非升级 | NIST AI RMF | `spawn_agent_checked` 验证子集关系 | ✅ 已达成 | — |
| S1.4 | 运行时能力撤销 | NIST AI RMF | 仅 `revoke_all`，无法撤销单个能力 | ⚠️ 部分 | 只能全量撤销，不支持细粒度运行时撤销 |
| S2.1 | 直接提示注入防护 | OWASP LLM01 | `phantom_action_detected` 幽灵动作检测，taint UserInput 标签 | ⚠️ 部分 | 无系统性 prompt injection 扫描，仅检测副作用缺失 |
| S2.2 | 间接提示注入防护 | OWASP LLM01 | taint 框架存在，但标签在检查点**硬编码构造**，`merge_taint`/`declassify` 生产零调用 | ⚠️ 部分 | 无真实污点传播：web_fetch 结果不带标签即进入上下文。见 [prompt-injection.md](prompt-injection.md) Gap-1 |
| S2.3 | 恶意指令检测（技能内容） | OWASP LLM05 | `verify.rs::scan_prompt_content` 已接线 4 处，Critical 级**删目录 + SecurityBlocked 阻断安装** | ✅ 已达成 | 针对 ClawHub 341 个恶意技能事件（2026-02）的响应 |
| S3.1 | 工具调用白名单/黑名单 | OWASP LLM08 | deny-wins + glob + 命名组，deny 优先于 allow | ✅ 已达成 | — |
| S3.2 | 危险工具人工审批 | OWASP LLM08 | `ApprovalManager`，oneshot channel 阻塞等待，超时自动拒绝 | ✅ 已达成 | — |
| S3.3 | 子 Agent 工具限制（深度感知） | OWASP LLM08 | `filter_tools_by_depth`，SUBAGENT_DENY_ALWAYS 列表 | ✅ 已达成 | — |
| S3.4 | 工具并发数量限制 | OWASP LLM08 | `subagent_max_concurrent = 5`，`subagent_max_depth = 10` | ✅ 已达成 | — |
| S4.1 | Secret 不泄露到 LLM | OWASP LLM06 | taint Secret 标签 + `net_fetch` sink 阻断 | ✅ 已达成 | — |
| S4.2 | LLM 输出中 Secret 扫描 | OWASP LLM06 | **无实现** | ❌ Gap | LLM 可能在回复中输出 API key，无正则扫描检测 |
| S4.3 | Shell 命令注入防护 | OWASP LLM02 | `contains_shell_metacharacters` + taint 双重检查 | ✅ 已达成 | — |
| S4.4 | Secret 内存零化 | NIST AI RMF | `Zeroizing<String>` 全局 API key 字段 | ✅ 已达成 | — |
| S4.5 | 路径穿越防护 | NIST AI RMF | `safe_resolve_path()` 所有文件工具 | ✅ 已达成 | — |
| S4.6 | Shell Bleed（脚本变量泄露） | OWASP LLM06 | `shell_bleed.rs` 有 296 行完整实现，但**零调用点（死代码）** | ❌ Gap | 见 [prompt-injection.md](prompt-injection.md) Gap-2。函数与测试俱在，未在 `tool_runner.rs` 接线 |
| S5.1 | 代码执行沙箱（WASM） | NIST AI RMF | wasmtime fuel + epoch 双重计量 | ✅ 已达成 | — |
| S5.2 | 代码执行沙箱（子进程） | NIST AI RMF | `env_clear()` + 受限 PATH | ✅ 已达成 | — |
| S5.3 | 代码执行沙箱（容器） | NIST AI RMF | Docker 沙箱（可选，默认关闭） | ⚠️ 部分 | 需手动启用，默认不隔离 |
| S5.4 | Agent 间内存隔离 | NIST AI RMF | SQLite scope 分隔（`agent:{id}`），能力检查 MemoryRead/Write | ⚠️ 部分 | 隔离基于命名约定，非密码学强隔离；具有 DB 访问权限的进程可绕过 |
| S5.5 | SSRF 防护 | OWASP LLM02 | 私有 IP + 云元数据端点 + IPv6 回环阻断 | ✅ 已达成 | — |
| S6.1 | 不可篡改审计日志 | NIST AI RMF | Merkle SHA-256 哈希链，tamper detection | ✅ 已达成 | — |
| S6.2 | 审计日志持久化 | NIST AI RMF | SQLite 持久化，daemon 重启后保留，启动时验证 | ✅ 已达成 | — |
| S6.3 | 审计日志完整性验证 API | NIST AI RMF | `GET /api/audit/verify`（SECURITY.md 提及） | ✅ 已达成 | — |
| S6.4 | 审计日志外发/导出 | NIST AI RMF | **无实现** | ❌ Gap | 无 SIEM 集成、无 Syslog 输出、无 S3 归档 |
| S6.5 | 审计日志存储防篡改（OS 层） | NIST AI RMF | SQLite 文件可被 root 直接修改 | ⚠️ 部分 | 链可检测篡改，但无法阻止 DB 文件被直接替换 |
| S7.1 | 节点间通信 HMAC 认证 | NIST AI RMF | HMAC-SHA256 + `ConstantTimeEq` 防时序攻击 | ✅ 已达成 | — |
| S7.2 | 防重放攻击 | NIST AI RMF | NonceTracker 5 分钟滑动窗口 | ✅ 已达成 | — |
| S7.3 | 节点间通信加密（传输层） | NIST AI RMF | **无 TLS**，明文 TCP + HMAC | ❌ Gap | OFP 连接未加密，局域网内可被窃听 |
| S7.4 | API 速率限制 | NIST AI RMF | GCRA `governor` 每 IP 令牌桶 | ✅ 已达成 | — |
| S7.5 | HTTP 安全头 | NIST AI RMF | CSP + X-Frame-Options + HSTS 等 | ✅ 已达成 | — |
| S8.1 | Manifest 完整性签名 | OWASP LLM05 | Ed25519 + SHA-256 哈希追踪 | ✅ 已达成 | — |
| S8.2 | 凭证加密存储 | OWASP LLM05 | AES-256-GCM 保管库 | ✅ 已达成 | — |
| S8.3 | 技能依赖完整性 | OWASP LLM05 | **无 lockfile 验证** | ❌ Gap | Python/npm 包依赖无哈希校验，可被供应链攻击 |
| S8.4 | 技能市场信任等级 | OWASP LLM05 | ClawHub 安装无签名验证 | ⚠️ 部分 | 仅 prompt injection 扫描，无数字签名链 |
| S9.1 | 多租户隔离 | NIST AI RMF | **单租户设计** | ❌ Gap | 无多租户，所有 Agent 共享同一个 kernel 实例 |
| S9.2 | Agent 运行时身份证书 | MITRE ATLAS | Ed25519 仅用于 manifest，运行时无独立 identity | ⚠️ 部分 | Agent 运行时身份仅靠 UUID，无 PKI 证书链 |
| S9.3 | 可信执行环境（TEE） | MITRE ATLAS | **无实现** | ❌ Gap | 不支持 SGX/TrustZone/AMD SEV 等 |
| S9.4 | 对抗性输入检测 | MITRE ATLAS | **无实现** | ❌ Gap | 无 jailbreak 检测、无越狱模式识别 |

---

## 四、Gap 严重性分析与修复建议

### 🔴 高优先级 Gap（影响生产安全）

#### Gap-1：OFP 传输未加密（S7.3）
**风险**：局域网内的任何节点可以嗅探 Agent 间的通信内容，包括 Agent 回复、任务数据。  
**修复方向**：为 `PeerNode` 的 TCP 连接加 TLS（`tokio-rustls`），共享证书或 TOFU 模式。  
**工作量**：中等（约200-300行，主要在 `peer.rs`）  
**关键文件**：`crates/openfang-wire/src/peer.rs`

```rust
// 当前：
let stream = TcpStream::connect(addr).await?;
// 目标：
let tls_stream = TlsConnector::connect(domain, stream).await?;
```

#### Gap-2：Shell Bleed 仅警告不阻断（S4.6）
**风险**：LLM 可能生成含 `$OPENAI_API_KEY` 引用的脚本并执行，Secret 被注入 shell 进程。  
**修复方向**：`scan_script_for_shell_bleed` 返回非空时阻断执行，向 LLM 返回错误要求修改。  
**工作量**：小（5-10行，在 `tool_runner.rs` 的 `shell_exec` handler）  
**关键文件**：`crates/openfang-runtime/src/tool_runner.rs`

#### Gap-3：LLM 输出中 Secret 扫描缺失（S4.2）
**风险**：Agent 可能把 API key 打印在回复中，通过 channel 发送给最终用户或写入日志。  
**修复方向**：在 `AgentLoopResult.response` 返回前，用正则扫描常见 Secret 格式并替换为 `[REDACTED]`。  
**工作量**：小（30-50行，在 `agent_loop.rs`）  
**关键文件**：`crates/openfang-runtime/src/agent_loop.rs`

```rust
// 在 return 前插入
let response = redact_secrets(result.response);
fn redact_secrets(text: String) -> String {
    // 匹配 sk-ant-..., sk-..., ghp_..., gsk_... 等模式
}
```

---

### 🟡 中优先级 Gap（安全加固建议）

#### Gap-4：技能依赖包无完整性校验（S8.3）
**风险**：用户运行 `pip install playwright`，如果 PyPI 被供应链攻击，恶意包可获取 Agent 权限。  
**修复方向**：技能 manifest 中允许声明 `pip_hashes`，安装时用 `pip install --require-hashes`。  
**工作量**：中等（修改 `SkillRequirements` 结构 + `installer.rs`）

#### Gap-5：Agent 内存隔离依赖命名约定（S5.4）
**风险**：有 DB 访问权限的进程可直接读写其他 Agent 的 `agent:{id}` scope 数据。  
**修复方向**：对敏感 Agent 数据的 DB 行加密（AES-256-GCM，密钥来自 AgentId 派生）。  
**工作量**：较大（需修改 `StructuredStore`）

#### Gap-6：运行时能力无法细粒度撤销（S1.4）
**风险**：Agent 在运行时获取了某个能力后，只能 revoke_all 或等待终止，无法仅撤销某一个。  
**修复方向**：在 `CapabilityManager` 中添加 `revoke(agent_id, capability)` 方法。  
**工作量**：小（15-20行）  
**关键文件**：`crates/openfang-kernel/src/capabilities.rs`

#### Gap-7：审计日志无外发能力（S6.4）
**风险**：本地 SQLite 是单点，无法接入 SIEM 系统，无法满足企业合规要求。  
**修复方向**：在 `AuditLog::record` 中添加可选 Webhook 回调（异步发送到 Splunk/Elastic 等）。  
**工作量**：中等

---

### 🟢 低优先级 Gap（长期路线图）

#### Gap-8：多租户隔离（S9.1）
**风险**：多用户共享同一 kernel 实例，用户 A 的 Admin 操作可影响用户 B 的 Agent。  
**修复方向**：每个租户独立 `OpenFangKernel` 实例，Axum 路由按 tenant_id 分发。  
**工作量**：大（架构级改动）

#### Gap-9：对抗性输入检测（S9.4）
**风险**：Jailbreak 攻击可绕过 Agent 的安全配置，执行超出授权的操作。  
**修复方向**：在 `prompt_builder.rs` 中接入 Llama Guard 或 OpenAI Moderation API 进行输入检测。  
**工作量**：中等（需外部服务调用）

#### Gap-10：TEE 支持（S9.3）
**风险**：宿主机 OS 被攻陷时，Agent 的密钥和执行上下文无保护。  
**修复方向**：长期路线图，考虑 Intel TDX 或 AMD SEV 支持。  
**工作量**：极大（平台依赖）

---

## 五、安全能力评分总结

| 类别 | 要求数 | ✅ 已达成 | ⚠️ 部分 | ❌ Gap | 完成度 |
|------|--------|---------|--------|--------|--------|
| S1. 访问控制 | 4 | 3 | 1 | 0 | 87.5% |
| S2. 提示注入防护 | 3 | 1 | 2 | 0 | 50% |
| S3. 工具执行控制 | 4 | 4 | 0 | 0 | 100% |
| S4. 数据流安全 | 6 | 4 | 0 | 2 | 66.7% |
| S5. 运行时隔离 | 5 | 3 | 2 | 0 | 70% |
| S6. 审计可观测性 | 5 | 3 | 1 | 1 | 70% |
| S7. 网络协议安全 | 5 | 4 | 0 | 1 | 80% |
| S8. 供应链安全 | 4 | 2 | 1 | 1 | 62.5% |
| S9. 高级防护 | 4 | 0 | 1 | 3 | 12.5% |
| **总计** | **40** | **24** | **8** | **8** | **70.0%** |

**量化评价**：在40项行业安全要求中，OpenFang 完整达成 24 项（60%），部分达成 8 项（20%），有 8 个明确 Gap（20%）。

> **修订记录（2026-08-07）**：初版将 S4.6（Shell Bleed）评为 ⚠️ 部分达成，依据是函数文档注释里的 "warnings are prepended to the tool result"。后续在 [prompt-injection.md](prompt-injection.md) 中逐一核实调用点，发现 `scan_script_for_shell_bleed()` 在整个代码库中零调用（仅有模块声明和自身单元测试），实为死代码，故下调为 ❌ Gap。S4 完成度由 75% 修正为 66.7%，总计由 71.3% 修正为 70.0%。
>
> 方法论教训：只读函数定义和文档注释会高估防护能力，必须 grep 实际调用点。

---

## 六、与同类产品安全对比（README 数据验证）

README 声称 OpenFang 有"16 项独立安全系统"，本文档代码审查确认：

| 安全系统 | 代码位置 | 是否实现 |
|---------|---------|---------|
| Capability-based permissions | `capability.rs` + `capabilities.rs` | ✅ |
| RBAC multi-user | `auth.rs` | ✅ |
| Privilege escalation prevention | `kernel_handle.rs::spawn_agent_checked` | ✅ |
| API authentication | `server.rs` + `session_auth.rs` | ✅ |
| Path traversal protection | `subprocess_sandbox.rs::safe_resolve_*` | ✅ |
| SSRF protection | `web_fetch.rs` | ✅ |
| Image validation | `media.rs` whitelist | ✅ |
| Prompt injection scanning | `verify.rs` | ✅ |
| Ed25519 signed manifests | `manifest_signing.rs` | ✅ |
| HMAC-SHA256 wire protocol | `peer.rs` | ✅ |
| Secret zeroization | `Zeroizing<String>` 全局 | ✅ |
| WASM dual metering | `sandbox.rs` | ✅ |
| Subprocess sandbox | `subprocess_sandbox.rs` | ✅ |
| Taint tracking | `taint.rs` | ✅ |
| GCRA rate limiter | `rate_limiter.rs` | ✅ |
| Merkle hash chain audit | `audit.rs` | ✅ |

**结论**：16 项全部有对应代码实现，README 声明属实。

---

## 七、关键结论

1. **工具执行控制是最强项**：deny-wins 策略、深度感知、人工审批三层叠加，完整实现 OWASP LLM08 要求。

2. **审计链是架构亮点**：Merkle 哈希链 + SQLite 持久化 + 启动验证，比同类产品（仅文件日志）高一个档次。

3. **提示注入防护是最大短板**：taint 追踪框架设计正确，但实际覆盖不完整——LLM 输出本身未被标记污点，间接注入（通过 web_fetch 结果）只在 shell_exec sink 点检测，不覆盖其他工具。

4. **OFP 传输加密是唯一生产级 Gap**：在多节点部署场景下，明文 TCP 传输是实际安全风险，需优先修复。

5. **高级防护（TEE、多租户、对抗检测）处于空白**：这与 v0.6.9 pre-1.0 的定位一致，属于长期路线图而非紧急缺陷。
