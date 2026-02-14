# Phase 3: Tailscale ACL Configuration

## Context

RALY is a hardened Mac Mini 2018 running Omarchy 3.x (Arch Linux). Tailscale
provides the ONLY remote access path to the machine via Tailscale SSH (`tailscale up
--ssh`). This is NOT openssh sshd — Tailscale runs its own SSH server within tailscaled,
authenticating users through Tailscale's control plane instead of SSH keys.

No ports are exposed to the public internet or the local LAN. The Boot container
(Telegram bot) uses outbound polling — no inbound ports needed.

### How Tailscale SSH works (important distinctions)

- `tailscale up --ssh` tells tailscaled to accept SSH connections
- Authentication is via Tailscale identity, NOT SSH keys or passwords
- openssh sshd is NOT involved and should NOT be running
- Connections traverse the WireGuard tunnel — no port 22 exposed on LAN
- UFW does not need to allow port 22 — traffic arrives through the Tailscale tunnel
- You connect with `ssh user@<tailscale-ip>` and Tailscale handles auth

## Prerequisites

- Phase 1 complete: UFW deny-all inbound, openssh sshd confirmed disabled
- Omarchy already provides: Tailscale installed, `tailscaled.service` enabled
- Internet connectivity confirmed
- LAN IP noted from Phase 0

## Steps

### 3.1 Verify Tailscale is Running

Omarchy installs Tailscale and enables `tailscaled.service`. Verify it's active:

```bash
systemctl is-enabled tailscaled    # → enabled
systemctl is-active tailscaled     # → active
tailscale version
```

Also verify the TUN kernel module is loaded (tailscaled crash-loops without it):

```bash
lsmod | grep tun
# Should show "tun" — if missing: sudo modprobe tun
```

### 3.2 Authenticate and Enable SSH

```bash
# Authenticate to your Tailscale account + enable SSH server
sudo tailscale up --ssh
```

If not already authenticated, this prints a URL. Open it in a browser on your
MacBook/phone, log in, and authorize the Mac Mini.

### 3.3 Note Tailscale IP and Status

```bash
tailscale ip -4
# Should be 100.x.x.x

tailscale status
# Shows your device name, IP, and connected peers
```

Record the Tailscale IP. This is how you'll access the machine remotely.

### 3.4 Enable MagicDNS (optional but recommended)

In the Tailscale admin console (<https://login.tailscale.com/admin>):

- Enable MagicDNS
- Your Mac Mini will be reachable as `<hostname>.<tailnet-name>.ts.net`

### 3.5 Configure Tailscale ACLs

In the Tailscale admin console (<https://login.tailscale.com/admin/acls>):

Replace the default "allow all" policy. Tailscale has **two independent permission layers**
that both must pass for a connection to succeed:

1. **Network ACLs** (`acls`) — controls raw IP+port connectivity between devices
2. **SSH ACLs** (`ssh`) — controls who can Tailscale SSH into what, as which user

Even if the network ACL allows `*:*`, SSH is denied unless a matching `ssh` rule also exists.
An intruder who joins your tailnet would match neither `autogroup:owner` nor `autogroup:self`,
so they get **zero** access — no network connectivity and no SSH.

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["autogroup:owner"],
      "dst": ["*:*"]
    }
  ],
  "ssh": [
    {
      "action": "accept",
      "src": ["autogroup:owner"],
      "dst": ["autogroup:self"],
      "users": ["autogroup:nonroot"]
    }
  ]
}
```

What this achieves:

- **Network**: Only owner devices can reach any device — general connectivity preserved
  (file sharing, LocalSend, etc.)
- **SSH**: Only owner can SSH, only into their own devices, only as non-root (sudo after)
- **Intruder with tailnet access**: No ACL rules match them → blocked from everything
- **Blast radius**: Even a compromised device token cannot SSH without owner identity

Do NOT tag the Mac Mini (e.g., `tag:server`). Tags strip user identity from a device,
removing it from `autogroup:self` and breaking the SSH rule above. Tags are for multi-user
tailnets with role-based access — unnecessary for a single-owner tailnet.

### 3.6 Verify Tailscale Funnel is OFF

Tailscale Funnel exposes services to the public internet. It must be OFF.

```bash
tailscale funnel status
# Should show: Funnel off
# Or: No Funnel configuration
```

If Funnel is on:

```bash
tailscale funnel off
```

### 3.7 Verify UFW Does NOT Have SSH Rules

Since we use Tailscale SSH (not openssh), UFW should NOT have port 22 rules.
Tailscale SSH traffic flows through the WireGuard tunnel, not through regular TCP.

```bash
sudo ufw status numbered
```

If there are any rules allowing port 22 (SSH), remove them:

```bash
# Delete any SSH allow rules — they would only benefit openssh sshd (which is disabled)
# sudo ufw delete <rule_number>
```

The only allow rule should be LocalSend (53317) if you use it.
On Omarchy the docker-dns is also allowed

### 3.8 Verify openssh sshd is Still Disabled

Double-check from Phase 1 — openssh sshd should NOT be running alongside Tailscale SSH:

```bash
systemctl is-active sshd 2>/dev/null    # → inactive (or not found)
systemctl is-enabled sshd 2>/dev/null   # → disabled (or not found)
```

If sshd is running, disable it:

```bash
sudo systemctl stop sshd
sudo systemctl disable sshd
```

### 3.9 Test Access Paths

#### 3.9.1 SSH via Tailscale (MUST WORK)

From another device with Tailscale installed:

```bash
# Via Tailscale IP:
ssh user@100.x.x.x

# Or via MagicDNS (if enabled):
ssh user@<hostname>.<tailnet>.ts.net
```

No SSH key needed — Tailscale handles authentication. You may be prompted to
approve the connection in the Tailscale admin console on first use.

#### 3.9.2 SSH via LAN IP (MUST FAIL)

From another device on the same LAN (not through Tailscale):

```bash
ssh user@<lan-ip>
# Should timeout or connection refused — no sshd listening, UFW blocks it
```

#### 3.9.3 Port scan from LAN (should show nothing useful)

```bash
# From another LAN device:
nmap -Pn <lan-ip>
# All ports should be filtered/closed (except 53317 LocalSend if enabled)
```

### 3.10 Check for NetworkManager Conflict

If Omarchy uses NetworkManager, it may fight tailscaled over the `tailscale0` interface.
Check and fix if needed:

```bash
# Check if NetworkManager is managing tailscale0:
nmcli device status | grep tailscale
# If it shows "connected" or "managed", create a drop-in to exclude it:
```

If managed, create `/etc/NetworkManager/conf.d/99-tailscale.conf`:

```ini
[keyfile]
unmanaged-devices=interface-name:tailscale0
```

Then restart both services:

```bash
sudo systemctl restart NetworkManager
sudo systemctl restart tailscaled
```

Skip this step if Omarchy does not use NetworkManager or if `tailscale0` is already unmanaged.

### 3.11 Verify Tailscale Starts on Boot

```bash
systemctl is-enabled tailscaled
# Output: enabled (Omarchy sets this by default)
```

Test with a reboot:

```bash
sudo reboot
# After reboot, verify from another Tailscale device:
ssh user@<tailscale-ip>
```

## Verification Checklist

- [x] TUN kernel module loaded (`lsmod | grep tun`)
- [x] `tailscaled` service enabled and active (Omarchy default)
- [x] Mac Mini authenticated and appearing in Tailscale admin console
- [x] Tailscale SSH enabled (`tailscale up --ssh`)
- [x] Tailscale IP (100.x.x.x) recorded
- [x] MagicDNS enabled (optional)
- [x] Tailscale ACLs: network restricted to `autogroup:owner`
- [x] Tailscale ACLs: SSH restricted to `autogroup:owner` → `autogroup:self`, non-root only
- [x] Mac Mini is NOT tagged (no `--advertise-tags`)
- [x] Tailscale Funnel is OFF
- [x] No UFW rules for port 22 (not needed with Tailscale SSH)
- [x] openssh sshd confirmed disabled
- [x] NetworkManager not managing `tailscale0` (if applicable)
- [x] SSH via Tailscale IP: WORKS (Tailscale auth, no SSH keys)
- [x] SSH via LAN IP: FAILS (no sshd, UFW blocks)
- [x] Tailscale survives reboot

## Outputs for Downstream Phases

- Tailscale IP → Phase 6 (verification testing)
- Tailscale SSH as only remote access → Phase 7 (ops reference)
- Tailscale running → Phase 6 (persistence test after reboot)

## Rollback

If locked out via Tailscale:

1. Physical access to Mac Mini (monitor + keyboard)
2. Login at physical console
3. Check Tailscale: `tailscale status`
4. Restart if needed: `sudo systemctl restart tailscaled`
5. Re-authenticate if needed: `sudo tailscale up --ssh`

If Tailscale service fails:

1. Physical access
2. `sudo systemctl status tailscaled` to diagnose
3. `journalctl -u tailscaled -n 50` for logs

## Operational Notes

- **Package updates**: When `omarchy-update` updates the `tailscale` package, the running
  `tailscaled` keeps the old binary. It does NOT auto-restart. After an update, restart
  manually: `sudo systemctl restart tailscaled`. Consider adding this to Phase 7 (ops).
- **DNS interaction**: Tailscale expects `systemd-resolved` for MagicDNS. If Omarchy uses
  a different resolver, Tailscale may overwrite `/etc/resolv.conf`, which can break DNS
  for the Boot container (Docker routes DNS via 172.17.0.1). Verify with `resolvectl status`
  after enabling MagicDNS.

## Internet Validation Instruction

Before executing this phase, perform a web search for:

- "Tailscale SSH setup Arch Linux 2025 2026"
- "Tailscale ACL configuration best practices"
- "Tailscale Funnel disable verification"
- "Tailscale MagicDNS setup"

Verify that:

1. `tailscale up --ssh` is still the correct command for enabling Tailscale SSH
2. The ACL syntax hasn't changed in recent Tailscale versions
3. Tailscale SSH auth flow is as described (no SSH keys)
4. No known issues with Tailscale + Arch Linux
5. Tailscale Funnel commands are current
