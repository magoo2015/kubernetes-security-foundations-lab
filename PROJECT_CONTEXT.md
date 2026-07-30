# Project context

## Goals

- Learn Kubernetes by applying security engineering practice on a disposable lab host.
- Establish a hardened Ubuntu baseline before introducing cluster software.
- Document decisions, evidence, and limitations so later phases can revisit controls safely.

## Scope constraints

**In scope for Phase 0:** host inspection, admin account validation, OS updates, SSH hardening, UFW, Fail2ban, time synchronization, unattended security updates, listening-port review, sanitized documentation.

**Out of scope until later phases:** K3s, Kubernetes, kubectl, Docker/containerd (manual), Helm, Terraform, Ansible, Prometheus, Grafana, Falco, Kubernetes audit logging, manifests, and application workloads.

## Current VPS architecture

- Provider: DigitalOcean droplet (temporary lab VPS)
- Role: future single-node K3s host (not installed yet)
- OS: Ubuntu 24.04 LTS
- Administrative user: `sysadmin` (member of `sudo`)
- Access: SSH key authentication to `sysadmin` on port 22
- Hostname (lab): see evidence placeholders (`<HOSTNAME>`)

## Current phase

**Phase 0 — Host hardening: complete**

Phase 1 (K3s) is **not started**.

## SSH access model

- Drop-in: `/etc/ssh/sshd_config.d/99-kubernetes-lab-hardening.conf`
- Effective posture (validated with `sshd -T` after reload):
  - `PermitRootLogin no`
  - `PasswordAuthentication no`
  - `KbdInteractiveAuthentication no`
  - `PubkeyAuthentication yes`
  - `PermitEmptyPasswords no`
  - `X11Forwarding no`
  - `MaxAuthTries 3`
  - `LoginGraceTime 30`
  - Port remains **22** (unchanged)
- TCP forwarding was **not** globally disabled (Cursor Remote SSH / controlled tunnels may need it).
- Cloud-init / image drop-ins also reinforce `PasswordAuthentication no`.

## Firewall posture

- **UFW:** active
  - Default incoming: deny
  - Default outgoing: allow
  - Allowed: OpenSSH (TCP 22) from Anywhere (IPv4/IPv6)
  - Not opened: 6443, 80, 443, NodePort range, Flannel, metrics, databases
- **Trusted source IP:** not bound in UFW because the admin source address may change; limitation documented.
- **DigitalOcean Cloud Firewall:** should independently restrict SSH to `<TRUSTED_ADMIN_IP>` for defense in depth. Confirm in the DigitalOcean UI; not managed from this repository.

## Fail2ban status

- Installed, enabled, active
- Jail `sshd` enabled via `/etc/fail2ban/jail.d/sshd.local`
- Modest lab settings: `maxretry=5`, `findtime=10m`, `bantime=1h`, `backend=systemd`, `port=ssh`
- Packaged `jail.conf` was not edited

## Time synchronization

- Timezone: `Etc/UTC`
- `timedatectl`: system clock synchronized; NTP service active
- No second competing time daemon installed

## Automatic update status

- `unattended-upgrades` installed, enabled, and active
- Periodic: update package lists + unattended upgrades enabled (`20auto-upgrades`)
- Allowed origins include release and security (plus ESM security origins when applicable)
- Automatic reboot: **not enabled** (default remains disabled; no auto-reboot configured)

## Key host-hardening decisions

1. Harden SSH via a named drop-in rather than rewriting the full `sshd_config`.
2. Disable root and password SSH only after confirming `sysadmin` key auth and sudo.
3. Keep SSH on port 22; rely on keys + Fail2ban + dual firewalls.
4. Enable UFW with SSH allow rule present first; avoid Kubernetes ports in Phase 0.
5. Prefer Fail2ban local overrides over editing distro `jail.conf`.
6. Keep UTC for consistent server logs.
7. Store system config backups under `/home/sysadmin/backups/` (outside Git).

## Known limitations

- Single host; no HA, no separate management plane.
- UFW allows SSH from Anywhere at the host layer (source IP may not be stable).
- Droplet agent and Cursor/local node listeners exist; reviewed but not removed.
- Passwordless sudo was temporarily enabled for Phase 0 automation and then removed; passworded sudo is restored.
- Not production hardened; no IDS beyond Fail2ban SSH jail, no full CIS benchmark run.
- UFW vs CNI/kube-proxy interaction must be revisited after K3s.

## Reboot requirement

- After Phase 0 package refresh: **no reboot required** (`/var/run/reboot-required` absent).
- Kernel in use: current running kernel; no reboot forced by this phase.

## Next phase

**Phase 1 — not started.** Explicitly: **K3s is not installed.**
