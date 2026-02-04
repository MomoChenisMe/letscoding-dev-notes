# OpenClaw Sandbox for Zeabur

適用於 [OpenClaw](https://openclaw.ai/) 的雲端沙盒環境，可部署於 Zeabur 平台。

## 基礎映像

本沙盒基於官方 OpenClaw Docker 映像：

- 映像：`ghcr.io/openclaw/openclaw:latest`
- 包含：OpenClaw + Node.js 22
- Heartbeat 問題已修復（2026.2.1+）

## 什麼是 OpenClaw？

[OpenClaw](https://github.com/openclaw/openclaw) 是一個 conversation-first 的個人 AI 助手，支援連接多種訊息平台：

- Telegram、WhatsApp、Discord、Slack
- Google Chat、Signal、iMessage、Microsoft Teams
- Matrix、Zalo、WebChat 等

特色：

- **對話優先** - 透過自然語言互動，無需複雜設定檔
- **多模型支援** - 支援 Anthropic、OpenAI 或本地模型
- **隱私優先** - 資料留在你手中

## 沙盒環境工具

| 工具 | 說明 |
|------|------|
| OpenClaw | 個人 AI 助手 CLI（官方映像內建） |
| Gemini CLI | Google Gemini AI 命令列工具 |
| Node.js 22 | JavaScript 執行環境（官方映像內建） |
| GitHub CLI | GitHub 命令列工具 |
| git | 版本控制 |

## Zeabur 部署設定

### 1. Volume 持久化設定

在 Zeabur 服務設定中新增 Volume，掛載到 `/home/node` 路徑：

1. 進入 Zeabur 專案 → 選擇服務
2. 點選「Storage」或「儲存」
3. 新增 Volume
4. 掛載路徑設定為：`/home/node`

持久化目錄結構：

```text
/home/node
├── .openclaw/             # OpenClaw 配置與工作區
│   ├── openclaw.json      # 主配置檔（JSON5 格式）
│   ├── .env               # 環境變數
│   ├── credentials/       # OAuth tokens
│   ├── agents/
│   │   └── main/
│   │       └── sessions/  # 對話歷史
│   └── workspace/
│       ├── HEARTBEAT.md   # Heartbeat 檢查清單
│       ├── AGENTS.md
│       └── SOUL.md
├── .gemini/               # Gemini CLI 認證與設定
└── .config/
    └── gh/                # GitHub CLI 認證與設定
```

### 2. 環境變數設定

Dockerfile 中的環境變數預設值可在 Zeabur 或 `docker run -e` 時覆蓋。

#### 使用者設定

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `TZ` | 時區設定 | `Asia/Taipei` |
| `GH_TOKEN` | GitHub Personal Access Token，用於 GitHub CLI 認證 | （空） |
| `TELEGRAM_BOT_TOKEN` | Telegram Bot Token，用於 OpenClaw 連接 Telegram | （空） |
| `GEMINI_API_KEY` | Gemini API Key（免登入使用 Gemini CLI） | （空） |

#### OpenClaw 設定

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `OPENCLAW_GATEWAY_PORT` | Gateway 端口 | `18789` |
| `OPENCLAW_BRIDGE_PORT` | Bridge 端口 | `18790` |
| `OPENCLAW_GATEWAY_TOKEN` | 認證令牌（建議在 Zeabur 設定） | （空） |

#### 系統設定（進階）

| 環境變數 | 說明 | 預設值 |
|----------|------|--------|
| `GEMINI_HOME` | Gemini CLI 設定目錄 | `/home/node/.gemini` |

設定方式：

1. 進入 Zeabur 專案 → 選擇服務
2. 點選「Variables」或「環境變數」
3. 新增環境變數

#### 取得各項 Token

**TZ（時區）**

常用時區值：

- 台灣：`Asia/Taipei`
- 日本：`Asia/Tokyo`
- 美西：`America/Los_Angeles`

**GH_TOKEN（GitHub Token）**

1. 前往 [GitHub Settings → Developer settings → Personal access tokens](https://github.com/settings/tokens)
2. 點選「Generate new token (classic)」
3. 勾選所需權限（建議：`repo`, `read:org`, `workflow`）
4. 複製 Token 並貼到 Zeabur 環境變數中

**TELEGRAM_BOT_TOKEN（Telegram Bot Token）**

1. 在 Telegram 中搜尋 [@BotFather](https://t.me/BotFather)
2. 發送 `/newbot` 建立新 Bot
3. 依照指示設定 Bot 名稱
4. 複製取得的 Token 並貼到 Zeabur 環境變數中

**GEMINI_API_KEY（Gemini API Key）**

1. 前往 [Google AI Studio](https://aistudio.google.com/apikey)
2. 點選「Create API Key」
3. 複製 API Key 並貼到 Zeabur 環境變數中

> 如果不設定 `GEMINI_API_KEY`，也可以在容器內執行 `gemini` 後選擇「Login with Google」進行 OAuth 登入。登入資訊會儲存在 `/home/node/.gemini` 中，容器重啟後不需重新登入。

### 3. CLI 工具登入

進入容器終端後：

```bash
# GitHub CLI 登入
gh auth login

# OpenClaw 初始化設定
openclaw onboard
```

認證資訊會儲存在 `/home/node` 中，容器重啟後不需重新登入。

> Gateway 會在容器啟動時自動執行，無需手動啟動。

### 4. Heartbeat 設定

在 `~/.openclaw/openclaw.json` 中設定定期心跳：

```json5
{
  "agents": {
    "defaults": {
      "heartbeat": {
        "every": "30m",
        "target": "telegram",
        "model": "gemini/gemini-2.0-flash-lite"
      }
    }
  }
}
```

| 選項 | 說明 | 預設值 |
|------|------|--------|
| `every` | 心跳間隔 | `30m` |
| `target` | 傳送目標（`last`/`telegram`/`discord` 等） | `last` |
| `model` | 指定模型（建議使用便宜模型節省成本） | 代理預設 |

> 成本優化：建議使用 Gemini Flash-Lite 等便宜模型執行 heartbeat，避免使用昂貴的 Opus 模型。

### 5. 沙箱模式設定（選用）

若需要工具隔離以提升安全性，可在 `~/.openclaw/openclaw.json` 中設定：

```json5
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "non-main",
        "scope": "agent"
      }
    }
  }
}
```

| 選項 | 說明 |
|------|------|
| `mode` | `non-main`：僅主代理外的代理在沙箱執行 / `all`：所有代理都在沙箱執行 |
| `scope` | `agent`：每個代理獨立沙箱 / `session`：同一會話共用沙箱 |

> 詳細說明請參考 [官方沙箱文件](https://docs.openclaw.ai/zh-CN/install/docker)

### 6. 瀏覽器工具設定（選用）

OpenClaw 支援瀏覽器自動化功能。由於 Zeabur 環境無法執行 Docker-in-Docker，需要部署獨立的 Browser 服務。

1. 在 Zeabur 新增服務，選擇「Docker Image」
2. 映像設定為：`ghcr.io/canyugs/openclaw-sandbox-browser:main`
3. 設定端口映射：`9222`（CDP）、`6080`（noVNC，除錯用）
4. 在 OpenClaw 容器的 `~/.openclaw/openclaw.json` 中設定：

```json5
{
  "browser": {
    "enabled": true,
    "defaultProfile": "sandbox",
    "profiles": {
      "sandbox": {
        "cdpUrl": "http://your-browser-service:9222"
      }
    }
  }
}
```

> 將 `your-browser-service` 替換為 Zeabur 內部服務名稱或 URL。

**環境變數（選用）**：

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `OPENCLAW_BROWSER_HEADLESS` | `1` | 設為 `0` 可啟用圖形模式（搭配 noVNC 除錯） |
| `OPENCLAW_BROWSER_ENABLE_NOVNC` | `1` | 設為 `0` 可關閉 noVNC |

### 7. 代理進階設定（選用）

#### 7.1 模型設定

- `model.primary`：主要模型
- `model.fallbacks`：備用模型清單（依序嘗試）
- `models`：模型允許清單（未列入的模型會被拒絕）

#### 7.2 並行與效能設定

- `workspace`：工作目錄路徑
- `maxConcurrent`：最大並行代理數（預設 1）
- `compaction.mode`：上下文壓縮模式（`default` / `safeguard`）

#### 7.3 子代理設定

- `subagents.maxConcurrent`：子代理最大並行數
- `subagents.archiveAfterMinutes`：自動歸檔時間

#### 完整範例

```json5
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "github-copilot/gpt-5-mini",
        "fallbacks": [
          "google-gemini-cli/gemini-3-flash-preview",
          "google-gemini-cli/gemini-3-pro-preview"
        ]
      },
      "models": {
        "google-gemini-cli/gemini-3-pro-preview": {},
        "google-gemini-cli/gemini-3-flash-preview": {},
        "github-copilot/claude-opus-4.5": {},
        "github-copilot/claude-sonnet-4.5": {},
        "github-copilot/gpt-5-mini": {}
      },
      "workspace": "/home/node/.openclaw/workspace",
      "maxConcurrent": 4,
      "compaction": {
        "mode": "safeguard"
      },
      "heartbeat": {
        "every": "3m",
        "target": "last",
        "prompt": "say hi"
      },
      "subagents": {
        "maxConcurrent": 8
      }
    }
  }
}
```

#### 設定說明

| 設定項 | 說明 | 預設值 |
|--------|------|--------|
| `model.primary` | 主要模型（格式：`provider/model`） | - |
| `model.fallbacks` | 備用模型清單 | `[]` |
| `models` | 模型允許清單 | - |
| `workspace` | 工作目錄 | `~/.openclaw/workspace` |
| `maxConcurrent` | 最大並行代理數 | `1` |
| `compaction.mode` | 壓縮模式 | `default` |
| `subagents.maxConcurrent` | 子代理並行數 | `1` |
| `subagents.archiveAfterMinutes` | 自動歸檔（分鐘） | `60` |

### 8. 多代理架構（選用）

使用 `agents.list` 可設定多個獨立代理：

**多代理用途**：

- 不同代理負責不同任務（如：主代理、助手代理、專家代理）
- 各代理有獨立的工作區、認證和工具權限
- 透過頻道綁定路由訊息到對應代理

#### 設定範例

```json5
{
  "agents": {
    "defaults": {
      // 共用預設設定...
    },
    "list": [
      {
        "id": "main",
        "default": true,
        "identity": {
          "name": "主助手",
          "emoji": "🤖"
        },
        "workspace": "/home/node/.openclaw/workspace",
        "model": {
          "primary": "anthropic/claude-sonnet-4-5"
        }
      },
      {
        "id": "coder",
        "identity": {
          "name": "程式助手",
          "emoji": "💻"
        },
        "workspace": "/home/node/.openclaw/agents/coder/workspace",
        "agentDir": "/home/node/.openclaw/agents/coder",
        "model": {
          "primary": "anthropic/claude-opus-4-5"
        },
        "tools": {
          "profile": "coding"
        }
      },
      {
        "id": "researcher",
        "identity": {
          "name": "研究助手",
          "emoji": "🔍"
        },
        "workspace": "/home/node/.openclaw/agents/researcher/workspace",
        "agentDir": "/home/node/.openclaw/agents/researcher",
        "model": {
          "primary": "google-gemini-cli/gemini-3-pro-preview"
        },
        "tools": {
          "profile": "full",
          "allow": ["group:web", "group:fs"]
        }
      }
    ]
  }
}
```

#### agents.list 欄位說明

| 欄位 | 說明 | 必填 |
|------|------|------|
| `id` | 代理識別碼（唯一） | 是 |
| `default` | 是否為預設代理 | 否 |
| `identity.name` | 代理顯示名稱 | 否 |
| `identity.emoji` | 代理表情符號 | 否 |
| `workspace` | 獨立工作目錄 | 否 |
| `agentDir` | 代理資料目錄 | 否 |
| `model` | 模型設定（覆蓋預設） | 否 |
| `tools.profile` | 工具權限（`minimal`/`messaging`/`coding`/`full`） | 否 |
| `tools.allow` | 額外允許的工具 | 否 |
| `tools.deny` | 禁止的工具 | 否 |
| `heartbeat` | 獨立心跳設定 | 否 |

#### 工具權限 Profile

| Profile | 說明 |
|---------|------|
| `minimal` | 僅基本狀態查詢 |
| `messaging` | 通訊 + 會話工具 |
| `coding` | 檔案系統 + 執行環境 |
| `full` | 無限制 |

## 本機測試

```bash
# 建構映像檔
docker build -t openclaw-sandbox .

# 執行容器（背景執行，Gateway 自動啟動）
docker run -d \
  -v openclaw-data:/home/node \
  -p 18789:18789 \
  -p 18790:18790 \
  openclaw-sandbox

# 確認 Gateway 已啟動
docker logs <container_id>

# 健康檢查
curl http://localhost:18789/health

# 進入容器進行初始化設定
docker exec -it <container_id> bash
openclaw onboard
```

## 安裝額外套件

容器重啟後，安裝在 `/home/node` 目錄下的資料會自動保留：

```bash
# npm 全域套件（需使用 sudo）
sudo npm i -g <package-name>
```

## 相關連結

- [OpenClaw 官網](https://openclaw.ai/)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [OpenClaw 文件](https://docs.openclaw.ai/)
- [OpenClaw Docker 映像](https://github.com/openclaw/openclaw/pkgs/container/openclaw)
- [Gateway 配置文件](https://docs.openclaw.ai/gateway/configuration)
- [模型設定文件](https://docs.openclaw.ai/concepts/models)
- [Heartbeat 設定文件](https://docs.openclaw.ai/gateway/heartbeat)
- [沙箱模式文件](https://docs.openclaw.ai/zh-CN/install/docker)
- [Browser 工具文件](https://docs.openclaw.ai/tools/browser)
- [Sandbox Browser 映像](https://github.com/canyugs/openclaw-sandbox-browser)
- [Zeabur](https://zeabur.com/)
