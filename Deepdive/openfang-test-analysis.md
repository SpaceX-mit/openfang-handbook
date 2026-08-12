# OpenFang Test 分析

> 回答 goal §86
> 数据来源：静态 grep，双方法交叉验证

---

## 0. 核心发现

**2,698 个测试中，只有 2 个断言检验"未授权操作被拒绝"。**

两个都在 `api_integration_test.rs`（第 1040、1058 行），都是 API bearer token 的 401。
`CapabilityManager::check()`、`tool_policy::resolve_tool_access`、
`shell_bleed` 这三个失效模块**零授权断言**。

这是 security.md §3 的"三个安全模块实现完整 + 单元测试全绿 + 零调用"
能通过 CI 的直接原因——**测试从未问过"该被拒的调用是否真的被拒了"**。

---

## 1. 测试规模分布

| Crate | 测试数 | 占比 | 源文件数 | 评价 |
|-------|-------|------|---------|------|
| `openfang-runtime` | 993 | 36.8% | 51 | ✅ 最重的执行层，覆盖合理 |
| `openfang-channels` | 512 | 19.0% | 46 | ✅ 40+ 渠道各自有测试 |
| `openfang-types` | 386 | 14.3% | 20 | ⚠️ 纯数据类型，占比偏高 |
| `openfang-kernel` | 305 | 11.3% | 23 | ⚠️ 9415 行 kernel.rs 仅此覆盖 |
| `openfang-api` | 146 | 5.4% | 13 | ⚠️ 12975 行 routes.rs |
| `openfang-skills` | 82 | 3.0% | 10 | — |
| `openfang-hands` | 54 | 2.0% | 3 | — |
| `openfang-extensions` | 54 | 2.0% | 8 | — |
| `openfang-migrate` | 50 | 1.9% | 3 | — |
| `openfang-cli` | 50 | 1.9% | 10 | ⚠️ 7478 行 main.rs |
| `openfang-memory` | 40 | 1.5% | 10 | 🔴 **13 表 + 迁移 + 并发，仅 40 个** |
| `openfang-wire` | 26 | 1.0% | 4 | 🔴 **HMAC + nonce 防重放，仅 26 个** |
| **合计** | **2,698** | 100% | 259 | — |

### 分布倒挂

覆盖密度与风险不匹配：

| | `openfang-types` | `openfang-memory` |
|---|---|---|
| 测试数 | 386 | 40 |
| 内容 | 纯数据结构 + serde | 13 张表、schema V8 迁移、WAL 并发、MessagePack |
| 崩溃后果 | 编译期就能发现大部分问题 | 数据丢失 / 静默分叉 |

`openfang-memory` 的测试数只有 `openfang-types` 的 1/10，
但它承载全部持久化状态。`openfang-wire` 26 个测试要覆盖
HMAC 签名、常量时间比较、NonceTracker 5 分钟窗口、TCP 握手——
这些是安全关键路径。

---

## 2. 集成测试清单（13 个文件，4 个 crate）

| 文件 | 覆盖 |
|------|------|
| `api/tests/api_integration_test.rs` | REST 端点 + **唯一的 2 个 401 断言** |
| `api/tests/daemon_lifecycle_test.rs` | daemon 启停 |
| `api/tests/load_test.rs` | 负载 |
| `api/tests/skill_config_api_test.rs` | 技能配置 API |
| `channels/tests/bridge_integration_test.rs` | 渠道桥接 |
| `kernel/tests/integration_test.rs` | 内核集成 |
| `kernel/tests/multi_agent_test.rs` | 多 Agent 协作 |
| `kernel/tests/wasm_agent_integration_test.rs` | **WASM Agent 真实执行 + fuel 耗尽** |
| `kernel/tests/workflow_integration_test.rs` | 工作流 |
| `migrate/tests/provider_json5_agents.rs` | OpenClaw 迁移 |
| `migrate/tests/provider_json5_default_model.rs` | 同上 |
| `migrate/tests/provider_json5_provider_catalog.rs` | 同上 |
| `migrate/tests/provider_legacy_yaml.rs` | 同上 |

**观察 1**：`openfang-migrate` 占 4/13 个集成测试文件，与 `openfang-kernel` 持平。
迁移工具（4606 行，一次性代码）与内核获得同等集成测试投入。

**观察 2**：`wasm_agent_integration_test.rs` 有真实断言：
```rust
assert_eq!(result.response, "hello from wasm");   // :167
assert_eq!(result.iterations, 1);                  // :168
// test_wasm_agent_fuel_exhaustion  :202
```
**这证明 WASM Agent 路径是活代码**（我在 security.md §5.3 初稿中
误判为"完全未使用"，已在该文件内更正）。WASM **Skill** 未实现，
WASM **Agent** 有测试。

**观察 3**：`openfang-runtime`（993 个测试、51 个文件、含 agent_loop）
**没有 tests/ 目录**——全部是 crate 内单元测试，无集成测试。
`tool_runner.rs` 5014 行的工具执行路径没有端到端测试。

---

## 3. 授权机制的测试覆盖矩阵

| 机制 | 单元测试 | 集成/E2E 断言 | 实际生效 |
|------|---------|--------------|---------|
| API bearer token | ✅ | ✅ **2 个 401** | ✅ |
| `CapabilityManager::check()` | ✅ 2 个 | ❌ **零** | ❌ 零调用 |
| `tool_policy::resolve_tool_access` | ✅ 11 个 | ❌ **零** | ❌ 零调用 |
| `filter_tools_by_depth`（子 Agent 剥离） | ✅ | ❌ **零** | ❌ 零调用 |
| `shell_bleed` 脚本变量扫描 | ✅ | ❌ **零** | ❌ 零调用 |
| `validate_capability_inheritance` | ✅ | ❌ 零 | ✅ 生效 |
| RBAC 四级角色 | ✅ | ❌ 零 | ✅ 生效 |
| `exec_policy` 三档 | ✅ | ❌ 零 | ✅ 生效 |
| 工具列表预过滤（fail-open） | ⚠️ 未见 | ❌ 零 | ✅ 生效但 fail-open |
| 审批门控 | ✅ | ❌ 零 | ✅ 生效 |

**10 个授权机制，只有 1 个有集成断言。**

---

## 4. 单元测试为什么发现不了

以 `tool_policy.rs` 的 11 个测试为例，它们的形态是：

```rust
#[test]
fn test_deny_wins() {
    let policy = ToolPolicy { agent_rules: vec![
        ToolPolicyRule { pattern: "shell_*".into(), effect: Allow },
        ToolPolicyRule { pattern: "shell_exec".into(), effect: Deny },
    ], ..Default::default() };
    let result = resolve_tool_access("shell_exec", &policy, 0);
    assert!(matches!(result, ToolAccessResult::Denied { .. }));
}
```

这个测试**完全正确**——它验证了"给定 policy，函数返回 Denied"。

它没有验证的是：**执行 `shell_exec` 时这个函数会被调用**。
答案是不会（零调用点）。

> **单元测试能证明"函数被调用时行为正确"，
> 不能证明"函数在该被调用的地方确实被调用了"。
> 对安全功能，后者才是要断言的东西。**

---

## 5. 三个能捕获缺陷的测试（各约 10 行）

### 5.1 捕获 tool_policy 死代码

```rust
#[tokio::test]
async fn denied_tool_is_actually_rejected_at_runtime() {
    let kernel = test_kernel();
    // manifest 声明只允许 web_search
    let id = kernel.spawn_agent(manifest_with_tools(&["web_search"])).unwrap();
    // 直接请求未声明的 shell_exec
    let result = execute_tool(&kernel, id, "shell_exec", json!({"command": "echo hi"})).await;
    assert!(result.is_error, "未声明的工具必须被拒绝");
    assert!(result.content.contains("denied") || result.content.contains("not allowed"));
}
```

**当前会失败** —— 预过滤只影响发给 LLM 的工具列表，
`tool_runner::execute()` 的 match 分支不校验声明。

### 5.2 捕获 fail-open

```rust
#[tokio::test]
async fn empty_tool_list_denies_all_not_allows_all() {
    let kernel = test_kernel();
    let id = kernel.spawn_agent(manifest_with_tools(&[])).unwrap();  // 空列表
    let tools = kernel.available_tools(id);
    assert!(tools.is_empty(), "空声明必须是 deny-all，当前是 fail-open 放行 53 个");
}
```

**当前会失败** —— `tools_unrestricted = declared_tools.is_empty()` → true。

### 5.3 捕获子 Agent 深度限制失效

```rust
#[tokio::test]
async fn subagent_cannot_call_cron_create() {
    let kernel = test_kernel();
    let parent = kernel.spawn_agent(manifest_with_tools(&["agent_spawn", "cron_create"])).unwrap();
    let child = spawn_child_of(&kernel, parent).await;
    let result = execute_tool(&kernel, child, "cron_create", json!({...})).await;
    assert!(result.is_error, "SUBAGENT_DENY_ALWAYS 应阻止子 Agent 创建 cron");
}
```

**当前会失败** —— `filter_tools_by_depth` 零调用。

---

## 6. 测试类型全景

| 类型 | 有？ | 数量 | 缺口 |
|------|------|------|------|
| 单元测试 | ✅ | 2,698（196 个 `#[cfg(test)]` 模块） | 数量足，但见 §4 的质量问题 |
| 集成测试 | ✅ | 13 个文件 / 4 个 crate | runtime 无集成测试 |
| E2E | ⚠️ | `daemon_lifecycle_test` + `wasm_agent_integration_test` | 无完整 Agent 任务链路 |
| **安全测试** | ❌ | **无专门套件** | 拒绝路径零覆盖 |
| Benchmark | ❌ | 无 | goal §87 要求，需实机 |
| Agent 评估 | ❌ | 无 | 无质量回归基线 |
| 模糊测试 | ❌ | 无 | manifest / wire 协议解析可 fuzz |

---

## 7. 对 Bianbu 的建议

1. **安全功能强制 E2E 拒绝测试**：每个 capability 变体至少一个
   "未授权 → 确实被拒"的断言。不接受只有单元测试的安全 PR。
2. **CI 加死代码检测**：`cargo machete` + pub 函数零调用检测
   （`dead_code` lint 不标记 pub，见 code-quality.md §6）。
3. **覆盖率按风险加权**：`memoryd` / 认证层的测试密度应高于纯数据类型，
   不是反过来。
4. **衡量标准换成"安全控制是否在执行路径上"**，不是行覆盖率。
   本次调研靠逐个 grep 调用点发现问题，这个动作应该自动化成 CI 断言。

---

## 8. 未覆盖

- 未运行 `cargo test --workspace`，2,698 是静态计数而非实际通过数
  （README 声称 1,767+ 和 2,696+ 两个不同数字，量级与静态计数一致）
- 未做覆盖率测量（需 `cargo-llvm-cov`）
- 未做 benchmark（goal §87）

---

## 相关文档

- [openfang-code-quality.md](openfang-code-quality.md) §10 — CI 三个盲区的合并结论
- [openfang-security.md](openfang-security.md) §3 — 三个死代码模块的完整证据
- [openfang-limitations.md](openfang-limitations.md) L-11 — 单元测试虚假安全感
- [openfang-adr-lessons.md](openfang-adr-lessons.md) ADR-016 / L2
