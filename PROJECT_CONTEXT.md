# Project context

## Goals

- Learn Kubernetes by applying security engineering practice on a disposable lab host.
- Establish a hardened Ubuntu baseline before introducing cluster software.
- Document decisions, evidence, and limitations so later phases can revisit controls safely.
- Build intuition for desired state, reconciliation, and core workload objects before adding RBAC, PSA, and NetworkPolicies.
- Practice least-privilege RBAC with validation, threat modeling, and detection-oriented notes.
- Harden workloads with Pod Security Admission (Restricted), SecurityContext, and resource limits.

## Scope constraints

**Completed — Phase 0:** host inspection, admin account validation, OS updates, SSH hardening, UFW, Fail2ban, time synchronization, unattended security updates, listening-port review, sanitized documentation.

**Completed — Phase 1:** official K3s install, cluster health checks, declarative nginx Deployment/Service in `lab-app`, scale / self-heal / rollout / rollback exercises, troubleshooting commands, sanitized evidence.

**Completed — Phase 2:** authorization overview docs; ServiceAccount `security-reader`; Role/RoleBinding (get/list/watch pods, services, deployments in `lab-app`); `kubectl auth can-i` validation; temporary unsafe `cluster-admin` binding demo then removal; common mistakes, threat model, detection notes; interview notes; portfolio / chat context / changelog; sanitized evidence.

**Completed — Phase 3:** Pod Security Admission overview (PSP history → PSA); namespace labels enforce/warn/audit = **restricted**; hardened nginx (`nginxinc/nginx-unprivileged:1.27.4`) with SecurityContext (non-root, drop ALL, RO rootfs, RuntimeDefault seccomp, runAsUser/Group/fsGroup); CPU/memory requests+limits; `automountServiceAccountToken: false` (workload-identity control, not PSA); temporary privileged+hostPID+hostNetwork demo then delete; threat model; detection docs; production considerations (Kyverno/Gatekeeper/signing); interview notes; evidence.

**Out of scope until later phases:** NetworkPolicies, Secrets hardening, Falco, Helm, Terraform, Prometheus, Grafana, GitOps, ArgoCD, Ingress, Load Balancers, multi-node clusters, Kubernetes audit logging configuration.

## Current VPS architecture

- Provider: DigitalOcean droplet (temporary lab VPS)
- Role: **single-node K3s server** (control-plane + worker)
- OS: Ubuntu 24.04 LTS
- Administrative user: `sysadmin` (member of `sudo`)
- Access: SSH key authentication to `sysadmin` on port 22
- Cluster: K3s **v1.36.2+k3s1**, containerd runtime
- Workload: `lab-app` / Deployment `nginx` (`nginxinc/nginx-unprivileged:1.27.4`) / Service `nginx` (ClusterIP :80 → targetPort 8080)
- PSA: `lab-app` enforce/warn/audit **restricted**
- RBAC: `security-reader` SA + Role + RoleBinding (namespace read-only on pods/services/deployments)
- Hostname (lab): see evidence placeholders (`<HOSTNAME>`)

## Current phase

**Phase 3 — Pod Security Admission & workload hardening: complete**

Next: Phase 4 (NetworkPolicies / workload isolation) when scheduled.

## Installed components (Phase 1–3)

| Component | Detail |
|-----------|--------|
| K3s | `v1.36.2+k3s1` via official `get.k3s.io` stable channel |
| kubectl / crictl / ctr | Symlinked to `k3s` |
| containerd | Bundled with K3s (`containerd://2.3.2-k3s2`) |
| CoreDNS / metrics-server | K3s defaults (observed via `kubectl cluster-info`) |
| kubeconfig | Host path `/etc/rancher/k3s/k3s.yaml`; user copy `~/.kube/config` (**not in Git**) |
| Workload manifests | `manifests/lab-app/` (Namespace + PSA labels, hardened Deployment, Service) |
| RBAC manifests | `manifests/rbac/` (ServiceAccount, Role, RoleBinding) |

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

## Key decisions (Phase 0–3)

1. Harden SSH via a named drop-in rather than rewriting the full `sshd_config`.
2. Keep SSH on port 22; rely on keys + Fail2ban + dual firewalls.
3. Enable UFW with SSH allow first; **do not open 6443** for Phase 1–3.
4. Use official K3s install; accept single-node SQLite datastore for lab simplicity.
5. Prefer ClusterIP Services (no public app exposure) for foundations demos.
6. Keep declarative manifests in Git; use imperative kubectl only for teaching exercises.
7. Store system config backups under `/home/sysadmin/backups/` (outside Git).
8. Use a dedicated `security-reader` ServiceAccount—do not grant powers to `default`.
9. Enumerate verbs/resources (no wildcards) in the Role; bind only via RoleBinding.
10. Unsafe `cluster-admin` demo must be deleted and revalidated; never leave it standing or committed as desired state.
11. Enforce **Restricted** PSA on `lab-app`; stock `nginx:1.27.4` cannot satisfy it—use unprivileged image + SecurityContext + emptyDir mounts.
12. Privileged / hostPID / hostNetwork demos are temporary only; restore Restricted and prove rejection afterward.

## Final layered security posture (Phase 3)

The NGINX workload in `lab-app` now has Restricted PSA; non-root UID/GID 101; `allowPrivilegeEscalation: false`; all capabilities dropped; `RuntimeDefault` seccomp; read-only root filesystem; CPU/memory requests and limits; `automountServiceAccountToken: false` (no auto-mounted API token); and no privileged / `hostPID` / `hostNetwork` workloads remaining. SA token disablement is a separate workload-identity control—not enforced by PSA.

## Known limitations

- Single host; no HA, no separate management plane.
- UFW allows SSH from Anywhere at the host layer (source IP may not be stable).
- Default K3s **human** kubeconfig is still cluster-admin; Phase 2 hardened a *workload* identity, not operator SSO.
- Default allow-all Pod networking; no NetworkPolicies yet.
- No audit logging / runtime detection (Falco) yet.
- PSA does not gate SA automount; other Pods may still mount tokens unless set explicitly (nginx is disabled).
- Temporary passwordless sudo was removed after Phase 3 validation.
- Not production hardened.

## Remaining roadmap

| Phase | Intent |
|-------|--------|
| **4** | NetworkPolicies / workload isolation |
| **5** | Audit logging, metrics/detection orientation |
| **6** | Threat scenarios and remediation practice |

Related deeper topics (Secrets patterns, Falco, GitOps, human break-glass RBAC, custom seccomp) land in later phases as scoped.

## Reboot requirement

- Phase 0–3: **no reboot required** for documented work.

## Next phase

**Phase 4 — not started.** NetworkPolicies / east-west isolation. Explicit non-goals remain deferred: Falco, Helm, Ingress, multi-node, audit log configuration (Phase 5+).
