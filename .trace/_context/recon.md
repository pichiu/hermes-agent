# Stage 1 偵察報告 — Hermes Agent

## 專案基本資訊

- **專案名稱**: Hermes Agent
- **版本**: 0.12.0
- **維護者**: Nous Research
- **授權**: MIT
- **專案定位**: 自我進化的 AI agent，具備學習迴路、技能創建與多平台部署能力

## 技術棧

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | >=3.11 | 主要語言 |
| Node.js | Node.js | bundled | TUI 前端 (Ink/React) |
| LLM 客戶端 | openai SDK | >=2.21.0 | 主要 API 介面（OpenAI 相容格式） |
| LLM 客戶端 | anthropic SDK | >=0.39.0 | Anthropic 直接 API |
| CLI 框架 | prompt_toolkit | >=3.0.52 | 互動式 TUI 輸入 |
| TUI 渲染 | rich | >=14.3.3 | 終端機格式化輸出 |
| 套件管理 | uv | latest | Python 虛擬環境與依賴管理 |
| HTTP 客戶端 | httpx | >=0.28.1 | 非同步 HTTP |
| 資料驗證 | pydantic | >=2.12.5 | 資料模型驗證 |
| 排程 | croniter | >=6.0.0 | cron 語法解析 |
| Web 搜尋 | exa-py | >=2.9.0 | Exa 搜尋引擎整合 |
| 網頁爬蟲 | firecrawl-py | >=4.16.0 | 網頁內容抓取 |
| TTS | edge-tts | >=7.2.7 | 免費文字轉語音 |
| STT | faster-whisper | >=1.0.0 | 本地語音辨識 |
| Messaging | python-telegram-bot | >=22.6 | Telegram 整合 |
| Messaging | discord.py | >=2.7.1 | Discord 整合 |
| Messaging | slack-bolt | >=1.18.0 | Slack 整合 |
| 記憶體 | honcho-ai | >=2.0.1 | 用戶模型（記憶插件） |
| MCP | mcp | >=1.2.0 | Model Context Protocol 整合 |
| RL 訓練 | atroposlib | git | Atropos RL 環境 |
| 測試 | pytest | >=9.0.2 | 測試框架 |
| Linting | ruff | latest | 程式碼靜態分析 |
| 容器化 | Docker | - | 部署容器 |
| CI/CD | GitHub Actions | - | 自動化測試與部署 |
| 套件格式 | setuptools | >=61.0 | Python 套件建置 |
| Nix | flake.nix | - | Nix 環境管理 |

## 目錄結構（3 層深度）

```
hermes-agent/
├── run_agent.py              # AIAgent 核心類別（~14k LOC）— 對話迴路主體
├── cli.py                    # HermesCLI 類別（~11k LOC）— 互動式 CLI 協調器
├── model_tools.py            # 工具協調層：discover_builtin_tools, handle_function_call
├── toolsets.py               # Toolset 定義：_HERMES_CORE_TOOLS 清單
├── hermes_state.py           # SessionDB — SQLite 會話儲存（FTS5 全文搜尋）
├── hermes_constants.py       # get_hermes_home()、display_hermes_home() — 路徑管理
├── hermes_logging.py         # setup_logging() — 多日誌檔案管理
├── hermes_time.py            # 時間工具函式
├── hermes                    # CLI 啟動腳本（指向 hermes_cli.main:main）
├── batch_runner.py           # 平行批次處理
├── mcp_serve.py              # MCP 伺服器模式
├── mini_swe_runner.py        # 輕量 SWE 評估執行器
├── rl_cli.py                 # RL 訓練 CLI
├── trajectory_compressor.py  # 軌跡壓縮（訓練資料生成）
├── toolset_distributions.py  # Toolset 分佈統計
├── utils.py                  # 通用工具函式
├── setup-hermes.sh           # 開發環境安裝腳本
├── pyproject.toml            # 套件配置、依賴定義
├── package.json              # Node.js 依賴（TUI 前端）
├── docker-compose.yml        # Docker 部署配置
├── Dockerfile                # 容器映像定義
├── flake.nix                 # Nix flake 定義
├── cli-config.yaml.example   # CLI 設定範例
├── .env.example              # 環境變數範例
│
├── agent/                    # Agent 內部模組
│   ├── anthropic_adapter.py  # Anthropic 原生 API 轉接器
│   ├── bedrock_adapter.py    # AWS Bedrock 轉接器
│   ├── gemini_native_adapter.py # Gemini 原生 API
│   ├── context_compressor.py # 對話上下文壓縮
│   ├── memory_manager.py     # 記憶體管理（多插件協調）
│   ├── memory_provider.py    # 記憶體提供者 ABC
│   ├── prompt_builder.py     # 系統提示建構
│   ├── prompt_caching.py     # Anthropic prompt caching 支援
│   ├── model_metadata.py     # 模型元資料與 token 估算
│   ├── skill_commands.py     # 技能斜線指令處理
│   ├── skill_utils.py        # 技能工具函式
│   ├── curator.py            # 記憶體策展（自動回顧）
│   ├── credential_pool.py    # 憑證池（多 API key 輪換）
│   ├── display.py            # KawaiiSpinner、工具輸出格式化
│   ├── error_classifier.py   # API 錯誤分類與 failover 邏輯
│   ├── retry_utils.py        # 重試策略（jitter backoff）
│   └── transports/           # 自訂 HTTP 傳輸層
│
├── tools/                    # 工具實作（auto-discovered）
│   ├── registry.py           # 工具自動發現與註冊系統
│   ├── terminal_tool.py      # 終端機執行工具
│   ├── file_tools.py         # 檔案操作工具
│   ├── web_tools.py          # 網頁搜尋與瀏覽工具
│   ├── browser_tool.py       # Playwright 瀏覽器控制
│   ├── mcp_tool.py           # MCP 工具整合
│   ├── delegate_tool.py      # 子代理委派工具
│   ├── memory_tool.py        # 記憶體讀寫工具
│   ├── skills_tool.py        # 技能管理工具
│   ├── code_execution_tool.py # 程式碼執行工具
│   ├── approval.py           # 危險操作審核
│   ├── checkpoint_manager.py # 會話檢查點管理
│   └── environments/         # 終端機後端
│       ├── local.py          # 本地終端機
│       ├── docker.py         # Docker 容器終端機
│       ├── ssh.py            # SSH 遠端終端機
│       ├── modal.py          # Modal 無伺服器終端機
│       ├── daytona.py        # Daytona 沙盒終端機
│       └── singularity.py    # Singularity 容器終端機
│
├── hermes_cli/               # CLI 子指令、安裝精靈
│   ├── main.py               # main() 進入點，profile 管理
│   ├── commands.py           # COMMAND_REGISTRY — 所有斜線指令定義
│   ├── skin_engine.py        # CLI 主題引擎
│   ├── plugins.py            # PluginManager — 插件發現與載入
│   ├── env_loader.py         # .env 檔案載入
│   └── curses_ui.py          # curses 互動選單（取代 simple_term_menu）
│
├── gateway/                  # 訊息閘道（多平台）
│   ├── run.py                # GatewayRunner — 主要閘道協調器
│   ├── session.py            # GatewaySession — 單一用戶會話管理
│   ├── platform_registry.py  # 平台適配器註冊
│   ├── delivery.py           # 訊息遞送（含分塊、媒體）
│   ├── hooks.py              # 閘道鉤子系統
│   └── platforms/            # 各平台適配器
│       ├── telegram.py       # Telegram Bot
│       ├── discord.py        # Discord Bot
│       ├── slack.py          # Slack App
│       ├── whatsapp.py       # WhatsApp（baileys 橋接）
│       ├── signal.py         # Signal（signal-cli）
│       ├── matrix.py         # Matrix（mautrix）
│       ├── email.py          # Email SMTP/IMAP
│       ├── api_server.py     # OpenAI 相容 API 伺服器
│       └── ...               # 其他 16+ 平台
│
├── plugins/                  # 插件系統
│   ├── memory/               # 記憶體後端插件（honcho, mem0, supermemory...）
│   ├── context_engine/       # 上下文引擎插件
│   ├── image_gen/            # 圖片生成插件
│   ├── kanban/               # Kanban 看板插件
│   └── example-dashboard/    # 儀表板插件範例
│
├── skills/                   # 內建技能（按分類）
│   ├── github/               # GitHub 技能
│   ├── mlops/                # MLOps 技能
│   ├── productivity/         # 生產力技能
│   └── ...                   # 其他 20+ 分類
│
├── optional-skills/          # 非預設啟用的重型技能
│   └── (15 分類，含 blockchain, security, web-development...)
│
├── tui_gateway/              # TUI 後端（Python JSON-RPC 伺服器）
├── ui-tui/                   # TUI 前端（Ink/React TypeScript）
│   └── src/                  # entry.tsx, app.tsx, gatewayClient.ts
│
├── acp_adapter/              # ACP 伺服器（VS Code/Zed/JetBrains 整合）
├── cron/                     # 排程器（jobs.py, scheduler.py）
├── environments/             # RL 訓練環境（Atropos）
├── tests/                    # pytest 測試套件（~15k 測試）
├── scripts/                  # 工具腳本（run_tests.sh, release.py）
├── website/                  # Docusaurus 文件網站
├── web/                      # Web 儀表板（React + Vite）
├── docs/                     # 文件目錄（含 PDF 規格）
├── plans/                    # 設計計畫文件
├── .plans/                   # 隱藏計畫文件
└── .github/workflows/        # CI/CD 工作流程（10 個 workflow）
```

## 架構模式

- **Monolith 為主**：核心邏輯集中在少數大型 Python 模組
- **Plugin-based 擴展**：記憶體後端、平台適配器、技能均為插件化設計
- **Registry Pattern**：工具自動發現（tools/registry.py）、指令中央登錄（hermes_cli/commands.py）
- **Profile 隔離**：多實例透過 HERMES_HOME 環境變數完全隔離

## 既有文件掃描

### 存在的文件

| 路徑 | 內容 |
|------|------|
| `README.md` | 專案概述、快速安裝、CLI 參考、文件連結 |
| `AGENTS.md` | AI 助理開發指南（36KB，最詳盡）|
| `CONTRIBUTING.md` | 貢獻指南 |
| `SECURITY.md` | 安全政策 |
| `docs/` | 含 PDF 規格與計畫文件 |
| `hermes-already-has-routines.md` | 內部例程說明 |
| `RELEASE_v0.2.0.md` ~ `RELEASE_v0.12.0.md` | 版本發布記錄 |
| `gateway/platforms/ADDING_A_PLATFORM.md` | 平台適配器開發指南 |
| `optional-skills/DESCRIPTION.md` | 可選技能說明 |
| 官方文件網站 | `hermes-agent.nousresearch.com/docs/` |

### 文件與程式碼落差

| 文件描述 | 實際程式碼 | 位置 |
|---------|---------|------|
| 文件說「~15k tests across ~700 files」 | 實際 `tests/` 目錄待驗證（AGENTS.md 的 tree 說明本身已說明「File counts shift constantly」） | `tests/` |
| README 列出 `hermes dashboard` 指令 | 在 `web/` 目錄實作，透過 `fastapi+uvicorn` 提供 | `web/`, `pyproject.toml:[web]` |
| 文件提到 ACP 整合用於 VS Code/Zed/JetBrains | `acp_adapter/` 存在完整實作 | `acp_adapter/` |

## 統計資訊

- 總檔案數：**3,009**（超過 500 個閾值）
- Python 檔案數：**1,390**
- 核心模組（>10k LOC）：`run_agent.py`、`cli.py`
- 次要核心模組（1k-10k LOC）：`model_tools.py`、`toolsets.py`、`hermes_state.py`、`trajectory_compressor.py`、`batch_runner.py`、`mcp_serve.py`

## 關鍵技術特色

1. **自我進化學習迴路**：agent 在複雜任務後自動創建技能，技能在使用中自我改善
2. **多提供商支援**：透過 OpenAI 相容 API 支援 200+ 模型（OpenRouter、NVIDIA NIM、直接廠商等）
3. **多平台閘道**：單一閘道程序支援 20+ 訊息平台
4. **可插拔記憶體後端**：honcho、mem0、supermemory 等多種記憶提供者
5. **六種終端機後端**：local、Docker、SSH、Modal、Daytona、Singularity
6. **Prompt Caching 優先**：特別設計避免破壞 Anthropic prompt cache，降低成本
7. **Profile 多實例**：透過 HERMES_HOME 完全隔離多個 Hermes 實例
8. **RL 研究整合**：與 Atropos 框架整合，支援軌跡生成與 RL 訓練
