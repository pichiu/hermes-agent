# Hermes Agent — 專案總覽與速查

## 一句話總結

Hermes Agent 是 Nous Research 開發的自我進化 AI agent，以 Python 實作，透過學習迴路自動創建技能、改善記憶、跨越 18+ 訊息平台（Telegram、Discord、WhatsApp 等）部署，並同時作為 RL 訓練資料生成平台。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | >=3.11 | 主要語言 |
| Runtime | Node.js | bundled | TUI 前端（Ink/React）|
| LLM 介面 | openai SDK | >=2.21.0 | OpenAI 相容 API 主介面 |
| LLM 介面 | anthropic SDK | >=0.39.0 | Anthropic 直接 API |
| CLI UI | prompt_toolkit | >=3.0.52 | 互動式 CLI 輸入 |
| 終端機渲染 | rich | >=14.3.3 | 格式化輸出 |
| 套件管理 | uv | latest | 虛擬環境與依賴管理 |
| HTTP 客戶端 | httpx | >=0.28.1 | 非同步 HTTP |
| 資料驗證 | pydantic | >=2.12.5 | 資料模型驗證 |
| 資料庫 | SQLite (FTS5) | stdlib | 會話儲存 + 全文搜尋 |
| 排程 | croniter | >=6.0.0 | cron 語法解析 |
| 瀏覽器自動化 | Playwright (chromium) | bundled | 瀏覽器工具 |
| Web 搜尋 | exa-py | >=2.9.0 | 語意搜尋 |
| Web 爬蟲 | firecrawl-py | >=4.16.0 | 網頁內容抓取 |
| TTS | edge-tts | >=7.2.7 | 文字轉語音（免費）|
| STT | faster-whisper | >=1.0.0 | 本地語音辨識 |
| Telegram | python-telegram-bot | >=22.6 | Telegram 整合 |
| Discord | discord.py | >=2.7.1 | Discord 整合 |
| Slack | slack-bolt | >=1.18.0 | Slack 整合 |
| 記憶 | honcho-ai | >=2.0.1 | 辯證用戶建模 |
| MCP | mcp | >=1.2.0 | Model Context Protocol |
| RL | atroposlib | git | Atropos RL 環境 |
| 測試 | pytest | >=9.0.2 | 測試框架（~15k 測試）|
| 容器化 | Docker | - | 生產部署 |
| CI/CD | GitHub Actions | - | 10 個 workflow |
| 建置 | setuptools | >=61.0 | Python 套件建置 |
| Nix | flake.nix | - | 可重現環境 |

---

## 關鍵指令速查

### 安裝與設定

```bash
# 快速安裝（Linux / macOS / WSL2）
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 開發環境安裝
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh               # 安裝 uv + venv + 依賴 + 符號連結

# 手動方式
uv venv venv --python 3.11 && source venv/bin/activate
uv pip install -e ".[all,dev]"
```

### 執行

```bash
hermes                          # 互動式 CLI（預設）
hermes --tui                    # Ink TUI 模式
hermes -p coder                 # 使用 "coder" profile
hermes model                    # 選擇 LLM 提供商和模型
hermes setup                    # 安裝精靈（首次設定）
hermes gateway run              # 啟動訊息閘道（前景）
hermes gateway start            # 啟動閘道（後台服務）
hermes doctor                   # 診斷設定問題
```

### 開發與測試

```bash
scripts/run_tests.sh             # 全套測試（CI 等效）
scripts/run_tests.sh tests/agent/# 特定目錄
scripts/run_tests.sh -v --tb=long# 傳遞 pytest 旗標
```

**⚠️ 切勿直接呼叫 `pytest`**，必須使用 `scripts/run_tests.sh` 確保 CI 環境等效。

### 技能管理

```bash
hermes skills                   # 列出技能
hermes skills install official/github/github-workflow
hermes skills install https://agentskills.io/...
/skills                         # CLI 內技能列表
```

### 設定管理

```bash
hermes config set model.default "anthropic/claude-opus-4.6"
hermes config set display.streaming true
hermes tools                    # 工具設定 UI
hermes logs --follow            # 即時日誌（agent.log）
hermes logs --session <id>      # 特定會話日誌
```

---

## 文件地圖

| 文件 | 內容 |
|------|------|
| [`INDEX.md`](INDEX.md) | 本文件：專案總覽、技術棧、指令速查 |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | 系統架構、模組關係、設計決策 |
| [`DATA_MODEL.md`](DATA_MODEL.md) | 資料模型、SQLite schema、會話生命週期 |
| [`API_SURFACE.md`](API_SURFACE.md) | CLI 指令、工具 API、Gateway API、斜線指令 |
| [`DEV_GUIDE.md`](DEV_GUIDE.md) | 開發者上手指南、測試策略、貢獻流程 |
| [`CODEBASE_MAP.md`](CODEBASE_MAP.md) | 程式碼地圖、檔案用途、模組依賴 |
| [`DISCOVERY_LOG.md`](DISCOVERY_LOG.md) | 探索紀錄、TODO/FIXME、待解問題 |

---

## 專案專屬術語表

| 術語 | 說明 |
|------|------|
| **AIAgent** | 核心 agent 類別（`run_agent.py`），管理對話迴路 |
| **HermesCLI** | 互動式 CLI 協調器（`cli.py`），管理 AIAgent 生命週期 |
| **GatewayRunner** | 訊息閘道主控制器（`gateway/run.py`），管理多平台適配器 |
| **Toolset** | 工具集合（如 `terminal`、`web`），用於啟用/停用一組工具 |
| **Skill** | 技能文件（Markdown），記錄可重用的 agent 行為模式 |
| **Curator** | 背景技能維護程序，自動整合、歸檔過時技能 |
| **SessionDB** | SQLite 會話資料庫（`hermes_state.py`），支援 FTS5 全文搜尋 |
| **Prompt caching** | Anthropic 快取前綴機制，降低多回合對話成本 |
| **HERMES_HOME** | Agent 主目錄（預設 `~/.hermes`），Profile 隔離的核心機制 |
| **Profile** | 完全隔離的 Hermes 實例（不同設定/記憶/API keys）|
| **Memory provider** | 可插拔記憶後端（builtin / honcho / mem0 等）|
| **Platform adapter** | 訊息平台適配器（Telegram / Discord / Slack 等）|
| **Trajectory** | Agent 工具呼叫軌跡，用於 RL 訓練資料生成 |
| **ACP** | Agent Control Protocol，VS Code/Zed/JetBrains 整合協議 |
| **Toolset distribution** | 批次執行時各 toolset 的使用比例設定 |
| **Kanban** | 多代理協調工具（`tools/kanban_tools.py`），用於平行任務 |
| **Budget grace call** | 迭代預算耗盡時的最後一次 API 呼叫（完成當前回應）|
| **SOUL.md** | Agent 人格文件，作為系統提示的首要身份定義 |
| **AGENTS.md** | 代碼倉庫的 AI 助理開發指南（Codex/Cursor 慣例）|
| **Dialectic reasoning** | Honcho 的辯證推理：從對話推導用戶深層模型 |
| **agentskills.io** | 開放技能市集標準，Hermes 相容格式 |
