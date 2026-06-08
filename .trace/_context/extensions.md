# Stage 2.4 Extension Points

## 四大擴展機制

### 1. Plugin 系統

**位置**：`plugins/` 目錄（含預裝插件）、`~/.hermes/plugins/`（用戶自定義）

**Plugin 載入流程**：
```python
# model_tools.py（⚠️ 路徑推測，需驗證）
discover_plugins() → scan ~/.hermes/plugins/ + plugins/
register_plugin(ctx)  # 每個 plugin 實作 register(ctx) 函式
```

**Plugin 可做的事**：
- `ctx.register_hook(event_name, callback)` — 觀測者 hook（只讀）
- `ctx.register_middleware(kind, callback)` — Middleware（可改寫請求）
- `ctx.register_cli(subparser)` — 新增 `hermes <cmd>` 子命令
- 自定義 memory provider（實作 `MemoryProvider` ABC）

**已知內建 Plugin 清單**：

| Plugin | 路徑 | 功能 |
|--------|------|------|
| Observability | `plugins/observability/` | Langfuse, NeMo Relay 追蹤 |
| Memory | `plugins/memory/` | Honcho, mem0, Supermemory 等 |
| Browser | `plugins/browser/` | 瀏覽器自動化整合 |
| Google Meet | `plugins/google_meet/` | 會議整合 |
| Teams Pipeline | `plugins/teams_pipeline/` | Microsoft Teams |
| Context Engine | `plugins/context_engine/` | Context 注入系統 |
| Dashboard Auth | `plugins/dashboard_auth/` | Web dashboard 認證 |
| Image Gen | `plugins/image_gen/` | 圖像生成 provider |
| Kanban | `plugins/kanban/` | Kanban 看板 |
| Platforms | `plugins/platforms/` | 額外 messaging 平台 |
| Security | `plugins/security/` | 安全策略 |
| Observability | `plugins/observability/` | 可觀測性整合 |

**啟用 Plugin**：
```bash
hermes plugins enable <plugin-name>
hermes plugins disable <plugin-name>
hermes plugins list
```

---

### 2. Observer Hook 系統（只讀）

**文件**：`docs/observability/README.md`，schema version: `hermes.observer.v1`

**Hook 事件清單**：

| Hook | 時機 |
|------|------|
| `on_session_start` | 新 session 建立 |
| `on_session_end` | run_conversation 結束 |
| `on_session_finalize` | Session 身份撤銷 |
| `on_session_reset` | Session 重置 |
| `pre_llm_call` | Turn 開始前（可注入 ephemeral context） |
| `post_llm_call` | Turn 完成後（含最終回應） |
| `pre_api_request` | LLM API 呼叫前（含完整 request） |
| `post_api_request` | LLM API 成功後（含 response） |
| `api_request_error` | LLM API 失敗後 |
| `pre_tool_call` | 工具執行前（可 block） |
| `post_tool_call` | 工具執行後（含 result） |
| `transform_tool_result` | 可替換工具結果 |
| `transform_llm_output` | 可替換最終 LLM 輸出 |
| `pre_approval_request` | 危險命令審批前 |
| `post_approval_response` | 審批後 |
| `subagent_start` | Subagent 建立 |
| `subagent_stop` | Subagent 完成 |

**性能優化**：Telemetry payload 建構 gated behind `has_hook(event_name)` — 若無人監聽，不建構資料

---

### 3. Middleware 系統（可改寫）

**文件**：`docs/middleware/README.md`，schema version: `hermes.middleware.v1`

**Middleware 種類**：

| Kind | 作用時機 | 可做的事 |
|------|---------|----------|
| `llm_request` | LLM API 呼叫前 | 改寫 provider kwargs（model、messages、extra_body 等） |
| `tool_request` | 工具執行前 | 改寫工具參數（比 guardrails/approval 更早） |
| `llm_execution` | LLM API 執行時 | 包裝/替換實際 API 呼叫 |
| `tool_execution` | 工具執行時 | 包裝/替換實際工具執行 |

**Middleware 鏈**：多個 plugin 可以 chain（registration order），execution middleware 使用 `next_call()` 模式

**注意**：`tool_request` middleware 在 approval 之前執行，改寫的參數才是 guardrails 看到的值

---

### 4. Toolset 系統

**位置**：`toolsets.py`

Toolset 是工具的命名集合，用戶可以選擇啟用/停用哪些工具集：

```bash
hermes tools enable web
hermes tools disable terminal
```

**常見 Toolset**：
- `core` — 基本工具（memory、skills、file 等）
- `terminal` — Shell 執行
- `web` — 網路搜尋、瀏覽
- `browser` — 瀏覽器自動化
- `image-gen` — 圖像生成
- `tts` — 語音合成
- `mcp` — MCP 整合
- `delegation` — Subagent 委派

---

### 5. Platform Adapter 系統（Gateway 擴展）

**新增平台**：實作 `gateway/platforms/base.py::BasePlatformAdapter` ABC

```python
class BasePlatformAdapter(ABC):
    # 必實作方法（⚠️ 具體方法需驗證）
    async def start(self)
    async def send_message(self, chat_id, text, ...)
    async def handle_webhook(self, request)
```

**Plugin Platform 支援**：Platform 也可以通過 plugin 系統擴展（`plugins/platforms/`），通過 `gateway/platform_registry.py` 注冊

**文件**：`gateway/platforms/ADDING_A_PLATFORM.md`

---

### 6. Skill 擴展（最輕量）

Skills 是 Markdown 文件，任何人都可以：
- 在對話中讓 agent 建立 skill（`/skills create`）
- 從 Skills Hub 安裝（`hermes skills install <name>`）
- 直接放入 `~/.hermes/skills/` 目錄
- 發布到 agentskills.io

---

### 如何新增功能而不動核心

| 需求 | 推薦方式 |
|------|---------|
| 新的程序化工作流 | 建立 Skill（Markdown 文件） |
| 新的 messaging 平台 | 實作 `BasePlatformAdapter`，放入 `plugins/platforms/` |
| 新的記憶 backend | 實作 `MemoryProvider` ABC，發布為獨立 repo |
| Telemetry/追蹤 | Observer Hook plugin |
| 請求路由/改寫 | Middleware plugin |
| 新的 LLM provider | 在 `hermes_constants.py` 加入 provider 設定，通常不需要新 adapter（OpenAI-compatible） |
| 新的工具 | 在 `tools/` 建立 `.py` 並呼叫 `registry.register()` |
