# Hermes Agent — 版本變更記錄

> 追蹤從 v0.8.0（2026.4.8）開始的版本差異，供開發者快速掌握各版本重點變化。

---

## v0.10.0 (2026.4.16) — Tool Gateway 版本

**Release commit:** `1dd6b5d`
**Since v0.9.0:** 180+ commits

> 付費 Nous Portal 訂閱用戶可透過現有訂閱使用 Web 搜尋、圖片生成、TTS、瀏覽器自動化，零額外 API 金鑰。

### 🌟 核心亮點

#### Nous Tool Gateway（主要新功能）
- 付費 Portal 用戶自動獲得以下工具的 gateway 存取權：
  - **Web 搜尋**（Firecrawl）
  - **圖片生成**（FAL / FLUX 2 Pro）
  - **TTS**（OpenAI TTS）
  - **瀏覽器自動化**（Browser Use）
- 以 `use_gateway` config 做 per-tool opt-in
- 與 `hermes tools` 和 `hermes status` 完整整合
- Runtime 在 gateway 和直接 API key 並存時優先使用 gateway
- 取代舊的隱藏環境變數 `HERMES_ENABLE_NOUS_MANAGED_TOOLS`
- 相關 PR：[#11206](https://github.com/NousResearch/hermes-agent/pull/11206)，文件：[#11208](https://github.com/NousResearch/hermes-agent/pull/11208)

### 🏗️ 架構與 Provider

| 功能 | 說明 |
|------|------|
| **Claude Opus 4.7 完整支援** | API 遷移完成，含 xhigh→max adaptive model 修正 |
| **xAI Responses API + TTS** | xAI 升級至 Responses API，新增 TTS provider |
| **Ollama Cloud** 內建 provider | 新增 Ollama Cloud 為直接可選的 provider |
| **多模型 FAL 支援** | FAL 整合多模型，`hermes tools` 顯示 picker |
| **Google Gemini CLI OAuth** | 透過 Cloud Code Assist 支援免費/付費 Gemini 存取 |
| **Ollama `think=false`** | reasoning_effort 為 none 時傳入以停用 thinking |
| **GLM-5.1** | 新增至 opencode-go catalogs |

### 🖥️ Dashboard / Web UI

| 功能 | 說明 |
|------|------|
| **Dashboard 主題系統** | 即時切換主題，含文件 |
| **Dashboard 插件系統** | 以自訂 tab 擴充 Web UI，新增 `dispatch_tool()` 到 PluginContext |
| **Dashboard 文件頁面** | themes、plugins、tool-gateway 說明頁加入 sidebar |

### 📱 Gateway 平台

| 平台 | 變更 |
|------|------|
| **Slack** | DM 預設改為 per-thread session |
| **Telegram** | 新增專屬 `TELEGRAM_PROXY` 環境變數 + `proxy_url` config；冷啟動 retry 優化 |
| **Feishu** | 補上缺少的 P2P chat entered / message recalled 事件處理器 |
| **DingTalk** | 修復 dingtalk-stream >= 0.20 SDK shape 相容性 |

### 🔧 工具系統

| 功能 | 說明 |
|------|------|
| **MCP circuit breaker** | 防止 MCP tool 失敗觸發 retry 燃燒迴圈 |
| **File sync teardown** | 遠端沙盒的變更在 teardown 時同步回本機 |
| **execute_code timeout** | 執行逾時時顯示給用戶而非靜默丟棄 |
| **Browser fallback** | Cloud provider 失敗時 fallback 到本機 Chromium |

### ⚙️ CLI / 設定

| 功能 | 說明 |
|------|------|
| **`-Q` / `--quiet` 模式** | 只輸出純回應文字 |
| **`hermes memory reset`** | 新增記憶重設指令 |
| **`config.yaml` 為唯一 CWD 來源** | 廢棄 `.env` 中的 CWD 相關變數 |
| **移除 context pressure warnings** | 不再顯示 context 壓力警告 |
| **`hermes skills reset`** | 解卡 bundled skills 的新指令 |
| **Vercel 部署支援** | 新增 Vercel deployment 選項 |

### 🐛 重要 Bug 修復

- `/model` 切換不再靜默 reroute 直接 provider 到 OpenRouter
- 修復 parallel subagent polling loop 的 `UnboundLocalError`
- 修復 cron 空回應未標記為 error 的問題（#8585）
- 修復 Honcho `anyOf` schema 破壞 Fireworks 等 provider
- 修復 post-tool empty response nudge 路徑的 UnboundLocalError
- `(No response generated)` 佔位符不再洩漏給用戶
- Telegram 中 exec approval prompt 現在正確轉義指令內容

---

## v0.9.0 (2026.4.13) — The Everywhere Release

**Release commit:** `1af2e18`
**Since v0.8.0:** 487 commits · 269 merged PRs · 167 resolved issues · 493 files changed · 24 contributors

> Hermes 行動化（Termux/Android）、新增 iMessage 和 WeChat、Fast Mode、背景程序監控、本地 Web Dashboard、最深度安全加固，支援 16 個訊息平台。

### 🌟 核心亮點

#### 新平台（+3，共 16 個）
| 平台 | 說明 |
|------|------|
| **BlueBubbles（iMessage）** | 完整 adapter：自動 webhook 註冊、setup wizard、崩潰恢復 |
| **WeChat（Weixin）** | 透過 iLink Bot API：streaming、媒體上傳、markdown 連結 |
| **WeCom Callback Mode** | 企業自建應用 adapter，atomic state persistence |

#### Fast Mode（`/fast`）
- OpenAI Priority Processing + Anthropic fast tier 的優先佇列路由
- CLI toggle `/fast` 即可啟用，顯著降低延遲
- 覆蓋 GPT-5.4、Codex、Claude 等模型
- PR：[#6875](https://github.com/NousResearch/hermes-agent/pull/6875), [#6960](https://github.com/NousResearch/hermes-agent/pull/6960), [#7037](https://github.com/NousResearch/hermes-agent/pull/7037)

#### 本地 Web Dashboard
- 瀏覽器介面：設定、session 監控、skills 管理、gateway 管理
- 適合不習慣 CLI 的用戶快速上手
- PR：[#8756](https://github.com/NousResearch/hermes-agent/pull/8756)

#### Termux / Android 支援
- 安裝路徑適配、TUI 行動螢幕優化、語音後端支援
- `/image` 指令可在 Android 上運作
- PR：[#6834](https://github.com/NousResearch/hermes-agent/pull/6834)

#### 背景程序監控（`watch_patterns`）
- 設定 pattern 監控背景 process 輸出，有匹配時即時通知
- 用途：監控錯誤、等待特定事件（"listening on port"）、觀察 build log
- PR：[#7635](https://github.com/NousResearch/hermes-agent/pull/7635)

### 🏗️ 架構與 Provider

| 功能 | 說明 |
|------|------|
| **xAI（Grok）原生 provider** | 直接 API 存取 + model catalog |
| **Xiaomi MiMo 一等 provider** | setup wizard、model catalog、空回應恢復 |
| **Qwen OAuth provider** | 含 portal request 支援 |
| **Pluggable Context Engine** | 透過 `hermes plugins` 插入自訂 context 管理引擎 |
| **結構化 API 錯誤分類** | 用於智慧 failover 決策 |
| **Credential exhaustion TTL** | 從 24 小時縮短為 1 小時 |
| **OpenRouter variant tags** | `:free`, `:extended`, `:fast` 在 model switch 時保留 |

### 📱 Gateway 平台改進

| 平台 | 主要變更 |
|------|------|
| **Discord** | allowed_channels whitelist、forum channel topic inheritance、DISCORD_REPLY_TO_MODE |
| **Slack** | 7 個社群 PR 合併、assistant thread lifecycle 處理 |
| **Matrix** | 從 matrix-nio 遷移至 mautrix-python、SQLite crypto store（E2EE 修復）|
| **Gateway Core** | 統一 proxy 支援（SOCKS + 系統 proxy）、inbound text batching、WSL 感知 |

### 🖥️ CLI / UX

| 功能 | 說明 |
|------|------|
| **`hermes backup` / `hermes import`** | 完整設定備份與還原，含 sessions、skills、memory |
| **`hermes dump`** | 可貼上的設定摘要，方便 debug 分享 |
| **`/debug` + `hermes debug share`** | 診斷工具集 + 上傳 debug 報告 |
| **`/compress <focus>`** | 帶 focus topic 的引導式 context 壓縮 |
| **Native `/model` picker modal** | CLI TUI 中的 provider → model 選擇彈窗 |
| **隨機 tips** | 新 session 啟動時顯示（279 則，CLI + gateway）|
| **`network.force_ipv4`** | 修復 IPv6 逾時問題的設定選項 |

### 🔧 工具系統

| 功能 | 說明 |
|------|------|
| **統一 spawn-per-call 執行層** | 環境執行的統一架構 |
| **統一 file sync** | mtime tracking、刪除、transactional state |
| **MCP `hermes mcp add --env/--preset`** | 新增環境變數和 preset 支援 |
| **Voxtral TTS** | Mistral AI TTS provider |
| **TTS 速度控制** | Edge TTS、OpenAI TTS、MiniMax 支援速度調整 |
| **Vision 自動調整大小** | 超大圖片自動縮放，上限提升至 20 MB |

### 🔒 安全加固（深度）

| 弱點 | 修復 |
|------|------|
| **SMS RCE** | Twilio webhook signature 驗證 |
| **Shell injection** | `_write_to_sandbox` 路徑加引號 |
| **Git argument injection** | checkpoint manager 路徑穿越防護 |
| **SSRF** | Slack 圖片上傳的 redirect bypass 防護 |
| **路徑穿越** | skill manager operations 邊界強制執行 |
| **Approval button 授權** | session 繼續需要認證 |
| **API bind guard** | 非 loopback 綁定強制要求 `API_SERVER_KEY` |

### 🧩 Skills 生態

- **集中式 skills 索引 + tree cache**：消除安裝時的 rate-limit 失敗
- **Creative divergence strategies** skill（@SHL0MS）
- **Creative ideation** skill（@SHL0MS）
- Google Workspace skill 遷移至 GWS CLI backend

### 🐛 重要 Bug 修復

- `/model` switch 現在跨 gateway 訊息持久保存
- 壓縮使用 live session model 而非 stale 設定
- Matrix E2EE 解密修復（SQLite crypto store）
- 修復 file path 中 macOS `/private/etc` symlink bypass
- 修復 Copilot GHE token poisoning（`GITHUB_TOKEN` 衝突）
- 修復 ASCII locale `UnicodeEncodeError` 覆蓋完整 request payload
- 修復 `.env` loading 導致 token 重複的問題

---

## v0.8.0 (2026.4.8) — Intelligence Release（基準版本）

**Release commit:** `86960cd`
**Since v0.7.x:** 209 merged PRs · 82 resolved issues

> 原生 Google AI Studio、即時 model 切換、自我優化 GPT/Codex 工具呼叫、惰性超時、approval buttons、互動式 model picker、MCP OAuth 2.1。

### 主要功能（快速摘要）

| 功能 | 說明 |
|------|------|
| **Google AI Studio（Gemini）原生 provider** | 含 models.dev 自動 context length 偵測 |
| **Live Model Switching（`/model`）** | 跨 CLI + 所有 gateway 平台即時切換，含 inline picker |
| **自我優化 GPT/Codex** | 自動診斷 5 個 tool-call 失敗模式並修補 |
| **Inactivity-based timeouts** | 追蹤實際 tool 活動，不殺掉仍在工作的 agent |
| **Approval buttons** | Slack 和 Telegram 的原生平台按鈕批准危險指令 |
| **MCP OAuth 2.1 PKCE + OSV 掃描** | 標準相容的 MCP 認證 + MCP 擴充套件惡意程式掃描 |
| **集中式日誌** | `~/.hermes/logs/` + `hermes logs` 指令 |
| **Plugin System 擴展** | CLI subcommand 註冊、request-scoped hooks、session lifecycle hooks |
| **Matrix Tier 1** | reactions、read receipts、rich formatting、room management |
| **安全加固** | SSRF、timing attack、tar traversal、credential leakage、cron path traversal |

詳細內容請見：`git show 86960cd:RELEASE_v0.8.0.md`

---

## 版本對照速查

| 版本 | 日期 | Commit | 主題 | 平台數 |
|------|------|--------|------|--------|
| v0.8.0 | 2026-04-08 | `86960cd` | Intelligence Release | 14 |
| v0.9.0 | 2026-04-13 | `1af2e18` | Everywhere Release | 16 |
| v0.10.0 | 2026-04-16 | `1dd6b5d` | Tool Gateway Release | 16 |

## 功能累積一覽（v0.8 → v0.10）

| 功能類別 | v0.8.0 | v0.9.0 新增 | v0.10.0 新增 |
|---------|--------|-------------|--------------|
| 訊息平台 | 13 | +3（iMessage、WeChat、WeCom）| — |
| Mobile 支援 | ✗ | Termux/Android | — |
| Web Dashboard | ✗ | ✓ | 主題系統、插件系統 |
| Fast Mode | ✗ | ✓（`/fast`）| — |
| Backup/Restore | ✗ | `hermes backup/import` | — |
| Tool Gateway | ✗ | ✗ | ✓（Portal 訂閱）|
| xAI/Grok | 部分 | 原生 provider | Responses API + TTS |
| Ollama Cloud | ✗ | ✗ | ✓ 內建 provider |
| Claude Opus 4.7 | ✗ | ✗ | ✓ 完整支援 |
| Background monitoring | ✗ | `watch_patterns` | — |
| Pluggable Context Engine | ✗ | ✓ | — |
| Quiet mode | ✗ | ✗ | `-Q` flag |
| Dashboard themes | ✗ | ✗ | ✓ |
| Dashboard plugins | ✗ | ✗ | ✓ |
