# Stage 2.2 Request / Data Flow

## 代表性 Use Case：CLI 使用者傳送訊息，Agent 執行工具並回應

### 完整流程圖

```
用戶輸入（cli.py REPL）
    │
    ▼
cli.py — 解析輸入（slash commands / 普通訊息）
    │ user_message: str
    ▼
AIAgent.run_conversation(user_message)           [agent/conversation_loop.py:364]
    │
    ├─ 1. Pre-turn setup
    │     ├─ _ensure_db_session()              — 建立/恢復 DB session 記錄
    │     ├─ set_session_context(session_id)   — 設定 logging context
    │     ├─ _restore_primary_runtime()        — 還原主要 LLM provider（若上一 turn fallback 了）
    │     ├─ sanitize user_message（去除代理字元）
    │     ├─ reset retry 計數器
    │     └─ IterationBudget(max_iterations=90)— 建立本 turn 的迭代預算
    │
    ├─ 2. System prompt 建構
    │     └─ agent/system_prompt.py            — Jinja2 渲染，含記憶、skills、工具清單
    │
    ├─ 3. _run() — 主要 Tool-calling Loop    [conversation_loop.py:834]
    │
    │   while api_call_count < max_iterations AND budget.remaining > 0:
    │
    │     3a. 組裝 API request
    │         ├─ conversation_history + user_message
    │         ├─ tool definitions（從 tools/registry.py 取得已啟用工具的 JSON schema）
    │         ├─ apply_anthropic_cache_control()（若為 Anthropic provider）
    │         └─ middleware：llm_request middleware 插入點（plugin 可改寫 request kwargs）
    │
    │     3b. 呼叫 LLM API
    │         ├─ pre_api_request observer hook 觸發
    │         ├─ agent._interruptible_api_call()
    │         │    ├─ OpenAI client.chat.completions.create()（或對應 adapter）
    │         │    ├─ llm_execution middleware 插入點
    │         │    ├─ jittered_backoff retry（tenacity）
    │         │    └─ 串流處理（stream_delta_callback → TTS pipeline）
    │         └─ post_api_request / api_request_error observer hook 觸發
    │
    │     3c. 解析回應
    │         ├─ finish_reason 判斷：stop / tool_calls / length
    │         ├─ _sanitize_tool_call_arguments()（修復損毀的工具參數）
    │         └─ 若 finish_reason == "stop"：退出迴圈，回傳最終 response
    │
    │     3d. 若有 tool_calls：工具執行
    │         ├─ pre_tool_call observer hook + tool_request middleware
    │         ├─ ToolCallGuardrailController.before_call()（危險命令審批）
    │         ├─ agent/tool_executor.py::execute_tool_calls_concurrent()（最多 8 並行）
    │         │    ├─ 查找 tools/registry.py 中的 handler
    │         │    ├─ tool_execution middleware 插入點
    │         │    ├─ handler(**args) — 實際工具執行
    │         │    └─ maybe_persist_tool_result()（大型結果存至磁碟，避免 context bloat）
    │         ├─ post_tool_call observer hook + transform_tool_result hook
    │         └─ 工具結果附加回 conversation_history
    │
    │     ↑ 繼續下一次迭代
    │
    ├─ 4. Post-turn processing
    │     ├─ background_review.py（背景觸發 skill 改善/記憶整理）
    │     ├─ memory_manager.sync_all()（同步記憶 provider）
    │     ├─ curator.py nudge（若達到閾值，觸發記憶摘要/skill 建立）
    │     └─ _save_transcript()（寫入 session transcript）
    │
    └─ 5. 回傳 Dict[str, Any]
          ├─ "response": str（最終 assistant 文字）
          ├─ "conversation_history": List[Dict]
          └─ "metadata": 使用量、session 資訊
```

## 工具呼叫的詳細路徑

```python
# tools/registry.py — 工具取得
entry = registry.get_entry(function_name)
handler = entry.handler

# agent/tool_executor.py — 執行
with ThreadPoolExecutor(max_workers=8) as executor:
    futures = [executor.submit(handler, **args) for tc in tool_calls]
    results = [f.result() for f in futures]

# 結果格式化為 tool message
tool_result_msg = make_tool_result_message(call_id, result, ...)
conversation_history.append(tool_result_msg)
```

## Gateway 模式的 Data Flow 差異

```
Telegram/Discord/Slack/等 → 平台 webhook
    │
    ▼
gateway/run.py::_handle_message()
    │
    ├─ DM pairing 授權驗證
    ├─ slash command 解析（/model, /new, /reset 等）
    ├─ 跨平台 session 路由（per user/chat 維持獨立 conversation_history）
    │
    └─ AIAgent.run_conversation()（同 CLI 路徑）
          │
          └─ 回應透過 platform.send_message() 送回
```

## Context 壓縮觸發

當 conversation_history 接近 context 上限時：
```
agent/context_compressor.py::ContextCompressor
    ├─ 估算 token 數（estimate_messages_tokens_rough）
    ├─ 若超過閾值：觸發 LLM 摘要壓縮
    └─ 保留系統提示詞、最近 N 輪對話、壓縮的歷史摘要
```

## 關鍵資料轉換層

| 層 | 轉換 | 檔案 |
|---|---|---|
| Input 清理 | Unicode surrogate 去除 | `agent/message_sanitization.py` |
| Provider 適配 | OpenAI schema → Anthropic/Gemini/等 | `agent/*_adapter.py` |
| Tool schema | Python 函式 → JSON Schema | `tools/registry.py::get_definitions()` |
| 大型結果 | 工具輸出 → 磁碟 + 參考 ID | `tools/tool_result_storage.py` |
| 記憶 context | MEMORY.md → XML fence 注入系統提示詞 | `agent/memory_manager.py` |
| Skill 文件 | Markdown → 注入 context | `tools/skills_tool.py::skill_view()` |
