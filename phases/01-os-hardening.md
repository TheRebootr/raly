# Phase 1: OS-Level Hardening

## Context

RALY is a hardened Mac Mini 2018 running Omarchy 3.x (Arch Linux). This phase locks
down the OS before any services are exposed. The machine will ultimately run a custom
Python "Boot" Telegram bot inside a long-lived Docker container (Debian Bookworm), with
Claude Code CLI running directly inside that container.

The Mac Mini doubles as an occasional desktop (monitor + keyboard, Chromium, printing,
LocalSend file transfers). Hardening must not break desktop functionality.

Remote access is via Tailscale SSH (`tailscale up --ssh`), which handles authentication
through Tailscale's control plane — no openssh sshd needed, no SSH keys to manage, no
passwords. Physical access (monitor + keyboard) is available as fallback.

Omarchy v3.3.3 already provides: UFW (installed + enabled), ufw-docker (installed +
after.rules configured), Docker (installed + daemon.json configured), PAM faillock
(deny=10, unlock_time=120), LUKS full-disk encryption, and Tailscale.

This phase adds hardening on top of that baseline. All configs use separate files so
`omarchy-update` won't overwrite them.

## Prerequisites

- Phase 0 complete: Docker running, internet working, disk space confirmed, LUKS verified
- Current UFW rules documented from Phase 0
- Tailscale SSH working (`tailscale up --ssh`)

## Steps

### 1.1 System Updates

Use Omarchy's update system, not raw pacman. This runs pacman + AUR updates + Omarchy
migrations in the correct sequence:

```bash
# Check Arch news for breaking changes first
# Visit: https://archlinux.org/news/
omarchy-update
```

### 1.2 Firewall (UFW)

UFW and ufw-docker are already installed by Omarchy. This step verifies and locks down
the default policies.

#### 1.2.1 Audit current rules

```bash
sudo ufw status numbered
```

Review the output. LocalSend (port 53317) is an Omarchy default for LAN file
transfers — keep it if you use it.

#### 1.2.2 Set default policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

#### 1.2.3 Enable UFW

```bash
sudo ufw enable
sudo ufw status verbose
```

At this point: all inbound is denied (except existing allow rules like LocalSend),
all outbound is allowed.

By default, Omarchy has this configured. Double check first.

#### 1.2.4 Verify ufw-docker is in place

Omarchy installs the `ufw-docker` package and configures `/etc/ufw/after.rules`
automatically. Do NOT manually edit `after.rules` — the package manages it with
proper rules (private subnet returns, logging deny chain, DNS passthrough).

### 1.3 Ensure openssh sshd is disabled

Tailscale SSH handles all remote access. openssh sshd should NOT be running — it
would be an unnecessary attack surface.

```bash
# Verify sshd is not running (Omarchy default: disabled)
# sudo systemctl disable sshd 2>/dev/null
# sudo systemctl stop sshd 2>/dev/null
systemctl is-enabled sshd
# Should report: disabled (or not-found)
```

### 1.4 Disk Encryption Verification

```bash
lsblk -f
# Confirm crypto_LUKS on root partition
# If not encrypted, this is a Phase 0 failure that should have been caught
```

Also verify swap is encrypted or disabled:

```bash
swapon --show
# If swap exists, verify it's on an encrypted partition
```

### 1.5 Kernel Security Parameters

Note: This is a generic Linux Hardening by Claude. I verify it first on my machine, by default Omarchy has everything configured perfectly so I skipped this

Create `/etc/sysctl.d/99-raly.conf` (Omarchy owns `99-sysctl.conf` — use a
separate file):

```ini
# --- Kernel hardening ---
kernel.dmesg_restrict = 1
kernel.kptr_restrict = 2
kernel.yama.ptrace_scope = 2
kernel.unprivileged_bpf_disabled = 1
kernel.perf_event_paranoid = 3
kernel.kexec_load_disabled = 1

# --- BPF JIT hardening ---
net.core.bpf_jit_harden = 2

# --- Filesystem hardening ---
fs.protected_hardlinks = 1
fs.protected_symlinks = 1
fs.protected_fifos = 2
fs.protected_regular = 2
fs.suid_dumpable = 0

# --- IPv4 hardening ---
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_syncookies = 1
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# net.ipv4.ip_forward is intentionally NOT set here.
# Docker requires ip_forward=1 and manages it via the daemon.

# --- IPv6 hardening (keep IPv6 ENABLED for Tailscale) ---
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_ra = 0
net.ipv6.conf.default.accept_ra = 0

# NOT included (intentional):
# kernel.unprivileged_userns_clone = 0  -- breaks Chromium sandbox
# kernel.sysrq = 0                     -- needed for keyboard recovery
```

Apply:

```bash
sudo sysctl --system
```

Verify key settings:

```bash
sysctl kernel.dmesg_restrict kernel.kptr_restrict kernel.yama.ptrace_scope \
  kernel.unprivileged_bpf_disabled net.ipv4.conf.all.rp_filter
```

### 1.6 Core Dump Restrictions

Note: I did not do this recommendation. I figure it might mess up my Omarchy updates/system.

Core dumps can leak secrets (Telegram bot token, API keys) from process memory.

Add to `/etc/security/limits.conf`:

```
* hard core 0
* soft core 0
```

Create `/etc/systemd/coredump.conf.d/disable.conf`:

```ini
[Coredump]
Storage=none
ProcessSizeMax=0
```

### 1.7 Kernel Module Blacklisting

Note: I skip this as well as I use my spare machine as a Desktop sometimes.

Blacklist modules that are unnecessary to reduce kernel attack surface.

Create `/etc/modprobe.d/raly-blacklist.conf`:

```
# Uncommon filesystems (reduce kernel attack surface)
blacklist cramfs
blacklist hfs
blacklist hfsplus

# Uncommon network protocols
blacklist dccp
blacklist sctp
blacklist rds
blacklist tipc
```

## Verification Checklist

- [x] System updated via `omarchy-update`
- [x] UFW enabled, default deny incoming, allow outgoing
- [x] `ufw-docker status` confirms Docker bypass prevention in place
- [x] openssh sshd disabled / not running
- [x] Tailscale SSH working as only remote access method
- [x] LUKS encryption confirmed on root partition
- [x] Kernel security parameters applied and verified (`sysctl --system`)
      Omarchy has these by default
- [x] Core dumps disabled
      Did not do
- [x] Kernel modules blacklisted
      Did not apply anything
- [x] Reboot test: all services come back up after `sudo reboot`
- [x] Post-reboot: Tailscale SSH still works, Chromium still works

## Outputs for Downstream Phases

- UFW deny-all baseline → Phase 3 (Tailscale ACL configuration)
- Sysctl hardening → Phase 6 (verification testing)
- OS hardened → Phase 2 (Boot container setup can proceed on a secure host)

## Rollback

If Tailscale SSH stops working:

1. Physical access to Mac Mini (monitor + keyboard)
2. Login at physical console
3. Check Tailscale: `tailscale status`
4. Restart if needed: `sudo systemctl restart tailscaled`

To remove sysctl hardening:

```bash
sudo rm /etc/sysctl.d/99-raly.conf
sudo sysctl --system
```
