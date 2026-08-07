# 渠道适配器（openfang-channels）

## 架构

```
crates/openfang-channels/src/
├── router.rs       ← ChannelRouter：消息路由到正确 Agent
├── bridge.rs       ← 渠道桥接（启动所有适配器，2813行）
├── types.rs        ← ChannelAdapter trait + 基础类型
├── formatter.rs    ← 输出格式化（Markdown/HTML/PlainText）
├── telegram.rs     ← Telegram Bot API（2843行）
├── slack.rs        ← Slack Events API
├── discord.rs      ← Discord Gateway（1635行）
├── feishu.rs       ← 飞书/Lark（1954行）
├── email.rs        ← SMTP + IMAP
├── whatsapp.rs     ← WhatsApp Web（via gateway）
├── wecom.rs        ← 企业微信
├── dingtalk.rs     ← 钉钉
├── dingtalk_stream.rs← 钉钉 Stream
├── teams.rs        ← Microsoft Teams
├── matrix.rs       ← Matrix
├── xmpp.rs         ← XMPP/Jabber
├── irc.rs          ← IRC
├── mqtt.rs         ← MQTT（IoT）
├── webhook.rs      ← 通用 Webhook
├── google_chat.rs  ← Google Chat
├── line.rs         ← LINE
├── viber.rs        ← Viber
├── signal.rs       ← Signal
├── nostr.rs        ← Nostr
├── mastodon.rs     ← Mastodon
├── bluesky.rs      ← Bluesky
├── reddit.rs       ← Reddit
├── linkedin.rs     ← LinkedIn
├── twitch.rs       ← Twitch
├── discord.rs      ← Discord
├── guilded.rs      ← Guilded
├── revolt.rs       ← Revolt
├── rocketchat.rs   ← Rocket.Chat
├── mattermost.rs   ← Mattermost
├── nextcloud.rs    ← Nextcloud Talk
├── keybase.rs      ← Keybase
├── gitter.rs       ← Gitter
├── zulip.rs        ← Zulip
├── gotify.rs       ← Gotify（推送通知）
├── ntfy.rs         ← ntfy（推送通知）
├── pumble.rs       ← Pumble
├── twist.rs        ← Twist
├── flock.rs        ← Flock
├── threema.rs      ← Threema
├── mumble.rs       ← Mumble
├── webex.rs        ← Cisco Webex
├── discourse.rs    ← Discourse
└── messenger.rs    ← Facebook Messenger
```

**合计 40+ 渠道**

---

## ChannelAdapter trait

```rust
#[async_trait]
pub trait ChannelAdapter: Send + Sync {
    /// 渠道唯一标识（如 "telegram"、"slack"）
    fn channel_name(&self) -> &str;

    /// 启动适配器（连接到平台、开始接收消息）
    async fn start(&self) -> Result<(), ChannelError>;

    /// 停止适配器
    async fn stop(&self) -> Result<(), ChannelError>;

    /// 发送文本消息
    async fn send_message(
        &self,
        recipient: &str,
        message: &str,
        thread_id: Option<&str>,
    ) -> Result<DeliveryReceipt, ChannelError>;

    /// 发送媒体（图片/文件）
    async fn send_media(
        &self,
        recipient: &str,
        media_type: &str,
        media_url: &str,
        caption: Option<&str>,
        thread_id: Option<&str>,
    ) -> Result<DeliveryReceipt, ChannelError>;

    /// 发送本地文件数据
    async fn send_file_data(
        &self,
        recipient: &str,
        data: Vec<u8>,
        filename: &str,
        mime_type: &str,
        thread_id: Option<&str>,
    ) -> Result<DeliveryReceipt, ChannelError>;
}
```

---

## 渠道配置（config.toml）

每个渠道的配置都在 `[channels.{name}]` 下，按需启用。

### Telegram

```toml
[channels.telegram]
enabled = true
bot_token = "123456:ABC-DEF..."
allowed_users = ["12345678"]           # 可选，限制允许用户
agent_id = "uuid-of-default-agent"    # 默认路由到哪个 Agent
```

### Slack

```toml
[channels.slack]
enabled = true
bot_token = "xoxb-..."
app_token = "xapp-..."
signing_secret = "..."
agent_id = "uuid"
```

### Discord

```toml
[channels.discord]
enabled = true
token = "Bot token..."
guild_ids = ["12345678"]              # 服务器 ID 白名单
agent_id = "uuid"
```

### 飞书（Feishu/Lark）

```toml
[channels.feishu]
enabled = true
app_id = "cli_..."
app_secret = "..."
verification_token = "..."
agent_id = "uuid"
```

### 企业微信（WeCom）

```toml
[channels.wecom]
enabled = true
corp_id = "..."
app_secret = "..."
agent_id_wecom = 1000001              # 应用 ID（数字）
token = "..."
encoding_aes_key = "..."
agent_id = "uuid"
```

### Email（SMTP + IMAP）

```toml
[channels.email]
enabled = true
smtp_host = "smtp.gmail.com"
smtp_port = 587
username = "bot@gmail.com"
password = "app-password"
imap_host = "imap.gmail.com"
imap_port = 993
agent_id = "uuid"
```

### Webhook（通用）

```toml
[channels.webhook]
enabled = true
path = "/webhook/my-bot"         # 监听路径
secret = "optional-hmac-secret" # 可选签名验证
agent_id = "uuid"
```

---

## 消息路由（ChannelRouter）

`router.rs` 负责将来自渠道的消息路由到正确的 Agent：

1. **按 agent_id 路由**：渠道配置中指定默认 Agent
2. **Agent Bindings**：`AgentBinding` — 将特定用户/群组绑定到特定 Agent
3. **多 Agent 渠道**：同一渠道的不同用户可路由到不同 Agent

```rust
pub struct AgentBinding {
    pub channel: String,           // 渠道名
    pub channel_id: String,        // 频道/用户 ID
    pub agent_id: AgentId,         // 目标 Agent
}
```

---

## 渠道行为配置（ChannelOverrides）

每个 Agent 可以针对不同渠道配置行为：

```toml
[agent.channels.telegram]
dm_policy = "respond"          # 私信处理：respond/allowed_only/ignore
group_policy = "mention_only"  # 群消息：all/mention_only/commands_only/ignore
rate_limit_per_user = 10       # 每用户每分钟消息数（0=不限）
threading = true               # 是否回复为线程
output_format = "telegram_html" # markdown/telegram_html/slack_mrkdwn/plain_text
prefix_style = "bracket"       # 消息前缀：off/bracket/bold_bracket
```

---

## 消息格式化（formatter.rs）

根据 `output_format` 自动转换输出格式：

- `Markdown` → 标准 Markdown
- `TelegramHtml` → `<b>bold</b>` `<i>italic</i>` `<code>code</code>` 等
- `SlackMrkdwn` → `*bold*` `_italic_` `` `code` `` 等
- `PlainText` → 去除所有 Markdown 标记

---

## 交付追踪（DeliveryTracker）

Kernel 中的 `DeliveryTracker` 追踪消息交付状态：
- 每个 Agent 最多保存 500 条收据
- 全局上限 10,000 条
- 用于避免重复交付（幂等性保证）

```rust
pub struct DeliveryReceipt {
    pub message_id: String,
    pub channel: String,
    pub recipient: String,
    pub delivered_at: chrono::DateTime<Utc>,
    pub status: DeliveryStatus,
}
```

---

## WhatsApp 特殊处理

WhatsApp 无官方 Bot API，通过 `whatsapp_gateway` 子进程实现（基于 WhatsApp Web）：

```
内核 → HTTP POST → whatsapp-gateway 进程 → WhatsApp Web
```

- 启动：`POST /api/whatsapp/qr/start`（获取 QR 码）
- 认证：用手机扫描 QR
- 状态：`GET /api/whatsapp/qr/status`
- 清理：kernel 关闭时通过 `whatsapp_gateway_pid` 终止子进程
