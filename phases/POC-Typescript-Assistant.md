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
- [x] Onboarding bypass at `~/boot-data/.claude.json`
- [x] `settings.json` with `skipDangerousModePermissionPrompt`
- [x] Telegram bot token (from @BotFather)
- [x] Your Telegram user ID (from @userinfobot)
- [x] Host directories: `~/boot-workspace`, `~/boot-data`
- [x] `boot:latest` Docker image built and tested

If you haven't done the Python POC, follow Steps 1-4 in
[POC-Python-Assistant.md](POC-Python-Assistant.md) first for the host setup.

### New for this POC

- [ ] **OpenAI API key** (optional) — only needed for voice message transcription.
      Get one at platform.openai.com. The bot works fine without it — you just can't
      send voice messages.

---

## Step 1: Create Environment File

Create `~/boot-data/.env.ts-poc`:

```bash
cat > ~/boot-data/.env.ts-poc << 'EOF'
TELEGRAM_BOT_TOKEN=<your-bot-token>
TELEGRAM_ALLOWED_USERS=<your-telegram-user-id>
CLAUDE_WORKING_DIR=/workspace/projects
# OPENAI_API_KEY=<your-openai-key>    # Uncomment for voice support
# ANTHROPIC_API_KEY=                  # Leave unset — uses CLI auth (subscription)
EOF

chmod 600 ~/boot-data/.env.ts-poc
```

**Auth mode:** This bot uses the `@anthropic-ai/claude-agent-sdk` (npm package) which
auto-detects Claude Code CLI credentials. With `~/.claude/` mounted into the container,
it picks up your subscription auth — no API key needed.

**Note on `TELEGRAM_ALLOWED_USERS`:** Comma-separated if you want multiple users.
For the POC, just your own user ID.

---

## Step 2: Start the Container

Same container, same security flags as the Python POC. The only difference is the
bot code you run inside it.

```bash
docker run -it \
  --name boot-ts-poc \
  --init \
  --memory=4g \
  --memory-swap=6g \
  --cpus=4 \
  --pids-limit=512 \
  --user 1000:1000 \
  --security-opt=no-new-privileges \
  --cap-drop ALL \
  -v ~/boot-workspace:/workspace \
  -v ~/boot-data:/data \
  -v ~/.claude:/home/node/.claude \
  -v ~/boot-data/.claude.json:/home/node/.claude.json \
  boot:latest \
  bash
```

You're now inside the Boot container. Everything below happens inside.

---

## Step 3: Install Bun

The `boot:latest` image has Node 22 but not Bun. Install it inside the container:

```bash
curl -fsSL https://bun.sh/install | bash
export PATH="$HOME/.bun/bin:$PATH"
bun --version    # verify 1.x
```

**Why Bun?** This bot uses Bun-specific APIs and won't run on Node.js. Bun is a
single binary — the install is fast and self-contained under `~/.bun/`.

**Persistence:** Bun installs to `~/.bun/` which lives on the container's writable
filesystem. It survives `docker stop`/`start` but NOT `docker rm` + re-run. For the
POC this is fine.

---

## Step 4: Clone and Configure

```bash
cd /workspace
git clone https://github.com/linuz90/claude-telegram-bot.git ts-poc-bot
cd ts-poc-bot

# Install dependencies
bun install
```

Load environment variables:

```bash
export $(grep -v '^#' /data/.env.ts-poc | grep -v '^$' | xargs)
```

---

## Step 5: Verify Auth and Run

```bash
# Verify Claude Code CLI works (same test as Python POC)
claude -p "say hello" --dangerously-skip-permissions
# Should get a response — proves subscription auth works from inside the container

# Verify ANTHROPIC_API_KEY is NOT set (would switch to pay-per-token)
echo $ANTHROPIC_API_KEY    # should be empty

# Create the working directory
mkdir -p /workspace/projects

# Run the bot
bun run src/index.ts
```

You should see the bot start up and connect to Telegram.

---

## Step 6: Test from Telegram

Open Telegram and message your bot:

1. **Text**: Send "hello" — should get a response
2. **Working directory**: Send "what directory are you in?" — should report `/workspace/projects`
3. **File creation**: Send "create a file called test.txt with hello world" — should create it
4. **Verify from host**: `ls ~/boot-workspace/projects/test.txt`
5. **Commands**: Try `/new` (fresh session), `/status` (check state)
6. **Photo** (optional): Send a photo with "describe this" — should analyze it
7. **Voice** (optional, needs OpenAI key): Send a voice message — should transcribe and respond

**If text works — the drop-in is validated.** Voice and photos are bonus features.

---

## Step 7: Boundary Testing

The security boundary is identical to the Python POC — same container image, same
flags, same mounts. If you already ran boundary tests in Step 6 of
[POC-Python-Assistant.md](POC-Python-Assistant.md), you don't need to repeat them.

The container boundary doesn't change based on what runs inside it. That's the point.

If you want a quick sanity check:

```bash
# Still can't read host shadow file
cat /etc/shadow                    # → Permission denied

# Still no capabilities
grep NoNewPrivs /proc/self/status  # → NoNewPrivs: 1

# Still PID 1 is tini
cat /proc/1/comm                   # → docker-init

# Resource limits still enforced
cat /sys/fs/cgroup/memory.max      # → 4294967296
cat /sys/fs/cgroup/pids.max        # → 512
```

---

## Step 8: Keep Running (optional)

Detach from the container (Ctrl+P, Ctrl+Q) or run in the background:

```bash
# From outside the container:
docker exec -d boot-ts-poc bash -c 'export PATH="$HOME/.bun/bin:$PATH" && cd /workspace/ts-poc-bot && export $(grep -v "^#" /data/.env.ts-poc | grep -v "^$" | xargs) && bun run src/index.ts'
```

---

## Comparison: Python vs TypeScript POC

| Aspect             | Python (RichardAtCT)             | TypeScript (linuz90)                   |
| ------------------ | -------------------------------- | -------------------------------------- |
| Runtime            | Python 3.12 + Poetry 2.x         | Bun 1.0+                               |
| SDK                | `claude-agent-sdk` (Python)      | `@anthropic-ai/claude-agent-sdk` (npm) |
| Bot framework      | Custom                           | Grammy (^1.38.4)                       |
| Auth mode          | CLI subprocess (`USE_SDK=false`) | Agent SDK with CLI auth detection      |
| Input types        | Text only                        | Text, voice, photos, docs, video       |
| Session management | Basic                            | Sessions with `/resume`, history       |
| Install            | `poetry install --without dev`   | `bun install`                          |
| Run command        | `poetry run claude-telegram-bot` | `bun run src/index.ts`                 |
| Extra deps         | None                             | OpenAI key (for voice only)            |

**Key takeaway:** Both bots work in the same container with zero changes to the security
model. The container boundary is assistant-agnostic — exactly as designed.

---

## Cleanup

Once validated and ready to move on:

```bash
docker stop boot-ts-poc && docker rm boot-ts-poc
rm -rf ~/boot-workspace/ts-poc-bot    # remove linuz90's code
rm ~/boot-data/.env.ts-poc            # remove POC config
# Keep ~/boot-workspace, ~/boot-data — these are production dirs
```
