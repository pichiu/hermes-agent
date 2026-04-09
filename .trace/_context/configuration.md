# Configuration & Environment — Hermes Agent

## 設定載入優先順序

```
高優先  CLI flags / env overrides（--model, --toolset）
   ↓    HERMES_HOME env var（profile 選擇）
   ↓    ~/.hermes/.env（用戶 API keys，由 load_hermes_dotenv() 載入）
   ↓    ./.env（開發環境 fallback）
   ↓    ~/.hermes/config.yaml（主設定，model / terminal / display 等）
低優先  DEFAULT_CONFIG（hermes_cli/config.py:214）
```

---

## 設定檔結構（`cli-config.yaml.example`）

### 模型設定

```yaml
model:
  default: "anthropic/claude-opus-4.6"   # 預設 LLM 模型
  provider: "auto"                        # auto | openrouter | nous | anthropic | openai-codex | gemini | ...
  base_url: "https://openrouter.ai/api/v1"
  # api_key 建議放在 .env，不要放設定檔

provider_routing:                         # 僅 OpenRouter 有效
  sort: "throughput"                      # price | throughput | latency
  only: ["anthropic", "google"]          # 限定 provider
  data_collection: "deny"               # 排除儲存資料的 provider

smart_model_routing:
  enabled: false
  max_simple_chars: 160
  max_simple_words: 28
  cheap_model:
    provider: openrouter
    model: google/gemini-2.5-flash
```

### Agent 行為設定

```yaml
agent:
  max_turns: 90             # 最大工具呼叫 iteration 數
  gateway_timeout: 1800     # Gateway session 閒置逾時（秒），0 = 無限
  tool_use_enforcement: "auto"  # auto | true | false | [model substring list]
```

### Terminal 執行環境

```yaml
terminal:
  backend: "local"          # local | docker | ssh | modal | daytona | singularity
  cwd: "."                  # 工作目錄
  timeout: 180              # 命令逾時（秒）
  persistent_shell: true    # 跨命令保留 cwd/env
  
  # Docker 設定
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_volumes: ["/host/path:/container/path"]
  container_cpu: 1
  container_memory: 5120    # MB
  container_disk: 51200     # MB
  container_persistent: true
  
  # SSH 設定
  # ssh_host, ssh_user, ssh_port, ssh_key（設於 .env 或此處）
  
  # Modal 設定
  # modal_image, container_cpu, container_memory, container_disk
  
  env_passthrough: []       # 額外透傳的 env var 名稱
```

### 壓縮設定

```yaml
compression:
  enabled: true
  threshold: 0.50          # Context 使用超過 50% 時觸發壓縮
  target_ratio: 0.20       # 壓縮後保留 threshold 的 20% 作為 tail
  protect_last_n: 20       # 最少保留最近 20 條訊息
  summary_model: ""        # 空 = 使用主模型；可設為快/便宜模型
  summary_provider: "auto"
```

### 輔助 LLM 任務

```yaml
auxiliary:
  vision:                  # 圖片分析（vision_analyze tool）
    provider: "auto"
    model: ""              # 例如 "google/gemini-2.5-flash"
    timeout: 30
  web_extract:             # 網頁內容擷取（web_extract tool）
    provider: "auto"
    timeout: 360
  compression:             # Context 壓縮摘要
    provider: "auto"
    timeout: 120
  session_search:          # Session 搜尋摘要
    provider: "auto"
  skills_hub:              # Skills Hub 操作
    provider: "auto"
  approval:                # 危險命令分析
    provider: "auto"       # 建議使用快速/便宜模型
```

### Memory 設定

```yaml
memory:
  enabled: true            # MEMORY.md（長期記憶）
  user_profile: true       # USER.md（用戶資料）
  provider: "local"        # local | honcho | hybrid
  nudge_interval: 10       # N 個 turn 後提醒更新記憶
  
honcho:                    # 僅在 memory.provider=honcho|hybrid 時有效
  host: ""                 # Honcho API host（空 = 官方 cloud）
  environment_name: ""     # Honcho 環境名稱
```

### 顯示設定

```yaml
display:
  compact: false           # 緊湊模式
  personality: "kawaii"    # kawaii | minimal | professional
  skin: "default"          # 視覺主題（見 docs/skins/）
  streaming: false         # 串流顯示（CLI 下預設 false）
  show_reasoning: false    # 顯示 thinking tokens
  inline_diffs: true       # 檔案修改時顯示 diff 預覽
  show_cost: false         # 狀態欄顯示費用
  bell_on_complete: false  # 完成時發出 bell
```

### 排程（Cron）設定

```yaml
cron:
  enabled: true
  timezone: "Asia/Taipei"  # 排程使用的時區
```

### 安全設定

```yaml
security:
  redact_secrets: true     # 日誌中脫敏 secrets
  approved_commands:       # 預先批准的命令 pattern（正則）
    - "^git "
    - "^npm "
    - "^pytest "
```

---

## 環境變數參考（`.env.example` 摘要）

### LLM Provider Keys

| 變數 | Provider |
|------|---------|
| `OPENROUTER_API_KEY` | OpenRouter（首選） |
| `ANTHROPIC_API_KEY` | Anthropic 直連 |
| `OPENAI_API_KEY` | OpenAI 直連 |
| `GOOGLE_API_KEY` / `GEMINI_API_KEY` | Google AI Studio |
| `GLM_API_KEY` | z.ai / ZhipuAI GLM |
| `KIMI_API_KEY` | Kimi / Moonshot AI |
| `MINIMAX_API_KEY` | MiniMax（全球） |
| `MINIMAX_CN_API_KEY` | MiniMax（中國） |
| `HF_TOKEN` | Hugging Face |
| `NOUS_API_KEY` | Nous Portal API key |

### Tool API Keys

| 變數 | 工具 |
|------|------|
| `EXA_API_KEY` | Exa web search |
| `PARALLEL_API_KEY` | Parallel web search/extract |
| `FIRECRAWL_API_KEY` | Firecrawl web scraping |
| `FAL_KEY` | fal.ai 圖片生成 |
| `ELEVENLABS_API_KEY` | ElevenLabs TTS（premium） |
| `BROWSERBASE_API_KEY` | Browserbase 瀏覽器 |
| `BROWSERBASE_PROJECT_ID` | Browserbase 專案 |

### Gateway / Platform Keys

| 變數 | 平台 |
|------|------|
| `TELEGRAM_BOT_TOKEN` | Telegram |
| `DISCORD_BOT_TOKEN` | Discord |
| `SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` | Slack |
| `MATRIX_HOMESERVER` + `MATRIX_USERNAME` + `MATRIX_PASSWORD` | Matrix |
| `HASS_TOKEN` + `HASS_URL` | Home Assistant |
| `SIGNAL_ACCOUNT` + `SIGNAL_HTTP_URL` | Signal |
| `DINGTALK_CLIENT_ID` + `DINGTALK_CLIENT_SECRET` | DingTalk |
| `FEISHU_APP_ID` + `FEISHU_APP_SECRET` | Feishu/Lark |

### 系統行為

| 變數 | 說明 |
|------|------|
| `HERMES_HOME` | Hermes 主目錄（預設 `~/.hermes`） |
| `HERMES_INFERENCE_PROVIDER` | 覆蓋 config.yaml model.provider |
| `HERMES_QUIET` | `1` = 靜默模式（gateway 預設啟用） |
| `HERMES_EXEC_ASK` | `1` = 危險命令需審批（gateway 預設啟用） |
| `TERMINAL_ENV` | 覆蓋 terminal backend（local/docker/ssh/modal/daytona/singularity） |
| `TERMINAL_CWD` | 工作目錄 |
| `TERMINAL_TIMEOUT` | 命令逾時秒數 |
| `HERMES_TIMEZONE` | 時區（覆蓋 config.yaml） |
| `HERMES_MAX_ITERATIONS` | 最大 tool call 次數 |
| `HERMES_AGENT_TIMEOUT` | Gateway agent 閒置逾時 |
| `HERMES_REDACT_SECRETS` | `true/false`，日誌脫敏 |
| `HERMES_DUMP_REQUESTS` | `1` = debug 輸出 API requests |
| `HERMES_MANAGED` | `true/brew/nix` = package manager 管理模式 |
| `HERMES_ENABLE_PROJECT_PLUGINS` | `1` = 啟用 `./.hermes/plugins/` |

---

## 設定載入實作（`hermes_cli/env_loader.py`）

```python
def load_hermes_dotenv(hermes_home, project_env=None):
    """
    載入順序：
    1. ~/.hermes/.env（主要）
    2. ./.env（開發 fallback，僅在 ~/.hermes/.env 不存在時）
    
    注意：os.getenv() 中已存在的值不會被覆蓋
    （.env 不會覆蓋 shell export 的環境變數）
    """
```

---

## 設定驗證

`hermes_cli/config.py:print_config_warnings()` 在啟動時驗證設定結構，
已知錯誤格式會在 `~/.hermes/logs/errors.log` 中記錄警告。

`hermes doctor` 命令（`hermes_cli/doctor.py`）可診斷：
- Python 版本
- 依賴安裝狀態
- API key 設定
- Terminal backend 可用性
- Gateway 連線狀態

---

## Profiles（多設定切換）

```bash
# 建立 profile
hermes --profile coder  # 使用 ~/.hermes/profiles/coder/ 作為 HERMES_HOME

# 持久化預設 profile
echo "coder" > ~/.hermes/active_profile
```

Profile 解析在所有 module import 前執行（`hermes_cli/main.py:83`），
確保所有路徑都使用正確的 HERMES_HOME。

---

## Feature Flags（env var based）

| 變數 | 功能 |
|------|------|
| `HERMES_DUMP_REQUESTS` | Debug：輸出完整 API 請求 |
| `HERMES_ENABLE_PROJECT_PLUGINS` | 啟用專案級 plugin 目錄 |
| `HERMES_MANAGED` | 宣告 package manager 管理模式 |
| `HERMES_OPTIONAL_SKILLS` | 覆蓋 optional-skills 目錄路徑 |
| `HERMES_PORTAL_BASE_URL` | 覆蓋 Nous Portal base URL |
| `TERMINAL_LOCAL_PERSISTENT` | 本地 backend 啟用 persistent shell |

---

## 記憶相關設定

```yaml
# ~/.hermes/MEMORY.md — 長期記憶（純文字 Markdown）
# ~/.hermes/USER.md   — 用戶個人資料（純文字 Markdown）
# ~/.hermes/SOUL.md   — Agent 人格定義（覆蓋 DEFAULT_AGENT_IDENTITY）
# ~/.hermes/state.db  — SQLite session + FTS5 索引
# ~/.hermes/cron/jobs.json — Cron job 定義
# ~/.hermes/logs/agent.log — 結構化日誌
# ~/.hermes/logs/errors.log — 錯誤日誌
# ~/.hermes/auth.json — OAuth credentials
# ~/.hermes/skills/   — 用戶自訂 skills
# ~/.hermes/cache/    — 圖片 cache 等
```
