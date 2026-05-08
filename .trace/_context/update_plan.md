# 更新計畫

> 產出日期：2026-05-08
> Base Commit 範圍：`601e5f1d`..`faa13e49`
> 版本：v0.12.0 → v0.13.0

## 變更摘要
- Base Commit 範圍：601e5f1d..faa13e49
- 變更檔案數：527（不含 .trace/）
- 總檔案數：3009（初次 trace 統計）+ 新增 ~200 檔案
- 變更幅度：~17%（中度更新，繼續增量更新）

---

## 受影響文件與更新策略

### CODEBASE_MAP.md — 需要更新
- **原因**：4 個新頂層目錄（`providers/`、`locales/`、`plugins/model-providers/`、`plugins/platforms/google_chat/`），核心模組（`hermes_cli/checkpoints.py`, `kanban_specify.py`, `kanban_diagnostics.py`）新增
- **影響段落**：目錄結構樹、「我要改 X 看哪裡」速查表
- **更新策略**：Main Agent 直接修改（局部新增段落）
- **需讀取的 context**：changelog.md

### INDEX.md — 需要更新
- **原因**：版本從 v0.12.0 → v0.13.0，新技術（i18n），術語表需新增 ProviderProfile / Checkpoint / /goal
- **影響段落**：一句話總結（版本）、技術棧表格（新增 i18n 相關）、術語表（3 個新術語）
- **更新策略**：Main Agent 直接修改
- **需讀取的 context**：changelog.md

### ARCHITECTURE.md — 需要更新（重要）
- **原因**：最大的架構變更：
  1. `ProviderProfile` 插件系統（新的第三個插件類型，原本只有 MemoryProvider + Plugin）
  2. 第 20 個平台：Google Chat（純插件型，非 built-in）
  3. 新的平台插件 hook：`env_enablement_fn` + `cron_deliver_env_var`
  4. 新的 lifecycle hook：`transform_llm_output`
  5. Checkpoints v2 整合到 agent
  6. `StreamingThinkScrubber` 加入 streaming pipeline
  7. Sessions 重啟後自動恢復
  8. i18n 層
- **影響段落**：元件清單（新增 providers/）、Extension Points 段落（新增 ProviderProfile）、Plugin hooks 清單（新增 transform_llm_output）、Gateway 序列圖、平台清單（+Google Chat）
- **更新策略**：Sub-Agent（多個段落需改寫 + Mermaid 元件圖更新）
- **需讀取的 context**：changelog.md、現有 ARCHITECTURE.md

### DATA_MODEL.md — 不需要更新
- **原因**：SQLite schema 無變更，session/message 資料模型不受影響。`X-Hermes-Session-Key` header 是 API layer 的事，DATA_MODEL 不需要記錄。

### API_SURFACE_part1.md — 需要更新
- **原因**：
  1. CLI 新增子指令（`hermes curator archive/prune/list-archived`）
  2. `AIAgent.__init__` 新增 checkpoint 參數（`checkpoints_enabled` 等 4 個）
  3. 斜線指令更新（`/goal`、`/steer`、`/queue`、`/curator` 等）
- **影響段落**：CLI 子指令表格、`AIAgent.__init__` 參數列表、斜線指令表格
- **更新策略**：Main Agent 直接修改
- **需讀取的 context**：changelog.md

### API_SURFACE_part2.md — 需要更新
- **原因**：
  1. 新工具 `video_analyze`（`tools/vision_tools.py`）
  2. API server 新 header `X-Hermes-Session-Key`
  3. 新插件 hook：`transform_llm_output`（在 error handling / plugin 文件中）
- **影響段落**：工具 API 表格（vision toolset 新增 video_analyze）、Gateway API（新 header）
- **更新策略**：Main Agent 直接修改
- **需讀取的 context**：changelog.md

### DEV_GUIDE.md — 需要更新
- **原因**：`AGENTS.md` 有重要更新：
  1. model-provider plugin 新章節（完整的開發指引）
  2. toolsets.py 必要步驟強調（新的 Known Pitfall 等級）
  3. config.yaml top-level sections 列表新增
  4. 測試規模更新（~17k tests/~900 files）
  5. `plugins/` 目錄描述更新（model-providers, kanban, observability 等）
- **影響段落**：Known Pitfalls（新增 toolset 必要步驟）、新增開發流程章節（新增 model-provider plugin 步驟）、設定管理段落（新增 config sections）
- **更新策略**：Main Agent 直接修改
- **需讀取的 context**：changelog.md、AGENTS.md diff

### DISCOVERY_LOG.md — 需要更新
- **原因**：追加本次更新的發現（架構決策記錄、新的待驗證項目、已關閉的問題）
- **影響段落**：追加新段落（不修改既有內容）
- **更新策略**：Main Agent 直接追加

---

## 執行順序
1. _context/changelog.md（✅ 已完成）
2. _context/update_plan.md（✅ 本文件）
3. CODEBASE_MAP.md（Main Agent）
4. INDEX.md（Main Agent）
5. ARCHITECTURE.md（Sub-Agent）
6. API_SURFACE_part1.md（Main Agent）
7. API_SURFACE_part2.md（Main Agent）
8. DEV_GUIDE.md（Main Agent）
9. DISCOVERY_LOG.md（Main Agent，追加）
10. TRACE_META.md 更新
