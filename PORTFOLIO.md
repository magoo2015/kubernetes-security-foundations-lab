# Portfolio — Kubernetes Security Foundations Lab

## Project overview

Hands-on **security engineering** lab on a disposable Ubuntu VPS: harden the host (Phase 0), stand up single-node **K3s** with a declarative nginx workload (Phase 1), then implement **namespace RBAC least privilege** with validation, an explicit unsafe over-privilege demo, and detection-oriented documentation (Phase 2).

Built to be explainable in interviews—not a copy-paste Helm chart dump.

## Skills demonstrated

- Host hardening: SSH, UFW, Fail2ban, unattended upgrades
- Kubernetes fundamentals: Deployments, Services, reconciliation, rollouts
- RBAC design: ServiceAccounts, Roles, RoleBindings
- Authorization testing: `kubectl auth can-i` + impersonation
- Threat modeling and detection engineering (design-level)
- Evidence hygiene: sanitized artifacts, no secrets in Git

## Security concepts

- Authentication → Authorization (RBAC) → Admission → datastore pipeline
- Least privilege and blast-radius reduction
- Dedicated identities vs `default` ServiceAccount
- Danger of `cluster-admin` on workload identities
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
lab-app namespace
   ├── Deployment/Service nginx (ClusterIP)
   ├── ServiceAccount security-reader
   ├── Role security-reader (get/list/watch pods, services, deployments)
   └── RoleBinding security-reader → SA
```

## Interview talking points

1. “I hardened the host before installing Kubernetes so the API wasn’t exposed on a soft SSH posture.”
2. “I treat RBAC as authorization only—after authn, before admission—and I validate with `can-i`, not assumptions.”
3. “I deliberately bound `cluster-admin` once to show the failure mode, then deleted it and revalidated—labs should end secure.”
4. “Default ServiceAccounts are a common footgun; dedicated SAs make audit and least privilege tractable.”
5. “Detection without audit logs is incomplete; I document the SIEM signals I’d enable next.”

## Commands learned

```bash
kubectl apply -f manifests/rbac/
kubectl get sa,role,rolebinding -n lab-app
kubectl auth can-i get pods -n lab-app \
  --as=system:serviceaccount:lab-app:security-reader
kubectl auth can-i delete pods -n lab-app \
  --as=system:serviceaccount:lab-app:security-reader
kubectl delete clusterrolebinding <unsafe-demo>
```

## Lessons learned

- Roles without bindings grant nothing—bindings are the operational control point.
- apiGroups matter (`deployments` → `apps`).
- Over-privilege is easy to grant and must be proven removed with the same tests used to prove it existed.
- Human admin kubeconfig may still be cluster-admin; workload RBAC is a separate control plane concern.

## Future roadmap

| Phase | Focus |
|-------|--------|
| 3 | NetworkPolicies / workload isolation |
| 4 | Audit logging, observability, detection |
| 5 | Threat scenarios and remediation practice |

Related: Pod Security Admission, Secrets patterns, Falco, human SSO / break-glass RBAC.
