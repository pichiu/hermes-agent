# Stage 2.5 外部整合

## LLM Providers

| Provider | 整合方式 | 關鍵檔案 | 備註 |
|----------|---------|----------|------|
| OpenRouter | OpenAI SDK（OpenAI-compatible） | `run_agent.py` | 預設路由，200+ 模型 |
| Nous Portal | OpenAI-compatible | `agent/nous_rate_guard.py` | 統一訂閱，含 Tool Gateway |
| Anthropic | 原生 + OpenAI-compatible | `agent/anthropic_adapter.py` | Prompt caching 支援 |
| Google Gemini | 原生 API | `agent/gemini_native_adapter.py`, `agent/gemini_schema.py` | |
| AWS Bedrock | Boto3 adapter | `agent/bedrock_adapter.py` | |
| Azure OpenAI | Azure Identity | `agent/azure_identity_adapter.py` | |
| OpenAI Codex | Responses API | `agent/codex_responses_adapter.py` | |
| Google Code Assist | | `agent/google_code_assist.py` | |
| Ollama | OpenAI-compatible | 直接用 OpenAI SDK | 本地部署 |
| LM Studio | OpenAI-compatible | `agent/lmstudio_reasoning.py` | 本地部署 |
| NVIDIA NIM | OpenAI-compatible | 設定 base_url | |
| Kimi/Moonshot | OpenAI-compatible | `agent/moonshot_schema.py` | |
| 各中國廠商 | 各自 adapter | 見 `hermes_constants.py` | MiniMax, GLM, MiMo 等 |

## 工具整合（外部 API 呼叫）

### Web 搜尋

| Provider | 套件 | 觸發工具 |
|----------|------|---------|
| Firecrawl | `firecrawl-py==4.17.0` | `web_search` |
| Exa | `exa-py==2.10.2` | `web_search` |
| Parallel-web | `parallel-web==0.4.2` | `web_search` |
| SerpAPI | ⚠️ 待驗證 | |

**懶載入**：搜尋 provider 依賴通過 `tools/lazy_deps.py` 在首次使用時安裝，避免安裝時強制下載所有依賴

### 圖像生成

| Provider | 套件 | 觸發工具 |
|----------|------|---------|
| FAL.ai | `fal-client==0.13.1` | `image_gen` |
| 其他（⚠️ 待驗證） | | |

**Provider 選擇**：`agent/image_gen_provider.py` + `agent/image_gen_registry.py`（Registry pattern）

### 語音合成（TTS）

| Provider | 套件 | 檔案 |
|----------|------|------|
| Edge TTS | `edge-tts==7.2.7` | 預設 TTS |
| ElevenLabs | ⚠️ 待驗證 | |
| OpenAI TTS | OpenAI SDK | |
| MiniMax TTS | ⚠️ 待驗證 | |

**Provider 選擇**：`agent/tts_provider.py` + `agent/tts_registry.py`

### 語音辨識（Transcription）

**Provider**：`agent/transcription_provider.py` + `agent/transcription_registry.py`

### Browser Automation

| Provider | 整合 | 檔案 |
|----------|------|------|
| Browser Use | Python 套件 | `tools/browser_tool.py` |
| Browserbase | Managed browser service | `tools/browser_supervisor.py` |
| Camofox | 自定義 Firefox | `tools/browser_camofox.py` |

### Execution Backends（Terminal Backends）

| Backend | 整合 | 檔案 |
|---------|------|------|
| Local | 直接 subprocess | `tools/terminal_tool.py` |
| Docker | Docker SDK | `tools/environments/` |
| SSH | Paramiko/SSH | `tools/environments/` |
| Singularity | HPC 容器 | `tools/environments/` |
| Modal | `modal==1.3.4` | serverless |
| Daytona | `daytona==0.155.0` | serverless sandbox |

### Messaging Platforms（Gateway）

| 平台 | 檔案 |
|------|------|
| Telegram | `gateway/platforms/telegram.py` |
| Discord | ⚠️ 未在 platforms/ 中找到獨立檔案，可能在 `plugins/platforms/` |
| Slack | `gateway/platforms/slack.py` |
| WhatsApp | `gateway/platforms/whatsapp.py` |
| Signal | `gateway/platforms/signal.py` |
| Matrix | `gateway/platforms/matrix.py` |
| Email | `gateway/platforms/email.py` |
| DingTalk | `gateway/platforms/dingtalk.py` |
| Feishu/Lark | `gateway/platforms/feishu.py` |
| WeCom | `gateway/platforms/wecom.py` |
| WeChat | `gateway/platforms/weixin.py` |
| Yuanbao | `gateway/platforms/yuanbao.py` |
| BlueBubbles | `gateway/platforms/bluebubbles.py` |
| SMS | `gateway/platforms/sms.py` |
| QQ Bot | `gateway/platforms/qqbot/` |
| Microsoft Graph | `gateway/platforms/msgraph_webhook.py` |
| Generic Webhook | `gateway/platforms/webhook.py` |
| HTTP API | `gateway/platforms/api_server.py` |

### MCP（Model Context Protocol）

**雙向整合**：
1. **MCP Client**：`tools/mcp_tool.py` — Hermes 作為 MCP client，連接外部 MCP server
2. **MCP Server**：`mcp_serve.py` — Hermes 自身作為 MCP server，暴露工具給外部 client

### ACP（Agent Client Protocol）

**編輯器整合**：`acp_adapter/` 目錄
- Hermes 作為 ACP server，讓 Zed/JetBrains/Neovim 等編輯器直接使用
- ACP session 與編輯器 cwd 綁定
- 使用 `hermes-acp` toolset（精簡工具集，適合編輯器工作流）

### Honcho（User Modeling）

```python
# plugins/memory/honcho_provider.py（⚠️ 確認路徑）
class HonchoProvider(MemoryProvider):
    # dialectic user modeling
    # async prefetch + sync
```

URL：https://github.com/plastic-labs/honcho

### Skills Hub（agentskills.io）

```python
# agent/skills_hub.py
# GitHub App JWT auth（PyJWT）
# 從 agentskills.io 搜尋/下載/發布 skills
```

### Microsoft / Google Services

| 服務 | 整合 | 工具 |
|------|------|------|
| Microsoft Graph | `tools/microsoft_graph_client.py` | Office 365、Teams |
| Microsoft Graph Auth | `tools/microsoft_graph_auth.py` | OAuth |
| Google OAuth | `agent/google_oauth.py` | Google 服務認證 |
| Feishu Docs | `tools/feishu_doc_tool.py`, `tools/feishu_drive_tool.py` | Feishu 文件/雲盤 |

## 失敗處理機制

| 機制 | 實作 | 適用場景 |
|------|------|---------|
| Jittered backoff retry | `agent/retry_utils.py::jittered_backoff()` | LLM API 呼叫失敗 |
| Provider failover | `agent/error_classifier.py::FailoverReason` | Provider 錯誤 |
| Lazy dependency install | `tools/lazy_deps.py` | 缺少可選依賴 |
| TCP connection cleanup | `agent._cleanup_dead_connections()` | Stale socket |
| Ollama context limit | `_ollama_context_limit_error()` | Local model context |
| Tool result size limit | `tools/tool_result_storage.py::enforce_turn_budget()` | 工具輸出過大 |
| Image size limit | Pillow 自動縮放（`Pillow==12.2.0`） | 圖像過大 |

**Retry 策略**：tenacity + 自定義 jitter（decorrelated jitter，防止 thundering herd）

**Circuit breaker**：⚠️ 未在程式碼中找到顯式 circuit breaker 實作，主要靠 retry + failover 組合

## 第三方 Credential 管理

`tools/credential_files.py`：管理各外部服務的 API key
`agent/credential_persistence.py`：跨 session credential 持久化
`agent/credential_pool.py`：Credential rotation（多個 API key 輪換）
`agent/credential_sources.py`：從環境變數、config.yaml、secret 服務讀取 credential
