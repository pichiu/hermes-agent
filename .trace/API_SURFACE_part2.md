# Hermes Agent API 與介面參考文件 — Part 2

> **Part 1**: [API_SURFACE_part1.md](./API_SURFACE_part1.md)（CLI 命令 + Slash Commands）
> 版本：v0.8.0 (v2026.4.8)　　來源：`tools/`, `toolsets.py`, `run_agent.py`, `gateway/platforms/api_server.py`, `acp_adapter/server.py`

---

## 3. Tool Schemas（LLM 可用的 Agent Tools）

來源：`tools/registry.py`（singleton）、各 `tools/*.py` 在 import 時呼叫 `registry.register()`。

### 3.1 完整工具清單

| # | 工具名稱 | Toolset | 說明 | 主要參數 |
|---|---------|---------|------|---------|
| 1 | `terminal` | `terminal` | 跨後端命令執行（local / Docker / SSH / Modal / Daytona / Singularity） | `command` (str), `background` (bool), `timeout` (int), `workdir` (str) |
| 2 | `process` | `terminal` | 管理背景程序（list / kill / output） | `action` (str: list\|kill\|output), `pid` (int) |
| 3 | `read_file` | `file` | 讀取檔案（支援 offset + limit 分頁，無大小上限） | `path` (str), `offset` (int), `limit` (int) |
| 4 | `write_file` | `file` | 寫入檔案（原子寫，最大 100,000 字元） | `path` (str), `content` (str) |
| 5 | `patch` | `file` | 模糊匹配的 patch 應用（fuzzy diff）| `path` (str), `diff` (str) |
| 6 | `search_files` | `file` | 搜尋檔案內容或路徑（ripgrep 風格） | `pattern` (str), `path` (str), `type` (str) |
| 7 | `web_search` | `web` | AI-native 語意網路搜尋（Exa / Tavily / Parallel Web） | `query` (str) |
| 8 | `web_extract` | `web` | 從 URL 抓取網頁內容（Markdown 格式，支援 PDF） | `urls` (list[str], max 5) |
| 9 | `vision_analyze` | `vision` | 分析圖片（支援 URL / base64 / 本地路徑） | `image` (str), `prompt` (str) |
| 10 | `image_generate` | `image_gen` | 以 fal.ai（FLUX）生成圖片 | `prompt` (str), `image_size` (str) |
| 11 | `browser_navigate` | `browser` | 瀏覽器導航至 URL | `url` (str) |
| 12 | `browser_snapshot` | `browser` | 擷取頁面截圖或 DOM snapshot | `format` (str: screenshot\|dom) |
| 13 | `browser_click` | `browser` | 點擊頁面元素 | `selector` (str) |
| 14 | `browser_type` | `browser` | 在輸入框輸入文字 | `selector` (str), `text` (str) |
| 15 | `browser_scroll` | `browser` | 滾動頁面 | `direction` (str), `amount` (int) |
| 16 | `browser_back` | `browser` | 返回上一頁 | — |
| 17 | `browser_press` | `browser` | 按下鍵盤按鍵 | `key` (str) |
| 18 | `browser_get_images` | `browser` | 取得頁面所有圖片 URL | — |
| 19 | `browser_vision` | `browser` | 對頁面截圖進行視覺分析 | `prompt` (str) |
| 20 | `browser_console` | `browser` | 讀取或執行瀏覽器 console | `code` (str) |
| 21 | `skills_list` | `skills` | 列出可用 skills（含描述） | `filter` (str, optional) |
| 22 | `skill_view` | `skills` | 讀取特定 skill 的內容 | `name` (str) |
| 23 | `skill_manage` | `skills` | 建立/編輯/刪除 skill 文件 | `action` (str: create\|edit\|delete), `name` (str), `content` (str) |
| 24 | `memory` | `memory` | 讀寫跨 session 持久記憶（MEMORY.md + USER.md） | `action` (str: add\|replace\|remove), `target` (str: memory\|user), `content` (str) |
| 25 | `todo` | `todo` | 任務清單管理（多步驟工作規劃與追蹤） | `action` (str: add\|update\|complete\|list), `item` (str) |
| 26 | `session_search` | `session_search` | 全文搜尋過去對話（SQLite FTS5） | `query` (str), `limit` (int) |
| 27 | `clarify` | `clarify` | 向用戶提問（多選或開放式） | `question` (str), `choices` (list[str], optional) |
| 28 | `execute_code` | `code_execution` | 在 sandbox 中執行 Python 腳本（可呼叫其他工具） | `code` (str), `language` (str, default: python) |
| 29 | `delegate_task` | `delegation` | 派遣 subagent 執行隔離子任務 | `goal` (str) OR `tasks` (list), `context` (str) |
| 30 | `cronjob` | `cronjob` | 管理 cron 排程（新增/列出/暫停/繼續/刪除/觸發） | `action` (str), `schedule` (str, cron expression), `command` (str) |
| 31 | `send_message` | `messaging` | 跨平台訊息發送（Telegram / Discord / Slack / SMS 等） | `platform` (str), `target` (str), `message` (str) |
| 32 | `text_to_speech` | `tts` | 文字轉語音（Edge TTS 免費 / ElevenLabs premium） | `text` (str), `voice` (str), `engine` (str) |
| 33 | `ha_list_entities` | `homeassistant` | 列出 Home Assistant 裝置實體 | `domain` (str, optional) |
| 34 | `ha_get_state` | `homeassistant` | 取得 Home Assistant 實體狀態 | `entity_id` (str) |
| 35 | `ha_list_services` | `homeassistant` | 列出可呼叫的 Home Assistant services | `domain` (str, optional) |
| 36 | `ha_call_service` | `homeassistant` | 呼叫 Home Assistant service | `domain` (str), `service` (str), `data` (dict) |
| 37 | `mixture_of_agents` | `moa` | 進階推理（多 agent 協作，Mixture of Agents） | `prompt` (str), `models` (list[str]) |
| 38 | `rl_list_environments` | `rl` | 列出 RL 訓練環境 | — |
| 39 | `rl_select_environment` | `rl` | 選擇 RL 訓練環境 | `name` (str) |
| 40 | `rl_get_current_config` | `rl` | 讀取當前 RL 訓練設定 | — |
| 41 | `rl_edit_config` | `rl` | 編輯 RL 訓練設定 | `config` (dict) |
| 42 | `rl_start_training` | `rl` | 啟動 RL 訓練 | — |
| 43 | `rl_check_status` | `rl` | 查詢 RL 訓練狀態 | — |
| 44 | `rl_stop_training` | `rl` | 停止 RL 訓練 | — |
| 45 | `rl_get_results` | `rl` | 取得 RL 訓練結果 | — |
| 46 | `rl_list_runs` | `rl` | 列出所有 RL 訓練 run | — |
| 47 | `rl_test_inference` | `rl` | 測試已訓練模型的推理 | `prompt` (str) |

> **注意**：MCP 工具（`mcp_tool.py`）會在執行時根據設定動態注冊，名稱視 MCP server 而定，不在此靜態列表中。

### 3.2 Toolset 分類總覽

| Toolset | 工具數量 | 說明 | 啟用所需條件 |
|---------|---------|------|-------------|
| `terminal` | 2 | 命令執行 + 程序管理 | — |
| `file` | 4 | 讀寫/patch/搜尋檔案 | — |
| `web` | 2 | 網路搜尋 + 網頁抓取 | `EXA_API_KEY` 或 `FIRECRAWL_API_KEY` 等 |
| `vision` | 1 | 圖片分析 | Auxiliary LLM 支援 vision |
| `image_gen` | 1 | AI 圖片生成 | `FAL_KEY` |
| `browser` | 10 | 瀏覽器自動化 | Browserbase 或本地 Playwright |
| `skills` | 3 | Skill 文件 CRUD | — |
| `memory` | 1 | 跨 session 記憶 | — |
| `todo` | 1 | 任務追蹤 | — |
| `session_search` | 1 | 過去對話搜尋 | — |
| `clarify` | 1 | 向用戶提問 | clarify_callback 已設定 |
| `code_execution` | 1 | Python sandbox | — |
| `delegation` | 1 | Subagent 派遣 | — |
| `cronjob` | 1 | Cron 排程管理 | — |
| `messaging` | 1 | 跨平台訊息 | Gateway 執行中 |
| `tts` | 1 | 文字轉語音 | Edge TTS（免費）或 `ELEVENLABS_API_KEY` |
| `homeassistant` | 4 | 智慧家庭控制 | `HASS_TOKEN` |
| `moa` | 1 | Mixture of Agents 推理 | 多 LLM 設定 |
| `rl` | 10 | RL 訓練管理 | Tinker + Atropos 設定 |

---

## 4. AIAgent Python API

來源：`run_agent.py:AIAgent`（約第 437 行起）

### 4.1 建構子

```python
agent = AIAgent(
    # LLM 連線
    base_url: str = None,              # LLM API endpoint（預設從 config.yaml 讀取）
    api_key: str = None,               # API key（預設從 .env 讀取）
    provider: str = None,              # provider 識別字（openrouter / anthropic / ...）
    api_mode: str = None,              # "chat_completions" | "codex_responses" | "anthropic_messages"
    model: str = "",                   # 模型名稱（預設 anthropic/claude-opus-4.6）

    # 迭代控制
    max_iterations: int = 90,          # 最大工具呼叫輪次
    tool_delay: float = 1.0,           # 工具呼叫間隔（秒）

    # Toolset 控制
    enabled_toolsets: List[str] = None,   # 只啟用指定 toolsets
    disabled_toolsets: List[str] = None,  # 停用指定 toolsets

    # Session 管理
    session_id: str = None,            # 預設自動生成
    persist_session: bool = True,      # 是否儲存到 SessionDB
    prefill_messages: List[Dict] = None,  # 預填充對話歷史（few-shot）

    # 輸出控制
    quiet_mode: bool = False,          # 抑制進度輸出
    verbose_logging: bool = False,     # 詳細日誌

    # Callbacks（平台層注入）
    tool_progress_callback: callable = None,   # fn(tool_name, args_preview)
    tool_start_callback: callable = None,
    tool_complete_callback: callable = None,
    thinking_callback: callable = None,
    reasoning_callback: callable = None,
    clarify_callback: callable = None,         # fn(question, choices) -> str
    step_callback: callable = None,
    stream_delta_callback: callable = None,
    status_callback: callable = None,

    # 進階
    platform: str = None,              # "cli" | "telegram" | "discord" | "whatsapp" | ...
    ephemeral_system_prompt: str = None,
    save_trajectories: bool = False,
    reasoning_config: Dict = None,     # {"effort": "none"} 等
    max_tokens: int = None,
    skip_context_files: bool = False,  # batch 處理時使用
    skip_memory: bool = False,
    checkpoints_enabled: bool = False,
)
```

### 4.2 主要 Public Methods

| 方法 | 簽名 | 說明 | 回傳值 |
|------|------|------|--------|
| `run_conversation` | `(user_message, system_message=None, conversation_history=None, task_id=None, stream_callback=None, persist_user_message=None) -> Dict` | 執行完整對話（含工具呼叫迴圈） | `{"final_response": str, "messages": list, ...}` |
| `chat` | `(message, stream_callback=None) -> str` | 簡化介面，僅回傳最終文字 | `str` |
| `switch_model` | `(new_model, new_provider, api_key='', base_url='', api_mode='')` | 動態切換模型（本 session） | — |
| `reset_session_state` | `()` | 清除對話歷史，保留設定 | — |
| `interrupt` | `(message=None)` | 中斷當前工具呼叫 | — |
| `clear_interrupt` | `()` | 清除中斷旗標 | — |
| `is_interrupted` | `() -> bool` | 查詢中斷狀態 | `bool` |
| `get_activity_summary` | `() -> dict` | 取得最近活動摘要 | `{"last_activity": ..., "desc": str}` |
| `shutdown_memory_provider` | `(messages=None)` | 關閉記憶 provider（清理資源） | — |
| `flush_memories` | `(messages=None, min_turns=None)` | 強制將記憶寫入持久儲存 | — |

### 4.3 `batch_runner.py` 使用範例

```python
# batch_runner.py 典型用法
from run_agent import AIAgent

agent = AIAgent(
    model="anthropic/claude-opus-4.6",
    enabled_toolsets=["file", "terminal"],
    save_trajectories=True,
    skip_context_files=True,   # 避免污染 batch 資料
    quiet_mode=True,
)

result = agent.run_conversation(
    user_message="Implement feature X",
    conversation_history=[],   # 可注入 few-shot
)
final_text = result["final_response"]
```

---

## 5. Gateway API

### 5.1 REST API Server（`gateway/platforms/api_server.py`）

Hermes API Server 是 OpenAI-compatible HTTP server，透過 `hermes gateway` 或直接執行 `api_server.py` 啟動。

| 方法 | Endpoint | 說明 |
|------|----------|------|
| `GET` | `/health` | 健康檢查 |
| `GET` | `/v1/health` | 健康檢查（別名） |
| `GET` | `/v1/models` | 列出可用模型（hermes-agent） |
| `POST` | `/v1/chat/completions` | OpenAI Chat Completions 格式（stateless；可透過 `X-Hermes-Session-Id` header 啟用 session 連續性） |
| `POST` | `/v1/responses` | OpenAI Responses API 格式（stateful，透過 `previous_response_id` 鏈接） |
| `GET` | `/v1/responses/{response_id}` | 取得已儲存的 response |
| `DELETE` | `/v1/responses/{response_id}` | 刪除已儲存的 response |
| `POST` | `/v1/runs` | 啟動 agent run（立即回傳 `run_id`，狀態碼 202） |
| `GET` | `/v1/runs/{run_id}/events` | SSE 串流：結構化 lifecycle events |
| `GET` | `/api/jobs` | 列出所有 cron jobs |
| `POST` | `/api/jobs` | 建立新 cron job |
| `GET` | `/api/jobs/{job_id}` | 取得單一 cron job |
| `DELETE` | `/api/jobs/{job_id}` | 刪除 cron job |
| `POST` | `/api/jobs/{job_id}/pause` | 暫停 cron job |
| `POST` | `/api/jobs/{job_id}/resume` | 恢復暫停的 cron job |
| `POST` | `/api/jobs/{job_id}/run` | 立即觸發 cron job 執行 |

**Session 連續性 Header**：`X-Hermes-Session-Id: <uuid>` 可讓 stateless Chat Completions endpoint 維持跨請求 context。

**POST Body 大小限制**：1 MB（`MAX_REQUEST_BYTES = 1_000_000`）

### 5.2 ACP Server（`acp_adapter/`）

ACP（Agent Client Protocol）server 讓編輯器（VS Code / Zed / JetBrains）可呼叫 Hermes。

啟動：`hermes acp` 或 `hermes-acp`（`acp_adapter/entry.py:main()`），底層為 FastAPI/uvicorn HTTP server。

| ACP 方法 | 說明 |
|---------|------|
| `initialize` | 初始化 agent，回傳 AgentCapabilities |
| `authenticate` | 認證（OAuth / token） |
| `new_session` | 建立新 session |
| `load_session` | 載入已存在的 session |
| `resume_session` | 繼續 session |
| `fork_session` | 分支 session（探索路徑） |
| `cancel` | 取消執行中的 session |
| `list_sessions` | 列出所有 sessions |
| `prompt` | 發送訊息並取得回應（核心方法） |
| `set_session_model` | 切換 session 使用的模型 |
| `set_session_mode` | 設定 session 模式 |
| `set_config_option` | 設定設定項目 |

**ACP 支援的 Slash Commands**（editor 內可用）：

| 命令 | 說明 |
|------|------|
| `/help` | 顯示可用命令 |
| `/model` | 顯示/切換當前模型 |
| `/tools` | 列出可用工具 |
| `/context` | 顯示對話 context 資訊 |
| `/reset` | 清除對話歷史 |
| `/compact` | 壓縮對話 context |
| `/version` | 顯示版本資訊 |

### 5.3 Webhook 設定

各平台 webhook 設定路徑：`~/.hermes/config.yaml`

```yaml
gateway:
  telegram:
    token: "BOT_TOKEN"
    webhook_url: "https://example.com/webhook"  # 或使用 polling
  discord:
    token: "BOT_TOKEN"
  slack:
    bot_token: "xoxb-..."
    signing_secret: "..."
    socket_mode: true  # Socket Mode 或 HTTP mode
  webhook:
    # Generic webhook platform
    url: "https://example.com/hermes-webhook"
```

---

## 6. Error Handling Patterns

### 6.1 工具呼叫回應格式

來源：`tools/registry.py:tool_error()`, `tool_result()`

所有工具 handler 必須回傳 **JSON 字串**：

```python
# 成功回應（任意結構）
{"success": true, "data": {...}}
{"items": [...], "count": 42}

# 錯誤回應（固定格式）
{"error": "file not found"}
{"error": "bad input", "success": false}
{"error": "Tool execution failed: ValueError: ..."}
```

工具 helper 函式：

```python
from tools.registry import tool_error, tool_result

return tool_error("something went wrong")           # {"error": "..."}
return tool_error("not found", code=404)            # {"error": "...", "code": 404}
return tool_result(success=True, data=payload)       # {"success": true, "data": ...}
return tool_result({"key": "value"})                 # {"key": "value"}
```

### 6.2 常見錯誤情境對照

| 情境 | 回應格式 | 來源 |
|------|---------|------|
| 未知工具名稱 | `{"error": "Unknown tool: <name>"}` | `registry.dispatch()` |
| 工具執行拋出例外 | `{"error": "Tool execution failed: ExceptionType: message"}` | `registry.dispatch()` |
| check_fn 回傳 False | 工具不出現在 LLM tool 清單中（靜默排除） | `registry.is_available()` |
| check_fn 拋出例外 | 標記為不可用（debug log），不崩潰 | `registry.is_available()` |
| SSRF 防護拒絕 | `{"error": "URL not allowed: private/loopback address"}` | `tools/url_safety.py` |
| API 429 / 5xx | 透過 jittered exponential backoff 自動重試 | `agent/retry_utils.py` |
| Provider 402（付款要求） | 自動切換 fallback provider | `agent/credential_pool.py` |

### 6.3 API Server 錯誤回應

REST API（`/v1/chat/completions` 等）回傳標準 HTTP 狀態碼：

| 狀態碼 | 說明 |
|--------|------|
| `200` | 成功 |
| `202` | 已接受（`/v1/runs` 非同步啟動） |
| `400` | 請求格式錯誤 |
| `404` | resource 不存在（response_id / run_id） |
| `413` | Request body 超過 1 MB |
| `500` | Agent 執行錯誤 |

### 6.4 重試與 Fallback 機制

```
LLM 呼叫失敗
  └─ 429 / 5xx
       └─ agent/retry_utils.py: jittered exponential backoff
  └─ 402 (Payment Required)
       └─ agent/credential_pool.py: 切換 API key 輪換池
  └─ 所有 key 失效
       └─ config.yaml: fallback_providers 清單依序嘗試
  └─ Firecrawl web_extract 失敗
       └─ 自動 fallback 到 auxiliary LLM 直接擷取網頁
```

---

## 附錄：設定注入點

| 設定項 | 來源 | 注入位置 |
|--------|------|---------|
| `HERMES_HOME` | env var / `~/.hermes` | `hermes_constants.get_hermes_home()` |
| LLM model | `config.yaml model.default` | `AIAgent.__init__:model` |
| LLM provider | `config.yaml` / `HERMES_INFERENCE_PROVIDER` | `hermes_cli/runtime_provider.py` |
| API key | `~/.hermes/.env` | `load_hermes_dotenv()` |
| Enabled toolsets | `config.yaml` / CLI flag | `AIAgent.__init__:enabled_toolsets` |
| Terminal backend | `config.yaml terminal.backend` / `TERMINAL_ENV` | `terminal_tool.py` |
| Memory provider | `config.yaml memory.provider` / plugins | `AIAgent.__init__ -> MemoryManager` |
| Platform hint | `AIAgent.__init__:platform` | `_build_system_prompt()` |
| MCP servers | `~/.hermes/mcp.json` 或 `config.yaml mcp` | `tools/mcp_tool.py` |

