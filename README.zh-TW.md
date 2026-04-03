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

**5. 以 channel 旗標重新啟動。**

沒有這個旗標 server 不會連線 — 請退出目前 session 並重新開啟：

```sh
claude --dangerously-load-development-channels server:line
```

**6. 允許你的 LINE 用戶 ID。**

建立 `~/.claude/channels/line/access.json`：

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"],
  "groups": {}
}
```

如何找到你的 LINE 用戶 ID：將機器人加為好友後傳任一訊息，查看 `~/.claude/channels/line/unknown-groups.log` — 你的用戶 ID 會在第一次傳訊時出現。將它加入 `allowFrom` 後再傳一次訊息即可。

## 存取控制

DM 政策、群組設定、提及偵測和完整的 `access.json` 格式說明，請見 **[ACCESS.md](./ACCESS.md)**。

快速參考：LINE 用戶 ID 以 `U` 開頭，群組 ID 以 `C` 開頭，聊天室 ID 以 `R` 開頭。預設政策為 `allowlist`，來自未知用戶的訊息會被靜默丟棄。

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

## 安全性

- Webhook 簽章以 **HMAC-SHA256** 驗證，使用 constant-time 比較（防 timing side-channel 攻擊）
- `upload_file` 僅允許存取 inbox 目錄內的檔案 — 防止透過 LINE 訊息進行 prompt injection 攻擊，竊取任意檔案
- 檔案上傳密碼使用 `crypto.randomBytes` 生成（96-bit 熵）
- `.env` 在啟動時自動 chmod 為 `0600`
- 未知群組 ID 在記錄前會先清除控制字元

## 授權

Apache-2.0
