# Stage 2.5 — 外部整合

## LLM 提供商整合

### 支援的提供商（run_agent.py）

| 提供商 | 識別方式 | API 模式 | 特殊處理 |
|--------|---------|---------|---------|
| OpenRouter | `provider="openrouter"` / URL 含 `openrouter.ai` | chat_completions | providers_allowed/ignored/order 參數 |
| Anthropic | `provider="anthropic"` / URL 含 `api.anthropic.com` | anthropic_messages 或 chat_completions | prompt caching, native cache layout |
| OpenAI Codex | `api_mode="codex_responses"` | codex_responses | Responses API |
| Nous Portal | `provider="nous"` | chat_completions | 速率限制守衛（nous_rate_guard） |
| xAI / Grok | `provider="xai"` / URL 含 `api.x.ai` | chat_completions | - |
| Google Gemini | `provider="google-gemini-cli"` / URL `cloudcode-pa://` | - | 原生 Gemini adapter |
| AWS Bedrock | `provider="bedrock"` | - | boto3 + `agent/bedrock_adapter.py` |
| Copilot ACP | `provider="copilot-acp"` / URL `acp://copilot` | - | stdio JSON-RPC |
| GLM / z.ai | URL 含 `z.ai` | chat_completions | 特殊 schema 前處理 |
| Kimi / Moonshot | URL `api.kimi.com` / `api.moonshot.ai` | chat_completions | `agent/moonshot_schema.py` |
| 本地 / Ollama | 本地 IP / `ollama.com` | chat_completions | 無 stale timeout |
| 任何 OpenAI 相容端點 | 自訂 `base_url` | chat_completions | - |

### Provider 解析流程（run_agent.py:~1059）

```
(provider, api_key, base_url) 輸入
    ↓
1. 明確 provider 標籤優先（"anthropic", "nous" 等）
2. base_url hostname 推斷（api.anthropic.com → anthropic）
3. API key 前綴推斷（sk-ant- → anthropic）
4. 預設 → openrouter
    ↓
api_mode 決定（"chat_completions" / "anthropic_messages" / "codex_responses"）
    ↓
特殊轉接器啟動（bedrock_adapter, gemini_native_adapter 等）
```

### 失敗處理

```
API 呼叫失敗
    ↓
agent/error_classifier.py:classify_api_error()
    │
    ├── RATE_LIMIT → jittered_backoff() 等待 + 重試
    ├── OVERLOAD → 短暫等待 + 重試
    ├── AUTH_ERROR → 嘗試刷新 OAuth token / fallback
    ├── CONTEXT_LENGTH → 觸發 context_compressor
    └── NETWORK_ERROR → jittered_backoff() + 最終 fallback
    
若主提供商所有重試耗盡：
    _try_activate_fallback() → 切換到 config.yaml fallback.chain
```

**Jittered Backoff**（agent/retry_utils.py）：
- Decorrelated jitter（非純指數）防止 thundering herd
- 多個 gateway sessions 同時重試時不會在同一時刻衝擊提供商

---

## Web 搜尋整合（tools/web_tools.py）

### Backend 優先順序

```
config.yaml web.backend 設定
    │
    ├── "parallel" → 同時查詢多個後端，取最快結果
    ├── "firecrawl" → Firecrawl API（預設，若有 key）
    ├── "exa"       → Exa Search API
    ├── "tavily"    → Tavily Search API（⚠️ 未驗證是否實作）
    └── 自動偵測：有哪個 key 就用哪個
```

| 服務 | 環境變數 | 用途 |
|------|---------|------|
| Exa | `EXA_API_KEY` | 語意搜尋（`exa-py>=2.9.0`） |
| Firecrawl | `FIRECRAWL_API_KEY` 或 `FIRECRAWL_API_URL` | 搜尋 + 爬蟲（`firecrawl-py>=4.16.0`） |
| parallel-web | - | 輕量平行 web 搜尋（`parallel-web>=0.4.2`） |

---

## 瀏覽器自動化（tools/browser_tool.py）

```
browser_navigate() → Playwright chromium
    ├── 本地執行（PLAYWRIGHT_BROWSERS_PATH=/opt/hermes/.playwright）
    ├── 無頭模式
    ├── browser_supervisor.py：管理瀏覽器實例生命週期
    └── browser_camofox.py：偽裝瀏覽器指紋（反爬蟲偵測）
```

---

## 語音整合

### TTS（Text-to-Speech）

| 方案 | 啟用條件 | 套件 |
|------|---------|------|
| Edge TTS | 免費，預設 | `edge-tts>=7.2.7` |
| ElevenLabs | `ELEVENLABS_API_KEY` 設定 | `elevenlabs>=1.0`（`[tts-premium]` extra）|

### STT（Speech-to-Text）

| 方案 | 啟用條件 | 套件 |
|------|---------|------|
| faster-whisper | 本地 GPU/CPU | `faster-whisper>=1.0.0`（`[voice]` extra）|
| OpenAI Whisper API | `OPENAI_API_KEY` 設定 | 透過 openai SDK |

---

## 圖片生成（tools/image_generation_tool.py）

```
image_generate(prompt) →
    agent/image_routing.py
        │
        ├── fal.ai（`FAL_KEY` 設定）— `fal-client>=0.13.1`
        └── 其他 image_gen providers（plugins/image_gen/）
```

---

## MCP 整合（tools/mcp_tool.py）

```
啟動時：mcp_tool.py 讀取 config.yaml mcp.servers
    ↓
tools/mcp_tool.py 發起 MCP 連接
    ├── stdio transport：subprocess + stdin/stdout JSON-RPC
    └── HTTP transport：httpx + SSE
    ↓
發現工具 → registry.register() 動態注入
    ↓
動態 toolset "mcp-<server>" 建立
    ↓
notifications/tools/list_changed → 執行期動態更新工具清單
```

### MCP OAuth（tools/mcp_oauth.py + mcp_oauth_manager.py）

支援 MCP 伺服器的 OAuth 2.0 流程（如 GitHub MCP、Google MCP 等）。

---

## 訊息平台整合（gateway/platforms/）

| 平台 | 套件 | 特殊需求 |
|------|------|---------|
| Telegram | `python-telegram-bot>=22.6` | Bot Token |
| Discord | `discord.py>=2.7.1` | Bot Token |
| Slack | `slack-bolt>=1.18.0`, `slack-sdk>=3.27.0` | App Token + Bot Token |
| WhatsApp | baileys 橋接（Node.js）| - |
| Signal | signal-cli 橋接 | - |
| Matrix | `mautrix[encryption]>=0.20` | homeserver + access token |
| Email | SMTP/IMAP（stdlib） | 帳號密碼 |
| SMS | `aiohttp`（Twilio REST）| `TWILIO_*` 設定 |
| DingTalk | `dingtalk-stream>=0.20` | - |
| Feishu/Lark | `lark-oapi>=1.5.3` | - |
| WeCom | HTTP callback | - |
| BlueBubbles | HTTP API | - |
| Home Assistant | `aiohttp` | `HASS_TOKEN` |
| API Server | `fastapi + uvicorn` | OpenAI 相容端點 |
| Webhook | `aiohttp` | 自訂 webhook |
| Microsoft Teams | HTTP + Bot Framework | `TEAMS_*` 設定（Docker only）|

---

## 記憶體提供者整合

| 提供商 | 套件 | 機制 |
|--------|------|------|
| Honcho | `honcho-ai>=2.0.1` | 辯證推理 + 用戶建模 |
| Mem0 | `mem0ai` | 結構化記憶 |
| Supermemory | REST API | 雲端記憶 |
| Hindsight | REST API | 事後分析型記憶 |
| OpenViking | REST API | ⚠️ 未驗證 |
| RetainDB | REST API | ⚠️ 未驗證 |
| ByteRover | REST API | ⚠️ 未驗證 |

---

## 工作環境後端（tools/environments/）

| 後端 | 套件 | 特性 |
|------|------|------|
| Local | stdlib | 直接本地執行 |
| Docker | docker CLI | 容器隔離 |
| SSH | paramiko 或 openssh | 遠端執行 |
| Modal | `modal>=1.0.0` | 無伺服器，閒置成本低 |
| Daytona | `daytona>=0.148.0` | 沙盒持久化環境 |
| Singularity | singularity CLI | HPC 叢集 |
| Vercel | `vercel>=0.5.7` | ⚠️ 列在 pyproject.toml，tools/environments/ 中需驗證 |

---

## RL 訓練整合（environments/）

```
Atropos API Server（NousResearch/atropos）
    ↓
environments/*.py 實作 Environment ABC
    │
    ├── task 定義（提示 + 評分函式）
    ├── 互動：AIAgent 執行 task
    └── score → Tinker GRPO 訓練
    
Tinker（thinking-machines-lab/tinker）
    ↓
LoRA adapter 訓練（GRPO）
    ↓
更新後的模型用於下一個 rollout batch
```

---

## 失敗處理策略彙整

| 失敗類型 | 策略 | 實作位置 |
|---------|------|---------|
| API rate limit | Jittered backoff + fallback chain | agent/retry_utils.py, run_agent.py |
| API overload | 短暫等待 + 重試 | run_agent.py |
| Auth failure | OAuth refresh + fallback | agent/retry_utils.py |
| Context too long | Context compression | agent/context_compressor.py |
| Network error | Jittered backoff | agent/retry_utils.py |
| Tool failure | 返回錯誤 JSON → agent 自行處理 | tools/registry.py |
| Process crash | atexit cleanup + checkpoint | tools/process_registry.py |
| Browser crash | browser_supervisor.py 自動重啟 | tools/browser_supervisor.py |
