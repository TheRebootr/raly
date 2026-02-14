# Phase 7: Ongoing Operations

## Context

RALY is now fully operational: a hardened Mac Mini running Omarchy 3.x with a custom
"Boot" Telegram harness executing Claude Code CLI tasks in ephemeral Docker containers.
All remote access via Tailscale SSH. This phase is a reference document for maintaining
the RALY security posture over time. It is not a build phase — it is the operational
runbook.

## Prerequisites

- Phase 6 complete: all verification tests pass
- System is in production operation

## 7.1 Maintenance Schedule

### Weekly

| Task | Command | What to Check |
|------|---------|---------------|
| System updates | `sudo pacman -Syu` | Check https://archlinux.org/news/ FIRST for breaking changes |
| Review audit logs | `sqlite3 ~/boot-data/boot.db "SELECT * FROM audit_log WHERE action='auth_rejected' AND timestamp > datetime('now', '-7 days');"` | Any unauthorized access attempts |
| Review health check logs | `cat ~/boot-data/logs/health.log \| tail -20` | Any health check failures |
| Check Tailscale status | `tailscale status` | All devices expected, no unknown devices |
| Check service status | `systemctl status boot.service` | Active, no restart loops |
| Check disk space | `df -h` | Under 80% usage |

### Monthly

| Task | Command | What to Check |
|------|---------|---------------|
| Dependency audit | `cd ~/boot-src && source venv/bin/activate && pip audit` | No known CVEs in 3 dependencies |
| Rebuild sandbox image | See "Image Rebuild" section below | Pick up base image security patches |
| Review Tailscale devices | https://login.tailscale.com/admin/machines | Revoke any unexpected devices |
| Rotate audit logs | Automatic via boot-log-rotate.timer | Verify old logs compressed, ancient logs deleted |
| Check fail2ban | `sudo fail2ban-client status sshd` | Review ban history |
| Update dependencies | `pip install --upgrade -r requirements.txt` | Only if pip audit found issues |

### Quarterly

| Task | Command | What to Check |
|------|---------|---------------|
| Full verification re-run | Execute Phase 6 test suite | All tests still pass |
| Review sysctl settings | `sysctl -a \| grep -E "dmesg_restrict\|kptr_restrict\|rp_filter"` | All hardening parameters still set |
| Review Docker config | `cat /etc/docker/daemon.json` | All hardening settings intact |
| Review UFW rules | `sudo ufw status verbose` | Only Tailscale SSH allowed |

## 7.2 Update Procedures

### System Update (Arch Linux)

```bash
# Step 1: Check Arch news for breaking changes
# Visit: https://archlinux.org/news/
# Look for manual intervention required

# Step 2: Update
sudo pacman -Syu

# Step 3: If kernel updated, reboot
sudo reboot

# Step 4: Verify services after reboot
systemctl status boot.service
tailscale status
docker info
```

### Boot Harness Update

You own this code. Updates are your commits.

```bash
cd ~/boot-src

# Make changes
# ...

# Test locally
source venv/bin/activate
python -m pytest tests/ -v

# Restart service to pick up changes
sudo systemctl restart boot.service
sudo systemctl status boot.service

# Verify via Telegram
# Send a message, confirm response

# Commit
git add .
git commit -m "description of change"
```

### Dependency Update

```bash
cd ~/boot-src
source venv/bin/activate

# Check for CVEs
pip audit

# If CVEs found, update specific package
pip install --upgrade python-telegram-bot==<new-version>

# Update requirements.txt with new pinned version
pip freeze | grep -E "python-telegram-bot|aiosqlite" > requirements.txt

# Test
python -m pytest tests/ -v

# Restart service
sudo systemctl restart boot.service
```

### Sandbox Image Rebuild

```bash
cd ~/boot-src

# Pull latest base
docker pull python:3.12-slim

# Record new digest
docker inspect python:3.12-slim --format '{{.RepoDigests}}'

# Update Dockerfile.sandbox with new digest (FROM line)

# Rebuild
docker build --no-cache -f Dockerfile.sandbox -t boot-sandbox:latest .

# Scan
trivy image boot-sandbox:latest

# Run verification tests from Phase 5.7
docker run --rm boot-sandbox:latest whoami
docker run --rm boot-sandbox:latest claude --version
# ... (run all image verification tests)

# Remove old images
docker image prune -f
```

### Omarchy Update

```bash
omarchy-update

# After update, verify Docker config wasn't reset:
cat /etc/docker/daemon.json
# Verify userns-remap, icc, etc. still present

# Verify UFW rules unchanged:
sudo ufw status verbose

# Verify sysctl settings unchanged:
sysctl kernel.dmesg_restrict kernel.kptr_restrict
```

## 7.3 Backup Procedures

### What to Backup

| Path | Contents | Priority | Method |
|------|----------|----------|--------|
| `~/boot-src/` | Your harness code | Critical | Git push to private repo |
| `~/boot-data/boot.db` | Sessions, audit log, conversation history | High | `cp` or `sqlite3 .backup` |
| `~/boot-data/config.env` | Secrets | Critical | Encrypted backup only |
| `~/boot-workspace/` | Active project files | Medium | Per-project git repos |

### Backup Commands

```bash
# Backup database (hot backup, safe while Boot is running)
sqlite3 ~/boot-data/boot.db ".backup '/tmp/boot-backup-$(date +%Y%m%d).db'"

# Push code to private repo
cd ~/boot-src
git push origin main

# Backup config (encrypt first)
gpg --symmetric --cipher-algo AES256 -o /tmp/config-backup.env.gpg ~/boot-data/config.env
```

### What NOT to Backup

- Docker images (rebuild from Dockerfile)
- Python venv (recreate from requirements.txt)
- System packages (reinstall from Omarchy)

## 7.4 Incident Response Playbook

### Suspected Compromise

**Severity: Critical. Time-sensitive. Follow steps IN ORDER.**

```bash
# 1. STOP Boot immediately
sudo systemctl stop boot.service

# 2. Kill all sandbox containers
docker kill $(docker ps -q --filter ancestor=boot-sandbox) 2>/dev/null

# 3. Disconnect from internet
sudo ip link set wlan0 down
# If using ethernet:
sudo ip link set enp0s31f6 down

# 4. Preserve evidence (before modifying anything)
cp ~/boot-data/boot.db ~/boot-data/boot-incident-$(date +%Y%m%d%H%M).db
cp -r ~/boot-data/logs ~/boot-data/logs-incident-$(date +%Y%m%d%H%M)

# 5. Review audit log
sqlite3 ~/boot-data/boot.db "
    SELECT timestamp, action, user_id, detail, result
    FROM audit_log
    ORDER BY timestamp DESC
    LIMIT 100;
"

# 6. Check for unexpected files
find ~/boot-workspace -name "*.sh" -o -name "*.py" -newer ~/boot-data/boot.db | head -20
ls -la ~/boot-workspace/*/

# 7. Check for unexpected processes
ps aux | grep -v "^\[" | grep -v "grep"

# 8. Check for unexpected network connections (while briefly online)
sudo ss -tlnp
sudo ss -tnp

# 9. Rotate ALL credentials
# - Telegram bot token: message @BotFather, /revoke, get new token
# - API keys: regenerate from provider dashboards
# - SSH keys: generate new key pair, replace authorized_keys
# - Tailscale: deauthorize Mac Mini, re-authorize with fresh key

# 10. Update config.env with new credentials
# 11. Review ~/boot-workspace/ for unauthorized modifications
# 12. Rebuild sandbox image from scratch
# 13. Restart services
# 14. Run full Phase 6 verification
```

### Boot Service Keeps Crashing

```bash
# Check service status
systemctl status boot.service

# Check recent logs
journalctl -u boot.service -n 100 --no-pager

# Check if it's a restart loop
journalctl -u boot.service | grep "Started\|Stopped\|Failed" | tail -20

# If config issue: fix config.env, restart
# If code issue: fix code, restart
# If dependency issue: recreate venv, reinstall deps, restart

# Manual test run (outside systemd, see stdout directly):
cd ~/boot-src
source venv/bin/activate
python -m boot.main
# Watch output for errors
```

### Database Corruption

```bash
# Check integrity
sqlite3 ~/boot-data/boot.db "PRAGMA integrity_check;"

# If corrupt:
# 1. Stop Boot
sudo systemctl stop boot.service

# 2. Attempt repair
sqlite3 ~/boot-data/boot.db ".recover" | sqlite3 ~/boot-data/boot-recovered.db

# 3. If recovery works:
mv ~/boot-data/boot.db ~/boot-data/boot-corrupt.db
mv ~/boot-data/boot-recovered.db ~/boot-data/boot.db

# 4. If recovery fails, start fresh (lose history):
rm ~/boot-data/boot.db
# Boot will recreate tables on next start

# 5. Restart
sudo systemctl start boot.service
```

### Disk Full

```bash
# Identify largest consumers
du -sh ~/* | sort -rh | head -10
du -sh ~/boot-data/logs/* | sort -rh
docker system df

# Clean up
# Docker artifacts:
docker system prune -f
docker image prune -a -f

# Old logs:
find ~/boot-data/logs -name "*.log.gz" -mtime +7 -delete

# Old audit entries:
sqlite3 ~/boot-data/boot.db "DELETE FROM audit_log WHERE timestamp < datetime('now', '-7 days');"
sqlite3 ~/boot-data/boot.db "VACUUM;"
```

## 7.5 Monitoring Summary

```
What           | How                              | Frequency
---------------|----------------------------------|----------
Boot running   | systemctl status boot.service    | health.timer (6h)
Disk space     | df -h                            | health.timer (6h)
Docker running | docker info                      | health.timer (6h)
DB integrity   | PRAGMA integrity_check           | health.timer (6h)
Auth attempts  | audit_log WHERE auth_rejected    | Weekly review
Rate limiting  | audit_log WHERE rate_limited     | Weekly review
CVEs           | pip audit                        | Monthly
Image vulns    | trivy image boot-sandbox         | Monthly rebuild
Tailscale      | tailscale status                 | Weekly
UFW            | ufw status verbose               | Quarterly
```

## Verification Checklist

This is a reference document. Verification is ongoing:

- [ ] Maintenance schedule followed (check logs for dates)
- [ ] No unresolved CVEs in dependencies
- [ ] Sandbox image rebuilt within last 30 days
- [ ] Audit log reviewed within last 7 days
- [ ] Backups current (code pushed, DB backed up)
- [ ] Tailscale device list clean
- [ ] Phase 6 verification re-run within last 90 days

## Internet Validation Instruction

Before relying on this document, perform a web search for:
- "Arch Linux rolling release maintenance best practices 2025 2026"
- "SQLite hot backup while in use"
- "Docker image security maintenance schedule"
- "incident response playbook server compromise"
- "Tailscale device revocation"

Verify that:
1. Arch Linux update procedures haven't changed
2. SQLite `.backup` command is safe during active writes (WAL mode)
3. The incident response steps are current and complete
4. Docker image rebuild frequency recommendation is adequate
5. No new operational security practices should be added
