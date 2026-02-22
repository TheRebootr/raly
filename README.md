# RALY — Run Assistants Locally Yourself

A personal learning journal about running AI assistants in Docker on my own machine.

## What This Is

This is a learning journal with a name. Not a framework, not a library, not a product.

I wanted to run AI assistants on my own hardware and understand what they can touch. I didn't know much about Linux security or Docker isolation when I started. So I learned, made mistakes, and wrote everything down in the `phases/` directory.

The "security model" is really just Docker flags (`--cap-drop ALL`, `--memory=8g`, `--pids-limit=512`) applied to a specific machine. Docker does the hard work. RALY documents which flags I chose, why, and what I learned along the way — including what I got wrong.

## What This Isn't

- **Not a security framework.** The 15 "security layers" listed below are mostly Docker's existing features plus standard host hardening. RALY didn't build them.
- **Not assistant-agnostic in a novel way.** Running different programs in Docker is just what Docker does. The POCs below prove the container works, not that RALY adds something on top.
- **Not a complete threat model.** The container protects the host, but the assistant has full network access and can read/write/delete everything in the mounted volumes. Prompt injection is explicitly your problem.

## What It's Actually Good For

- **Specificity over generic checklists.** This documents what I did on one Mac Mini running Omarchy (Arch Linux), not "here's a list of things you should probably do."
- **Honest tradeoff documentation.** No AppArmor on Arch — accepted. `--read-only` dropped because agents install packages — accepted. Most projects hide their compromises.

If you're in a similar situation — curious about AI assistants, cautious about giving them access, not a sysadmin — the `phases/` directory might save you some time. Or at least some of the same mistakes.

## Drop-In Tests

To validate the container works, I dropped real bots into it:

- [RichardAtCT/claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) (Python) — first test, validated the boundary end-to-end
- [linuz90/claude-telegram-bot](https://github.com/linuz90/claude-telegram-bot) (TypeScript/Bun) — second test, different runtime, same container

Both worked with zero changes to the container setup. That's not a RALY feature — that's Docker. But it confirmed the setup is correct.

## Boot

After enough evaluating, I started wanting an assistant that's just _mine_ — built the way I want, for the things I actually do.

That's Boot. Same idea as Omarchy — opinionated, scoped, built for one person's workflow. Talks to me over Telegram, runs Claude Code CLI inside the container, works on whatever I point it at. Not trying to be the best assistant. Just mine.

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
                   ├── Claude Code CLI (via Agent SDK, full network for API access)
                   ├── TypeScript/Bun bot (Telegram polling + multi-input handling)
                   ├── Volume: ~/BootDrive/workspace → /workspace
                   └── Volume: ~/BootDrive/data → /data (SQLite, config)
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

| Phase                                                 | Description                                         | Status |
| ----------------------------------------------------- | --------------------------------------------------- | ------ |
| [Phase 0](phases/00-preflight.md)                     | Pre-flight checks                                   | Done   |
| [Phase 1](phases/01-os-hardening.md)                  | OS-level hardening (sysctl, UFW, core dumps)        | Done   |
| [Phase 2](phases/02-boot-container.md)                | Container setup (Dockerfile, volumes, systemd)      | Done   |
| [Phase 3](phases/03-tailscale-setup.md)               | Tailscale ACL configuration                         | Done   |
| [POC: Python](phases/POC-Python-Assistant.md)         | Validation with RichardAtCT's bot as a drop-in test | Done   |
| [POC: TypeScript](phases/POC-Typescript-Assistant.md) | Validation with linuz90's bot as a second drop-in   | Done   |

### Boot Layer (reference assistant implementation)

| Phase                                             | Description                              | Status  |
| ------------------------------------------------- | ---------------------------------------- | ------- |
| [Phase 4](phases/boot/04-telegram-bot-setup.md)   | Telegram bot token + directory structure            | Done    |
| [Phase 5.1](phases/boot/05.1-security-module.md)  | Security (auth, rate limit, audit)                  | Fork    |
| [Phase 5.2](phases/boot/05.2-config-module.md)    | Config (env, MCP, safety prompts)                   | Fork    |
| [Phase 5.3](phases/boot/05.3-telegram-handler.md) | Telegram handlers (multi-input)                     | Fork    |
| [Phase 5.4](phases/boot/05.4-claude-executor.md)  | Claude session (Agent SDK)                          | Fork    |
| [Phase 5.5](phases/boot/05.5-session-memory.md)   | Session persistence + memory                        | Fork    |
| [Phase 5.6](phases/boot/05.6-cron-scheduler.md)   | Health checks + scheduled tasks                     | Planned |
| [Phase 6](phases/boot/06-verification.md)         | Full verification                                   | Planned |
| [Phase 7](phases/boot/07-ongoing-ops.md)          | Ongoing operations reference                        | Planned |

## Container Spec

```
Image:      node:22-bookworm-slim + Bun 1.3.9 + Claude Code CLI
Build:      Multi-stage (no compilers in runtime image)
Runtime:    --init (zombie reaping — NON-NEGOTIABLE)
Memory:     --memory=8g --memory-swap=12g
CPU:        --cpus=4
PIDs:       --pids-limit=512
User:       --user 1000:1000 (matches host UID)
Security:   --cap-drop ALL --security-opt=no-new-privileges
Optional:   --read-only (immutable rootfs — use for locked-down, non-agentic deployments)
Tmpfs:      /tmp (512MB, noexec) + /home/node (256MB, noexec) — when using --read-only
Volumes:    ~/BootDrive/workspace:/workspace (projects — blast radius)
            ~/BootDrive/data:/data (SQLite, config, Claude auth)
            Source baked into image (edit on host → rebuild → redeploy)
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
L5:  Container resource limits (8GB RAM, 4 CPUs, 512 PIDs)
L6:  No Docker socket, no --privileged, non-root user
L7:  Capability + privilege hardening (cap-drop ALL, no-new-privileges)
L8:  Immutable container (read-only rootfs, noexec tmpfs, source baked into image)
L9:  Multi-stage image (no compilers in runtime)
L10: Auth middleware + input validation (inside Boot)
L11: Telegram allowlist (single numeric user ID)
L12: Token bucket rate limiter
L13: Mounted volumes as explicit blast radius
L14: SQLite audit trail
L15: Minimal npm dependencies (grammy, claude-agent-sdk, mcp-sdk, openai, zod)
```

## Known Operational Concerns

| Concern                                 | Impact                                   | Mitigation                                                   |
| --------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| Docker DNS breaks on network change     | Bot loses API access                     | Health check + auto-restart                                  |
| Container drift from runtime installs   | Read-only rootfs blocks system installs  | Deps in Dockerfile (multi-stage build), rebuild when changed |
| `omarchy-update` restarts Docker daemon | Container dies                           | systemd `Restart=always` + crash-resilient design            |
| No AppArmor/SELinux on Arch             | Fewer kernel-level restrictions          | Accepted trade-off, Docker seccomp still active              |
| Mac Mini 2018 thermal throttling        | 20-40% perf loss during sustained builds | Workloads are bursty, chassis recovers between API waits     |

## Roadmap

- **Now**: Reference repo — phase docs, manual setup, learn the system
- **Next**: `raly` CLI (npm package) — automate prerequisite checks, `raly up boot`, `raly down boot`, `raly status`
- **Later**: Drop-in assistant configs — community-contributed setups for different AI assistants

## References & Inspired By

- [linuz90/claude-telegram-bot](https://github.com/linuz90/claude-telegram-bot) — TypeScript/Bun Telegram bot for Claude Code. Boot is forked from this. MIT license.
- [RichardAtCT/claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) — Python Claude Code + Telegram bot. Used as RALY's first POC test payload.
- [Anthropic Claude Code Sandboxing](https://www.anthropic.com/engineering/claude-code-sandboxing) — Official sandboxing guidance that informed the container security model.
- [Omarchy](https://omarchy.com) by DHH / Basecamp — The opinionated Arch Linux distro that RALY's system layer is tested on. Desktop-first, Docker-native, Tailscale-ready.
- [Tailscale](https://tailscale.com) — Zero-config WireGuard VPN. Replaces openssh sshd entirely in RALY's architecture.
- [Docker](https://www.docker.com) — Container runtime. The security boundary between your assistant and your host.

## License

MIT
