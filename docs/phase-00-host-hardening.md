# Phase 0 — Host hardening

## Why host hardening precedes Kubernetes

Cluster software expands the attack surface (API server, kubelet, CNI, workloads). Hardening the Ubuntu host first reduces the chance that a pre-cluster compromise (weak SSH, open ports, missing patches) undermines everything that follows. Real-world security engineering applies the same order: identity and remote access, patching, host firewall, brute-force controls, reliable time, then platform install.

## Initial state (summary)

- Ubuntu 24.04 LTS on a DigitalOcean VPS
- Admin user `sysadmin` with sudo and working SSH keys
- SSH already partially locked down in main config / cloud-init (`PermitRootLogin no`, `PasswordAuthentication no`)
- UFW installed; later confirmed active with SSH allow
- Fail2ban installed and running with a basic `sshd` jail
- Time synchronized via systemd-timesyncd (NTP active, UTC)
- No K3s / kubectl / Docker present

## Administrative account validation

- Current user: `sysadmin`
- Groups include `sudo`; `sudo whoami` → `root`
- Home: `/home/sysadmin`
- `~/.ssh` mode `700`; `authorized_keys`, private key, and SSH `config` mode `600`
- GitHub SSH authentication succeeded for the configured identity
- Private key material was never displayed or committed

## SSH hardening decisions

- Created drop-in `/etc/ssh/sshd_config.d/99-kubernetes-lab-hardening.conf` instead of rewriting the entire default file.
- Backed up prior config to `/home/sysadmin/backups/phase-00-ssh-<TIMESTAMP>/` (outside Git).
- Validated with `sshd -t` before reload; inspected effective values with `sshd -T`.
- Second SSH session tested before `systemctl reload ssh`.
- Did **not** change the SSH port.
- Did **not** disable TCP forwarding globally.
- Did **not** add obsolete cipher/KEX overrides; Ubuntu 24.04 OpenSSH defaults retained.

### Controls mapped to practice

| Control | Why |
|---------|-----|
| `PermitRootLogin no` | Prevents direct root remote access; force named admin + sudo |
| `PasswordAuthentication no` | Reduces credential stuffing / password guessing surface |
| `PubkeyAuthentication yes` | Keeps key-based admin path |
| `MaxAuthTries 3` / `LoginGraceTime 30` | Limits unauthenticated connection abuse |
| Config validate then reload | Avoid lockout; prefer reload over restart |

## Effective SSH settings (post-reload)

From `sudo sshd -T` (agent validation after reload, then independently reconfirmed by the operator):

- `port 22`
- `permitrootlogin no`
- `passwordauthentication no`
- `kbdinteractiveauthentication no`
- `pubkeyauthentication yes`
- `permitemptypasswords no`
- `x11forwarding no`
- `maxauthtries 3`
- `logingracetime 30`

These values are considered final for Phase 0. No further SSH setting changes or reload are planned unless a regression appears.

## UFW configuration

- Default incoming: **deny**
- Default outgoing: **allow**
- Allow: OpenSSH profile (TCP 22) for IPv4 and IPv6
- Explicitly **not** opened: TCP 6443, 80, 443, NodePort 30000–32767, Flannel, metrics, DB ports

UFW and Kubernetes networking can interact in complex ways (CNI, kube-proxy, forwarded traffic). Revisit rules after K3s; do not add broad forwarding or cluster ports during Phase 0.

## DigitalOcean Cloud Firewall assumptions

Host UFW alone is not enough defense in depth. The DigitalOcean Cloud Firewall should restrict SSH to `<TRUSTED_ADMIN_IP>` when that address is stable. Phase 0 left UFW SSH open to Anywhere because the admin source IP may change (e.g., residential/dynamic). Confirm cloud firewall rules in the provider UI; they are not stored as secrets or IPs in this repo.

## Fail2ban configuration

Local override: `/etc/fail2ban/jail.d/sshd.local`

```ini
[sshd]
enabled = true
port = ssh
backend = systemd
maxretry = 5
findtime = 10m
bantime = 1h
```

Provides temporary bans after repeated SSH failures without aggressive lab brute-force testing. Distro `jail.conf` left untouched.

## Package update status

- `apt-get update` + `apt-get upgrade`: **0 packages upgraded** at Phase 0 execution time
- No kernel update applied in this run
- `/var/run/reboot-required` **absent** — reboot not required for Phase 0

## Time synchronization

- Timezone: UTC (preferred for server logging)
- Clock synchronized; NTP service active
- No competing NTP daemon installed

## Automatic security updates

- `unattended-upgrades` present, enabled, active
- Origins: distro release + security (+ ESM security origins when available)
- Automatic reboot: **disabled** / not configured

## Listening-port review

Public listeners observed:

- TCP 22 — `sshd` / `ssh.socket` (expected)

Localhost-only:

- systemd-resolved DNS (53 on loopback)
- Cursor/node local ports on `127.0.0.1` (IDE remote helpers)

No unexpected public Kubernetes or application listeners. DigitalOcean `droplet-agent` is installed and active as a provider agent; not disabled without further need.

## Remaining risks

- SSH from Anywhere on the host firewall if cloud firewall is misconfigured
- Single-node blast radius once K3s is added
- Fail2ban only covers SSH jail in Phase 0
- Provider metadata and droplet agent expand the software surface slightly

## Sudo access note

Temporary passwordless sudo was enabled so the remote development agent could complete Phase 0 without a TTY password prompt. That rule was removed after validation; sudo again requires the `sysadmin` password (confirmed by interactive `sudo` prompts).

## Revisit after K3s installation

- UFW rules vs API server (6443), kubelet, NodePorts, CNI
- Whether host firewall should allow only trusted admin to API
- Audit logging, Falco/runtime detection, and metrics ports
- Whether TCP forwarding and Fail2ban settings still fit operator workflows
