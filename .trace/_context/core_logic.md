# Core Logic — Hermes Agent

## 專案的「心臟」：三個核心機制

Hermes Agent 的核心非 trivial 邏輯集中在三個互相交織的機制：

1. **Closed Learning Loop**（自我改進）
2. **AIAgent 工具呼叫迴圈**（執行引擎）
3. **Context 壓縮管理**（長對話維持）

---

## 1. Closed Learning Loop（自我改進機制）

### 架構模式：Observer + Nudge-based Trigger

Agent 不只是執行任務，還會在適當時機「主動」進行記憶整理和 skill 創建。

#### 記憶 Nudge（`run_agent.py:7032-7041`）

```python
# 每 N 個 user turn 觸發一次記憶整理提示
if (self._memory_nudge_interval > 0
        and "memory" in self.valid_tool_names
        and self._memory_store):
    self._turns_since_memory += 1
    if self._turns_since_memory >= self._memory_nudge_interval:
        _should_review_memory = True
        self._turns_since_memory = 0
```

觸發後，agent 在系統提示中看到「你距上次更新記憶已有 N 個對話，考慮更新 MEMORY.md」的提醒。

#### Skill Nudge（`run_agent.py:7270-7272`）

```python
# 每 N 個 tool-calling iteration 觸發一次 skill 創建提示
if (self._skill_nudge_interval > 0
        and "skill_manage" in self.valid_tool_names):
    self._iters_since_skill += 1
```

複雜任務（需多次工具呼叫）後，agent 被提示「你剛完成了複雜任務，考慮創建 skill 以備未來使用」。

#### Skills 三層載入（`tools/skills_tool.py`）

```
Level 0: skills_list()        → 只返回名稱 + 描述（~3k tokens）
Level 1: skill_view(name)     → 載入完整 SKILL.md 內容
Level 2: skill_view(name, path) → 載入 references/ 下的特定檔案
```

這種漸進式揭露設計確保大量 skill 不會一次性塞滿 context window。

#### Skill 格式（agentskills.io 標準）

```yaml
---
name: skill-name            # 最長 64 字元
description: Brief desc     # 最長 1024 字元
version: 1.0.0
platforms: [macos, linux]   # 平台限制（可選）
metadata:
  hermes:
    tags: [research, web]
    related_skills: [arxiv]
---

# Skill 主要內容（Markdown）
完整指令說明...
```

---

## 2. AIAgent 工具呼叫迴圈

### 架構模式：ReAct（Reason → Act → Observe → Repeat）

主迴圈位於 `run_agent.py:7222`：

```python
while api_call_count < self.max_iterations and self.iteration_budget.remaining > 0:
    # 1. 中斷檢查
    if self._interrupt_requested: break
    
    # 2. 消耗 iteration budget
    if not self.iteration_budget.consume(): break
    
    # 3. 建立 API 請求（加密頭、prompt caching、ephemeral context...）
    api_messages = self._prepare_api_messages(messages)
    
    # 4. 呼叫 LLM（streaming 優先）
    response = self._interruptible_streaming_api_call(api_kwargs)
    
    # 5. 解析回應
    if finish_reason == "stop":
        # 最終回覆 → 跳出迴圈
        break
    elif finish_reason == "tool_calls":
        # 執行工具 → 繼續迴圈
        results = execute_tools(response.tool_calls)
        messages.extend(tool_results)
    elif finish_reason == "length":
        # Context 太長 → 觸發壓縮 → 繼續
        messages = compress_context(messages)
```

### IterationBudget（`run_agent.py:168-209`）

```
AIAgent.max_iterations = 90（預設，parent + all subagents 共享）
每個 API call 消耗 1 iteration
execute_code turns 會退還（refund）預算
Subagent 有獨立預算（delegation.max_iterations = 50）
```

### 平行工具執行（`run_agent.py:265-306`）

```python
_PARALLEL_SAFE_TOOLS = frozenset({
    "web_search", "web_extract", "read_file", "search_files",
    "session_search", "skill_view", "vision_analyze", ...  # 唯讀工具
})

_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})  # 需用戶互動

if _should_parallelize_tool_batch(tool_calls):
    # ThreadPoolExecutor(max_workers=8) 並行執行
```

---

## 3. Context 壓縮管理（`agent/context_compressor.py`）

### 架構模式：Sliding Window + LLM Summarization

```
ContextCompressor 演算法：

1. 剪枝舊工具結果（cheap，無 LLM 呼叫）
   └─ 替換為 "[Old tool output cleared to save context space]"

2. 保護 head（前 N 條訊息，含系統提示）
3. 保護 tail（後 M token，最近 ~20K tokens）

4. 中間段落 → 輔助 LLM 摘要（structured template）：
   - Goal: 任務目標
   - Progress: 已完成什麼
   - Decisions: 關鍵決策
   - Files: 修改過的檔案
   - Next Steps: 後續計畫

5. 建立新 session（SQLite parent_session_id chain）
   └─ compression_triggered_splitting: session 延續鏈
```

摘要 token 預算（`context_compressor.py:38-43`）：
- 最小：2000 tokens
- 比例：壓縮內容的 20%
- 上限：12,000 tokens

---

## 4. 系統提示組裝（`agent/prompt_builder.py`）

### 架構模式：Layered Builder（7 層）

```python
def _build_system_prompt():
    # 層次（按優先順序）：
    [1] Agent identity        SOUL.md 或 DEFAULT_AGENT_IDENTITY
    [2] Tool-aware guidance   memory/session_search/skills 指引
    [3] Tool enforcement      針對 GPT/Gemini 的工具呼叫強制
    [4] User system prompt    外部注入（來自 gateway/batch）
    [5] Memory block          MEMORY.md + USER.md 快照
    [6] External memory       Honcho 等 plugin 的系統提示
    [7] Context files         AGENTS.md, .cursorrules（with injection scan）
    [8] Skills index          已啟用 skills 的 Level 0 metadata
    [9] Date/time             當前時區時間
    [10] Platform hint        "You are responding via Telegram"
```

**設計重點**：系統提示在 session 第一個 turn 建立後快取（`self._cached_system_prompt`），後續不重建，以最大化 Anthropic prefix cache 命中率。

---

## 5. Prompt Injection 防護（`agent/prompt_builder.py:36-53`）

Context 檔案（AGENTS.md、.cursorrules、SOUL.md）在注入前會掃描：

```python
_CONTEXT_THREAT_PATTERNS = [
    (r'ignore\s+(previous|all|above|prior)\s+instructions', "prompt_injection"),
    (r'do\s+not\s+tell\s+the\s+user', "deception_hide"),
    (r'system\s+prompt\s+override', "sys_prompt_override"),
    (r'curl\s+[^\n]*\$\{?\w*(KEY|TOKEN|SECRET|PASSWORD|CREDENTIAL|API)', "exfil_curl"),
    (r'cat\s+[^\n]*(\.env|credentials|\.netrc|\.pgpass)', "read_secrets"),
    ...
]

_CONTEXT_INVISIBLE_CHARS = {'\u200b', '\u200c', '\u200d', ...}  # 隱形 Unicode
```

偵測到威脅 → 替換為 `[BLOCKED: filename contained potential prompt injection (...)]`。

---

## 6. API Mode 自動偵測

Hermes 統一使用 `openai.OpenAI` client，但透過 `api_mode` 切換協議：

```
api_mode = "chat_completions"     # OpenAI / OpenRouter / 大多數 provider
api_mode = "codex_responses"      # OpenAI Codex（Responses API）
api_mode = "anthropic_messages"   # 直接 Anthropic API（含原生工具格式）
```

自動偵測邏輯（`run_agent.py:581-596`）：
```python
if "api.anthropic.com" in base_url:  → anthropic_messages
elif base_url.endswith("/anthropic"):  → anthropic_messages（第三方相容端點）
elif provider == "openai-codex":       → codex_responses
else:                                  → chat_completions（預設）
```

---

## 7. 記憶系統（三種模式）

```
模式 1: local（內建）
  ├─ MEMORY.md  → 長期事實、偏好、專案資訊
  ├─ USER.md    → 用戶個人資料、溝通風格
  └─ 工具：memory tool → 讀/寫/搜尋/重置

模式 2: honcho（外部 plugin）
  ├─ Honcho AI dialectic 推理
  ├─ 用戶行為建模（跨 session）
  └─ prefetch: 每 turn 查詢相關記憶
  
模式 3: hybrid（同時啟用兩者）
  └─ MemoryManager.build_system_prompt() 合併兩者輸出
```

記憶注入時機：
- 系統提示（build 階段，frozen snapshot）
- 用戶訊息前（ephemeral，每 turn prefetch）

---

## 8. Subagent 委派架構（`tools/delegate_tool.py`）

```
DELEGATE_BLOCKED_TOOLS = {
    "delegate_task",  # 防止無限遞歸（MAX_DEPTH = 2）
    "clarify",        # 不允許用戶互動
    "memory",         # 防止寫入共享 MEMORY.md
    "send_message",   # 無跨平台副作用
    "execute_code",   # 子 agent 應逐步推理
}

MAX_CONCURRENT_CHILDREN = 3  # 最多同時 3 個並行 subagent
MAX_DEPTH = 2                # parent(0) → child(1) → rejected(2)
```

Subagent 是完整的 `AIAgent` 實例，有獨立 context、toolset、task_id。
Parent context 只看到委派呼叫和最終摘要。
