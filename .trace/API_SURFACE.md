# API_SURFACE.md — Hermes Agent API 與介面參考文件

> 版本：0.12.0 | 維護者：Nous Research | 授權：MIT

本文件因超過 500 行上限，已拆分為兩個部分：

- **[API_SURFACE_part1.md](./API_SURFACE_part1.md)**（304 行）
  - CLI 指令參考（hermes 子指令清單、旗標）
  - 斜線指令（/new、/model、/skills 等）— 彙整自 COMMAND_REGISTRY
  - Python Library API（AIAgent.__init__、AIAgent.chat()、AIAgent.run_conversation()）

- **[API_SURFACE_part2.md](./API_SURFACE_part2.md)**（343 行）
  - Gateway OpenAI 相容 API（api_server.py 端點清單）
  - 工具 API 概覽（tools/registry.py 中約 68 個工具，按分類整理為表格）
  - Authentication 模型（API keys、OAuth、DM pairing、Bearer token）
  - Error Handling Pattern（工具失敗 JSON 格式、LLM API 失敗策略）
  - API 介面層 Mermaid 示意圖
