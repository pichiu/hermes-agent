# Stage 2.1 — Entry Points

## 啟動進入點概覽

Hermes Agent 有 6 個不同的啟動進入點，共享 AIAgent 核心類別：

```
┌─────────────────────────────────────────────────────────┐
│                   Entry Points                           │
├─────────────┬──────────────┬──────────────┬─────────────┤
│  CLI 互動   │  Messaging   │  ACP 整合    │  批次/RL    │
│  hermes     │  Gateway     │  IDE 編輯器  │  研究工具   │
│ (cli.py)    │(gateway/)    │(acp_adapter/)│(batch/env)  │
└──────┬──────┴──────┬───────┴──────┬───────┴──────┬──────┘
       │             │              │              │
       └─────────────┴──────────────┴──────────────┘
                            │
                     AIAgent(run_agent.py)
```

## 1. CLI 互動模式（主要進入點）

### 進入點鏈

```
hermes 腳本 (hermes_cli/main.py:main → __main__)
  ↓
hermes_cli/main.py:main()           # argparse 路由
  ↓
hermes_cli/main.py:cmd_chat()       # line 1219
  ↓
cli.py:main()                       # 組裝 HermesCLI kwargs
  ↓
cli.py:HermesCLI.__init__()         # line 1994
  ↓
cli.py:HermesCLI._init_agent()      # 延遲初始化 AIAgent
  ↓
run_agent.py:AIAgent.__init__()     # line 897
```

### 初始化序列（hermes_cli/main.py）

1. `_apply_profile_override()` — 在任何模組 import 前設定 `HERMES_HOME` 環境變數（profile 多實例支援）
2. `load_hermes_dotenv()` — 從 `~/.hermes/.env` 載入 API keys
3. `setup_logging(mode="cli")` — 初始化 `agent.log`、`errors.log`（`hermes_logging.py:setup_logging()`）
4. `_apply_ipv4()` — 選擇性強制 IPv4（network.force_ipv4 設定）
5. `_has_any_provider_configured()` — 首次運行守衛：無提供商設定則引導 `hermes setup`
6. `prefetch_update_check()` — 背景版本更新查詢
7. `sync_skills(quiet=True)` — 同步捆綁技能（`tools/skills_sync.py`）

### HermesCLI 初始化（cli.py:1994）

```python
self.console = Console()                # Rich console
self.config = CLI_CONFIG                # load_cli_config() — YAML + 環境變數
self._session_db = SessionDB()          # SQLite 會話資料庫（hermes_state.py）
```

### AIAgent 延遲初始化（cli.py:_init_agent → ~line 3638）

```python
self.agent = AIAgent(
    model=...,                          # 來自 config.yaml
    platform="cli",                     # 告知 agent 平台
    session_id=self.session_id,         # SQLite 會話 ID
    session_db=self._session_db,        # 共享 SessionDB 實例
    clarify_callback=...,               # 互動式問題回調
    tool_progress_callback=...,         # 工具進度顯示回調
)
```

## 2. Messaging Gateway（`hermes gateway`）

### 進入點鏈

```
hermes_cli/main.py:cmd_gateway_run()    # gateway run 子指令
  ↓
gateway/run.py:run_gateway(config)      # line ~15004
  ↓
gateway/run.py:GatewayRunner.__init__() # line 1088
  ↓
GatewayRunner.run()                     # 非同步主迴圈
  ↓
platform adapters 啟動（asyncio）
```

### GatewayRunner 初始化序列（gateway/run.py:1088）

1. 載入 `GatewayConfig`（`gateway/config.py`）
2. 初始化 `SessionStore`（`gateway/session.py`）— 管理多用戶會話
3. 初始化 `DeliveryRouter`（`gateway/delivery.py`）— 訊息遞送
4. 啟動各平台適配器（`gateway/platforms/`）
5. 每個平台適配器在收到訊息時透過 `GatewaySession` 建立 `AIAgent` 實例

### 特點

- **非同步架構**：asyncio 事件迴圈
- **多用戶隔離**：每個 session_key 有獨立的 AIAgent 實例
- **平台 registry**：`gateway/platform_registry.py` 動態發現平台適配器

## 3. ACP 適配器（`hermes acp`）

### 進入點鏈

```
hermes_cli/main.py:cmd_acp()
  ↓
acp_adapter/entry.py:main()
  ↓
acp_adapter/server.py:ACPServer
  ↓ (每個 IDE 請求)
acp_adapter/session.py → AIAgent
```

- 透過 stdio JSON-RPC 與 VS Code/Zed/JetBrains 通訊
- 提供工具呼叫結果、檔案差異、終端指令的 IDE 原生渲染

## 4. Python Library（程式碼庫方式）

```python
from run_agent import AIAgent

agent = AIAgent(
    base_url="http://localhost:30000/v1",
    model="claude-sonnet-4-6"
)
response = agent.run_conversation("Tell me about Python updates")
```

- 最簡介面：`agent.chat(message)` → `str`
- 完整介面：`agent.run_conversation(...)` → `dict`

## 5. 批次處理（`batch_runner.py`）

```python
# batch_runner.py:main() — Fire CLI
python batch_runner.py --tasks tasks.json --workers 4
```

- 平行執行多個 AIAgent 實例
- 輸出 ShareGPT 格式軌跡（供 RL 訓練）
- 使用 `IterationBudget` 控制每個任務的工具呼叫上限

## 6. RL 訓練環境（`environments/`）

```python
# environments/ 中的 Atropos 環境
# 透過 rl_cli.py 或 Atropos API 啟動
```

- 與 Atropos 框架整合
- 使用 GRPO + LoRA 訓練工具呼叫模型

## AIAgent.__init__ 關鍵初始化步驟（run_agent.py:897）

```
__init__
  ├── 解析提供商設定（base_url, api_key, provider）
  ├── get_tool_definitions()         → model_tools.py → tools/registry.py（自動發現工具）
  ├── _build_system_prompt()         → agent/prompt_builder.py
  ├── 載入記憶體（_init_memory）      → agent/memory_manager.py
  ├── 載入技能（_load_active_skills） → agent/skill_commands.py
  ├── 載入 context files             → SOUL.md, AGENTS.md, .cursorrules
  ├── 初始化 SessionDB               → hermes_state.py:SessionDB
  └── 初始化工具護衛（tool_guardrails） → tools/approval.py
```

## 設定載入優先順序

```
環境變數（HERMES_HOME, API keys）
  ↑ 優先於
~/.hermes/.env（API keys 專屬）
  ↑ 優先於
~/.hermes/config.yaml（行為設定）
  ↑ 優先於
cli-config.yaml.example 中的 hardcoded 預設值
```

## 關鍵路徑檔案

| 檔案 | 角色 |
|------|------|
| `hermes` | Shell 啟動腳本 → `hermes_cli/main.py:main` |
| `hermes_cli/main.py` | argparse 路由、profile 管理、初始化序列 |
| `cli.py:HermesCLI` | 互動式 CLI 協調器，管理 AIAgent 生命週期 |
| `run_agent.py:AIAgent` | 核心 agent 類別，所有路徑都通過這裡 |
| `model_tools.py` | 工具協調層，觸發工具自動發現 |
| `tools/registry.py` | 工具自動發現 registry |
| `gateway/run.py:GatewayRunner` | 訊息閘道主控制器 |
| `hermes_constants.py:get_hermes_home()` | 所有路徑的單一真實來源 |
