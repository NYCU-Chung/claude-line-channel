# LINE Channel for Claude Code

[繁體中文](./README.zh-TW.md) | English

Claude Code's channel system lets messaging platforms connect directly to a Claude Code session. This plugin adds **LINE** support — the only third-party LINE channel plugin available. Official support currently covers only Discord and Telegram.

Connect a LINE bot to your Claude Code with an MCP server. When someone messages the bot, the server forwards the message to Claude and provides tools to reply. Claude can respond to DMs and group chats, download media, and send images — all from within a Claude Code session.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with `curl -fsSL https://bun.sh/install | bash`.
- A publicly accessible HTTPS endpoint for the webhook (e.g. behind nginx or Caddy).

## Quick Setup
> Single-user DM setup. See [ACCESS.md](./ACCESS.md) for groups and multi-user setups.

**1. Create a LINE Official Account and enable Messaging API.**

As of September 2024, Messaging API channels can no longer be created directly from the LINE Developers Console. The new flow:

1. Sign in to [LINE Official Account Manager](https://manager.line.biz/) and create a LINE Official Account.
2. In the account settings, find **Messaging API** and click **Enable**. Select or create a Provider when prompted.
3. Open [LINE Developers Console](https://developers.line.biz/console/) and navigate to the channel that was automatically created under your Provider.

On the channel page in LINE Developers Console:
- **Basic settings** tab → copy the **Channel secret**
- **Messaging API** tab → click **Issue** to generate a **Channel access token (long-lived)** and copy it
- In LINE Official Account Manager → **Auto-response messages** → turn off **Auto-reply messages** — Claude will handle replies

**2. Install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

Add the plugin marketplace:
```
claude plugin marketplace add NYCU-Chung/claude-line-channel
```

Install the plugin:
```
claude plugin install line@claude-line-channel
```

**3. Configure credentials.**

```sh
mkdir -p ~/.claude/channels/line
cat > ~/.claude/channels/line/.env << 'EOF'
LINE_CHANNEL_ACCESS_TOKEN=<your long-lived access token>
LINE_CHANNEL_SECRET=<your channel secret>
LINE_WEBHOOK_PORT=3456
EOF
chmod 600 ~/.claude/channels/line/.env
```

> To run multiple bots on one machine (different tokens, separate allowlists), point `LINE_STATE_DIR` at a different directory per instance.

**4. Expose the webhook.**

Point your LINE channel's webhook URL at `https://your-server/webhook`. Use nginx, Caddy, or any reverse proxy to forward HTTPS to `http://localhost:3456`.

Verify in the LINE Developers Console — the webhook should return HTTP 200.

**5. Relaunch with the channel flag.**

The server won't connect without this — exit your session and start a new one:

```sh
claude --dangerously-load-development-channels server:line
```

**6. Allow your LINE user ID.**

Create `~/.claude/channels/line/access.json`:

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"],
  "groups": {}
}
```

To find your LINE user ID: add the bot as a friend and send it any message. Check `~/.claude/channels/line/unknown-groups.log` — your user ID appears there on first contact. Add it to `allowFrom` and message again.

## Access control

See **[ACCESS.md](./ACCESS.md)** for DM policies, group configuration, mention detection, and the full `access.json` schema.

Quick reference: LINE user IDs start with `U`, group IDs with `C`, room IDs with `R`. Default policy is `allowlist` — messages from unknown users are dropped silently.

> **⚠️ Security trap:** `allowFrom: []` (empty array) does **not** mean "block everyone" — it means **allow everyone**. The check is skipped when the list is empty. Always put at least one user ID in `allowFrom` before exposing the webhook publicly, or set `dmPolicy: "disabled"` to block all DMs until you're ready.

> **⚠️ Machine access:** Claude Code has full access to your machine. Treat the LINE bot's `allowFrom` list the same way you'd treat SSH authorized keys — only add LINE user IDs you trust completely.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send a text message to a DM or group chat. Takes `chat_id` + `text`. Auto-chunks long messages using the free Reply API (within 25 s of the inbound message), falls back to Push API. |
| `get_content` | Download a media message (image/video/audio/file) sent by a LINE user to the inbox directory. Returns the file path; images also return an inline preview. |
| `send_image` | Send an image to a LINE chat via a publicly accessible HTTPS URL. |
| `upload_file` | Upload a file **from the inbox directory only** to gofile.io with a password and expiry. Returns the download link and password. |
| `fetch_messages` | LINE does not expose a message history API for bots — this tool returns a note about that limitation. |

## Multiple sessions (line-router)

To run multiple Claude Code sessions sharing one LINE channel, use `examples/line-router.ts`. It verifies the HMAC signature once and fans the webhook out to each session's port.

```sh
# Session 1: LINE_WEBHOOK_PORT=3461
# Session 2: LINE_WEBHOOK_PORT=3462
# Router listens on 3456 and forwards to both

LINE_CHANNEL_SECRET=<secret> bun examples/line-router.ts
```

## Known limitations and gotchas

Things we discovered running this in production:

- **LINE only allows one webhook URL per channel.** If you want multiple Claude Code sessions to share one LINE channel (e.g. one session per group), use `examples/line-router.ts` to fan out the webhook to each session's port. Without it, only one session receives messages.
- **Reply tokens expire in 30 seconds.** The plugin uses the free Reply API for the first response after each inbound message, then falls back to the paid Push API. If Claude takes more than 30 s to respond, the reply costs Push API quota.
- **LINE has no message history API.** The bot only sees messages that arrive while it is running. Claude Code automatically maintains a rolling `history.log` in the state directory (`~/.claude/channels/line/history.log`) — instruct Claude to read it on startup to restore context after a restart.
- **Bot must be a friend before users can DM it.** LINE does not allow DMs to bots unless the user has added the bot as a friend first.
- **Mention detection requires the bot's user ID**, which is fetched asynchronously on startup. The webhook returns HTTP 503 for a few seconds during this window — LINE will retry automatically.
- **Group IDs vs room IDs:** Multi-person chats started from a group have IDs starting with `C`; chats started from a direct invitation (rooms) start with `R`. They are different and must be configured separately in `access.json`.

## Security

- Webhook signature verified with **HMAC-SHA256** using constant-time comparison (no timing side-channel)
- `upload_file` is restricted to the inbox directory — prompt injection via LINE messages cannot cause arbitrary file exfiltration
- File upload passwords generated with `crypto.randomBytes` (96-bit entropy)
- `.env` file chmod'd to `0600` on startup
- Unknown group IDs sanitized before logging

## License

Apache-2.0
