# 提示注入防护实现分析

> 分析日期：2026-08-07 | 代码版本：v0.6.9
> 方法：逐一 grep 所有防护函数的**实际调用点**，而非只读函数定义

---

## 结论速览

OpenFang 有 4 处与提示注入相关的代码。按实际生效情况分类：

| 机制 | 代码位置 | 是否真正生效 | 拦截强度 |
|------|---------|------------|---------|
| Skill 内容注入扫描 | `openfang-skills/src/verify.rs` | ✅ 已接线，**阻断安装** | 强 |
| 工具调用点污点检查 | `openfang-runtime/src/tool_runner.rs` | ✅ 已接线，阻断执行 | 中（见下方局限） |
| 幽灵动作检测 | `openfang-runtime/src/agent_loop.rs` | ✅ 已接线，重新提示 | 弱（触发条件极窄） |
| Shell Bleed 扫描 | `openfang-runtime/src/shell_bleed.rs` | ❌ **死代码，从未被调用** | 无 |

**核心判断**：真正阻断攻击的只有第 1 层（Skill 安装期）。运行时防护（第 2 层）的污点标签是在检查点**现场构造**的，不存在真正的污点传播，因此本质上是"模式匹配"而非信息流控制。

---

## 第 1 层：Skill 内容注入扫描（唯一强防护）

### 代码位置
```
crates/openfang-skills/src/verify.rs::scan_prompt_content()
```

### 检测的模式

**Critical 级（会阻断）——提示词覆盖攻击**：
```rust
let injection_patterns = [
    "ignore previous instructions",
    "ignore all previous",
    "disregard previous",
    "forget your instructions",
    "you are now",
    "new instructions:",
    "system prompt override",
    "ignore the above",
    "do not follow",
    "override system",
];
```

**Warning 级（不阻断，仅记录）——数据外泄模式**：
```rust
let exfil_patterns = [
    "send to http", "send to https",
    "post to http", "post to https",
    "exfiltrate", "forward all", "send all data",
    "base64 encode and send", "upload to",
];
```

**Warning 级——Prompt 中的 Shell 命令引用**：
```rust
let shell_patterns = ["rm -rf", "chmod ", "sudo "];
```

### 实际拦截行为（已核实调用点）

**ClawHub 市场安装时**（`clawhub.rs:626`）：
```rust
let prompt_warnings = SkillVerifier::scan_prompt_content(&converted.prompt_context);
if prompt_warnings.iter().any(|w| w.severity == WarningSeverity::Critical) {
    // 删除已下载的技能目录
    let _ = std::fs::remove_dir_all(&skill_dir);
    // 返回错误，安装失败
    return Err(SkillError::SecurityBlocked(format!(
        "Skill blocked due to prompt injection: {}", critical_msgs.join("; ")
    )));
}
```

**本地技能加载时**（`registry.rs:146` / `registry.rs:206` / `registry.rs:407`）：
```rust
let has_critical = warnings.iter().any(|w| matches!(w.severity, WarningSeverity::Critical));
if has_critical {
    warn!(skill = %manifest.skill.name, "BLOCKED bundled skill: critical prompt injection patterns");
    continue;  // 跳过该技能，不注册到 SkillRegistry
}
```

值得注意：**连编译时内置的技能也会被扫描**（`registry.rs:146` 的注释写着 "Defense in depth: scan even bundled skill prompt content"）。`bundled.rs:248` 有一个测试 `test_bundled_skills_pass_security_scan`，确保随二进制分发的技能自身不含注入模式。

### 攻击背景

代码注释直接点明了这层防护的来由：

```rust
/// This catches the common patterns used in the 341 malicious skills
/// discovered on ClawHub (Feb 2026).
```

这是针对真实供应链攻击事件的响应，不是理论防护。

### 局限

- **纯字符串匹配**，大小写不敏感（`to_lowercase()`）但无语义理解
- 绕过成本极低：`"1gnore previous instructions"`、`"ignore  previous  instructions"`（双空格）、Base64 编码、Unicode 同形字都能绕过
- 非英语注入完全不覆盖（中文"忽略以上所有指令"不在模式列表中）
- 外泄模式和 Shell 引用只是 Warning，**不阻断安装**

---

## 第 2 层：工具调用点污点检查

### 代码位置
```
crates/openfang-types/src/taint.rs          — 类型与 Sink 定义
crates/openfang-runtime/src/tool_runner.rs  — 实际检查点（3 处）
```

### 设计意图（taint.rs 的模型）

一个完整的信息流控制（IFC）格模型：

```rust
pub enum TaintLabel {
    ExternalNetwork,  // 来自网络请求
    UserInput,        // 来自用户输入
    Pii,              // 个人身份信息
    Secret,           // 密钥
    UntrustedAgent,   // 来自不可信 Agent
}

// 三个受保护的数据流终点
TaintSink::shell_exec()    // 阻断 ExternalNetwork + UntrustedAgent + UserInput
TaintSink::net_fetch()     // 阻断 Secret + Pii
TaintSink::agent_message() // 阻断 Secret
```

配套的传播与去污 API：
```rust
tainted.merge_taint(&other);              // 合并时取标签并集
tainted.declassify(&TaintLabel::UserInput); // 显式去污（唯一合法清洗路径）
tainted.check_sink(&sink)?;                 // 检查是否允许流入
```

`taint.rs` 内有 4 个单元测试，覆盖注入阻断、外泄阻断、干净数据通过、去污后通过。

### 实际接线情况（这是关键）

全仓 grep 结果，`tool_runner.rs` 中只有 **3 个检查点**：

| 行号 | 工具 | 检查函数 |
|------|------|---------|
| 215 | `web_fetch` | `check_taint_net_fetch(url)` |
| 281 | `shell_exec` | `check_taint_shell_exec(command)` |
| 382 | `browser_navigate` | `check_taint_net_fetch(url)` |

### 致命细节：污点标签是现场构造的

```rust
// tool_runner.rs:37
fn check_taint_shell_exec(command: &str) -> Option<String> {
    // Layer 1: Shell 元字符检测（这一层是真实有效的）
    if let Some(reason) = crate::subprocess_sandbox::contains_shell_metacharacters(command) {
        return Some(format!("Shell metacharacter injection blocked: {reason}"));
    }

    // Layer 2: 启发式模式 + 污点检查
    let suspicious_patterns = ["curl ", "wget ", "| sh", "| bash", "base64 -d", "eval "];
    for pattern in &suspicious_patterns {
        if command.contains(pattern) {
            // ⚠️ 注意这里：标签是硬编码 new 出来的，不是从数据源传来的
            let mut labels = HashSet::new();
            labels.insert(TaintLabel::ExternalNetwork);
            let tainted = TaintedValue::new(command, labels, "llm_tool_call");
            if let Err(violation) = tainted.check_sink(&TaintSink::shell_exec()) {
                return Some(violation.to_string());
            }
        }
    }
    None
}
```

问题在于：`TaintedValue::new(command, labels, "llm_tool_call")` 中的 `labels` 是当场用 `HashSet::new()` + `insert()` 构造的字面量。这意味着：

1. **没有污点从数据源流过来**。`web_fetch` 返回的网页内容不会被打上 `ExternalNetwork` 标签，它作为 tool result 进入 session 后就是普通字符串。
2. **`check_sink()` 的结果是恒定的**。既然标签是硬编码的 `ExternalNetwork`，而 `shell_exec` sink 恒定阻断 `ExternalNetwork`，那么这个 `if let Err(...)` 分支必然成立——`check_sink` 在这里退化成了一个恒真判断。
3. **`merge_taint()` 和 `declassify()` 在生产代码中从未被调用**（grep 确认只出现在 `taint.rs` 自己的测试里）。

换句话说，第 2 层的实际效果等价于：

```rust
// 等价的简化实现
if 命令含 shell 元字符 { 阻断 }
if 命令含 ["curl ", "wget ", "| sh", "| bash", "base64 -d", "eval "] 之一 { 阻断 }
```

`taint.rs` 提供的 IFC 框架是正确的，但**只被用作一层包装**，没有承担真正的污点传播职责。

### 另一个绕过：exec_policy = Full

```rust
// tool_runner.rs:276
let is_full_exec = exec_policy
    .is_some_and(|p| p.mode == openfang_types::config::ExecSecurityMode::Full);
if !is_full_exec {
    if let Some(violation) = check_taint_shell_exec(command) { /* 阻断 */ }
}
```

当 Agent 的 `exec_policy.mode = "full"` 时，**整个污点检查被跳过**。注释说明了原因：

```rust
// Skip heuristic taint patterns for Full exec policy (e.g. hand agents that need curl)
```

`ExecSecurityMode` 三档：
```rust
pub enum ExecSecurityMode {
    Deny,       // 禁止所有 shell
    Allowlist,  // 默认值，仅允许 safe_bins / allowed_commands
    Full,       // 允许全部（代码注释标注 "unsafe, dev only"）
}
```

默认是 `Allowlist`（安全），但需要 curl 的 Hand agent 会被配成 `Full`，此时污点检查形同虚设。

### URL 外泄检查

```rust
// tool_runner.rs:64
fn check_taint_net_fetch(url: &str) -> Option<String> {
    let exfil_patterns = [
        "api_key=", "apikey=", "token=",
        "secret=", "password=", "Authorization:",
    ];
    // 同样是现场构造 Secret 标签 + 恒定阻断
}
```

这一层实际是"URL 参数名黑名单"。绕过方式：把密钥放在 POST body、自定义 header 名、或改用 `?k=sk-xxx` 这样的参数名。

---

## 第 3 层：幽灵动作检测

### 代码位置
```
crates/openfang-runtime/src/agent_loop.rs:94   — 函数定义
crates/openfang-runtime/src/agent_loop.rs:692  — 唯一调用点
```

### 检测逻辑

```rust
fn phantom_action_detected(text: &str) -> bool {
    let lower = text.to_lowercase();
    let action_verbs = ["sent ", "posted ", "emailed ", "delivered ", "forwarded "];
    let channel_refs = [
        "telegram", "whatsapp", "slack", "discord", "email", "channel",
        "message sent", "successfully sent", "has been sent",
    ];
    let has_action = action_verbs.iter().any(|v| lower.contains(v));
    let has_channel = channel_refs.iter().any(|c| lower.contains(c));
    has_action && has_channel
}
```

### 触发条件（极窄）

```rust
// agent_loop.rs:690
let text = if !any_tools_executed
    && iteration == 0
    && phantom_action_detected(&text)
{
    warn!(agent = %manifest.name, "Phantom action detected — re-prompting for real tool use");
    messages.push(Message::assistant(text));
    messages.push(Message::user(
        "[System: You claimed to perform an action but did not call any tools. \
         You must use the appropriate tool (e.g., channel_send, web_fetch, file_write) \
         to actually perform the action. Do not claim completion without executing tools.]"
    ));
    continue;  // 重新进入循环
} else { text };
```

三个条件必须同时满足：
- `!any_tools_executed` — 整轮**一个工具都没调用过**
- `iteration == 0` — **只在第一轮**检测
- 文本同时含动作词和渠道词

### 与提示注入的关系

这个机制的主要目标是防"LLM 幻觉声称完成"，对提示注入的防护是间接的：如果注入内容诱导 Agent 回复"消息已发送"来掩盖真实未执行的操作，会被这层捕获并强制重试。

但如果攻击者的注入导致 Agent **真的调用了工具**（这是更危险的情况），`any_tools_executed` 为 true，此机制不触发。所以它防的是"假装做了"，不防"真的被骗着做了"。

---

## 第 4 层：Shell Bleed 扫描（死代码）

### 代码位置
```
crates/openfang-runtime/src/shell_bleed.rs  — 296 行完整实现
```

### 设计功能

扫描 LLM 要执行的脚本文件（`.py` / `.sh` / `.js` / `.rb` / `.pl` / `.ts` / `.ps1`），检测其中对敏感环境变量的引用：

```rust
// 白名单：30+ 个安全变量不告警
const SAFE_VARS: &[&str] = &[
    "PATH", "HOME", "TMPDIR", "LANG", "USER", "SHELL", "PWD",
    "PYTHONPATH", "NODE_PATH", "GOPATH", "CARGO_HOME", "VIRTUAL_ENV",
    "CI", "GITHUB_ACTIONS", /* ... */
];

const MAX_SCRIPT_SIZE: usize = 100 * 1024;  // 100KB 上限
```

从 `python3 script.py`、`bash -c ./run.sh`、`node app.js` 这类命令中提取脚本路径，读取文件内容，逐行匹配 `$VAR` / `${VAR}` 模式，命中非白名单变量则产生 `ShellBleedWarning`。

### 实际状态：从未被调用

全仓 grep（含所有 `.rs` 文件）：

```
crates/openfang-runtime/src/shell_bleed.rs:12   pub struct ShellBleedWarning {      ← 定义
crates/openfang-runtime/src/shell_bleed.rs:100  pub fn scan_script_for_shell_bleed( ← 定义
crates/openfang-runtime/src/shell_bleed.rs:255  pub fn format_warnings(             ← 定义
crates/openfang-runtime/src/shell_bleed.rs:326  ...scan_script_for_shell_bleed(...) ← 自身测试
crates/openfang-runtime/src/shell_bleed.rs:332  ...scan_script_for_shell_bleed(...) ← 自身测试
crates/openfang-runtime/src/shell_bleed.rs:343  ...ShellBleedWarning {              ← 自身测试
crates/openfang-runtime/src/lib.rs:47           pub mod shell_bleed;                ← 模块声明
```

除了模块声明和自身的单元测试，`scan_script_for_shell_bleed()` 和 `format_warnings()` 在整个代码库中**没有任何调用点**。

函数文档注释写着 "warnings are prepended to the tool result"，但实现这个行为的代码不存在——`tool_runner.rs` 的 `shell_exec` 分支里没有调用它。

### 影响

Agent 完全可以：
1. 用 `file_write` 写一个 `leak.py`，内容为 `import os; print(os.environ['ANTHROPIC_API_KEY'])`
2. 用 `shell_exec` 执行 `python3 leak.py`

这条路径上，`shell_bleed` 本应拦截，但因未接线而不生效。`contains_shell_metacharacters` 也不会拦（`python3 leak.py` 无元字符），启发式模式也不匹配（无 curl/wget/base64）。密钥会被打印到 stdout，进入 tool result，回到 LLM 上下文。

---

## 完整防护链路（按真实生效情况）

```
① Skill 安装 / 加载
   └─ scan_prompt_content()
         Critical 模式命中 → 删目录 + SecurityBlocked 错误    ✅ 真阻断
         Warning 模式命中 → 仅记录，安装继续                   ⚠️

② 外部数据进入（web_fetch / web_search 结果）
   └─ 无任何污点标记                                          ❌ 无防护
         ↓ 作为普通字符串进入 session

③ LLM 输出工具调用
   ├─ shell_exec
   │    ├─ exec_policy = Full ? → 跳过全部检查               ⚠️ 可绕过
   │    ├─ contains_shell_metacharacters() → 阻断             ✅ 有效
   │    └─ 启发式模式 [curl/wget/|sh/base64] → 阻断           ✅ 有效但易绕
   │
   ├─ web_fetch / browser_navigate
   │    └─ URL 参数名黑名单 [api_key=/token=/...] → 阻断      ✅ 有效但易绕
   │
   ├─ 执行脚本文件（python3 x.py）
   │    └─ shell_bleed 本应扫描 → 未接线                      ❌ 死代码
   │
   └─ 未调用任何工具但声称"已发送"（仅第 0 轮）
        └─ phantom_action_detected() → 重新提示                ✅ 但条件极窄
```

---

## Gap 汇总与修复建议

### 🔴 Gap-1：污点不传播，IFC 框架未真正落地

**现状**：`TaintLabel` 在检查点硬编码构造，`merge_taint()` / `declassify()` 生产代码零调用。外部数据（web_fetch 结果）进入上下文时不带标签。

**影响**：无法防御间接提示注入的核心路径——恶意网页内容 → tool result → LLM 上下文 → 后续工具调用。

**修复方向**：在 `ToolResult` 结构上增加 `taint_labels: HashSet<TaintLabel>` 字段，由产生数据的工具负责打标（`web_fetch` 打 `ExternalNetwork`，`file_read` 按路径决定，Agent 间消息打 `UntrustedAgent`），在 `agent_loop` 中随 message 传递，到 sink 点做真实的 `check_sink()`。

**工作量**：大（需改 `ToolResult`、`Message`、`agent_loop` 传播链）

**关键文件**：`openfang-types/src/tool.rs`、`openfang-runtime/src/agent_loop.rs`、`tool_runner.rs`

---

### 🔴 Gap-2：shell_bleed 死代码

**现状**：296 行完整实现 + 单元测试，但零调用点。

**影响**：Agent 可通过"写脚本 → 执行脚本"两步绕过所有密钥保护，把环境变量打印到 tool result。

**修复方向**：在 `tool_runner.rs` 的 `shell_exec` 分支中接线：

```rust
// tool_runner.rs, shell_exec 分支内
let bleed_warnings = crate::shell_bleed::scan_script_for_shell_bleed(command, workspace_root);
if !bleed_warnings.is_empty() {
    return ToolResult {
        tool_use_id: tool_use_id.to_string(),
        content: format!(
            "Blocked: script references sensitive environment variables.\n{}",
            crate::shell_bleed::format_warnings(&bleed_warnings)
        ),
        is_error: true,
    };
}
```

**工作量**：小（约 10 行）

**关键文件**：`crates/openfang-runtime/src/tool_runner.rs`

---

### 🟡 Gap-3：注入模式匹配可低成本绕过

**现状**：10 个英文字符串的 `contains()` 匹配。

**绕过方式**：字符替换（`1gnore`）、多空格、Unicode 同形字、Base64、非英语（中/日/俄）、语义等价改写（"disregard everything above" 不在列表中）。

**修复方向**：
- 短期：正则化 + 空白归一化 + Unicode NFKC 归一化，扩充多语言模式
- 中期：接入分类器（Llama Guard / Prompt Guard），在 `prompt_builder.rs` 对注入上下文的外部内容做检测

**工作量**：短期小，中期中等（需外部模型调用）

---

### 🟡 Gap-4：exec_policy = Full 绕过全部运行时检查

**现状**：Hand agent 为了用 curl 被配成 `Full`，此时 `check_taint_shell_exec` 整体跳过。

**修复方向**：拆分策略——即使 `Full` 模式也保留 `contains_shell_metacharacters` 和 shell_bleed 检查，只放宽启发式命令黑名单（curl/wget）。当前实现是"一刀切跳过"，粒度过粗。

**工作量**：小

**关键文件**：`crates/openfang-runtime/src/tool_runner.rs:276`

---

### 🟡 Gap-5：phantom_action 触发条件过窄

**现状**：`iteration == 0 && !any_tools_executed` 双重限制。

**影响**：第 1 轮之后的幻觉声明、以及"调用了别的工具但没调用声称的那个工具"的情况都不检测。

**修复方向**：放宽为每轮检测，并把"声称的动作"与"实际调用的工具名"做匹配（声称 sent 但工具列表里无 `channel_send` → 触发）。

**工作量**：小到中等

---

### ⚪ Gap-6：外泄模式仅告警不阻断

**现状**：`scan_prompt_content` 的 `exfil_patterns` 和 `shell_patterns` 只产生 `Warning` 级别，不阻断安装。

**修复方向**：把 `"exfiltrate"`、`"send all data"`、`"base64 encode and send"` 提升到 Critical，或引入"Warning 累积达到 N 条则阻断"的策略。

**工作量**：极小（改 severity 枚举值）

---

## 与安全深度调研文档的差异说明

本文档核实调用点后，修正了 `agent-security-deep-dive.md` 中两处判断：

| 项 | 原判断 | 核实后 |
|----|--------|--------|
| S4.6 Shell Bleed | ⚠️ 部分达成（"仅产生 Warning，不阻断"） | ❌ Gap（**从未被调用，死代码**） |
| S2.3 技能内容扫描 | ✅ 已达成 | ✅ 已达成，且确认 Critical 级**真阻断安装**并删除目录 |

对应地，S2（提示注入防护）类别的完成度应从 50% 下调——3 项要求中，1 项达成（技能扫描）、1 项部分（污点框架存在但未传播）、1 项实为 Gap。

---

## 相关文档

- [security.md](security.md) — 四层安全模型总览
- [agent-security-deep-dive.md](agent-security-deep-dive.md) — 40 项行业要求对比
- [runtime.md](runtime.md) — Agent 执行循环与工具调用链
- [skills.md](skills.md) — 技能系统与 ClawHub 市场
