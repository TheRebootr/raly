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

### 2.6 Create `.dockerignore`

Create `~/boot-src/.dockerignore` to keep the build context clean:

```
.git
__pycache__
*.pyc
.env
```

Without this, `docker build` copies everything in `~/boot-src` into the build context.

### 2.7 Build Boot Container Image

Create the Dockerfile at `~/boot-src/Dockerfile`:

```dockerfile
# =============================================================================
# Stage 1: BUILD — compile native modules, install Python deps
# =============================================================================
FROM node:22-bookworm-slim AS builder

# Build tools needed for native npm/pip packages (not kept in final image)
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-venv \
    build-essential \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Claude Code CLI globally
# Pin to a specific version once a working version is confirmed.
# Using @latest means every rebuild may get a different version.
RUN npm install -g @anthropic-ai/claude-code@latest

# Create Python venv for Boot's dependencies
# This venv is copied to the runtime image, so deps survive --read-only rootfs.
RUN python3 -m venv /opt/boot-venv
# Boot's requirements will be installed here once they exist (Phase 5.x).
# For now, the venv is empty but ready.
# Example: COPY requirements.txt /tmp/ && /opt/boot-venv/bin/pip install -r /tmp/requirements.txt

# =============================================================================
# Stage 2: RUNTIME — clean image without compilers
# =============================================================================
FROM node:22-bookworm-slim

# Runtime-only packages (no build-essential, no gcc, no make)
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-venv \
    git \
    curl \
    ca-certificates \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Copy Claude Code CLI from builder
COPY --from=builder /usr/local/lib/node_modules /usr/local/lib/node_modules
COPY --from=builder /usr/local/bin/claude /usr/local/bin/claude

# Copy Python venv from builder
COPY --from=builder /opt/boot-venv /opt/boot-venv

# Non-root user: node user already exists in node: images at UID 1000
# Verify UID matches host user
RUN id node

# Create app directory
RUN mkdir -p /app && chown node:node /app

# Working directories for mounted volumes
RUN mkdir -p /workspace /data && chown node:node /workspace /data

USER node

# Put venv on PATH so Boot's Python deps are available
ENV PATH="/opt/boot-venv/bin:$PATH"

WORKDIR /app

# Boot application will be mounted or copied here
# During development: -v ~/boot-src:/app:ro
# For production: COPY . /app (deps already in /opt/boot-venv)

# Entrypoint: the Boot Python application
# Will be set once the application code exists (Phase 5.x)
CMD ["bash"]
```

**Why multi-stage:** The build stage installs `build-essential` (gcc, make) to compile
native npm/pip modules. The runtime stage copies only the compiled artifacts. This means
the final image has no compilers — an attacker who gains code execution inside the
container cannot compile C code.

**Python dependency strategy:** Boot's Python dependencies (e.g. `python-telegram-bot`)
are installed into `/opt/boot-venv` during the Docker build. This venv is baked into the
image and survives `--read-only` rootfs. When dependencies change, rebuild the image.
During Phase 5.x, a `requirements.txt` will be added and the `COPY + pip install` line
in the build stage uncommented.

Build the image:

```bash
cd ~/boot-src
docker build -t boot:latest .
```

Record the image digest:

```bash
docker inspect boot:latest --format '{{.Id}}'
```

### 2.8 Scan Boot Image

```bash
trivy image boot:latest
```

Review trivy output for HIGH and CRITICAL vulnerabilities. Document any that cannot be
patched (upstream issues) as accepted risk.

### 2.9 Test Container Lifecycle

Test that the container starts, runs, and stops correctly with all the production flags.
Note: `--restart` is NOT used here — in production, systemd manages restarts.

```bash
# Start with production flags
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

# Verify it's running
docker ps --filter name=boot-test

# Verify resource limits are applied
docker inspect boot-test --format '{{.HostConfig.Memory}}'
# Should show: 4294967296 (4GB in bytes)

# Verify security options
docker inspect boot-test --format '{{.HostConfig.SecurityOpt}}'
# Should show: [no-new-privileges]

# Verify capabilities are dropped
docker inspect boot-test --format '{{.HostConfig.CapDrop}}'
# Should show: [ALL]

# Verify read-only rootfs
docker inspect boot-test --format '{{.HostConfig.ReadonlyRootfs}}'
# Should show: true

# Verify user
docker exec boot-test whoami
# Should show: node

# Verify volumes are accessible
docker exec boot-test ls -la /workspace
docker exec boot-test ls -la /data

# Verify /tmp is writable (tmpfs)
docker exec boot-test touch /tmp/test && echo "/tmp writable" || echo "/tmp NOT writable"

# Verify rootfs is read-only (write to /usr should fail)
docker exec boot-test touch /usr/test 2>&1
# Should show: Read-only file system

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

### 2.10 Create systemd Unit

Create `/etc/systemd/system/boot-container.service`:

**Important:** Replace `YOUR_USER` below with the actual username on the Mac Mini.
Systemd does not expand `~` — absolute paths are required.

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
  --security-opt=no-new-privileges \
  --cap-drop ALL \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=512m \
  --tmpfs /home/node:rw,noexec,nosuid,size=256m \
  -v /home/YOUR_USER/boot-workspace:/workspace \
  -v /home/YOUR_USER/boot-data:/data \
  -v /home/YOUR_USER/boot-src:/app:ro \
  boot:latest \
  python3 /app/boot/main.py

ExecStop=/usr/bin/docker stop -t 30 boot

[Install]
WantedBy=multi-user.target
```

**Notes:**

- `--restart` is NOT used — systemd manages restarts via `Restart=always`.
- `boot-src:/app:ro` — Boot's source code is mounted read-only. The container cannot
  modify its own harness code. If you want Claude Code to edit Boot's source (self-modification),
  remove `:ro` — but understand the implication: the AI can rewrite its own control harness.
- `--read-only` makes the container rootfs immutable. Only `/tmp` (512MB), `/home/node`
  (256MB), and the mounted volumes are writable. Both tmpfs mounts are `noexec` — no
  binary payloads can be executed from them even if written there.
- In production, the code would be COPYed into the image and the `boot-src` mount removed.

Enable but do NOT start yet (Boot application code doesn't exist until Phase 5.x):

```bash
sudo systemctl daemon-reload
sudo systemctl enable boot-container.service
# Do NOT start — no application code yet
```

### 2.11 Clean Up

```bash
docker system prune -f
```

## Container Security Model

Boot's container runs with these constraints:

| Constraint              | Value                 | Purpose                                                    |
| ----------------------- | --------------------- | ---------------------------------------------------------- |
| `--init`                | tini as PID 1         | Zombie reaping + signal forwarding                         |
| `--memory=4g`           | Hard limit            | OOM-killed if exceeded, protects host                      |
| `--memory-swap=6g`      | Swap limit            | 2GB swap buffer for peaks                                  |
| `--cpus=4`              | CPU quota             | Reserves 2 host cores for desktop                          |
| `--pids-limit=512`      | Process limit         | Fork bomb protection                                       |
| `--user 1000:1000`      | Non-root              | Matches host UID, no privilege inside                      |
| `--no-new-privileges`   | Security option       | Blocks setuid/setgid privilege escalation                  |
| `--cap-drop ALL`        | Drop all capabilities | No Linux capabilities (NET_RAW, MKNOD, etc.)              |
| `--read-only`           | Immutable rootfs      | Container filesystem cannot be modified                    |
| `--tmpfs /tmp`          | 512MB, noexec, nosuid | Scratch space, no executable payloads, lost on restart     |
| `--tmpfs /home/node`    | 256MB, noexec, nosuid | User home scratch, no executable payloads, lost on restart |
| Volumes                 | workspace + data only | Blast radius is these two directories                      |
| `boot-src:/app:ro`      | Read-only source      | Boot cannot modify its own harness code                    |
| Network                 | Default bridge        | Full outbound (needed for Anthropic API, package installs) |

**What the container CANNOT do:**

- Access the Docker socket (not mounted)
- Access host filesystem outside mounted volumes
- Escalate to root (no sudo, non-root user, no-new-privileges, all caps dropped)
- Modify its own binaries or rootfs (read-only filesystem)
- Modify its own harness source code (boot-src mounted read-only)
- Execute binaries dropped into /tmp or /home/node (noexec on tmpfs)
- Use raw sockets, mknod, or any Linux capability (all dropped)
- Consume more than 4GB RAM / 4 CPUs
- Survive `docker stop boot` (kill switch)

**What the container CAN do (by design):**

- Reach the internet (Anthropic API, npm/pip registries, git)
- Write to /tmp (512MB) and /home/node (256MB) — tmpfs, noexec, lost on restart
- Read/write workspace and data volumes
- Run Claude Code CLI with full capability inside the container

### Accepted tradeoff: credentials and state share one volume

`~/boot-data` holds both secrets (Telegram bot token, Anthropic API key/auth) and
mutable state (SQLite databases, session data, config) in a single read-write volume.

If the container is compromised, an attacker has access to both credentials AND can
tamper with audit logs. A more paranoid design would:
- Mount individual credential files as read-only bind mounts
- Keep mutable state (SQLite) in a separate writable volume

We accept the single-volume design because:
1. The container IS the trust boundary — if it's compromised, credentials are already
   in-memory regardless of mount layout
2. Splitting adds operational complexity (more mounts, more paths, more to manage)
3. The blast radius is already accepted — `~/boot-data` is one of two volumes the user
   explicitly chose to expose

If your threat model requires credential/state separation, split `~/boot-data` into
`~/boot-secrets` (mounted read-only) and `~/boot-state` (mounted read-write).

## Ops Concerns

### Container drift

With `--read-only` and `noexec` on tmpfs, runtime `npm install -g` / `pip install` to
system paths will fail, and executables written to /tmp cannot run. This is by design.

All persistent dependencies belong in the Dockerfile:
- npm packages → installed globally in the build stage, copied to runtime
- Python packages → installed into `/opt/boot-venv` in the build stage, copied to runtime
- When deps change → rebuild the image (`docker build`), restart the container

User-local installs to /workspace are still possible (e.g. `npm install` in a project
directory), but these are scoped to the workspace volume, not system-wide.

### DNS breakage

Omarchy's `daemon.json` sets `"dns": ["172.17.0.1"]`, routing container DNS through the
Docker bridge IP instead of the standard embedded DNS (127.0.0.11). The Docker daemon
forwards these queries to host DNS. When the host network changes (WiFi reconnect,
Tailscale toggle), DNS resolution inside containers may break or cache stale results.
Boot's application must handle DNS errors as transient and retry. If persistent,
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
- [ ] `.dockerignore` created in `~/boot-src`
- [ ] Host directories created: `~/boot-workspace`, `~/boot-data`, `~/boot-src`
- [ ] Boot image built successfully (`docker images boot`)
- [ ] Boot image scanned with trivy, vulnerabilities reviewed
- [ ] Test container: starts, runs as UID 1000, volumes accessible, network works
- [ ] Test container: Claude Code CLI installed and responds to `--version`
- [ ] Test container: no Docker socket access
- [ ] Test container: `no-new-privileges` security option confirmed
- [ ] Test container: all capabilities dropped (`--cap-drop ALL`)
- [ ] Test container: read-only rootfs confirmed, /tmp writable, rootfs writes fail
- [ ] systemd unit created and enabled (but not started)
- [ ] systemd unit uses absolute paths (no `~`, correct username)
- [ ] Test artifacts cleaned up

## Outputs for Downstream Phases

- Docker service enabled → Boot container can start at boot
- `boot:latest` image → Boot application will run inside this
- Host directories → Phase 4 (Telegram bot token stored in `~/boot-data`)
- systemd unit → Phase 5.x (start the unit once application code exists)
- trivy available → Phase 7 (periodic image rescanning)

## Removed from Original Phase 2 (and why)

| Removed                                            | Reason                                                                        |
| -------------------------------------------------- | ----------------------------------------------------------------------------- |
| Per-container security contract for sub-containers  | Old architecture. No sub-containers. Boot's own container gets these flags.    |
| `--network none` verification                      | Boot container needs network for Claude API access.                            |
| userns-remap discussion                            | No longer relevant — Boot runs as UID 1000, no sub-containers.                |
| Modify `daemon.json`                               | Omarchy owns it. Never touch.                                                 |
| Phase 5.7 (Dockerfile.sandbox)                     | Merged into this phase. The Boot container IS the sandbox.                     |

**Clarification:** `--cap-drop ALL`, `--security-opt=no-new-privileges`, and `--read-only`
were previously specified for ephemeral sub-containers. They are now applied to Boot's own
container — the security properties are the same, just applied at the right level.

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
