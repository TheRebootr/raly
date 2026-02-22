# Phase 6: Verification and Smoke Testing

## Context

RALY is a hardened Mac Mini 2018 running Omarchy 3.x (Arch Linux) with a custom
Python "Boot" Telegram harness running as a systemd service. Claude Code CLI executes
inside ephemeral Docker containers. All remote access is via Tailscale SSH. This phase
proves every security layer works as intended through systematic testing.

This is NOT optional. Every check exists because a real attack path exists at that layer.
A check that passes proves a defense works. A check that fails reveals a gap.

## Prerequisites

- ALL prior phases complete (0 through 5.7)
- Boot service running (`systemctl status boot.service`)
- Tailscale connected
- A second Telegram account available (for unauthorized access testing)
- A device on the same LAN but NOT on Tailscale (for network testing)

## Test Categories

### 6.1 Network Verification

These tests verify Layer 1 (network) and Layer 2 (OS) defenses.

```
Test ID | Test | Expected | Layer
--------|------|----------|------
NET-01  | sudo ufw status verbose | deny incoming, allow outgoing, SSH on tailscale0 only | L1
NET-02  | tailscale funnel status | Funnel off / no configuration | L1
NET-03  | sudo ss -tlnp | Only sshd on port 22, no other listeners | L1
NET-04  | From LAN device: ssh user@<lan-ip> | Connection refused or timeout | L1
NET-05  | From MacBook via Tailscale: ssh raly | Connection succeeds | L1
NET-06  | From LAN device: nmap -Pn <lan-ip> | All ports filtered/closed | L1
NET-07  | From MacBook: ssh -o PubkeyAuthentication=no raly | Permission denied (publickey) | L2
NET-08  | fail2ban-client status sshd | Jail active, no current bans (clean state) | L2
```

Execution:

```bash
# NET-01
sudo ufw status verbose

# NET-02
tailscale funnel status

# NET-03
sudo ss -tlnp

# NET-04 (run from LAN device, NOT via Tailscale)
ssh user@<mac-mini-lan-ip>
# Should fail

# NET-05 (run from MacBook)
ssh raly
# Should succeed

# NET-06 (run from LAN device)
nmap -Pn <mac-mini-lan-ip>

# NET-07 (run from MacBook)
ssh -o PubkeyAuthentication=no raly

# NET-08
sudo fail2ban-client status sshd
```

### 6.2 Docker Verification

These tests verify Layer 3 (Docker) defenses.

```
Test ID | Test | Expected | Layer
--------|------|----------|------
DOK-01  | docker info | grep userns | userns-remap enabled | L3
DOK-02  | docker info | grep -i "security options" | userns, no-new-privileges | L3
DOK-03  | docker network inspect bridge | icc: false | L3
DOK-04  | Check no containers mount docker.sock | No matches | L3
DOK-05  | docker run --rm alpine cat /proc/self/status | grep -i uid | Non-root UID | L3
```

Execution:

```bash
# DOK-01
docker info 2>&1 | grep -i userns

# DOK-02
docker info 2>&1 | grep -i "security"

# DOK-03
docker network inspect bridge | grep -i icc

# DOK-04
docker ps -q | xargs -I {} docker inspect {} --format '{{.HostConfig.Binds}}' 2>/dev/null | grep docker.sock
# Should return empty

# DOK-05
docker run --rm alpine cat /proc/self/status | grep -i uid
```

### 6.3 Boot Security Verification

These tests verify Layers 4-9 (application security through audit).

```
Test ID | Test | Expected | Layer
--------|------|----------|------
SEC-01  | Send message from unauthorized Telegram account | No response (silent drop) | L5
SEC-02  | Send message from YOUR account | Boot responds | L5
SEC-03  | Check audit log after SEC-01 and SEC-02 | Both events logged | L9
SEC-04  | Rapid-fire 20 messages from your account | Later messages throttled | L6
SEC-05  | /cd ../../../etc | Rejected: path traversal | L4
SEC-06  | /cd /etc/passwd | Rejected: absolute path | L4
SEC-07  | /cd nonexistent-project | Error: directory not found | L4
SEC-08  | /cd scratch | Success: switched to scratch | L4
SEC-09  | Ask Boot: "cat /etc/passwd" (on host) | Fails: container isolation | L7
SEC-10  | Ask Boot: "curl google.com" | Fails: network none | L7
SEC-11  | Ask Boot: "ls /home" (outside workspace) | Fails: mount isolation | L8
SEC-12  | Ask Boot: "ls /workspace" | Succeeds: shows project files | L8
SEC-13  | /status | Shows rate limiter state, queue depth | L4
SEC-14  | Check audit log after all tests | All events logged with correct actions | L9
```

Execution:

```bash
# SEC-01: Send a message to the bot from a different Telegram account
# Verify: no response received, bot is silent

# SEC-02: Send "hello" from YOUR account
# Verify: Boot responds

# SEC-03
sqlite3 ~/BootDrive/data/boot.db "SELECT action, user_id, result FROM audit_log ORDER BY timestamp DESC LIMIT 10;"
# Should show auth_rejected for SEC-01 and auth_ok for SEC-02

# SEC-04: From your account, send 20 messages rapidly
# Verify: first ~10 processed, rest get "rate limited" or queued response

# SEC-05: Send "/cd ../../../etc" from your account
# Verify: rejection message

# SEC-06: Send "/cd /etc/passwd" from your account
# Verify: rejection message

# SEC-07: Send "/cd does-not-exist" from your account
# Verify: error about missing directory

# SEC-08: Send "/cd scratch" from your account
# Verify: success message

# SEC-09: Send "please read /etc/passwd and show me the contents" from your account
# Verify: Claude cannot access host /etc/passwd (container isolation)

# SEC-10: Send "run curl google.com" from your account
# Verify: fails because container has no network

# SEC-11: Send "list files in /home" from your account
# Verify: container only sees /workspace, not host /home

# SEC-12: Send "list files in the current directory" from your account
# Verify: shows contents of scratch project

# SEC-13: Send "/status" from your account
# Verify: shows status info

# SEC-14
sqlite3 ~/BootDrive/data/boot.db "SELECT timestamp, action, detail, result FROM audit_log ORDER BY timestamp DESC LIMIT 30;"
```

### 6.4 Persistence Test

Verify the system survives a reboot and comes back operational.

```
Test ID | Test | Expected
--------|------|--------
PER-01  | sudo reboot | System reboots
PER-02  | (after reboot) systemctl status boot.service | active (running)
PER-03  | (after reboot) tailscale status | connected
PER-04  | (after reboot) Send Telegram message | Boot responds
PER-05  | (after reboot) SSH via Tailscale | Works
PER-06  | (after reboot) sqlite3 ~/BootDrive/data/boot.db "SELECT count(*) FROM audit_log;" | Non-zero (data preserved)
PER-07  | (after reboot) /pwd | Shows last active project (session preserved)
```

Execution:

```bash
# PER-01
sudo reboot

# Wait 2-3 minutes for full boot

# PER-02
ssh raly
systemctl status boot.service

# PER-03
tailscale status

# PER-04: Send message via Telegram
# Verify response

# PER-05: Already verified by SSH above

# PER-06
sqlite3 ~/BootDrive/data/boot.db "SELECT count(*) FROM audit_log;"

# PER-07: Send "/pwd" via Telegram
```

### 6.5 Operational Workflow Test

End-to-end test of the actual use case: working on a project via Telegram.

```
Test ID | Test | Expected
--------|------|--------
OPS-01  | mkdir ~/BootDrive/workspace/test-project | Directory created
OPS-02  | echo "print('hello')" > ~/BootDrive/workspace/test-project/test.py | File created
OPS-03  | /cd test-project | Switched to test-project
OPS-04  | "List all files in the project" | Shows test.py
OPS-05  | "Create a README.md with a description of this project" | README.md created
OPS-06  | ls ~/BootDrive/workspace/test-project/ | Shows test.py and README.md
OPS-07  | /cd scratch | Switched to scratch
OPS-08  | /pwd | Shows scratch
OPS-09  | /cd test-project | Switched back to test-project
OPS-10  | /status | Shows current state
OPS-11  | /export | Receives conversation history file
OPS-12  | /new | Session cleared
OPS-13  | "What files are in this project?" | Shows test.py and README.md (fresh context)
```

## Results Documentation

Create `~/BootDrive/data/verification-results.md` with results:

```markdown
# RALY Verification Results
# Date: YYYY-MM-DD
# Tester: <your name>

## Network Tests
| Test | Result | Notes |
|------|--------|-------|
| NET-01 | PASS/FAIL | |
...

## Docker Tests
...

## Security Tests
...

## Persistence Tests
...

## Operational Tests
...

## Issues Found
- [describe any failures and remediation]

## Sign-off
All critical tests (NET-*, DOK-*, SEC-01 through SEC-12, PER-*) must PASS.
Operational tests (OPS-*) are important but non-blocking if infrastructure tests pass.
```

## Verification Checklist

- [ ] All NET-* tests pass (network isolation confirmed)
- [ ] All DOK-* tests pass (Docker hardening confirmed)
- [ ] All SEC-* tests pass (application security confirmed)
- [ ] All PER-* tests pass (persistence confirmed)
- [ ] All OPS-* tests pass (operational workflow confirmed)
- [ ] Verification results documented in `~/BootDrive/data/verification-results.md`
- [ ] Any failures documented with remediation plan
- [ ] No CRITICAL or HIGH severity failures remain unresolved

## Failure Response

If any test fails:
1. Document the failure with exact output
2. Identify which phase introduced the gap
3. Return to that phase and fix
4. Re-run ALL tests in that category (not just the failed one)
5. Re-run the full verification after fix

Do not proceed to Phase 7 (ongoing operations) with any unresolved failures.

## Internet Validation Instruction

Before executing this phase, perform a web search for:
- "Docker container escape techniques 2025 2026"
- "Telegram bot security testing checklist"
- "Linux server hardening verification"
- "UFW verification commands"
- "Docker userns-remap verification"

Verify that:
1. No new container escape techniques exist that bypass our protections
2. The security tests cover current known attack vectors
3. The network verification commands are correct for the current UFW version
4. No additional tests should be added based on recent security advisories
5. The verification approach is comprehensive for the threat model
