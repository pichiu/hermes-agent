# Data Flow — Hermes Agent

## 代表性 Use Case：用戶透過 Telegram 發送訊息，agent 執行 terminal 命令並回覆

---

## 完整流程（從輸入到輸出）

### 1. Platform 接收訊息（`gateway/platforms/telegram.py`）

```
用戶在 Telegram 傳送：「幫我列出 /tmp 目錄下的檔案」
  │
  ▼
TelegramAdapter._on_message(update, context)
  ├─ 建立 MessageEvent 物件
  │    └─ text = "幫我列出 /tmp 目錄下的檔案"
  │    └─ source.user_id = "12345678"
  │    └─ source.platform = Platform.TELEGRAM
  └─ await callback(event)  → GatewayRunner._handle_message(event)
```

### 2. Gateway 訊息分派（`gateway/run.py:1767`）

```
GatewayRunner._handle_message(event)
  │
  ├─ _is_user_authorized(source)   # 檢查 authorized_users 白名單
  │    └─ 未授權 → 發送 pairing code → return
  │
  ├─ 檢查是否為 slash command
  │    ├─ /new → 清除 session → return
  │    ├─ /model → 切換 LLM → return
  │    └─ /skills 等 → 處理後 return
  │
  ├─ session_key = build_session_key(source)
  │    └─ "{platform}:{chat_id}" (e.g. "telegram:12345678")
  │
  ├─ 檢查是否已有 running agent（防止並發）
  │    └─ 有 → 發送 interrupt 信號給現有 agent
  │
  ├─ session_history = SessionStore.get_history(session_key)
  │    └─ 從 SQLite 讀取對話歷史
  │
  └─ await _handle_message_with_agent(event, source, session_key)
```

### 3. Agent 建立與執行（`gateway/run.py:2302`）

```
_handle_message_with_agent(event, source, session_key)
  │
  ├─ runtime_kwargs = _resolve_runtime_agent_kwargs()
  │    └─ 解析 provider、api_key、base_url（from config.yaml + .env）
  │
  ├─ 建立 AIAgent 實例（run_agent.py:AIAgent.__init__）
  │    ├─ model = config.model.default（e.g. "anthropic/claude-opus-4.6"）
  │    ├─ platform = "telegram"
  │    ├─ enabled_toolsets = ["core", "web", "terminal", "file", ...]
  │    ├─ session_db = SessionDB（SQLite）
  │    └─ memory_manager = MemoryManager + BuiltinMemoryProvider
  │
  └─ agent_result = await _run_agent(agent, event, history, ...)
```

### 4. AIAgent 對話迴圈（`run_agent.py:6911`）

```
AIAgent.run_conversation(user_message, conversation_history)
  │
  ├─ _build_system_prompt()        # 組裝系統提示（第一次才建立，之後快取）
  │    ├─ [1] Agent identity（SOUL.md 或 DEFAULT_AGENT_IDENTITY）
  │    ├─ [2] Memory guidance
  │    ├─ [3] Skills guidance
  │    ├─ [4] MEMORY.md + USER.md 內容（冷凍快照）
  │    ├─ [5] Context files（AGENTS.md, .cursorrules）
  │    ├─ [6] 當前日期時間
  │    └─ [7] 平台 hint（"You are responding via Telegram"）
  │
  ├─ messages = [...conversation_history, {"role": "user", "content": user_message}]
  │
  ├─ 外部記憶 prefetch（Honcho 等 plugin）
  │    └─ _ext_prefetch_cache = memory_manager.prefetch_all(user_message)
  │
  └─ while api_call_count < max_iterations:   # 主工具呼叫迴圈
       │
       ├─ api_messages = [準備 API 訊息]
       │    ├─ 注入 ephemeral context（external memory prefetch）
       │    ├─ 注入 plugin pre_llm_call hooks
       │    ├─ 加上 system prompt（prepend）
       │    ├─ apply_anthropic_cache_control()（如果是 Claude）
       │    └─ _sanitize_api_messages()（修復孤兒 tool result）
       │
       ├─ response = _interruptible_streaming_api_call(api_messages)
       │    └─ 使用 openai.OpenAI client（provider 無關）
       │    └─ model = "anthropic/claude-opus-4.6"（via OpenRouter）
       │
       ├─ 解析 response
       │    ├─ finish_reason = "stop" → 最終回覆，跳出迴圈
       │    └─ finish_reason = "tool_calls" → 繼續處理工具呼叫
       │
       └─ 工具呼叫處理（若 finish_reason == "tool_calls"）
```

### 5. 工具呼叫執行（`run_agent.py` + `model_tools.py`）

```
[LLM 回傳 tool_calls = [{"function": {"name": "terminal", "arguments": '{"command":"ls /tmp"}'}}]]
  │
  ├─ _should_parallelize_tool_batch(tool_calls)
  │    └─ 單個 terminal 呼叫 → sequential
  │
  ├─ handle_function_call("terminal", {"command": "ls /tmp"})
  │    └─ tools/registry.py → dispatch → tools/terminal_tool.py
  │         ├─ 危險命令偵測：approval.py（ls 是安全的，跳過）
  │         ├─ 選擇 backend：local（TERMINAL_ENV=local）
  │         ├─ 執行：subprocess.run("ls /tmp")
  │         └─ return {"output": "...", "exit_code": 0}
  │
  ├─ tool_result 放入 messages
  │    └─ {"role": "tool", "content": result, "tool_call_id": "call_xyz"}
  │
  └─ api_call_count += 1 → 繼續主迴圈
       └─ LLM 看到工具結果 → 生成最終回覆
       └─ finish_reason = "stop" → 跳出迴圈
```

### 6. 回覆投遞（`gateway/delivery.py`）

```
final_response = "以下是 /tmp 目錄的檔案列表：..."
  │
  ├─ 儲存 conversation 到 SessionStore（SQLite）
  │
  ├─ memory_manager.sync_all(user_msg, final_response)
  │    └─ BuiltinMemoryProvider: 無自動更新（需 agent 主動呼叫 memory tool）
  │
  └─ DeliveryRouter.deliver(final_response, source)
       └─ TelegramAdapter.send(chat_id, response_text)
            └─ bot.send_message(chat_id, text, parse_mode="MarkdownV2")
```

---

## CLI 路徑的差異

相較於 gateway 路徑，CLI 路徑的差異：

| 步驟 | Gateway | CLI |
|------|---------|-----|
| 輸入接收 | Platform webhook / polling | prompt_toolkit 鍵盤輸入 |
| Session 管理 | SessionStore (per chat_id) | SessionDB (per CLI session) |
| Agent 建立 | 每訊息可能複用快取 AIAgent | 整個 CLI 生命週期共用一個 |
| 工具回饋 | 靜默（HERMES_QUIET=1） | 即時 streaming + KawaiiSpinner |
| 危險命令 | 平台按鈕審批（Telegram inline buttons） | prompt_toolkit 互動提示 |

---

## 平行工具呼叫路徑

當 LLM 回傳多個 tool_calls 且通過並行安全性檢查時：

```
_should_parallelize_tool_batch(tool_calls) → True
  │
  └─ ThreadPoolExecutor(max_workers=8)
       ├─ Thread 1: handle_function_call("web_search", ...)
       ├─ Thread 2: handle_function_call("read_file", ...)  
       └─ Thread 3: handle_function_call("web_extract", ...)
       
  並行安全性規則（run_agent.py:216-233）：
  - _NEVER_PARALLEL_TOOLS = {"clarify"}  # 需要用戶互動，不可並行
  - _PARALLEL_SAFE_TOOLS = {read_only tools}  # 讀取型工具可並行
  - _PATH_SCOPED_TOOLS 相互不重疊時才能並行  # 防止相同檔案寫入衝突
```

---

## 訊息格式轉換

```
Telegram 原始訊息
  ↓ TelegramAdapter
MessageEvent { text, media_urls, source }
  ↓ GatewayRunner
user_message: str（純文字 + 媒體 placeholder）
  ↓ AIAgent.run_conversation
messages: [{"role": "user/assistant/tool", "content": ...}]  # OpenAI format
  ↓ 針對 API mode 轉換
api_messages: [OpenAI / Anthropic / Codex format]
  ↓ openai.OpenAI client
HTTP POST → LLM provider
```

---

## SQLite Session 持久化（`hermes_state.py`）

每次 `run_conversation()` 完成後：

```
SessionDB.save_message(session_id, role, content, ...)
  ├─ INSERT INTO messages
  ├─ UPDATE sessions SET message_count, token_count, ...
  └─ FTS5 index 自動更新（messages_fts）
       └─ 支援 session_search tool 的全文搜尋
```
