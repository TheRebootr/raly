# RALY — Run Assistants Locally Yourself

A developer guide for running AI assistants inside Docker containers on your own machine — with known blast radius, one-command kill switch, and zero cloud dependencies.

**RALY is not a product — yet.** Today it's a reference repo. Fork it, adapt it, learn from it. Tomorrow: `raly up boot`.

## The Idea

Every new AI assistant (Claude Code, OpenClaw, Nanoclaw, moltbot, etc.) wants to run on your machine with full access. You want to try them. You also want to:

- **Know exactly what it can touch** — not "trust me, it's sandboxed"
- **Kill it with one command** — `docker stop boot`
- **Run it on YOUR hardware** — no cloud VMs, no subscriptions, no vendor lock-in
- **Test any assistant** — swap one out, drop another in, same system

RALY documents how to set this up properly: harden the host, build the container, define the blast radius, and run your assistant inside it.

## Boot — First-Class Citizen

Boot is RALY's reference AI assistant. It's a Python-based harness that:

- Receives commands from a single authorized Telegram user
- Runs Claude Code CLI inside an isolated Docker container (Debian Bookworm)
- Works on your projects through mounted volumes (the accepted blast radius)
- Can be killed instantly with `docker stop boot`

Boot is opinionated. It runs on an opinionated Linux distro (Omarchy). It's scoped. It's laser-focused. It exists to prove the system works — and to show that building your own assistant on this foundation is simple.

**But Boot is just the first tenant.** The system layer is assistant-agnostic. You could drop [RichardAtCT's claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram), OpenClaw, or any other assistant into the same container setup.

## Architecture

```
You (Telegram) ──→ Telegram API ──→ [polling, no inbound ports]
                                         │
           Mac Mini / Linux Server ←── Tailscale SSH (only remote access)
             │
             ├── UFW: deny all inbound
             ├── Sysctl hardening (99-raly.conf)
             ├── LUKS full-disk encryption
             │
             └── Boot Container (Debian Bookworm)
                   ├── Claude Code CLI (full network for API access)
                   ├── Python bot (Telegram polling + task routing)
                   ├── Volume: ~/boot-workspace → /workspace
                   └── Volume: ~/boot-data → /data (SQLite, config)
```

### Key Design Decisions

- **The assistant lives in a container, not on the host.** The assistant gets its own Debian environment with full outbound access. The container boundary *is* the security boundary.
- **No sub-containers.** No Docker-in-Docker, no per-task ephemeral containers. The assistant runs directly inside its container. Simple.
- **The user is the trust boundary.** You accept responsibility for prompt injection risks. RALY protects against unauthorized access, not against yourself.
- **Tailor-fit, not template.** Every decision is made for a specific machine, OS, and threat model. Generic hardening checklists are traps.

## Target Environment

Built for and tested on:

- **Hardware**: Mac Mini 2018 (i5-8500B 6-core, 32GB RAM)
- **Host OS**: [Omarchy](https://omarchy.com) 3.3.3 (Arch Linux)
- **Remote Access**: Tailscale SSH (no openssh sshd)
- **Container Base**: `node:22-bookworm-slim` (Debian Bookworm, LTS until 2028)

Adaptable to other Linux systems. The Omarchy-specific checks are a reference — the system prereqs (Docker, UFW, disk encryption) are standard on any hardened Linux or macOS.

## Phase Structure

The project is split into two layers:

### System Layer (host hardening — assistant-agnostic)

| Phase | Description | Status |
|-------|-------------|--------|
| [Phase 0](phases/00-preflight.md) | Pre-flight checks | Done |
| [Phase 1](phases/01-os-hardening.md) | OS-level hardening (sysctl, UFW, core dumps) | Done |
| [Phase 2](phases/02-boot-container.md) | Container setup (Dockerfile, volumes, systemd) | Ready |
| [Phase 3](phases/03-tailscale-setup.md) | Tailscale ACL configuration | Ready |
| [POC Plan](phases/POC-PLAN.md) | Validation with RichardAtCT's bot as a drop-in test | Ready |

### Boot Layer (reference assistant implementation)

| Phase | Description | Status |
|-------|-------------|--------|
| [Phase 4](phases/boot/04-telegram-bot-setup.md) | Telegram bot token + directory structure | Planned |
| [Phase 5.1](phases/boot/05.1-security-module.md) | security.py (auth, rate limit, audit) | Planned |
| [Phase 5.2](phases/boot/05.2-config-module.md) | config.py | Planned |
| [Phase 5.3](phases/boot/05.3-telegram-handler.md) | telegram.py (message handling) | Planned |
| [Phase 5.4](phases/boot/05.4-claude-executor.md) | executor.py (Claude CLI subprocess) | Planned |
| [Phase 5.5](phases/boot/05.5-session-memory.md) | session.py + memory.py (SQLite state) | Planned |
| [Phase 5.6](phases/boot/05.6-cron-scheduler.md) | Health checks + scheduled tasks | Planned |
| [Phase 6](phases/boot/06-verification.md) | Full verification | Planned |
| [Phase 7](phases/boot/07-ongoing-ops.md) | Ongoing operations reference | Planned |

## Container Spec

```
Image:      node:22-bookworm-slim + Python 3.12 + Claude Code CLI
Runtime:    --init (zombie reaping — NON-NEGOTIABLE)
Memory:     --memory=4g --memory-swap=6g
CPU:        --cpus=4
PIDs:       --pids-limit=512
Restart:    --restart=unless-stopped
User:       --user 1000:1000 (matches host UID)
Volumes:    ~/boot-workspace:/workspace (projects — blast radius)
            ~/boot-data:/data (SQLite, config, Claude auth)
NEVER:      --privileged, Docker socket mount
```

## Security Layers

```
L0:  LUKS disk encryption
L1:  Tailscale SSH (no openssh sshd, WireGuard encrypted)
L2:  UFW deny-all inbound
L3:  Sysctl kernel hardening (BPF, ptrace, perf restrictions)
L4:  Container boundary (isolated filesystem, process namespace, cgroups)
L5:  Container resource limits (4GB RAM, 4 CPUs, 512 PIDs)
L6:  No Docker socket, no --privileged, non-root user
L7:  Telegram allowlist (single numeric user ID)
L8:  Token bucket rate limiter
L9:  Mounted volumes as explicit blast radius
L10: SQLite audit trail
```

## Known Operational Concerns

| Concern | Impact | Mitigation |
|---------|--------|------------|
| Docker DNS breaks on network change | Bot loses API access | Health check + auto-restart |
| Container drift from runtime installs | Dockerfile diverges from reality | Keep deps on volumes, periodic rebuild |
| `omarchy-update` restarts Docker daemon | Container dies | `--restart=unless-stopped` + crash-resilient design |
| No AppArmor/SELinux on Arch | Fewer kernel-level restrictions | Accepted trade-off, Docker seccomp still active |
| Mac Mini 2018 thermal throttling | 20-40% perf loss during sustained builds | Workloads are bursty, chassis recovers between API waits |

## Roadmap

- **Now**: Reference repo — phase docs, manual setup, learn the system
- **Next**: `raly` CLI (npm package) — automate prerequisite checks, `raly up boot`, `raly down boot`, `raly status`
- **Later**: Drop-in assistant configs — community-contributed setups for different AI assistants

## Prior Art

- [RichardAtCT/claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) — Claude Code + Telegram bot (no Docker, host-level execution). Used as POC test payload.
- [Anthropic Claude Code Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing) — Official sandboxing guidance.

## License

MIT
