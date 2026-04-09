# External Integrations — Hermes Agent

## LLM Provider 整合

### 統一 API 層（OpenAI SDK）

所有 LLM provider 透過 `openai.OpenAI` client 呼叫，實現 provider agnostic 設計：

```python
# run_agent.py 初始化
self.client = openai.OpenAI(
    api_key=api_key,
    base_url=base_url,  # 可指向任何 OpenAI-compatible endpoint
    timeout=httpx.Timeout(...)
)
```

### 支援的 Provider（`hermes_cli/providers.py` + `.env.example`）

| Provider | Key ENV | Base URL |
|---------|---------|---------|
| OpenRouter | `OPENROUTER_API_KEY` | `https://openrouter.ai/api/v1` |
| Nous Portal | OAuth / `NOUS_API_KEY` | `https://inference-api.nousresearch.com/v1` |
| Anthropic direct | `ANTHROPIC_API_KEY` | `https://api.anthropic.com` |
| OpenAI direct | `OPENAI_API_KEY` | `https://api.openai.com/v1` |
| OpenAI Codex | OAuth (`hermes login`) | Responses API path |
| Google AI Studio | `GOOGLE_API_KEY` / `GEMINI_API_KEY` | Google's OpenAI-compatible endpoint |
| z.ai / GLM | `GLM_API_KEY` | `https://api.z.ai/api/paas/v4` |
| Kimi / Moonshot | `KIMI_API_KEY` | `https://api.kimi.com/coding/v1` |
| MiniMax | `MINIMAX_API_KEY` | `https://api.minimax.io/v1` |
| Hugging Face | `HF_TOKEN` | HF Inference Providers endpoint |
| GitHub Copilot | `GITHUB_TOKEN` | Copilot API |
| Local (Ollama, vLLM) | N/A | `http://localhost:11434/v1` |

### 失敗處理

```
Retry 機制（agent/retry_utils.py）：
  - Jittered exponential backoff（429 / 5xx errors）
  - 最大重試次數：可設定
  
Credential Pool（agent/credential_pool.py）：
  - 多個 API key 輪換
  - 失效 key 標記 + 自動切換
  - OAuth token sync（Codex pool）

Fallback Provider（config.yaml:fallback_providers）：
  - 主 provider 失敗 → 試 fallback 清單
  - 402（Payment Required）→ 自動換 provider
```

---

## Web 搜尋與抓取整合

### Exa（`tools/web_tools.py`）

```python
import exa_py
client = exa_py.Exa(api_key=EXA_API_KEY)
# AI-native 語意搜尋
results = client.search(query, num_results=10, use_autoprompt=True)
```

### Firecrawl（`tools/web_tools.py`）

```python
from firecrawl import FirecrawlApp
app = FirecrawlApp(api_key=FIRECRAWL_API_KEY)
# 動態網頁抓取 + Markdown 轉換
result = app.scrape_url(url, params={"formats": ["markdown"]})
```

### Parallel Web（`tools/web_tools.py`）

```
import parallel_web
# 並行多 URL 抓取，自動選擇最快 extractor
```

失敗處理：Firecrawl 失敗 → fallback 到輔助 LLM 直接擷取網頁。

---

## 訊息平台整合

### 架構：Gateway + Adapter 模式

```
GatewayRunner（gateway/run.py）
  ├─ asyncio event loop（單 process，多 platform 並行）
  ├─ 每 platform → 獨立 Adapter 實例
  └─ 共享 SessionStore + DeliveryRouter
```

### Telegram（`gateway/platforms/telegram.py`）

```python
from telegram.ext import Application, MessageHandler
app = Application.builder().token(TELEGRAM_BOT_TOKEN).build()
# Webhook 或 polling 模式
# 語音訊息 → 自動轉錄（faster-whisper）
# 批准按鈕 → InlineKeyboardMarkup
```

### Discord（`gateway/platforms/discord.py`）

```python
import discord
client = discord.Client(intents=discord.Intents.default())
# Voice memo transcription
# Channel controls + ignored channels
```

### Slack（`gateway/platforms/slack.py`）

```python
from slack_bolt import App
app = App(token=SLACK_BOT_TOKEN, signing_secret=SLACK_SIGNING_SECRET)
# Thread context preservation
# Socket Mode 或 HTTP mode
```

### WhatsApp（`gateway/platforms/whatsapp.py` + `scripts/whatsapp-bridge/`）

WhatsApp 使用 Node.js bridge（`scripts/whatsapp-bridge/`），Python 側透過 HTTP 與 bridge 通訊。
Bridge 使用非官方 API — ⚠️ 未驗證穩定性。

### 其他平台

| 平台 | 實作方式 | 特殊說明 |
|------|---------|---------|
| Signal | REST API（signal-cli） | `SIGNAL_ACCOUNT` + `SIGNAL_HTTP_URL` |
| Matrix | matrix-nio[e2e] | 需要 E2E encryption（olm 依賴複雜） |
| Email | aiohttp | IMAP 收 + SMTP 發 |
| Mattermost | REST API + WebSocket | `MATTERMOST_HOME_CHANNEL` |
| Home Assistant | aiohttp | `HASS_TOKEN` + webhook |
| DingTalk | dingtalk-stream | `DINGTALK_CLIENT_ID` + `DINGTALK_CLIENT_SECRET` |
| Feishu/Lark | lark-oapi | `FEISHU_APP_ID` + `FEISHU_APP_SECRET` |
| WeCom | aiohttp | `WECOM_BOT_ID` + `WECOM_SECRET` |
| SMS | aiohttp | 第三方 SMS 閘道 |

---

## MCP (Model Context Protocol) 整合（`tools/mcp_tool.py`）

```
MCP client 支援：
  - stdio transport（本地 MCP server）
  - HTTP transport（遠端 MCP server）
  - OAuth 2.1 PKCE 認證（v0.8.0 新增）
  - OSV 惡意軟體掃描（安裝 MCP 擴充前）
  - Resource + Prompt 發現
  - Sampling（server 發起 LLM 請求）
  - 自動重連

設定路徑：~/.hermes/mcp.json（或 config.yaml mcp 區段）
```

### ACP (Agent Client Protocol)（`acp_adapter/`）

```
ACP server 讓 editor 整合（VS Code / Zed / JetBrains）可以：
1. 向 Hermes agent 發送工具呼叫請求
2. Editor 的 MCP 生態系直接流入 agent tool 清單
3. hermes-acp CLI 啟動 FastAPI/uvicorn HTTP server
```

---

## 圖片生成整合（`tools/image_generation_tool.py`）

```python
import fal_client
result = fal_client.run(
    "fal-ai/flux/schnell",   # 或其他 fal.ai 模型
    arguments={"prompt": text, "image_size": "landscape_4_3"},
)
```
需要 `FAL_KEY` env var。

---

## Text-to-Speech 整合（`tools/tts_tool.py`）

```python
# Edge TTS（免費，無需 API key）
import edge_tts
communicate = edge_tts.Communicate(text, voice)
await communicate.save(path)

# ElevenLabs（premium，需 ELEVENLABS_API_KEY）
from elevenlabs import generate, play
audio = generate(text=text, voice=voice, model="eleven_multilingual_v2")
```

---

## Honcho AI 記憶整合（`plugins/honcho_plugin/` + `docs/honcho-integration-spec.md`）

```
honcho-ai SDK（plugin，可選）
  ├─ dialectic 推理：跨 session 用戶行為建模
  ├─ prefetch_all(query) → 每 turn 取得相關記憶
  ├─ sync(user_msg, assistant_msg) → 儲存對話
  └─ host/peer resolution（per HERMES_HOME profile）
```

---

## 語音識別整合（`tools/transcription_tools.py`）

```python
# faster-whisper（本地，需 faster-whisper 安裝）
from faster_whisper import WhisperModel
model = WhisperModel("base", device="cpu")
segments, info = model.transcribe(audio_file)
```

Gateway 在收到語音訊息時自動轉錄（Telegram / Discord / WhatsApp / Slack）。

---

## RL 訓練整合（`tools/rl_training_tool.py` + `environments/`）

```
Atropos（軌跡 API server）
  └─ atroposlib（git dependency）
  └─ RL environments（environments/*.py）
  
Tinker（訓練 service）
  └─ tinker（git submodule）
  └─ LoRA + GRPO 訓練
  
WandB（實驗追蹤）
  └─ wandb SDK（optional）
  └─ run 監控 + metrics
  
FastAPI + uvicorn
  └─ RL environment HTTP API
```

---

## 瀏覽器自動化整合（`tools/browser_tool.py`）

```
Browserbase（雲端）
  └─ browser_providers/browserbase.py
  └─ BROWSERBASE_API_KEY + BROWSERBASE_PROJECT_ID
  
Playwright（本地，需 npx playwright install chromium）
  └─ browser_providers/playwright.py
  └─ Camofox（browser fingerprint 管理，optional）
  
工具集：browser_navigate, browser_snapshot, browser_click,
         browser_type, browser_scroll, browser_press,
         browser_vision, browser_console, browser_get_images
```

---

## 安全與合規

### SSRF 防護（`tools/url_safety.py`）

```python
# 阻擋 private IP ranges、localhost、link-local 等
def check_url_safety(url: str) -> bool:
    # IPv4 private ranges: 10.x, 172.16-31.x, 192.168.x
    # IPv6 loopback, link-local, etc.
```

### OSV 漏洞掃描（`tools/osv_check.py`）

安裝 MCP 擴充前掃描 OSV 資料庫。

### Credential Redaction（`agent/redact.py`）

日誌輸出前自動脫敏 API keys、token、password。
