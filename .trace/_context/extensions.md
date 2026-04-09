# Extension Points — Hermes Agent

## 擴充點總覽

Hermes Agent 提供五個主要擴充機制，讓開發者可以在不動核心的情況下新增功能：

---

## 1. Tool Registry（`tools/registry.py`）

### 擴充方式：自我注冊 Pattern

每個 tool 在 import 時呼叫 `registry.register()`，自動加入 singleton ToolRegistry：

```python
# tools/my_tool.py 範例
from tools.registry import registry

def my_tool_handler(param1: str) -> str:
    return f"Hello {param1}"

registry.register(
    name="my_tool",
    toolset="custom",
    schema={
        "type": "function",
        "function": {
            "name": "my_tool",
            "description": "My custom tool",
            "parameters": {
                "type": "object",
                "properties": {
                    "param1": {"type": "string", "description": "..."}
                },
                "required": ["param1"],
            }
        }
    },
    handler=my_tool_handler,
    check_fn=lambda: True,     # 可用性檢查
    is_async=False,
    description="My custom tool",
    emoji="🔧",
    max_result_size_chars=50_000,
)
```

新增工具後，只需在 `toolsets.py` 的相應 toolset 中加入工具名稱，就會自動出現在 agent 的工具清單中。

### ToolEntry 欄位（`tools/registry.py:24-46`）

| 欄位 | 說明 |
|------|------|
| `name` | 工具名稱（唯一識別） |
| `toolset` | 所屬 toolset 名稱 |
| `schema` | OpenAI function calling JSON schema |
| `handler` | 實際執行函式 |
| `check_fn` | 可用性檢查（API key 是否存在等） |
| `requires_env` | 必要的 env var 清單 |
| `is_async` | 是否為 async handler |
| `description` | 人類可讀描述 |
| `emoji` | 顯示用 emoji |
| `max_result_size_chars` | 結果最大字元數 |

---

## 2. Plugin System（`hermes_cli/plugins.py`）

### 擴充方式：目錄掛載 + Hook 注冊

Plugin 來源（三個）：
1. `~/.hermes/plugins/<name>/`（用戶 plugins）
2. `./.hermes/plugins/<name>/`（專案 plugins，需 `HERMES_ENABLE_PROJECT_PLUGINS=1`）
3. `hermes_agent.plugins` entry-point group（pip 安裝的 plugins）

### Plugin 結構

```
~/.hermes/plugins/my-plugin/
├── plugin.yaml          # Manifest
└── __init__.py          # register(ctx) 函式
```

```yaml
# plugin.yaml
name: my-plugin
version: 1.0.0
description: My plugin
requires_env_vars: []
```

```python
# __init__.py
def register(ctx):
    # 注冊 hook callbacks
    ctx.register_hook("pre_llm_call", my_pre_llm_hook)
    ctx.register_hook("on_session_start", my_session_start)
    
    # 注冊工具
    ctx.register_tool(name="...", schema=..., handler=...)
    
    # 注冊 CLI 子命令
    ctx.register_cli_command(name="myplugin", handler=my_cmd_fn)
    
    # 注冊 memory provider
    from agent.memory_provider import MemoryProvider
    ctx.register_memory_provider(MyMemoryProvider())
```

### 可用 Hook 點（`plugins.py:55-66`）

| Hook | 觸發時機 | 參數 |
|------|---------|------|
| `pre_tool_call` | 工具呼叫前 | tool_name, args |
| `post_tool_call` | 工具呼叫後 | tool_name, result |
| `pre_llm_call` | LLM API 呼叫前（每 turn） | session_id, user_message, history, model, platform |
| `post_llm_call` | LLM API 呼叫後 | response |
| `pre_api_request` | 每次 API request 前 | task_id, model, provider, api_call_count, ... |
| `post_api_request` | 每次 API request 後 | - |
| `on_session_start` | 新 session 開始（首 turn） | session_id, model, platform |
| `on_session_end` | Session 結束 | - |
| `on_session_finalize` | Session 最終結束 | - |
| `on_session_reset` | `/reset` 命令後 | - |

**設計限制**：`pre_llm_call` 只能注入 user message 的 context，不能修改 system prompt（保護 prompt cache prefix）。

### Honcho Plugin（參考實作：`plugins/honcho_plugin/`）

官方 Honcho AI 記憶整合就是一個 plugin，展示了如何實作外部 memory provider。

---

## 3. Memory Provider ABC（`agent/memory_provider.py`）

### 擴充方式：實作 MemoryProvider ABC

```python
from agent.memory_provider import MemoryProvider

class MyMemoryProvider(MemoryProvider):
    @property
    def name(self) -> str:
        return "my-memory"
    
    def is_available(self) -> bool:
        return bool(os.getenv("MY_MEMORY_API_KEY"))
    
    def initialize(self, session_id: str, **kwargs) -> None:
        """Session 開始時初始化"""
        self.session_id = session_id
    
    def system_prompt_block(self) -> str:
        """返回注入系統提示的記憶區塊"""
        return f"<memory>\n{self._load_memories()}\n</memory>"
    
    def prefetch(self, query: str, *, session_id: str = "") -> str:
        """每 turn 前，根據 query 取得相關記憶"""
        return self._semantic_search(query)
    
    def sync(self, user_msg: str, assistant_msg: str, session_id: str = "") -> None:
        """每 turn 後，同步對話到後端"""
        self._store_turn(user_msg, assistant_msg)
    
    def get_tool_schemas(self) -> list:
        """返回該 provider 新增的工具 schemas"""
        return []
    
    def on_session_end(self, session_id: str = "") -> None:
        """Session 結束時清理"""
        pass
```

**限制**：每個 `AIAgent` 實例最多一個外部 memory provider（`memory_manager.py` 拒絕第二個）。

---

## 4. Platform Adapter（`gateway/platforms/base.py`）

### 擴充方式：繼承 BasePlatformAdapter

```python
from gateway.platforms.base import BasePlatformAdapter, MessageEvent, SendResult

class MyPlatformAdapter(BasePlatformAdapter):
    @property
    def platform(self):
        return Platform.MY_PLATFORM
    
    async def start(self) -> bool:
        """啟動平台連線（webhook 或 polling）"""
        ...
    
    async def stop(self) -> None:
        """清理連線"""
        ...
    
    async def send(self, chat_id: str, text: str, **kwargs) -> SendResult:
        """發送訊息到平台"""
        ...
```

步驟：
1. 在 `gateway/platforms/` 建立新的 adapter 檔案
2. 在 `gateway/config.py:Platform` enum 中加入新平台
3. 在 `gateway/run.py:GatewayRunner.start()` 中加入 adapter 建立邏輯
4. 參考 `gateway/platforms/ADDING_A_PLATFORM.md`

---

## 5. Skills（`skills/` 目錄）

### 擴充方式：新增 Markdown SKILL.md

```
~/.hermes/skills/my-skill/
├── SKILL.md           # 主要指令（必要）
├── references/
│   ├── api.md         # 參考文件
│   └── examples.md
└── templates/
    └── output.md      # 輸出模板
```

Skills 是**最低阻力**的擴充方式——不需要 Python，只需 Markdown。
Agent 決定何時呼叫 skill（透過 nudge 機制），或用戶用 `/skill-name` 明確觸發。

### 何時用 Skill vs Tool（CONTRIBUTING.md 摘要）

| 情境 | 選擇 |
|------|------|
| 可用 shell 命令 + 現有工具實現 | Skill |
| 包裝外部 CLI 或 API | Skill |
| 需要自訂 API auth 管理 | Tool |
| 需要處理 binary 資料或串流 | Tool |
| 需要精確執行的邏輯 | Tool |

---

## 6. Terminal Backends（`tools/environments/`）

### 擴充方式：繼承 base.py 環境類別

```python
from tools.environments.base import BaseEnvironment

class MyEnvironment(BaseEnvironment):
    def execute(self, command: str, **kwargs) -> str:
        """在自訂環境中執行命令"""
        ...
    
    def cleanup(self) -> None:
        """清理環境"""
        ...
```

目前六個 backend：local / docker / ssh / modal / daytona / singularity。
透過 `TERMINAL_ENV` env var 或 `config.yaml:terminal.backend` 選擇。

---

## 擴充優先順序建議

```
最簡單 → Skills（SKILL.md，無需程式碼）
         ↓
Plugin hooks（pre/post LLM call，session 生命週期）
         ↓
Tool（registry.register，自訂工具）
         ↓
Memory Provider（MemoryProvider ABC）
         ↓
Platform Adapter（BasePlatformAdapter，新訊息平台）
最複雜 → Terminal Backend（BaseEnvironment，新執行環境）
```
