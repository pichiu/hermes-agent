# Stage 2.3 核心領域邏輯

## 「心臟」：三個互相協作的核心系統

### 1. Tool-calling Loop（`agent/conversation_loop.py`）

Hermes 的核心執行迴路：ReAct pattern（Reason + Act）

```python
# conversation_loop.py:834
while api_call_count < agent.max_iterations AND budget.remaining > 0:
    response = LLM_API_call(messages, tools)
    
    if finish_reason == "stop":
        break  # LLM 決定完成
    
    if finish_reason == "tool_calls":
        results = execute_tools_concurrent(response.tool_calls)
        messages.append(tool_results)
        continue  # 繼續迭代
    
    if finish_reason == "length":
        handle_context_overflow()  # 壓縮或截斷
```

**關鍵設計**：
- `max_iterations = 90`：防止無限迴圈
- `IterationBudget`（`agent/iteration_budget.py`）：更細粒度的預算控制，支援 subagent 繼承父 agent 的預算
- 並行工具執行（最多 8 線程）：`ThreadPoolExecutor` 在 `agent/tool_executor.py`
- Interrupt handling：`Ctrl+C` 可以中斷工具執行，agent 會優雅退出當前 turn

---

### 2. Skill 系統（自我改善核心）

Skill 是 Hermes 最獨特的設計：**程序性記憶文件**（Markdown）+ **FTS5 全文搜尋**

```
~/.hermes/skills/
├── user-created/          # 用戶手動創建
├── agent-created/         # Agent 自動創建
│   ├── skill-name.md      # agentskills.io 格式的 Markdown
│   └── .usage/            # 使用統計（skill_usage.py）
└── imported/              # 從外部安裝的 skills
```

**Skill 生命週期**：

```
任務完成 → background_review.spawn_background_review_thread()
              └─ 輔助 LLM agent 分析本次對話
                    ├─ 決定是否創建新 skill
                    ├─ skill_manage create → 寫入 ~/.hermes/skills/agent-created/
                    └─ 更新 skill 使用統計（.usage/ 目錄）

使用 skill → tools/skills_tool.py::skill_view()
              ├─ 讀取 Markdown 文件
              ├─ 注入當前 conversation context
              └─ _skill_view_with_bump()：更新使用次數/時間戳

Curator（背景維護）→ agent/curator.py::maybe_run_curator()
              ├─ 7 天無活動後觸發（DEFAULT_INTERVAL_HOURS = 24*7）
              ├─ 生命週期管理：active → stale（30天）→ archived（90天）
              └─ 輔助 LLM agent 執行：pin/archive/consolidate/patch
```

**Skill 搜尋**：FTS5 SQLite 全文搜尋，在 `tools/skill_manager_tool.py` 和 `tools/session_search_tool.py` 中使用

---

### 3. Memory 系統

**三層記憶架構**：

```
Layer 1: 系統提示詞記憶（MEMORY.md）
    ├─ ~/.hermes/MEMORY.md — 全局記憶
    ├─ ~/.hermes/USER.md   — 用戶模型
    └─ SOUL.md             — Agent 人格/身份
    
Layer 2: Session 搜尋（FTS5）
    ├─ 歷史對話 SQLite 索引
    └─ tools/session_search_tool.py — 跨 session 召回

Layer 3: 外部 Memory Provider（可選，只能選一個）
    ├─ Honcho（plastic-labs）— dialectic user modeling
    ├─ mem0
    ├─ Supermemory
    ├─ Byterover
    ├─ Hindsight
    ├─ Holographic
    ├─ OpenViking
    └─ RetainDB
```

**MemoryManager 介面**（`agent/memory_manager.py`）：
```python
class MemoryManager:
    def build_system_prompt(self) -> str  # 注入記憶 context
    def prefetch_all(self, user_message: str)  # 前置載入相關記憶
    def sync_all(self, user_msg, assistant_response)  # 寫回記憶
    def queue_prefetch_all(self, user_msg)  # 異步預取下一 turn
```

**記憶注入方式**：XML fence 標籤注入系統提示詞
```xml
<memory-context>
[System note: The following is recalled memory context...]
{memory content}
</memory-context>
```

---

### 4. Provider Adapter Pattern（多 LLM 支援）

**所有 LLM 呼叫統一走 OpenAI SDK**（OpenAI-compatible endpoint），但各 provider 有適配層：

| Adapter | 檔案 | 功能 |
|---------|------|------|
| Anthropic | `agent/anthropic_adapter.py` | 原生 Anthropic API（非 OpenAI 相容路徑） |
| Gemini | `agent/gemini_native_adapter.py` | Google Gemini 原生 API |
| Bedrock | `agent/bedrock_adapter.py` | AWS Bedrock |
| Azure | `agent/azure_identity_adapter.py` | Azure OpenAI |
| Codex | `agent/codex_responses_adapter.py` | OpenAI Responses API |
| LMStudio | `agent/lmstudio_reasoning.py` | 本地 LM Studio |

**Fallback 機制**：
```python
# agent/error_classifier.py
class FailoverReason(Enum):
    RATE_LIMIT = "rate_limit"
    CONTEXT_LENGTH = "context_length"
    API_ERROR = "api_error"
    ...

# 當主 provider 失敗，自動切換到 fallback_model
agent._restore_primary_runtime()  # 下一 turn 嘗試恢復主 provider
```

---

### 5. Context 壓縮

**觸發條件**：conversation_history 接近模型 context 上限

**`agent/context_compressor.py::ContextCompressor`**：
- `estimate_messages_tokens_rough()`：粗估 token 數（字元數 / 3.5）
- 觸發後：spawn 輔助 LLM call → 生成歷史摘要
- 保留：系統提示詞 + 壓縮摘要 + 最近 N 輪

**Prompt Caching**（Anthropic 專屬）：
```python
# agent/prompt_caching.py
apply_anthropic_cache_control(messages)
# 在系統提示詞和固定 prefix 上設置 cache_control，節省 token 費用
```

---

### 6. Tool Guardrails（危險命令審批）

`agent/tool_guardrails.py::ToolCallGuardrailController`：

```python
# 審批流程
decision = guardrails.before_call(tool_name, args)
if decision == BLOCK:
    return error_result
elif decision == REQUIRE_APPROVAL:
    user_choice = prompt_user_approval()
    # once / session / always / deny
    # pre_approval_request + post_approval_response observer hooks
```

**Tirith Security**（`tools/tirith_security.py`）：基於規則的安全策略引擎

---

### 核心 Abstraction 總結

| 模式 | 實作 |
|------|------|
| ReAct Loop | `agent/conversation_loop.py` 的 while 迴圈 |
| Adapter | `agent/*_adapter.py`（provider 適配） |
| Registry | `tools/registry.py`（工具自注冊） |
| Observer Hook | `model_tools.py::_emit_*_hook()`（只讀 telemetry） |
| Middleware Chain | `docs/middleware/`（可改寫 LLM/工具 請求） |
| Strategy | Memory provider（只選一個，可替換） |
| Pipeline | trajectory_compressor.py（研究用 trajectory 處理） |
