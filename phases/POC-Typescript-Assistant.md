# POC Validation: linuz90/claude-telegram-bot (TypeScript + Bun)

## Context

The Python POC (RichardAtCT's bot) validated the container security boundary end-to-end.
This second POC tests a different assistant drop-in — [linuz90/claude-telegram-bot](https://github.com/linuz90/claude-telegram-bot),
a TypeScript bot built on Bun + Grammy + the Claude Agent SDK (npm). The goal is to
confirm RALY's architecture is truly assistant-agnostic: same container, same security
flags, different runtime, different bot.

**Goal**: A working TypeScript Telegram bot inside the same Boot container, proving the
architecture handles non-Python assistants with minimal friction.

---

## Prerequisites

### Reusable from Python POC

If you already ran the Python POC, you have everything you need:

- [x] Claude Code CLI authenticated on the host (`claude /login` done)
- [x] Credentials at `~/.claude/.credentials.json`
- [x] Onboarding bypass at `~/BootDrive/data/.claude.json`
- [x] `settings.json` with `skipDangerousModePermissionPrompt`
- [x] Telegram bot token (from @BotFather)
- [x] Your Telegram user ID (from @userinfobot)
- [x] Host directories: `~/BootDrive/workspace`, `~/BootDrive/data`

If you haven't done the Python POC, follow Steps 1-4 in
[POC-Python-Assistant.md](POC-Python-Assistant.md) first for the host setup.

### New for this POC

- [ ] **OpenAI API key** (optional) — only needed for voice message transcription.
      Get one at platform.openai.com. The bot works fine without it — you just can't
      send voice messages.

---

## Step 1: Create Environment File

Create `~/BootDrive/data/.env.boot`:

```bash
cat > ~/BootDrive/data/.env.boot << 'EOF'
TELEGRAM_BOT_TOKEN=<your-bot-token>
TELEGRAM_ALLOWED_USERS=<your-telegram-user-id>
CLAUDE_WORKING_DIR=/workspace/projects
# OPENAI_API_KEY=<your-openai-key>    # Uncomment for voice support
# ANTHROPIC_API_KEY=                  # Leave unset — uses CLI auth (subscription)
EOF

chmod 600 ~/BootDrive/data/.env.boot
```

**Auth mode:** This bot uses the `@anthropic-ai/claude-agent-sdk` (npm package) which
auto-detects Claude Code CLI credentials. With `~/.claude/` mounted into the container,
it picks up your subscription auth — no API key needed.

**Note on `TELEGRAM_ALLOWED_USERS`:** Comma-separated if you want multiple users.
For the POC, just your own user ID.

---

## Step 2: Build and Start

Everything is defined in `~/BootDrive/compose.yml`. The Dockerfile at
`~/BootDrive/app/Dockerfile` bakes in Bun, the bot source, all npm dependencies,
and CLI tools (ripgrep, fd, jq, tree) via a multi-stage build — no manual
installation needed.

```bash
cd ~/BootDrive

# Build the image (first time or after code changes)
docker compose build

# Start the bot
docker compose up -d

# Follow logs to verify startup
docker compose logs -f
```

You should see:

```
Boot  | Config loaded: 1 allowed users, working dir: /workspace/projects
Boot  | Claude Telegram Bot - TypeScript Edition
Boot  | Bot started: @YourBotName
```

Press `Ctrl+C` to stop following logs (the bot keeps running in the background).

**What compose.yml handles for you:** `--init`, `--memory=8g`, `--memory-swap=12g`,
`--cpus=4`, `--pids-limit=512`, `--cap-drop ALL`, `--security-opt=no-new-privileges`,
all volume mounts, and env vars from `.env`. Same security flags as the Python POC,
declared once in a file instead of a 15-line shell command.

**Source is baked into the image.** To change code: edit on host → `docker compose build`
→ `docker compose up -d`. There is no `/app` volume mount.

---

## Step 3: Verify Auth

Shell into the running container to verify Claude Code CLI works:

```bash
docker compose exec Boot bash

# Inside the container:
claude -p "say hello" --dangerously-skip-permissions
# Should get a response — proves subscription auth works

# Verify ANTHROPIC_API_KEY is NOT set (would switch to pay-per-token)
echo $ANTHROPIC_API_KEY    # should be empty

# Verify CLI tools are available
rg --version               # ripgrep
fd --version               # fd-find
jq --version               # jq
tree --version             # tree

exit
```

---

## Step 4: Test from Telegram

Open Telegram and message your bot:

1. **Text**: Send "hello" — should get a response
2. **Working directory**: Send "what directory are you in?" — should report `/workspace/projects`
3. **File creation**: Send "create a file called test.txt with hello world" — should create it
4. **Verify from host**: `ls ~/BootDrive/workspace/projects/test.txt`
5. **Commands**: Try `/new` (fresh session), `/status` (check state)
6. **Photo** (optional): Send a photo with "describe this" — should analyze it
7. **Voice** (optional, needs OpenAI key): Send a voice message — should transcribe and respond

**If text works — the drop-in is validated.** Voice and photos are bonus features.

---

## Step 5: Boundary Testing

The security boundary is identical to the Python POC — same container image, same
flags, same mounts. If you already ran boundary tests in Step 6 of
[POC-Python-Assistant.md](POC-Python-Assistant.md), you don't need to repeat them.

The container boundary doesn't change based on what runs inside it. That's the point.

If you want a quick sanity check:

```bash
docker compose exec Boot bash

# Still can't read host shadow file
cat /etc/shadow                    # → Permission denied

# Still no capabilities
grep NoNewPrivs /proc/self/status  # → NoNewPrivs: 1

# Still PID 1 is tini
cat /proc/1/comm                   # → docker-init

# Resource limits still enforced
cat /sys/fs/cgroup/memory.max      # → 8589934592
cat /sys/fs/cgroup/pids.max        # → 512
```

---

## Step 6: Managing the Bot

```bash
cd ~/BootDrive

docker compose up -d       # start in background
docker compose logs -f     # follow logs
docker compose restart     # restart the bot
docker compose down        # stop and remove container
docker compose build       # rebuild after code/Dockerfile changes
docker compose up -d       # start with rebuilt image

# QMD sidecar (not started by default — future use)
docker compose --profile search up -d    # start Boot + QMD
docker compose --profile search down     # stop both
```

---

## Comparison: Python vs TypeScript POC

| Aspect             | Python (RichardAtCT)             | TypeScript (linuz90)                   |
| ------------------ | -------------------------------- | -------------------------------------- |
| Runtime            | Python 3.12 + Poetry 2.x         | Bun 1.3.9                              |
| SDK                | `claude-agent-sdk` (Python)      | `@anthropic-ai/claude-agent-sdk` (npm) |
| Bot framework      | Custom                           | Grammy (^1.38.4)                       |
| Auth mode          | CLI subprocess (`USE_SDK=false`) | Agent SDK with CLI auth detection      |
| Input types        | Text only                        | Text, voice, photos, docs, video       |
| Session management | Basic                            | Sessions with `/resume`, history       |
| Install            | `poetry install --without dev`   | `docker compose build` (baked in)      |
| Run command        | `poetry run claude-telegram-bot` | `docker compose up -d`                 |
| CLI tools          | None                             | ripgrep, fd, jq, tree (baked in)       |
| Extra deps         | None                             | OpenAI key (for voice only)            |

**Key takeaway:** Both bots work in the same container with zero changes to the security
model. The container boundary is assistant-agnostic — exactly as designed.

---

## Cleanup

Once validated and ready to move on:

```bash
cd ~/BootDrive
docker compose down                          # stop and remove container
docker rmi bootdrive-boot                    # remove built image (optional)
rm ~/BootDrive/app/.env                # remove secrets
# Keep ~/BootDrive/workspace, ~/BootDrive/data — these are production dirs
```
