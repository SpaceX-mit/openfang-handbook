# Runtime 运行时深度解析

## 文件位置

```
crates/openfang-runtime/src/
├── agent_loop.rs        ← 核心执行循环（5493行）
├── kernel_handle.rs     ← KernelHandle trait 定义
├── llm_driver.rs        ← LLM 驱动 trait + 类型
├── tool_runner.rs       ← 内置工具执行（5014行）
├── model_catalog.rs     ← 模型目录（4866行，支持 100+ 模型）
├── prompt_builder.rs    ← Prompt 构建器（1016行）
├── compactor.rs         ← 上下文压缩器（1520行）
├── loop_guard.rs        ← 无限循环防护（949行）
├── context_budget.rs    ← 上下文 token 预算
├── context_overflow.rs  ← 上下文溢出恢复
├── session_repair.rs    ← 会话修复（1464行）
├── sandbox.rs           ← WASM 沙箱
├── subprocess_sandbox.rs← 子进程沙箱（1240行）
├── audit.rs             ← Merkle 链审计日志
├── a2a.rs               ← A2A 协议实现
├── mcp.rs               ← MCP 客户端
├── mcp_server.rs        ← MCP 服务端
├── browser.rs           ← Playwright 浏览器自动化（1362行）
├── web_fetch.rs         ← 带 SSRF 防护的 HTTP 获取
├── web_search.rs        ← 多提供商搜索
├── embedding.rs         ← 向量嵌入驱动 trait
├── routing.rs           ← 模型智能路由
├── retry.rs             ← 指数退避重试
├── llm_errors.rs        ← LLM 错误规范化（1049行）
├── hooks.rs             ← 插件生命周期钩子
├── python_runtime.rs    ← Python 运行时
├── tts.rs               ← 文字转语音
├── image_gen.rs         ← 图像生成
├── shell_bleed.rs       ← Shell 注入防护
├── taint.rs             ← 污点传播追踪（在 types crate）
└── drivers/             ← 各 LLM 提供商实现
    ├── anthropic.rs
    ├── openai.rs
    ├── gemini.rs
    ├── vertex.rs
    ├── bedrock.rs
    ├── copilot.rs
    ├── claude_code.rs
    ├── qwen_code.rs
    ├── fallback.rs
    └── mod.rs
```

---

## Agent 执行循环详解

### 入口函数

```rust
// 非流式（返回最终结果）
pub async fn run_agent_loop(
    manifest: &AgentManifest,
    session: &mut Session,
    driver: &dyn LlmDriver,
    kernel: Option<Arc<dyn KernelHandle>>,
    web_ctx: Option<&WebToolsContext>,
    skill_registry: Option<&SkillRegistry>,
    mcp_connections: &[McpConnection],
    // ...
) -> OpenFangResult<AgentLoopResult>

// 流式（通过 mpsc channel 推送 StreamEvent）
pub async fn run_agent_loop_streaming(...)
```

### 循环核心逻辑

```
while iteration < MAX_ITERATIONS(50):
    1. 构建 CompletionRequest
       ├─ system prompt (manifest.system_prompt)
       ├─ 注入相关记忆（语义搜索）
       ├─ 注入工作区上下文（当前目录、文件摘要）
       └─ 工具定义列表（内置 + 技能 + MCP）

    2. LLM 调用 → CompletionResponse
       ├─ 指数退避重试（最多 MAX_RETRIES=3 次）
       ├─ 提供商冷却追踪（auth_cooldown）
       └─ loop_guard 检查（防止内容循环）

    3. 如果 stop_reason == ToolUse:
       ├─ 对每个 tool_call:
       │    ├─ 能力检查（capability check）
       │    ├─ 审批检查（approval policy）
       │    ├─ 污点检查（taint tracking）
       │    ├─ 超时控制（120s for tools, 600s for agent calls）
       │    └─ 执行工具 → ToolResult
       └─ 将 ToolResult 追加到 session，继续循环

    4. 如果 stop_reason == EndTurn / MaxTokens:
       ├─ MaxTokens: 最多 MAX_CONTINUATIONS=5 次续写
       └─ EndTurn: 检查幽灵动作、静默令牌，返回结果

    5. 计量：metering.record(usage)
    6. 审计：audit_log.record(...)
    7. 持久化：session_store.save(session)
```

### 关键常量

| 常量 | 值 | 说明 |
|------|-----|------|
| `MAX_ITERATIONS` | 50 | 防止无限工具调用循环 |
| `MAX_RETRIES` | 3 | LLM 调用重试次数 |
| `MAX_CONTINUATIONS` | 5 | MaxTokens 续写次数 |
| `TOOL_TIMEOUT_SECS` | 120 | 工具执行超时（可被 `OPENFANG_TOOL_TIMEOUT_SECS=0` 禁用） |
| `AGENT_TOOL_TIMEOUT_SECS` | 600 | Agent 间调用超时（可被 `OPENFANG_AGENT_TOOL_TIMEOUT_SECS=0` 禁用） |
| `MAX_AGENT_CALL_DEPTH` | 5 | Agent 递归调用深度上限 |

---

## LLM 驱动抽象

### LlmDriver trait

```rust
#[async_trait]
pub trait LlmDriver: Send + Sync {
    async fn complete(&self, request: CompletionRequest)
        -> Result<CompletionResponse, LlmError>;

    // 可选，默认用非流式模拟流式
    async fn complete_streaming(
        &self,
        request: CompletionRequest,
        tx: mpsc::Sender<StreamEvent>,
    ) -> Result<(), LlmError> { ... }
}
```

### CompletionRequest

```rust
pub struct CompletionRequest {
    pub model: String,
    pub messages: Vec<Message>,
    pub tools: Vec<ToolDefinition>,
    pub max_tokens: u32,
    pub temperature: f32,
    pub system: Option<String>,
    pub thinking: Option<ThinkingConfig>,  // Claude extended thinking
}
```

### StreamEvent 类型

| 变体 | 说明 |
|------|------|
| `TextDelta { text }` | 增量文本 |
| `ThinkingDelta { text }` | 思考过程增量（Claude） |
| `ToolUseStart { id, name }` | 工具调用开始 |
| `ToolInputDelta { text }` | 工具输入 JSON 增量 |
| `ToolUseEnd { id, name, input }` | 工具调用完成 |
| `ContentComplete { stop_reason, usage }` | 整体完成 |
| `PhaseChange { phase, detail }` | 阶段变化（UX 指示器） |
| `ToolExecutionResult { ... }` | 工具执行结果 |

---

## 内置工具列表

`tool_runner.rs` 中实现，通过 `builtin_tool_definitions()` 注册：

### 文件系统
- `file_read` — 读取文件（支持 glob、行范围）
- `file_write` — 写入文件（全量覆盖）
- `file_edit` — 精确字符串替换
- `file_list` — 列目录（递归）
- `file_delete` — 删除文件/目录
- `file_search` — 在文件中搜索模式

### Shell/进程
- `shell_exec` — 执行 shell 命令（受沙箱控制）
- `process_start` — 启动持久进程（REPL、服务器）
- `process_send` — 向持久进程发送输入
- `process_read` — 读取进程输出
- `process_stop` — 终止持久进程

### 网络
- `web_fetch` — 带 SSRF 防护的 HTTP 获取（转 Markdown）
- `web_search` — 多提供商搜索（DuckDuckGo、Brave、Bing 等）
- `image_download` — 下载图像

### Agent 间协作
- `agent_send` — 向另一个 Agent 发消息（同步等待回复）
- `agent_spawn` — 创建新 Agent（返回 ID）
- `agent_list` — 列出所有运行中 Agent
- `agent_find` — 按名字/标签搜索 Agent
- `agent_kill` — 终止 Agent
- `agent_activate` — 唤醒暂停的 Agent

### 内存
- `memory_store` — 存储键值对到共享内存
- `memory_recall` — 检索记忆
- `knowledge_add` — 向知识图谱添加实体/关系
- `knowledge_query` — 查询知识图谱

### 任务队列
- `task_post` — 发布任务
- `task_claim` — 认领任务
- `task_complete` — 完成任务
- `task_list` — 列出任务

### 渠道通信
- `channel_send` — 通过指定渠道发送消息（支持 threading）
- `channel_send_media` — 发送媒体文件

### 媒体/AI
- `image_understand` — 图像描述（视觉模型）
- `audio_transcribe` — 音频转文字（Whisper）
- `tts_speak` — 文字转语音
- `browser_open` / `browser_click` / `browser_type` / `browser_screenshot` — 浏览器自动化

### Cron
- `cron_create` / `cron_list` / `cron_cancel` — Agent 定时任务

### 补丁
- `apply_patch` — 应用 unified diff 格式补丁

---

## 上下文管理

### context_budget.rs
- 追踪当前 session token 使用量
- `apply_context_guard()` — 在达到上限前截断 tool result
- `truncate_tool_result_dynamic()` — 按需动态截断

### compactor.rs（1520行）
- 当上下文接近上限时，将早期消息摘要化
- 使用 LLM 调用生成摘要，替换原始消息列表
- 保留最近 N 条消息完整

### context_overflow.rs
- 上下文溢出恢复策略枚举 `RecoveryStage`
- 先截断，再压缩，最后回退到新 session

### session_repair.rs（1464行）
- 检测并修复损坏的会话（工具调用未配对等问题）
- 在 LLM 返回"conversation must start with user" 错误时触发

---

## 沙箱系统

### WASM 沙箱（sandbox.rs）
- 使用 `wasmtime` 运行 WASM 技能模块
- 内存限制、CPU 限制
- `host_functions.rs` 提供 WASM 可调用的宿主函数

### 子进程沙箱（subprocess_sandbox.rs，1240行）
- 用于 Python/Shell/Node 技能
- `contains_shell_metacharacters()` — Shell 注入检测
- 防止路径穿越攻击

### Docker 沙箱（docker_sandbox.rs）
- 可选的 Docker 容器隔离（需要 Docker daemon）

---

## 幽灵动作检测

```rust
fn phantom_action_detected(text: &str) -> bool {
    // LLM 声称已执行操作（"sent", "posted"等）
    // 但实际上没有调用任何工具
    // → 重新触发，强制执行实际工具调用
}
```

---

## 模型路由（routing.rs）

三档路由：
- `simple_model`：prompt tokens < `simple_threshold`（默认 100）
- `complex_model`：prompt tokens > `complex_threshold`（默认 500）
- `medium_model`：介于两者之间

---

## 提供商健康监测（provider_health.rs）

- `ProbeCache` — 缓存提供商健康状态
- 自动故障转移到 `fallback_providers`
- `auth_cooldown.rs` — 认证失败后冷却期
