# RALY — Run Assistants Locally Yourself

A safe space to evaluate, test, and build with AI assistants on your own machine — with known blast radius, one-command kill switch, and zero cloud dependencies.

## Why This Exists

I'm not a Linux expert. I'm not a networking expert. I'm a developer who's curious, excited about AI assistants, and cautious enough to know I should be careful.

AI assistants are big topic right now — OpenClaw, Nanoclaw, and whatever drops tomorrow. I want to try all of them. I want to experiment, build things, and have fun. But I also want to know that when I give an AI assistant access to my machine, I understand exactly what it can touch and what happens if something goes wrong.

I couldn't find a simple guide that said: "here's how to set up a safe playground for AI assistants on your own hardware, with known tradeoffs, in a way that doesn't require a PhD in systems administration."

So I'm building one. And I'm documenting every step because maybe you want the same thing.

This is a personal project. I'm learning as I go and testing on a spare machine, making mistakes, and writing it all down. If you know more about Linux, containers, or security than I do **— I'd love your help making this easier and safer for everyone**.

Today it's a reference repo. Fork it, adapt it, learn from it.

## The Problem

Every new AI assistant wants full access to your machine. You want to try them. You also want to:

- **Know exactly what it can touch** — not "trust me, it's sandboxed"
- **Kill it with one command** — `docker stop boot`
- **Run it on YOUR hardware** — no cloud VMs, no subscriptions, no vendor lock-in
- **Test any assistant** — swap one out, drop another in, same system

RALY is the framework and tooling that guards you so you can have fun and learn.

## The SideQuest: Boot — First-Class Citizen

RALY lets you try any assistant safely. But after enough evaluating, you start wanting something that's just _yours_ — built the way you want, for the things you actually do.

That's Boot. It's the assistant I'm building for myself using RALY as the foundation. Same idea as Omarchy — opinionated, scoped, built for one person's workflow. Not trying to be the best assistant out there. Just mine.

Right now Boot talks to me over Telegram, runs Claude Code CLI inside a container, and works on whatever I point it at. It's a POC. The point isn't Boot itself — it's showing that once RALY handles the boring security stuff, building your own assistant on top is the fun part.

The system layer doesn't care what assistant you run. Drop in [RichardAtCT's claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram), OpenClaw, or whatever you want. Try them all. Then build your own Boot.

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

- **The assistant lives in a container, not on the host.** The assistant gets its own Debian environment with full outbound access. The container boundary _is_ the security boundary.
- **No sub-containers.** No Docker-in-Docker, no per-task ephemeral containers. The assistant runs directly inside its container. Simple.
- **The user is the trust boundary.** You accept responsibility for prompt injection risks. RALY protects against unauthorized access, not against yourself.
- **Tailor-fit, not template.** Every decision is made for a specific machine, OS, and threat model. Generic hardening checklists are traps.

## Target Environment

Tested on a **dedicated experiment machine** — not my daily driver. I'd recommend the same while you're learning. Dedicate a spare machine or VM to this.

- **Hardware**: Mac Mini 2018 (i5-8500B 6-core, 32GB RAM)
- **Host OS**: [Omarchy](https://omarchy.com) 3.3.3 (Arch Linux)
- **Remote Access**: Tailscale SSH (no openssh sshd)
- **Container Base**: `node:22-bookworm-slim` (Debian Bookworm, LTS until 2028)

Adaptable to other Linux systems. The Omarchy-specific checks are a reference — the system prereqs (Docker, UFW, disk encryption) are standard on any hardened Linux or macOS.

## Phase Structure

The project is split into two layers:

### System Layer (host hardening — assistant-agnostic)

| Phase                                   | Description                                         | Status |
| --------------------------------------- | --------------------------------------------------- | ------ |
| [Phase 0](phases/00-preflight.md)       | Pre-flight checks                                   | Done   |
| [Phase 1](phases/01-os-hardening.md)    | OS-level hardening (sysctl, UFW, core dumps)        | Done   |
| [Phase 2](phases/02-boot-container.md)  | Container setup (Dockerfile, volumes, systemd)      | Done   |
| [Phase 3](phases/03-tailscale-setup.md) | Tailscale ACL configuration                         | Done   |
| [POC Plan](phases/POC-PLAN.md)          | Validation with RichardAtCT's bot as a drop-in test | Ready  |

### Boot Layer (reference assistant implementation)

| Phase                                             | Description                              | Status  |
| ------------------------------------------------- | ---------------------------------------- | ------- |
| [Phase 4](phases/boot/04-telegram-bot-setup.md)   | Telegram bot token + directory structure | Planned |
| [Phase 5.1](phases/boot/05.1-security-module.md)  | security.py (auth, rate limit, audit)    | Planned |
| [Phase 5.2](phases/boot/05.2-config-module.md)    | config.py                                | Planned |
| [Phase 5.3](phases/boot/05.3-telegram-handler.md) | telegram.py (message handling)           | Planned |
| [Phase 5.4](phases/boot/05.4-claude-executor.md)  | executor.py (Claude CLI subprocess)      | Planned |
| [Phase 5.5](phases/boot/05.5-session-memory.md)   | session.py + memory.py (SQLite state)    | Planned |
| [Phase 5.6](phases/boot/05.6-cron-scheduler.md)   | Health checks + scheduled tasks          | Planned |
| [Phase 6](phases/boot/06-verification.md)         | Full verification                        | Planned |
| [Phase 7](phases/boot/07-ongoing-ops.md)          | Ongoing operations reference             | Planned |

## Container Spec

```
Image:      node:22-bookworm-slim + Python 3.12 + Claude Code CLI
Build:      Multi-stage (no compilers in runtime image)
Runtime:    --init (zombie reaping — NON-NEGOTIABLE)
Memory:     --memory=4g --memory-swap=6g
CPU:        --cpus=4
PIDs:       --pids-limit=512
User:       --user 1000:1000 (matches host UID)
Security:   --cap-drop ALL --security-opt=no-new-privileges --read-only
Tmpfs:      /tmp (512MB, noexec) + /home/node (256MB, noexec)
Volumes:    ~/boot-workspace:/workspace (projects — blast radius)
            ~/boot-data:/data (SQLite, config, Claude auth)
            ~/boot-src:/app:ro (source code — read-only)
Lifecycle:  systemd unit (Restart=always), NOT Docker --restart flag
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
L7:  Capability + privilege hardening (cap-drop ALL, no-new-privileges)
L8:  Immutable container (read-only rootfs, noexec tmpfs, source read-only)
L9:  Multi-stage image (no compilers in runtime)
L10: Auth middleware + input validation (inside Boot)
L11: Telegram allowlist (single numeric user ID)
L12: Token bucket rate limiter
L13: Mounted volumes as explicit blast radius
L14: SQLite audit trail
L15: 3 Python dependencies (auditable in minutes)
```

## Known Operational Concerns

| Concern                                 | Impact                                   | Mitigation                                               |
| --------------------------------------- | ---------------------------------------- | -------------------------------------------------------- |
| Docker DNS breaks on network change     | Bot loses API access                     | Health check + auto-restart                              |
| Container drift from runtime installs   | Read-only rootfs blocks system installs  | Deps in Dockerfile (multi-stage build), rebuild when changed |
| `omarchy-update` restarts Docker daemon | Container dies                           | systemd `Restart=always` + crash-resilient design        |
| No AppArmor/SELinux on Arch             | Fewer kernel-level restrictions          | Accepted trade-off, Docker seccomp still active          |
| Mac Mini 2018 thermal throttling        | 20-40% perf loss during sustained builds | Workloads are bursty, chassis recovers between API waits |

## Roadmap

- **Now**: Reference repo — phase docs, manual setup, learn the system
- **Next**: `raly` CLI (npm package) — automate prerequisite checks, `raly up boot`, `raly down boot`, `raly status`
- **Later**: Drop-in assistant configs — community-contributed setups for different AI assistants

## References & Inspired By

- [RichardAtCT/claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) — Claude Code + Telegram bot running on bare metal. Proved the concept works. Used as RALY's POC test payload.
- [Anthropic Claude Code Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing) — Official sandboxing guidance that informed the container security model.
- [Omarchy](https://omarchy.com) by DHH / Basecamp — The opinionated Arch Linux distro that RALY's system layer is tested on. Desktop-first, Docker-native, Tailscale-ready.
- [Tailscale](https://tailscale.com) — Zero-config WireGuard VPN. Replaces openssh sshd entirely in RALY's architecture.
- [Docker](https://www.docker.com) — Container runtime. The security boundary between your assistant and your host.

## License

MIT
