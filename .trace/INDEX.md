# Hermes Agent — 專案總覽與速查

## 一段話總結

**Hermes Agent** 是由 Nous Research 開發的開源自我改進 AI agent（v0.10.0，2026-04-16）。它以「closed learning loop」為核心——從完成的任務中創建可重用的 skill 文件、在使用過程中改進技能、定期整理長期記憶——實現真正跨 session 的持續學習。可透過 Telegram、Discord、Slack、WhatsApp、Signal 等 16+ 平台存取，也可在 CLI 互動；後端執行環境從 $5 VPS 到 GPU cluster 到 serverless 均支援，空閒時近乎零成本。付費 Nous Portal 用戶可透過 **Tool Gateway** 使用 Web 搜尋、圖片生成、TTS、瀏覽器自動化，無需額外 API 金鑰。

---

## 技術棧總覽

| 類別 | 技術 | 版本 | 用途 |
|------|------|------|------|
| Runtime | Python | ≥3.11 | 主要語言 |
| Runtime | Node.js | 18+ | Browser tools、WhatsApp bridge |
| LLM API | openai SDK | ≥2.21,<3 | 統一 LLM 呼叫層（provider agnostic） |
| LLM API | anthropic SDK | ≥0.39,<1 | Anthropic 原生 API + prompt caching |
| HTTP | httpx | ≥0.28,<1 | Async HTTP client |
| CLI TUI | prompt_toolkit | ≥3.0,<4 | 互動式 terminal UI |
| Display | rich | ≥14.3,<15 | 終端格式化輸出 |
| Config | pyyaml + python-dotenv | ≥6.0 / ≥1.2 | YAML + .env 設定載入 |
| Validation | pydantic | ≥2.12,<3 | 資料驗證 |
| DB | SQLite (FTS5) | stdlib | Session 持久化 + 全文搜尋 |
| Messaging | python-telegram-bot | ≥22.6,<23 | Telegram |
| Messaging | discord.py | ≥2.7,<3 | Discord |
| Messaging | slack-bolt / slack-sdk | ≥1.18,<2 | Slack |
| Messaging | aiohttp | ≥3.13,<4 | Async HTTP（gateway） |
| Memory | honcho-ai | ≥2.0,<3 | 外部 dialectic 記憶（optional） |
| MCP | mcp | ≥1.2,<2 | Model Context Protocol |
| ACP | agent-client-protocol | ≥0.9,<1 | Editor 整合（VS Code/Zed/JetBrains） |
| Web Search | exa-py | ≥2.9,<3 | AI-native 語意搜尋 |
| Web Extract | firecrawl-py | ≥4.16,<5 | 網頁內容抓取 |
| Image Gen | fal-client | ≥0.13,<1 | fal.ai 圖片生成 |
| TTS | edge-tts | ≥7.2,<8 | 免費 Edge TTS |
| Cron | croniter | ≥6.0,<7 | Cron expression 解析 |
| Serverless | modal | ≥1.0,<2 | Modal 無伺服器後端 |
| Container | Docker (debian:13.4) | - | 容器化執行 |
| Packaging | uv / Homebrew / Nix | - | 多種安裝管道 |
| CI | GitHub Actions | - | Tests + Docker + supply chain audit |
| Test | pytest + pytest-asyncio | ≥9.0,<10 | 測試框架（~3000 tests） |

---

## 關鍵指令速查

```bash
# 安裝
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 開發環境設置
git clone --recurse-submodules https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv venv --python 3.11 && source venv/bin/activate
uv pip install -e ".[all,dev]"

# 執行
hermes                     # 互動式 CLI
hermes setup               # 初次設置精靈
hermes model               # 切換 LLM 模型
hermes tools               # 管理工具啟用/停用
hermes gateway start       # 啟動訊息平台 gateway

# 測試
python -m pytest tests/ -q --ignore=tests/integration --ignore=tests/e2e -n auto
python -m pytest tests/e2e/ -v

# 設定
hermes config set model.default "anthropic/claude-opus-4.6"
hermes doctor              # 診斷環境問題
hermes update              # 更新到最新版

# 進階
hermes sessions browse     # 瀏覽歷史 session
hermes cron list           # 查看排程任務
hermes skills browse       # 瀏覽 skills hub
hermes honcho setup        # 設置 Honcho 記憶整合

# v0.9.0+ 新增
hermes backup              # 備份設定、sessions、skills、memory
hermes import              # 從備份還原
hermes dump                # 輸出可分享的設定摘要
hermes debug share         # 上傳 debug 報告到 pastebin
hermes skills reset        # 重設 bundled skills（解卡用）
/fast                      # 切換 Fast Mode（OpenAI/Anthropic 優先佇列）
/compress <topic>          # 帶 focus topic 的 context 壓縮
/debug                     # 快速診斷（所有平台可用）

# v0.10.0+ 新增
hermes memory reset        # 重設 memory
hermes -Q                  # Quiet mode：只輸出純文字回應
```

---

## 文件地圖

| 文件 | 內容 |
|------|------|
| [INDEX.md](INDEX.md) | ← 你在這裡：專案總覽、技術棧、指令速查 |
| [CHANGELOG.md](CHANGELOG.md) | 版本變更記錄（v0.8.0 → v0.9.0 → v0.10.0） |
| [ARCHITECTURE.md](ARCHITECTURE.md) | 系統架構圖、元件清單、通訊模式、設計決策 |
| [CODEBASE_MAP.md](CODEBASE_MAP.md) | 程式碼地圖、目錄說明、「我想改 X 看哪裡」速查 |
| [DATA_MODEL.md](DATA_MODEL.md) | SQLite schema、Session/Message 資料結構、記憶模型 |
| [API_SURFACE.md](API_SURFACE.md) | CLI 命令、Slash commands、Tool schemas、HTTP API |
| [DEV_GUIDE.md](DEV_GUIDE.md) | 開發環境設置、本地開發流程、測試策略、踩坑指南 |
| [DISCOVERY_LOG.md](DISCOVERY_LOG.md) | Web 發現、文件落差、TODO/FIXME、技術債 |

---

## 專案專屬術語表

| 術語 | 定義 |
|------|------|
| **Skill** | 可重用的 Markdown 知識文件，agent 從經驗中創建，透過 `/skill-name` 或 nudge 觸發 |
| **Closed Learning Loop** | 任務完成 → 創建 skill → 記憶整理 → 下次改進，形成閉環 |
| **Nudge** | 系統在適當時機提醒 agent 更新記憶或創建 skill 的機制 |
| **Gateway** | 單一 process 連接所有訊息平台（Telegram/Discord/...）的橋接層 |
| **Toolset** | 一組工具的邏輯分組（如 `web`、`terminal`、`file`），用於批量啟用/停用 |
| **Session** | 一段連續的對話，儲存在 SQLite，支援跨 session 搜尋 |
| **Context Compression** | 對話過長時自動摘要中段內容，以便在 context window 內繼續 |
| **IterationBudget** | Parent + 所有 subagent 共享的工具呼叫次數上限（預設 90） |
| **HERMES_HOME** | Hermes 主目錄（預設 `~/.hermes`），可透過 env var 或 profile 切換 |
| **Profile** | 使用不同 HERMES_HOME 的設定集，如 `~/.hermes/profiles/coder` |
| **SOUL.md** | Agent 人格定義檔，覆蓋預設 agent identity |
| **Terminal Backend** | 命令執行環境：local / docker / ssh / modal / daytona / singularity |
| **Pairing** | Gateway 平台授權機制：未知用戶發送訊息 → 生成 pairing code |
| **Subagent** | 由主 agent 透過 `delegate_task` 派遣的獨立 AIAgent 實例 |
| **Atropos** | Nous Research 的 RL 訓練框架，Hermes 透過 tinker-atropos submodule 整合 |
| **ACP** | Agent Client Protocol，允許 editor 向 Hermes 發送工具呼叫請求 |
| **MCP** | Model Context Protocol，允許連接外部工具 server |
| **Honcho** | 外部 dialectic AI 記憶後端，提供比 MEMORY.md 更深度的用戶建模 |
| **agentskills.io** | Skill 格式的開放標準，Hermes 相容此格式 |
| **Auxiliary LLM** | 用於 side tasks（vision、compression、web_extract）的次要 LLM，通常比主模型便宜 |
| **Prompt Caching** | Anthropic prefix cache 機制，系統提示快取以降低 75% 輸入 token 費用 |
| **Tool Gateway** | Nous Portal 訂閱附帶的 managed tool 服務（web search、image gen、TTS、browser），無需個人 API key |
| **Fast Mode** | `/fast` 切換的優先佇列模式，對 OpenAI 和 Anthropic 模型顯著降低延遲 |
| **watch_patterns** | 背景 process 輸出監控的 pattern 設定，有匹配時即時通知（v0.9.0+）|
| **Pluggable Context Engine** | 透過 plugin 替換 context 管理邏輯（filtering、summarization、injection）的插槽 |
| **Blueprint（iMessage/WeChat/WeCom）** | v0.9.0 新增的 3 個平台：BlueBubbles iMessage、Weixin WeChat、WeCom Callback Mode |
