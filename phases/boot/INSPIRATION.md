# Boot — Architectural Inspiration Map

## Three Reference Repos

We're building Boot from scratch, but we're not designing from scratch. These three
working repos provide proven patterns for each layer of the stack.

| Repo | What It Is | License | Key Strength |
|------|-----------|---------|--------------|
| [HKUDS/nanobot](https://github.com/HKUDS/nanobot) | Lightweight personal AI assistant framework (~4K lines Python) | — | Overall agent architecture, multi-channel gateway, LLM abstraction, tool system, config |
| [RichardAtCT/claude-code-telegram](https://github.com/RichardAtCT/claude-code-telegram) | Claude Code Telegram bot (Python) | — | Claude CLI subprocess pattern, subscription auth, Telegram bot lifecycle, agentic mode |
| [linuz90/claude-telegram-bot](https://github.com/linuz90/claude-telegram-bot) | Claude Telegram bot (TypeScript/Bun) | — | Grammy patterns, session management, multi-input (voice/photo/docs), Agent SDK auth |

Plus [seigneurcui/memubot](https://github.com/seigneurcui/memubot) (Apache 2.0) for
structured memory — already documented in Phase 5.8.

---

## Pattern Map: Which Repo Informs Which Phase

### Phase 5.2 — config.py
**Primary: nanobot**
- YAML-based config with sensible defaults
- Multi-provider LLM config (we only need Claude, but the abstraction pattern is clean)
- Gateway/channel config separated from agent config

**Secondary: RichardAtCT**
- ENV-based config for secrets (bot token, API keys)
- `USE_SDK`, `AGENTIC_MODE`, `DEBUG` as feature flags

**Boot approach:** ENV for secrets, YAML (or simple Python dataclass) for structure.
Don't over-abstract — Boot has one LLM (Claude) and one channel (Telegram).

### Phase 5.3 — telegram.py
**Primary: linuz90 (TypeScript)**
- Grammy framework patterns (clean handler registration, middleware chain)
- Multi-input handling: text, photos, voice, documents
- Session management with `/new`, `/resume`, `/status` commands
- Allowed-user filtering as middleware

**Secondary: RichardAtCT**
- Telegram long-polling (no webhooks, no inbound ports — matches our security model)
- `ALLOWED_USERS` single-ID allowlist
- Conversation tracking per user

**Secondary: nanobot**
- Channel abstraction — Telegram as one implementation of a generic "gateway" interface
- Message routing: user input → agent → response → channel

**Boot approach:** python-telegram-bot or aiogram (Python equivalents of Grammy).
Long-polling only. Start with text, add photo/voice/doc support later.
Borrow linuz90's command set (`/new`, `/status`) and RichardAtCT's allowlist pattern.

### Phase 5.4 — executor.py
**Primary: RichardAtCT**
- Claude Code CLI as subprocess (`USE_SDK=false`)
- `--dangerously-skip-permissions` for non-interactive execution
- Subscription auth via mounted `~/.claude/` credentials
- Stdout/stderr capture, timeout handling
- `APPROVED_DIRECTORY` for workspace scoping

**Secondary: linuz90**
- Agent SDK approach (alternative to subprocess)
- Session/conversation resume across invocations

**Boot approach:** CLI subprocess (proven in POC). Keep executor.py at ~100 lines.
The RichardAtCT pattern is exactly what we validated.

### Phase 5.5 — session.py + memory.py
**Primary: RichardAtCT**
- SQLite-backed session state
- Conversation history per project
- Database URL config (`sqlite:////data/bot.db`)

**Secondary: linuz90**
- Session resume across bot restarts
- History export

**Boot approach:** Already specified in Phase 5.5. SQLite + aiosqlite, per-project
conversation history, context injection into CLI calls.

### Phase 5.8 — structured memory
**Primary: memubot** (already documented in Phase 5.8)

### Overall Agent Loop
**Primary: nanobot**
- Lightweight agent loop: receive message → build context → call LLM → return response
- ~4K lines total — proof that a capable agent doesn't need framework bloat
- Tool/capability registration system
- Gateway abstraction (even though we only need Telegram, the pattern keeps things clean)

**Boot approach:** Keep it simple. The agent loop is:
```
Telegram message
  → auth check (5.1)
  → session lookup (5.5)
  → retrieve relevant memories (5.8)
  → build context (history + memories + CLAUDE.md)
  → executor.py → Claude CLI subprocess
  → save response to history (5.5)
  → extract memories async (5.8)
  → send response to Telegram (5.3)
```

---

## Key Lessons from Each Repo

### nanobot — "Keep it small"
- Full agent framework in ~4K lines of Python
- Proves you don't need LangChain, CrewAI, or heavyweight frameworks
- Clean separation: config / gateway / agent / LLM / tools
- Multi-provider LLM abstraction done right (simple interface, provider-specific impl)

### RichardAtCT — "CLI subprocess works"
- Claude Code CLI as subprocess is production-viable
- Subscription auth (no API key) works via mounted credentials
- SQLite is sufficient for bot state
- Poetry project structure with proper dependency management

### linuz90 — "Good UX patterns"
- `/new` (fresh session), `/resume` (continue), `/status` (check state) — users need these
- Multi-input from day one (photos, voice, docs) — Telegram supports it, why not use it
- Grammy's middleware pattern (auth → rate limit → handler) is clean and composable
- Agent SDK can auto-detect CLI credentials — alternative to raw subprocess

---

## What Boot Does NOT Borrow

- **nanobot's multi-provider LLM support** — Boot only uses Claude. No OpenAI/DeepSeek/etc.
- **nanobot's multi-channel gateway** — Boot only uses Telegram. No Discord/Slack/WhatsApp.
- **linuz90's Bun runtime** — Boot is Python. Bun was validated but not our stack.
- **RichardAtCT's FastAPI/APScheduler/structlog** — Too many deps. Boot targets ~3 deps.
- **memubot's pgvector/OpenAI embeddings** — SQLite + Claude, no external embedding service.
- **Any framework's LangChain dependency** — Direct LLM calls, no abstraction layers.

---

## The 3-Dependency Target

Boot aims for minimal external dependencies (from OVERVIEW.md):

1. **python-telegram-bot** (or aiogram) — Telegram interface
2. **aiosqlite** — Async SQLite for state/memory
3. **Claude Code CLI** — LLM backend (system dependency, not pip)

Everything else is stdlib. This is achievable because:
- nanobot proves a full agent fits in ~4K lines with minimal deps
- RichardAtCT proves Claude CLI subprocess needs zero extra libraries
- The memory system (5.8) uses stdlib sqlite3 + optional numpy later
