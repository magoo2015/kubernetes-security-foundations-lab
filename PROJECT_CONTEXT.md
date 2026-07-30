# Project context

## Goals

- Learn Kubernetes by applying security engineering practice on a disposable lab host.
- Establish a hardened Ubuntu baseline before introducing cluster software.
- Document decisions, evidence, and limitations so later phases can revisit controls safely.
- Build intuition for desired state, reconciliation, and core workload objects before adding RBAC, PSA, and NetworkPolicies.

## Scope constraints

**Completed — Phase 0:** host inspection, admin account validation, OS updates, SSH hardening, UFW, Fail2ban, time synchronization, unattended security updates, listening-port review, sanitized documentation.

**Completed — Phase 1:** official K3s install, cluster health checks, declarative nginx Deployment/Service in `lab-app`, scale / self-heal / rollout / rollback exercises, troubleshooting commands, sanitized evidence.

**Out of scope until later phases:** RBAC hardening, NetworkPolicies, Pod Security Admission, Falco, Helm, Terraform, Prometheus, Grafana, GitOps, ArgoCD, Ingress, Load Balancers, multi-node clusters, advanced Secrets management, Kubernetes audit logging configuration.

## Current VPS architecture

- Provider: DigitalOcean droplet (temporary lab VPS)
- Role: **single-node K3s server** (control-plane + worker)
- OS: Ubuntu 24.04 LTS
- Administrative user: `sysadmin` (member of `sudo`)
- Access: SSH key authentication to `sysadmin` on port 22
- Cluster: K3s **v1.36.2+k3s1**, containerd runtime
- Workload: `lab-app` / Deployment `nginx` / Service `nginx` (ClusterIP)
- Hostname (lab): see evidence placeholders (`<HOSTNAME>`)

## Current phase

**Phase 1 — Kubernetes foundations: complete**

Next: Phase 2 (RBAC / least-privilege) when scheduled.

## Installed components (Phase 1)

| Component | Detail |
|-----------|--------|
| K3s | `v1.36.2+k3s1` via official `get.k3s.io` stable channel |
| kubectl / crictl / ctr | Symlinked to `k3s` |
| containerd | Bundled with K3s (`containerd://2.3.2-k3s2`) |
| CoreDNS / metrics-server | K3s defaults (observed via `kubectl cluster-info`) |
| kubeconfig | Host path `/etc/rancher/k3s/k3s.yaml`; user copy `~/.kube/config` (**not in Git**) |
| Manifests | `manifests/lab-app/` (Namespace, Deployment, Service) |

## SSH access model

- Drop-in: `/etc/ssh/sshd_config.d/99-kubernetes-lab-hardening.conf`
- Effective posture (validated with `sshd -T`):
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

## Firewall posture

- **UFW:** active
  - Default incoming: deny
  - Default outgoing: allow
  - Allowed: OpenSSH (TCP 22) from Anywhere (IPv4/IPv6)
  - **Not opened:** 6443 (API), 80, 443, NodePort range
- K3s may listen on `*:6443` locally; **UFW blocks public access** to the API.
- After K3s install, UFW routed forwarding may show as **deny** (host firewall still denies unexpected inbound).
- **DigitalOcean Cloud Firewall:** should independently restrict SSH to `<TRUSTED_ADMIN_IP>` for defense in depth. Confirm in the DigitalOcean UI; not managed from this repository.

## Fail2ban status

- Installed, enabled, active
- Jail `sshd` enabled via `/etc/fail2ban/jail.d/sshd.local`
- Modest lab settings: `maxretry=5`, `findtime=10m`, `bantime=1h`, `backend=systemd`, `port=ssh`

## Time synchronization

- Timezone: `Etc/UTC`
- `timedatectl`: system clock synchronized; NTP service active

## Automatic update status

- `unattended-upgrades` installed, enabled, and active
- Automatic reboot: **not enabled**

## Key decisions (Phase 0 + 1)

1. Harden SSH via a named drop-in rather than rewriting the full `sshd_config`.
2. Keep SSH on port 22; rely on keys + Fail2ban + dual firewalls.
3. Enable UFW with SSH allow first; **do not open 6443** for Phase 1.
4. Use official K3s install; accept single-node SQLite datastore for lab simplicity.
5. Prefer ClusterIP Services (no public app exposure) for foundations demos.
6. Keep declarative manifests in Git; use imperative kubectl only for teaching exercises.
7. Store system config backups under `/home/sysadmin/backups/` (outside Git).

## Known limitations

- Single host; no HA, no separate management plane.
- UFW allows SSH from Anywhere at the host layer (source IP may not be stable).
- Default K3s kubeconfig is cluster-admin; RBAC least privilege not yet applied.
- Default allow-all Pod networking; no NetworkPolicies yet.
- No Pod Security Admission hardening yet.
- No audit logging / runtime detection (Falco) yet.
- Passwordless sudo may be temporarily enabled for agent automation; must be removed after the phase.
- Not production hardened.

## Remaining roadmap

| Phase | Intent |
|-------|--------|
| **2** | RBAC, least-privilege identities, avoid standing cluster-admin for daily use |
| **3** | NetworkPolicies / workload isolation |
| **4** | Audit logging, metrics/detection orientation |
| **5** | Threat scenarios and remediation practice |

Related deeper topics (PSA, Secrets patterns, Falco, GitOps) land in later phases as scoped.

## Reboot requirement

- Phase 0/1: **no reboot required** for documented work (`/var/run/reboot-required` absent at Phase 0; K3s install did not require reboot).

## Next phase

**Phase 2 — not started.** Explicit non-goals from Phase 1 remain deferred: RBAC, NetworkPolicies, Pod Security Admission, Falco, Helm, Ingress, multi-node.
