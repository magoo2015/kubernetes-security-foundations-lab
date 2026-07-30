# Evidence — Phase 0 (sanitized)

Sensitive values replaced with placeholders. No private keys, tokens, or full public IPs.

## Platform

| Item | Value |
|------|--------|
| OS | Ubuntu 24.04.4 LTS |
| Kernel | 6.8.0-136-generic |
| CPU | 2 |
| Memory | ~3.8 GiB |
| Disk (`/`) | ~77G total, low single-digit % used at capture |
| Hostname | `<HOSTNAME>` |
| Droplet public IP | `<DROPLET_PUBLIC_IP>` |
| Admin user | `<USERNAME>` (`sysadmin`) |

## Administrative access

- User: `sysadmin` in group `sudo`
- `whoami` → `sysadmin`
- `sudo whoami` → `root`
- `~/.ssh` → `700` owned by `sysadmin`
- `authorized_keys` → `600` (contents not recorded)
- Private key → `600` (contents not recorded)
- SSH client config → `600`

## SSH

- Service: active (socket-activated via `ssh.socket`)
- Syntax: `sshd -t` → OK
- Effective (`sshd -T` after reload), operator-reconfirmed:
  - port 22
  - permitrootlogin no
  - passwordauthentication no
  - pubkeyauthentication yes
  - kbdinteractiveauthentication no
  - permitemptypasswords no
  - x11forwarding no
  - maxauthtries 3
  - logingracetime 30
- No further SSH value changes or reload planned for Phase 0
- Drop-in: `/etc/ssh/sshd_config.d/99-kubernetes-lab-hardening.conf`
- Backup path (host-local): `/home/sysadmin/backups/phase-00-ssh-<TIMESTAMP>/`
- Sudo: temporary NOPASSWD removed; password prompt restored

## UFW

- Status: active
- Default: deny (incoming), allow (outgoing), disabled (routed)
- Rules: OpenSSH (22/tcp) ALLOW IN Anywhere (+ IPv6)
- Kubernetes/app ports: none opened
- Trusted admin IP at cloud layer: `<TRUSTED_ADMIN_IP>` (confirm in DigitalOcean UI)

## Fail2ban

- Service: enabled + active
- Jails: `sshd`
- Currently banned: 0
- Override: `/etc/fail2ban/jail.d/sshd.local` (maxretry 5, findtime 10m, bantime 1h)

## Time synchronization

- Time zone: Etc/UTC
- System clock synchronized: yes
- NTP service: active

## Automatic security updates

- `unattended-upgrades`: enabled + active
- `APT::Periodic::Update-Package-Lists "1"`
- `APT::Periodic::Unattended-Upgrade "1"`
- Automatic reboot: not enabled

## Listening ports (summary)

| Scope | Ports / services |
|-------|------------------|
| Public | TCP 22 (`sshd`) |
| Localhost | systemd-resolved DNS; local Cursor/node helpers on `127.0.0.1` |
| Absent | 6443, 80, 443, NodePort range |

## Package updates

- `apt-get update` + `upgrade`: 0 upgraded, 0 newly installed
- Kernel update in this run: no
- `/var/run/reboot-required`: absent

## GitHub SSH

- `ssh -T git@github.com` → authentication succeeded (account identity not required in evidence beyond success)

## Cluster software

- `k3s`: absent
- `kubectl`: absent

## Final Git status (captured after docs were written; no commit created)

Phase 0 - Host Hardening ✅
Phase 1 - K3s Installation & Kubernetes Fundamentals
Phase 2 - RBAC
Phase 3 - Pod Security
Phase 4 - Network Policies
Phase 5 - Secrets & ConfigMaps
Phase 6 - Audit Logging
Phase 7 - Falco Runtime Security

```text
On branch main
Your branch is up to date with 'origin/main'.

Untracked files:
  .gitignore
  PROJECT_CONTEXT.md
  README.md
  docs/
  evidence/

Tracked files (git ls-files):
  LICENSE
```
