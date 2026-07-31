# Portfolio — Kubernetes Security Foundations Lab

## Project overview

Hands-on **security engineering** lab on a disposable Ubuntu VPS: harden the host (Phase 0), stand up single-node **K3s** with a declarative nginx workload (Phase 1), implement **namespace RBAC least privilege** (Phase 2), then enforce **Pod Security Admission (Restricted)** with a hardened SecurityContext and resource limits (Phase 3).

Built to be explainable in interviews—not a copy-paste Helm chart dump.

## Skills demonstrated

- Host hardening: SSH, UFW, Fail2ban, unattended upgrades
- Kubernetes fundamentals: Deployments, Services, reconciliation, rollouts
- RBAC design: ServiceAccounts, Roles, RoleBindings
- Authorization testing: `kubectl auth can-i` + impersonation
- Pod Security Admission: Privileged / Baseline / Restricted; namespace labels
- Workload hardening: SecurityContext, capabilities, seccomp, read-only rootfs, non-root
- Workload identity: disable unused ServiceAccount token automount
- Resource governance: requests/limits and noisy-neighbor thinking
- Threat modeling and detection engineering (design-level)
- Evidence hygiene: sanitized artifacts, no secrets in Git

## Security concepts

- Authentication → Authorization (RBAC) → Admission (PSA) → datastore pipeline
- Least privilege and blast-radius reduction (identity **and** container privileges)
- Dedicated identities vs `default` ServiceAccount
- Danger of `cluster-admin` on workload identities
- Danger of privileged / hostPID / hostNetwork Pods
- Defense in depth: host firewall still blocks public API (`6443`)

## Architecture (current)

```text
Internet
   |
  SSH :22 (UFW allow)     ← Fail2ban on sshd
   |
Ubuntu 24.04 + K3s (single node)
   |
API Server :6443 (localhost / SSH session only; UFW deny public)
   |
lab-app namespace  [PSA enforce/warn/audit = restricted]
   ├── Deployment/Service nginx (ClusterIP; unprivileged UID 101)
   │     SecurityContext: non-root, drop ALL, RO rootfs, RuntimeDefault seccomp
   │     automountServiceAccountToken: false (API unused)
   │     Resources: CPU/memory requests + limits
   ├── ServiceAccount security-reader
   ├── Role security-reader (get/list/watch pods, services, deployments)
   └── RoleBinding security-reader → SA
```

## Interview talking points

1. “I hardened the host before installing Kubernetes so the API wasn’t exposed on a soft SSH posture.”
2. “I treat RBAC as authorization only—after authn, before admission—and I validate with `can-i`, not assumptions.”
3. “PSA Restricted is admission policy by namespace label; it replaced the old PodSecurityPolicy model.”
4. “Stock nginx failed Restricted because it runs as root without seccomp or capability drops—I switched to an unprivileged image and emptyDirs for a read-only rootfs.”
5. “I temporarily ran a privileged+hostPID+hostNetwork Pod to show the failure mode, then deleted it and proved Restricted rejects it again.”
6. “PSA does not stop SA token mounts—I set `automountServiceAccountToken: false` on nginx because it never calls the API.”
7. “Detection without audit logs is incomplete; I document the SIEM signals I’d enable next.”

## Commands learned

```bash
kubectl apply -f manifests/lab-app/
kubectl get ns lab-app --show-labels
kubectl exec -n lab-app deploy/nginx -- id
kubectl exec -n lab-app deploy/nginx -- cat /proc/1/status | grep Cap
kubectl auth can-i get pods -n lab-app \
  --as=system:serviceaccount:lab-app:security-reader
```

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

## Lessons learned

- Roles without bindings grant nothing—bindings are the operational control point.
- apiGroups matter (`deployments` → `apps`).
- Restricted PSA forces real workload changes, not just labels.
- PSA and SA token automount are separate controls—disable unused tokens explicitly.
- Over-privilege demos (RBAC or privileged Pods) must end with delete + revalidation evidence.
- Human admin kubeconfig may still be cluster-admin; workload RBAC and PSA are separate layers.

## Future roadmap

| Phase | Focus |
|-------|--------|
| 4 | NetworkPolicies / workload isolation |
| 5 | Audit logging, observability, detection |
| 6 | Threat scenarios and remediation practice |

Related: Secrets patterns, Falco, human SSO / break-glass RBAC, image signing in CI.
