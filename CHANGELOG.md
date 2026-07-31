# Changelog

All notable lab phases for this repository.

## [Phase 3] — 2026-07-31

### Features added

- Pod Security Admission labels on `lab-app` (enforce/warn/audit = **restricted**)
- Hardened nginx Deployment: `nginxinc/nginx-unprivileged:1.27.4`, non-root UID/GID 101, drop ALL capabilities, `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem`, RuntimeDefault seccomp, emptyDir mounts
- `automountServiceAccountToken: false` on nginx (workload does not call the API)
- Documented CPU/memory requests and limits (resource exhaustion / noisy neighbor / DoS)
- Temporary privileged + hostPID + hostNetwork Pod demo, then deletion and Restricted revalidation
- Sanitized evidence under `evidence/phase-03/`

### Security concepts

- PSP removal → Pod Security Admission (Privileged / Baseline / Restricted)
- SecurityContext fields as container blast-radius controls
- Seccomp RuntimeDefault and capability dropping
- PSA does not prevent SA token mounting — disable unused automount as a separate identity control
- Production policy layers: Kyverno, Gatekeeper, image signing/provenance, GitOps/CI enforcement

### Documentation added / updated

- `docs/phase-03-pod-security-admission.md`
- `docs/interview-notes.md`, `docs/lab-notes.md`
- `PORTFOLIO.md`, `CHAT_CONTEXT.md`, `CHANGELOG.md`
- Updates to `README.md`, `PROJECT_CONTEXT.md`

### Lessons

- Stock `nginx:1.27.4` fails Restricted (root, caps, no seccomp, privilege escalation defaults).
- Unprivileged image + writable emptyDirs make RO rootfs practical.
- Once enforce=restricted, even debug/curl Pods need compliant SecurityContext.
- Disabling SA automount is orthogonal to PSA; nginx keeps serving HTTP without a mounted token.

## [Phase 2] — 2026-07-31

### Features added

- ServiceAccount `security-reader` in namespace `lab-app`
- Role `security-reader` (get/list/watch on pods, services, deployments only)
- RoleBinding `security-reader` binding SA → Role
- Manifests under `manifests/rbac/`
- Temporary unsafe `cluster-admin` ClusterRoleBinding demo, then removal and revalidation
- Sanitized evidence under `evidence/phase-02/`

### Security concepts

- API request pipeline: Authentication → Authorization (RBAC) → Admission → etcd
- Least-privilege namespace RBAC; dedicated ServiceAccounts
- Risks of default SA, wildcards, and cluster-admin on workloads
- Threat model + detection engineering notes (design only)

### Documentation added / updated

- `docs/phase-02-rbac-least-privilege.md`
- `docs/interview-notes.md`
- `PORTFOLIO.md`, `CHAT_CONTEXT.md`, `CHANGELOG.md`
- Updates to `README.md`, `PROJECT_CONTEXT.md`, `docs/lab-notes.md`

## [Phase 1] — 2026-07-30

### Features added

- Single-node K3s (`v1.36.2+k3s1`)
- Declarative `lab-app` nginx Deployment + ClusterIP Service
- Scale, self-heal, rollout, rollback exercises
- Evidence under `evidence/phase-01/`

### Security concepts

- API server as trust boundary; keep `:6443` off UFW allow
- ClusterIP-only application exposure for foundations

### Documentation added / updated

- `docs/phase-01-kubernetes-foundations.md`
- Updates to `README.md`, `PROJECT_CONTEXT.md`, `docs/lab-notes.md`

## [Phase 0] — 2026-07-30

### Features added

- Ubuntu host hardening baseline
- SSH hardening drop-in, UFW, Fail2ban, time sync, unattended-upgrades
- Evidence under `evidence/phase-00/`

### Security concepts

- Harden host before installing cluster software
- Defense in depth (SSH + host firewall + Fail2ban)

### Documentation added / updated

- `docs/phase-00-host-hardening.md`
- Initial `README.md`, `PROJECT_CONTEXT.md`
