# 配置文件完整参考（~/.openfang/config.toml）

## 文件类型

```
openfang-types/src/config.rs（4701行）— 所有配置类型定义
KernelConfig — 顶层结构体
```

**重要规则**：新增字段必须同时：
1. 在 struct 中添加字段
2. 添加 `#[serde(default)]`
3. 在 `Default` impl 中提供默认值
4. 确保 `Serialize`/`Deserialize` 正确实现

---

## 顶层结构

```toml
# 监听地址
listen = "0.0.0.0:4200"

# API 认证密钥（空字符串 = 无认证）
api_key = ""

# 默认 LLM 模型
[model]
default = "claude-sonnet-4-20250514"
```

---

## API 密钥配置

```toml
[api_keys]
anthropic = ""          # 或 ANTHROPIC_API_KEY 环境变量
openai = ""             # 或 OPENAI_API_KEY
groq = ""               # 或 GROQ_API_KEY
gemini = ""             # 或 GEMINI_API_KEY / GOOGLE_API_KEY
deepseek = ""           # 或 DEEPSEEK_API_KEY
mistral = ""            # 或 MISTRAL_API_KEY
cohere = ""
xai = ""
together = ""
perplexity = ""
fireworks = ""
```

---

## 模型配置

```toml
[model]
default = "claude-sonnet-4-20250514"     # 全局默认模型

# 热重载覆盖（daemon 运行时可更新，无需重启）
# default_model_override 通过 API 设置，存在 kernel.default_model_override 中
```

---

## 预算配置

```toml
[budget]
global_limit_usd = 100.0        # 全局花费上限（0 = 不限）
per_agent_limit_usd = 10.0      # 单个 Agent 上限（0 = 不限）
reset_period = "monthly"        # 重置周期：daily / weekly / monthly / never
warn_at_percent = 80            # 达到上限的 80% 时警告
```

---

## Dashboard 认证

```toml
[auth]
enabled = false
password_hash = ""          # Argon2id 格式，用 `openfang auth hash-password` 生成
session_secret = ""         # 32字节随机十六进制（用 `openfang auth gen-secret` 生成）
session_timeout_hours = 24
```

---

## 故障转移提供商链

```toml
[[fallback_providers]]
name = "anthropic"           # 提供商名（与 api_keys 字段名对应）
priority = 1                 # 数字越小优先级越高
timeout_secs = 30            # 单次请求超时
models = []                  # 指定模型列表（空 = 所有可用模型）

[[fallback_providers]]
name = "openai_compat"
priority = 2
timeout_secs = 60
base_url = "http://localhost:11434/v1"  # OpenAI 兼容端点
api_key = "ollama"
```

---

## OFP 网络

```toml
[network]
enabled = false
listen_addr = "0.0.0.0:5678"
secret = ""                  # 节点间共享密钥（强制 HMAC 认证）
peer_id = ""                 # 本地节点 UUID（空 = 自动生成）

[[network.peers]]
address = "192.168.1.100:5678"
name = "remote-node"
secret = ""                  # 可覆盖全局 secret
```

---

## MCP 服务器

```toml
[[mcp_servers]]
name = "filesystem"
transport = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/home/user"]

[[mcp_servers]]
name = "custom-api"
transport = "sse"
url = "http://localhost:8080/mcp/sse"

[[mcp_servers]]
name = "github"
transport = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]
env = { GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxx" }
```

---

## 渠道配置

### Telegram

```toml
[channels.telegram]
enabled = false
bot_token = ""
allowed_users = []            # 空 = 允许所有用户
allowed_groups = []           # 空 = 允许所有群组
agent_id = ""                 # 默认路由 Agent UUID

[channels.telegram.overrides]  # 行为覆盖
dm_policy = "respond"
group_policy = "mention_only"
rate_limit_per_user = 0       # 0 = 不限速
threading = false
output_format = "telegram_html"
lifecycle_reactions = true    # 发送 ⏳🤔✅❌ 反应
```

### Slack

```toml
[channels.slack]
enabled = false
bot_token = ""                # xoxb-...
app_token = ""                # xapp-...（Socket Mode）
signing_secret = ""
agent_id = ""
```

### Discord

```toml
[channels.discord]
enabled = false
token = ""                    # Bot Token
guild_ids = []                # 服务器 ID 白名单（空 = 所有服务器）
agent_id = ""
```

### 飞书/Lark

```toml
[channels.feishu]
enabled = false
app_id = ""
app_secret = ""
verification_token = ""
encrypt_key = ""              # 加密密钥（可选）
agent_id = ""
```

### 企业微信

```toml
[channels.wecom]
enabled = false
corp_id = ""
app_secret = ""
agent_id_wecom = 0            # 微信应用 ID（整数）
token = ""
encoding_aes_key = ""
agent_id = ""
```

### 钉钉

```toml
[channels.dingtalk]
enabled = false
app_key = ""
app_secret = ""
robot_code = ""
agent_id = ""
```

### Email（SMTP + IMAP）

```toml
[channels.email]
enabled = false
smtp_host = ""
smtp_port = 587
smtp_tls = true
username = ""
password = ""
imap_host = ""
imap_port = 993
poll_interval_secs = 30
agent_id = ""
```

### WhatsApp

```toml
[channels.whatsapp]
enabled = false
gateway_url = "http://localhost:3000"  # whatsapp-gateway 地址
agent_id = ""
```

### Webhook（通用）

```toml
[channels.webhook]
enabled = false
path = "/webhook"
secret = ""                   # HMAC-SHA256 签名密钥（可选）
agent_id = ""
```

---

## 日志/追踪

```toml
[logging]
level = "info"                # trace / debug / info / warn / error
format = "text"               # text / json
file = ""                     # 日志文件路径（空 = 仅 stderr）
```

---

## 沙箱配置

```toml
[sandbox]
use_docker = false            # 是否使用 Docker 隔离
docker_image = "openfang-sandbox:latest"
wasm_memory_limit_mb = 256   # WASM 模块内存上限
subprocess_timeout_secs = 30 # 技能子进程超时
```

---

## Python 运行时

```toml
[python]
executable = "python3"        # Python 可执行文件路径
venv_path = ""                # 虚拟环境路径（空 = 使用系统 Python）
```

---

## 技能全局配置

```toml
[skills]
# 每个技能的配置，键为技能 ID
[skills.database-query]
DATABASE_URL = "postgres://localhost/mydb"
```

---

## 广播配置

```toml
[broadcast]
enabled = false
channels = ["telegram", "slack"]  # 广播到哪些渠道
agent_id = ""                     # 发送广播的 Agent
```

---

## Agent Bindings（多 Agent 渠道路由）

```toml
[[agent_bindings]]
channel = "telegram"
channel_id = "123456789"      # 用户/群组 ID
agent_id = "uuid-of-agent"   # 路由到此 Agent

[[agent_bindings]]
channel = "slack"
channel_id = "C0123456789"    # Slack 频道 ID
agent_id = "uuid-of-agent2"
```

---

## 思考配置（Claude Extended Thinking）

```toml
[thinking]
enabled = false
budget_tokens = 10000         # 思考 token 预算（1024-100000）
```

---

## 配置热重载

某些字段支持无需重启 daemon 的热重载（通过 `config_reload.rs`）：

| 字段 | 热重载支持 |
|------|-----------|
| `default_model` | ✅（`default_model_override`） |
| `fallback_providers` | ✅（`fallback_providers_override`） |
| `channels.*` | ✅（`POST /api/channels/reload`） |
| `skills.*` | ✅（`POST /api/skills/reload`） |
| `api_key` | ❌（需重启） |
| `listen` | ❌（需重启） |

---

## 完整最小配置示例

```toml
# ~/.openfang/config.toml
listen = "0.0.0.0:4200"

[api_keys]
anthropic = "sk-ant-..."

[model]
default = "claude-haiku-4-5-20251001"

[budget]
global_limit_usd = 50.0

[channels.telegram]
enabled = true
bot_token = "123456:ABC..."
agent_id = "00000000-0000-0000-0000-000000000000"
```

---

## 配置文件 schema 维护要点

修改 `KernelConfig` 时的完整 checklist：

```
□ 在结构体字段上加 #[serde(default)]
□ 在 Default impl 中添加默认值
□ 确保 Serialize + Deserialize 已 derive
□ 如需热重载：在 config_reload.rs 添加热更新逻辑
□ 如需 API 控制：在 routes.rs 添加 GET/PUT 端点
□ 运行 cargo build --workspace --lib 验证编译
```
