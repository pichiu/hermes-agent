# Stage 1 偵察報告 (Reconnaissance)

## 專案概覽

**Hermes Agent** 是由 [Nous Research](https://nousresearch.com) 開發的開源、自我改善的 AI agent 框架。

- **定位**：The self-improving AI agent — 唯一內建學習迴路的 agent，從經驗建立 skill，使用中自動改善，並跨 session 建立使用者模型
- **版本**：0.16.0（`pyproject.toml`）
- **授權**：MIT
- **語言**：Python 3.11–3.13（主體），TypeScript/Node.js（TUI、Web dashboard）
- **官方文件**：https://hermes-agent.nousresearch.com/docs/

## 技術棧

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | 3.11–3.13 | 主要 agent 核心 |
| Runtime | Node.js | 20+ | TUI (ui-tui)、Web frontend (web/) |
| Package manager | uv | latest | Python 依賴管理 |
| HTTP 框架 | FastAPI | >=0.104 | API server、ACP server、MCP server |
| ASGI | Uvicorn | >=0.24 | FastAPI 運行時 |
| LLM SDK | openai | 2.24.0 | 所有 LLM 呼叫（OpenAI-compatible endpoint） |
| LLM SDK (可選) | anthropic | 0.86.0 | 直接 Anthropic API |
| 資料驗證 | pydantic | 2.13.4 | 資料模型與驗證 |
| CLI 框架 | python-fire | 0.7.1 | CLI 子命令 |
| Terminal UI | prompt_toolkit | 3.0.52 | 互動式 CLI |
| HTTP client | httpx | 0.28.1 | 外部 API 呼叫 |
| Retry | tenacity | 9.1.4 | API retry 邏輯 |
| Cron | croniter | 6.0.0 | 排程任務 |
| Config | python-dotenv + ruamel.yaml | | 環境變數與 YAML config |
| 模板 | jinja2 | 3.1.6 | 系統提示詞渲染 |
| 圖像 | Pillow | 12.2.0 | 圖像縮放（vision tools） |
| Auth | PyJWT | 2.12.1 | Skills Hub JWT auth |
| 前端打包 | Vite + React (⚠️ 推測) | | Web dashboard (web/) |
| 容器化 | Docker | | Dockerfile + docker-compose |
| CI/CD | GitHub Actions | | .github/workflows/ |
| Nix | flake.nix | | 開發環境 reproducibility |
| Process manager | psutil | 7.2.2 | 跨平台 process 管理 |

## 目錄結構（3 層深度）

```
hermes-agent/                      # 根目錄
├── agent/                         # 核心 agent 邏輯
│   ├── conversation_loop.py       # 主要對話迴路（~3900 行），模型呼叫、工具調度
│   ├── system_prompt.py           # 系統提示詞建構
│   ├── tool_executor.py           # 工具執行引擎
│   ├── memory_manager.py          # 記憶管理
│   ├── skill_*.py                 # Skill 系統（skill_commands, skill_utils 等）
│   ├── chat_completion_helpers.py # LLM API 呼叫輔助
│   ├── context_compressor.py      # Context 壓縮
│   ├── prompt_caching.py          # Anthropic prompt caching
│   ├── transports/                # 傳輸層
│   └── *_adapter.py              # 各 LLM provider adapter
├── tools/                         # 40+ 工具實作
│   ├── terminal_tool.py           # Shell 執行
│   ├── file_tools.py              # 檔案操作
│   ├── web_tools.py               # Web 搜尋/瀏覽
│   ├── memory_tool.py             # 記憶讀寫
│   ├── skills_tool.py             # Skill 管理
│   ├── delegate_tool.py           # Subagent 委派
│   ├── mcp_tool.py                # MCP 整合
│   ├── browser_tool.py            # Browser automation
│   ├── image_generation_tool.py   # 圖像生成
│   ├── tts_tool.py                # 語音合成
│   ├── registry.py                # 工具註冊表
│   └── lazy_deps.py               # 懶載入依賴
├── gateway/                       # Messaging gateway
│   ├── run.py                     # Gateway 入口
│   ├── platforms/                 # 20+ 平台介面卡
│   │   ├── telegram.py            # Telegram bot
│   │   ├── discord.py (⚠️ 未找到) | # Discord
│   │   ├── slack.py               # Slack
│   │   ├── whatsapp.py            # WhatsApp
│   │   ├── signal.py              # Signal
│   │   ├── email.py               # Email
│   │   ├── matrix.py              # Matrix
│   │   ├── api_server.py          # HTTP webhook server
│   │   └── ...                    # DingTalk, Feishu, Wecom 等
│   ├── session.py                 # Gateway session 管理
│   └── hooks.py                   # Gateway hooks
├── skills/                        # 預建 skills（隨 Hermes 出貨）
│   ├── apple/                     # Apple 相關（macOS 自動化）
│   ├── autonomous-ai-agents/      # AI agent 工作流
│   ├── devops/                    # DevOps 工作流
│   ├── software-development/      # 軟體開發
│   ├── research/                  # 研究工作流
│   └── ...（19 個類別）
├── optional-skills/               # 官方但非預設的 skills
│   ├── email/                     # Email 整合
│   ├── github/                    # GitHub 工作流
│   ├── note-taking/               # 筆記
│   └── ...（20 個類別）
├── plugins/                       # 可選插件系統
│   ├── memory/                    # 記憶 provider（Honcho, mem0 等）
│   ├── observability/             # 可觀測性（Langfuse, NeMo Relay）
│   ├── browser/                   # 瀏覽器整合
│   ├── platforms/                 # 額外平台
│   └── ...（16 個插件）
├── acp_adapter/                   # Agent Client Protocol（編輯器整合）
├── acp_registry/                  # ACP 工具註冊
├── ui-tui/                        # TUI 前端（Node.js）
├── web/                           # Web dashboard 前端（React + Vite）
├── website/                       # 文件站台
├── hermes_cli/                    # CLI 子命令（hermes model, hermes setup 等）
├── tui_gateway/                   # TUI gateway bridge
├── apps/                          # 其他 app entry points
├── tests/                         # 測試（26 個子目錄）
├── run_agent.py                   # AIAgent 主類別（~234K bytes）
├── cli.py                         # CLI 主程式（~737K bytes）
├── hermes_state.py                # 全局狀態（~187K bytes）
├── hermes_constants.py            # 常數定義
├── hermes_logging.py              # Logging 基礎建設
├── toolsets.py                    # Toolset 定義（工具組合）
├── model_tools.py                 # 模型工具整合
├── mcp_serve.py                   # MCP server 模式
├── batch_runner.py                # 批次執行（研究用）
├── trajectory_compressor.py       # Trajectory 壓縮
├── Dockerfile                     # Docker 容器化
├── docker-compose.yml             # Docker Compose
└── pyproject.toml                 # 套件定義與依賴

```

## 架構模式

**Monolith with Plugin Architecture**：核心是一個整合的 Python monolith（`run_agent.py` + `agent/`），通過明確定義的 plugin 系統對外擴展。

主要設計特徵：
1. **OpenAI-compatible API 抽象**：所有 LLM provider 均通過 OpenAI SDK 呼叫（adapter pattern），不鎖定單一供應商
2. **Provider Adapter Pattern**：`agent/*_adapter.py` 提供各 provider 的轉接層（Anthropic, Gemini, Bedrock, Codex 等）
3. **工具即函式**：工具系統以 Python 函式為單元，通過 `tools/registry.py` 注冊，JSON Schema 描述給模型
4. **Skill 系統**：Markdown 文件即 skill，agentskills.io 相容，支援 FTS5 全文搜尋
5. **Gateway Pattern**：單一 gateway process 統一路由多個 messaging 平台的訊息
6. **Observer Hook 系統**：讀寫分離，observer hook 只讀，middleware 可改寫請求/執行

## 既有文件掃描

| 文件位置 | 內容 | 品質 |
|----------|------|------|
| `README.md` | 功能介紹、安裝、入門指引 | 完整 |
| `README.zh-CN.md` | 簡體中文版 README | 完整 |
| `CONTRIBUTING.md` | 貢獻指南、架構說明、開發環境 | 詳細 |
| `AGENTS.md` | AI agent 工作指引（給 AI 助手的說明） | 詳細 |
| `SECURITY.md` | 安全政策 | 存在 |
| `docs/middleware/README.md` | Middleware 系統詳細文件 | 完整 |
| `docs/observability/README.md` | Observer hook 系統文件 | 完整 |
| `docs/security/network-egress-isolation.md` | 網路隔離文件 | 存在 |
| `docs/kanban/multi-gateway.md` | Multi-gateway 文件 | 存在 |
| `hermes-already-has-routines.md` | 已有 routine 說明 | 存在 |

### 落差分析（文件 vs. 程式碼）

目前未發現重大落差，既有文件與程式碼結構基本一致。以下為待驗證項目：
- `docs/observability/README.md` 提到 `plugins/observability/nemo_relay/README.md`，需確認該文件是否存在
- `CONTRIBUTING.md` 說明記憶 provider 插件應為獨立 repo，但 `plugins/memory/` 仍有多個內建 provider

## CI/CD

GitHub Actions workflows（`.github/workflows/`）：
- `tests.yml` — 測試
- `lint.yml` — 代碼 lint
- `docker-publish.yml` — Docker 映像發布
- `deploy-site.yml` — 文件站台部署
- `supply-chain-audit.yml` — 供應鏈安全審計
- `osv-scanner.yml` — OSV 漏洞掃描
- `upload_to_pypi.yml` — PyPI 發布
- `skills-index.yml` — Skills Hub 索引更新
- `build-windows-installer.yml` — Windows 安裝程式

## 安全措施

- 所有依賴精確版本鎖定（exact pin，非 range）— 應對 supply chain 攻擊
- OSV 漏洞掃描（CI）
- 命令審批系統（tool_guardrails.py）
- Tirith 安全框架整合（tirith_security.py）
- DM pairing 認證（messaging gateway）
- 路徑安全（path_security.py）
