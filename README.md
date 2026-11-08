# codimd_mcp

讓 Claude Code 把跨 session 的記憶（第二大腦）存到 CodiMD 的 MCP 插件。內容包含：

- `src/mcp.js`：stdio MCP server，提供 `memory_*` 工具
- `src/session-start.js`：SessionStart hook，每次開 session 時把記憶標題注入 context
- `skill/second-brain/`：告訴 agent 何時、如何寫入記憶的 skill
- `src/import.js`、`src/import-docs.js`：把本機 Claude Code 記憶或文件匯入 CodiMD

這個插件只是 client。伺服器端的 `/api/memory` 與 API token 頁面在 CodiMD fork 裡：<https://github.com/paulboul1013/codimd>，請先把它架起來。

## 需求

- Node.js 18 以上（執行時沒有 npm 相依套件）
- 一台有 `/api/memory` 的 CodiMD（上面那個 fork）
- Claude Code

## 安裝

以下用 `$MCP_DIR` 代表這個 repo 的絕對路徑。

```sh
git clone https://github.com/paulboul1013/codimd_mcp.git
cd codimd_mcp
export MCP_DIR="$PWD"
```

### 1. 建立 API token

登入 CodiMD，打開 `/settings/api-tokens` 建立一個 token。

### 2. 存放 token

```sh
mkdir -p ~/.config/codimd-memory
cp "$MCP_DIR/config/codimd-memory.config.example.json" ~/.config/codimd-memory/config.json
chmod 600 ~/.config/codimd-memory/config.json
```

編輯 `~/.config/codimd-memory/config.json`，填入 `token`。檔案權限若讓 group 或 others 可讀，server 會拒絕發出請求。

| 設定 | 環境變數（優先） | 說明 |
| --- | --- | --- |
| `token` | `CODIMD_MEMORY_TOKEN` | 必填 |
| `serverUrl` | `CODIMD_URL` | 選填。沒設定時依序嘗試 `http://localhost:3000`，再讀 `https://paulboul1013.github.io/codimd/target.json` 裡的 tunnel 網址 |

如果你的 CodiMD 不在本機也不是用那個 tunnel，請一定要設定 `serverUrl`。

### 3. 註冊 MCP server

對所有專案生效：

```sh
claude mcp add --scope user codimd-memory -- node "$MCP_DIR/src/mcp.js"
```

只想對單一專案生效，可以參考 `config/mcp.example.json` 在該專案根目錄建立 `.mcp.json`。

確認註冊成功：

```sh
claude mcp list
```

### 4. 設定權限、hook 與關閉本機 auto memory

把 `config/claude-settings.example.json` 的內容合併到 `~/.claude/settings.json`，並把 `/ABSOLUTE/PATH/TO/codimd_mcp` 換成 `$MCP_DIR`：

- `permissions.allow`：讓三個讀取工具（`memory_index`、`memory_search`、`memory_read`）不必每次詢問。寫入工具刻意不放進去，每次寫入都要你同意。
- `hooks.SessionStart`：開 session 時執行 `src/session-start.js`。
- `autoMemoryEnabled: false`：關掉 Claude Code 內建的本機記憶，改用 CodiMD。

### 5. 安裝 second-brain skill

```sh
mkdir -p ~/.claude/skills
ln -s "$MCP_DIR/skill/second-brain" ~/.claude/skills/second-brain
```

不想讓之後 `git pull` 的修改自動生效，就改用 `cp -r`。

### 6. 驗證

手動執行 hook，應該會印出 global 與目前專案的記憶標題：

```sh
echo '{"cwd": "'"$PWD"'"}' | node "$MCP_DIR/src/session-start.js"
```

若印出 `第二大腦：CodiMD 無法連線…`，代表連不到 CodiMD、沒有 token，或 token 被拒絕。接著重開 Claude Code，在 session 裡請它呼叫 `memory_index` 確認工具可用。

## 匯入既有資料（選用）

```sh
node "$MCP_DIR/src/import.js" --dry-run      # 預覽要匯入的 ~/.claude/projects/*/memory/
node "$MCP_DIR/src/import.js"                # 匯入
node "$MCP_DIR/src/import-docs.js" --dry-run docs/   # 預覽把文件匯成筆記
```

細節見 [docs/usage.md](docs/usage.md)。

## 工具一覽

| 工具 | 類型 | 用途 |
| --- | --- | --- |
| `memory_index` | 讀 | global 與目前專案的筆記標題 |
| `memory_search` | 讀 | 依文字、scope、type、是否封存搜尋 |
| `memory_read` | 讀 | 讀取單一筆記全文 |
| `memory_create` | 寫 | 建立筆記 |
| `memory_import_files` | 寫 | 把本機文件檔匯入成筆記，一個檔一則 |
| `memory_update` | 寫 | 取代筆記內容 |
| `memory_append` | 寫 | 在筆記後面追加內容 |
| `memory_archive` | 寫 | 封存筆記（不刪除） |

## 開發

```sh
npm install
npm test
```

`test/mcp.test.js` 的 `returns API errors as tool errors with the server message` 偶爾會逾時（約 1/10），原本 codimd repo 裡的同一份測試也一樣。

## 文件

- [docs/usage.md](docs/usage.md)：hook、伺服器位址查找、匯入工具的完整說明
- [docs/plans/second-brain/spec.md](docs/plans/second-brain/spec.md) 與 `tickets/`：設計規格
- [docs/adr/0001-memory-scoped-api-token.md](docs/adr/0001-memory-scoped-api-token.md)：為何使用限定 memory 範圍的 API token
