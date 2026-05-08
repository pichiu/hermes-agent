# Changelog — 601e5f1..faa13e49

> 產出日期：2026-05-08
> Base Commit 範圍：`601e5f1d57cfd4ceefee50a6df05a860a1a602e8`..`faa13e49f81480771ceeb55991bb0c27edf1a5fb`
> Commit 數：6567（非 merge）| 檔案變更：527 | 新增：63,202 行 / 刪除：4,784 行
> 版本：v0.12.0 → v0.13.0（"The Tenacity Release"，發布日 2026-05-07）

---

## 重大功能亮點（摘自 RELEASE_v0.13.0.md）

### 架構性變更
1. **Providers 插件化**（`providers/` + `plugins/model-providers/`）
   - 新增 `providers/base.py`：`ProviderProfile` 宣告式 ABC，描述 inference provider 的 auth、endpoint、quirks
   - 新增 `plugins/model-providers/`：29 個 LLM 提供商獨立為插件（anthropic, openai-codex, gemini, bedrock, deepseek, ollama-cloud, openrouter, nvidia, qwen-oauth, arcee, azure-foundry, etc.）
   - Discovery 系統：`providers/__init__._discover_providers()` 懶惰掃描，掃描順序：bundled → user plugins → legacy `providers/<name>.py`
   - **不與 PluginManager 共用**：PluginManager 記錄 `kind: model-provider` manifest 但不 import（避免雙重實例化）

2. **平台插件 hook 機制**（`gateway/platform_registry.py`）
   - 新 hook：`env_enablement_fn`、`cron_deliver_env_var`
   - IRC 和 Teams 已遷移至此機制
   - Google Chat 作為第一個純插件平台（`plugins/platforms/google_chat/`），成為第 20 個訊息平台

3. **`transform_llm_output` 插件 hook**
   - 新增 lifecycle hook，讓 plugin 在 LLM 輸出進入對話前改寫或過濾
   - 用途：context window reducer、content filter

### 核心 Agent 新功能
4. **Checkpoints v2**（`hermes_cli/checkpoints.py` + `tools/checkpoint_manager.py`）
   - 真實剪枝（real pruning）、磁碟配額保護
   - `AIAgent.__init__` 新參數：`checkpoints_enabled`, `checkpoint_max_snapshots`, `checkpoint_max_total_size_mb`, `checkpoint_max_file_size_mb`

5. **`StreamingThinkScrubber`**（`agent/think_scrubber.py`）
   - 即時剝除 `<think>` tag streaming output
   - 整合於 `run_agent.py` L131、L1319、L6816、L6912

6. **`/goal` 指令**（Ralph loop）
   - 鎖定 agent 在多輪對話中持續追蹤單一目標
   - 自動暫停（auto-pause when judge model returns unparseable output）

7. **Sessions 重啟後自動恢復**
   - Gateway bounce、`/update` 重啟、原始碼熱重載後，對話自動 resume

8. **Post-write delta lint**
   - `write_file` + `patch` 工具呼叫後，自動對 Python/JSON/YAML/TOML 執行語法檢查

9. **新工具：`video_analyze`**（`tools/vision_tools.py`）
   - Gemini 及相容多模態模型的影片分析工具

### 國際化
10. **i18n 支援**（`locales/` + `agent/i18n.py`）
    - 8 個 locale 檔案（en/zh/ja/de/es/fr/tr/uk）
    - 僅翻譯 user-facing static messages（approval prompt、gateway slash command 回覆）
    - Agent 生成的輸出、log、tool output 保持英文

### Gateway 新功能
11. **平台 allowlist**：`allowed_channels`/`allowed_chats`/`allowed_rooms` 跨 Slack、Telegram、Mattermost、Matrix、DingTalk
12. **QQBot 增強**：chunked upload、inline keyboard approvals、quoted attachment
13. **API server**：新 header `X-Hermes-Session-Key` 給 memory provider 穩定 session ID

### Kanban 增強
14. **`hermes_cli/kanban_specify.py`**：`specify` 子指令，使用 auxiliary LLM 細化 kanban 任務
15. **`hermes_cli/kanban_diagnostics.py`**：任務困境信號的通用診斷引擎

### Web 搜尋後端
16. **Brave Search（免費層）**、**DDGS**、**SearXNG** 新後端
17. **Web 工具拆分**：search vs extract vs browse 可選不同後端

### ACP 增強（VS Code/Zed/JetBrains）
18. **`/steer`**：直接指揮進行中的 agent
19. **`/queue`**：排隊後續任務
20. **圖片附件傳遞**：`feat(acp): pass image file attachments through as image_url parts`

### 安全性修復（8 個 P0 closures）
21. Redaction 預設開啟
22. Discord role-allowlist guild-scoped（CVSS 8.1 cross-guild DM bypass 已關閉）
23. WhatsApp 預設拒絕陌生人
24. TOCTOU windows 已關閉（auth.json + MCP OAuth）
25. Cron prompt-injection 掃描已組合的 skill content
26. `hermes debug share` 上傳前先 redact

### MCP 增強
27. SSE transport + OAuth forwarding
28. Stale-pipe retries
29. Image results 以 MEDIA tag 回傳（不再丟棄）

### 新 optional skills（6 個）
- `optional-skills/finance/` × 6（3-statement-model, comps-analysis, dcf-model, excel-author, lbo-model, merger-model, pptx-author）
- `optional-skills/productivity/shop-app/`
- `optional-skills/research/searxng-search/`

### 測試規模
- pytest suite：~15k tests/700 files（v0.12.0）→ ~17k tests/900 files（v0.13.0）

---

## 分類後的檔案變更清單

### 結構性變更
- `A providers/__init__.py`, `providers/base.py`, `providers/README.md`
- `A plugins/model-providers/` × 29 個提供商
- `A plugins/platforms/google_chat/` × 4 個檔案
- `A locales/` × 8 個 locale 檔案
- `A optional-skills/finance/` × 12 個檔案
- `A optional-skills/research/searxng-search/`
- `A optional-skills/productivity/shop-app/`
- `A hermes_cli/checkpoints.py`
- `A hermes_cli/kanban_specify.py`, `kanban_diagnostics.py`
- `A gateway/platforms/qqbot/chunked_upload.py`, `keyboards.py`
- `A scripts/lint_diff.py`, `scripts/setup_open_webui.sh`
- `A README.zh-CN.md`, `RELEASE_v0.13.0.md`
- `A .github/workflows/lint.yml`
- `A agent/i18n.py`, `agent/think_scrubber.py`

### 核心邏輯修改
- `M run_agent.py`（checkpoints, think_scrubber, video_analyze, /goal）
- `M cli.py`
- `M agent/` × 14 個檔案（anthropic_adapter, auxiliary_client, bedrock_adapter, context_compressor, credential_pool, display, error_classifier, image_routing, memory_manager, memory_provider, model_metadata, models_dev, prompt_builder, redact, usage_pricing）
- `M hermes_cli/` × 18 個檔案

### API / Gateway
- `M gateway/platforms/api_server.py`
- `M gateway/platforms/` × 14 個平台
- `M gateway/run.py`, `gateway/config.py`, `gateway/platform_registry.py`
- `M hermes_cli/commands.py`（新增 `/curator`、更新 `/goal`）

### 設定 / 建置
- `M AGENTS.md`（新增 model-provider plugin 說明、toolset 強調、config sections 列表）
- `M pyproject.toml`
- `M Dockerfile`, `docker-compose.yml`
- `M setup-hermes.sh`, `scripts/install.sh`
- `M .env.example`, `cli-config.yaml.example`
