# Phase 2: Boot Container Setup

## Context

RALY runs on Omarchy 3.3.3 which installs and maintains Docker as a core dependency.
Omarchy owns `/etc/docker/daemon.json` and manages Docker through `omarchy-update`. We
do not touch daemon config — Omarchy owns it entirely.

Boot lives inside a single long-lived Docker container (Debian Bookworm). The container
runs the Python Telegram bot and Claude Code CLI with full network access. The container
boundary is the security boundary. Mounted volumes (`~/boot-workspace`, `~/boot-data`)
are the accepted blast radius.

This phase:
1. Verifies Docker is healthy and enabled
2. Creates host directories for Boot's volumes
3. Builds the Boot container image
4. Scans the image for vulnerabilities
5. Creates a systemd unit to manage the container
6. Tests the container lifecycle

### Why Boot lives in a container (not on host)

- Claude Code CLI needs Anthropic API access — no `--network none` hacks needed
- `docker stop boot` = kill switch. Blast radius = mounted volumes only.
- Boot has its own Debian OS — no Omarchy config file ownership conflicts
- executor.py is a simple subprocess call to `claude --print`, not a Docker command builder
- Can scale to multiple Boot containers for different purposes later

### Why we don't touch `daemon.json`

Omarchy's current `/etc/docker/daemon.json`:

```json
{
    "log-driver": "json-file",
    "log-opts": { "max-size": "10m", "max-file": "5" },
    "dns": ["172.17.0.1"],
    "bip": "172.17.0.1/16"
}
```

This configures logging, DNS, and the bridge network — all Omarchy's concern.
`omarchy-update` can overwrite this file at any time.

## Prerequisites

- Phase 1 complete: OS hardened, UFW enabled with deny-all inbound
- Docker installed by Omarchy (`docker --version` succeeds)
- User in `docker` group (`groups` shows `docker`)

## Steps

### 2.1 Enable Docker Service

Omarchy installs Docker but may not enable the service (it supports socket activation).
Boot needs Docker available at boot without socket activation latency.

```bash
sudo systemctl enable docker.service
sudo systemctl start docker.service

# Verify:
systemctl is-enabled docker.service   # → enabled
systemctl is-active docker.service    # → active
docker info >/dev/null 2>&1 && echo "Docker healthy" || echo "Docker FAILED"
```

### 2.2 Verify Docker Daemon Health

```bash
# Docker version (should be 27+ for current Omarchy)
docker --version

# Daemon info — check for warnings or errors
docker info 2>&1

# Confirm daemon.json is Omarchy's (should show log-driver, dns, bip)
cat /etc/docker/daemon.json
```

Things to verify in `docker info` output:
- Server Version matches `docker --version`
- Storage Driver: overlay2
- Logging Driver: json-file
- No warnings about deprecated features

### 2.3 Docker Socket Permissions

The Docker socket is equivalent to root access. Verify it's properly restricted.

```bash
ls -la /var/run/docker.sock
```

Expected: `srw-rw---- 1 root docker`

**Critical rule: NEVER mount `/var/run/docker.sock` inside Boot's container.**
The container does not need Docker access. It runs Claude Code CLI directly.

### 2.4 Install trivy

trivy scans the Boot container image for vulnerabilities.

```bash
sudo pacman -S trivy --needed
trivy --version
```

trivy is in Arch's `extra` repository. No AUR or curl install needed.

### 2.5 Create Host Directories

These directories will be bind-mounted into Boot's container. They persist across
container rebuilds and are the accepted blast radius.

```bash
# Project workspace — Claude Code works here
mkdir -p ~/boot-workspace

# Persistent data — SQLite, config, Claude auth
mkdir -p ~/boot-data

# Boot source code — Python application
mkdir -p ~/boot-src

# Verify ownership (must be UID 1000 to match container user)
ls -la ~ | grep boot
```

Set permissions:

```bash
chmod 750 ~/boot-workspace ~/boot-data ~/boot-src
```

### 2.6 Build Boot Container Image

Create the Dockerfile at `~/boot-src/Dockerfile`:

```dockerfile
FROM node:22-bookworm-slim

# System packages: Python 3 + dev tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-venv \
    git \
    build-essential \
    curl \
    ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Claude Code CLI globally
RUN npm install -g @anthropic-ai/claude-code@latest

# Non-root user: node user already exists in node: images at UID 1000
# Verify UID matches host user
RUN id node

# Create app directory
RUN mkdir -p /app && chown node:node /app

# Working directories for mounted volumes
RUN mkdir -p /workspace /data && chown node:node /workspace /data

USER node
WORKDIR /app

# Boot application will be mounted or copied here
# During development: -v ~/boot-src:/app
# For production: COPY . /app && pip install -r requirements.txt

# Entrypoint: the Boot Python application
# Will be set once the application code exists (Phase 5.x)
CMD ["bash"]
```

Build the image:

```bash
cd ~/boot-src
docker build -t boot:latest .
```

Record the image digest:

```bash
docker inspect boot:latest --format '{{.Id}}'
```

### 2.7 Scan Boot Image

```bash
trivy image boot:latest
```

Review trivy output for HIGH and CRITICAL vulnerabilities. Document any that cannot be
patched (upstream issues) as accepted risk.

### 2.8 Test Container Lifecycle

Test that the container starts, runs, and stops correctly with all the production flags:

```bash
# Start with production flags
docker run -d \
  --name boot-test \
  --init \
  --memory=4g \
  --memory-swap=6g \
  --cpus=4 \
  --pids-limit=512 \
  --restart=unless-stopped \
  --user 1000:1000 \
  -v ~/boot-workspace:/workspace \
  -v ~/boot-data:/data \
  boot:latest \
  sleep infinity

# Verify it's running
docker ps --filter name=boot-test

# Verify resource limits are applied
docker inspect boot-test --format '{{.HostConfig.Memory}}'
# Should show: 4294967296 (4GB in bytes)

# Verify user
docker exec boot-test whoami
# Should show: node

# Verify volumes are accessible
docker exec boot-test ls -la /workspace
docker exec boot-test ls -la /data

# Verify network works (outbound)
docker exec boot-test curl -s -o /dev/null -w "%{http_code}" https://api.anthropic.com
# Should show: some HTTP response (401 or similar — point is it reaches the internet)

# Verify Claude Code CLI is installed
docker exec boot-test claude --version

# Verify Python is available
docker exec boot-test python3 --version

# Verify no Docker socket access
docker exec boot-test ls /var/run/docker.sock 2>&1
# Should show: No such file or directory

# Stop and remove test container
docker stop boot-test
docker rm boot-test
```

### 2.9 Create systemd Unit

Create `/etc/systemd/system/boot-container.service`:

```ini
[Unit]
Description=Boot AI Assistant Container
After=docker.service
Requires=docker.service

[Service]
Type=simple
Restart=always
RestartSec=10

# Stop any existing container, ignore errors if not running
ExecStartPre=-/usr/bin/docker stop boot
ExecStartPre=-/usr/bin/docker rm boot

ExecStart=/usr/bin/docker run \
  --name boot \
  --init \
  --memory=4g \
  --memory-swap=6g \
  --cpus=4 \
  --pids-limit=512 \
  --user 1000:1000 \
  -v /home/<your-username>/boot-workspace:/workspace \
  -v /home/<your-username>/boot-data:/data \
  -v /home/<your-username>/boot-src:/app \
  boot:latest \
  python3 /app/boot/main.py

ExecStop=/usr/bin/docker stop -t 30 boot

[Install]
WantedBy=multi-user.target
```

**Note:** The `--restart` flag is NOT used here because systemd manages restarts.
The `-v ~/boot-src:/app` mount is for development — in production, the code would be
COPYed into the image and this mount removed.

Enable but do NOT start yet (Boot application code doesn't exist until Phase 5.x):

```bash
sudo systemctl daemon-reload
sudo systemctl enable boot-container.service
# Do NOT start — no application code yet
```

### 2.10 Clean Up

```bash
docker system prune -f
```

## Container Security Model

Boot's container runs with these constraints:

| Constraint | Value | Purpose |
|---|---|---|
| `--init` | tini as PID 1 | Zombie reaping + signal forwarding |
| `--memory=4g` | Hard limit | OOM-killed if exceeded, protects host |
| `--memory-swap=6g` | Swap limit | 2GB swap buffer for peaks |
| `--cpus=4` | CPU quota | Reserves 2 host cores for desktop |
| `--pids-limit=512` | Process limit | Fork bomb protection |
| `--user 1000:1000` | Non-root | Matches host UID, no privilege inside |
| Volumes | workspace + data only | Blast radius is these two directories |
| Network | Default bridge | Full outbound (needed for Anthropic API, package installs) |

**What the container CANNOT do:**
- Access the Docker socket (not mounted)
- Access host filesystem outside mounted volumes
- Escalate to root (no sudo, non-root user)
- Consume more than 4GB RAM / 4 CPUs
- Survive `docker stop boot` (kill switch)

**What the container CAN do (by design):**
- Reach the internet (Anthropic API, npm/pip registries, git)
- Install packages inside the container (may cause drift — see Ops Concerns)
- Read/write workspace and data volumes
- Run Claude Code CLI with full capability inside the container

## Ops Concerns

### Container drift
Runtime `npm install` / `pip install` inside the container diverges from the Dockerfile.
Keep dependency files (`package.json`, `requirements.txt`) on the data volume. When
rebuilding, restore from these files.

### DNS breakage
Docker DNS (127.0.0.11) breaks when the host network changes (WiFi reconnect, Tailscale
toggle). Boot's application must handle DNS errors as transient and retry. If persistent,
restart the container: `docker restart boot`.

### Daemon restarts
`omarchy-update` can restart Docker, killing Boot. The systemd unit with `Restart=always`
brings it back. Boot's application must be crash-resilient (SQLite state tracking).

## Verification Checklist

- [ ] Docker service enabled and started (`systemctl is-enabled docker.service`)
- [ ] `docker info` runs clean with no errors
- [ ] `/etc/docker/daemon.json` is **unmodified** (matches Omarchy's original)
- [ ] Docker socket permissions: `srw-rw---- root docker`
- [ ] trivy installed and working (`trivy --version`)
- [ ] Host directories created: `~/boot-workspace`, `~/boot-data`, `~/boot-src`
- [ ] Boot image built successfully (`docker images boot`)
- [ ] Boot image scanned with trivy, vulnerabilities reviewed
- [ ] Test container: starts, runs as UID 1000, volumes accessible, network works
- [ ] Test container: Claude Code CLI installed and responds to `--version`
- [ ] Test container: no Docker socket access
- [ ] systemd unit created and enabled (but not started)
- [ ] Test artifacts cleaned up

## Outputs for Downstream Phases

- Docker service enabled → Boot container can start at boot
- `boot:latest` image → Boot application will run inside this
- Host directories → Phase 4 (Telegram bot token stored in `~/boot-data`)
- systemd unit → Phase 5.x (start the unit once application code exists)
- trivy available → Phase 7 (periodic image rescanning)

## Removed from Original Phase 2 (and why)

| Removed | Reason |
|---|---|
| Per-container security contract (`--network none`, `--cap-drop ALL`, etc.) | Old architecture. Boot's container has full network. No sub-containers. |
| `--network none` verification | Boot container needs network for Claude API access. |
| userns-remap discussion | No longer relevant — Boot runs as UID 1000, no sub-containers. |
| Modify `daemon.json` | Omarchy owns it. Never touch. |
| Phase 5.7 (Dockerfile.sandbox) | Merged into this phase. The Boot container IS the sandbox. |

## Internet Validation Instruction

Before executing this phase, perform a web search for:
- "node:22-bookworm-slim Docker image latest 2026"
- "Claude Code CLI npm install latest version 2026"
- "Docker systemd unit file best practices"
- "tini Docker init process"
- "trivy container image scanning Arch Linux"

Verify that:
1. `node:22-bookworm-slim` is the current LTS Node.js image on Debian Bookworm
2. Claude Code CLI's npm package name is `@anthropic-ai/claude-code`
3. The systemd unit file syntax is correct for the current systemd version
4. trivy is still in Arch's `extra` repository
5. No breaking changes in Docker's `--init` flag behavior
