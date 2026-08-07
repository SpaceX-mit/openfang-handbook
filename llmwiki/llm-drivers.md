# LLM 驱动（openfang-runtime/src/drivers/）

## 支持的提供商

| 驱动文件 | 提供商 | 说明 |
|----------|--------|------|
| `anthropic.rs` | Anthropic | Claude 系列（claude-opus-5、claude-sonnet-4 等） |
| `openai.rs` | OpenAI + 兼容 API | GPT-4o、o1、o3，及 Ollama/LocalAI/vLLM |
| `gemini.rs` | Google Gemini | gemini-2.0-flash、gemini-1.5-pro 等 |
| `vertex.rs` | Google Vertex AI | Gemini via Vertex |
| `bedrock.rs` | AWS Bedrock | Claude on AWS |
| `copilot.rs` | GitHub Copilot | via OIDC token |
| `claude_code.rs` | Claude Code | claude-code-xxx 模型路由 |
| `qwen_code.rs` | Qwen/阿里云 | qwen-coder、qwen-plus 等 |
| `fallback.rs` | 故障转移链 | 按优先级尝试多个提供商 |

---

## LlmDriver trait

```rust
#[async_trait]
pub trait LlmDriver: Send + Sync {
    async fn complete(&self, request: CompletionRequest)
        -> Result<CompletionResponse, LlmError>;

    // 默认实现：用非流式模拟
    async fn complete_streaming(
        &self,
        request: CompletionRequest,
        tx: mpsc::Sender<StreamEvent>,
    ) -> Result<(), LlmError>;
}
```

---

## 各驱动配置

### Anthropic（anthropic.rs）

**激活条件**：`ANTHROPIC_API_KEY` 环境变量或 `config.toml` 的 `api_keys.anthropic`

**特性**：
- Extended Thinking（`ThinkingConfig`）
- Prompt Caching（`cache_read_tokens`/`cache_write_tokens` 计费）
- 流式输出（原生 SSE）
- Vision（图像理解）

**模型示例**：
```
claude-opus-5
claude-sonnet-4-20250514
claude-haiku-4-5-20251001
```

**端点**：`https://api.anthropic.com/v1/messages`

---

### OpenAI（openai.rs）

**激活条件**：`OPENAI_API_KEY`

**特性**：
- 兼容任何 OpenAI 格式 API（Ollama、LocalAI、vLLM、LMStudio）
- 通过 `base_url` 配置自定义端点
- o1/o3 系列推理模型（特殊参数处理）

**自定义端点配置**（config.toml）：
```toml
[providers.openai_compat]
name = "ollama"
base_url = "http://localhost:11434/v1"
api_key = "ollama"
models = ["llama3.1:70b", "qwen2.5-coder:32b"]
```

---

### Gemini（gemini.rs）

**激活条件**：`GEMINI_API_KEY` / `GOOGLE_API_KEY`

**端点**：`https://generativelanguage.googleapis.com/v1beta`

**模型示例**：
```
gemini-2.0-flash
gemini-1.5-pro
gemini-1.5-flash
```

---

### Google Vertex AI（vertex.rs）

**激活条件**：`GOOGLE_CLOUD_PROJECT` + 服务账号凭证

**端点**：Vertex AI API，支持 Gemini 系列

**配置**（config.toml）：
```toml
[providers.vertex]
project = "my-project"
location = "us-central1"
credentials_path = "~/.config/gcloud/application_default_credentials.json"
```

---

### AWS Bedrock（bedrock.rs）

**激活条件**：`AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY`

**支持模型**：
- Claude on Bedrock（`anthropic.claude-3-5-sonnet-20241022-v2:0` 等）

**配置**（config.toml）：
```toml
[providers.bedrock]
region = "us-east-1"
```

---

### 故障转移驱动（fallback.rs）

按优先级依次尝试多个提供商：

```toml
[[fallback_providers]]
name = "anthropic"
priority = 1
timeout_secs = 30

[[fallback_providers]]
name = "openai"
priority = 2
timeout_secs = 30

[[fallback_providers]]
name = "ollama"
priority = 3
base_url = "http://localhost:11434/v1"
```

**热重载**：`fallback_providers_override` 字段支持不重启 daemon 更新故障转移链。

---

## 模型目录（model_catalog.rs）

**文件**：`crates/openfang-runtime/src/model_catalog.rs`（4866行）

包含 100+ 预定义模型，每个模型有：
- `context_window`：上下文窗口大小
- `max_output_tokens`：最大输出 token
- `supports_tools`：是否支持工具调用
- `supports_vision`：是否支持视觉
- `supports_thinking`：是否支持 extended thinking
- `cost_per_1m_input`：每百万 input token 价格（USD）
- `cost_per_1m_output`：每百万 output token 价格（USD）

用于：
- 计费计算（MeteringEngine）
- 上下文预算管理
- 模型路由决策

---

## LlmError 类型

```rust
pub enum LlmError {
    Http(String),
    Api { status: u16, message: String },
    RateLimited { retry_after_ms: u64 },
    Parse(String),
    MissingApiKey(String),
    Overloaded { retry_after_ms: u64 },
    AuthenticationFailed(String),
    ModelNotFound(String),
}
```

**自动重试逻辑**：
- `RateLimited`/`Overloaded`：指数退避（最多 `MAX_RETRIES=3`）
- `AuthenticationFailed`：触发提供商冷却期（`auth_cooldown.rs`），切换故障转移链
- `ModelNotFound`：不重试，直接返回错误

---

## 配置路径汇总

### 环境变量（优先级最高）

```bash
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=...
GROQ_API_KEY=gsk_...
DEEPSEEK_API_KEY=...
MISTRAL_API_KEY=...
```

### config.toml

```toml
[api_keys]
anthropic = "sk-ant-..."
openai = "sk-..."
groq = "gsk_..."
gemini = "..."

[model]
default = "claude-sonnet-4-20250514"  # 全局默认模型

[providers.openai_compat]
# OpenAI 兼容 API（Ollama 等）
```

---

## StubDriver（无配置时）

当没有任何 LLM 提供商时：

```rust
impl LlmDriver for StubDriver {
    async fn complete(&self, _req: CompletionRequest) -> Result<CompletionResponse, LlmError> {
        Err(LlmError::MissingApiKey(
            "No LLM provider configured. Set an API key (e.g. GROQ_API_KEY) and restart."
        ))
    }
}
```

Dashboard 仍然可以启动和显示，只是发消息时会返回错误而不是 panic。
