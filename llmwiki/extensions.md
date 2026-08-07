# Extensions 一键集成系统（openfang-extensions）

## 概述

Extensions 提供"一键安装"的第三方服务集成，基于 MCP（Model Context Protocol）服务器模板。

```
crates/openfang-extensions/src/
├── lib.rs        ← 核心类型（IntegrationTemplate、InstalledIntegration 等）
├── registry.rs   ← IntegrationRegistry（25 个内置模板 + 安装状态）
├── bundled.rs    ← 编译时内置的25个集成模板
├── installer.rs  ← 安装/卸载逻辑（一键流程）
├── credentials.rs← CredentialResolver（凭证优先级链）
├── vault.rs      ← 加密凭证保管库（AES-256-GCM）
├── oauth.rs      ← OAuth2 PKCE 流程（本地回调）
└── health.rs     ← HealthMonitor（自动重连 + 指数退避）
```

---

## 25 个内置集成

| 类别 | 集成名称 |
|------|---------|
| Dev Tools | github, gitlab, jira, linear, sentry, sonarqube |
| Productivity | notion, obsidian, google_drive, dropbox, airtable |
| Communication | slack, gmail, outlook, zoom |
| Data | postgres, mysql, mongodb, redis, elasticsearch |
| Cloud | aws, gcp, azure |
| AI & Search | brave_search, exa, tavily |

每个集成是一个编译时嵌入的 TOML 文件（通过 `include_str!`）。

---

## IntegrationTemplate

```rust
pub struct IntegrationTemplate {
    pub id: String,
    pub name: String,
    pub description: String,
    pub category: IntegrationCategory,
    pub version: String,
    pub transport: McpTransportTemplate,
    pub required_env: Vec<RequiredEnvVar>,
    pub optional_env: Vec<RequiredEnvVar>,
    pub oauth: Option<OAuthConfig>,
    pub docs_url: Option<String>,
    pub icon: Option<String>,
}
```

### MCP 传输模板

```rust
pub enum McpTransportTemplate {
    Stdio {
        command: String,          // 如 "npx"
        args: Vec<String>,        // 如 ["@modelcontextprotocol/server-github"]
    },
    Sse { url: String },
    Http { url: String },
}
```

**示例（GitHub 集成模板 TOML）**：
```toml
[integration]
id = "github"
name = "GitHub"
description = "Access GitHub repositories, issues, and pull requests"
category = "dev_tools"
version = "1.0.0"

[integration.transport]
type = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[[integration.required_env]]
name = "GITHUB_PERSONAL_ACCESS_TOKEN"
description = "GitHub Personal Access Token with repo scope"
```

---

## 安装流程

```bash
openfang add github
# 或通过 API：
POST /api/integrations/github/install
{ "config": { "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxx" } }
```

流程：
1. 查找 `IntegrationTemplate`
2. 收集必填 env vars（来自用户输入或 vault）
3. 将凭证写入 CredentialVault
4. 将 `McpServerConfigEntry` 写入 `~/.openfang/integrations.toml`
5. 追加到 `kernel.effective_mcp_servers`
6. 建立 MCP 连接

---

## 凭证解析器（CredentialResolver）

三级优先链：
```
1. Vault（加密存储，优先级最高）
2. dotenv 文件（~/.openfang/.env）
3. 系统环境变量
```

```rust
pub struct CredentialResolver {
    vault: Vault,
    dotenv_path: PathBuf,
}

impl CredentialResolver {
    pub fn resolve(&self, key: &str) -> Option<String> {
        self.vault.get(key)          // 1. 先查 vault
            .or_else(|| dotenv_get(key))  // 2. 再查 dotenv
            .or_else(|| std::env::var(key).ok())  // 3. 最后查环境变量
    }
}
```

---

## 凭证保管库（Vault，vault.rs）

- **加密**：AES-256-GCM
- **密钥来源**：`OPENFANG_VAULT_KEY` 环境变量
- **存储路径**：`~/.openfang/vault.json`（加密 JSON）
- **OS Keyring 集成**：在 macOS/Windows 可选使用 OS 原生密钥链

**操作**：
```
PUT  /api/vault/{key}     ← 存储凭证
GET  /api/vault/{key}     ← 读取（返回掩码值）
DELETE /api/vault/{key}   ← 删除
POST /api/vault/unlock    ← 解锁（提供 master key）
```

**错误**：`ExtensionError::VaultLocked` — 当 `OPENFANG_VAULT_KEY` 未设置时。

---

## OAuth2 PKCE（oauth.rs）

支持以下提供商的 OAuth2 授权码 + PKCE 流程：

- Google（Drive、Gmail、Calendar）
- GitHub
- Microsoft（Outlook、OneDrive）
- Slack

**流程**：
1. `POST /api/integrations/{name}/oauth/start` — 生成授权 URL
2. 用户在浏览器中授权
3. OpenFang 在 `localhost:PORT` 监听回调
4. 交换 code for token，存入 Vault

---

## 健康监测（HealthMonitor，health.rs）

- 定期 ping 已安装的集成（MCP 服务器）
- 故障时指数退避重连
- 通过 `GET /api/integrations/{name}/health` 查询状态

```rust
pub enum IntegrationHealth {
    Healthy,
    Degraded(String),   // 有错误但仍工作
    Unhealthy(String),  // 无法连接
    Unknown,            // 未检查
}
```

---

## effective_mcp_servers 合并逻辑

启动时，`kernel.effective_mcp_servers` 由两部分合并：

```
config.toml 中手动配置的 [[mcp_servers]]
+
integrations.toml 中已安装集成生成的 McpServerConfigEntry
=
effective_mcp_servers（RwLock<Vec<McpServerConfigEntry>>）
```

热重装时（安装/卸载集成），只更新 `effective_mcp_servers`，不需要重启。

---

## IntegrationStatus

```rust
pub enum IntegrationStatus {
    Available,    // 已有模板，未安装
    Installed,    // 已安装，正常
    Error(String),// 安装错误
}
```

---

## API 端点

```
GET  /api/integrations              ← 列出所有（含状态）
GET  /api/integrations/{name}       ← 单个集成详情
POST /api/integrations/{name}/install   ← 安装
DELETE /api/integrations/{name}         ← 卸载
POST /api/integrations/{name}/configure ← 更新配置
GET  /api/integrations/{name}/health    ← 健康状态
POST /api/integrations/{name}/oauth/start ← 启动 OAuth 流程
```
