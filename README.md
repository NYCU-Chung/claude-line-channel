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

**5. Set up a session CLAUDE.md (recommended).**

Copy the included template to your working directory. Claude will read it on startup and know how to behave as a LINE bot — including reading `history.log` for context after restarts.

```sh
cp ~/.claude/plugins/cache/claude-line-channel/line/*/examples/CLAUDE.md ~/my-line-bot/CLAUDE.md
```

Customize it to fit your use case (persona, language, rules, etc.).

**6. Relaunch with the channel flag.**

The server won't connect without this — exit your session and start a new one:

```sh
cd ~/my-line-bot
claude --dangerously-load-development-channels server:line
```

**7. Allow your LINE user ID.**

Create `~/.claude/channels/line/access.json`:

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"],
  "groups": {}
}
```

To find your LINE user ID: add the bot as a friend and send it any message. Check `~/.claude/channels/line/unknown-groups.log` — your user ID appears there on first contact. Add it to `allowFrom` and message again.

> Steps 5–7 assume you run Claude from a dedicated directory (`~/my-line-bot/`). The `CLAUDE.md` in that directory is loaded automatically on session start.

## Usage

Once setup is complete, Claude Code runs as a persistent session that listens for LINE messages.

**DMs**

Add the bot as a friend on LINE and send it a message. The plugin forwards the message to Claude, which responds directly in the same conversation. No additional configuration is required beyond the allowlist entry in `access.json`.

**Groups**

1. Add the bot to a LINE group.
2. The bot's group ID will appear in `~/.claude/channels/line/unknown-groups.log` on first message.
3. Add it to `access.json` under `groups`:

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

With `requireMention: true`, Claude only responds when @mentioned in the group. If @mention isn't available (e.g. older LINE clients), you can also trigger it with a keyword by adding `mentionPatterns`:

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

Any message matching one of the regex patterns counts as a mention. Set `requireMention: false` to respond to every message in the group.

**Customizing Claude's behavior**

Copy `examples/CLAUDE.md` to your working directory and edit it. This file is loaded automatically when Claude Code starts — use it to set a persona, language, response style, or any rules specific to your use case.

```sh
cp examples/CLAUDE.md ~/my-line-bot/CLAUDE.md
# then edit ~/my-line-bot/CLAUDE.md
```

**Keeping context across restarts**

Claude Code maintains a rolling `history.log` at `~/.claude/channels/line/history.log`. The included `CLAUDE.md` template instructs Claude to read it on startup, so conversation context is preserved even after the session restarts.

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

## Production deployment (tmux + watchdog)

For long-running deployments, use tmux to keep sessions alive across SSH disconnects. A watchdog script handles automatic restarts when the MCP server dies.

### Directory layout

```
~/line-dm/
├── launch.sh      # tmux entry point — restart loop + rolling context pruning
├── start.sh       # MCP server startup, referenced by mcpServers in .claude.json
└── CLAUDE.md      # persona and instructions for Claude
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

Referenced by `mcpServers.line` in your project entry inside `~/.claude.json`. Kills any orphaned bun process holding the port before starting:

```bash
#!/bin/bash
fuser -k 3461/tcp 2>/dev/null || true
LINE_STATE_DIR=~/.claude/channels/line-dm \
LINE_WEBHOOK_PORT=3461 \
exec bun run --cwd ~/.claude/plugins/cache/claude-line-channel/line/0.1.0 start
```

Add a project entry to `~/.claude.json`:

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

> **Why `mcpServers` is required**: the plugin system registers the server as `plugin:line:line`, which the channel system cannot match. A `mcpServers` entry ensures a server named exactly `line` is available.

### Creating the tmux session

```bash
tmux new-session -d -s line-dm "cd ~/line-dm && bash launch.sh"
```

### Watchdog

The `--dangerously-load-development-channels` flag shows a one-time confirmation dialog on startup. A watchdog handles this and restarts Claude when bun crashes:

```bash
#!/bin/bash
# Run in a separate tmux session: tmux new-session -d -s watchdog "bash watchdog.sh"
GRACE=0
while true; do
  PANE_PID=$(tmux list-panes -t line-dm -F '#{pane_pid}' 2>/dev/null | head -1)
  # Auto-confirm the development channels dialog
  tmux send-keys -t line-dm "" Enter 2>/dev/null

  if [ "$GRACE" -le 0 ] && ! ss -tlnp | grep -q ':3461 '; then
    CLAUDE_PID=$(pstree -p "$PANE_PID" 2>/dev/null | grep -o 'claude([0-9]*)' | head -1 | grep -o '[0-9]*')
    if [ -n "$CLAUDE_PID" ]; then
      echo "$(date): bun MCP server down, restarting Claude ($CLAUDE_PID)"
      kill "$CLAUDE_PID"
      GRACE=6  # 60-second grace period before next health check
    fi
  fi
  [ "$GRACE" -gt 0 ] && GRACE=$((GRACE - 1))
  sleep 10
done
```

## Known limitations and gotchas

Things we discovered running this in production:

- **`claude plugin install` uses SSH to clone from GitHub — set up HTTPS if you don't have SSH keys.**
  This affects everyone without a GitHub SSH key configured (including fresh VPS setups). You'll see `Permission denied (publickey)` or `Host key verification failed`. Fix it before running `plugin install`:
  ```bash
  git config --global url."https://github.com/".insteadOf "git@github.com:"
  ```

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

## Troubleshooting

### `server:line · no MCP server configured with that name`

Claude found a development channel (`server:line`) but has no MCP server with that name. This happens because the plugin system registers the server as `plugin:line:line`, not `line`.

Fix: add a `mcpServers.line` entry to your project in `~/.claude.json` as shown in the `start.sh` section above.

If the plugin cache versions are all marked orphaned, Claude will refuse to run them. Remove the markers:

```bash
rm -f ~/.claude/plugins/cache/claude-line-channel/line/*/.orphaned_at
```

### `1 MCP server failed` in the status bar

The MCP server crashed during startup. Most common cause: an orphaned bun process is holding the webhook port. Add `fuser -k <port>/tcp` at the top of `start.sh` to clear it on each restart.

To debug manually:

```bash
LINE_STATE_DIR=~/.claude/channels/line-dm LINE_WEBHOOK_PORT=3461 bash ~/line-dm/start.sh
```

### Webhook arrives but Claude does not respond

1. Confirm bun is running: `ss -tlnp | grep 3461`
2. Check for orphaned bun processes: `ps -u $(whoami) -o pid,ppid,cmd | grep bun`
3. Test the webhook path directly (bypassing line-router):
   ```bash
   SECRET=$(grep LINE_CHANNEL_SECRET ~/.claude/channels/line-dm/.env | cut -d= -f2)
   PAYLOAD='{"destination":"U0","events":[{"type":"message","mode":"active","timestamp":1000000000000,"source":{"type":"user","userId":"Utest"},"webhookEventId":"ev1","deliveryContext":{"isRedelivery":false},"message":{"id":"m1","type":"text","quoteToken":"q","text":"ping"}}]}'
   SIG=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$SECRET" -binary | base64)
   curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:3461/webhook \
     -H "Content-Type: application/json" -H "x-line-signature: $SIG" -d "$PAYLOAD"
   ```
   Expected: `200`. A `403` means the HMAC secret is wrong.
4. Watch the Claude session for the `← line ·` notification: `tmux capture-pane -t line-dm -p | tail -20`

### Messages marked read but Claude does not reply

When running multiple sessions via line-router, check whether the router is forwarding correctly:

```bash
tmux capture-pane -t line-router -p | grep error | tail -10
```

`Unable to connect` errors for unused ports are harmless — only the port matching the active session matters.

## License

Apache-2.0
