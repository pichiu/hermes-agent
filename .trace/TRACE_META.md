# Trace Metadata

## 分支資訊
- **Base Branch**: main
- **Trace Branch**: trace/docs

## 最後 Trace 資訊
- **Base Commit Hash**: faa13e49f81480771ceeb55991bb0c27edf1a5fb
- **日期**: 2026-05-08
- **Trace 類型**: incremental
- **涵蓋範圍**: 全部（API server + CLI + Messaging Gateway，v0.12→v0.13 增量）

## 文件清單

| 文件 | 對應 Base Commit | 最後更新日期 |
|------|-----------------|-------------|
| INDEX.md | faa13e4 | 2026-05-08 |
| ARCHITECTURE.md | faa13e4 | 2026-05-08 |
| DATA_MODEL.md | 601e5f1 | 2026-05-05 |
| API_SURFACE.md | faa13e4 | 2026-05-08 |
| API_SURFACE_part1.md | faa13e4 | 2026-05-08 |
| API_SURFACE_part2.md | faa13e4 | 2026-05-08 |
| DEV_GUIDE.md | faa13e4 | 2026-05-08 |
| CODEBASE_MAP.md | faa13e4 | 2026-05-08 |
| DISCOVERY_LOG.md | faa13e4 | 2026-05-08 |

## 中繼 Context 檔案（_context/）

| 檔案 | 內容 |
|------|------|
| _context/recon.md | 技術棧、目錄結構、既有文件摘要、落差分析（v0.12 初次 trace）|
| _context/web_findings.md | 線上搜尋結果摘要與連結（v0.12 初次 trace）|
| _context/entry_points.md | 所有啟動進入點與初始化序列（v0.12 初次 trace）|
| _context/data_flow.md | 代表性 use case 完整資料流（v0.12 初次 trace）|
| _context/core_logic.md | 核心領域邏輯（自我進化、Prompt Caching、工具分派）|
| _context/extensions.md | 所有擴充點（Plugin、Memory Provider、Platform Adapter）|
| _context/integrations.md | 外部整合（LLM 提供商、Web 工具、Messaging 平台）|
| _context/configuration.md | 設定層次、主要設定類別、環境變數 |
| _context/changelog.md | v0.12→v0.13 變更記錄（2026-05-08 增量更新）|
| _context/update_plan.md | v0.12→v0.13 更新計畫（2026-05-08 增量更新）|

## 變更歷程

| 日期 | 類型 | Base Commit 範圍 | 更新的文件 | 摘要 |
|------|------|-----------------|-----------|------|
| 2026-05-05 | full | initial..601e5f1 | 全部 | 初次 trace（v0.12.0）|
| 2026-05-08 | incremental | 601e5f1..faa13e4 | INDEX, ARCHITECTURE, CODEBASE_MAP, API_SURFACE×3, DEV_GUIDE, DISCOVERY_LOG | v0.13.0 升版：providers 插件化、Google Chat、i18n、Checkpoints v2、/goal、security wave |
