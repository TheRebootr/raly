# Phase 4: Telegram Bot Setup + Directory Structure

## Context

RALY is a hardened Mac Mini 2018 running Omarchy 3.x (Arch Linux). Boot is a TypeScript/Bun
Telegram bot forked from [linuz90/claude-telegram-bot](https://github.com/linuz90/claude-telegram-bot)
(MIT license), customized for RALY's architecture. It runs inside a long-lived Docker container
(Debian Bookworm) and delegates work to Claude Code CLI as a subprocess. This phase creates the
Telegram bot identity, verifies credentials, and sets up the directory structure.

Architecture summary:
- `~/BootDrive/app/` — Boot's source code (TypeScript/Bun) + Dockerfile
- `~/BootDrive/data/` — Boot's brain (SQLite, config, Claude auth) — bind-mounted as /data
- `~/BootDrive/workspace/` — Project files — bind-mounted as /workspace (accepted blast radius)

## Prerequisites

- Phase 0 complete: machine verified
- Phase 2 complete: container spec finalized
- Telegram account exists (or will be created)
- Docker running on host

## Steps

### 4.1 Create Telegram Bot

1. Open Telegram on your phone or desktop
2. Search for `@BotFather` (verified blue checkmark)
3. Send `/newbot`
4. Choose a display name — use something non-obvious, not "Boot AI Server" or "Claude Bot"
   (security through obscurity is not security, but there's no reason to advertise)
5. Choose a username (must end in `bot`) — e.g., `myutil_7x2_bot`
6. BotFather responds with a bot token like: `7123456789:AAH...rest-of-token`
7. **SAVE THIS TOKEN SECURELY** — treat it like a password

Additional BotFather settings:

```
/setprivacy  → Enable (bot only sees messages directed at it in groups)
/setjoingroups → Disable (bot cannot be added to groups)
```

### 4.2 Get Your Telegram User ID

1. Open Telegram
2. Search for `@userinfobot` (or `@getmyid_bot`)
3. Send any message to it
4. It replies with your numeric user ID (e.g., `123456789`)
5. **SAVE THIS ID** — this is the ONLY ID that will be authorized in Boot's allowlist

### 4.3 Verify Bot Token and User ID

Create a temporary test script to verify credentials. Run on the Mac Mini:

```bash
# One-time test — delete after verification
python3 -c "
import urllib.request, json, sys

TOKEN = 'YOUR_BOT_TOKEN_HERE'

# Test 1: Verify token is valid
url = f'https://api.telegram.org/bot{TOKEN}/getMe'
resp = json.loads(urllib.request.urlopen(url).read())
if resp['ok']:
    print(f'Bot verified: @{resp[\"result\"][\"username\"]}')
else:
    print('ERROR: Invalid token')
    sys.exit(1)

# Test 2: Check for messages (send a message to your bot first via Telegram)
url = f'https://api.telegram.org/bot{TOKEN}/getUpdates'
resp = json.loads(urllib.request.urlopen(url).read())
if resp['result']:
    user_id = resp['result'][-1]['message']['from']['id']
    print(f'Last message from user ID: {user_id}')
    print('Verify this matches YOUR user ID')
else:
    print('No messages yet — send a message to your bot via Telegram, then re-run')
"
```

Before running: send a message to your bot from your Telegram account.

Verification:
- Bot token resolves to correct bot username
- Your user ID from the message matches what @userinfobot reported

### 4.4 Test Unauthorized Access

Send a message to your bot from a DIFFERENT Telegram account (friend, secondary account).
Then re-run the getUpdates check — note the different user ID. The bot's security module
will reject any ID not in the allowlist, but verify now that you can distinguish IDs.

### 4.5 Create Directory Structure

```bash
# Boot source code (forked from linuz90/claude-telegram-bot)
mkdir -p ~/BootDrive/app/src/handlers

# Boot data (secrets, database, Claude auth) — bind-mounted as /data
mkdir -p ~/BootDrive/data/logs

# Boot workspace (mounted into container as /workspace)
mkdir -p ~/BootDrive/workspace/projects
```

### 4.6 Set Permissions

```bash
# BootDrive/data: only your user can access (contains secrets + Claude credentials)
chmod 700 ~/BootDrive/data
chmod 700 ~/BootDrive/data/logs

# BootDrive/app: readable, your code
chmod 755 ~/BootDrive/app

# BootDrive/workspace: container user is node (UID 1000), which matches host user
# No ACLs needed — same UID inside and outside the container
chmod 755 ~/BootDrive/workspace
```

### 4.7 Create Environment File

```bash
touch ~/BootDrive/data/.env
chmod 600 ~/BootDrive/data/.env
```

Edit `~/BootDrive/data/.env`:

```bash
# RALY Boot Configuration
# This file is chmod 600 and passed to the container via --env-file or compose

# Required
TELEGRAM_BOT_TOKEN=<your-bot-token-from-step-4.1>
TELEGRAM_ALLOWED_USERS=<your-user-id-from-step-4.2>

# Recommended
CLAUDE_WORKING_DIR=/workspace/projects
# OPENAI_API_KEY=sk-...  # For voice transcription (optional)

# Optional — rate limiting
RATE_LIMIT_ENABLED=true
RATE_LIMIT_REQUESTS=20
RATE_LIMIT_WINDOW=60

# Optional — extended thinking keywords
# THINKING_KEYWORDS=think,reason,analyze
# THINKING_DEEP_KEYWORDS=ultrathink

# Optional — audit logging
# AUDIT_LOG_PATH=/data/logs/audit.log
```

### 4.8 Verify Fork Source

The bot source at `~/BootDrive/app/` is a fork of
[linuz90/claude-telegram-bot](https://github.com/linuz90/claude-telegram-bot).

Expected source structure:

```
~/BootDrive/app/
├── Dockerfile           # Multi-stage build (Bun + Node 22)
├── package.json         # Dependencies (grammy, claude-agent-sdk, etc.)
├── bun.lockb            # Bun lockfile
├── .env.example         # Template for environment variables
└── src/
    ├── index.ts         # Entry point, handler registration
    ├── config.ts        # Env parsing, MCP loading, safety prompts
    ├── session.ts       # ClaudeSession class, streaming, persistence
    ├── security.ts      # Rate limiter, path validation, command safety
    ├── formatting.ts    # Markdown→HTML conversion for Telegram
    ├── types.ts         # Shared TypeScript types
    ├── utils.ts         # Audit logging, transcription, typing indicators
    └── handlers/
        ├── index.ts     # Handler exports
        ├── commands.ts  # /start, /new, /stop, /status, /resume, /restart
        ├── text.ts      # Text message handling
        ├── voice.ts     # Voice→text transcription (OpenAI)
        ├── audio.ts     # Audio file transcription
        ├── photo.ts     # Image analysis + media group buffering
        ├── document.ts  # PDF extraction, archives, routing
        ├── video.ts     # Video/video note handling
        ├── callback.ts  # Inline button handling (ask_user MCP)
        └── streaming.ts # StreamingState, status callbacks
```

Key dependencies (from `package.json`):
- `grammy` — Telegram bot framework
- `@anthropic-ai/claude-agent-sdk` — Claude Code CLI subprocess management
- `@modelcontextprotocol/sdk` — MCP server support
- `openai` — Voice transcription (Whisper)
- `zod` — Schema validation

### 4.9 Clean Up Test Script

Delete the verification script from step 4.3. Don't leave bot tokens in command history:

```bash
history -d $(history | grep "BOT_TOKEN\|TOKEN=" | awk '{print $1}') 2>/dev/null
# Or clear history entirely if preferred:
# history -c && history -w
```

## Verification Checklist

- [ ] Telegram bot created via BotFather, token saved securely
- [ ] Bot privacy enabled, group joining disabled
- [ ] Your Telegram user ID obtained and recorded
- [ ] Bot token verified working (getMe API call succeeded)
- [ ] Your user ID confirmed via getUpdates match
- [ ] Unauthorized user ID is different from yours (tested with different account)
- [ ] `~/BootDrive/app/src/` directory exists with TypeScript source files
- [ ] `~/BootDrive/data/` exists, permissions `700`, contains `.env` (permissions `600`)
- [ ] `~/BootDrive/data/logs/` exists, permissions `700`
- [ ] `~/BootDrive/workspace/projects/` exists
- [ ] `.env` populated with token and user ID
- [ ] `.env` is NOT in any git repo
- [ ] `package.json` has pinned dependencies
- [ ] Dockerfile present and builds successfully
- [ ] Test script deleted, bot token not in shell history

## Outputs for Downstream Phases

- Bot token → loaded from `.env` by container at startup
- User ID → used by `security.ts` allowlist
- Directory structure → Phase 5.x (all modules use these paths)
- `.env` path → passed to container via `--env-file` or compose `env_file:`
- Source code → baked into Docker image via `COPY` (edit on host, rebuild to deploy)

## Security Notes

- The bot token grants full control of the bot. Anyone with it can read messages
  sent to the bot and send messages as the bot. Guard it like a password.
- `.env` at `chmod 600` means only your user can read it. The container receives
  env vars at startup (not the file itself).
- The workspace directory is the accepted blast radius. Boot's data directory
  (`~/BootDrive/data/`) is also mounted but contains only state and credentials.
- Boot source code is baked into the image — not volume-mounted at runtime.
