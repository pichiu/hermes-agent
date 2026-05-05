# Stage 2.3 — 核心領域邏輯

## 核心抽象層概覽

Hermes Agent 的「心臟」由三個相互交織的核心邏輯組成：

1. **自我進化學習迴路**（Skills + Curator + Background Review）
2. **Prompt Caching 架構**（系統提示凍結 + 記憶注入策略）
3. **多層工具分派系統**（Registry + Toolset 解析 + 插件鉤子）

---

## 1. 自我進化學習迴路

### 技能創建觸發機制（run_agent.py:13920）

```python
# 條件：每 N 次工具呼叫迭代觸發一次技能回顧
if (self._skill_nudge_interval > 0              # 預設 10
        and self._iters_since_skill >= self._skill_nudge_interval
        and "skill_manage" in self.valid_tool_names):
    _should_review_skills = True
    self._iters_since_skill = 0

# 觸發背景回顧（在 final_response 送出後才執行）
if final_response and not interrupted and _should_review_skills:
    self._spawn_background_review(
        messages_snapshot=list(messages),
        review_skills=True,
    )
```

`_iters_since_skill` 計數器在 `run_agent.py:10868` 每次工具呼叫後遞增。

### 背景回顧（run_agent.py:3559 `_spawn_background_review`）

```
主 AIAgent 完成回應
    ↓
_spawn_background_review() — 在獨立 Thread 中
    ↓
建立輔助 AIAgent（使用 auxiliary client 而非主 client）
    ↓
注入對話快照作為上下文
    ↓
要求 review_agent 呼叫 skill_manage 來：
    - 建立新技能（若任務夠複雜）
    - 改善現有技能（若發現更好做法）
    - 標記過時技能
    ↓
review_agent 執行後返回（不影響主 session）
```

**關鍵設計**：`review_agent._skill_nudge_interval = 0`（run_agent.py:3632），防止回顧 agent 遞迴觸發自己的技能回顧。

### Curator — 定期技能維護（agent/curator.py:1656）

```
AIAgent 閒置 >= min_idle_hours（預設 2 小時）
且距上次 Curator 執行 >= interval_hours（預設 168 小時/7 天）
    ↓
maybe_run_curator() 觸發
    ↓
建立專用 Curator AIAgent
    ↓
掃描所有 agent-created 技能
    ↓
自動狀態轉換：
    active → stale（30 天未用）
    stale → archived（90 天未用）
    ↓
可 pin（防止自動歸檔）、consolidate（合併重疊技能）、patch（修補落後技能）
    ↓
生成 Curator 報告（儲存到 ~/.hermes/skills/.curator_reports/）
```

**嚴格不變量**：Curator 只觸碰 agent-created 技能，永不自動刪除（只歸檔）。

### 技能生命週期

```
建立 → active → stale（30d） → archived（90d）
         ↑                                    
         └── pinned（防止所有自動轉換）
```

---

## 2. Prompt Caching 架構（agent/prompt_caching.py）

### 系統提示凍結策略（run_agent.py:4870 `_build_system_prompt`）

系統提示組裝層次（從上到下）：

```
1. Agent 身份          SOUL.md（存在時）或 DEFAULT_AGENT_IDENTITY
2. 工具導引文字        HERMES_AGENT_HELP_GUIDANCE
3. 條件工具導引        memory/session_search/skills guidance（取決於載入的工具）
4. Context Files       AGENTS.md + .cursorrules（含注入攻擊掃描）
5. 日期時間            frozen at build time（確保 cache 一致性）
6. 平台提示            PLATFORM_HINTS["cli"] / "telegram" / ...
7. 記憶快照            MEMORY.md + USER.md 的凍結副本
```

**建構一次，永久重用**：
```python
if self._cached_system_prompt is None:
    # 第一回合：從 scratch 建構
    self._cached_system_prompt = self._build_system_prompt(system_message)
    # 持久化到 SQLite（供 gateway 跨訊息重用）
    self._session_db.update_system_prompt(self.session_id, self._cached_system_prompt)
else:
    # 後續回合：從 SQLite 恢復，保持 cache 前綴完全一致
    active_system_prompt = self._cached_system_prompt
```

### 記憶注入策略（保護 cache prefix）

```
❌ 禁止：         system_prompt += memory_context
✅ 正確：         user_message += "\n\n" + memory_context_block

❌ 禁止：         system_prompt += plugin_context
✅ 正確：         user_message += "\n\n" + plugin_context
```

原理：Anthropic prefix caching 需要系統提示在所有 API 呼叫中保持 bit-perfect 一致。

### Anthropic Cache Control（agent/prompt_caching.py）

```python
# 對 Claude 模型套用 cache_control 標記
if self._use_prompt_caching:
    api_messages = apply_anthropic_cache_control(
        api_messages,
        cache_ttl=self._cache_ttl,         # 5min 或 1h
        native_anthropic=self._use_native_cache_layout,
    )
    # 注入 breakpoints: system prompt + 最後 3 條訊息
```

---

## 3. 多層工具分派系統

### 工具 Registry（tools/registry.py）

**Pattern**：Self-registering tools（每個工具檔案 import 時自動註冊）

```python
# tools/terminal_tool.py 示例
@registry.register(
    name="terminal",
    toolset="terminal",
    description="Execute terminal commands...",
)
async def terminal_handler(command: str, task_id: str = None, **kwargs) -> str:
    ...
```

```python
# tools/registry.py 核心
class ToolRegistry:
    _schemas: dict       # name → JSON schema
    _handlers: dict      # name → callable
    _toolsets: dict      # name → toolset name
    
    def dispatch(self, name, args, **kwargs):
        handler = self._handlers[name]
        if asyncio.iscoroutinefunction(handler):
            return _run_async(handler(**args, **kwargs))
        return handler(**args, **kwargs)
```

### Toolset 解析（toolsets.py）

```
用戶設定 enabled_toolsets = ["terminal", "web"]
    ↓
resolve_toolset("terminal") → {"tools": ["terminal", "process"]}
resolve_toolset("web") → {"tools": ["web_search", "web_extract"]}
    ↓
_HERMES_CORE_TOOLS = 預設核心工具集（含 terminal, files, skills, memory 等）
    ↓
hermes-cli toolset = _HERMES_CORE_TOOLS + platform 特有工具
    ↓
MCP toolset 動態追加（mcp-<server> → MCP 工具）
```

**_HERMES_CORE_TOOLS 核心工具**（toolsets.py:31）：
- Web: `web_search`, `web_extract`
- Terminal: `terminal`, `process`
- Files: `read_file`, `write_file`, `patch`, `search_files`
- Vision: `vision_analyze`, `image_generate`
- Skills: `skills_list`, `skill_view`, `skill_manage`
- Browser: `browser_navigate` + 11 個 browser 工具
- AI: `clarify`, `execute_code`, `delegate_task`
- 記憶: `todo`, `memory`, `session_search`
- 排程: `cronjob`
- 訊息: `send_message`（閘道運行時啟用）
- 智慧家電: `ha_*`（HASS_TOKEN 存在時啟用）

### 插件鉤子（hermes_cli/plugins.py）

工具呼叫生命週期中的 6 個鉤點：

```
pre_llm_call     → 每回合第一次 API 呼叫前（可注入 context）
pre_api_request  → 每次 API 呼叫前（觀察）
pre_tool_call    → 每次工具呼叫前（可封鎖）
post_tool_call   → 每次工具呼叫後（觀察，含 duration_ms）
transform_tool_result → 每次工具結果前（可轉換結果）
on_session_start → 新會話建立時（一次性）
on_session_end   → 每個 run_conversation 結束時
```

---

## 4. 上下文壓縮（agent/context_compressor.py:319）

### ContextCompressor 類別

```python
class ContextCompressor(ContextEngine):
    # 觸發閾值：context_length * compression_ratio（預設 0.85）
    threshold_tokens: int
    # 保護前 N 條 + 後 N 條不壓縮
    protect_first_n: int = 3
    protect_last_n: int = 4
    
    def compress(self, messages, system_prompt):
        # 1. 選取中間部分（去掉保護區間）
        # 2. 啟動輔助 AIAgent 生成摘要
        # 3. 替換中間訊息為單一摘要訊息
        # 4. 建立新 SQLite 會話（parent_session_id 連結原始會話）
        # 5. 清除 _cached_system_prompt
```

### 觸發點

1. **preflight 壓縮**（run_agent.py:10680）：回合開始前評估
2. **runtime 壓縮**：API 返回 context_length 錯誤時觸發

---

## 5. 多提供商路由（agent/error_classifier.py）

### FailoverReason 分類

```python
class FailoverReason(Enum):
    RATE_LIMIT = "rate_limit"
    OVERLOAD = "overload"
    AUTH_ERROR = "auth_error"
    CONTEXT_LENGTH = "context_length"
    NETWORK_ERROR = "network_error"
```

### Fallback 邏輯

```
主提供商錯誤（rate limit / overload / auth）
    ↓
_try_activate_fallback()
    ↓
遍歷 fallback_chain（config.yaml fallback.chain）
    ↓
切換到備用提供商（改 base_url + api_key）
    ↓
繼續當前回合（retry_count = 0 重置）
```

**Credential Pool**（agent/credential_pool.py）：多 API key 輪換，提升 RPM 上限。

---

## 6. 會話資料庫（hermes_state.py:SessionDB）

### SQLite 結構

```sql
-- 會話表
CREATE TABLE sessions (
    id TEXT PRIMARY KEY,
    platform TEXT,
    title TEXT,
    system_prompt TEXT,           -- 凍結的系統提示
    parent_session_id TEXT,       -- 壓縮後的前代連結
    created_at INTEGER,
    ended_at INTEGER,
    end_reason TEXT,
    source TEXT,
    model TEXT,
    message_count INTEGER
);

-- 訊息表（支援 FTS5 全文搜尋）
CREATE VIRTUAL TABLE session_messages USING fts5(
    session_id, role, content,
    tokenize="unicode61"
);
```

### session_search 工具

FTS5 搜尋橫跨所有歷史對話，支援 `session_search("kubernetes deployment")` 召回相關歷史記憶。

---

## 關鍵設計模式

| 模式 | 應用位置 | 說明 |
|------|---------|------|
| Self-registering tools | `tools/registry.py` | import 時自動發現 |
| Cache-aware updates | `run_agent.py:_build_system_prompt` | 系統提示只建一次 |
| Background fork | `run_agent.py:_spawn_background_review` | 技能回顧不阻塞主流程 |
| Plugin hooks | `hermes_cli/plugins.py` | 6 個生命週期鉤點 |
| Credential pool | `agent/credential_pool.py` | 多 key 輪換提升 RPM |
| Profile isolation | `hermes_constants.py:get_hermes_home()` | HERMES_HOME 隔離 |
| Deferred invalidation | slash commands + `--now` | cache-aware 設定更新 |
