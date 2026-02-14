# RALY — CLAUDE.md

## What This Project Is

RALY (Run Assistants Locally Yourself) is a developer reference repo for running AI
assistants inside Docker containers on your own machine. The system layer is
assistant-agnostic — any AI assistant (coding, chat, agentic) can be dropped in.
"Boot" is RALY's first-class reference assistant — a Python-based harness that runs
Claude Code CLI inside a Docker container on a Mac Mini 2018 (Omarchy 3.3.3 / Arch Linux).
The user communicates with Boot exclusively via Telegram. The phases/ directory contains
step-by-step implementation specs.

## Core Architecture

- **Host (Omarchy)**: Runs Docker, manages the Boot container via systemd. Does not run Boot
  code directly. Protected by UFW, sysctl hardening, Tailscale, LUKS.
- **Boot Container (Debian Bookworm)**: Long-lived Docker container based on
  `node:22-bookworm-slim`. Runs the Python Telegram bot + Claude Code CLI. Has full outbound
  network access. The container IS the security boundary.
- **Volumes**: `~/boot-workspace` (project files) and `~/boot-data` (SQLite, config) are
  bind-mounted. These are the accepted blast radius — if Boot is compromised, only these
  volumes are affected.

## Critical Rules

### Omarchy Ownership — HANDS OFF

- `/etc/docker/daemon.json` — Omarchy owns this. Never modify.
- `/etc/ufw/after.rules` — Owned by `ufw-docker` package. Never edit manually.
- `/etc/sysctl.d/99-sysctl.conf` — Owned by Omarchy. Use `99-raly.conf` instead.
- `/etc/pam.d/system-auth` — Modified by Omarchy (faillock settings).

### Container Rules — NON-NEGOTIABLE

- Always use `--init` (tini). Boot spawns subprocesses; without init, zombies accumulate
  and signals don't propagate.
- Never mount the Docker socket into Boot's container.
- Never run Boot's container with `--privileged`.
- Container user must be UID 1000:1000 (matches host user).
- Resource limits: `--memory=4g --cpus=4 --pids-limit=512`.

### Things That Will Break If You Set Them

- `kernel.unprivileged_userns_clone = 0` → breaks Chromium sandbox
- `kernel.sysrq = 0` → user needs keyboard recovery
- `net.ipv6.conf.all.disable_ipv6 = 1` → breaks Tailscale
- `net.ipv4.ip_forward` → Docker manages this

### Desktop Services — Keep Them

This is a dual-use machine (server + occasional desktop). Do NOT disable:
cups, avahi, bluetooth, sddm, LocalSend (port 53317).

## Design Principles

1. **Tailor-fit, not template.** Every action must align with this specific machine, this
   specific OS, this specific threat model. Generic hardening checklists are traps.
2. **Boot is a tenant, not a prisoner.** The container has full network. Isolation comes from
   the container boundary and volume scoping, not from crippling Boot's capabilities.
3. **Respect Omarchy.** Use drop-in configs and separate files. Never modify files Omarchy
   owns. `omarchy-update` runs pacman + AUR + migrations that can re-enable services or
   change configs.
4. **Design for crashes.** Docker daemon restarts (during updates) will kill Boot. Sleep/wake
   breaks network connections. Boot must track state in SQLite and handle cold restarts.
5. **The user is the trust boundary.** Single Telegram user ID allowlist. The user accepts
   responsibility for prompt injection and context injection risks.

## Phase Structure

See `phases/OVERVIEW.md` for the full phase map and dependency graph.

Key phases:

- Phase 0-1: Host hardening (preflight, OS, sysctl)
- Phase 2: Boot container setup (Dockerfile, volumes, systemd unit)
- Phase 3: Tailscale ACLs
- Phase 4: Telegram bot token + host directory structure
- Phase 5.x: Boot application code (security, config, telegram, executor, session, health)
- Phase 6-7: Verification and operations

## File Locations

```
~/boot-workspace/     → Mounted as /workspace in container (project files)
~/boot-data/          → Mounted as /data in container (SQLite, config, Claude auth)
~/boot-src/           → Boot source code (Python), built into container image
phases/               → This directory — implementation specs (not deployed)
```

## Known Operational Concerns

- Docker DNS breaks on host network changes (confirmed Docker bug, no clean fix)
- Container drift from runtime installs (npm, pip) — need rebuild procedures
- No AppArmor/SELinux on Arch — Docker seccomp is the only kernel-level MAC
- Mac Mini 2018 thermal throttling during sustained builds (~20-40% perf loss)
