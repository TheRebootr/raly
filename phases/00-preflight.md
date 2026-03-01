# Phase 0: Pre-Flight Checks

## Context

RALY is a hardened Mac Mini 2018 (i5 6-core, 32GB, Omarchy 3.x Arch Linux) that
will run a custom AI harness inside a long-lived Docker container (Debian
Bookworm), reachable only via Telegram. This phase verifies the machine is in the
expected state before any hardening begins.

## Prerequisites

- Mac Mini has Omarchy 3.x freshly installed and booted
- WiFi working (firmware extracted pre-install)
- Ethernet available as fallback
- Physical or SSH access to the machine

## Steps

### 0.1 Verify Omarchy Version up to date

Check and apply for update in launcher.

### 0.2 Verify Docker

```bash
docker info
docker compose version
```

Expected: Docker daemon running, Docker Compose v2+ available.
If Docker is not running: `sudo systemctl start docker && sudo systemctl enable docker`

### 0.3 Verify Internet Connectivity

```bash
ping -c 3 1.1.1.1
ping -c 3 google.com
```

Both should succeed. If DNS fails but IP works, DNS config issue.

### 0.4 Verify WiFi Stability

```bash
iwctl station wlan0 show
# Check: Connected, signal strength, SSID
```

If WiFi is unstable, strongly prefer Ethernet for server use. Note the interface name.

### 0.5 Check Disk Space

```bash
df -h
```

### 0.6 Verify Docker Group Membership

```bash
groups
```

Output must include `docker`. If not:

```bash
sudo usermod -aG docker $USER
# Then log out and back in
```

### 0.7 Check Current Firewall State

```bash
sudo ufw status verbose
```

Record current rules. Note any existing rules (Omarchy may have set up LocalSend on 53317).
This is the baseline for Phase 1 hardening.

### 0.8 Note Network IPs

```bash
ip addr show
hostname -I
```

Record: LAN IP, interface names. You'll compare against Tailscale IP in Phase 3.

### 0.9 Verify LUKS Encryption

```bash
lsblk -f
```

Look for `crypto_LUKS` in the FSTYPE column. If not encrypted, this is a critical gap
to address before proceeding (data at rest is unprotected).

### 0.10 Check Available RAM and CPU

```bash
free -h
lscpu | grep -E "^CPU\(s\)|^Model name"
```

Verify: 32GB RAM, 6-core i5. These numbers inform Boot container resource limits in Phase 2.

## Verification Checklist

- [x] Omarchy up to date
- [x] Docker daemon running, `docker info` succeeds
- [x] Docker Compose v2+ available
- [x] Internet connectivity confirmed (both IP and DNS)
- [x] WiFi connected with acceptable signal (or Ethernet active)
- [x] At least 20GB free disk space
- [x] Current user is in `docker` group
- [x] Current UFW rules documented
- [x] LAN IP and interface names recorded
- [x] LUKS encryption confirmed on root partition
- [x] RAM and CPU specs verified (32GB, 6-core)

## Outputs for Downstream Phases

- Omarchy version number → Phase 1 (update baseline)
- Current UFW rules → Phase 1 (firewall hardening)
- LAN IP → Phase 3 (Tailscale comparison)
- Disk space available → Phase 2 (Boot container image + workspace sizing)
- Docker group confirmed → Phase 2 (Boot container management)

## Failure Conditions

If any of the following, STOP and resolve before proceeding:

1. Docker not installed or not running
2. No internet connectivity
3. Less than 10GB free disk space
4. Root partition not LUKS encrypted (discuss risk acceptance)

## Internet Validation Instruction

Before executing this phase, perform a web search for:

- "Omarchy 3.x Arch Linux post-install checklist 2025 2026"
- "Arch Linux server pre-flight checks"
- "Mac Mini 2018 Linux compatibility issues"

Verify that no critical known issues exist with this hardware/OS combination that would
affect the plan. If new information contradicts any step above, note the discrepancy
and adjust accordingly before proceeding.
