# Omarchy Layer Execution + POC Validation Plan

## Context

We've redesigned RALY from "Boot runs on host, sub-containers per task" to "Boot lives
inside a single long-lived Docker container." The Omarchy layer (Phases 0-3) establishes the
secure host baseline and builds the Boot container. Before writing any RALY application
code, we validate the security boundary by dropping RichardAtCT's claude-code-telegram into
the container as a working POC.

**Goal**: A running Telegram bot inside a Docker container on the hardened host, proving the
architecture works end-to-end before building our own implementation.

---

## Prerequisites (gather before starting)

You need these things ready before execution:

- [ ] **Claude Pro or Max subscription** — or an Anthropic API key from console.anthropic.com.
      The POC is configured for subscription auth (CLI subprocess mode). If using an API key
      instead, see the "API key alternative" note in Step 4.2.
- [ ] **Telegram bot token** — create via @BotFather on Telegram (`/newbot`). Record the token.
- [ ] **Your Telegram user ID** — message @userinfobot on Telegram, it replies with your numeric ID.
- [ ] **Telegram bot username** — the username you chose in BotFather (without the @).

---

## Steps 1-3: COMPLETE

Phases 0-3 are done. The host is hardened, the Boot container image is built, trivy
scanned, lifecycle tested, and Tailscale SSH is configured with ACLs. See individual
phase docs for verification checklists.

Remaining from Phase 2: systemd unit creation — deferred to Phase 5.x (no app code yet).

---

## Step 4: POC Preparation

### 4.1 Create Telegram Bot

If not done yet:

1. Message @BotFather on Telegram → `/newbot`
2. Choose a name and username
3. Record the bot token
4. Message @userinfobot → record your numeric user ID

### 4.2 Authenticate Claude Code CLI on the Host

The POC uses your Claude subscription (Pro/Max) instead of an API key. The CLI
authenticates via OAuth in a browser, so this must happen on the host — not inside
the container.

```bash
# Install Claude Code CLI on the host if not already present
npm install -g @anthropic-ai/claude-code@latest

# Authenticate with your Claude subscription
claude /login
# Opens a browser → log in with your claude.ai account → authorize
```

Verify it worked:

```bash
claude -p "say hello" --dangerously-skip-permissions
# Should get a response using your subscription
```

Credentials are stored in `~/.claude/.credentials.json`. These will be mounted into
the container in Step 5.1.

**Important:** Make sure `ANTHROPIC_API_KEY` is NOT set in your shell environment.
If set, the CLI silently uses the API key (pay-per-token) instead of your subscription.

```bash
echo $ANTHROPIC_API_KEY    # should be empty
```

Create the onboarding bypass file so the CLI doesn't prompt interactively inside the
container:

```bash
echo '{"hasCompletedOnboarding": true}' > ~/boot-data/.claude.json
```

### 4.3 Create POC Environment File

Create `~/boot-data/.env.poc`:

```bash
TELEGRAM_BOT_TOKEN=<your-bot-token>
TELEGRAM_BOT_USERNAME=<your-bot-username>
APPROVED_DIRECTORY=/workspace
ALLOWED_USERS=<your-telegram-user-id>
USE_SDK=false
AGENTIC_MODE=true
DEBUG=true
DATABASE_URL=sqlite:///data/bot.db
CLAUDE_MAX_TURNS=10
CLAUDE_TIMEOUT_SECONDS=300
```

**Why `USE_SDK=false`:** The `claude-agent-sdk` Python package (SDK mode) only supports
API key auth — it cannot use subscription credentials (confirmed by Anthropic, GitHub
# 5891). CLI subprocess mode (`USE_SDK=false`) invokes the `claude` binary which fully
supports subscription auth via the mounted OAuth credentials.

**No `ANTHROPIC_API_KEY`:** Intentionally omitted. The CLI will use the subscription
credentials mounted from `~/.claude/` instead.

**API key alternative:** If you later get an API key, set `USE_SDK=true` and add
`ANTHROPIC_API_KEY=<your-key>` to use the Python SDK directly. This avoids the CLI
subprocess overhead and doesn't require mounting credentials.

**Security note**: This file contains secrets. It lives on the `boot-data` volume
(the accepted blast radius). Don't commit it to git.

```bash
chmod 600 ~/boot-data/.env.poc
```

---

## Step 5: POC Execution — RichardAtCT's Bot in Your Container

This is where we prove the architecture works.

### 5.1 Start the Container

**Note:** The POC container intentionally omits `--read-only` and `noexec` tmpfs flags
because it needs interactive package installation (pip, poetry, git clone). Production
uses the full hardened flags from Phase 2 (`--read-only`, `--cap-drop ALL`,
`noexec` tmpfs, etc.). The boundary test in Step 2.6 already validated those flags.

```bash
docker run -it \
  --name boot-poc \
  --init \
  --memory=4g \
  --memory-swap=6g \
  --cpus=4 \
  --pids-limit=512 \
  --user 1000:1000 \
  --security-opt=no-new-privileges \
  --cap-drop ALL \
  -v ~/boot-workspace:/workspace \
  -v ~/boot-data:/data \
  -v ~/.claude:/home/node/.claude \
  -v ~/boot-data/.claude.json:/home/node/.claude.json:ro \
  boot:latest \
  bash
```

**Credential mounts explained:**

- `~/.claude:/home/node/.claude` — OAuth tokens from `claude /login`. Mounted read-write
  so the CLI can refresh expired access tokens (they expire every 8-12 hours). Both host
  and container use UID 1000, so permissions align.
- `~/boot-data/.claude.json:/home/node/.claude.json:ro` — Onboarding bypass. Prevents the
  CLI from launching an interactive setup wizard inside the container.

You're now inside the Boot container. Everything below happens inside.

### 5.2 Install Poetry and Clone the Repo

The bot requires Poetry 2.x (`poetry-core>=2.0.0` build system).

```bash
pip3 install --user "poetry>=2.0"
export PATH="$HOME/.local/bin:$PATH"
poetry --version    # verify 2.x

cd /workspace
git clone https://github.com/RichardAtCT/claude-code-telegram.git poc-bot
cd poc-bot
```

### 5.3 Install Dependencies

Note: `--no-dev` is deprecated in Poetry 2.x — use `--without dev`.

```bash
poetry install --without dev
```

If Poetry has issues with the `node` user home directory:

```bash
export POETRY_VIRTUALENVS_IN_PROJECT=true
poetry install --without dev
```

**Heads up:** The bot has grown since initial planning. It now pulls in `fastapi`,
`uvicorn`, `apscheduler`, `structlog`, `claude-agent-sdk`, and more. Expect a longer
install than a minimal bot would need. This is fine — it's a POC, not production.

### 5.4 Configure and Run

**Auth mode:** With `USE_SDK=false`, the bot spawns `claude` as a subprocess. The CLI
picks up your subscription credentials from the mounted `~/.claude/` directory. No API
key needed.

```bash
# Load env vars
export $(grep -v '^#' /data/.env.poc | xargs)

# Create the approved directory
mkdir -p /workspace/projects

# Verify Claude Code CLI works with your subscription (network + auth test)
claude -p "say hello" --dangerously-skip-permissions
# Should get a response — proves subscription auth works from inside the container
# If this fails with auth errors, re-run `claude /login` on the host

# Verify ANTHROPIC_API_KEY is NOT set (would override subscription)
echo $ANTHROPIC_API_KEY    # should be empty

# Run the bot
poetry run claude-telegram-bot --debug
```

### 5.5 Test from Telegram

Open Telegram and message your bot:

1. Send "hello" → should get a response
2. Send "what directory are you in?" → should report `/workspace/projects` or similar
3. Send "create a file called test.txt with hello world" → should create it
4. Verify from host: `ls ~/boot-workspace/projects/test.txt`

**If this works — your security boundary is validated with a real workload.**

### 5.6 Keep the Bot Running (optional)

If you want to leave it running for extended testing, detach from the container
(Ctrl+P, Ctrl+Q) or run it in the background:

```bash
# From outside the container:
docker exec -d boot-poc bash -c 'cd /workspace/poc-bot && export $(grep -v "^#" /data/.env.poc | xargs) && poetry run claude-telegram-bot'
```

---

## Step 6: Boundary Testing

With the POC running, verify the security boundary is real.

### 6.1 Filesystem Isolation

From inside the container:

```bash
ls /etc/shadow              # → permission denied (non-root)
ls /home/<your-username>/        # → no such directory (host fs not visible)
cat /etc/hostname           # → container ID, not host hostname
ls /var/run/docker.sock     # → no such file
```

**The container can only see**: `/workspace` (your projects), `/data` (config/SQLite),
and the container's own Debian filesystem. Nothing from the host.

### 6.2 Resource Limits

From inside the container (Arch uses cgroup v2 by default):

```bash
# Memory limit check (cgroup v2)
cat /sys/fs/cgroup/memory.max
# → 4294967296 (4GB)

# PID limit check (cgroup v2)
cat /sys/fs/cgroup/pids.max
# → 512
```

### 6.3 User Verification

```bash
whoami     # → node
id         # → uid=1000(node) gid=1000(node)
sudo ls    # → command not found (no sudo)
```

### 6.4 File Permission Match

From inside the container, create a file:

```bash
touch /workspace/permission-test.txt
```

From the host, verify ownership:

```bash
ls -la ~/boot-workspace/permission-test.txt
# → should be owned by your user (UID 1000), not root
```

This confirms UID 1000 matching works. No permission hell.

### 6.5 Network Verification

From inside the container:

```bash
# Outbound works (by design)
curl -s -o /dev/null -w "%{http_code}" https://api.telegram.org
curl -s -o /dev/null -w "%{http_code}" https://api.anthropic.com

# Can the container reach the host? (document the result)
curl -s -o /dev/null -w "%{http_code}" http://172.17.0.1:53317 2>&1
# host bridge IP, LocalSend port — may or may not work depending on UFW
```

### 6.6 Kill Switch Test

From the host:

```bash
docker stop boot-poc
```

Verify: the bot stops responding on Telegram immediately. Everything inside the
container is frozen. No processes survive on the host.

```bash
docker start boot-poc
# Bot should come back (if you set up auto-restart for the bot process)
```

---

## Step 7: Stress Testing

### 7.1 Docker Daemon Restart (simulates omarchy-update)

With the POC bot running:

```bash
sudo systemctl restart docker
```

Check:

- Does the container come back? (No — we didn't set `--restart` flag for the POC.
  The systemd unit in production would handle this.)
- Is SQLite data intact in `~/boot-data/`?

### 7.2 DNS Breakage Test

With the POC bot running:

1. Disconnect WiFi on the Mac Mini (or toggle the interface)
2. Reconnect WiFi
3. From inside the container: `curl https://api.anthropic.com/`
4. Does DNS resolve? If not, this confirms the Docker DNS bug.
5. Restart the container: `docker restart boot-poc`
6. Does DNS work after container restart?

**Document the result.** This tells you how aggressive your health-check/auto-restart
needs to be in production.

### 7.3 Mid-Task Kill

1. Send a long message to the bot via Telegram (something that takes 30+ seconds)
2. While Claude is processing: `docker stop boot-poc`
3. Check: is there any corruption in `~/boot-data/`?
4. Start the container again and check if the bot recovers cleanly

### 7.4 Reboot Test

```bash
sudo reboot
```

After reboot:

- Is Docker running? (`systemctl is-active docker`)
- Is Tailscale running? (`tailscale status`)
- If you had `--restart=unless-stopped` (or the systemd unit), does Boot come back?

---

## Decision Point

After completing all steps, you have concrete answers to:

| Question                                            | Expected Answer                                |
| --------------------------------------------------- | ---------------------------------------------- |
| Does the container boundary actually isolate?       | Yes — can't see host filesystem                |
| Do file permissions work across the boundary?       | Yes — UID 1000 matches                         |
| Does Claude Code CLI work inside the container?     | Yes — subscription auth via mounted credentials |
| Does a real Telegram bot work inside the container? | Yes — RichardAtCT's bot runs                   |
| What happens on daemon restart?                     | Container dies, comes back with restart policy |
| Does DNS break on network change?                   | Probably yes — need health check               |
| Is the kill switch real?                            | Yes — `docker stop` kills everything           |

**If all answers match expectations**: proceed to `boot/` phases and build your
own implementation on this validated foundation.

**If something surprises you**: document it, adjust the architecture, re-test before
building on a shaky foundation.

---

## Cleanup After POC

Once validated and ready to move on:

```bash
docker stop boot-poc && docker rm boot-poc
rm -rf ~/boot-workspace/poc-bot       # remove RichardAtCT's code
rm ~/boot-data/.env.poc               # remove POC config
# Keep ~/boot-workspace, ~/boot-data, ~/boot-src — these are production dirs
```

The `boot:latest` image stays — it's your production image. The systemd unit is
enabled and waiting for your own Boot application code (Phase 5.x).
