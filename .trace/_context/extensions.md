# Stage 2.4 — Extension Points

## 擴充點概覽

Hermes Agent 有 4 個主要擴充機制：

```
┌─────────────────────────────────────────────────────────┐
│                   擴充點（Extension Points）              │
├──────────────┬──────────────┬────────────┬──────────────┤
│  Plugin      │  Memory      │  Context   │  Platform    │
│  System      │  Providers   │  Engines   │  Adapters    │
│  (插件系統)  │  (記憶提供者)│  (上下文)  │  (訊息平台)  │
└──────────────┴──────────────┴────────────┴──────────────┘
```

---

## 1. Plugin System（hermes_cli/plugins.py）

### 插件發現來源（4 個）

1. **捆綁插件**：`<repo>/plugins/<name>/`（排除 memory/ 和 context_engine/）
2. **用戶插件**：`~/.hermes/plugins/<name>/`
3. **專案插件**：`./.hermes/plugins/<name>/`（需 `HERMES_ENABLE_PROJECT_PLUGINS=1`）
4. **Pip 插件**：`hermes_agent.plugins` entry-point group

後來源覆蓋前來源（同名插件以後者優先）。

### 插件結構

```
plugins/my-plugin/
├── plugin.yaml           # 元資料（name, description, version）
└── __init__.py           # register(ctx) 函式
```

### 插件 Context API（PluginContext）

```python
def register(ctx: PluginContext):
    # 1. 註冊生命週期鉤子
    ctx.register_hook("pre_tool_call", my_pre_tool_hook)
    ctx.register_hook("post_tool_call", my_post_tool_hook)
    
    # 2. 註冊新工具
    ctx.register_tool(
        name="my_tool",
        description="...",
        handler=my_tool_handler,
        parameters={...},  # JSON Schema
    )
    
    # 3. 註冊 CLI 子指令
    ctx.register_cli_command("my-plugin", "description", my_handler)
    # 讓 `hermes my-plugin <subcmd>` 生效
```

### 生命週期鉤子（VALID_HOOKS，hermes_cli/plugins.py:78）

| 鉤子名稱 | 觸發時機 | 可影響行為 |
|---------|---------|-----------|
| `pre_llm_call` | 每回合第一次 API 前 | 可注入 context（返回 dict 含 `context` key）|
| `pre_api_request` | 每次 API 呼叫前 | 觀察（無返回值） |
| `post_api_request` | 每次 API 呼叫後 | 觀察 |
| `pre_tool_call` | 每次工具呼叫前 | 返回字串可封鎖工具 |
| `post_tool_call` | 每次工具呼叫後 | 觀察（含 duration_ms）|
| `transform_tool_result` | 工具結果返回前 | 返回字串可替換結果 |
| `transform_terminal_output` | 終端機輸出後 | 可轉換輸出 |
| `on_session_start` | 新會話建立（一次性）| 初始化 session 狀態 |
| `on_session_end` | 每個 run_conversation 結束 | 清理 |
| `on_session_finalize` | 會話真正結束時 | 最終清理 |
| `on_session_reset` | /new 或 /reset 後 | 清理 |
| `pre_gateway_dispatch` | Gateway 收到訊息後、分派前 | 可攔截/改寫/丟棄訊息 |
| `pre_approval_request` | 危險指令審核請求前 | 觀察 |
| `post_approval_response` | 審核回應後 | 觀察 |
| `subagent_stop` | 子代理停止時 | 觀察 |

### 現有捆綁插件

```
plugins/
├── memory/           # 記憶提供者（獨立發現路徑）
├── context_engine/   # 上下文引擎（獨立發現路徑）
├── image_gen/        # 圖片生成提供者
├── kanban/           # Kanban 多代理協調
├── example-dashboard/# 儀表板範例
├── observability/    # 可觀測性插件
├── spotify/          # Spotify 整合
├── platforms/        # 額外訊息平台（非 gateway 官方支援的）
├── disk-cleanup/     # 磁碟清理
├── google_meet/      # Google Meet
├── hermes-achievements/ # 成就系統
└── strike-freedom-cockpit/ # 自訂駕駛艙
```

---

## 2. Memory Provider 插件（plugins/memory/<name>/）

### MemoryProvider ABC（agent/memory_provider.py）

```python
class MemoryProvider(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...

    @abstractmethod
    def is_available(self) -> bool: ...
    
    @abstractmethod
    def initialize(self, session_id: str, **kwargs) -> None: ...
    
    @abstractmethod
    def system_prompt_block(self) -> str: ...
    
    @abstractmethod
    def prefetch(self, query: str) -> str: ...
    
    @abstractmethod
    def sync_turn(self, user_msg: str, assistant_msg: str) -> None: ...
    
    @abstractmethod
    def get_tool_schemas(self) -> list: ...
    
    @abstractmethod
    def handle_tool_call(self, name: str, args: dict) -> str: ...
    
    @abstractmethod
    def shutdown(self) -> None: ...
    
    # 可選鉤子（override 啟用）：
    def on_turn_start(self, turn: int, message: str, **kwargs) -> None: ...
    def on_session_end(self, messages: list) -> None: ...
    def on_session_switch(self, new_session_id: str, **kwargs) -> None: ...
    def on_pre_compress(self, messages: list) -> str: ...
    def on_memory_write(self, action, target, content, metadata=None) -> None: ...
    def on_delegation(self, task, result, **kwargs) -> None: ...
```

### 啟用方式

```yaml
# ~/.hermes/config.yaml
memory:
  provider: honcho  # 或 mem0, supermemory, hindsight...
```

### CLI 整合（plugins/memory/<name>/cli.py）

```python
# 若插件定義 register_cli(subparser)，hermes honcho 等子指令自動掛載
def register_cli(subparser):
    subparser.add_parser("setup", help="Configure Honcho...")
```

---

## 3. Context Engine 插件（plugins/context_engine/<name>/）

### ContextEngine ABC（agent/context_engine.py:32）

```python
class ContextEngine(ABC):
    def compress(
        self,
        messages: list,
        system_prompt: str,
        model: str,
        **kwargs
    ) -> tuple[list, str]:
        """壓縮對話歷史，返回 (compressed_messages, updated_system_prompt)"""
```

- 預設實作：`agent/context_compressor.py:ContextCompressor`（摘要型壓縮）
- 外部引擎：`plugins/context_engine/` 下的自訂實作

---

## 4. Platform Adapters（gateway/platforms/）

### BasePlatformAdapter（gateway/platforms/base.py）

```python
class BasePlatformAdapter(ABC):
    async def connect(self) -> None: ...
    async def disconnect(self) -> None: ...
    async def send_message(self, session_key, content, **kwargs) -> None: ...
    async def handle_incoming(self, event: MessageEvent) -> None: ...
    
    # 進階功能（可選）：
    async def send_media(self, ...) -> None: ...
    def get_platform_name(self) -> str: ...
    async def set_typing(self, session_key, typing: bool) -> None: ...
```

### 新增平台

參考 `gateway/platforms/ADDING_A_PLATFORM.md`，基本步驟：

1. 建立 `gateway/platforms/<name>.py`，繼承 `BasePlatformAdapter`
2. 在 `gateway/platform_registry.py` 中登記
3. 在 `gateway/config.py` 中加入設定 schema
4. 實作 token lock（`acquire_scoped_lock()`）防止多 profile 衝突

---

## 5. Skills 擴充（技能文件）

### 技能文件格式（SKILL.md）

```markdown
---
name: my-skill
description: 一句話說明
version: "1.0"
platforms: [linux, macos]
metadata:
  hermes:
    tags: [productivity, automation]
    category: devops
    config:
      MY_KEY: "required config key (stored under skills.config.MY_KEY)"
---

# My Skill

The agent reads this section as instructions when the skill is loaded...
```

### 技能目錄

- **`skills/`**：捆綁技能，預設可用
- **`optional-skills/`**：需要明確安裝（`hermes skills install official/<cat>/<name>`）
- **`~/.hermes/skills/`**：用戶技能（agent-created 或用戶手動創建）
- **agentskills.io**：社群技能市集（`hermes skills install <source>/<name>`）

---

## 6. MCP 伺服器（動態工具擴充）

```yaml
# ~/.hermes/config.yaml
mcp:
  servers:
    github:
      transport: stdio
      command: npx
      args: ["-y", "@modelcontextprotocol/server-github"]
    my-api:
      transport: http
      url: http://localhost:8080/mcp
      filter:
        allow: ["get_data", "list_items"]   # 白名單
```

每個 MCP 伺服器自動產生 `mcp-<server>` toolset，工具注入所有 `hermes-*` platform toolsets。

---

## 擴充點選擇指南

| 需求 | 使用的擴充點 |
|------|------------|
| 新的記憶後端 | `MemoryProvider` ABC |
| 攔截/監控工具呼叫 | Plugin `pre_tool_call` 鉤子 |
| 新的 CLI 子指令 | Plugin `register_cli_command()` |
| 新的訊息平台 | `BasePlatformAdapter` |
| 新的 AI 工具 | `registry.register()` 或 MCP 伺服器 |
| 可重用 agent 行為 | Skills 文件（SKILL.md）|
| 自訂上下文壓縮 | `ContextEngine` ABC |
| 外部工具（無需修改代碼）| MCP 伺服器配置 |
