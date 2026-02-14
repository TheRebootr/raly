# RALY Phase Index

## System Summary

Target: Mac Mini 2018 (i5-8500B 6-core, 32GB RAM) running Omarchy 3.3.3 (Arch Linux).
Dual-use: headless server + occasional desktop (Chromium, printing, LocalSend).
Goal: Custom "Boot" harness (Python, ~500-800 lines, 3 dependencies) running inside a
long-lived Docker container (Debian Bookworm), reachable only via Telegram from a single
authorized user. Claude Code CLI runs directly inside Boot's container with full network
access. The container is the security boundary; mounted volumes are the accepted blast radius.
Remote access: Tailscale SSH (`tailscale up --ssh`) — no openssh sshd.

## Phase Map

| File                           | Phase                              | Depends On             | Duration      | Session |
| ------------------------------ | ---------------------------------- | ---------------------- | ------------- | ------- |
| `00-preflight.md`              | Pre-Flight Checks                  | Nothing                | 10 min        | 1       |
| `01-os-hardening.md`           | OS-Level Hardening                 | Phase 0                | 10 min        | 1       |
| `02-boot-container.md`         | Boot Container Setup               | Phase 1                | 45 min        | 1       |
| `03-tailscale-setup.md`        | Tailscale ACL Configuration        | Phase 1                | 15 min        | 1       |
| `04-telegram-bot-setup.md`     | Telegram Bot Token + Host Dirs     | Phase 0                | 20 min        | 2       |
| `05-pre-build-study.md`        | Pre-Build Research                 | None (can run anytime) | 3-4 hours     | 2       |
| `05.1-security-module.md`      | security.py                        | Phase 4, Phase 5 study | 1 day         | 3       |
| `05.2-config-module.md`        | config.py                          | Phase 4                | 2 hours       | 4       |
| `05.3-telegram-handler.md`     | telegram.py                        | Phases 5.1, 5.2        | 4 hours       | 4       |
| `05.4-claude-executor.md`      | executor.py (subprocess, ~100 LOC) | Phase 5.2              | 2 hours       | 5       |
| `05.5-session-memory.md`       | session.py, memory.py              | Phase 5.2              | 3 hours       | 5       |
| `05.6-cron-scheduler.md`       | Health checks + scheduled tasks    | Phase 5.3              | 2 hours       | 6       |
| `06-verification.md`           | Full Verification                  | All prior phases       | 30 min        | 7       |
| `07-ongoing-ops.md`            | Ongoing Operations                 | Phase 6                | Reference doc | -       |

## Dependency Graph

```
Phase 0 (preflight)
  ├── Phase 1 (OS hardening)
  │     ├── Phase 2 (Boot container setup — image, volumes, systemd unit)
  │     └── Phase 3 (Tailscale)
  └── Phase 4 (Telegram bot token + host directories)

Phase 2 + Phase 4 → All Phase 5.x (code runs inside Boot container)
  ├── Phase 5.2 (config) ← needs bot token from Phase 4
  │     ├── Phase 5.1 (security) ← also depends on 5-study
  │     ├── Phase 5.3 (telegram handler) ← also depends on 5.1
  │     ├── Phase 5.4 (executor) ← simplified, no sub-containers
  │     └── Phase 5.5 (session/memory)
  └── Phase 5.3 → Phase 5.6 (health/cron)

Phase 5-study (research) can run in parallel with Phases 0-4.
Phase 6 (verification) runs after everything.
Phase 7 (ops) is a reference document.
```

## Architecture (condensed)

```
Internet → Telegram API → [polling, no inbound ports]
                              ↓
Mac Mini (Omarchy 3.x) ← Tailscale SSH (no openssh sshd)
  ├── UFW: deny all inbound (LocalSend 53317 allowed on LAN)
  ├── ufw-docker: managed by Omarchy package (don't edit after.rules)
  ├── sysctl hardening (99-raly.conf, separate from Omarchy's 99-sysctl.conf)
  ├── boot-container.service (systemd → docker run)
  │
  └── Boot Container (node:22-bookworm-slim based, Debian)
        ├── Claude Code CLI (Node.js, full network for API access)
        ├── Python 3.12+ (Boot harness)
        │     ├── security.py (auth, rate limit, input validation, audit)
        │     ├── telegram.py (message handling, routing)
        │     ├── executor.py (subprocess → claude CLI, no sub-containers)
        │     └── session.py + memory.py (SQLite state)
        ├── Volume: ~/boot-workspace → /workspace (projects, blast radius)
        └── Volume: ~/boot-data → /data (SQLite, config, Claude auth)
```

## Boot Container Specification

```
Image base: node:22-bookworm-slim (Debian Bookworm, LTS until 2028)
Build:      Multi-stage (build-essential in build stage only, not in runtime)
Runtime:    --init (tini, zombie reaping + signal forwarding — NON-NEGOTIABLE)
Memory:     --memory=4g --memory-swap=6g
CPU:        --cpus=4 (reserves 2 host cores for desktop)
PIDs:       --pids-limit=512
User:       --user 1000:1000 (matches host UID)
Security:   --cap-drop ALL --security-opt=no-new-privileges --read-only
Tmpfs:      /tmp (512MB, noexec) + /home/node (256MB, noexec)
Volumes:    ~/boot-workspace:/workspace (project files — accepted blast radius)
            ~/boot-data:/data (SQLite, config, Claude auth credentials)
            ~/boot-src:/app:ro (Boot source code — read-only)
Network:    default bridge (full outbound, no inbound ports needed)
Lifecycle:  systemd unit (Restart=always), NOT --restart flag
NOT:        --privileged, Docker socket mount, --network none
```

### Why Boot lives in a container (not on host)

1. **Solves the network problem**: Claude Code CLI needs Anthropic API access. No --network
   none hacks, no proxies, no restricted Docker networks. The container has network.
2. **Clean security boundary**: If Boot is compromised, `docker stop boot` kills it.
   Blast radius = the mounted volumes, nothing else on the host.
3. **No Omarchy conflicts**: Boot has its own Debian. No config file ownership fights.
4. **Simple executor**: executor.py runs `claude --print` as a subprocess. No Docker
   command builder, no sub-container management. ~100 lines instead of ~300.
5. **Scalable**: Can run multiple Boot containers for different purposes later.

### What was removed from the old design

- Phase 5.7 (Dockerfile.sandbox) → merged into Phase 2 (the container IS Boot)
- Per-task ephemeral sub-containers → Claude runs directly inside Boot
- DockerCommandBuilder class in executor.py → simple subprocess call
- `--network none` on task execution → container boundary is the isolation
- userns-remap consideration → container user is 1000:1000, matching host

## Security Layers

```
L0:  LUKS disk encryption (Omarchy default)
L1:  Tailscale SSH — no openssh sshd, no inbound ports, WireGuard encrypted
L2:  UFW deny-all inbound + ufw-docker bypass prevention (Omarchy managed)
L3:  Sysctl kernel hardening (BPF, ptrace, perf restrictions)
L4:  Boot container boundary (isolated filesystem, process namespace, cgroups)
L5:  Container resource limits (4GB RAM, 4 CPUs, 512 PIDs)
L6:  Container restrictions (no Docker socket, no --privileged, non-root user)
L7:  Capability + privilege hardening (cap-drop ALL, no-new-privileges)
L8:  Immutable container (read-only rootfs, noexec tmpfs, source mounted read-only)
L9:  Multi-stage image (no compilers in runtime — gcc/make removed)
L10: Auth middleware + input validation (inside Boot)
L11: Telegram allowlist (single numeric ID)
L12: Token bucket rate limiter (requests + cost)
L13: Mounted volumes as explicit blast radius (workspace + data only)
L14: SQLite audit trail (every action logged)
L15: 3 Python dependencies (auditable in minutes)
```

## Known Operational Concerns

```
DNS breakage:       Omarchy routes container DNS through 172.17.0.1 (bridge IP).
                    Breaks on host network changes. Boot must retry. May need restart.
Container drift:    Read-only rootfs blocks system-wide installs. Deps belong in
                    Dockerfile (multi-stage build). Rebuild image when deps change.
Daemon restarts:    omarchy-update can restart Docker daemon, killing Boot.
                    systemd Restart=always brings it back. Design for crash resilience.
Thermal throttle:   Sustained builds (webpack, cargo) throttle on Mac Mini 2018.
                    Workloads are bursty — chassis recovers between API wait periods.
No AppArmor:        Arch has no AppArmor/SELinux. Docker seccomp profile is the only MAC.
                    Accepted trade-off — consistent with host OS choice.
```

## Omarchy Baseline (do not duplicate or override)

```
Omarchy provides and manages (via omarchy-update):
  - UFW + ufw-docker package (don't edit /etc/ufw/after.rules)
  - Docker + daemon.json (hands off — Omarchy owns entirely)
  - PAM faillock (deny=10, unlock_time=120)
  - LUKS full-disk encryption
  - Tailscale (tailscaled.service)
  - Pacman signature verification (Required on core/extra)
  - Desktop services (cups, avahi, bluetooth, sddm) — keep, dual-use machine

Do NOT set (will break things):
  - kernel.unprivileged_userns_clone = 0  → breaks Chromium sandbox
  - kernel.sysrq = 0                     → needed for keyboard recovery
  - net.ipv6.conf.all.disable_ipv6 = 1   → breaks Tailscale
  - net.ipv4.ip_forward                  → Docker manages it
```
