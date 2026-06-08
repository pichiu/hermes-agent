# Stage 1 線上搜尋結果

## 搜尋查詢與發現

### 1. 架構與 Skills 系統

**查詢**：`Hermes Agent NousResearch architecture skills system 2025`

**關鍵發現**：
- Hermes Agent 採用三層架構：User Interface → Core Agent Logic → Execution Backends
- Skills 系統遵循 agentskills.io 開放標準，使用 FTS5 全文搜尋索引 skill 文件
- Skills 為 Markdown 格式的程序性記憶文件，按需載入（progressive disclosure）以減少 token 用量
- Agent 完成複雜任務後會自動建立 skill document，下次遇到類似問題時自動查找並應用

**相關連結**：
- [DeepWiki — hermes-agent 架構](https://deepwiki.com/NousResearch/hermes-agent)
- [Skills System 文件](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
- [官方架構文件](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture)
- [i-scoop 介紹文章](https://www.i-scoop.eu/hermes-agent-from-nous-research/)

---

### 2. 學習迴路與 agentskills.io

**查詢**：`hermes-agent NousResearch agentskills.io learning loop`

**關鍵發現**：
- 學習迴路核心：任務完成後建立 Skill Document（可搜尋的 Markdown 檔案）
- FTS5 session search：跨 session 召回歷史對話（LLM 摘要）
- Honcho 整合（plastic-labs/honcho）：dialectic user modeling（辯證式使用者建模）
- Skills 格式與 agentskills.io 開放標準相容，可與其他相容 agent 分享
- 社群資源：
  - [awesome-hermes-agent](https://github.com/0xNyk/awesome-hermes-agent) — 社群 skills、工具、整合清單
  - [hermes-agent-docs](https://github.com/mudrii/hermes-agent-docs) — 社群文件

**相關連結**：
- [Inside Hermes Agent（Substack）](https://mranand.substack.com/p/inside-hermes-agent-how-a-self-improving)
- [DEV Community 深度解析](https://dev.to/truongpx396/hermes-agent-deep-dive-build-your-own-guide-1pcc)
- [hermes-ai.net — 非官方介紹](https://hermes-ai.net/)
- [webvise.io — 自我改善功能介紹](https://webvise.io/blog/hermes-agent-self-improving-ai)

---

### 3. Gateway Platforms 與 ACP

**查詢**：`NousResearch hermes-agent gateway platforms ACP adapter 2025`

**關鍵發現**：
- Gateway 支援 20+ 平台介面卡（Telegram、Discord、Slack、WhatsApp、Signal、Email、Matrix、DingTalk、Feishu、Wecom 等）
- ACP（Agent Client Protocol）支援：讓 Zed、JetBrains、Neovim、Toad 等編輯器直接使用 Hermes
- ACP 協議於 2025 年 10 月獲得 Zed 和 JetBrains 官方合作
- ACP 協議版本 v0.11.0（2026 年 3 月 4 日），GitHub 2300+ stars
- hermes-acp toolset 針對編輯器工作流設計（read_file、write_file、patch、search_files）

**相關連結**：
- [ACP Internals 文件](https://hermes-agent.nousresearch.com/docs/developer-guide/acp-internals)
- [ACP Editor Integration](https://hermes-agent.nousresearch.com/docs/user-guide/features/acp)
- [Adding a Platform Adapter](https://hermes-agent.nousresearch.com/docs/developer-guide/adding-platform-adapters)
- [Programmatic Integration](https://hermes-agent.nousresearch.com/docs/developer-guide/programmatic-integration)
- [AI Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)

---

## 社群與生態系

| 資源 | URL | 說明 |
|------|-----|------|
| Discord | https://discord.gg/NousResearch | 官方社群 |
| Skills Hub | https://agentskills.io | Skills 分享平台 |
| GitHub Issues | https://github.com/NousResearch/hermes-agent/issues | Bug & 功能請求 |
| PyPI | https://pypi.org/project/hermes-agent/ | 套件發布 |

## 相關專案

| 專案 | 說明 |
|------|------|
| [computer-use-linux](https://github.com/avifenesh/computer-use-linux) | Linux 桌面控制 MCP server |
| [HermesClaw](https://github.com/AaronWong1999/hermesclaw) | 社群 WeChat bridge |
| [awesome-hermes-agent](https://github.com/0xNyk/awesome-hermes-agent) | 社群資源清單 |
| [Honcho](https://github.com/plastic-labs/honcho) | User modeling 整合 |

## 重要技術背景

1. **供應鏈攻擊防護**：`pyproject.toml` 所有依賴精確版本鎖定，原因是 2026-05-12 發生的 Mini Shai-Hulud worm 攻擊（`mistralai 2.4.6`）
2. **OpenAI SDK 作為統一介面**：所有 LLM 呼叫通過 OpenAI-compatible API，避免廠商鎖定
3. **Daytona/Modal serverless**：支援 serverless 執行環境，閒置時近乎零成本
4. **Nous Portal**：統一訂閱入口，提供 300+ 模型、網路搜尋、圖像生成、TTS、Cloud browser

## 未找到的資訊（需深入 trace）

- 具體的 context 壓縮演算法細節
- Skill 評分/改善機制的完整流程
- Gateway session 的持久化機制
- Credential pool 的設計細節
