# CLAUDE.md — claude-line-channel

This is a Claude Code channel plugin that connects LINE Messaging API to Claude Code via MCP.

## Architecture

The plugin runs as a single Bun process with two transports:

```
LINE app → LINE servers → HTTPS webhook → server.ts (Bun HTTP)
                                               ↓
                               MCP notification → Claude Code (stdio)
                                               ↓
                               Claude calls reply tool → LINE API → user
```

- **MCP stdio transport** — Claude Code spawns the plugin as a subprocess and communicates via stdin/stdout using the MCP protocol.
- **Bun HTTP server** — Listens on `LINE_WEBHOOK_PORT` (default 3456) for incoming LINE webhook events. Verifies HMAC-SHA256 signatures before processing.

## Key files

| File | Purpose |
|---|---|
| `server.ts` | Main plugin — MCP server + LINE webhook handler |
| `examples/line-router.ts` | Fan-out router for multi-session setups (one LINE channel → multiple Claude sessions) |
| `.claude-plugin/marketplace.json` | Plugin marketplace manifest, allows `claude plugin marketplace add` |

## State directory

Everything at runtime lives in `LINE_STATE_DIR` (default `~/.claude/channels/line/`):

```
~/.claude/channels/line/
├── .env              ← LINE_CHANNEL_ACCESS_TOKEN, LINE_CHANNEL_SECRET, LINE_WEBHOOK_PORT
├── access.json       ← dmPolicy, allowFrom[], groups{}
├── inbox/            ← media downloaded via get_content tool
├── unknown-groups.log ← group IDs not yet in access.json (for setup reference)
└── history.log       ← rolling log of all received messages (maintained by Claude Code)
```

`access.json` is read on every inbound message — changes take effect immediately without a restart.

## Security model

- **`allowFrom: []` = allow everyone**, not block everyone. The allowlist check is skipped when the array is empty. Always populate it before exposing the webhook publicly.
- `upload_file` tool uses `realpathSync` to verify the path resolves inside `inbox/` — Claude cannot be prompt-injected into uploading arbitrary host files.
- HMAC signature uses `timingSafeEqual` (constant-time) to prevent timing side-channels.
- Passwords for gofile.io uploads use `crypto.randomBytes(12)` — not `Math.random()`.

## Running locally for development

```sh
# Install dependencies
bun install

# Set credentials
mkdir -p ~/.claude/channels/line
echo 'LINE_CHANNEL_ACCESS_TOKEN=xxx' >> ~/.claude/channels/line/.env
echo 'LINE_CHANNEL_SECRET=yyy'       >> ~/.claude/channels/line/.env
echo 'LINE_WEBHOOK_PORT=3456'        >> ~/.claude/channels/line/.env

# Start Claude Code with this plugin
claude --dangerously-load-development-channels server:line
```

For local webhook testing, use [ngrok](https://ngrok.com/) or [localtunnel](https://localtunnel.github.io/www/) to expose port 3456.

## Multi-session setup

LINE allows only one webhook URL per channel. To route one webhook to multiple Claude Code sessions, run `line-router.ts` in front:

```
LINE webhook → line-router.ts (port 3456) → session A (port 3461)
                                           → session B (port 3462)
```

Each session sets its own `LINE_WEBHOOK_PORT` and `LINE_STATE_DIR`.

## access.json schema

```json
{
  "dmPolicy": "allowlist",
  "allowFrom": ["U..."],
  "groups": {
    "C...": { "requireMention": true, "allowFrom": [] }
  },
  "mentionPatterns": ["\\bclaude\\b"],
  "textChunkLimit": 5000,
  "chunkMode": "newline"
}
```

- `dmPolicy`: `"allowlist"` (default) or `"disabled"`
- `allowFrom`: user IDs allowed to DM. **Empty = allow all.**
- `groups`: keyed by group ID (`C...`) or room ID (`R...`)
- `requireMention`: only respond when bot is @mentioned or message matches `mentionPatterns`
- `textChunkLimit`: max characters per LINE message (LINE limit is 5000)
- `chunkMode`: `"newline"` splits at paragraph/line boundaries; `"length"` splits at exact character count

## LINE ID formats

| Prefix | Type |
|---|---|
| `U...` | User |
| `C...` | Group (multi-person chat created from a group) |
| `R...` | Room (multi-person chat created by invitation) |

Unknown IDs are logged to `unknown-groups.log` on first contact.

## Reply API vs Push API

LINE's Reply API is free but tokens expire 30 seconds after the inbound message. The plugin:
1. Tries Reply API first (single-use token, free)
2. Falls back to Push API if the token has expired (costs monthly quota)

If Claude consistently takes more than 30 s to respond, consider optimizing prompt length or using a faster model.

## Dependency notes

- `@modelcontextprotocol/sdk` — MCP server/transport
- No other runtime dependencies — `fetch`, `crypto`, and `Bun.serve` are all built-in to Bun
- `examples/line-router.ts` has zero dependencies beyond Bun built-ins
