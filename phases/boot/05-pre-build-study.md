# Phase 5 Pre-Build: Research and Pattern Extraction

## Context

RALY's Boot harness is a custom Python process (~500-800 lines) that receives
Telegram messages, validates through a security module, and delegates Claude Code CLI
tasks inside ephemeral Docker containers. Before writing any code, study existing
implementations to extract proven patterns. This research directly informs the
security-first architecture of Phases 5.1-5.7.

## Prerequisites

- None — this phase can run in parallel with Phases 0-4
- Internet access required for repository study

## Purpose

This is NOT a "read everything" exercise. It's targeted extraction of specific patterns
that will be implemented in Boot. Each study target has explicit questions to answer.

## Study Targets

### Target 1: RichardAtCT/claude-code-telegram (Primary — 2-3 hours)

Repository: https://github.com/RichardAtCT/claude-code-telegram

This is the most architecturally relevant reference. Study order:

#### 1a. Entry Point — Trace Request Lifecycle

Read `src/main.py` or equivalent entry point.
Questions to answer:
- How does the app bootstrap? What's the startup sequence?
- How are Telegram handlers registered?
- How does a message flow from receipt to Claude execution?
- What middleware chain exists between message receipt and action?

#### 1b. Security Module — THE GOLD

Read `src/security/` (all files in this directory).
Questions to answer:
- How is Telegram user authentication implemented?
- Is it middleware-based or per-handler?
- How is rate limiting implemented? Token bucket? Sliding window?
- What input validation exists? Path traversal? Injection?
- How is audit logging structured? What fields are captured?
- Are there any security patterns we should adopt verbatim?
- Are there any security gaps we should avoid?

#### 1c. Config Module

Read `src/config/` (all files).
Questions to answer:
- How are secrets loaded? Environment variables? File?
- How is config validated at startup?
- Is config typed/structured or loose dictionaries?
- What happens on invalid config — silent default or loud failure?

#### 1d. Claude Execution

Read `src/claude/` (all files).
Questions to answer:
- How is Claude Code CLI invoked? subprocess? SDK?
- How is the subprocess managed? Timeout? Kill?
- How is output captured and returned to the user?
- Error handling — retries? Backoff?
- Is Docker sandboxing used? If so, what flags?

#### 1e. Message Handlers

Read `src/handlers/` (all files).
Questions to answer:
- What commands are supported?
- How is session/context managed per conversation?
- How are long responses handled (Telegram's 4096 char limit)?
- How are file uploads/downloads handled?
- Queue management for concurrent requests?

### Target 2: linuz90/claude-telegram-bot (Secondary — 1 hour)

Repository: https://github.com/linuz90/claude-telegram-bot

#### 2a. Bot Setup (TypeScript/grammY)

Read `src/index.ts` or main entry.
Questions to answer:
- How is the bot initialized?
- Any clever patterns in message handling?
- How does it handle Telegram API rate limits?

#### 2b. CLAUDE.md Personality Pattern

Read the CLAUDE.md file and any personality configuration.
Questions to answer:
- How is the bot's personality/behavior defined?
- Is CLAUDE.md per-project or global?
- What instructions produce the best Claude behavior?

#### 2c. MCP Integration (for future reference)

Read `ask_user_mcp/` if it exists.
Questions to answer:
- How do interactive buttons work in Telegram via MCP?
- Is this pattern worth adopting for Boot?
- (This is for future enhancement, not initial build)

### Target 3: HKUDS/nanobot (Secondary — 1-2 hours)

Repository: https://github.com/HKUDS/nanobot

Ultra-lightweight personal AI assistant framework (~4K lines Python). The best reference
for Boot's overall agent architecture — proves a full agent doesn't need framework bloat.

#### 3a. Agent Loop

Read the core agent entry point.
Questions to answer:
- How does the main agent loop work? (receive → context → LLM → respond)
- How is conversation context built before each LLM call?
- How are tools/capabilities registered and invoked?

#### 3b. Gateway / Channel Abstraction

Read the gateway/channel layer.
Questions to answer:
- How does nanobot abstract over Telegram/Discord/etc.?
- Is the abstraction worth adopting, or overkill for single-channel Boot?
- How does message routing work (user input → agent → response → channel)?

#### 3c. LLM Abstraction

Read the LLM provider layer.
Questions to answer:
- How does it support multiple LLM backends (Anthropic, OpenAI, etc.)?
- What's the interface a provider must implement?
- Can we simplify this for Claude-only Boot?

#### 3d. Config System

Read config loading and validation.
Questions to answer:
- YAML-based config — is this cleaner than ENV-only?
- How are defaults structured?
- What's the separation between agent config and channel config?

### Target 4: Anthropic's Claude Code Sandboxing Post (30 min)

URL: https://www.anthropic.com/engineering/claude-code-sandboxing

Questions to answer:
- What is Claude Code's built-in `/sandbox` mode?
- What isolation does it provide? (filesystem, network, IPC)
- Does it use bubblewrap on Linux?
- Can it replace our Docker wrapping?
- Can it be layered WITH our Docker wrapping (belt and suspenders)?
- What are the limitations?

### Target 5: godagoo/claude-telegram-relay (Quick scan — 15 min)

Repository: https://github.com/godagoo/claude-telegram-relay

Questions to answer:
- How are systemd service templates structured?
- Any useful patterns for service management?
- How is auto-restart configured?

## Deliverables

See also: [`INSPIRATION.md`](INSPIRATION.md) — maps which repo informs which Boot phase.

After completing all study targets, produce a single document:
`~/BootDrive/app/RESEARCH-NOTES.md`

Structure:

```markdown
# Boot Pre-Build Research Notes

## Security Patterns to Adopt
- [pattern]: [source]: [how to implement in Boot]
- ...

## Config Patterns to Adopt
- ...

## Execution Patterns to Adopt
- ...

## Message Handling Patterns to Adopt
- ...

## Patterns to AVOID
- [pattern]: [source]: [why it's wrong for Boot]
- ...

## Open Questions Resolved
- Claude Code sandboxing: [can we use built-in /sandbox?] [answer]
- ...

## Dependency Decisions
- python-telegram-bot: [version] [why]
- aiosqlite: [version] [why]
- [any others?]
```

## Verification Checklist

- [ ] RichardAtCT/claude-code-telegram: security module fully read and patterns extracted
- [ ] RichardAtCT: request lifecycle traced end-to-end
- [ ] RichardAtCT: config validation pattern documented
- [ ] RichardAtCT: Claude CLI execution approach documented
- [ ] RichardAtCT: message handler patterns documented
- [ ] linuz90/claude-telegram-bot: CLAUDE.md personality pattern extracted
- [ ] linuz90: any useful grammY patterns noted
- [ ] HKUDS/nanobot: agent loop pattern documented
- [ ] HKUDS/nanobot: gateway/channel abstraction evaluated
- [ ] HKUDS/nanobot: LLM abstraction pattern documented
- [ ] HKUDS/nanobot: config system approach noted
- [ ] Anthropic sandboxing post read: built-in sandbox capabilities documented
- [ ] Decision made: Docker-only vs Docker+built-in sandbox vs built-in-only
- [ ] godagoo: systemd service template patterns extracted
- [ ] RESEARCH-NOTES.md written to ~/BootDrive/app/
- [ ] All "Patterns to Adopt" have concrete implementation notes
- [ ] All "Patterns to AVOID" have clear rationale

## Outputs for Downstream Phases

- Security patterns → Phase 5.1 (security.py) [RichardAtCT]
- Config patterns → Phase 5.2 (config.py) [nanobot, RichardAtCT]
- Message handling → Phase 5.3 (telegram.py) [linuz90, RichardAtCT, nanobot]
- Execution patterns → Phase 5.4 (executor.py) [RichardAtCT]
- CLAUDE.md template → Phase 5.5 (memory.py) [linuz90]
- Agent loop architecture → Phase 5.3/5.4 (overall flow) [nanobot]
- Structured memory patterns → Phase 5.8 (knowledge.py) [memubot]
- systemd templates → Phase 5.6 (cron/scheduler) [godagoo]
- Sandbox decision → Phase 5.4 [Anthropic blog]

## Internet Validation Instruction

Before executing this phase, perform a web search for:
- "RichardAtCT claude-code-telegram GitHub"
- "linuz90 claude-telegram-bot GitHub"
- "Anthropic claude code sandboxing blog post"
- "godagoo claude-telegram-relay GitHub"
- "claude code telegram bot Python 2025 2026"

Verify that:
1. All repositories still exist and are publicly accessible
2. The directory structures described above match the current state of the repos
3. No newer, better reference implementations have emerged since this plan was written
4. The Anthropic sandboxing blog post URL is correct and accessible
5. Check if any of these projects have been archived, deprecated, or superseded
