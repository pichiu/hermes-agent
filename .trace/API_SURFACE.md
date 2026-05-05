# API_SURFACE.md — Hermes Agent API 與介面參考文件

> 版本：0.12.0 | 維護者：Nous Research | 授權：MIT
> 本文件涵蓋 CLI 指令、斜線指令、Python Library API、Gateway OpenAI 相容 API、工具 API 與 Authentication 模型。

---

## 目錄

1. [CLI 指令參考](#1-cli-指令參考)
2. [斜線指令（Slash Commands）](#2-斜線指令slash-commands)
3. [Python Library API](#3-python-library-api)
4. [Gateway OpenAI 相容 API](#4-gateway-openai-相容-api)
5. [工具 API 概覽](#5-工具-api-概覽)
6. [Authentication 模型](#6-authentication-模型)
7. [Error Handling Pattern](#7-error-handling-pattern)

---

## 1. CLI 指令參考

進入點：`hermes` 腳本 → `hermes_cli/main.py:main()`

### 1.1 全域旗標

| 旗標 | 說明 |
|------|------|
| `--profile <name>` | 選擇 profile（覆寫 `HERMES_HOME`） |
| `--model <model>` | 指定模型（可覆寫 config.yaml） |
| `--version` | 顯示版本並退出 |

### 1.2 頂層子指令清單

| 子指令 | 說明 |
|--------|------|
| `hermes` （無參數） | 啟動互動式 CLI（等同 `hermes chat`） |
| `hermes chat` | 啟動互動式對話 CLI |
| `hermes model` | 管理預設模型設定 |
| `hermes fallback` | 設定 fallback model chain（add / remove / list / test） |
| `hermes gateway` | 管理訊息閘道（run / start / stop / restart / status / install / uninstall / setup） |
| `hermes setup` | 首次安裝精靈（API keys、模型設定） |
| `hermes whatsapp` | WhatsApp 橋接設定 |
| `hermes slack` | Slack App 安裝精靈（含 manifest 生成） |
| `hermes login` | OAuth / API key 登入 |
| `hermes logout` | 清除登入憑證 |
| `hermes auth` | 憑證池管理（add / list / remove / reset / status / logout / spotify） |
| `hermes status` | 顯示系統狀態 |
| `hermes cron` | 排程任務管理（list / create / edit / pause / resume / run / remove / status / tick） |
| `hermes webhook` | Webhook 管理（subscribe / list / remove / test） |
| `hermes hooks` | 閘道 hook 管理（list / test / revoke） |
| `hermes doctor` | 環境診斷 |
| `hermes dump` | 匯出 agent 狀態 |
| `hermes debug` | 上傳除錯報告並取得分享連結 |
| `hermes backup` | 備份 Hermes 設定與狀態 |
| `hermes import` | 匯入備份 |
| `hermes config` | 設定管理（show / edit / set / path / env-path / check / migrate） |
| `hermes pairing` | DM pairing 管理 |
| `hermes skills` | 技能管理（browse / search / install / inspect / list / check / update / audit / uninstall / reset / publish / snapshot） |
| `hermes mcp` | MCP 伺服器管理 |
| `hermes sessions` | 會話管理（list / export / delete / prune / stats / rename / browse） |
| `hermes insights` | 用量分析與統計 |
| `hermes claw` | Claw 工具（migrate / cleanup） |
| `hermes version` | 顯示版本資訊 |
| `hermes update` | 更新 Hermes Agent 至最新版本 |
| `hermes uninstall` | 卸載 Hermes |
| `hermes acp` | 啟動 ACP 適配器（VS Code / Zed / JetBrains） |
| `hermes profile` | Profile 管理（list / use / create / delete / show / alias / rename / export / import） |
| `hermes completion` | Shell 自動補全安裝 |
| `hermes dashboard` | 啟動 Web 儀表板（fastapi + uvicorn） |
| `hermes logs` | 查看 agent 日誌 |

### 1.3 hermes chat 常用旗標

```
hermes [chat] [QUERY]
  --model MODEL          覆寫本次對話模型
  --session SESSION_ID   續接指定會話
  --toolset TOOLSET      限定啟用的 toolset（逗號分隔）
  --no-tools             停用所有工具
  --quiet                靜音模式（無進度輸出）
  --save-trajectories    儲存對話軌跡（JSONL 格式）
```

---

## 2. 斜線指令（Slash Commands）

在互動式 CLI 中，以 `/` 開頭輸入斜線指令。定義來源：`hermes_cli/commands.py:COMMAND_REGISTRY`。

### 2.1 Session 管理

| 指令 | 說明 |
|------|------|
| `/new` | 開始新會話（新 session ID + 清空歷史） |
| `/clear` | 清除畫面並開始新會話 |
| `/redraw` | 強制重繪 UI（修復終端機漂移） |
| `/history` | 顯示對話歷史 |
| `/save` | 儲存當前對話 |
| `/retry` | 重試最後一則訊息 |
| `/undo` | 移除最後一組 user/assistant 交換 |
| `/title <標題>` | 設定當前會話標題 |
| `/branch` | 分支當前會話（探索不同路徑） |
| `/compress` | 手動壓縮對話 context |
| `/rollback` | 列出或還原檔案系統 checkpoints |
| `/snapshot` | 建立或還原 Hermes 設定/狀態快照 |
| `/stop` | 終止所有背景行程 |
| `/approve` | 核准待審的危險指令 |
| `/deny` | 拒絕待審的危險指令 |
| `/background <prompt>` | 在背景執行 prompt |
| `/agents` | 顯示活躍 agent 與執行中任務 |
| `/queue <prompt>` | 排入下一輪 prompt（不中斷當前） |
| `/steer <message>` | 在下次工具呼叫後注入訊息 |
| `/goal <目標>` | 設定跨回合的持續目標 |
| `/status` | 顯示會話資訊 |
| `/sethome` | 設定此聊天為 home channel |
| `/resume <name>` | 續接先前命名的會話 |
| `/restart` | 優雅重啟 gateway（排完執行中任務） |

### 2.2 Configuration

| 指令 | 說明 |
|------|------|
| `/config` | 顯示當前設定 |
| `/model [model-name]` | 切換本次會話模型 |
| `/gquota` | 顯示 Google Gemini Code Assist 配額用量 |
| `/personality <name>` | 設定預定義人格 |
| `/statusbar` | 切換 context/model 狀態列 |
| `/verbose` | 循環切換工具進度顯示：off → new → all → verbose |
| `/footer` | 切換 gateway 執行期 metadata footer |
| `/yolo` | 切換 YOLO 模式（跳過所有危險指令審核） |
| `/reasoning` | 管理 reasoning effort 與顯示 |
| `/fast` | 切換 fast mode（OpenAI Priority / Anthropic Fast） |
| `/skin [theme]` | 顯示或切換 UI 主題 |
| `/indicator` | 選擇 TUI busy indicator 樣式 |
| `/voice` | 切換語音模式 |
| `/busy` | 控制 Hermes 執行中時 Enter 的行為 |

### 2.3 Tools & Skills

| 指令 | 說明 |
|------|------|
| `/tools [list\|disable\|enable] [name...]` | 管理工具啟用狀態 |
| `/toolsets` | 列出可用 toolsets |
| `/skills [search\|install\|inspect\|...]` | 搜尋、安裝、管理技能 |
| `/cron` | 管理排程任務 |
| `/curator` | 背景技能維護（status / run / pin / archive） |
| `/kanban` | 多 profile 協作看板（tasks / links / comments） |
| `/reload` | 重新載入 .env 變數至執行中會話 |
| `/reload-mcp` | 從 config 重新載入 MCP 伺服器 |
| `/reload-skills` | 重新掃描 ~/.hermes/skills/ |
| `/browser` | 透過 CDP 連接瀏覽器工具 |
| `/plugins` | 列出已安裝插件及狀態 |

### 2.4 Info

| 指令 | 說明 |
|------|------|
| `/commands` | 瀏覽所有指令與技能（分頁） |
| `/help` | 顯示可用指令 |
| `/profile` | 顯示當前 profile 名稱與 home 目錄 |
| `/usage` | 顯示當前會話的 token 用量與 rate limit |
| `/insights` | 顯示用量洞察與分析 |
| `/platforms` | 顯示 gateway/messaging 平台狀態 |
| `/copy` | 複製最後一則 assistant 回應至剪貼簿 |
| `/paste` | 附加剪貼簿圖片至下一則 prompt |
| `/image <path>` | 附加本地圖片至下一則 prompt |
| `/update` | 更新 Hermes Agent 至最新版本 |
| `/debug` | 上傳除錯報告 |
| `/quit` 或 `/exit` | 退出 CLI |

---

## 3. Python Library API

來源：`run_agent.py:AIAgent`

### 3.1 AIAgent.__init__

```python
from run_agent import AIAgent

agent = AIAgent(
    base_url: str = None,          # LLM API endpoint（選填，預設 openrouter）
    api_key: str = None,           # API key（選填，從 env 讀取）
    provider: str = None,          # 提供商識別：'anthropic'、'openrouter'、'nous' 等
    api_mode: str = None,          # 'chat_completions' | 'anthropic_messages' | 'codex_responses'
    model: str = "",               # 模型名稱（OpenRouter 格式：provider/model）
    max_iterations: int = 90,      # 最大工具呼叫迭代次數
    tool_delay: float = 1.0,       # 工具呼叫間延遲（秒）
    enabled_toolsets: List[str] = None,   # 僅啟用指定 toolsets
    disabled_toolsets: List[str] = None,  # 停用指定 toolsets
    save_trajectories: bool = False,      # 是否儲存對話軌跡（JSONL）
    verbose_logging: bool = False,
    quiet_mode: bool = False,
    platform: str = None,          # 平台識別（'cli'、'telegram' 等）
    session_id: str = None,        # 會話 ID（選填，自動生成）
    session_db=None,               # 共享 SessionDB 實例
    # Callback hooks
    tool_progress_callback: callable = None,
    tool_start_callback: callable = None,
    tool_complete_callback: callable = None,
    thinking_callback: callable = None,
    reasoning_callback: callable = None,
    clarify_callback: callable = None,
    step_callback: callable = None,
    stream_delta_callback: callable = None,
    # 進階設定
    max_tokens: int = None,
    reasoning_config: Dict[str, Any] = None,
    request_overrides: Dict[str, Any] = None,
    prefill_messages: List[Dict[str, Any]] = None,
    fallback_model: Dict[str, Any] = None,
    checkpoints_enabled: bool = False,
)
```

### 3.2 AIAgent.chat()

```python
def chat(self, message: str, stream_callback: Optional[callable] = None) -> str
```

**說明**：最簡單的聊天介面，傳回最終文字回應。

**參數**：
- `message`：使用者訊息
- `stream_callback`：可選 callback，每個文字 delta 觸發（用於 TTS pipeline）

**回傳**：`str`（final_response 字串）

**範例**：
```python
agent = AIAgent(model="anthropic/claude-sonnet-4-6")
response = agent.chat("幫我寫一個 Python hello world")
print(response)
```

### 3.3 AIAgent.run_conversation()

```python
def run_conversation(
    self,
    user_message: str,
    system_message: str = None,
    conversation_history: List[Dict[str, Any]] = None,
    task_id: str = None,
    stream_callback: Optional[callable] = None,
    persist_user_message: Optional[str] = None,
) -> Dict[str, Any]
```

**說明**：完整對話執行，含工具呼叫迴圈，直至完成。

**參數**：
- `user_message`：使用者訊息
- `system_message`：自訂 system message（覆寫 ephemeral_system_prompt）
- `conversation_history`：先前對話訊息列表（選填）
- `task_id`：任務唯一識別碼（並發隔離用，選填，自動生成）
- `stream_callback`：文字串流 callback
- `persist_user_message`：儲存至歷史的清理版訊息（當 user_message 含 API 合成前綴時使用）

**回傳**：`Dict[str, Any]`，包含：
- `"final_response"`：最終回應文字
- 完整訊息歷史與工具呼叫記錄

**範例**：
```python
result = agent.run_conversation(
    "分析這份 CSV 並畫圖",
    conversation_history=previous_messages
)
print(result["final_response"])
```

### 3.4 run_agent.py:main()（CLI 模式）

```python
def main(
    query: str = None,
    model: str = "",
    api_key: str = None,
    base_url: str = "",
    max_turns: int = 10,
    enabled_toolsets: str = None,   # 逗號分隔字串
    disabled_toolsets: str = None,
    list_tools: bool = False,
    save_trajectories: bool = False,
    save_sample: bool = False,
    verbose: bool = False,
    log_prefix_chars: int = 20,
)
```

可直接執行：`python run_agent.py --query "..." --model anthropic/claude-sonnet-4-6`（使用 Python Fire CLI）

---

## 4. Gateway OpenAI 相容 API

來源：`gateway/platforms/api_server.py`

預設監聽：`http://127.0.0.1:8642`（可透過設定覆寫）

任何 OpenAI 相容前端（Open WebUI、LobeChat、LibreChat、AnythingLLM 等）均可將 base_url 指向此 server。

### 4.1 端點清單

| 方法 | 路徑 | 說明 |
|------|------|------|
| `POST` | `/v1/chat/completions` | OpenAI Chat Completions 格式（無狀態；透過 `X-Hermes-Session-Id` header 可選擇性延續會話） |
| `POST` | `/v1/responses` | OpenAI Responses API 格式（有狀態，透過 `previous_response_id` 延續） |
| `GET` | `/v1/responses/{response_id}` | 取得儲存的回應 |
| `DELETE` | `/v1/responses/{response_id}` | 刪除儲存的回應 |
| `GET` | `/v1/models` | 列出可用模型（回傳 hermes-agent 作為可用模型） |
| `GET` | `/v1/capabilities` | 機器可讀的 API capabilities（供外部 UI 使用） |
| `POST` | `/v1/runs` | 啟動 run，立即回傳 run_id（HTTP 202） |
| `GET` | `/v1/runs/{run_id}` | 取得當前 run 狀態 |
| `GET` | `/v1/runs/{run_id}/events` | SSE stream 結構化 lifecycle 事件 |
| `POST` | `/v1/runs/{run_id}/stop` | 中斷執行中的 agent |
| `GET` | `/health` | 健康檢查 |
| `GET` | `/health/detailed` | 詳細狀態（用於跨容器 dashboard 探測） |

### 4.2 Session 延續

透過 `X-Hermes-Session-Id` header 可在無狀態端點（`/v1/chat/completions`）之間延續會話：

```http
POST /v1/chat/completions
Authorization: Bearer <API_SERVER_KEY>
X-Hermes-Session-Id: <session-id>
Content-Type: application/json
```

回應 header 會包含：
```http
X-Hermes-Session-Id: <session-id>
```

---

## 5. 工具 API 概覽

來源：`tools/registry.py`（自動發現）、各 `tools/*.py`

所有工具透過 `registry.register()` 統一註冊，以 JSON Schema 描述輸入，handler 函式執行邏輯。

### 5.1 工具分類總覽表

| 分類（Toolset） | 工具名稱 | 來源檔案 |
|----------------|---------|---------|
| **terminal** | `terminal` | `tools/terminal_tool.py` |
| **terminal** | `process` | `tools/process_registry.py` |
| **file** | `read_file` | `tools/file_tools.py` |
| **file** | `write_file` | `tools/file_tools.py` |
| **file** | `patch` | `tools/file_tools.py` |
| **file** | `search_files` | `tools/file_tools.py` |
| **web** | `web_search` | `tools/web_tools.py` |
| **web** | `web_extract` | `tools/web_tools.py` |
| **browser** | `browser_navigate` | `tools/browser_tool.py` |
| **browser** | `browser_snapshot` | `tools/browser_tool.py` |
| **browser** | `browser_click` | `tools/browser_tool.py` |
| **browser** | `browser_type` | `tools/browser_tool.py` |
| **browser** | `browser_scroll` | `tools/browser_tool.py` |
| **browser** | `browser_back` | `tools/browser_tool.py` |
| **browser** | `browser_press` | `tools/browser_tool.py` |
| **browser** | `browser_get_images` | `tools/browser_tool.py` |
| **browser** | `browser_vision` | `tools/browser_tool.py` |
| **browser** | `browser_console` | `tools/browser_tool.py` |
| **browser** | `browser_cdp` | `tools/browser_cdp_tool.py` |
| **browser** | `browser_dialog` | `tools/browser_dialog_tool.py` |
| **vision** | `vision_analyze` | `tools/vision_tools.py` |
| **video** | `video_analyze` | `tools/vision_tools.py` |
| **code** | `execute_code` | `tools/code_execution_tool.py` |
| **memory** | `memory` | `tools/memory_tool.py` |
| **skills** | `skills_list` | `tools/skills_tool.py` |
| **skills** | `skill_view` | `tools/skills_tool.py` |
| **skills** | `skill_manage` | `tools/skill_manager_tool.py` |
| **delegate** | `delegate_task` | `tools/delegate_tool.py` |
| **moa** | `mixture_of_agents` | `tools/mixture_of_agents_tool.py` |
| **clarify** | `clarify` | `tools/clarify_tool.py` |
| **messaging** | `send_message` | `tools/send_message_tool.py` |
| **tts** | `text_to_speech` | `tools/tts_tool.py` |
| **image** | `image_generate` | `tools/image_generation_tool.py` |
| **todo** | `todo` | `tools/todo_tool.py` |
| **session_search** | `session_search` | `tools/session_search_tool.py` |
| **cron** | `cronjob` | `tools/cronjob_tools.py` |
| **kanban** | `kanban_show` | `tools/kanban_tools.py` |
| **kanban** | `kanban_complete` | `tools/kanban_tools.py` |
| **kanban** | `kanban_block` | `tools/kanban_tools.py` |
| **kanban** | `kanban_heartbeat` | `tools/kanban_tools.py` |
| **kanban** | `kanban_comment` | `tools/kanban_tools.py` |
| **kanban** | `kanban_create` | `tools/kanban_tools.py` |
| **kanban** | `kanban_link` | `tools/kanban_tools.py` |
| **discord** | `discord` | `tools/discord_tool.py` |
| **discord** | `discord_admin` | `tools/discord_tool.py` |
| **homeassistant** | `ha_list_entities` | `tools/homeassistant_tool.py` |
| **homeassistant** | `ha_get_state` | `tools/homeassistant_tool.py` |
| **homeassistant** | `ha_list_services` | `tools/homeassistant_tool.py` |
| **homeassistant** | `ha_call_service` | `tools/homeassistant_tool.py` |
| **feishu** | `feishu_doc_read` | `tools/feishu_doc_tool.py` |
| **feishu** | `feishu_drive_list_comments` | `tools/feishu_drive_tool.py` |
| **feishu** | `feishu_drive_list_comment_replies` | `tools/feishu_drive_tool.py` |
| **feishu** | `feishu_drive_reply_comment` | `tools/feishu_drive_tool.py` |
| **feishu** | `feishu_drive_add_comment` | `tools/feishu_drive_tool.py` |
| **yuanbao** | `yb_query_group_info` | `tools/yuanbao_tools.py` |
| **yuanbao** | `yb_query_group_members` | `tools/yuanbao_tools.py` |
| **yuanbao** | `yb_send_dm` | `tools/yuanbao_tools.py` |
| **yuanbao** | `yb_search_sticker` | `tools/yuanbao_tools.py` |
| **yuanbao** | `yb_send_sticker` | `tools/yuanbao_tools.py` |
| **rl** | `rl_list_environments` | `tools/rl_training_tool.py` |
| **rl** | `rl_select_environment` | `tools/rl_training_tool.py` |
| **rl** | `rl_get_current_config` | `tools/rl_training_tool.py` |
| **rl** | `rl_edit_config` | `tools/rl_training_tool.py` |
| **rl** | `rl_start_training` | `tools/rl_training_tool.py` |
| **rl** | `rl_check_status` | `tools/rl_training_tool.py` |
| **rl** | `rl_stop_training` | `tools/rl_training_tool.py` |
| **rl** | `rl_get_results` | `tools/rl_training_tool.py` |
| **rl** | `rl_list_runs` | `tools/rl_training_tool.py` |
| **rl** | `rl_test_inference` | `tools/rl_training_tool.py` |
| **mcp**（動態） | `mcp-<server>.*` | `tools/mcp_tool.py`（執行期動態注入） |

> 工具總數約 61 個內建工具，加上 MCP 動態工具（數量不定）。完整清單可於執行中 CLI 使用 `/tools list` 查看。

### 5.2 工具 registry.register() 簽章

```python
registry.register(
    name: str,                       # 工具名稱（唯一）
    toolset: str,                    # 所屬 toolset 分類
    schema: dict,                    # JSON Schema（OpenAI function calling 格式）
    handler: callable,               # 執行函式
    check_fn: callable = None,       # 前置需求檢查（回傳 None 表示通過）
    emoji: str = "",                 # 顯示 emoji
    max_result_size_chars: int = ..., # 回傳結果最大長度
    requires_env: ... = None,        # 需要的環境後端
    is_async: bool = False,          # 是否為 async handler
)
```

### 5.3 工具呼叫觸發路徑

```
AIAgent.run_conversation()
  → LLM 回傳 tool_calls
  → model_tools.py:handle_function_call()
  → tools/registry.py:dispatch()
  → handler(args, **kwargs)
  → JSON 結果回傳給 LLM
```

---

## 6. Authentication 模型

### 6.1 API Keys（LLM 提供商）

設定優先順序（高→低）：

1. 環境變數（如 `OPENROUTER_API_KEY`、`ANTHROPIC_API_KEY`）
2. `~/.hermes/.env`（由 `hermes_cli/env_loader.py` 載入）
3. `hermes_cli/main.py:load_hermes_dotenv()` 啟動時自動載入

常見 API Key 環境變數：

| 環境變數 | 用途 |
|---------|------|
| `OPENROUTER_API_KEY` | OpenRouter（預設提供商） |
| `ANTHROPIC_API_KEY` | Anthropic 直接 API |
| `OPENAI_API_KEY` | OpenAI / Whisper API |
| `EXA_API_KEY` | Exa 搜尋引擎 |
| `FIRECRAWL_API_KEY` | Firecrawl 爬蟲 |
| `FAL_KEY` | fal.ai 圖片生成 |
| `ELEVENLABS_API_KEY` | ElevenLabs TTS |
| `HASS_TOKEN` | Home Assistant |
| `TWILIO_*` | SMS（Twilio） |
| `API_SERVER_KEY` | Gateway API Server Bearer token |

憑證池管理（多 API key 輪換）：`agent/credential_pool.py`
- 使用 `hermes auth add` 新增 pooled credentials
- 自動在 rate limit 時輪換

### 6.2 OAuth 流程

用於 MCP 伺服器（如 GitHub MCP、Google MCP）：
- 實作：`tools/mcp_oauth.py` + `tools/mcp_oauth_manager.py`
- 支援 OAuth 2.0 authorization code flow
- Spotify：PKCE flow（`hermes auth spotify`）
- 憑證儲存於 `~/.hermes/` profile 目錄

### 6.3 DM Pairing（訊息平台配對）

用於將 Telegram / Discord 等訊息平台 DM 配對至 Hermes 帳號：
- 管理指令：`hermes pairing`（`hermes_cli/main.py:pairing_parser`）
- 設定儲存於 `~/.hermes/config.yaml`（`gateway.pairing` section）
- ⚠️ 詳細 pairing token 交換流程未驗證

### 6.4 Gateway API Server 認證

來源：`gateway/platforms/api_server.py:_validate_api_key()`

```
Authorization: Bearer <API_SERVER_KEY>
```

- 若未設定 `API_SERVER_KEY` 且 server 監聽在公開 IP，啟動時警告
- 使用 `hmac.compare_digest()` 防止 timing attack
- 錯誤回應（HTTP 401）：
  ```json
  {"error": {"message": "Invalid API key", "type": "invalid_request_error", "code": "invalid_api_key"}}
  ```

---

## 7. Error Handling Pattern

### 7.1 工具失敗 JSON 格式

來源：`tools/registry.py:dispatch()`

所有工具執行異常均捕捉並以下列格式回傳（字串 JSON，傳回 LLM）：

```json
{"error": "Tool execution failed: ExceptionType: 錯誤訊息"}
```

未知工具：
```json
{"error": "Unknown tool: tool_name"}
```

工具自訂錯誤（使用 `tools/registry.py:tool_error()` 輔助函式）：
```json
{"error": "file not found"}
```
或附加 `success` 欄位：
```json
{"error": "bad input", "success": false}
```

### 7.2 LLM API 失敗處理策略

來源：`agent/error_classifier.py`、`agent/retry_utils.py`

| 錯誤類型 | 策略 | 實作位置 |
|---------|------|---------|
| `RATE_LIMIT` | Jittered backoff 等待 + 重試 | `agent/retry_utils.py` |
| `OVERLOAD` | 短暫等待 + 重試 | `run_agent.py` |
| `AUTH_ERROR` | OAuth token 刷新 / fallback | `agent/retry_utils.py` |
| `CONTEXT_LENGTH` | 觸發 context compression | `agent/context_compressor.py` |
| `NETWORK_ERROR` | Jittered backoff 最終 fallback | `agent/retry_utils.py` |

所有重試耗盡後：切換至 `config.yaml` 的 `fallback.chain`（`run_agent.py:_try_activate_fallback()`）

**Jittered Backoff 實作**：`agent/retry_utils.py`
- 使用 Decorrelated jitter（非純指數）
- 防止多個 gateway sessions 同時衝擊同一提供商（thundering herd prevention）

### 7.3 工具危險操作審核

來源：`tools/approval.py`

agent 呼叫判定危險的工具前，會暫停並等待使用者審核：
- CLI 中透過 `/approve` 或 `/deny` 處理
- YOLO 模式（`/yolo`）：跳過所有審核
- Gateway 模式：透過訊息平台傳送審核請求

---

*文件生成日期：2026-05-05 | 資料來源：hermes_cli/commands.py、run_agent.py、gateway/platforms/api_server.py、tools/*.py*
