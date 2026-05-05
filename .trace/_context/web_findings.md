# Stage 1 線上資源搜尋結果

## 官方資源

| 資源 | URL | 關鍵內容 |
|------|-----|---------|
| GitHub 主庫 | https://github.com/NousResearch/hermes-agent | 原始碼、Issues、PR |
| 官方文件 | https://hermes-agent.nousresearch.com/docs/ | 完整使用文件 |
| 官方網站 | https://hermes-agent.nousresearch.com/ | 專案介紹 |
| 架構文件 | https://hermes-agent.nousresearch.com/docs/developer-guide/architecture | 系統架構詳述 |
| 技能文件 | https://hermes-agent.nousresearch.com/docs/user-guide/features/skills | 技能系統說明 |
| MCP 整合 | https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp | MCP 配置說明 |
| RL 訓練 | https://hermes-agent.nousresearch.com/docs/user-guide/features/rl-training | RL 環境說明 |
| 工具參考 | https://hermes-agent.nousresearch.com/docs/reference/tools-reference | 61 個工具清單 |
| 記憶體提供者 | https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers | 7 種記憶提供者 |
| Honcho 整合 | https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho | Honcho 辯證推理 |
| Toolsets 參考 | https://hermes-agent.nousresearch.com/docs/reference/toolsets-reference | 52 個 toolset |
| 提示組裝 | https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly | 系統提示建構 |

## 社群與生態系

| 資源 | URL | 說明 |
|------|-----|------|
| Discord | https://discord.gg/NousResearch | Nous Research 官方社群 |
| agentskills.io | https://agentskills.io | 開放技能標準市集 |
| awesome-hermes-agent | https://github.com/0xNyk/awesome-hermes-agent | 社群整理資源清單 |
| hermes-agent-docs (社群) | https://github.com/mudrii/hermes-agent-docs | 社群翻譯/補充文件 |
| hermes-agent-self-evolution | https://github.com/NousResearch/hermes-agent-self-evolution | 技能/提示自動進化（DSPy+GEPA）|
| DeepWiki 分析 | https://deepwiki.com/NousResearch/hermes-agent | 自動生成的程式碼分析 |
| Honcho 整合指南 | https://docs.honcho.dev/v3/guides/integrations/hermes | Honcho 官方整合說明 |

## 關鍵技術發現

### 1. 專案規模與採用情況

**發布日期**: 2026 年 2 月 25 日  
**GitHub Stars**: 64,200+（截至 2026 年 5 月）  
**技能數量**: 118 個（96 捆綁 + 22 可選），跨 26+ 分類  
**支援平台**: 18 個內建訊息平台

來源：[Hermes Agent Review](https://kisztof.medium.com/hermes-agent-review-nous-researchs-self-improving-ai-agent-e72bc244435a)、[NxCode Complete Guide](https://www.nxcode.io/resources/news/hermes-agent-complete-guide-self-improving-ai-2026)

### 2. 架構核心組件

官方文件描述的核心架構：

```
Entry Points
├── CLI (cli.py)
├── Gateway (gateway/run.py)
├── ACP (acp_adapter/)
├── Batch Runner (batch_runner.py)
├── API Server (gateway/platforms/api_server.py)
└── Python Library (AIAgent class)
        ↓
AIAgent Core
├── Prompt Builder    — 組裝 system prompt（人格 + 記憶 + 技能 + 上下文）
├── Provider Resolution — 18+ 提供商映射，含 OAuth 流程
├── Tool Registry     — 61 工具 across 52 toolsets（auto-discovered）
└── Terminal Backends — 7 種後端（local, Docker, SSH, Daytona, Modal, Singularity, Vercel）
```

來源：[Architecture Docs](https://hermes-agent.nousresearch.com/docs/developer-guide/architecture)

### 3. 自我進化學習迴路

- 當 agent 完成複雜任務（5+ 工具呼叫）時，**自動生成技能文件**（Markdown 格式）
- 下次遇到類似任務時，載入相關技能而非從頭推理
- **效能提升**：使用自創技能的 agent 完成研究任務比全新實例快 40%
- 技能在使用中**自動修補**（發現落後時）
- 自動策展器（curator）定期回顧、整合重複技能、歸檔過時技能

來源：[Skills System Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)

### 4. 記憶體提供者生態

支援的 7 種記憶提供者：
1. **honcho** — 辯證推理 + 深層用戶建模（Plastic Labs）
2. **mem0** — 結構化記憶存儲
3. **supermemory** — 雲端記憶服務
4. **hindsight** — 事後分析型記憶
5. **holographic** — 向量記憶
6. **retaindb** — 資料庫型記憶
7. **byterover** — 輕量記憶後端

**Honcho 辯證推理機制**：
- 每次對話後分析交換內容，推導用戶偏好、習慣、目標
- 提供兩層 context 注入：基礎層（會話摘要 + 用戶代表）+ 辯證補充層
- 5 個專用工具：`honcho_profile`, `honcho_search`, `honcho_context`, `honcho_reasoning`, `honcho_conclude`

來源：[Memory Providers Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)、[Honcho Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/honcho)

### 5. MCP (Model Context Protocol) 整合

- 在啟動時發現 MCP 伺服器，工具自動注入所有 `hermes-*` platform toolsets
- 每個 MCP 伺服器動態建立 `mcp-<server>` toolset
- 支援 stdio（命令型）和 HTTP 兩種傳輸方式
- 支援 `notifications/tools/list_changed`：伺服器可在執行期動態更新工具清單

來源：[MCP Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp)

### 6. RL 訓練生態

- 與 **Atropos** 整合：軌跡 API 伺服器協調環境互動、管理 rollout 群組
- 與 **Tinker** 整合：處理模型權重、LoRA 訓練、採樣/推理
- 使用 GRPO（Group Relative Policy Optimization）+ LoRA adapters
- `trajectory_compressor.py`：壓縮長 agent 軌跡至可訓練 token 預算
- 批次軌跡生成：千次並行工具呼叫軌跡，輸出 ShareGPT 格式供微調

來源：[RL Training Docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/rl-training)、[DeepWiki Analysis](https://deepwiki.com/NousResearch/hermes-agent/10.5-rl-training-environments)

### 7. 相關生態與整合

- **agentskills.io 標準**：Hermes 相容的開放技能標準，跨 agent 平台共享技能
- **hermes-agent-self-evolution**：使用 DSPy + GEPA 自動最佳化技能、提示和程式碼
- **HermesClaw**：社群 WeChat 橋接，讓 Hermes 和 OpenClaw 共用同一個 WeChat 帳號
- **OpenClaw 遷移**：自動匯入設定、記憶、技能、API 金鑰

### 8. 部署模式

- **本地端**：直接在 Linux/macOS/WSL2/Termux 運行
- **$5 VPS**：輕量雲端部署
- **Docker**：容器化部署（提供 docker-compose.yml）
- **Modal**：無伺服器部署，閒置成本幾乎為零
- **Daytona**：沙盒持久化環境
- **Singularity**：科學計算叢集環境

## 搜尋發現但未在程式碼中驗證的聲明

⚠️ 以下資訊來自線上搜尋，尚未完全對照程式碼驗證：

- 「61 個工具 across 52 個 toolsets」（文件數字，程式碼中數字可能隨版本更新）
- 「7 種終端機後端」（加入 Vercel Sandbox；程式碼中的 `environments/` 確認 6 種，Vercel 在 `pyproject.toml` 中有 optional extra）
- 「效能提升 40%」（Nous Research 內部 benchmark，未公開原始資料）
- 「118 個技能（96 捆綁 + 22 可選）」（版本相關，代碼倉庫數字可能不同）
