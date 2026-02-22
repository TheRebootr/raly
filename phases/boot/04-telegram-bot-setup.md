# Phase 4: Telegram Bot Setup + Directory Structure

## Context

RALY is a hardened Mac Mini 2018 running Omarchy 3.x (Arch Linux). The Boot harness
is a custom Python process (~500-800 lines) that receives commands via Telegram polling,
validates them through a security module, and delegates work to Claude Code CLI inside
ephemeral Docker containers. This phase creates the Telegram bot identity, verifies
credentials, and sets up the directory structure for Boot's code, data, and workspace.

Architecture summary:
- `~/BootDrive/app/` — Boot's source code (the harness)
- `~/BootDrive/data/` — Boot's brain (SQLite, secrets, logs) — NEVER exposed to containers
- `~/BootDrive/workspace/` — THE ONLY directory containers can access, per-project subdirs

## Prerequisites

- Phase 0 complete: machine verified
- Telegram account exists (or will be created)
- Docker running with userns-remap from Phase 2 (needed for workspace permissions)

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
Then re-run the getUpdates check — note the different user ID. The harness (Phase 5.1)
will reject any ID not in the allowlist, but verify now that you can distinguish IDs.

### 4.5 Create Directory Structure

```bash
# Boot source code
mkdir -p ~/BootDrive/app/boot

# Boot data (secrets, database, logs) — NEVER in workspace, NEVER in containers
mkdir -p ~/BootDrive/data/logs

# Boot workspace (mounted into containers, per-project)
mkdir -p ~/BootDrive/workspace/scratch
```

### 4.6 Set Permissions

```bash
# BootDrive/data: only your user can access (contains secrets)
chmod 700 ~/BootDrive/data
chmod 700 ~/BootDrive/data/logs

# BootDrive/app: readable, your code
chmod 755 ~/BootDrive/app

# BootDrive/workspace: needs to be accessible by both your user and Docker's remapped user
# With userns-remap, container root maps to host UID in the dockremap subuid range.
# Check the mapped UID:
REMAP_UID=$(grep dockremap /etc/subuid | cut -d: -f2)
echo "Docker remapped UID starts at: $REMAP_UID"

# Option A: Use ACLs for dual access (preferred)
sudo pacman -S acl --needed
# Allow your user AND the remapped UID to read/write workspace
setfacl -R -m u:${REMAP_UID}:rwx ~/BootDrive/workspace
setfacl -R -d -m u:${REMAP_UID}:rwx ~/BootDrive/workspace

# Option B: Simpler but less precise — make workspace world-readable
# chmod -R 777 ~/BootDrive/workspace
# (Not recommended — too permissive)
```

### 4.7 Create Secrets File

```bash
# Create config.env with restricted permissions
touch ~/BootDrive/data/config.env
chmod 600 ~/BootDrive/data/config.env
```

Edit `~/BootDrive/data/config.env`:

```bash
# RALY Boot Configuration
# This file is chmod 600 and NEVER enters a container or repo

TELEGRAM_BOT_TOKEN=<your-bot-token-from-step-4.1>
TELEGRAM_ALLOWED_USERS=<your-user-id-from-step-4.2>

# Claude Code CLI auth — if using CLI, authenticate interactively first:
#   claude auth login
# If using API key instead:
# ANTHROPIC_API_KEY=<your-api-key>

# Rate limiting
RATE_LIMIT_REQUESTS_PER_MINUTE=10
RATE_LIMIT_MAX_COST_PER_HOUR=100

# Workspace
BOOT_WORKSPACE=/home/<your-username>/BootDrive/workspace
BOOT_DATA=/home/<your-username>/BootDrive/data
```

### 4.8 Create Source File Stubs

```bash
touch ~/BootDrive/app/boot/__init__.py
touch ~/BootDrive/app/boot/main.py
touch ~/BootDrive/app/boot/config.py
touch ~/BootDrive/app/boot/security.py
touch ~/BootDrive/app/boot/telegram.py
touch ~/BootDrive/app/boot/executor.py
touch ~/BootDrive/app/boot/session.py
touch ~/BootDrive/app/boot/memory.py
touch ~/BootDrive/app/requirements.txt
touch ~/BootDrive/app/Dockerfile.sandbox
touch ~/BootDrive/app/CLAUDE.md
```

### 4.9 Create requirements.txt

```bash
cat > ~/BootDrive/app/requirements.txt << 'EOF'
python-telegram-bot==21.6
aiosqlite==0.20.0
EOF
```

That's 2 runtime dependencies (3rd is `sqlite3` from stdlib). Pin exact versions.
Check for latest stable versions before finalizing.

### 4.10 Initialize Git Repository (optional but recommended)

```bash
cd ~/BootDrive/app
git init
cat > .gitignore << 'EOF'
__pycache__/
*.pyc
*.pyo
.env
config.env
*.db
venv/
.venv/
EOF
git add .
git commit -m "Initial Boot harness structure"
```

This is your backup mechanism. Optionally push to a private repo.

### 4.11 Set Up Python Virtual Environment

```bash
cd ~/BootDrive/app
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 4.12 Clean Up Test Script

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
- [ ] `~/BootDrive/app/boot/` directory exists with all stub files
- [ ] `~/BootDrive/data/` exists, permissions `700`, contains `config.env` (permissions `600`)
- [ ] `~/BootDrive/data/logs/` exists, permissions `700`
- [ ] `~/BootDrive/workspace/scratch/` exists
- [ ] Workspace permissions allow Docker remapped UID access (ACL or permissions set)
- [ ] `config.env` populated with token, user ID, and paths
- [ ] `config.env` is NOT in any git repo
- [ ] `requirements.txt` has pinned dependencies
- [ ] Python venv created and dependencies installed
- [ ] Git repo initialized in `~/BootDrive/app/` with proper `.gitignore`
- [ ] Test script deleted, bot token not in shell history

## Outputs for Downstream Phases

- Bot token → Phase 5.2 (config.py loads it)
- User ID → Phase 5.1 (security.py allowlist)
- Directory structure → Phase 5.x (all modules use these paths)
- `config.env` path → Phase 5.2 (config.py reads from this)
- Workspace ACL setup → Phase 5.4 (executor.py mounts workspace into containers)
- requirements.txt → Phase 5.x (dependency list)
- venv → Phase 5.x (development environment)

## Security Notes

- The bot token grants full control of the bot. Anyone with it can read messages
  sent to the bot and send messages as the bot. Guard it like a password.
- `config.env` at `chmod 600` means only your user can read it. The Boot service
  (Phase 5) must run as your user to access it.
- The workspace directory is the ONLY place containers can write. Boot's data and
  source code are never mounted into containers.

## Internet Validation Instruction

Before executing this phase, perform a web search for:
- "python-telegram-bot latest version PyPI 2025 2026"
- "aiosqlite latest version PyPI 2025 2026"
- "Telegram BotFather create bot 2025"
- "Docker userns-remap volume permissions ACL"
- "Telegram bot token security best practices"

Verify that:
1. `python-telegram-bot` version 21.6 is current (or find the latest stable)
2. `aiosqlite` version 0.20.0 is current (or find the latest stable)
3. BotFather workflow hasn't changed
4. ACL approach for Docker volume permissions is still recommended
5. No new Telegram bot security advisories
