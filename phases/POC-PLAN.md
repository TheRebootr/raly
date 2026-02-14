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

- [ ] **Anthropic API key** — from console.anthropic.com. This is for the `ANTHROPIC_API_KEY`
  env var. Note: API key means pay-as-you-go billing, not subscription credits.
- [ ] **Telegram bot token** — create via @BotFather on Telegram (`/newbot`). Record the token.
- [ ] **Your Telegram user ID** — message @userinfobot on Telegram, it replies with your numeric ID.
- [ ] **Telegram bot username** — the username you chose in BotFather (without the @).

---

## Step 1: Complete Phase 0-1 Gaps

Phases 0-1 are done except two items that were skipped. Both are cheap insurance.

### 1.1 Core Dump Restrictions (Phase 1.6 — was skipped)

Core dumps can leak secrets (API keys, bot tokens) from process memory.

Add to `/etc/security/limits.conf`:
```
* hard core 0
* soft core 0
```

Create `/etc/systemd/coredump.conf.d/disable.conf`:
```ini
[Coredump]
Storage=none
ProcessSizeMax=0
```

### 1.2 Kernel Module Blacklist (Phase 1.7 — was skipped)

Create `/etc/modprobe.d/raly-blacklist.conf`:
```
blacklist cramfs
blacklist hfs
blacklist hfsplus
blacklist dccp
blacklist sctp
blacklist rds
blacklist tipc
```

These are uncommon filesystems and network protocols. Reduces kernel attack surface.
No USB/thunderbolt blacklisting — desktop use.

---

## Step 2: Execute Phase 2 — Build the Boot Container

### 2.1 Docker Service

Verify Docker is enabled and healthy. This should already be the case from Phase 0.

```bash
sudo systemctl enable docker.service
systemctl is-enabled docker.service   # → enabled
systemctl is-active docker.service    # → active
```

Verify daemon.json is untouched (Omarchy's):
```bash
cat /etc/docker/daemon.json
# Should show: log-driver, log-opts, dns, bip — nothing else
```

Verify socket permissions:
```bash
ls -la /var/run/docker.sock
# → srw-rw---- 1 root docker
```

### 2.2 Install trivy

```bash
sudo pacman -S trivy --needed
```

### 2.3 Create Host Directories

```bash
mkdir -p ~/boot-workspace ~/boot-data ~/boot-src
chmod 750 ~/boot-workspace ~/boot-data ~/boot-src
```

These will be bind-mounted into the container. `boot-workspace` is the blast radius.

### 2.4 Write the Dockerfile

Create `~/boot-src/Dockerfile`:

```dockerfile
FROM node:22-bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-venv \
    git \
    build-essential \
    curl \
    ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Claude Code CLI
RUN npm install -g @anthropic-ai/claude-code@latest

# node user already exists at UID 1000 in node: images
RUN mkdir -p /workspace /data /app && chown node:node /workspace /data /app

USER node
WORKDIR /app

CMD ["bash"]
```

**Why `node:22-bookworm-slim`**: Debian Bookworm (LTS 2028), glibc (no musl issues),
Anthropic uses this themselves. `node` user is UID 1000 — matches your host user.

### 2.5 Build and Scan

```bash
cd ~/boot-src
docker build -t boot:latest .
docker inspect boot:latest --format '{{.Id}}'    # record digest
trivy image boot:latest
```

Review trivy output. Document any HIGH/CRITICAL CVEs as accepted risk if they're
in base packages you can't control.

### 2.6 Test Container Lifecycle

This is the critical test — run the container with ALL production flags and verify
everything works:

```bash
docker run -d \
  --name boot-test \
  --init \
  --memory=4g \
  --memory-swap=6g \
  --cpus=4 \
  --pids-limit=512 \
  --user 1000:1000 \
  --security-opt=no-new-privileges \
  --cap-drop ALL \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=512m \
  --tmpfs /home/node:rw,noexec,nosuid,size=256m \
  -v ~/boot-workspace:/workspace \
  -v ~/boot-data:/data \
  boot:latest \
  sleep infinity
```

**Verify from outside:**
```bash
docker inspect boot-test --format '{{.HostConfig.Memory}}'
# → 4294967296 (4GB in bytes)
docker inspect boot-test --format '{{.HostConfig.SecurityOpt}}'
# → [no-new-privileges]
docker inspect boot-test --format '{{.HostConfig.CapDrop}}'
# → [ALL]
docker inspect boot-test --format '{{.HostConfig.ReadonlyRootfs}}'
# → true
```

**Verify from inside:**
```bash
docker exec boot-test whoami                    # → node
docker exec boot-test claude --version          # → Claude Code CLI version
docker exec boot-test python3 --version         # → Python 3.x
docker exec boot-test ls /var/run/docker.sock   # → No such file
docker exec boot-test touch /usr/test 2>&1      # → Read-only file system
docker exec boot-test touch /tmp/test           # → succeeds (tmpfs writable)
docker exec boot-test curl -s -o /dev/null -w "%{http_code}" https://api.anthropic.com
# → some HTTP code (proves outbound network works)
```

Clean up:
```bash
docker stop boot-test && docker rm boot-test
```

### 2.7 Create systemd Unit

Create `/etc/systemd/system/boot-container.service` — the full spec is in
`phases/02-boot-container.md` section 2.10. Don't start it yet.

```bash
sudo systemctl daemon-reload
sudo systemctl enable boot-container.service
```

---

## Step 3: Verify Phase 3 — Tailscale

Most of this is already done by Omarchy. Quick verification pass:

```bash
systemctl is-active tailscaled              # → active
tailscale ip -4                             # → 100.x.x.x
tailscale status                            # → shows your devices
```

**Verify Tailscale SSH is enabled:**
```bash
sudo tailscale up --ssh
# If already up, this is a no-op
```

**Verify Funnel is OFF:**
```bash
tailscale funnel status
# → Funnel off / No configuration
```

**Verify from another device on your tailnet:**
```bash
ssh <your-user>@<tailscale-ip>
# Should work with Tailscale auth (no SSH key needed)
```

**Verify LAN SSH is blocked:**
From a non-Tailscale device on the same LAN:
```bash
ssh <your-user>@<lan-ip>
# Should timeout / connection refused
```

**ACL configuration** — do this in the Tailscale admin console
(https://login.tailscale.com/admin/acls). The ACL JSON is in
`phases/03-tailscale-setup.md` section 3.5.

---

## Step 4: POC Preparation

### 4.1 Create Telegram Bot

If not done yet:
1. Message @BotFather on Telegram → `/newbot`
2. Choose a name and username
3. Record the bot token
4. Message @userinfobot → record your numeric user ID

### 4.2 Create POC Environment File

Create `~/boot-data/.env.poc`:

```bash
TELEGRAM_BOT_TOKEN=<your-bot-token>
TELEGRAM_BOT_USERNAME=<your-bot-username>
APPROVED_DIRECTORY=/workspace
ALLOWED_USERS=<your-telegram-user-id>
USE_SDK=true
ANTHROPIC_API_KEY=<your-api-key>
AGENTIC_MODE=true
DEBUG=true
```

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
  boot:latest \
  bash
```

You're now inside the Boot container. Everything below happens inside.

### 5.2 Install Poetry and Clone the Repo

```bash
pip3 install --user poetry
export PATH="$HOME/.local/bin:$PATH"

cd /workspace
git clone https://github.com/RichardAtCT/claude-code-telegram.git poc-bot
cd poc-bot
```

### 5.3 Install Dependencies

```bash
poetry install --no-dev
```

If Poetry has issues with the `node` user home directory:
```bash
export POETRY_VIRTUALENVS_IN_PROJECT=true
poetry install --no-dev
```

### 5.4 Configure and Run

```bash
# Load env vars
export $(grep -v '^#' /data/.env.poc | xargs)

# Create the approved directory
mkdir -p /workspace/projects

# Verify Claude Code CLI works with your API key
ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY claude -p "say hello"
# Should get a response — proves API access works from inside the container

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

From inside the container:
```bash
# Memory limit check
cat /sys/fs/cgroup/memory.max 2>/dev/null || cat /sys/fs/cgroup/memory/memory.limit_in_bytes
# → 4294967296 (4GB)

# PID limit check
cat /sys/fs/cgroup/pids.max 2>/dev/null || cat /sys/fs/cgroup/pids/pids.max
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

| Question | Expected Answer |
|----------|----------------|
| Does the container boundary actually isolate? | Yes — can't see host filesystem |
| Do file permissions work across the boundary? | Yes — UID 1000 matches |
| Does Claude Code CLI work inside the container? | Yes — API key auth |
| Does a real Telegram bot work inside the container? | Yes — RichardAtCT's bot runs |
| What happens on daemon restart? | Container dies, comes back with restart policy |
| Does DNS break on network change? | Probably yes — need health check |
| Is the kill switch real? | Yes — `docker stop` kills everything |

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
