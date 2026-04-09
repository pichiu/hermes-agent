# Stage 1 Reconnaissance — Hermes Agent

## 一句話摘要

Hermes Agent 是由 Nous Research 開發的開源自我改進 AI agent，以「closed learning loop」為核心——從經驗中創建 skill 檔案、在使用過程中改進、跨 session 持久化記憶；支援 Telegram / Discord / Slack / WhatsApp / Signal 等多平台 gateway，可在任意硬體（$5 VPS、GPU cluster、serverless）上執行。

---

## 技術棧

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | ≥3.11 | 主要執行環境 |
| Runtime | Node.js | 18+ | Browser tools、WhatsApp bridge |
| 套件管理 | uv / pip | latest | Python 依賴管理 |
| LLM API | openai (SDK) | ≥2.21,<3 | OpenAI-compatible API 呼叫（all providers） |
| LLM API | anthropic (SDK) | ≥0.39,<1 | Anthropic 直接整合 + prompt caching |
| HTTP | httpx | ≥0.28,<1 | Async HTTP 客戶端 |
| CLI | fire | ≥0.7,<1 | CLI argument parsing |
| CLI TUI | prompt_toolkit | ≥3.0,<4 | 互動式 terminal UI |
| TUI display | rich | ≥14.3,<15 | 終端顯示、表格、進度條 |
| Config | pyyaml | ≥6.0,<7 | YAML config 解析 |
| Config | python-dotenv | ≥1.2,<2 | .env 載入 |
| Validation | pydantic | ≥2.12,<3 | 資料驗證 |
| Template | jinja2 | ≥3.1,<4 | 系統提示模板 |
| Retry | tenacity | ≥9.1,<10 | API 重試邏輯 |
| DB | SQLite (FTS5) | stdlib | session 持久化 + 全文搜尋 |
| Web search | exa-py | ≥2.9,<3 | AI-native web search |
| Web extract | firecrawl-py | ≥4.16,<5 | 網頁內容抓取 |
| Web parallel | parallel-web | ≥0.4.2,<1 | 並行 web 搜尋 |
| Image gen | fal-client | ≥0.13,<1 | fal.ai 圖片生成 |
| TTS | edge-tts | ≥7.2,<8 | 免費 Edge TTS |
| TTS premium | elevenlabs | ≥1.0,<2 | ElevenLabs TTS（optional） |
| Auth | PyJWT[crypto] | ≥2.12,<3 | Skills Hub GitHub App JWT |
| Messaging | python-telegram-bot | ≥22.6,<23 | Telegram adapter |
| Messaging | discord.py | ≥2.7,<3 | Discord adapter |
| Messaging | slack-bolt / slack-sdk | ≥1.18,<2 | Slack adapter |
| Messaging | aiohttp | ≥3.13,<4 | async HTTP（gateway） |
| Matrix | matrix-nio[e2e] | ≥0.24,<1 | Matrix adapter（optional） |
| Memory | honcho-ai | ≥2.0,<3 | 外部 dialectic 記憶（optional） |
| MCP | mcp | ≥1.2,<2 | Model Context Protocol client |
| ACP | agent-client-protocol | ≥0.9,<1 | ACP server（VS Code / Zed / JetBrains） |
| Cron | croniter | ≥6.0,<7 | cron expression 解析 |
| Serverless | modal | ≥1.0,<2 | Modal serverless backend（optional） |
| Cloud IDE | daytona | ≥0.148,<1 | Daytona 開發環境（optional） |
| STT | faster-whisper | ≥1.0,<2 | 本地語音識別（optional） |
| RL | atroposlib + tinker | git | RL 訓練整合（optional） |
| Test | pytest / pytest-asyncio | ≥9.0,<10 | 測試框架 |
| CI | GitHub Actions | - | CI/CD pipeline |
| Container | Docker (debian:13.4) | - | 容器化執行 |
| Nix | flake.nix | - | NixOS 支援 |
| Packaging | Homebrew formula | - | macOS 安裝 |

---

## 目錄結構（3層深度 Annotated）

```
hermes-agent/
├── run_agent.py          # AIAgent 核心類別 — 主對話迴圈 (~7000行)
├── model_tools.py        # 工具 orchestration 層 — _discover_tools, handle_function_call
├── toolsets.py           # Toolset 定義 — _HERMES_CORE_TOOLS, TOOLSETS dict
├── cli.py                # HermesCLI — 互動式 CLI orchestrator (~14000行)
├── hermes_state.py       # SessionDB — SQLite FTS5 session 儲存
├── hermes_constants.py   # 共享常數 — get_hermes_home(), API base URLs
├── hermes_logging.py     # 結構化日誌 — agent.log + errors.log
├── hermes_time.py        # 時區工具
├── batch_runner.py       # 平行 batch 軌跡生成
├── trajectory_compressor.py  # 訓練資料軌跡壓縮
├── mcp_serve.py          # MCP server 模式
├── mini_swe_runner.py    # SWE benchmark runner
├── rl_cli.py             # RL 訓練 CLI
├── toolset_distributions.py  # Toolset 分佈分析
├── utils.py              # 通用工具 — atomic_yaml_write, is_truthy_value
│
├── agent/                # Agent internals
│   ├── prompt_builder.py     # 系統提示組裝 — identity, skills index, context files
│   ├── context_compressor.py # 自動 context 壓縮 (summarize middle turns)
│   ├── prompt_caching.py     # Anthropic prompt caching
│   ├── auxiliary_client.py   # 輔助 LLM client (vision, summarization, cheap tasks)
│   ├── model_metadata.py     # 模型 context 長度、token 估算
│   ├── models_dev.py         # models.dev 即時 context 偵測
│   ├── memory_manager.py     # 記憶 orchestrator — builtin + plugin providers
│   ├── memory_provider.py    # MemoryProvider ABC
│   ├── builtin_memory_provider.py  # 內建 MEMORY.md + USER.md 記憶
│   ├── display.py            # KawaiiSpinner, 工具預覽格式化
│   ├── skill_commands.py     # Skill slash commands (CLI + gateway 共用)
│   ├── skill_utils.py        # Skill 解析工具
│   ├── smart_model_routing.py  # Smart routing (cheap model for simple turns)
│   ├── usage_pricing.py      # 費用估算
│   ├── credential_pool.py    # 多 API key 輪換池
│   ├── anthropic_adapter.py  # Anthropic SDK ↔ OpenAI format 轉換
│   ├── redact.py             # 日誌脫敏
│   ├── retry_utils.py        # Jittered backoff retry
│   ├── trajectory.py         # 軌跡儲存 helper
│   ├── title_generator.py    # Session 標題生成
│   ├── insights.py           # /insights 使用量分析
│   ├── subdirectory_hints.py # 子目錄 context hints
│   ├── context_references.py # Context 引用追蹤
│   └── copilot_acp_client.py # GitHub Copilot ACP client
│
├── hermes_cli/           # CLI 子命令 + 設置
│   ├── main.py           # 入口點 — 所有 hermes 子命令 (argparse)
│   ├── config.py         # DEFAULT_CONFIG, OPTIONAL_ENV_VARS, migration
│   ├── commands.py       # Slash command 定義 + SlashCommandCompleter
│   ├── callbacks.py      # Terminal callbacks (clarify, sudo approval)
│   ├── setup.py          # 互動式設置精靈
│   ├── skin_engine.py    # Skin/theme engine — CLI 視覺客製化
│   ├── skills_config.py  # hermes skills 管理
│   ├── tools_config.py   # hermes tools 管理
│   ├── skills_hub.py     # /skills 命令 (搜尋、瀏覽、安裝)
│   ├── models.py         # 模型目錄、provider 模型清單
│   ├── model_switch.py   # /model 切換 pipeline (CLI + gateway)
│   ├── auth.py           # Provider credential 解析
│   ├── providers.py      # Provider 清單 + base URL 對應
│   ├── env_loader.py     # .env 載入順序
│   ├── doctor.py         # hermes doctor 診斷
│   └── ...               # 其他 CLI 子命令
│
├── tools/                # Tool 實作 (每個 tool 一個 file)
│   ├── registry.py       # 中央工具 registry (singleton)
│   ├── terminal_tool.py  # terminal — 跨後端命令執行
│   ├── file_tools.py     # read_file, write_file, patch, search_files
│   ├── web_tools.py      # web_search, web_extract
│   ├── browser_tool.py   # browser 自動化 (Browserbase/Playwright)
│   ├── delegate_tool.py  # delegate_task — subagent 派遣
│   ├── mcp_tool.py       # MCP client (~1050行)
│   ├── memory_tool.py    # memory — 讀寫 MEMORY.md
│   ├── vision_tools.py   # vision_analyze — 圖片分析
│   ├── image_generation_tool.py  # image_generate (fal.ai)
│   ├── tts_tool.py       # text_to_speech (Edge TTS / ElevenLabs)
│   ├── code_execution_tool.py    # execute_code sandbox
│   ├── session_search_tool.py    # session_search (FTS5)
│   ├── skill_manager_tool.py     # skill CRUD
│   ├── cronjob_tools.py  # cronjob — cron 管理
│   ├── send_message_tool.py      # 跨平台訊息發送
│   ├── todo_tool.py      # todo 清單管理
│   ├── clarify_tool.py   # clarify — 向用戶提問
│   ├── approval.py       # 危險命令偵測 + 批准流程
│   ├── process_registry.py  # 背景程序管理
│   ├── tool_result_storage.py  # 大型工具結果持久化
│   ├── homeassistant_tool.py   # Home Assistant 智慧家庭
│   ├── transcription_tools.py  # 語音轉文字
│   └── environments/     # Terminal backends
│       ├── local.py      # 本地執行
│       ├── docker.py     # Docker 隔離
│       ├── ssh.py        # SSH 遠端
│       ├── modal.py      # Modal serverless
│       ├── daytona.py    # Daytona 雲端 IDE
│       └── singularity.py # Singularity HPC
│
├── gateway/              # 訊息平台 gateway
│   ├── run.py            # GatewayRunner — 主迴圈、訊息分派
│   ├── session.py        # SessionStore — 對話持久化
│   ├── config.py         # Platform enum, PlatformConfig, HomeChannel
│   ├── hooks.py          # Gateway hooks (builtin_hooks/)
│   ├── delivery.py       # 訊息投遞
│   ├── pairing.py        # DM pairing (安全授權)
│   ├── stream_consumer.py  # 串流回應消費
│   ├── mirror.py         # 跨平台訊息鏡像
│   ├── channel_directory.py  # channel 目錄管理
│   ├── status.py         # gateway 狀態
│   └── platforms/        # Platform adapters
│       ├── base.py       # BasePlatformAdapter ABC
│       ├── telegram.py   # Telegram
│       ├── discord.py    # Discord
│       ├── slack.py      # Slack
│       ├── whatsapp.py   # WhatsApp (Node.js bridge)
│       ├── signal.py     # Signal
│       ├── email.py      # Email
│       ├── matrix.py     # Matrix
│       ├── mattermost.py # Mattermost
│       ├── homeassistant.py  # Home Assistant
│       ├── dingtalk.py   # DingTalk
│       ├── feishu.py     # Feishu/Lark
│       ├── wecom.py      # WeCom
│       ├── sms.py        # SMS
│       ├── webhook.py    # Generic webhook
│       └── api_server.py # REST API server
│
├── cron/                 # 排程器
│   ├── jobs.py           # Job 儲存 (~/.hermes/cron/jobs.json)
│   └── __init__.py       # Scheduler entry
│
├── acp_adapter/          # ACP server (editor 整合)
│   ├── entry.py          # hermes-acp CLI 入口
│   ├── server.py         # ACP server 實作
│   ├── tools.py          # ACP tool 橋接
│   └── session.py        # ACP session 管理
│
├── plugins/              # Plugin 系統
│   └── honcho_plugin/    # Honcho AI 記憶 plugin（參考實作）
│
├── skills/               # 內建 skills（隨安裝打包）
│   ├── research/         # 學術研究 skill
│   ├── data-science/     # 資料科學 skill
│   ├── github/           # GitHub workflow skill
│   ├── dogfood/          # Web QA 測試 skill
│   ├── ...               # 其他 20+ 個 skills
│   └── index-cache/      # skills hub 索引快取
│
├── optional-skills/      # Optional skills（非預設啟用）
│   ├── security/         # 安全相關 skills
│   ├── mlops/            # ML Ops skills
│   └── ...               # 其他 optional skills
│
├── environments/         # RL 訓練環境 (Atropos)
├── tinker-atropos/       # RL 訓練 git submodule
├── tests/                # Pytest 測試套件 (~3000 tests)
│   ├── agent/            # AIAgent 單元測試
│   ├── cli/              # CLI 測試
│   ├── gateway/          # Gateway 測試
│   ├── tools/            # Tool 測試
│   ├── e2e/              # End-to-end 測試
│   └── integration/      # 需外部服務的整合測試
│
├── docs/                 # 專案文件（少量，主要在外部網站）
│   ├── acp-setup.md      # ACP 設置指南
│   ├── honcho-integration-spec.md  # Honcho 整合規格
│   └── migration/        # 遷移指南
│
├── website/              # 文件網站（mintlify）
├── landingpage/          # 落地頁靜態 HTML
├── docker/               # Docker 相關設定
├── packaging/            # 打包設定（Homebrew）
├── scripts/              # 安裝腳本、WhatsApp bridge
│   └── whatsapp-bridge/  # Node.js WhatsApp bridge
├── plans/                # 設計規劃文件
│   └── gemini-oauth-provider.md
├── assets/               # 靜態資源 (banner.png)
├── datagen-config-examples/  # 資料生成設定範例
└── tinker-atropos/       # RL submodule（tinker + atropos）
```

---

## 架構模式

**Monolith with plugin extensions** — 核心是單一 Python 應用，透過以下機制進行功能擴充：
- `tools/registry.py` singleton：工具自我注冊 pattern
- `plugins/` 目錄：第三方 memory provider、CLI 子命令
- `skills/` 目錄：Markdown-based procedural skill 文件
- `gateway/platforms/` 目錄：每平台獨立 adapter

---

## 現有文件與程式碼落差分析

| 描述 | 文件說法 | 實際狀況 | 位置 |
|------|----------|----------|------|
| 測試數量 | AGENTS.md 說「~3000 tests」 | tests/ 目錄下有多個子目錄，實際數量未驗證 | `tests/` |
| MCP tool 行數 | AGENTS.md 說「~1050 lines」 | 檔案存在但未實際計算行數 | `tools/mcp_tool.py` |
| cli.py 大小 | AGENTS.md 未提及 | 實測約 390KB，14000+ 行 | `cli.py` |
| Optional skills | CONTRIBUTING.md 描述放在 optional-skills/ | 確認存在於 `optional-skills/` | `optional-skills/` |
| WhatsApp bridge | README 提及 Node.js bridge | 存在於 `scripts/whatsapp-bridge/` | `scripts/whatsapp-bridge/` |
| tinker-atropos | README 說是 git submodule | `.gitmodules` 確認存在 | `tinker-atropos/`, `.gitmodules` |

---

## CI/CD 設定

- **GitHub Actions workflows**：
  - `tests.yml`：Unit tests（排除 integration + e2e）+ e2e tests，使用 `pytest -n auto` 平行執行
  - `docker-publish.yml`：Docker image 發佈
  - `nix.yml`：NixOS flake 驗證
  - `deploy-site.yml`：文件網站部署
  - `supply-chain-audit.yml`：供應鏈安全稽核
  - `docs-site-checks.yml`：文件網站檢查

---

## 設定載入機制

1. `HERMES_HOME` env var → 預設 `~/.hermes`
2. `~/.hermes/.env`（最高優先）→ 用戶 API keys
3. 專案根目錄 `.env`（開發環境 fallback）
4. `~/.hermes/config.yaml`（主設定）→ 模型、provider、toolset 等
5. System env vars（最低優先）

---

## 版本資訊

- 當前版本：**v0.8.0 (v2026.4.8)**，發佈於 2026-04-08
- 專案啟動：2026 年初（約 3,496 commits，3 個月內）
- Python 要求：≥3.11
- MIT License，由 Nous Research 開發
