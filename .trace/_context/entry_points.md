# Entry Points — Hermes Agent

## 程式入口點總覽

Hermes Agent 有四個主要入口點，全部定義於 `pyproject.toml:101-104`：

```toml
[project.scripts]
hermes       = "hermes_cli.main:main"       # 主 CLI（hermes 命令）
hermes-agent = "run_agent:main"              # 直接執行 AIAgent（batch/dev）
hermes-acp   = "acp_adapter.entry:main"     # ACP server（editor 整合）
```

---

## 入口點 1：`hermes` CLI (`hermes_cli/main.py`)

### 啟動流程

```
hermes_cli/main.py:main()
  │
  ├─ _apply_profile_override()      (main.py:83)
  │    └─ 讀取 --profile/-p 或 ~/.hermes/active_profile
  │    └─ 設定 HERMES_HOME env var（module import 前執行）
  │
  ├─ load_hermes_dotenv()           (hermes_cli/env_loader.py)
  │    └─ 載入 ~/.hermes/.env，再 fallback 到 ./.env
  │
  ├─ _setup_logging(mode="cli")     (hermes_logging.py)
  │    └─ 寫入 ~/.hermes/logs/agent.log + errors.log
  │
  └─ main() → argparse → subcommand dispatch
       ├─ (無 subcommand) → cmd_chat()      → HermesCLI → AIAgent
       ├─ "setup"         → cmd_setup()     → 互動式設置精靈
       ├─ "model"         → cmd_model()     → 模型選擇 TUI
       ├─ "tools"         → cmd_tools()     → 工具啟用/停用 TUI
       ├─ "gateway"       → cmd_gateway()   → GatewayRunner.start()
       ├─ "cron"          → cmd_cron()      → Cron 排程管理
       ├─ "doctor"        → cmd_doctor()    → 環境診斷
       ├─ "honcho"        → cmd_honcho()    → Honcho 記憶管理
       ├─ "skills"        → cmd_skills()    → Skills Hub 管理
       ├─ "acp"           → cmd_acp()       → ACP server 啟動
       ├─ "sessions"      → sessions browse → SessionDB picker
       └─ "update"/"uninstall"/...
```

### 關鍵觸發點

- Profile 解析必須在所有模組 import 前執行（`main.py:83-137`），因為模組會在 import 時快取 `HERMES_HOME`。
- `_has_any_provider_configured()` 函式（`main.py:181`）在啟動時檢查是否有 LLM provider，未設定時引導用戶執行 `hermes setup`。

---

## 入口點 2：CLI 互動聊天 (`cli.py:HermesCLI`)

### HermesCLI 初始化

`cli.py` 包含 `HermesCLI` 類別（約 14,000 行），是互動式 CLI 的 orchestrator：

```
HermesCLI.__init__()
  ├─ 解析 CLI flags（--model, --tool, --toolset, --debug, --quiet, --worktree...）
  ├─ 載入設定（config.yaml + .env）
  ├─ 初始化 SessionDB（hermes_state.py:SessionDB）
  ├─ 建立 AIAgent 實例（run_agent.py:AIAgent）
  ├─ 初始化 prompt_toolkit Application（完整 TUI）
  └─ 進入 run() 主迴圈
```

### CLI 工具初始化 (lazy)

工具在 `model_tools.py` import 時（`_discover_tools()`）自動注冊，無需手動觸發。

---

## 入口點 3：`run_agent.py:AIAgent.__init__`

直接使用 `AIAgent` 的場景（batch processing、subagent、RL training）。

### 初始化步驟（`run_agent.py:454-750`）

```
AIAgent.__init__()
  ├─ 解析 api_mode（chat_completions / codex_responses / anthropic_messages）
  │    └─ 自動偵測：URL pattern 判斷（api.anthropic.com → anthropic_messages）
  │
  ├─ 初始化 OpenAI client（openai.OpenAI，provider 無關）
  ├─ 載入工具 definitions（get_tool_definitions() → tools/registry.py）
  ├─ 初始化 ContextCompressor（agent/context_compressor.py）
  ├─ 初始化 SessionDB 連線（hermes_state.py:SessionDB.connect()）
  ├─ 初始化 MemoryManager + BuiltinMemoryProvider（agent/memory_manager.py）
  ├─ 初始化 IterationBudget（預設 max=90）
  ├─ 載入 CheckpointManager（tools/checkpoint_manager.py）
  └─ 若有 honcho plugin，載入 HonchoPlugin
```

---

## 入口點 4：Gateway (`gateway/run.py:GatewayRunner`)

### Gateway 啟動流程

```
hermes gateway start → hermes_cli/main.py:cmd_gateway()
  └─ 執行 gateway.run 模組（subprocess 或 asyncio）

gateway/run.py 頂層（import time）：
  ├─ _ensure_ssl_certs()           # 自動偵測 CA bundle
  ├─ load_hermes_dotenv()          # 載入 .env
  ├─ config.yaml → os.environ 橋接 # terminal.backend, auxiliary.* 等
  ├─ print_config_warnings()       # 設定驗證
  └─ os.environ["HERMES_QUIET"] = "1"

GatewayRunner.start()  (gateway/run.py:1044)
  ├─ 載入 ~/.hermes/config.yaml gateway 設定
  ├─ 為每個啟用的平台建立 adapter 實例
  │    └─ TelegramAdapter / DiscordAdapter / SlackAdapter / ...
  ├─ 啟動各 adapter（asyncio.gather）
  ├─ 啟動 CronScheduler（如果啟用）
  └─ 進入 asyncio event loop，等待 platform events
```

---

## 入口點 5：ACP Server (`acp_adapter/entry.py`)

```
hermes-acp → acp_adapter/entry.py:main()
  └─ 啟動 FastAPI/uvicorn HTTP server
  └─ 接受 VS Code / Zed / JetBrains 的工具呼叫請求
  └─ 橋接到 AIAgent 執行
```

---

## 工具依賴鏈（Import 順序）

```
tools/registry.py        ← 無依賴，singleton ToolRegistry
       ↑
tools/*.py               ← 在 import 時呼叫 registry.register()
       ↑
model_tools.py           ← import 所有 tools/*.py，觸發自動注冊
       ↑
run_agent.py / cli.py / batch_runner.py / environments/*.py
```

循環 import 預防：`tools/registry.py` 不 import `model_tools.py` 或任何 tool 檔案。

---

## 初始化時的 DI/設定注入

| 設定項 | 來源 | 注入位置 |
|--------|------|---------|
| HERMES_HOME | env var / ~/.hermes | `hermes_constants.get_hermes_home()` |
| LLM model | config.yaml model.default | `AIAgent.__init__:model` |
| LLM provider | config.yaml model.provider / HERMES_INFERENCE_PROVIDER | `hermes_cli/runtime_provider.py` |
| API key | ~/.hermes/.env | `load_hermes_dotenv()` |
| Enabled toolsets | config.yaml / CLI flag | `AIAgent.__init__:enabled_toolsets` |
| Terminal backend | config.yaml terminal.backend / TERMINAL_ENV | `terminal_tool.py` |
| Memory provider | config.yaml memory.provider / plugins | `AIAgent.__init__ → MemoryManager` |
| Platform (CLI/gateway) | `AIAgent.__init__:platform` | `_build_system_prompt()` platform hint |
