# DISCOVERY_LOG.md — 探索紀錄與待解問題

> 產出日期：2026-05-05  
> 版本：Hermes Agent v0.12.0  
> 維護者：Nous Research

---

## 1. Web Search 發現摘要

| 來源 | 連結 | 關鍵 Takeaway |
|------|------|--------------|
| GitHub 主庫 | https://github.com/NousResearch/hermes-agent | 原始碼、Issues、PR；截至 2026-05 已達 64,200+ stars |
| 官方架構文件 | https://hermes-agent.nousresearch.com/docs/developer-guide/architecture | 核心 5 大入口點、61 工具 across 52 toolsets |
| 技能文件 | https://hermes-agent.nousresearch.com/docs/user-guide/features/skills | 自我進化迴路說明（技能創建 → 修補 → Curator 策展）|
| MCP 整合 | https://hermes-agent.nousresearch.com/docs/user-guide/features/mcp | stdio/HTTP 兩種傳輸、`notifications/tools/list_changed` 動態更新 |
| RL 訓練 | https://hermes-agent.nousresearch.com/docs/user-guide/features/rl-training | Atropos + Tinker + GRPO + LoRA pipeline |
| 記憶體提供者 | https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers | 7 種記憶提供者，含 Honcho 辯證推理 |
| Honcho 整合 | https://docs.honcho.dev/v3/guides/integrations/hermes | Honcho 官方整合說明（5 個專用工具）|
| DeepWiki 分析 | https://deepwiki.com/NousResearch/hermes-agent | 自動生成程式碼分析，含 RL 環境詳述 |
| agentskills.io | https://agentskills.io | 開放技能標準市集，Hermes 相容格式 |
| hermes-agent-self-evolution | https://github.com/NousResearch/hermes-agent-self-evolution | DSPy + GEPA 自動最佳化技能、提示和程式碼 |
| awesome-hermes-agent | https://github.com/0xNyk/awesome-hermes-agent | 社群整理資源清單（含 HermesClaw WeChat 橋接）|

### 關鍵 Takeaway 彙整

- **規模**：2026-02-25 發布，3 個月內達 64,200+ stars，顯示快速採用。
- **自我進化**：Nous Research 內部 benchmark 聲稱使用自創技能可加速 40%，但**未公開原始資料**（⚠️ 未驗證）。
- **生態系**：已出現社群標準（agentskills.io）、自動進化工具（self-evolution repo）及橋接器（HermesClaw）。
- **部署彈性**：支援 $5 VPS 到 HPC 叢集（Singularity），無伺服器（Modal）閒置成本幾乎為零。

---

## 2. 文件與程式碼之間的落差

| 編號 | 文件描述 | 實際程式碼情況 | 嚴重度 |
|------|---------|--------------|--------|
| G-01 | 文件說「7 種終端機後端」，列出 Vercel Sandbox | `tools/environments/vercel_sandbox.py` 確實存在，`recon.md` 最初說「6 種」| 低（已確認實作）|
| G-02 | 文件說「61 工具 across 52 toolsets」 | 工具數隨版本更新；`toolsets.py` 為真實來源，文件數字可能落後 | 中 |
| G-03 | 文件說「118 個技能（96 捆綁 + 22 可選）」 | `skills/` 和 `optional-skills/` 實際數量需重新計數；AGENTS.md 明示「File counts shift constantly」| 中 |
| G-04 | `recon.md` 記憶提供者中 OpenViking 標注為「⚠️ 未驗證」 | `plugins/memory/openviking/` 目錄存在，`hermes_cli/main.py:9448` 和 `hermes_cli/config.py:923` 均有引用 | 低（已確認實作）|
| G-05 | `recon.md` 記憶提供者中 RetainDB / ByteRover 標注為「⚠️ 未驗證」 | `plugins/memory/retaindb/` 與 `plugins/memory/byterover/` 目錄存在；`agent/redact.py:99,102` 已有 API key pattern 定義 | 低（已確認實作）|
| G-06 | `integrations.md` 中 Tavily 標注「⚠️ 未驗證是否實作」 | `tools/web_tools.py:322-341` 已有完整 `_tavily_request()` 實作，且在 backend 選擇邏輯（:129）中正常處理 | 低（已確認實作）|
| G-07 | README 列出 `hermes dashboard` 指令 | `web/` 目錄以 `fastapi+uvicorn` 提供，實作存在但需確認指令入口是否完整串接 | 中 |
| G-08 | 文件描述「效能提升 40%」 | Nous Research 內部 benchmark，未公開原始資料或測試方法論 | 高（⚠️ 未驗證）|
| G-09 | `web_findings.md` 說 `managed_modal.py` | `tools/environments/managed_modal.py` 確實存在（非 context 文件所列）| 低 |

---

## 3. 程式碼中發現的 TODO / FIXME / HACK 彙整

> grep 指令：`grep -rn "TODO|FIXME|HACK|WORKAROUND" /home/user/hermes-agent/ --include="*.py"`

### 3.1 待完成項目（TODO）

| 位置 | 行號 | 內容摘要 |
|------|------|---------|
| `gateway/platforms/yuanbao.py` | 4562 | `TODO (T06): fetch real chat name/member-count from Yuanbao API.` — Yuanbao 聊天室名稱與成員數尚未從 API 取得，目前為佔位符 |
| `agent/auxiliary_client.py` | 2887 | 引用 OpenAI SDK 內部 TODO，說明 SDK 本身未解決的非同步問題（`# TODO(someday): ...`）|

### 3.2 執行期警告（⚠️ Runtime Warnings — run_agent.py）

以下為 `run_agent.py` 中的執行期警告點，代表已知的邊緣情況：

| 行號 | 警告情境 |
|------|---------|
| 1555 | API key 無效或遺漏 |
| 1612 | 工具缺少必要依賴（missing requirements）|
| 6497 | 提供商長時間無回應（stale timeout）|
| 7228 | 工具呼叫中途連線中斷 |
| 9217 | 會話壓縮次數過多（可能影響上下文品質）|
| 10240 | 達到最大迭代次數（max_iterations）|
| 10835 | Iteration budget 耗盡 |
| 11371 | 收到空白/格式錯誤的回應 |
| 11527 | 回應被截斷（finish_reason='length'）|
| 11592 | Thinking budget 耗盡 |
| 11656 | 工具呼叫被截斷，重試中 |
| 11908 | 訊息中發現無效 surrogate 字元 |
| 12198 | Thinking block signature 無效 |
| 12315 | Anthropic long-context tier 問題 |
| 12452 | 請求 payload 過大（413）|
| 12590 | Context length 超限，觸發壓縮降級 |

---

## 4. 未解答的疑問與模糊地帶

| 編號 | 問題 | 來源 |
|------|------|------|
| Q-01 | `managed_modal.py` 與 `modal.py` 有何差異？`managed_modal.py` 是較新的封裝還是實驗性功能？ | `tools/environments/` 目錄存在兩個 Modal 相關檔案 |
| Q-02 | `plugins/memory/holographic/` 依賴的 `holographic` 套件（`hrr` 別名）是哪個 PyPI 套件？pyproject.toml 是否有 optional extra？ | `plugins/memory/holographic/store.py:12-14` |
| Q-03 | Yuanbao 平台的「T06 TODO」（傳送訊息時的群組名稱）何時會完成？目前有無降級行為或錯誤拋出？ | `gateway/platforms/yuanbao.py:4562` |
| Q-04 | `hermes-agent-self-evolution`（DSPy + GEPA）是否為官方支援的功能，或純屬實驗性 side project？其技能格式是否與主庫的 SKILL.md 規格相容？ | `web_findings.md` |
| Q-05 | `agentskills.io` 技能市集是否由 Nous Research 官方維護？第三方技能的安全審查機制為何？ | `web_findings.md` |
| Q-06 | `batch_runner.py` 的平行批次處理是否有工作數量上限？大規模批次（1000+ 任務）的記憶體與連線管理策略為何？ | `recon.md` 統計資訊 |
| Q-07 | Gateway 中的訊息遞送（`gateway/delivery.py`）在分塊（chunking）時，若中途平台斷線，是否有冪等重送機制？ | `integrations.md` |
| Q-08 | `trajectory_compressor.py` 壓縮長軌跡至「可訓練 token 預算」的演算法細節為何？如何確保壓縮不損失關鍵推理步驟？ | `recon.md` |
| Q-09 | ACP adapter（`acp_adapter/`）與 VS Code/Zed/JetBrains 的整合是否需要額外安裝 extension？是否已在官方 marketplace 發布？ | `recon.md` 文件落差 |
| Q-10 | `hermes_cli/skin_engine.py` 的主題引擎支援哪些自訂維度？是否有公開的主題格式規格？ | `recon.md` 目錄結構 |

---

## 5. 已知技術債

以下來自 `AGENTS.md` 的 Known Pitfalls 與 DO NOT 清單：

### 5.1 路徑硬編碼（AGENTS.md:618-621）

**問題**：程式碼若硬編碼 `~/.hermes` 路徑，會導致 Profile 功能失效。  
**正確做法**：所有程式碼路徑用 `get_hermes_home()`，用戶顯示訊息用 `display_hermes_home()`。  
**背景**：此問題是 PR #3575 修復的 5 個 bug 的共同根源。  
**技術債**：需持續稽核新貢獻是否引入硬編碼路徑。

### 5.2 `simple_term_menu` 殘留（AGENTS.md:623-628）

**問題**：`simple_term_menu` 在 tmux/iTerm2 中有 ghost-duplication 渲染 bug。  
**現狀**：`hermes_cli/main.py` 中仍有殘留的舊呼叫點（legacy fallback）。  
**正確做法**：新互動選單必須使用 `hermes_cli/curses_ui.py`。  
**技術債**：舊 `simple_term_menu` 呼叫點尚未全部遷移。

### 5.3 ANSI escape `\033[K` 洩漏（AGENTS.md:630-631）

**問題**：在 `prompt_toolkit` 的 `patch_stdout` 模式下，`\033[K` 會顯示為字面文字 `?[K`。  
**解法**：改用空格填充 `f"\r{line}{' ' * pad}"`。  
**技術債**：若 spinner/display 相關模組有新開發者，容易重複犯此錯誤。

### 5.4 `_last_resolved_tool_names` 全域變數（AGENTS.md:633-634）

**問題**：`model_tools.py:213` 中的 `_last_resolved_tool_names` 是 process-global，在子代理執行期間會暫時失效。  
**現狀**：`tools/delegate_tool.py:1287,1810` 以 save/restore 模式處理，但任何新讀取此全域的程式碼若未注意，將讀到過期值。  
**技術債**：全域狀態設計，理想應改為 context-local 傳遞。

### 5.5 Gateway 雙重訊息守衛（AGENTS.md:639-648）

**問題**：Gateway 有兩個序列守衛：(1) `gateway/platforms/base.py` 的 `_pending_messages`，(2) `gateway/run.py` 的指令攔截層。任何需要在 agent 阻塞時觸達 runner 的新指令，必須繞過兩個守衛並以 inline 方式分派，不可透過 `_process_message_background()`（會有 session lifecycle race condition）。  
**技術債**：守衛機制設計較複雜，新開發者容易在新增平台指令時犯錯。

### 5.6 Squash Merge 靜默 Revert（AGENTS.md:650-656）

**問題**：從過時分支做 squash merge 時，該分支版本的不相關檔案會靜默覆蓋 main 上的近期修復。  
**解法**：squash merge 前確保分支已同步 main，並用 `git diff HEAD~1..HEAD` 驗證。  
**技術債**：CI 流程未自動阻擋此情況，仰賴人工檢查。

### 5.7 Dead Code 接線風險（AGENTS.md:658-661）

**問題**：未使用的程式碼通常有理由從未上線。若未做 E2E 驗證就將死程式碼接入 live code path，可能引入隱性 bug。  
**技術債**：需要 E2E 測試（使用真實 import 而非 mock）搭配臨時 HERMES_HOME。

### 5.8 測試隔離規則（AGENTS.md:663-677）

**問題**：測試若寫入真實 `~/.hermes/` 會汙染開發者環境。  
**機制**：`tests/conftest.py` 的 `_isolate_hermes_home` autouse fixture 自動重導至 temp dir。  
**技術債**：Profile 相關測試還需額外 mock `Path.home()`，規則較複雜，容易遺漏。

---

## 6. 需要更深入調查的區域

| 編號 | 區域 | 原因 | 優先度 |
|------|------|------|--------|
| I-01 | `agent/context_compressor.py` 壓縮品質 | 多次壓縮（`run_agent.py:9217` 警告）後的上下文品質衰退問題需實測 | 高 |
| I-02 | `tools/environments/managed_modal.py` | 與 `modal.py` 的職責劃分不明確，文件中未提及 | 高 |
| I-03 | `gateway/delivery.py` 分塊遞送 | 長訊息分塊邏輯、媒體處理、平台斷線重送機制尚未調查 | 高 |
| I-04 | `agent/credential_pool.py` 多 key 輪換 | key 耗盡後的行為、輪換策略（round-robin vs. weighted）需確認 | 中 |
| I-05 | `acp_adapter/` ACP 整合 | VS Code/Zed/JetBrains extension 發布狀態、協定規格未確認 | 中 |
| I-06 | `environments/` RL 訓練環境 | Atropos 環境的 reward function 設計、現有任務類型、與 Tinker 的訓練 loop 細節 | 中 |
| I-07 | `cron/` 排程器 | jobs.py 和 scheduler.py 的持久化機制（restart survival）、與 session 的關係 | 中 |
| I-08 | `hermes_cli/skin_engine.py` 主題格式 | 未公開的主題規格，對社群開發者不友善 | 低 |
| I-09 | `tools/browser_camofox.py` 指紋偽裝 | 反爬蟲偵測的合法性與維護成本 | 低 |
| I-10 | `plugins/memory/holographic/` 依賴套件 | `hrr` 別名的 PyPI 套件來源不明（⚠️ 未驗證）| 低 |

---

## 7. 與維護者確認的問題清單

以下問題需要與 Nous Research 維護者確認，無法僅從程式碼或文件得出答案：

### 架構決策類

1. **`managed_modal.py` vs `modal.py`**：兩個 Modal 環境後端的設計意圖為何？`managed_modal.py` 是否為 `modal.py` 的繼任者，抑或對應不同使用情境？

2. **`hermes-agent-self-evolution` 官方化計畫**：DSPy + GEPA 自動最佳化的 self-evolution repo 是否計畫合併進主庫？技能格式是否保持向下相容？

3. **效能提升 40% 的 Benchmark 方法論**：「使用自創技能快 40%」的測試條件（任務類型、模型、硬體）為何？是否有意公開原始資料？

### 穩定性與 Roadmap 類

4. **Yuanbao 平台的 T06 TODO**（`gateway/platforms/yuanbao.py:4562`）：這個「取得真實聊天室資訊」的 TODO 是否列在 roadmap 上？目前的降級行為（fallback）是什麼？

5. **多次上下文壓縮的品質保證**：當 `run_agent.py:9217` 觸發「Session compressed N times」警告時，有無品質監控機制防止資訊嚴重流失？

6. **Gateway 雙重守衛的長期計畫**：`AGENTS.md:639` 描述的雙重守衛設計是否計畫在未來重構為更簡潔的架構？

### 生態系類

7. **agentskills.io 安全審查**：第三方技能安裝（`hermes skills install <source>/<name>`）是否有程式碼審查或沙盒機制防止惡意技能？

8. **ACP extension 發布狀態**：`acp_adapter/` 對應的 VS Code/Zed/JetBrains extension 是否已在各自的 marketplace 上架？安裝文件在哪裡？

9. **`holographic` memory provider 的依賴**：`plugins/memory/holographic/` 使用的 `hrr` 套件是否有 PyPI 頁面，或需要從 source 安裝？

---

## 附錄：驗證狀態修正清單

以下為本次調查中，相較 context 文件內「⚠️ 未驗證」標注，已在程式碼中確認的項目：

| 原始標注 | 確認結果 | 確認位置 |
|---------|---------|---------|
| Vercel Sandbox 後端「⚠️ 未驗證」 | 已確認實作 | `tools/environments/vercel_sandbox.py:1` |
| OpenViking 記憶提供者「⚠️ 未驗證」 | 已確認實作 | `plugins/memory/openviking/`、`hermes_cli/main.py:9448` |
| RetainDB「⚠️ 未驗證」 | 已確認實作 | `plugins/memory/retaindb/`、`agent/redact.py:99` |
| ByteRover「⚠️ 未驗證」 | 已確認實作 | `plugins/memory/byterover/`、`agent/redact.py:102` |
| Tavily 搜尋「⚠️ 未驗證是否實作」 | 已確認完整實作 | `tools/web_tools.py:322-341` |

---

<!-- 以下段落更新於 2026-05-08, commit range: 601e5f1..faa13e49, v0.12→v0.13 -->

## v0.13.0 增量更新發現（2026-05-08）

### 重大架構決策（v0.13 新增）

**D-01：ProviderProfile 的雙重 Discovery 系統設計**
`ProviderProfile` 由 `providers._discover_providers()` 懶惰載入，與 PluginManager 完全分離。這個決策的核心原因是「避免雙重實例化」— 若 PluginManager 也 import model-provider plugins，每個 `ProviderProfile` 會被初始化兩次，可能觸發重複 API call（如 `fetch_models()`）。新增 model-provider plugin 時要注意這個差異：`plugin.yaml` 的 `kind: model-provider` 僅供 PluginManager 記錄，不觸發 import。

**D-02：`env_enablement_fn` 平台 hook 的意涵**
IRC 和 Teams 從 built-in `gateway/platforms/` 遷移至 `plugins/platforms/` 後，使用 `env_enablement_fn` hook 決定是否啟動平台。這代表「需要特殊環境設定的平台」未來都應透過插件機制，而非修改 `gateway/run.py` 核心。Google Chat 是第一個從一開始就是插件型的平台，其他新平台預計跟進。

### 新的文件/程式碼落差（v0.13 新增）

**G-10：Checkpoints v2 的恢復語意未說明**
`hermes_cli/checkpoints.py` 提供了快照管理，但恢復語意（覆蓋 vs 分支、部分恢復、在 gateway 模式下的行為）在文件中未說明。`/rollback` 指令的完整語意也未在任何文件中描述。

**G-11：`transform_llm_output` hook 的副作用邊界**
新的 `transform_llm_output` plugin hook 允許插件在 LLM 輸出進入對話前改寫，但官方文件未說明輸出長度、格式（是否可以是 non-JSON）、回傳 None 的行為，以及是否可以拋出例外中斷對話。

**G-12：i18n locale 覆蓋率**
`locales/en.yaml` 的 `approval` 段落僅涵蓋 approval prompt 和少量 gateway 指令回覆。哪些訊息被翻譯、哪些留英文，未在任何地方有完整列表。`tests/agent/test_i18n.py` 強制所有 locale 需要 catalog parity，但沒有說明「可以安全添加 locale 的訊息範圍」。

### 待驗證問題（v0.13 新增）

10. **`StreamingThinkScrubber` 的 flush 語意**：`think_scrubber.flush()` 在串流結束時被呼叫（`run_agent.py:6816`），flush 的輸出會被丟棄還是附加到回應中？⚠️ 未驗證
11. **Checkpoints v2 與 git worktree 的關係**：v0.12 的 checkpoint 使用 shadow git repo，v0.13 release notes 說「no more orphan shadow repos」，新機制是純檔案 snapshot 嗎？⚠️ 未驗證
12. **Google Chat OAuth 流程**：`plugins/platforms/google_chat/oauth.py` 存在，但 OAuth app 設定流程（GCP 專案、Workspace 授權、webhook URL）未在核心文件中說明。⚠️ 未驗證

### 已由 v0.13 關閉的問題

- G-04（DM Pairing 機制）：`gateway/pairing.py` 在 v0.13 中新增了 lockout 修正（`fix(pairing): enforce lockout on approve_code`），安全性有改善，但 UX 文件仍缺失。
- 安全性波：redaction 預設開啟（`agent/redact.py`）、Discord role-allowlist guild-scoped、WhatsApp 預設拒絕陌生人 — 這 3 個 P0 安全問題已在 v0.13 關閉。

<!-- 更新結束 -->
