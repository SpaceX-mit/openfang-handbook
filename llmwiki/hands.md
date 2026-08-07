# Hands 自主能力包（openfang-hands）

## 概念

Hand 是预制的、领域完整的 Agent 配置包。区别于普通 Agent：

| | 普通 Agent | Hand |
|---|----------|------|
| 交互方式 | 你和 Agent 对话 | Agent 自主工作，你定期检查 |
| 配置方式 | 手动写 manifest | 从市场一键激活 |
| 目标 | 回答问题 | 24/7 自主完成任务 |
| 例子 | 聊天助手 | 内容监控、SEO 优化、竞品追踪 |

---

## 文件结构

```
crates/openfang-hands/src/
├── lib.rs       ← 核心类型（HandDefinition、HandInstance 等）
├── registry.rs  ← HandRegistry（定义 + 实例管理，1365行）
└── bundled.rs   ← 编译时内置 Hands
```

---

## 核心类型

### HandDefinition（HAND.toml 定义）

```rust
pub struct HandDefinition {
    pub id: String,
    pub name: String,
    pub description: String,
    pub category: HandCategory,
    pub version: String,
    pub author: String,
    pub tags: Vec<String>,
    pub requirements: Vec<HandRequirement>,
    pub settings: Vec<HandSetting>,
    pub agent_manifest: String,   // 激活时使用的 Agent TOML manifest
    pub readme: Option<String>,
    pub icon: Option<String>,
}
```

### HandCategory 枚举

```rust
pub enum HandCategory {
    Content,       // 内容创作/营销
    Security,      // 安全监控
    Productivity,  // 效率工具
    Development,   // 开发工具
    Communication, // 通信
    Data,          // 数据分析
    Finance,       // 财务
    Other,
}
```

### HandRequirement（依赖检查）

```rust
pub struct HandRequirement {
    pub name: String,
    pub requirement_type: RequirementType,  // Binary / EnvVar / ApiKey
    pub description: String,
    pub required: bool,     // false = 可选依赖
    pub install_info: HandInstallInfo,
}

// 安装指引（跨平台）
pub struct HandInstallInfo {
    pub macos: Option<String>,      // brew install playwright
    pub windows: Option<String>,    // choco install playwright
    pub linux_apt: Option<String>,  // apt install playwright
    pub linux_dnf: Option<String>,
    pub linux_pacman: Option<String>,
    pub pip: Option<String>,        // pip install playwright
    pub signup_url: Option<String>, // API key 注册地址
    pub docs_url: Option<String>,
}
```

### HandSetting（用户可配置项）

```rust
pub struct HandSetting {
    pub key: String,
    pub label: String,
    pub description: String,
    pub setting_type: HandSettingType,
    pub default: String,
    pub options: Vec<HandSettingOption>,  // 枚举类型用
}

pub enum HandSettingType {
    Text,
    Password,    // 遮蔽显示
    Select,      // 枚举选择
    Toggle,      // 布尔开关
    Number,
}
```

### HandInstance（激活实例）

```rust
pub struct HandInstance {
    pub id: Uuid,                    // 实例 UUID
    pub hand_id: String,             // 来自哪个 Hand 定义
    pub agent_id: AgentId,           // 关联的 Agent ID
    pub status: HandStatus,
    pub settings: HashMap<String, Value>, // 用户配置的设置值
    pub activated_at: DateTime<Utc>,
    pub last_activity: Option<DateTime<Utc>>,
}

pub enum HandStatus {
    Active,
    Paused,
    Error(String),
}
```

---

## Hand 激活流程

```
POST /api/hands/{hand_id}/activate
  { "config": {"setting_key": "value"} }

  │
  ├─ 1. 检查 HandDefinition 存在
  ├─ 2. 合并用户设置 + 默认值
  ├─ 3. 渲染 agent_manifest（填入设置值）
  ├─ 4. kernel.spawn_agent(rendered_manifest)
  ├─ 5. 创建 HandInstance（关联 agent_id）
  └─ 6. 返回 HandInstance 信息
```

---

## HAND.toml 格式示例

```toml
[hand]
id = "content-monitor"
name = "Content Monitor"
description = "Monitor websites and social media for mentions"
category = "content"
version = "1.0.0"
author = "OpenFang"
tags = ["monitoring", "content", "automation"]

[[hand.requirements]]
name = "playwright"
requirement_type = "binary"
description = "Browser automation"
required = false
[hand.requirements.install_info]
pip = "pip install playwright && playwright install chromium"

[[hand.requirements]]
name = "OPENAI_API_KEY"
requirement_type = "api_key"
description = "OpenAI API key for content analysis"
required = true
[hand.requirements.install_info]
signup_url = "https://platform.openai.com"

[[hand.settings]]
key = "target_url"
label = "Target URL"
description = "Website or RSS feed to monitor"
setting_type = "text"
default = ""

[[hand.settings]]
key = "check_interval"
label = "Check Interval"
setting_type = "select"
default = "hourly"
[[hand.settings.options]]
value = "hourly"
label = "Every hour"
[[hand.settings.options]]
value = "daily"
label = "Once a day"

[hand.agent_manifest]
# TOML manifest 模板（{{setting_key}} 被替换为用户设置）
content = '''
[agent]
name = "content-monitor-{{instance_id}}"
model = "gpt-4o"
system_prompt = """
You are a content monitoring agent. Check {{target_url}} every {{check_interval}}.
"""
tools = ["web_fetch", "web_search", "memory_store", "channel_send"]

[agent.autonomous]
max_restarts = 10
heartbeat_interval_secs = 300
'''
```

---

## HandRegistry 关键操作

```rust
// 加载内置 Hands（编译时嵌入）
registry.load_bundled()

// 从文件加载自定义 Hand
registry.load_from_file(path, audit_callback)

// 获取所有 Hand 定义
registry.list_definitions() -> Vec<HandDefinition>

// 激活 Hand（创建 Agent + 实例）
registry.activate(hand_id, settings) -> HandResult<HandInstance>

// 暂停/恢复/停用
registry.pause(instance_id)
registry.resume(instance_id)
registry.deactivate(instance_id)

// 查询状态
registry.get_instance(instance_id) -> Option<HandInstance>
registry.list_active() -> Vec<HandInstance>
```

---

## 审计集成

每次加载/重载 HAND.toml 时，`HandRegistry` 计算文件的 SHA-256 并调用审计回调：

```rust
type HandAuditCallback = Arc<dyn Fn(&str, &str) + Send + Sync>;
// 参数：(hand_id, sha256_hex)
// 由 kernel 挂接到 AuditLog::record(ConfigChange, ...)
```

这确保 Hand 配置变更留在 Merkle 审计链中（防篡改记录）。

---

## API 端点

```
GET  /api/hands                           ← 所有 Hand 定义
GET  /api/hands/active                    ← 活跃实例列表
GET  /api/hands/{id}                      ← 单个 Hand 详情（含依赖状态）
POST /api/hands/{id}/check-deps           ← 检查依赖是否满足
POST /api/hands/{id}/install-deps         ← 安装缺失依赖
POST /api/hands/{id}/activate             ← 激活（创建实例）
POST /api/hands/instances/{id}/pause      ← 暂停
POST /api/hands/instances/{id}/resume     ← 恢复
POST /api/hands/instances/{id}/deactivate ← 停用
GET  /api/hands/instances/{id}/settings   ← 获取配置
PUT  /api/hands/instances/{id}/settings   ← 更新配置
GET  /api/hands/stats                     ← 统计信息
```
