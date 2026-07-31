# CHAT_CONTEXT.md

Hand-off context for a new assistant/chat session. Read this before changing the lab.

## What this project is

**Kubernetes Security Foundations Lab** — educational Security Engineering repo on a temporary DigitalOcean Ubuntu 24.04 VPS. Prioritize least privilege, explainability, sanitized evidence, and interview readiness. Do **not** redesign the repository structure casually.

## Current architecture

- Single-node **K3s v1.36.2+k3s1** (control-plane + worker), containerd, SQLite datastore
- Admin user: `sysadmin` (sudo); SSH keys only; port 22
- UFW: deny inbound except OpenSSH; **6443 not allowed** publicly
- Fail2ban sshd jail; unattended-upgrades; timezone UTC
- Workload: namespace `lab-app`, Deployment `nginx` (1 replica), Service `nginx` ClusterIP
- RBAC: SA/Role/RoleBinding `security-reader` in `lab-app` (read-only pods/services/deployments)
- kubeconfig: `~/.kube/config` (from `/etc/rancher/k3s/k3s.yaml`) — **never commit**

## Completed phases

| Phase | Status | Summary |
|-------|--------|---------|
| **0** | Complete | Host hardening (SSH drop-in, UFW, Fail2ban, time sync, unattended-upgrades) |
| **1** | Complete | K3s install, lab-app nginx manifests, scale/rollout/rollback, evidence |
| **2** | Complete | RBAC least privilege, can-i validation, temporary unsafe CRB demo + cleanup |

## Current Kubernetes objects (Phase 2 end state)

- `Namespace/lab-app`
- `Deployment/nginx`, `Service/nginx`, ReplicaSet + Pod (Running)
- `ServiceAccount/security-reader` (+ `default`)
- `Role/security-reader`, `RoleBinding/security-reader`
- **Absent:** any `ClusterRoleBinding` granting `cluster-admin` to `security-reader`

## Security controls implemented

- Host: SSH hardening, UFW, Fail2ban, unattended-upgrades
- Cluster exposure: API not opened in UFW; app ClusterIP only
- RBAC: dedicated SA with minimal Role; validated denies for delete/create/secrets/nodes
- Process: unsafe over-privilege demo documented then removed; evidence under `evidence/phase-02/`

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
│   ├── interview-notes.md
│   └── lab-notes.md
├── manifests/
│   ├── lab-app/          # NS, Deployment, Service
│   └── rbac/             # SA, Role, RoleBinding
└── evidence/
    ├── phase-00/
    ├── phase-01/
    └── phase-02/
```

## Roadmap

1. **Phase 3** — NetworkPolicies / isolation  
2. **Phase 4** — Audit / observability / detection  
3. **Phase 5** — Threat scenarios & remediation  

Deferred: PSA, Secrets deep-dive, Falco, Helm, Terraform, Ingress/LB, multi-node, GitOps.

## Lessons learned

- Manifest apply order matters (numeric prefixes in `lab-app/`).
- RBAC rules need correct `apiGroups` (`apps` for Deployments).
- `kubectl auth can-i` + impersonation is the fast authorization unit test.
- Ending secure after an unsafe demo requires delete + revalidation evidence.

## Important design decisions

1. Security-first learning order: host → foundations → RBAC → network → audit.
2. Official K3s install; ClusterIP-only app; no public API.
3. Declarative RBAC in `manifests/rbac/`; unsafe CRB never committed as standing config.
4. Sanitize evidence (IPs/tokens); keep kubeconfig out of Git.

## Current limitations

- Operator kubeconfig is still cluster-admin (K3s default).
- No NetworkPolicies, PSA, audit sink, or runtime detection yet.
- Single node; not HA / multi-tenant production.
- Passwordless sudo drop-in may still exist for automation—remove when done.

## Suggested next phase

**Phase 3 — Network policy and workload isolation:** default-deny ingress/egress in `lab-app`, allow DNS + intentional paths, document CNI behavior on K3s, collect evidence. Do not implement PSA/audit/Falco unless the phase brief expands.

## How to continue safely

1. Read `README.md`, `PROJECT_CONTEXT.md`, and the latest `docs/phase-0X-*.md`.
2. Do not weaken Phase 0/1 controls or reintroduce `cluster-admin` bindings permanently.
3. Keep evidence sanitized; do not commit secrets.
4. Prefer updating docs + CHANGELOG when a phase completes.
