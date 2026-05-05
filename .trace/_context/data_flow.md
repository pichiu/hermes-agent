# Stage 2.2 — Request / Data Flow

## 代表性 Use Case：CLI 用戶發送訊息

以「用戶在 CLI 輸入 `ls -la /tmp`」為例，完整追蹤從輸入到輸出：

```
用戶輸入
  │
  ▼
[cli.py] HermesCLI.run()
  │  prompt_toolkit 接收輸入
  │  process_command() 確認非斜線指令
  │
  ▼
[cli.py] HermesCLI.chat(user_input)
  │  包裝 conversation_history
  │
  ▼
[run_agent.py] AIAgent.run_conversation(user_message)
  │
  ├── [初始化] _ensure_db_session()           # 建立/恢復 SQLite 會話
  ├── [記憶] memory_manager.on_turn_start()   # 通知記憶提供者新回合
  ├── [記憶] prefetch_all()                   # 預取外部記憶（Honcho 等）
  │
  ├── [系統提示] _cached_system_prompt        # 第一回合才建構，後續重用
  │   └── agent/prompt_builder.py             # 組裝 persona + 記憶 + 技能 + 平台提示
  │
  ├── [壓縮] 評估 token 數，必要時壓縮上下文  # agent/context_compressor.py
  │
  ├── [Plugin] invoke_hook("pre_llm_call")    # 插件可注入額外 context
  │
  ├── [構建] api_messages = 組裝請求
  │   ├── 注入外部記憶 prefetch（加入用戶訊息）
  │   ├── 注入 plugin context（加入用戶訊息，不動系統提示）
  │   ├── 套用 Anthropic cache_control 標記
  │   └── 清理非法字元（surrogates, strict-API 欄位）
  │
  ├── [主迴圈] while api_call_count < max_iterations:
  │   │
  │   ├── _interruptible_api_call(api_kwargs) # 可中斷的 API 呼叫
  │   │   └── client.chat.completions.create(...)  # OpenAI 相容 API
  │   │       └── streaming: 逐 token 回調給 UI
  │   │
  │   ├── [工具呼叫] 若 response.tool_calls:
  │   │   ├── _should_parallelize_tool_batch()  # 評估是否可平行化
  │   │   ├── For each tool_call:
  │   │   │   ├── [Plugin] invoke_hook("pre_tool_call")   # 可攔截/封鎖
  │   │   │   ├── handle_function_call(name, args, task_id)
  │   │   │   │   └── registry.dispatch(name, args)       # tools/registry.py
  │   │   │   │       └── tools/*.py 中的具體實作
  │   │   │   ├── [Plugin] invoke_hook("post_tool_call")  # 觀察結果
  │   │   │   └── [Plugin] invoke_hook("transform_tool_result")  # 可轉換結果
  │   │   └── messages.append(tool_result_message)
  │   │
  │   └── [完成] 若無 tool_calls → final_response = response.content
  │
  ├── [持久化] _persist_session(messages)     # 寫入 SQLite
  ├── [記憶] memory_manager.sync_turn()       # 同步到外部記憶提供者
  ├── [技能] 評估是否應觸發技能創建/改善
  ├── [Plugin] invoke_hook("on_session_end")
  │
  └── return {"final_response": ..., "messages": ..., "completed": ...}
  │
  ▼
[cli.py] 格式化並顯示 final_response
  │  Rich 渲染（Markdown → 終端機）
  │  KawaiiSpinner 停止
  └── 等待下一個用戶輸入
```

## 訊息資料結構

### 內部訊息格式（OpenAI Chat Format）

```python
# 用戶訊息
{"role": "user", "content": "ls -la /tmp"}

# 助理訊息（含工具呼叫）
{
    "role": "assistant",
    "content": None,
    "tool_calls": [
        {
            "id": "call_abc123",
            "type": "function",
            "function": {
                "name": "terminal",
                "arguments": '{"command": "ls -la /tmp"}'
            }
        }
    ],
    "reasoning": "<think>...</think>"  # 僅用於軌跡儲存，不送 API
}

# 工具結果訊息
{
    "role": "tool",
    "tool_call_id": "call_abc123",
    "content": '{"output": "total 8\\n..."}'
}
```

### 系統提示結構

```
system_prompt =
    [persona: SOUL.md 內容]
    [identity: DEFAULT_AGENT_IDENTITY]
    [platform hints: PLATFORM_HINTS["cli"]]
    [memory: MEMORY_GUIDANCE]
    [session search: SESSION_SEARCH_GUIDANCE]
    [skills: SKILLS_GUIDANCE]
    [active skills: 載入的技能內容]
    [context files: AGENTS.md, .cursorrules]
    [memory entries: MEMORY.md / USER.md]
    [honcho: 辯證推理 context]
```

## Gateway 訊息流（Telegram → Agent）

```
Telegram Bot API (webhook/polling)
  │
  ▼
gateway/platforms/telegram.py:TelegramAdapter
  │  接收更新事件
  │  訊息去重、媒體下載（照片/語音）
  │  語音 → 文字（STT：faster-whisper / Whisper API）
  │
  ▼
gateway/run.py:GatewayRunner._handle_message()
  │  
  ├── 路由斜線指令（/stop, /new, /approve 等）
  ├── 建立/恢復 GatewaySession
  │
  ▼
gateway/session.py:GatewaySession.process_message()
  │  
  │  建立 AIAgent（每條訊息重建，但 conversation_history 從 SessionStore 讀取）
  │
  ▼
AIAgent.run_conversation(user_message, conversation_history=history)
  │  （同 CLI 流程）
  │
  ▼
gateway/delivery.py:DeliveryRouter.deliver()
  │  分塊長訊息（Telegram 4096 字元限制）
  │  傳送媒體（圖片、音訊）
  └── 回覆用戶
```

## 工具呼叫分派（tools/registry.py）

```python
# tools/registry.py 的 dispatch 方法
def dispatch(self, name, args, task_id=None, **kwargs):
    handler = self._handlers[name]      # import-time 自動發現的 handler
    if asyncio.iscoroutinefunction(handler):
        return _run_async(handler(**args, **kwargs))
    else:
        return handler(**args, **kwargs)
```

### 工具自動發現機制

```python
# tools/registry.py:discover_builtin_tools()
# 掃描 tools/ 目錄下所有 .py 檔案（除 registry.py 本身）
# 每個工具檔案在 import 時呼叫 registry.register()：

# 範例：tools/terminal_tool.py
@registry.register(
    name="terminal",
    toolset="terminal",
    description="Execute terminal commands...",
    # schema, handler...
)
```

## 上下文壓縮流程

當訊息 tokens 接近模型上限時：

```
[評估] estimate_request_tokens_rough() → 超過閾值
  ↓
[啟動] agent/context_compressor.py:ContextCompressor
  ↓
[選擇] 保留 first_n 條 + last_n 條訊息
  ↓
[摘要] 用壓縮模型對中間部分生成摘要（次要 AIAgent 呼叫）
  ↓
[替換] 中間訊息替換為摘要訊息
  ↓
[新會話] 創建新 SQLite 會話（保留 parent_session_id 連結）
  ↓
[cache 失效] _cached_system_prompt = None（強制重建系統提示）
```

## 關鍵設計決策

### Prompt Caching 優先
- 系統提示只在第一回合建構，後續 100% 重用（`_cached_system_prompt`）
- 外部記憶 prefetch 注入**用戶訊息**而非系統提示（保護 cache prefix）
- Plugin context 也注入用戶訊息，同樣不碰系統提示

### 會話連續性（Gateway 模式）
- Gateway 對每條訊息建立新的 `AIAgent`（無狀態設計）
- 但 `conversation_history` 從 `SessionStore` 讀取，`system_prompt` 從 SQLite 讀取
- 這樣 Anthropic cache prefix 完全一致，跨訊息保持 cache 命中

### 工具並行化
- `_should_parallelize_tool_batch()` 評估工具呼叫是否可並行（run_agent.py:376）
- 並行路徑抽取 `_extract_parallel_scope_path()` 防止路徑衝突
- 使用 ThreadPoolExecutor，每個 worker 有獨立的 asyncio event loop
