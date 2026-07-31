# CHAT_CONTEXT.md

Hand-off context for a new assistant/chat session. Read this before changing the lab.

## What this project is

**Kubernetes Security Foundations Lab** — educational Security Engineering repo on a temporary DigitalOcean Ubuntu 24.04 VPS. Prioritize least privilege, explainability, sanitized evidence, and interview readiness. Do **not** redesign the repository structure casually.

## Current architecture

- Single-node **K3s v1.36.2+k3s1** (control-plane + worker), containerd, SQLite datastore
- Admin user: `sysadmin` (sudo); SSH keys only; port 22
- UFW: deny inbound except OpenSSH; **6443 not allowed** publicly
- Fail2ban sshd jail; unattended-upgrades; timezone UTC
- Workload: namespace `lab-app`, Deployment `nginx` (`nginxinc/nginx-unprivileged:1.27.4`, 1 replica), Service `nginx` ClusterIP (:80 → 8080)
- PSA on `lab-app`: enforce/warn/audit = **restricted**
- Pod SecurityContext: non-root UID/GID 101, RuntimeDefault seccomp, fsGroup 101
- Container SecurityContext: `allowPrivilegeEscalation: false`, `readOnlyRootFilesystem: true`, `capabilities.drop: [ALL]`
- `automountServiceAccountToken: false` on nginx (does not call the API; PSA does not enforce this)
- Resources: CPU 50m/250m, memory 64Mi/128Mi
- RBAC: SA/Role/RoleBinding `security-reader` in `lab-app` (read-only pods/services/deployments)
- kubeconfig: `~/.kube/config` (from `/etc/rancher/k3s/k3s.yaml`) — **never commit**

## Completed phases

| Phase | Status | Summary |
|-------|--------|---------|
| **0** | Complete | Host hardening (SSH drop-in, UFW, Fail2ban, time sync, unattended-upgrades) |
| **1** | Complete | K3s install, lab-app nginx manifests, scale/rollout/rollback, evidence |
| **2** | Complete | RBAC least privilege, can-i validation, temporary unsafe CRB demo + cleanup |
| **3** | Complete | PSA Restricted, hardened SecurityContext, resource limits, SA automount disabled, temporary privileged Pod demo + cleanup |

## Current Kubernetes objects (Phase 3 end state)

- `Namespace/lab-app` (PSA Restricted labels; `phase=03`)
- `Deployment/nginx`, `Service/nginx`, ReplicaSet + Pod (Running, non-root)
- `ServiceAccount/security-reader` (+ `default`)
- `Role/security-reader`, `RoleBinding/security-reader`
- **Absent:** `cluster-admin` CRB for `security-reader`; privileged / hostNetwork / hostPID demo Pods

## Security controls implemented

- Host: SSH hardening, UFW, Fail2ban, unattended-upgrades
- Cluster exposure: API not opened in UFW; app ClusterIP only
- RBAC: dedicated SA with minimal Role; validated denies for delete/create/secrets/nodes
- Workload: Restricted PSA; non-root; drop ALL caps; RO rootfs; RuntimeDefault seccomp; CPU/memory limits; SA automount off
- Process: unsafe privileged demo documented then removed; evidence under `evidence/phase-03/`

## Current workload protections

| Control | State |
|---------|-------|
| PSA Restricted (enforce/warn/audit) | Enabled on `lab-app` |
| runAsNonRoot / runAsUser 101 | Enforced |
| allowPrivilegeEscalation false | Set |
| capabilities drop ALL | Set |
| readOnlyRootFilesystem | Set (+ emptyDir mounts) |
| seccomp RuntimeDefault | Set |
| automountServiceAccountToken | false (nginx) |
| Resource requests/limits | Set |

## Repository layout

```text
kubernetes-security-foundations-lab/
├── README.md
├── PROJECT_CONTEXT.md
├── PORTFOLIO.md
├── CHAT_CONTEXT.md
├── CHANGELOG.md
├── docs/
│   ├── phase-00-host-hardening.md
│   ├── phase-01-kubernetes-foundations.md
│   ├── phase-02-rbac-least-privilege.md
│   ├── phase-03-pod-security-admission.md
│   ├── interview-notes.md
│   └── lab-notes.md
├── manifests/
│   ├── lab-app/          # NS+PSA, hardened Deployment, Service
│   └── rbac/             # SA, Role, RoleBinding
└── evidence/
    ├── phase-00/
    ├── phase-01/
    ├── phase-02/
    └── phase-03/
```

## Roadmap

1. **Phase 4** — NetworkPolicies / isolation
2. **Phase 5** — Audit / observability / detection
3. **Phase 6** — Threat scenarios & remediation

Deferred: Secrets deep-dive, Falco, Helm, Terraform, Ingress/LB, multi-node, GitOps, custom seccomp profiles.

## Lessons learned

- Manifest apply order matters (numeric prefixes in `lab-app/`).
- RBAC rules need correct `apiGroups` (`apps` for Deployments).
- `kubectl auth can-i` + impersonation is the fast authorization unit test.
- Stock nginx runs as root and fails Restricted; unprivileged image + emptyDirs unlock RO rootfs.
- PSA `enforce` rejects unsafe Pods; temporary demos must restore Restricted and re-prove rejection.
- Probe/debug Pods also need Restricted-compliant SecurityContext once enforce is on.

## Important design decisions

1. Security-first learning order: host → foundations → RBAC → PSA/workload → network → audit.
2. Official K3s install; ClusterIP-only app; no public API.
3. Declarative RBAC in `manifests/rbac/`; unsafe CRB never committed as standing config.
4. Restricted PSA labels live on the Namespace manifest; unsafe privileged Pod never stored as desired state.
5. Sanitize evidence (IPs/tokens); keep kubeconfig out of Git.

## Final layered security posture (Phase 3)

The NGINX workload in `lab-app` now has:

- Restricted Pod Security Admission
- Non-root execution as UID/GID 101
- `allowPrivilegeEscalation: false`
- All Linux capabilities dropped
- `RuntimeDefault` seccomp
- Read-only root filesystem
- CPU and memory requests and limits
- `automountServiceAccountToken: false`
- No automatically mounted Kubernetes API token
- No privileged, `hostPID`, or `hostNetwork` workloads remaining

Disabling the ServiceAccount token is a separate workload-identity hardening control and is not enforced by Pod Security Admission.

## Current limitations

- Operator kubeconfig is still cluster-admin (K3s default).
- No NetworkPolicies, audit sink, or runtime detection yet.
- PSA does not gate SA automount; other Pods may still mount tokens unless set explicitly.
- Single node; not HA / multi-tenant production.
- Temporary passwordless sudo was removed after Phase 3 validation.

## Suggested next phase

**Phase 4 — Network policy and workload isolation:** default-deny ingress/egress in `lab-app`, allow DNS + intentional paths, document CNI behavior on K3s, collect evidence. Do not implement audit/Falco unless the phase brief expands.

## How to continue safely

1. Read `README.md`, `PROJECT_CONTEXT.md`, and the latest `docs/phase-0X-*.md`.
2. Do not weaken Phase 0–3 controls or reintroduce privileged / cluster-admin demos permanently.
3. Keep evidence sanitized; do not commit secrets.
4. Prefer updating docs + CHANGELOG when a phase completes.
