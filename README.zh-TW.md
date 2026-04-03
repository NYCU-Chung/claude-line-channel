# LINE Channel for Claude Code

[English](./README.md) | 繁體中文

Claude Code 的 channel 系統讓訊息平台可以直接接入 Claude Code session。這個 plugin 新增了 **LINE** 支援——目前是唯一的第三方 LINE channel plugin。官方目前只支援 Discord 和 Telegram。

透過 MCP server 將 LINE 機器人接入 Claude Code。當有人傳訊息給機器人時，MCP server 會將訊息轉發給 Claude，並提供工具讓 Claude 回覆。Claude 可以回應私訊和群組訊息、下載媒體檔案、傳送圖片——全部在 Claude Code session 內完成。

## 事前準備

- [Bun](https://bun.sh) — MCP server 使用 Bun 執行。安裝指令：`curl -fsSL https://bun.sh/install | bash`
- 一個可從外部存取的 HTTPS 端點，用來接收 webhook（例如透過 nginx 或 Caddy）

## 快速設定
> 單人私訊設定。群組和多人設定請見 [ACCESS.md](./ACCESS.md)。

**1. 建立 LINE 官方帳號並開啟 Messaging API。**

自 2024 年 9 月起，Messaging API 頻道無法再直接從 LINE Developers Console 建立，新流程如下：

1. 登入 [LINE Official Account Manager](https://manager.line.biz/)，建立一個 LINE 官方帳號。
2. 進入帳號設定，找到 **Messaging API** 並點選「啟用」。系統會要求選擇或建立 Provider。
3. 開啟 [LINE Developers Console](https://developers.line.biz/console/)，在你的 Provider 下找到自動建立好的頻道。

在 LINE Developers Console 的頻道頁面：
- **Basic settings** 分頁 → 複製 **Channel secret**
- **Messaging API** 分頁 → 點選「Issue」產生 **Channel access token（長期）** 並複製
- 回到 LINE Official Account Manager → **自動回應訊息** → 關閉**自動回應** — 由 Claude 負責回覆

**2. 安裝 plugin。**

以下指令在 Claude Code session 內執行 — 先執行 `claude` 開啟 session。

加入 plugin marketplace：
```
claude plugin marketplace add NYCU-Chung/claude-line-channel
```

安裝 plugin：
```
claude plugin install line@claude-line-channel
```

**3. 設定憑證。**

```sh
mkdir -p ~/.claude/channels/line
cat > ~/.claude/channels/line/.env << 'EOF'
LINE_CHANNEL_ACCESS_TOKEN=<你的長期 access token>
LINE_CHANNEL_SECRET=<你的 channel secret>
LINE_WEBHOOK_PORT=3456
EOF
chmod 600 ~/.claude/channels/line/.env
```

> 若要在同一台機器執行多個機器人（不同 token、不同 allowlist），請將 `LINE_STATE_DIR` 指向不同的目錄。

**4. 暴露 webhook 端點。**

將 LINE 頻道的 webhook URL 設定為 `https://你的伺服器/webhook`。使用 nginx、Caddy 或任何反向代理將 HTTPS 流量轉發至 `http://localhost:3456`。

在 LINE Developers Console 驗證 webhook — 應回傳 HTTP 200。

**5. 建立 session CLAUDE.md（建議）。**

將內附的模板複製到你的工作目錄。Claude 啟動時會自動讀取它，學會如何作為 LINE bot 運作——包括在重啟後讀取 `history.log` 以恢復對話脈絡。

```sh
cp ~/.claude/plugins/cache/claude-line-channel/line/*/examples/CLAUDE.md ~/my-line-bot/CLAUDE.md
```

可依需求自訂（角色設定、語言、規則等）。

**6. 以 channel 旗標重新啟動。**

沒有這個旗標 server 不會連線 — 請退出目前 session 並重新開啟：

```sh
cd ~/my-line-bot
claude --dangerously-load-development-channels server:line
```

**7. 允許你的 LINE 用戶 ID。**

建立 `~/.claude/channels/line/access.json`：

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"],
  "groups": {}
}
```

如何找到你的 LINE 用戶 ID：將機器人加為好友後傳任一訊息，查看 `~/.claude/channels/line/unknown-groups.log` — 你的用戶 ID 會在第一次傳訊時出現。將它加入 `allowFrom` 後再傳一次訊息即可。

> 步驟 5–7 假設你在一個專用目錄（`~/my-line-bot/`）執行 Claude。該目錄下的 `CLAUDE.md` 會在 session 啟動時自動載入。

## 使用方式

設定完成後，Claude Code 會持續運行並監聽 LINE 訊息。

**私訊**

將機器人加為好友後傳訊息，plugin 會將訊息轉發給 Claude，Claude 會在同一個對話中直接回覆。除了在 `access.json` 的 `allowFrom` 填入你的用戶 ID 之外，不需要額外設定。

**群組**

1. 將機器人加入 LINE 群組。
2. 第一則訊息後，群組 ID 會出現在 `~/.claude/channels/line/unknown-groups.log`。
3. 將它加入 `access.json` 的 `groups` 欄位：

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"],
  "groups": {
    "Cxxxxxxxxxxxxxxxxxxxxxxxxxxxxx": {
      "requireMention": true,
      "allowFrom": []
    }
  }
}
```

`requireMention: true` 表示只有在群組內 @ 機器人時才會回應。如果無法 @（例如某些舊版 LINE 用戶端），也可以透過關鍵字觸發，在 `mentionPatterns` 加入 regex：

```json
{
  "groups": {
    "Cxxxxxxxxxxxxxxxxxxxxxxxxxxxxx": {
      "requireMention": true,
      "allowFrom": []
    }
  },
  "mentionPatterns": ["^Claude", "\\bclaudebot\\b"]
}
```

訊息符合任一 pattern 就視同被提及。設為 `requireMention: false` 則回應群組內每一則訊息。

**自訂 Claude 的行為**

將 `examples/CLAUDE.md` 複製到你的工作目錄並編輯。Claude Code 啟動時會自動載入這個檔案——用它來設定角色、語言、回應風格或任何你需要的規則。

```sh
cp examples/CLAUDE.md ~/my-line-bot/CLAUDE.md
# 然後編輯 ~/my-line-bot/CLAUDE.md
```

**重啟後保留對話脈絡**

Claude Code 會在 `~/.claude/channels/line/history.log` 維護一個滾動的訊息記錄。內附的 `CLAUDE.md` 模板會指示 Claude 在啟動時讀取它，確保 session 重啟後對話脈絡不會中斷。

## 存取控制

DM 政策、群組設定、提及偵測和完整的 `access.json` 格式說明，請見 **[ACCESS.zh-TW.md](./ACCESS.zh-TW.md)**。

快速參考：LINE 用戶 ID 以 `U` 開頭，群組 ID 以 `C` 開頭，聊天室 ID 以 `R` 開頭。預設政策為 `allowlist`，來自未知用戶的訊息會被靜默丟棄。

> **⚠️ 安全陷阱：** `allowFrom: []`（空陣列）**不代表封鎖所有人**，而是**允許所有人存取**。當列表為空時，驗證邏輯會直接跳過。在公開 webhook 之前，請務必在 `allowFrom` 填入至少一個用戶 ID，或將 `dmPolicy` 設為 `"disabled"` 以封鎖所有私訊直到設定完成。

> **⚠️ 機器存取權限：** Claude Code 擁有對你機器的完整存取權限。請把 LINE bot 的 `allowFrom` 名單當作 SSH authorized keys 來看待——只加入你完全信任的 LINE 用戶 ID。

## Claude 可使用的工具

| 工具 | 用途 |
| --- | --- |
| `reply` | 傳送文字訊息到私訊或群組。接受 `chat_id` 和 `text`。長訊息自動分段，25 秒內優先使用免費的 Reply API，逾時改用 Push API。 |
| `get_content` | 下載 LINE 用戶傳送的媒體訊息（圖片/影片/音訊/檔案）到 inbox 目錄。回傳檔案路徑，圖片另附預覽。 |
| `send_image` | 透過公開 HTTPS URL 傳送圖片到 LINE 聊天。 |
| `upload_file` | 將 **inbox 目錄內**的檔案上傳至 gofile.io，並附上密碼和到期時間。回傳下載連結和密碼。 |
| `fetch_messages` | LINE 未開放訊息歷史 API — 此工具會回傳相關說明。 |

## 多 session（line-router）

若要讓多個 Claude Code session 共用同一個 LINE 頻道，可使用 `examples/line-router.ts`。它會驗證一次 HMAC 簽章，再將 webhook 分發給各 session 的 port。

```sh
# Session 1：LINE_WEBHOOK_PORT=3461
# Session 2：LINE_WEBHOOK_PORT=3462
# Router 監聽 3456 並同時轉發給兩個 session

LINE_CHANNEL_SECRET=<secret> bun examples/line-router.ts
```

## 正式部署（tmux + watchdog）

長期運行時，使用 tmux 讓 session 在 SSH 斷線後仍然存活，並搭配 watchdog 在 MCP server 崩潰時自動重啟。

### 目錄結構

```
~/line-dm/
├── launch.sh      # tmux 進入點 — 重啟迴圈 + rolling context 修剪
├── start.sh       # MCP server 啟動腳本，被 .claude.json 的 mcpServers 引用
└── CLAUDE.md      # Claude 的 persona 與指令
```

### launch.sh

```bash
#!/bin/bash
PROJ=~/.claude/projects/$(realpath ~/line-dm | sed 's|/|-|g' | sed 's|^-||')
MAX_SIZE=3000000

JSONL=$(ls "$PROJ"/*.jsonl 2>/dev/null | head -1)
if [ -n "$JSONL" ] && [ "$(wc -c < "$JSONL")" -gt "$MAX_SIZE" ]; then
  LINES=$(wc -l < "$JSONL")
  KEEP=$(( LINES * 2000000 / $(wc -c < "$JSONL") ))
  tail -n "$KEEP" "$JSONL" > "$JSONL.tmp" && mv "$JSONL.tmp" "$JSONL"
fi

CONTINUE=""
ls "$PROJ"/*.jsonl 2>/dev/null | grep -q . && CONTINUE="--continue"

while true; do
  claude --dangerously-skip-permissions $CONTINUE \
    --dangerously-load-development-channels server:line
  CONTINUE="--continue"
  echo "Claude exited, restarting in 5 seconds..."
  sleep 5
done
```

### start.sh

被 `~/.claude.json` 中專案的 `mcpServers.line` 引用。每次啟動前先清除殘留的 bun 程序：

```bash
#!/bin/bash
fuser -k 3461/tcp 2>/dev/null || true
LINE_STATE_DIR=~/.claude/channels/line-dm \
LINE_WEBHOOK_PORT=3461 \
exec bun run --cwd ~/.claude/plugins/cache/claude-line-channel/line/0.1.0 start
```

在 `~/.claude.json` 加入專案設定：

```json
{
  "projects": {
    "/home/user/line-dm": {
      "mcpServers": {
        "line": {
          "command": "bash",
          "args": ["/home/user/line-dm/start.sh"]
        }
      }
    }
  }
}
```

> **為什麼需要 `mcpServers`**：plugin 系統將 server 以 `plugin:line:line` 命名，channel system 無法匹配。`mcpServers` 設定確保有一個名稱為 `line` 的 server 可用。

### 建立 tmux session

```bash
tmux new-session -d -s line-dm "cd ~/line-dm && bash launch.sh"
```

### Watchdog

`--dangerously-load-development-channels` 首次啟動時會顯示一次性確認對話框。Watchdog 可自動確認並在 bun 崩潰時重啟 Claude：

```bash
#!/bin/bash
# 在另一個 tmux session 執行：tmux new-session -d -s watchdog "bash watchdog.sh"
GRACE=0
while true; do
  PANE_PID=$(tmux list-panes -t line-dm -F '#{pane_pid}' 2>/dev/null | head -1)
  # 自動確認 development channels 對話框
  tmux send-keys -t line-dm "" Enter 2>/dev/null

  if [ "$GRACE" -le 0 ] && ! ss -tlnp | grep -q ':3461 '; then
    CLAUDE_PID=$(pstree -p "$PANE_PID" 2>/dev/null | grep -o 'claude([0-9]*)' | head -1 | grep -o '[0-9]*')
    if [ -n "$CLAUDE_PID" ]; then
      echo "$(date): bun MCP server 停止回應，重啟 Claude ($CLAUDE_PID)"
      kill "$CLAUDE_PID"
      GRACE=6  # 60 秒 grace period
    fi
  fi
  [ "$GRACE" -gt 0 ] && GRACE=$((GRACE - 1))
  sleep 10
done
```

## 已知限制與踩坑紀錄

實際運行後發現的注意事項：

- **`claude plugin install` 會用 SSH clone GitHub — 若沒有 SSH key 請先設定 HTTPS。**
  這個問題影響所有未設定 GitHub SSH key 的環境（包括全新的 VPS）。會出現 `Permission denied (publickey)` 或 `Host key verification failed`。在執行 `plugin install` 前先執行這行：
  ```bash
  git config --global url."https://github.com/".insteadOf "git@github.com:"
  ```

- **LINE 每個頻道只允許一個 webhook URL。** 若要讓多個 Claude Code session 共用同一個 LINE 頻道（例如每個群組各一個 session），請使用 `examples/line-router.ts` 將 webhook 分發到各 session 的 port。沒有這個中間層，只有一個 session 能收到訊息。
- **Reply token 在 30 秒後失效。** Plugin 在每則訊息到達後的 30 秒內使用免費的 Reply API 回覆，之後改用需要扣配額的 Push API。若 Claude 回應超過 30 秒，該次回覆會消耗 Push API 配額。
- **LINE 沒有訊息歷史 API。** 機器人只看得到它正在運行時收到的訊息。Claude Code 會自動在 state 目錄維護一個滾動的 `history.log`（`~/.claude/channels/line/history.log`）——可以指示 Claude 在啟動時讀取它，以在重啟後恢復對話脈絡。
- **用戶必須先加機器人為好友才能傳私訊。** LINE 不允許向未加為好友的機器人傳送私訊。
- **提及偵測需要機器人的用戶 ID**，這個 ID 在啟動時非同步取得。這個過程中 webhook 會回傳 HTTP 503，LINE 會自動重試。
- **群組 ID 與聊天室 ID 不同：** 從群組發起的多人聊天 ID 以 `C` 開頭；透過邀請建立的聊天室 ID 以 `R` 開頭。兩者不同，需在 `access.json` 中分別設定。

## 安全性

- Webhook 簽章以 **HMAC-SHA256** 驗證，使用 constant-time 比較（防 timing side-channel 攻擊）
- `upload_file` 僅允許存取 inbox 目錄內的檔案 — 防止透過 LINE 訊息進行 prompt injection 攻擊，竊取任意檔案
- 檔案上傳密碼使用 `crypto.randomBytes` 生成（96-bit 熵）
- `.env` 在啟動時自動 chmod 為 `0600`
- 未知群組 ID 在記錄前會先清除控制字元

## 問題排查

### `server:line · no MCP server configured with that name`

Claude 找到了 development channel（`server:line`），但沒有對應名稱的 MCP server。原因是 plugin 系統將 server 命名為 `plugin:line:line`，而非 `line`。

修復方式：在 `~/.claude.json` 的專案設定中加入 `mcpServers.line`（參見上方 `start.sh` 章節）。

若 plugin cache 版本全部被標記為 orphaned，Claude 會拒絕執行。移除標記：

```bash
rm -f ~/.claude/plugins/cache/claude-line-channel/line/*/.orphaned_at
```

### 狀態列顯示 `1 MCP server failed`

MCP server 啟動時崩潰。最常見原因：殘留的 bun 程序佔住 webhook port。在 `start.sh` 開頭加入 `fuser -k <port>/tcp` 可在每次重啟時自動清除。

手動除錯：

```bash
LINE_STATE_DIR=~/.claude/channels/line-dm LINE_WEBHOOK_PORT=3461 bash ~/line-dm/start.sh
```

### Webhook 收到了，但 Claude 沒有回應

1. 確認 bun 在運行：`ss -tlnp | grep 3461`
2. 檢查殘留 bun 程序：`ps -u $(whoami) -o pid,ppid,cmd | grep bun`
3. 繞過 line-router，直接打 bun 測試：
   ```bash
   SECRET=$(grep LINE_CHANNEL_SECRET ~/.claude/channels/line-dm/.env | cut -d= -f2)
   PAYLOAD='{"destination":"U0","events":[{"type":"message","mode":"active","timestamp":1000000000000,"source":{"type":"user","userId":"Utest"},"webhookEventId":"ev1","deliveryContext":{"isRedelivery":false},"message":{"id":"m1","type":"text","quoteToken":"q","text":"ping"}}]}'
   SIG=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$SECRET" -binary | base64)
   curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3461/webhook \
     -H "Content-Type: application/json" -H "x-line-signature: $SIG" -d "$PAYLOAD"
   ```
   預期回傳 `200`。`403` 表示 HMAC secret 錯誤。
4. 觀察 Claude session 是否出現 `← line ·`：`tmux capture-pane -t line-dm -p | tail -20`

### 訊息已讀但 Claude 沒有回覆

使用 line-router 多 session 時，確認 router 是否正常轉發：

```bash
tmux capture-pane -t line-router -p | grep error | tail -10
```

未使用 port 的 `Unable to connect` 錯誤是正常現象，不影響運作。

## 授權

Apache-2.0
