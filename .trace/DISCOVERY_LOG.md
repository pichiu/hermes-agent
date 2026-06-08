# Hermes Agent 探索紀錄與待解問題

**建立日期**：2026-06-08  
**版本**：Hermes Agent 0.16.0  
**探索階段**：Stage 1（Web Search）+ Stage 2（程式碼 Recon）

---

## 1. Web Search 發現摘要

### 架構定位

Hermes Agent 由 Nous Research 開發，定位為「The self-improving AI agent」，核心差異化為內建學習迴路（Learning Loop）。與其他 agent 框架最大的不同在於：

- **Skills 即程序性記憶**：任務完成後，背景 LLM agent 自動分析並生成 Skill Document（Markdown 格式），供後續任務查找使用
- **FTS5 全文搜尋索引**：跨 session 召回歷史 skill 與對話（SQLite FTS5）
- **agentskills.io 生態系相容**：Skills 格式為開放標準，可與其他相容 agent 共享

### 生態系現況（截至 2026-06）

| 資源 | URL | 說明 |
|------|-----|------|
| 官方文件 | https://hermes-agent.nousresearch.com/docs/ | 完整 |
| Skills Hub | https://agentskills.io | Skills 分享平台 |
| 社群 Discord | https://discord.gg/NousResearch | 官方社群 |
| PyPI | https://pypi.org/project/hermes-agent/ | 套件發布 |
| awesome-hermes-agent | https://github.com/0xNyk/awesome-hermes-agent | 社群資源清單 |
| DeepWiki | https://deepwiki.com/NousResearch/hermes-agent | 自動生成架構文件 |

### 重要技術背景

1. **供應鏈攻擊應對**：2026-05-12 發生 Mini Shai-Hulud worm（`mistralai 2.4.6` 套件）攻擊，導致 `pyproject.toml` 所有依賴鎖定精確版本
2. **ACP 協議**：Agent Client Protocol v0.11.0（2026-03-04），已獲 Zed、JetBrains 官方合作，讓編輯器直接嵌入 Hermes
3. **Nous Portal**：統一訂閱入口，提供 300+ 模型、網路搜尋、圖像生成、TTS、Cloud Browser
4. **Honcho 整合**：plastic-labs/honcho 提供 dialectic user modeling（辯證式使用者建模）

---

## 2. 既有文件與程式碼落差

| 狀態 | 項目 | 說明 |
|------|------|------|
| 落差 | Discord gateway 位置 | `recon.md` 標注為「未找到」，實際確認在 `plugins/platforms/discord/`（非 `gateway/platforms/`） |
| 落差 | `plugins/platforms/` 平台數量 | recon 文件描述 `gateway/platforms/` 含 20+ 平台，但 Discord/IRC/LINE/Mattermost 等實際位於 `plugins/platforms/` 子目錄 |
| 落差 | `agent/secret_sources/` | configuration.md 標注「⚠️ 待驗證」，實際確認目錄**不存在**，相關功能可能以不同形式實作 |
| 落差 | `plugins/observability/nemo_relay/` | 實際確認存在 `nemo_relay/` 子目錄，文件記載正確 |
| 落差 | Web frontend 技術棧 | recon.md 標注「Vite + React（⚠️ 推測）」，已驗證：確為 Vite + React（含 `@xterm`、`@react-three/fiber`、`@observablehq/plot`、`@nous-research/ui` 0.18.2） |
| 落差 | CONTRIBUTING.md vs. 程式碼 | 文件說明記憶 provider 應為獨立 repo，但 `plugins/memory/` 仍內建多個 provider，未完全外包 |
| ✓ 一致 | 三層架構 | CLI → AIAgent → conversation_loop 架構與文件完全一致 |
| ✓ 一致 | max_iterations = 90 | `run_agent.py:388` 已驗證 |
| ✓ 一致 | Observer Hook 事件清單 | `docs/observability/README.md` 與程式碼一致 |

---

## 3. TODO / FIXME / HACK 彙整

從原始碼掃描結果（`grep -rn "TODO\|FIXME\|HACK\|XXX\|WORKAROUND"`）：

| 位置 | 類型 | 內容摘要 |
|------|------|---------|
| `agent/auxiliary_client.py:4457` | TODO | OpenAI SDK 內部標注的 `TODO(someday)` 被引用，涉及 SDK 尚未實作的功能 |
| `agent/transports/types.py:30` | XXX | Codex transport 的 `call_id`/`response_item_id` 格式說明（`call_XXX`、`fc_XXX`） |
| `agent/context_compressor.py:288` | 注意 | 保留 `ensure_ascii=False` 以防 CJK/emoji 被膨脹為 `\uXXXX` 格式（有說明注釋） |
| `agent/prompt_builder.py:151` | 設計決定 | 明確禁止 agent 把 TODO 存入記憶（防止記憶污染） |
| `tools/memory_tool.py:666` | 設計決定 | 同上，memory tool 層面的限制 |
| `tools/file_operations.py:25` | 範例程式碼 | 工具使用範例中含 `search("TODO")` 字串（非實際 TODO） |
| `tools/todo_tool.py:209,272` | Schema 定義 | `TODO_SCHEMA` 為任務管理工具的正式 schema 定義，非技術債 |

**結論**：原始碼中未發現高風險的 `FIXME` 或 `HACK` 標注；主要為設計決定注釋與 SDK 外部 TODO 引用。

---

## 4. 未解答的疑問（⚠️ 未驗證項目整理）

### 已解答（本次探索確認）

| 原始疑問 | 確認結果 |
|---------|---------|
| Discord 平台位置 | `plugins/platforms/discord/`（目錄形式，非單一 `.py`） |
| `agent/secret_sources/` 是否存在 | **不存在**，可能整合在 `agent/credential_sources.py` 中 |
| NeMo Relay README | `plugins/observability/nemo_relay/` 確認存在 |
| Web 前端技術棧 | Vite + React + TypeScript，確認非推測 |
| `plugins/platforms/` 有哪些平台 | discord, google_chat, homeassistant, irc, line, mattermost, ntfy, simplex, teams |

### 仍未解答

| 疑問 | 影響範圍 | 優先度 |
|------|---------|------|
| Context 壓縮演算法的具體實作（token 估算只用字元數 / 3.5？是否有更精確路徑？） | 長對話品質 | 高 |
| Skill 評分/改善機制的完整流程（curator 如何決定 pin/archive？） | 學習迴路品質 | 高 |
| Gateway session 持久化機制（跨 restart 如何恢復？） | 可靠性 | 中 |
| Credential pool 輪換策略（Round-robin？Weighted？） | 高流量穩定性 | 中 |
| `agent/credential_sources.py` 是否支援 1Password/Vault 整合 | Secret 管理 | 中 |
| `~/.hermes/.container-mode` 的確切格式與作用 | 容器部署 | 低 |
| SerpAPI 是否為有效的 web 搜尋 provider | 搜尋功能 | 低 |
| ElevenLabs / MiniMax TTS 整合狀態 | TTS 功能 | 低 |
| Circuit breaker 是否存在（⚠️ 未找到顯式實作） | 穩定性 | 中 |
| `hermes-acp` toolset 的工具清單（與標準 toolset 的差異） | 編輯器整合 | 低 |

---

## 5. 已知技術債

### 大型檔案（高維護成本）

| 檔案 | 大小 | 說明 | 風險 |
|------|------|------|------|
| `cli.py` | ~737 KB | 主 CLI REPL，完整 TUI 邏輯，包含 prompt_toolkit 整合 | 高：單一檔案過大，難以單元測試 |
| `run_agent.py` | ~234 KB | AIAgent 主類別 + 初始化 | 高：核心類別與初始化混合 |
| `hermes_state.py` | ~187 KB | 全局狀態 | 中：全局狀態設計限制測試性 |
| `agent/conversation_loop.py` | ~3900 行 | 主要對話迴路 | 高：核心邏輯高度集中 |
| `agent/auxiliary_client.py` | 含 4457+ 行 | 輔助 LLM client | 中 |
| `cli-config.yaml.example` | ~62 KB | 設定範例文件 | 低：只是文件，但維護負擔重 |

### 歷史包袱

1. **Gateway vs. Plugin Platform 分裂**：部分平台在 `gateway/platforms/`，部分在 `plugins/platforms/`，邊界不清晰（Discord、IRC、LINE 等移至 plugin，但 Telegram、Slack 仍在 gateway）
2. **`agent/` 目錄的 Adapter 爆炸**：隨 LLM provider 增加，`agent/*_adapter.py` 數量持續增長，尚無統一 provider registry
3. **`cli.py` 的 Monolith 問題**：737KB 的單一檔案涵蓋 REPL、slash commands、串流輸出等，重構風險極高
4. **Token 估算粗略**：`context_compressor.py` 使用字元數 / 3.5 估算 token，對 CJK 語言（每字 ~1 token）可能導致提前或過晚觸發壓縮
5. **依賴版本強鎖（supply chain 事件後遺症）**：精確 pin 版本解決安全問題，但升級第三方庫的開銷顯著增加

---

## 6. 需要更深入調查的區域

依優先度排序：

| 優先度 | 區域 | 建議調查方式 |
|--------|------|------------|
| 🔴 高 | `agent/conversation_loop.py`：`_run()` 函式的完整迴路邏輯 | 精讀 + 繪製流程圖 |
| 🔴 高 | `agent/context_compressor.py`：完整壓縮算法（trigger 閾值、保留策略） | 精讀 + 測試實驗 |
| 🔴 高 | `agent/curator.py`：Skill 生命週期管理邏輯（pin/archive/consolidate 決策） | 精讀 |
| 🟡 中 | `gateway/session.py`：Session 持久化與跨 restart 恢復機制 | 精讀 |
| 🟡 中 | `agent/credential_sources.py`：Secret 讀取來源（是否含 1Password/Vault） | 精讀 |
| 🟡 中 | `agent/iteration_budget.py`：Subagent budget 繼承邏輯 | 精讀 |
| 🟡 中 | `plugins/platforms/` 各平台：Gateway 與 Plugin 平台的架構差異 | 比較 `gateway/platforms/telegram.py` vs `plugins/platforms/discord/` |
| 🟢 低 | `acp_adapter/`：ACP 協議實作細節（與 Zed/JetBrains 整合方式） | 精讀 |
| 🟢 低 | `tools/tool_result_storage.py`：`enforce_turn_budget()` 的裁切策略 | 精讀 |
| 🟢 低 | `batch_runner.py`：研究/評測批次執行的完整架構 | 精讀 |

---

## 7. 社群與維護者確認清單

需要向 NousResearch 維護者或社群確認的問題：

### 架構決定

- [ ] **Gateway vs. Plugin Platform 邊界**：新平台應放在 `gateway/platforms/` 還是 `plugins/platforms/`？決定標準是什麼（官方 vs. 社群？維護方式？）
- [ ] **`agent/secret_sources/` 是否計畫中**：configuration.md 提到從 1Password/Vault 讀取 secret，這是已存在的功能（`credential_sources.py`）還是計畫中的功能？
- [ ] **Circuit breaker 設計**：目前只有 retry + failover，是否計畫加入 circuit breaker 避免 thundering herd 在多 instance 部署時發生？

### 技術細節

- [ ] **Token 估算**：context_compressor 的字元 / 3.5 估算是否有精確模式（tiktoken 或 model tokenizer）？何時會切換？
- [ ] **Skill Curator 決策依據**：curator LLM 用什麼 prompt 決定 skill 的 pin/archive？是否有可配置的閾值？
- [ ] **ACP 協議版本相容性**：hermes-agent 0.16.0 支援的 ACP 版本範圍？v0.11.0 是否向下相容？

### 生態系

- [ ] **SerpAPI 支援狀態**：web_findings.md 提到 SerpAPI，但 pyproject.toml 未列為依賴，是否仍支援或已移除？
- [ ] **agentskills.io 審核機制**：社群發布的 skill 是否有安全審查機制？安裝第三方 skill 的安全風險如何控管？
- [ ] **Honcho 整合成熟度**：dialectic user modeling 在 0.16.0 是 experimental 還是 stable？是否建議生產環境使用？

### 部署與維運

- [ ] **Multi-gateway 文件**：`docs/kanban/multi-gateway.md` 描述的多 gateway 部署是否已生產就緒？負載分配策略為何？
- [ ] **Windows 支援等級**：`build-windows-installer.yml` 存在，Windows 是 tier-1 還是 best-effort 支援？
- [ ] **Docker 映像更新頻率**：`docker-publish.yml` 的映像是否跟隨每個 release tag 發布到 Docker Hub？

---

*本文件由 Hermes Agent codebase 探索自動整理，部分資訊來自 web search（標注原始資料來源）。標注 ⚠️ 的項目表示資訊尚未從原始程式碼直接驗證。*
