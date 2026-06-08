# Hermes Agent — 專案總覽與速查

## 一句話總結

**Hermes Agent** 是由 Nous Research 開發的開源、自我改善 AI agent 框架，給需要長期運行 AI 助手的開發者和重度用戶使用，解決「每次對話都要從零開始」的問題——它從經驗建立可重用 skill、自動整理記憶、支援 Telegram/Discord/Slack 等 20+ 平台訊息接收，並可運行在 $5 VPS 到 GPU 叢集的任何環境。

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | 3.11–3.13 | 主要 agent 核心 |
| Runtime | Node.js | 20+ | TUI 前端、Web dashboard |
| Package manager | uv | latest | Python 依賴管理 |
| HTTP 框架 | FastAPI | >=0.104 | ACP/MCP server |
| ASGI | Uvicorn | >=0.24 | FastAPI 運行時 |
| LLM SDK | openai | 2.24.0 | 統一 LLM 呼叫介面 |
| LLM SDK | anthropic | 0.86.0（可選） | Anthropic 原生 API |
| 資料驗證 | pydantic | 2.13.4 | 資料模型 |
| CLI 框架 | python-fire | 0.7.1 | 子命令路由 |
| Terminal UI | prompt_toolkit | 3.0.52 | 互動式 CLI REPL |
| HTTP client | httpx | 0.28.1 | 外部 API 呼叫 |
| Retry | tenacity | 9.1.4 | API retry 邏輯 |
| Cron | croniter | 6.0.0 | 排程任務 |
| Config | ruamel.yaml + python-dotenv | | YAML + 環境變數 |
| 模板引擎 | jinja2 | 3.1.6 | 系統提示詞渲染 |
| 圖像處理 | Pillow | 12.2.0 | 視覺工具圖像縮放 |
| Auth | PyJWT | 2.12.1 | Skills Hub JWT |
| 前端 | React + Vite（⚠️ 推測） | | Web dashboard |
| 容器化 | Docker | | 部署 |
| CI/CD | GitHub Actions | | 自動化測試/發布 |
| Nix | flake.nix | | 開發環境 reproducibility |
| Process mgmt | psutil | 7.2.2 | 跨平台 process 管理 |

## 關鍵指令速查

### 安裝與啟動

```bash
# 安裝
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

# 從原始碼執行
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh
./hermes

# 基本指令
hermes              # 啟動互動 CLI
hermes model        # 選擇 LLM provider 和模型
hermes tools        # 設定啟用的工具
hermes gateway      # 啟動 messaging gateway
hermes setup        # 完整設定嚮導
hermes doctor       # 診斷問題
hermes update       # 更新版本
```

### 開發者指令

```bash
# 開發環境建置
uv venv .venv --python 3.11
source .venv/bin/activate
uv pip install -e ".[all,dev]"

# 執行測試
scripts/run_tests.sh

# 直接呼叫 Python 模組
python -m gateway.run          # 啟動 gateway
python batch_runner.py         # 批次執行
python mcp_serve.py            # 啟動 MCP server
```

### Session 內斜線命令

| 命令 | 說明 |
|------|------|
| `/new` / `/reset` | 開始新對話 |
| `/model [provider:model]` | 切換模型 |
| `/skills` | 瀏覽 skills |
| `/compress` | 手動壓縮 context |
| `/usage` | 查看 token 使用量 |
| `/retry` | 重試上一個回應 |
| `/stop` | 中斷當前工作（gateway） |
| `/personality [name]` | 切換 agent 人格 |
| `/platforms` | 查看平台狀態 |

## 文件地圖

| 文件 | 說明 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 系統架構、元件圖、Sequence diagram |
| [DATA_MODEL.md](./DATA_MODEL.md) | 資料模型、狀態管理 |
| [API_SURFACE.md](./API_SURFACE.md) | CLI 命令參考、API endpoint、工具清單 |
| [DEV_GUIDE.md](./DEV_GUIDE.md) | 開發者上手指南、測試、貢獻流程 |
| [CODEBASE_MAP.md](./CODEBASE_MAP.md) | 程式碼地圖、速查表 |
| [DISCOVERY_LOG.md](./DISCOVERY_LOG.md) | 探索紀錄、待解問題、技術債 |

## 專案專屬術語表

| 術語 | 定義 |
|------|------|
| **Skill** | Markdown 格式的程序性記憶文件，agentskills.io 標準相容。Agent 可自動建立、搜尋、改善 |
| **Toolset** | 工具的命名集合（如 `web`、`terminal`、`browser`），可整組啟用/停用 |
| **Gateway** | 長駐進程，統一接收 20+ messaging 平台訊息，路由到 AIAgent |
| **ACP** | Agent Client Protocol，讓 Zed/JetBrains 等編輯器直接使用 Hermes |
| **MCP** | Model Context Protocol，讓 Hermes 與外部工具/資源互通 |
| **Curator** | 背景執行的 skill 維護 agent，定期整理/歸檔老舊 skills |
| **Background Review** | 每個 turn 後自動觸發的背景分析，決定是否建立/改善 skill |
| **Session** | 一次連續對話的記錄單位，有唯一 session_id |
| **Turn** | Session 中一次用戶輸入到 agent 回應的完整週期 |
| **Trajectory** | 完整的對話記錄（含工具呼叫），用於研究/訓練 |
| **HERMES_HOME** | 用戶資料目錄（預設 `~/.hermes`），含 config、memory、skills、sessions |
| **Nous Portal** | Nous Research 的統一訂閱服務，提供 300+ 模型和 Tool Gateway |
| **Tool Gateway** | Nous Portal 的工具路由服務（網路搜尋、圖像生成、TTS 等） |
| **Honcho** | plastic-labs 開發的 dialectic user modeling 服務，Hermes 可整合 |
| **Observer Hook** | 只讀 telemetry hook，不影響執行（Schema: `hermes.observer.v1`） |
| **Middleware** | 可改寫 LLM 請求或工具參數的執行前後攔截器（Schema: `hermes.middleware.v1`） |
| **IterationBudget** | 每個 turn 的最大工具呼叫次數預算（預設 90），可在 subagent 間繼承 |
| **FTS5** | SQLite 的全文搜尋引擎，用於 skill 搜尋和 session 搜尋 |
| **Ephemeral system prompt** | 臨時注入系統提示詞，不持久化 |
| **Subagent** | 由主 agent 委派的子 agent（delegate_tool），可並行執行 |
