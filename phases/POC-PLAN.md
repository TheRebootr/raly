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
container. Every key pre-answers an interactive dialog that would hang in a non-interactive
context:

```bash
cat > ~/boot-data/.claude.json << 'EOF'
{
  "hasCompletedOnboarding": true,
  "hasAcknowledgedDangerousPermissions": true,
  "hasTrustDialogAccepted": true,
  "hasCompletedProjectOnboarding": true,
  "shiftEnterKeyBindingInstalled": true
}
EOF
```

The `--dangerously-skip-permissions` flag requires a separate setting in `settings.json`
(not `.claude.json`) — without it the CLI silently hangs waiting for interactive acceptance
([#25503](https://github.com/anthropics/claude-code/issues/25503)):

```bash
echo '{"skipDangerousModePermissionPrompt": true}' > ~/.claude/settings.json
```

Since `~/.claude/` is bind-mounted rw into the container, this file will be visible at
`/home/node/.claude/settings.json`.

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
DATABASE_URL=sqlite:////data/bot.db
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

**Note:** `--read-only` is not used here (or in default production config) because agentic
use cases need runtime package installation. It's available as an optional hardening flag
for locked-down deployments — see Phase 2, "Optional: `--read-only` mode".

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
  -v ~/boot-data/.claude.json:/home/node/.claude.json \
  boot:latest \
  bash
```

**Credential mounts explained:**

- `~/.claude:/home/node/.claude` — OAuth tokens from `claude /login`. Mounted read-write
  so the CLI can refresh expired access tokens (they expire every 8-12 hours). Both host
  and container use UID 1000, so permissions align.
- `~/boot-data/.claude.json:/home/node/.claude.json` — Global state (onboarding bypass,
  startup counters, etc.). Must be **read-write** — the CLI writes to this file at startup
  and silently hangs if it can't.

You're now inside the Boot container. Everything below happens inside.

### 5.2 Install Poetry and Clone the Repo

The bot requires Poetry 2.x (`poetry-core>=2.0.0` build system).

**Why not pip?** The `boot:latest` image has a Python venv on PATH (`/opt/boot-venv/bin`),
and venv pip rejects `--user` installs. System pip isn't in the runtime image either. The
official Poetry installer works cleanly — it only needs `python3` and `curl`.

```bash
curl -sSL https://install.python-poetry.org | python3 -
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

With the POC running, verify the security boundary is real. Each test maps to a
specific container flag or architectural decision from Phase 2/CLAUDE.md.

### 6.1 Filesystem Isolation

Tests: container can't see host filesystem. Only `/workspace` and `/data` (bind mounts)
plus the container's own Debian filesystem are visible.

From inside the container:

```bash
# Container's /etc/shadow exists but is unreadable as non-root
# NOTE: `ls` will succeed (it only needs directory permission on /etc/).
# The real test is reading the file:
cat /etc/shadow             # → Permission denied
cat /etc/hostname           # → container ID (e.g. "a1b2c3d4e5f6"), not host hostname

# Host filesystem is invisible
ls /home/therebootr/        # → No such file or directory
ls /var/run/docker.sock     # → No such file or directory

# Only mounted volumes show host content
ls /workspace               # → your project files from ~/boot-workspace
ls /data                    # → config/SQLite from ~/boot-data
```

**Why this matters:** The container's `/etc/shadow` is the _image's_ shadow file (Debian
system accounts), not the host's. The host filesystem is completely invisible — this is
the core isolation guarantee.

### 6.2 Privilege & Capability Restrictions

Tests: `--cap-drop ALL`, `--security-opt=no-new-privileges`, no root access.

```bash
# User verification
whoami                      # → node
id                          # → uid=1000(node) gid=1000(node)
sudo ls                     # → command not found (no sudo in image)

# --cap-drop ALL: no Linux capabilities at all
# These all require capabilities that have been dropped:
mount -t tmpfs none /tmp    # → permission denied (needs CAP_SYS_ADMIN)
ip link set lo down         # → permission denied (needs CAP_NET_ADMIN)
mknod /dev/null2 c 1 3     # → permission denied (needs CAP_MKNOD)
chown root /workspace       # → permission denied (needs CAP_CHOWN)

# --security-opt=no-new-privileges: setuid/setgid binaries can't escalate
# Even if a setuid binary existed, it couldn't gain privileges.
# Verify the flag is active:
grep NoNewPrivs /proc/self/status
# → NoNewPrivs: 1
```

**What --cap-drop ALL means:** Linux capabilities are fine-grained root powers (mount
filesystems, change network config, load kernel modules, etc.). Dropping all of them
means the container process has zero elevated permissions, even if it somehow became
UID 0.

### 6.3 Init Process (--init)

Tests: PID 1 is tini (Docker's init), not your shell. Without `--init`, your shell
becomes PID 1 and can't reap zombie processes — Claude Code CLI spawns subprocesses
that would accumulate as zombies over time.

```bash
cat /proc/1/comm
# → "docker-init" (Docker's bundled tini), NOT "bash"
# The slim image has no `ps` command — /proc/1/comm is the direct way to check.
```

### 6.4 Resource Limits

Tests: `--memory=4g`, `--memory-swap=6g`, `--cpus=4`, `--pids-limit=512`.

From inside the container (Arch uses cgroup v2 by default):

```bash
# Memory limit (cgroup v2)
cat /sys/fs/cgroup/memory.max
# → 4294967296 (4GB)

# Swap limit = memory-swap minus memory = 2GB swap
cat /sys/fs/cgroup/memory.swap.max
# → 2147483648 (2GB)

# PID limit (cgroup v2)
cat /sys/fs/cgroup/pids.max
# → 512

# CPU limit (cgroup v2): --cpus=4 → 400000 per 100000 period
cat /sys/fs/cgroup/cpu.max
# → 400000 100000
```

### 6.5 Volume Scope (Blast Radius)

Tests: the container can only write to `/workspace` and `/data`. These are the accepted
blast radius — if compromised, only these directories are affected.

From inside the container:

```bash
# Can write to mounted volumes
touch /workspace/scope-test.txt   # → succeeds
touch /data/scope-test.txt        # → succeeds

# Can write to container filesystem (--read-only is not used by default)
# This is expected — agents need to install packages at runtime.
# If using optional --read-only mode, this would fail.
touch /opt/test.txt               # → succeeds (writable rootfs)
```

From the host — verify only the expected directories are affected:

```bash
ls ~/boot-workspace/scope-test.txt    # → exists, owned by UID 1000
ls ~/boot-data/scope-test.txt         # → exists, owned by UID 1000

# Credential mount is read-write (needed for token refresh)
ls -la ~/.claude/                     # → confirm files not unexpectedly modified
```

**Volume scope is the blast radius:** Even without `--read-only`, the container boundary
still isolates. The accepted blast radius is `/workspace` + `/data` (bind mounts) plus
the ephemeral container filesystem (lost on recreation). Host files outside these mounts
are invisible.

### 6.6 File Permission Match

From inside the container, create a file:

```bash
touch /workspace/permission-test.txt
```

From the host, verify ownership:

```bash
ls -la ~/boot-workspace/permission-test.txt
# → should be owned by your user (UID 1000), not root
```

This confirms UID 1000 matching works. No permission hell between host and container.

### 6.7 Network Verification

Tests: full outbound access (by design), no inbound exposure, no listening services.

**How container networking works here:** Docker's default bridge gives the container its
own network namespace with a private IP (172.17.0.x). Outbound connections work via NAT.
Inbound connections from the external network require explicit `-p` port mapping — which
we don't use. There is no firewall or sshd inside the container (the slim image doesn't
ship them, and `--cap-drop ALL` prevents installing firewall rules anyway).

From inside the container:

```bash
# Outbound works (required for Telegram + Anthropic APIs)
curl -s -o /dev/null -w "%{http_code}" https://api.telegram.org
# → 200 or 301
curl -s -o /dev/null -w "%{http_code}" https://api.anthropic.com
# → 200 or 403 (no auth header, but DNS + TLS works)

# Verify no services are listening inside the container
# (slim image has no ss/netstat — read /proc/net directly)
cat /proc/net/tcp /proc/net/tcp6 2>/dev/null
# → only the header line, or connections YOUR bot opened (outbound to Telegram)
# → no entries in LISTEN state (0A in the "st" column = LISTEN)
# If the bot is running, you'll see ESTABLISHED connections (01) — that's expected.

# No sshd, no firewall — verify the tools don't exist
which sshd                  # → not found (not in the image)
which iptables              # → not found (not in the image)
which ufw                   # → not found (not in the image)

# Can the container reach the host? (document, don't fix)
curl -s --connect-timeout 3 -o /dev/null -w "%{http_code}" http://172.17.0.1:53317
# host bridge IP, LocalSend port — result depends on host UFW rules
```

From the host — verify no ports are published:

```bash
docker port boot-poc
# → should be empty (no -p flags were used)
```

**No inbound exposure by design:** No `-p` flags = no port mappings. The bot communicates
purely via outbound long-polling to the Telegram API. Even if a process inside the
container listens on a port, it's only reachable from the host via the container's
bridge IP — not from the external network.

### 6.8 Kill Switch Test

From the host:

```bash
docker stop boot-poc
```

Verify: the bot stops responding on Telegram immediately. Everything inside the
container is frozen. No processes survive on the host.

```bash
docker start boot-poc
# Container restarts, but the bot process does NOT auto-start.
# This is expected — the POC runs the bot interactively.
# You need to exec in and restart it manually:
docker exec -it boot-poc bash
# then re-run the bot startup commands from Step 5.4
```

**Production difference:** The systemd unit + container entrypoint will handle
auto-restart. The POC validates the kill switch, not the recovery.

---

## Step 7: Stress Testing

These tests verify the architecture survives real-world disruptions. Run them with
the POC bot actively responding to Telegram messages.

### 7.1 Docker Daemon Restart (simulates omarchy-update)

`omarchy-update` runs pacman which may restart the Docker daemon. Verify data survives.

```bash
# Send a message to the bot, confirm it's responding
sudo systemctl restart docker
```

Check:

- Container status: `docker ps -a --filter name=boot-poc`
  (Expected: Exited — no `--restart` flag in POC)
- Is SQLite data intact? `ls -la ~/boot-data/bot.db`
- Manual recovery: `docker start boot-poc && docker exec -it boot-poc bash`
  then re-run bot startup from Step 5.4

### 7.2 DNS Breakage Test

Docker's embedded DNS can break when the host network changes. This is a known issue
on this machine (see CLAUDE.md: "Docker DNS breaks on host network changes").

With the POC bot running:

1. Toggle the network interface on the Mac Mini (disconnect/reconnect WiFi or Ethernet)
2. From inside the container: `curl --connect-timeout 5 https://api.anthropic.com/`
3. Does DNS resolve? If not, this confirms the Docker DNS bug
4. Restart the container: `docker restart boot-poc`
5. Does DNS work after container restart?

**Document the result.** This determines how aggressive the health-check/auto-restart
needs to be in production. If DNS breaks on network change but recovers on container
restart, a periodic health check with `docker restart` is sufficient.

### 7.3 Mid-Task Kill

Tests graceful degradation when the container is killed during active Claude processing.

1. Send a complex message to the bot via Telegram (something that takes 30+ seconds)
2. While Claude is processing: `docker stop boot-poc`
3. Check data integrity: `sqlite3 ~/boot-data/bot.db ".tables"` (no corruption)
4. Check workspace: `ls ~/boot-workspace/` (no partial/corrupted files)
5. Start the container and verify the bot recovers: start it, re-run bot, send a message

**What we're validating:** SQLite handles interrupted writes via WAL journaling.
Workspace files may be partially written (acceptable — same as killing any editor).
The bot should start cleanly without needing manual cleanup.

### 7.4 Reboot Test

Full machine reboot — validates the entire host stack comes back.

```bash
sudo reboot
```

After reboot, verify from a Tailscale SSH session or local terminal:

```bash
systemctl is-active docker              # → active
tailscale status                        # → connected
docker ps -a --filter name=boot-poc     # → Exited (no restart policy)
```

Manual recovery for POC:

```bash
docker start boot-poc
docker exec -it boot-poc bash
# re-run bot startup from Step 5.4
```

**Production difference:** The systemd unit (Phase 5.x) will auto-start the container
after boot. The POC only validates that Docker and Tailscale survive the reboot.

---

## Decision Point

After completing all steps, you have concrete answers to:

| Question                                            | Expected Answer                                  |
| --------------------------------------------------- | ------------------------------------------------ |
| Does the container boundary actually isolate?       | Yes — can't see host filesystem                  |
| Are Linux capabilities dropped?                     | Yes — no mount, no net config, no raw sockets    |
| Can processes escalate privileges?                  | No — NoNewPrivs=1, no sudo, no setuid escalation |
| Is PID 1 an init process?                           | Yes — tini handles zombie reaping + signals      |
| Are resource limits enforced?                       | Yes — 4GB mem, 2GB swap, 4 CPUs, 512 PIDs        |
| Is the blast radius limited to volumes?             | Yes — only /workspace and /data are writable     |
| Do file permissions work across the boundary?       | Yes — UID 1000 matches                           |
| Does Claude Code CLI work inside the container?     | Yes — subscription auth via mounted credentials  |
| Does a real Telegram bot work inside the container? | Yes — RichardAtCT's bot runs                     |
| Are any ports exposed inbound?                      | No — bot uses outbound polling only              |
| What happens on daemon restart?                     | Container dies, needs restart policy/systemd     |
| Does DNS break on network change?                   | Probably yes — need health check                 |
| Is the kill switch real?                            | Yes — `docker stop` kills everything             |

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
