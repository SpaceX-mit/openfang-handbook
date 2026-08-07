# 技能系统（openfang-skills）

## 概述

技能（Skill）是可插拔的工具包，通过 TOML manifest + 可执行代码扩展 Agent 能力。

```
crates/openfang-skills/src/
├── lib.rs           ← 类型定义、SkillRuntime 枚举
├── registry.rs      ← SkillRegistry（已安装技能的运行时索引）
├── loader.rs        ← 技能执行器（Python/Node/Shell 调度）
├── installer.rs     ← 安装/卸载逻辑
├── bundled.rs       ← 编译时内置技能
├── clawhub.rs       ← ClawHub 市场客户端（981行）
├── marketplace.rs   ← 市场抽象
├── openclaw_compat.rs← OpenClaw 格式兼容
├── config_injection.rs← 技能配置注入
└── verify.rs        ← 技能完整性校验
```

---

## 技能运行时类型

```rust
pub enum SkillRuntime {
    Python,      // Python 3 子进程
    Node,        // Node.js 子进程（OpenClaw 兼容）
    Shell,       // Bash 脚本子进程
    Wasm,        // WASM 模块（wasmtime 沙箱，规划中）
    Builtin,     // 内置技能（编译进二进制）
    PromptOnly,  // 仅注入 Prompt 上下文，无可执行代码
}
```

---

## 技能 Manifest 格式（SKILL.toml）

```toml
[skill]
name = "database-query"
version = "1.0.0"
description = "Query databases using natural language"
author = "example"
license = "MIT"

[skill.runtime]
runtime_type = "python"
entry = "main.py"

[skill.requirements]
python_packages = ["psycopg2", "sqlalchemy"]
env_vars = [
    { name = "DATABASE_URL", description = "PostgreSQL connection string" }
]

[skill.tools.provided]
[[skill.tools.provided]]
name = "db_query"
description = "Execute a SQL query and return results"
input_schema = '''
{
    "type": "object",
    "properties": {
        "query": {"type": "string", "description": "SQL query"},
        "limit": {"type": "integer", "default": 100}
    },
    "required": ["query"]
}
'''

[skill.context]
# 注入 LLM 系统 prompt 的额外上下文（PromptOnly 模式使用）
system_prompt_append = """
You have access to a PostgreSQL database. Use the db_query tool to execute SQL.
"""
```

---

## SkillManifest 关键字段

```rust
pub struct SkillManifest {
    pub skill: SkillInfo,
    pub runtime: SkillRuntimeConfig,
    pub requirements: SkillRequirements,
    pub tools: SkillTools,        // 提供的工具列表
    pub context: Option<SkillContext>, // prompt 注入
}

pub struct SkillRequirements {
    pub python_packages: Vec<String>,
    pub npm_packages: Vec<String>,
    pub env_vars: Vec<EnvVarRequirement>,
    pub system_binaries: Vec<String>,
}
```

---

## 技能执行流程

```
Agent loop 收到 tool_call
  │
  ├─ 在 SkillRegistry 中查找工具名
  │    找到 → skill_manifest + skill_dir
  │
  ├─ loader::execute_skill_tool(manifest, dir, tool_name, input)
  │
  ├─ 按 runtime_type 派发：
  │    Python → 启动 `python main.py`，stdin 注入 JSON payload
  │    Node   → 启动 `node main.js`，stdin 注入 JSON payload
  │    Shell  → 启动 `bash main.sh`，stdin 注入 JSON payload
  │    PromptOnly → 返回"使用内置工具"提示
  │
  └─ stdout 解析为 SkillToolResult { output: Value, is_error: bool }
```

**通信协议（stdin/stdout）**：
```json
// stdin（发送给技能脚本）
{
  "tool": "db_query",
  "input": {"query": "SELECT * FROM users LIMIT 10"}
}

// stdout（技能返回）
{
  "output": [{"id": 1, "name": "Alice"}, ...],
  "is_error": false
}
```

---

## SkillRegistry

```rust
pub struct SkillRegistry {
    skills: HashMap<String, LoadedSkill>,
    skill_dir: PathBuf,
}

pub struct LoadedSkill {
    pub manifest: SkillManifest,
    pub dir: PathBuf,
    pub source: SkillSource,
    pub config_overrides: HashMap<String, String>, // 用户配置覆盖
}
```

**技能目录**：`~/.openfang/skills/{skill-name}/`

**热重载**：`POST /api/skills/reload` 重新扫描技能目录，无需重启 daemon。

---

## 技能来源（SkillSource）

```rust
pub enum SkillSource {
    Native,                           // 手动安装
    Bundled,                          // 随 OpenFang 打包的内置技能
    OpenClaw,                         // 从 OpenClaw 迁移
    ClawHub { slug: String, version: String }, // 从 ClawHub 市场安装
}
```

---

## ClawHub 市场（clawhub.rs，981行）

ClawHub 是 OpenFang 的技能市场（类似 npm 但面向 Agent 技能）：

```
GET  https://clawhub.sh/api/v1/skills?q=keyword   ← 搜索
GET  https://clawhub.sh/api/v1/skills/browse       ← 浏览
GET  https://clawhub.sh/api/v1/skills/{slug}       ← 详情
GET  https://clawhub.sh/api/v1/skills/{slug}/code  ← 源码
POST /api/clawhub/install/{slug}                   ← 安装
```

**速率限制**：ClawHub 有请求限制，`SkillError::RateLimited` 友好提示"请稍后重试"。

---

## 配置注入（config_injection.rs）

用户可以通过 API 覆盖技能的环境变量配置，无需修改 manifest：

```bash
PUT /api/skills/{id}/config
{
  "DATABASE_URL": "postgres://localhost/mydb",
  "MAX_RESULTS": "50"
}
```

这些覆盖存储在 `kernel.skill_config_overrides`，在技能重载时应用，不修改 boot-time `KernelConfig`。

---

## OpenClaw 兼容（openclaw_compat.rs）

OpenClaw 是 OpenFang 的前代产品。兼容层支持：
- 读取 OpenClaw 的 `CLAW.json` manifest
- 转换为 OpenFang `SKILL.toml` 格式
- 保持 Node.js 运行时兼容性

迁移工具：`openfang migrate`（`openfang-migrate` crate）

---

## 内置技能（bundled.rs）

编译时嵌入的技能，无需安装，随 OpenFang 分发：
- 通过 `include_str!()` 宏将 SKILL.toml 内容编译进二进制
- 在 `SkillRegistry::load_bundled()` 时自动注册

---

## 技能安全

- **子进程沙箱**：Python/Node/Shell 技能在独立子进程中运行
- **环境隔离**：技能子进程只接收明确传递的环境变量
- **超时控制**：技能执行超过 `TOOL_TIMEOUT_SECS`（120秒）被强制终止
- **能力检查**：技能工具也需要通过 `CapabilityManager` 检查（`ToolInvoke(name)` 能力）
