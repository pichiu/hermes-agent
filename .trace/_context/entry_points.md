# Stage 2.1 Entry Points

## 主要啟動路徑

### 1. CLI 互動模式

```
hermes（shell 腳本）
  └─> hermes_cli/main.py::main()
        └─> cli.py（主 CLI REPL）
              └─> AIAgent.__init__() [run_agent.py:320]
                    └─> agent/agent_init.py::init_agent()
                          └─> run_conversation() [agent/conversation_loop.py:364]
```

**CLI 入口**：`hermes`（根目錄的 shell 腳本）→ `hermes_cli/main.py::main()` → 根據子命令路由

**互動 REPL**：`cli.py`（~737KB）是完整的 TUI REPL，包含 prompt_toolkit 多行編輯、slash-command autocomplete、串流輸出顯示

### 2. Gateway 模式

```
hermes gateway start
  └─> hermes_cli/main.py（gateway 子命令）
        └─> gateway/run.py（長駐 gateway 進程）
              └─> 各平台 adapter（telegram.py, slack.py, 等）
                    └─> _handle_message() → AIAgent.run_conversation()
```

**Gateway**：`gateway/run.py` 是單一長駐進程，統一接收 20+ messaging 平台的訊息，透過共用 session 路由到 AIAgent。

### 3. ACP Server 模式（編輯器整合）

```
hermes acp
  └─> acp_adapter/（ACP 協議處理）
        └─> FastAPI/Uvicorn 伺服器
              └─> AIAgent.run_conversation()
```

### 4. MCP Server 模式

```
hermes mcp（或直接呼叫）
  └─> mcp_serve.py
        └─> FastAPI + MCP 協議
              └─> 暴露 Hermes 工具給外部 MCP client
```

### 5. 批次/研究模式

```
python batch_runner.py
  └─> AIAgent（批次實例化）
        └─> 並行 run_conversation() 呼叫
              └─> trajectory 儲存
```

## AIAgent 初始化流程

`AIAgent.__init__()` 是一個薄包裝，實際工作在 `agent/agent_init.py::init_agent()`（~1400 行）：

1. **Provider 自動偵測**：從環境變數和 config.yaml 解析 `provider`、`base_url`、`api_key`
2. **OpenAI client 建立**：通過 `_create_openai_client()` 建立 OpenAI SDK 實例（所有 provider 均使用 OpenAI-compatible API）
3. **工具載入**：`model_tools.py` 呼叫 `tools/registry.py::discover_builtin_tools()` → 掃描 `tools/*.py` 中的 `registry.register()` 呼叫
4. **Toolset 過濾**：根據 `enabled_toolsets`/`disabled_toolsets` 過濾工具集合
5. **Memory 初始化**：`MemoryManager` + 可選的外部 memory provider（Honcho 等）
6. **Context Engine**：`agent/context_engine.py` 初始化（前綴 context 注入系統）
7. **系統提示詞建構**：`agent/system_prompt.py` 渲染 Jinja2 模板
8. **DB session**：`agent._ensure_db_session()` 建立 session 記錄
9. **Skill 索引**：FTS5 索引載入，供 skill 搜尋使用
10. **Plugin 載入**：`model_tools.py` 掃描並載入已啟用的 plugin

## 關鍵初始化常數

- `max_iterations = 90`：每個 turn 的最大工具呼叫迭代次數 (`run_agent.py:388`)
- `tool_delay = 1.0`：工具呼叫間的延遲秒數
- `MINIMUM_CONTEXT_LENGTH`：Ollama 最小 context 長度驗證（`agent/model_metadata.py`）

## 多入口公共路徑

無論哪個入口，最終都匯聚到：

```python
# agent/conversation_loop.py:364
def run_conversation(agent, user_message, ...) -> Dict[str, Any]:
    agent._ensure_db_session()
    # 設定 task_id, turn_id, session 上下文
    # 重置 retry 計數器
    # 呼叫 _run()（主要工具迴路）
```

`_run()` 是實際的 tool-calling loop（LLM 呼叫 → 工具執行 → 繼續/停止 判斷）。
