# OpenFang LLM Wiki

深度技术研究文档，供 LLM 代理快速理解 OpenFang 代码库使用。

## 项目基本信息

| 属性 | 值 |
|------|-----|
| 语言 | Rust (edition 2021) |
| 版本 | 0.6.9 |
| Crate 数量 | 14 |
| 代码行数 | ~137K LOC（160K 含注释） |
| 测试数量 | 1,767+ |
| 最低 Rust | 1.75 |
| 默认端口 | 4200 |
| 配置文件 | `~/.openfang/config.toml` |
| 二进制产物 | `target/release/openfang.exe` |

## 文档索引

| 文件 | 内容 |
|------|------|
| [architecture.md](architecture.md) | 总体架构、Crate 依赖图、数据流 |
| [kernel.md](kernel.md) | Kernel 核心 — 子系统总览、生命周期管理 |
| [runtime.md](runtime.md) | Agent 运行时、执行循环、LLM 调用链 |
| [types.md](types.md) | 核心类型系统（openfang-types） |
| [memory.md](memory.md) | 内存子系统：会话、知识图、语义检索 |
| [api.md](api.md) | REST API 完整端点参考 |
| [llm-drivers.md](llm-drivers.md) | LLM 提供商驱动（10 家） |
| [wire-protocol.md](wire-protocol.md) | OFP 点对点协议 |
| [a2a.md](a2a.md) | Google A2A 跨框架协议 |
| [channels.md](channels.md) | 40+ 消息渠道适配器 |
| [skills.md](skills.md) | 技能（插件）系统 |
| [hands.md](hands.md) | Hands 自主能力包 |
| [extensions.md](extensions.md) | Extensions 一键集成系统 |
| [security.md](security.md) | 安全模型：能力、审计、污点追踪 |
| [config.md](config.md) | 配置文件完整字段参考 |

## 快速定位

**想改 LLM 驱动？** → [llm-drivers.md](llm-drivers.md) + `crates/openfang-runtime/src/drivers/`

**想加新 API 端点？** → [api.md](api.md) — 必须在 `server.rs` 注册路由 + `routes.rs` 实现

**想加新消息渠道？** → [channels.md](channels.md) + `crates/openfang-channels/`

**想理解 Agent 执行流？** → [runtime.md](runtime.md) → `agent_loop.rs`

**想加新能力检查？** → [security.md](security.md) → `crates/openfang-types/src/capability.rs`

**想修改配置字段？** → [config.md](config.md) — struct + `Default` impl + Serialize/Deserialize
