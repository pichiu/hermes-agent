# Stage 1 Web Findings — Hermes Agent

## 線上資源搜尋結果

### 官方資源

| 資源 | URL | 摘要 |
|------|-----|------|
| 官方文件 | https://hermes-agent.nousresearch.com/docs/ | 完整文件，使用 Mintlify 建置 |
| 官方首頁 | https://hermes-agent.nousresearch.com/ | 產品介紹頁 |
| Nous Research 首頁 | https://nousresearch.com/hermes-agent/ | 研究機構官方介紹 |
| GitHub Repo | https://github.com/NousResearch/hermes-agent | 主要開源倉庫，MIT license |
| Skills Hub | https://hermes-agent.nousresearch.com/docs/skills/ | Skills hub 社群索引 |
| Architecture 文件 | https://hermes-agent.nousresearch.com/docs/developer-guide/architecture/ | 官方架構說明（與 AGENTS.md 一致） |
| CLI Reference | https://hermes-agent.nousresearch.com/docs/reference/cli-commands/ | 完整 CLI 命令參考 |
| RL Training 文件 | https://hermes-agent.nousresearch.com/docs/user-guide/features/rl-training | RL 訓練整合指南 |
| Memory Providers | https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers | 記憶 provider 系統說明 |
| Honcho 文件 | https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho/ | Honcho 整合詳細說明 |
| Messaging Gateway | https://hermes-agent.nousresearch.com/docs/user-guide/messaging/ | Gateway 設置指南 |
| Skills System | https://hermes-agent.nousresearch.com/docs/user-guide/features/skills | Skills 系統架構 |

### 發佈資訊

| 版本 | 發佈日期 | 重要特性 |
|------|----------|---------|
| v0.8.0 (v2026.4.8) | 2026-04-08 | Background task auto-notifications, Google AI Studio native, MCP OAuth 2.1, plugin system expansion, Matrix tier-1 |
| v0.7.0 (v2026.4.3) | 2026-04-03 | SQLite+FTS5 session search, learning loop, closed skill creation loop |
| v0.6.0 | 2026-03 末 | MCP support, typed SDK workflows |
| v0.5.0 | 2026 早期 | - |

### 社群資源

| 資源 | URL | 摘要 |
|------|-----|------|
| Awesome Hermes Agent | https://github.com/0xNyk/awesome-hermes-agent | 社群整理的 skills、tools、integrations 清單 |
| Discord | https://discord.gg/NousResearch | 官方 Discord 社群 |
| agentskills.io | https://agentskills.io | Skill 標準格式開放社群，Hermes 相容 |
| DEV Community 文章 | https://dev.to/arshtechpro/hermes-agent-a-self-improving-ai-agent-that-runs-anywhere-2b7d | 外部介紹文章 |
| Bitcoin News 文章 | https://news.bitcoin.com/what-is-hermes-agent-nous-researchs-self-improving-ai-explained/ | 媒體報導 |

---

## 關鍵技術發現摘要

### 1. Skills 系統架構（三層載入）
來源：https://hermes-agent.nousresearch.com/docs/user-guide/features/skills

Skills 採用「漸進式揭露」(progressive disclosure) 模式以最小化 token 使用：
- **Level 0**：`skills_list()` — 返回基本 metadata（~3k tokens）
- **Level 1**：`skill_view(name)` — 載入完整 skill 內容
- **Level 2**：`skill_view(name, path)` — 讀取特定參考檔案

Skill 格式基於 Markdown + YAML frontmatter（agentskills.io 開放標準）。

### 2. 記憶系統（三種模式）
來源：https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers

- **local（內建）**：`MEMORY.md` + `USER.md` 純文字檔案
- **honcho**：Honcho AI 外部 dialectic 記憶 + 用戶建模
- **hybrid**：同時啟用兩者

Memory provider 採用 plugin ABC 設計，允許自訂後端（vector store、custom DB）。

### 3. ACP 整合（Editor 工具呼叫）
來源：https://hermes-agent.nousresearch.com/docs/integrations/

Agent Client Protocol (ACP) 讓 VS Code / Zed / JetBrains 直接向 Hermes 發送工具呼叫請求，也允許 editor 的 MCP 生態系直接流入 agent。

### 4. RL 訓練 Pipeline
來源：https://hermes-agent.nousresearch.com/docs/user-guide/features/rl-training + https://hermes-agent.nousresearch.com/docs/developer-guide/environments/

三元件架構：
- **Atropos**：軌跡 API server，協調環境互動，計算 advantages
- **Tinker**：訓練服務，處理 model weights、LoRA training、GRPO
- **Environments**：Python 類別定義任務、評分、reward functions

所有 RL 操作透過 agent 的 tool interface 操作（`rl_training_tool.py`）。

### 5. Messaging Gateway 設計
來源：https://hermes-agent.nousresearch.com/docs/user-guide/messaging/

單一 gateway process 連接所有平台，支援：
- Telegram, Discord, Slack, WhatsApp, Signal, Email, Matrix, Mattermost, Home Assistant, DingTalk, Feishu/Lark, WeCom, SMS
- 語音訊息自動轉錄（faster-whisper）
- 跨平台 conversation continuity
- DM pairing 安全授權機制

### 6. Terminal Backends（六種執行環境）
來源：AGENTS.md + README

| Backend | 用途 |
|---------|------|
| local | 直接在本地執行 |
| docker | Docker 容器隔離 |
| ssh | SSH 遠端執行 |
| modal | Modal serverless（休眠時幾乎零成本） |
| daytona | Daytona 雲端 IDE |
| singularity | Singularity HPC 環境 |

### 7. 安全機制（v0.8.0 新增強化）
- 危險命令偵測（`tools/approval.py`）+ 平台原生審批按鈕（Slack、Telegram）
- Prompt injection 偵測（context files 掃描）
- MCP OAuth 2.1 PKCE + OSV 惡意軟體掃描
- SSRF 防護、timing attack 緩解、tar traversal 防護
- 跨 session 隔離

---

## GitHub Issues/Discussions 發現

| Issue/PR | 重點 |
|----------|------|
| #5143 | 「Multi-Role Auto-Routing via Gateway Hooks」— gateway hook 支援多角色路由 |
| #5779 | Background process auto-notifications — 背景任務完成自動通知 |
| #5181 | Live /model switching — 跨平台即時模型切換 |
| #5420 | MCP OAuth 2.1 PKCE 實作 |
| #5305 | OSV 惡意軟體掃描整合 |

---

## 與競品的差異化定位

根據媒體文章與官方文件分析：

1. **Closed learning loop**：與一般 agent 不同，Hermes 在完成任務後主動創建 skill 並改進
2. **Model agnostic**：透過 OpenAI-compatible API 支援 200+ 模型，無 vendor lock-in
3. **Platform agnostic**：同一 agent 同時服務 CLI + 14+ 訊息平台
4. **Research-ready**：內建 RL 訓練 pipeline，直接產出下一代工具呼叫模型的訓練資料
5. **Truly serverless**：Modal / Daytona 後端讓 agent 在空閒時幾乎零成本

---

## 未找到的資訊

- ⚠️ 未驗證：agentskills.io 的 Hermes skill 數量（社群 hub 資料）
- ⚠️ 未驗證：GitHub Discussions 是否有 pinned architecture 討論
- ⚠️ 未驗證：效能基準測試數據
