# API_SURFACE — Part 1/2：CLI、斜線指令、Python Library API

> 版本：0.13.0 | 維護者：Nous Research | 授權：MIT
> 接續：[API_SURFACE_part2.md](./API_SURFACE_part2.md)（Gateway API、工具 API、Authentication、Error Handling）

---

## 目錄

1. [CLI 指令參考](#1-cli-指令參考)
2. [斜線指令（Slash Commands）](#2-斜線指令slash-commands)
3. [Python Library API](#3-python-library-api)

---

## 1. CLI 指令參考

進入點：`hermes` 腳本 → `hermes_cli/main.py:main()`

### 1.1 全域旗標

| 旗標 | 說明 |
|------|------|
| `--profile <name>` | 選擇 profile（覆寫 `HERMES_HOME`） |
| `--model <model>` | 指定模型（可覆寫 config.yaml） |
| `--version` | 顯示版本並退出 |

### 1.2 頂層子指令清單

| 子指令 | 說明 |
|--------|------|
| `hermes`（無參數） | 啟動互動式 CLI（等同 `hermes chat`） |
| `hermes chat` | 啟動互動式對話 CLI |
| `hermes model` | 管理預設模型設定 |
| `hermes fallback` | 設定 fallback model chain（add / remove / list / test） |
| `hermes gateway` | 管理訊息閘道（run / start / stop / restart / status / install / uninstall / setup） |
| `hermes setup` | 首次安裝精靈（API keys、模型設定） |
| `hermes whatsapp` | WhatsApp 橋接設定 |
| `hermes slack` | Slack App 安裝精靈（含 manifest 生成） |
| `hermes login` | OAuth / API key 登入 |
| `hermes logout` | 清除登入憑證 |
| `hermes auth` | 憑證池管理（add / list / remove / reset / status / logout / spotify） |
| `hermes status` | 顯示系統狀態 |
| `hermes cron` | 排程任務管理（list / create / edit / pause / resume / run / remove / status / tick） |
| `hermes webhook` | Webhook 管理（subscribe / list / remove / test） |
| `hermes hooks` | 閘道 hook 管理（list / test / revoke） |
| `hermes doctor` | 環境診斷 |
| `hermes dump` | 匯出 agent 狀態 |
| `hermes debug` | 上傳除錯報告並取得分享連結 |
| `hermes backup` | 備份 Hermes 設定與狀態 |
| `hermes import` | 匯入備份 |
| `hermes config` | 設定管理（show / edit / set / path / env-path / check / migrate） |
| `hermes pairing` | DM pairing 管理 |
| `hermes skills` | 技能管理（browse / search / install / inspect / list / check / update / audit / uninstall / reset / publish / snapshot） |
| `hermes mcp` | MCP 伺服器管理 |
| `hermes sessions` | 會話管理（list / export / delete / prune / stats / rename / browse） |
| `hermes insights` | 用量分析與統計 |
| `hermes claw` | Claw 工具（migrate / cleanup） |
| `hermes version` | 顯示版本資訊 |
| `hermes update` | 更新 Hermes Agent 至最新版本 |
| `hermes uninstall` | 卸載 Hermes |
| `hermes acp` | 啟動 ACP 適配器（VS Code / Zed / JetBrains） |
| `hermes profile` | Profile 管理（list / use / create / delete / show / alias / rename / export / import） |
| `hermes completion` | Shell 自動補全安裝 |
| `hermes dashboard` | 啟動 Web 儀表板（fastapi + uvicorn） |
| `hermes logs` | 查看 agent 日誌 |

### 1.3 hermes chat 常用旗標

```
hermes [chat] [QUERY]
  --model MODEL          覆寫本次對話模型
  --session SESSION_ID   續接指定會話
  --toolset TOOLSET      限定啟用的 toolset（逗號分隔）
  --no-tools             停用所有工具
  --quiet                靜音模式（無進度輸出）
  --save-trajectories    儲存對話軌跡（JSONL 格式）
```

---

## 2. 斜線指令（Slash Commands）

在互動式 CLI 中，以 `/` 開頭輸入斜線指令。定義來源：`hermes_cli/commands.py:COMMAND_REGISTRY`。

### 2.1 Session 管理

| 指令 | 說明 |
|------|------|
| `/new` | 開始新會話（新 session ID + 清空歷史） |
| `/clear` | 清除畫面並開始新會話 |
| `/redraw` | 強制重繪 UI（修復終端機漂移） |
| `/history` | 顯示對話歷史 |
| `/save` | 儲存當前對話 |
| `/retry` | 重試最後一則訊息 |
| `/undo` | 移除最後一組 user/assistant 交換 |
| `/title <標題>` | 設定當前會話標題 |
| `/branch` | 分支當前會話（探索不同路徑） |
| `/compress` | 手動壓縮對話 context |
| `/rollback` | 列出或還原檔案系統 checkpoints |
| `/snapshot` | 建立或還原 Hermes 設定/狀態快照 |
| `/stop` | 終止所有背景行程 |
| `/approve` | 核准待審的危險指令 |
| `/deny` | 拒絕待審的危險指令 |
| `/background <prompt>` | 在背景執行 prompt |
| `/agents` | 顯示活躍 agent 與執行中任務 |
| `/queue <prompt>` | 排入下一輪 prompt（不中斷當前） |
| `/steer <message>` | 在下次工具呼叫後注入訊息 |
| `/goal <目標>` | 設定跨回合的持續目標 |
| `/status` | 顯示會話資訊 |
| `/sethome` | 設定此聊天為 home channel |
| `/resume <name>` | 續接先前命名的會話 |
| `/restart` | 優雅重啟 gateway（排完執行中任務） |

### 2.2 Configuration

| 指令 | 說明 |
|------|------|
| `/config` | 顯示當前設定 |
| `/model [model-name]` | 切換本次會話模型 |
| `/gquota` | 顯示 Google Gemini Code Assist 配額用量 |
| `/personality <name>` | 設定預定義人格 |
| `/statusbar` | 切換 context/model 狀態列 |
| `/verbose` | 循環切換工具進度顯示：off → new → all → verbose |
| `/footer` | 切換 gateway 執行期 metadata footer |
| `/yolo` | 切換 YOLO 模式（跳過所有危險指令審核） |
| `/reasoning` | 管理 reasoning effort 與顯示 |
| `/fast` | 切換 fast mode（OpenAI Priority / Anthropic Fast） |
| `/skin [theme]` | 顯示或切換 UI 主題 |
| `/indicator` | 選擇 TUI busy indicator 樣式 |
| `/voice` | 切換語音模式 |
| `/busy` | 控制 Hermes 執行中時 Enter 的行為 |

### 2.3 Tools & Skills

| 指令 | 說明 |
|------|------|
| `/tools [list\|disable\|enable] [name...]` | 管理工具啟用狀態 |
| `/toolsets` | 列出可用 toolsets |
| `/skills [search\|install\|inspect\|...]` | 搜尋、安裝、管理技能 |
| `/cron` | 管理排程任務 |
| `/curator` | 背景技能維護（status / run / pin / archive） |
| `/kanban` | 多 profile 協作看板（tasks / links / comments） |
| `/reload` | 重新載入 .env 變數至執行中會話 |
| `/reload-mcp` | 從 config 重新載入 MCP 伺服器 |
| `/reload-skills` | 重新掃描 ~/.hermes/skills/ |
| `/browser` | 透過 CDP 連接瀏覽器工具 |
| `/plugins` | 列出已安裝插件及狀態 |

### 2.4 Info

| 指令 | 說明 |
|------|------|
| `/commands` | 瀏覽所有指令與技能（分頁） |
| `/help` | 顯示可用指令 |
| `/profile` | 顯示當前 profile 名稱與 home 目錄 |
| `/usage` | 顯示當前會話的 token 用量與 rate limit |
| `/insights` | 顯示用量洞察與分析 |
| `/platforms` | 顯示 gateway/messaging 平台狀態 |
| `/copy` | 複製最後一則 assistant 回應至剪貼簿 |
| `/paste` | 附加剪貼簿圖片至下一則 prompt |
| `/image <path>` | 附加本地圖片至下一則 prompt |
| `/update` | 更新 Hermes Agent 至最新版本 |
| `/debug` | 上傳除錯報告 |
| `/quit` 或 `/exit` | 退出 CLI |

---

## 3. Python Library API

來源：`run_agent.py:AIAgent`

### 3.1 AIAgent.__init__

```python
from run_agent import AIAgent

agent = AIAgent(
    base_url: str = None,          # LLM API endpoint（選填，預設 openrouter）
    api_key: str = None,           # API key（選填，從 env 讀取）
    provider: str = None,          # 提供商識別：'anthropic'、'openrouter'、'nous' 等
    api_mode: str = None,          # 'chat_completions' | 'anthropic_messages' | 'codex_responses'
    model: str = "",               # 模型名稱（OpenRouter 格式：provider/model）
    max_iterations: int = 90,      # 最大工具呼叫迭代次數
    tool_delay: float = 1.0,       # 工具呼叫間延遲（秒）
    enabled_toolsets: List[str] = None,   # 僅啟用指定 toolsets
    disabled_toolsets: List[str] = None,  # 停用指定 toolsets
    save_trajectories: bool = False,      # 是否儲存對話軌跡（JSONL）
    verbose_logging: bool = False,
    quiet_mode: bool = False,
    platform: str = None,          # 平台識別（'cli'、'telegram' 等）
    session_id: str = None,        # 會話 ID（選填，自動生成）
    session_db=None,               # 共享 SessionDB 實例
    # Callback hooks
    tool_progress_callback: callable = None,
    tool_start_callback: callable = None,
    tool_complete_callback: callable = None,
    thinking_callback: callable = None,
    reasoning_callback: callable = None,
    clarify_callback: callable = None,
    step_callback: callable = None,
    stream_delta_callback: callable = None,
    # 進階設定
    max_tokens: int = None,
    reasoning_config: Dict[str, Any] = None,
    request_overrides: Dict[str, Any] = None,
    prefill_messages: List[Dict[str, Any]] = None,
    fallback_model: Dict[str, Any] = None,
    # [v0.13] Checkpoints v2
    checkpoints_enabled: bool = False,
    checkpoint_max_snapshots: int = 20,
    checkpoint_max_total_size_mb: int = 500,
    checkpoint_max_file_size_mb: int = 10,
)
```

### 3.2 AIAgent.chat()

```python
def chat(self, message: str, stream_callback: Optional[callable] = None) -> str
```

**說明**：最簡單的聊天介面，傳回最終文字回應。

**來源**：`run_agent.py:13970`

**參數**：
- `message`：使用者訊息
- `stream_callback`：可選 callback，每個文字 delta 觸發（用於 TTS pipeline）

**回傳**：`str`（`final_response` 字串）

**範例**：
```python
agent = AIAgent(model="anthropic/claude-sonnet-4-6")
response = agent.chat("幫我寫一個 Python hello world")
print(response)
```

### 3.3 AIAgent.run_conversation()

```python
def run_conversation(
    self,
    user_message: str,
    system_message: str = None,
    conversation_history: List[Dict[str, Any]] = None,
    task_id: str = None,
    stream_callback: Optional[callable] = None,
    persist_user_message: Optional[str] = None,
) -> Dict[str, Any]
```

**說明**：完整對話執行，含工具呼叫迴圈，直至完成。

**來源**：`run_agent.py:10432`

**參數**：
- `user_message`：使用者訊息
- `system_message`：自訂 system message（覆寫 `ephemeral_system_prompt`）
- `conversation_history`：先前對話訊息列表（選填）
- `task_id`：任務唯一識別碼（並發隔離用，選填，自動生成）
- `stream_callback`：文字串流 callback（用於 TTS pipeline 提前開始音訊生成）
- `persist_user_message`：儲存至歷史的清理版訊息（當 `user_message` 含 API 合成前綴時使用）

**回傳**：`Dict[str, Any]`，包含：
- `"final_response"`：最終回應文字
- 完整訊息歷史與工具呼叫記錄

**範例**：
```python
result = agent.run_conversation(
    "分析這份 CSV 並畫圖",
    conversation_history=previous_messages,
)
print(result["final_response"])
```

### 3.4 run_agent.py:main()（CLI 直接執行模式）

來源：`run_agent.py:13985`，使用 Python Fire CLI。

```python
def main(
    query: str = None,
    model: str = "",
    api_key: str = None,
    base_url: str = "",
    max_turns: int = 10,
    enabled_toolsets: str = None,   # 逗號分隔字串
    disabled_toolsets: str = None,
    list_tools: bool = False,
    save_trajectories: bool = False,
    save_sample: bool = False,
    verbose: bool = False,
    log_prefix_chars: int = 20,
)
```

直接執行範例：
```bash
python run_agent.py --query "查詢最新 Python 新聞" --model anthropic/claude-sonnet-4-6
```

---

*接續 [API_SURFACE_part2.md](./API_SURFACE_part2.md)*
*文件生成日期：2026-05-05*
