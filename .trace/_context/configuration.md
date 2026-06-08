# Stage 2.6 設定與環境

## 設定載入優先順序

```
環境變數（.env / 系統環境）
    ↓（覆蓋）
~/.hermes/config.yaml（主要設定檔）
    ↓（覆蓋）
程式碼預設值
```

**注意**：部分設定（如 HERMES_HOME、HERMES_PROFILE）**只能**從環境變數讀取，不能持久化到 config.yaml（`hermes_cli/config.py:197`）

## 關鍵路徑

| 路徑 | 用途 |
|------|------|
| `~/.hermes/` | `HERMES_HOME`，所有用戶資料的根目錄 |
| `~/.hermes/config.yaml` | 主要設定檔（model、toolsets、terminal、memory 等） |
| `~/.hermes/MEMORY.md` | 全局記憶文件 |
| `~/.hermes/USER.md` | 用戶模型文件 |
| `~/.hermes/SOUL.md` | Agent 人格/身份文件 |
| `~/.hermes/skills/` | Skills 目錄 |
| `~/.hermes/plugins/` | 用戶安裝的 plugins |
| `~/.hermes/sessions/` | Session transcripts 和 SQLite DB |
| `~/.hermes/.container-mode` | 容器模式 metadata（⚠️ 待驗證） |
| `~/.hermes/.curator_state` | Skill curator 排程狀態 |
| `~/.hermes/active_profile` | 當前 profile 名稱 |

**Windows 路徑**：`%LOCALAPPDATA%\hermes\`

## 環境變數（來源：`.env.example`）

### LLM Provider Keys

| 變數 | Provider |
|------|---------|
| `OPENROUTER_API_KEY` | OpenRouter（預設路由） |
| `ANTHROPIC_API_KEY` | Anthropic 直接 |
| `GOOGLE_API_KEY` / `GEMINI_API_KEY` | Google AI Studio |
| `OLLAMA_API_KEY` | Ollama Cloud |
| `NOVITA_API_KEY` | NovitaAI |
| `GLM_API_KEY` | z.ai/GLM |
| `KIMI_API_KEY` | Kimi/Moonshot |
| `MINIMAX_API_KEY` | MiniMax |
| `OPENCODE_ZEN_API_KEY` | OpenCode Zen |
| `XAI_API_KEY` | xAI |
| `ARCEEAI_API_KEY` | Arcee AI |
| `NOUS_PORTAL_TOKEN` | Nous Portal 訂閱 |

### Tool Provider Keys

| 變數 | 工具 |
|------|------|
| `FIRECRAWL_API_KEY` | Firecrawl 網路搜尋 |
| `EXA_API_KEY` | Exa 搜尋 |
| `FAL_KEY` | FAL 圖像生成 |
| `ELEVEN_LABS_API_KEY` | ElevenLabs TTS |
| `TELEGRAM_BOT_TOKEN` | Telegram gateway |
| `DISCORD_BOT_TOKEN` | Discord gateway |
| `SLACK_BOT_TOKEN` | Slack gateway |
| `MODAL_TOKEN_ID` + `MODAL_TOKEN_SECRET` | Modal serverless |
| `DAYTONA_API_KEY` | Daytona sandbox |

### 系統環境變數

| 變數 | 用途 |
|------|------|
| `HERMES_HOME` | 覆蓋用戶資料目錄（預設 `~/.hermes`） |
| `HERMES_PROFILE` | 啟用命名 profile |
| `HERMES_CONFIG` | 覆蓋 config.yaml 路徑 |
| `HERMES_ENV` | 運行環境標記 |
| `HERMES_QUIET` | 抑制啟動訊息（CLI 設定） |
| `TERMINAL_ENV` | Terminal backend（local/docker/ssh/modal/daytona） |
| `PYTHONUTF8` / `PYTHONIOENCODING` | Windows UTF-8 設定（bootstrap） |
| `HERMES_HOME_MODE` | HERMES_HOME 目錄權限模式（for web server 穿越） |

## config.yaml 主要設定項

從 `cli-config.yaml.example`（62KB）和 README 推導：

```yaml
# ~/.hermes/config.yaml

model:
  provider: openrouter          # LLM provider 名稱
  default: anthropic/claude-opus-4.6  # 預設模型
  ollama_num_ctx: 65536         # Ollama context 長度
  context_length: 65536         # 模型 context 上限

toolsets:
  enabled:
    - core
    - terminal
    - web
  disabled:
    - browser                   # 停用 browser 工具

memory:
  mode: local                   # local / honcho / hybrid
  
gateway:
  platforms:
    telegram:
      bot_token: ${TELEGRAM_BOT_TOKEN}
      allowed_users: [user1, user2]
    discord:
      ...
      
terminal:
  backend: local                # local / docker / ssh / modal / daytona

delegation:
  max_concurrent_children: 3
  max_spawn_depth: 2
```

## 設定讀取 API

```python
# hermes_cli/config.py
from hermes_cli.config import cfg_get

value = cfg_get("model.provider", default="openrouter")
# 支援 dot-notation 路徑，YAML nested 結構
```

## Profile 系統

Hermes 支援多個命名 profile（e.g. work、personal、dev）：
```bash
HERMES_PROFILE=work hermes
hermes config set profile.active work
```

Profile 切換會改變 `HERMES_HOME` 下使用的子目錄。

## Feature Flags

⚠️ 未找到顯式的 feature flag 系統，功能啟用主要通過：
1. Toolset 啟用/停用（`hermes tools enable/disable`）
2. Plugin 啟用/停用（`hermes plugins enable/disable`）
3. Environment variable 覆蓋（`env_var_enabled()` 在 `utils.py` 中）
4. Config.yaml 設定項

## Secrets 管理

- **主要方式**：環境變數（`.env` 或系統環境）
- **工具**：`agent/secret_sources/` — 從 1Password、Vault 等讀取 secret（⚠️ 待驗證）
- **Credential Pool**：`agent/credential_pool.py` — 多個同類 API key 輪換使用
- **不建議**：直接寫入 config.yaml（雖然支援但有 git 洩漏風險）

## 安全相關設定

```yaml
# 命令審批設定（tool_guardrails）
approval:
  auto_approve_patterns:
    - "git status"
    - "ls *"
  auto_deny_patterns:
    - "rm -rf /"
```

`SECURITY.md` 文件存在，提供安全政策說明。
