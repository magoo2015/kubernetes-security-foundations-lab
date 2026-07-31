# Phase 2 — RBAC & least privilege

## Objectives

Teach Kubernetes **authorization** from a security engineering perspective: identities (ServiceAccounts), scoped permissions (Roles), bindings, validation with `kubectl auth can-i`, unsafe over-privilege demos that are immediately reverted, threat modeling, and detection ideas.

**Explicit non-goals (later phases):** Pod Security Admission, NetworkPolicies, Secrets hardening, audit logging configuration, Falco.

## Authorization overview

Every request to the Kubernetes API Server is processed in a fixed pipeline. Understanding **where RBAC sits** prevents mixing up authentication, authorization, and admission.

### Authentication vs authorization vs admission

| Stage | Question answered | Examples |
|-------|-------------------|----------|
| **Authentication** | *Who* is calling? | Client certs, bearer tokens (ServiceAccount JWT), OIDC, webhook auth |
| **Authorization** | Is this identity *allowed* to do this verb on this resource? | **RBAC** (Roles/ClusterRoles + bindings), ABAC (legacy), Node, Webhook |
| **Admission** | Should this object be *mutated or rejected* after authz? | Validating/Mutating webhooks, PSA, LimitRanger, ResourceQuota |

RBAC does **not** identify callers and does **not** rewrite PodSpecs. It only answers allow/deny for an already-authenticated subject on a specific API request.

### Request flow (ASCII)

```text
kubectl / client
        |
        v
   API Server
        |
        v
  Authentication   ← establish identity (User / ServiceAccount)
        |
        v
  Authorization    ← RBAC (Role / ClusterRole + Binding)
        |
        v
  Admission        ← mutate/validate objects (later phases)
        |
        v
  etcd / datastore ← persist accepted state
```

If authentication fails → 401. If RBAC denies → 403. If admission rejects → 4xx with admission message. Only then is state written.

## ServiceAccounts

### What they are

A **ServiceAccount** is a namespaced identity for processes running in the cluster (typically Pods). Humans usually authenticate as Users (certs, OIDC); workloads authenticate as ServiceAccounts.

### Why every Pod receives one

If a PodSpec omits `serviceAccountName`, Kubernetes assigns the namespace **`default`** ServiceAccount. Controllers, kubelets, and the API expect an identity for projected tokens and for any in-cluster API calls.

### How tokens are mounted

Modern clusters project a short-lived audience-bound token into the Pod (TokenRequest API), typically at `/var/run/secrets/kubernetes.io/serviceaccount/token`, plus CA and namespace files. Older clusters mounted a long-lived Secret-backed token (discouraged).

### Risks of the default ServiceAccount

- Many teams grant permissions to `default`, so **every Pod** in the namespace inherits them.
- Shared identity blurs blast radius: compromise of any Pod ≈ compromise of that permission set.
- Harder to audit (“which workload needed this?”).

### Dedicated ServiceAccounts (preferred)

Create a ServiceAccount per workload or permission set (here: `security-reader`), bind only the Role it needs, and avoid binding powerful Roles to `default`.

```bash
kubectl get serviceaccounts -n lab-app
kubectl apply -f manifests/rbac/
```

## Role (least privilege)

`manifests/rbac/10-role.yaml` defines Role `security-reader` in `lab-app` with **only**:

| apiGroup | resources | verbs |
|----------|-----------|-------|
| `""` (core) | `pods`, `services` | `get`, `list`, `watch` |
| `apps` | `deployments` | `get`, `list`, `watch` |

No Secrets, no Nodes, no create/update/delete, no wildcards.

## RoleBinding

`manifests/rbac/20-rolebinding.yaml` binds Role `security-reader` to ServiceAccount `security-reader`.

**Roles do nothing until bound.** Creating a Role only stores a permission *template*; the RoleBinding (or ClusterRoleBinding) attaches that template to subjects.

## Validation (`kubectl auth can-i`)

Impersonate the ServiceAccount (requires your kubeconfig user can impersonate):

```bash
AS="--as=system:serviceaccount:lab-app:security-reader"

kubectl auth can-i get pods -n lab-app $AS          # yes
kubectl auth can-i get services -n lab-app $AS      # yes
kubectl auth can-i get deployments.apps -n lab-app $AS  # yes
kubectl auth can-i delete pods -n lab-app $AS       # no
kubectl auth can-i create deployments.apps -n lab-app $AS  # no
kubectl auth can-i get secrets -n lab-app $AS       # no
kubectl auth can-i list nodes $AS                   # no
```

Sanitized outputs: `evidence/phase-02/`.

## Unsafe example (temporary)

To show the cost of over-privilege, a **temporary** ClusterRoleBinding of `cluster-admin` to `security-reader` was applied, validated (`can-i '*' '*' → yes`), then **deleted**. The lab must finish without that binding.

Why dangerous: `cluster-admin` can read all Secrets, create privileged Pods, modify RBAC, and effectively own the cluster. Binding it to a workload identity turns any Pod compromise into full cluster compromise.

## Common RBAC mistakes

| Mistake | Why it hurts |
|---------|----------------|
| Using `cluster-admin` everywhere | No least privilege; every leak is catastrophic |
| Using `default` ServiceAccount for apps | Shared identity; accidental broad bindings affect all Pods |
| Wildcard verbs (`*`) | Grants delete/escalate paths you did not intend |
| Wildcard resources (`*`) | Often includes Secrets and RBAC objects |
| Namespace confusion | Role in `lab-app` ≠ ClusterRole; wrong binding scope |
| Forgotten bindings | Role exists but subject still denied (or old binding still grants) |
| Overly broad ClusterRoles | Cluster-wide read/write when namespace Role would suffice |

## Threat model (summary)

| Threat | Impact | Likelihood (lab) | Mitigation | Detection |
|--------|--------|------------------|------------|-----------|
| Compromised Pod | Abuse mounted SA token for API calls | Medium | Dedicated SA; minimal Role; `automountServiceAccountToken: false` if unused | Authz failures; unexpected SA API usage |
| Credential theft | Stolen kubeconfig / token → API as that identity | Medium | Protect kubeconfig; short-lived tokens; no public 6443 | New sessions; unusual source; can-i / audit |
| Privilege escalation | Bind self to privileged Role/ClusterRole | Low–Med if RBAC weak | Deny `bind`/`escalate`/`*` on rbac; avoid cluster-admin | New RoleBindings / ClusterRoleBindings |
| Lateral movement | Read Secrets / create Pods in other NS | Medium if wildcards | Namespace Roles; no cluster-wide read | Cross-namespace access; Secret gets |
| Accidental admin | Human error binding cluster-admin | High without review | PR review; policy; temporary break-glass only | Alert on cluster-admin bindings |

## Detection engineering (documentation only)

With Kubernetes **audit logs** (Phase 4+), useful signals include:

- `create`/`update`/`patch` on `clusterrolebindings` / `rolebindings`
- Binding `roleRef.name: cluster-admin` or subjects unexpected ServiceAccounts
- High volume of `ResponseStatus` with code `403` for one identity
- `create` pods with privileged capabilities after a binding change
- Impersonation headers used outside break-glass procedures

SIEM ideas: correlation rules for new ClusterRoleBindings, unexpected `cluster-admin` assignments, ServiceAccount token use from unusual nodes, repeated authorization failures, privilege-escalation verb patterns (`bind`, `escalate`, `impersonate`).

## Apply / verify

```bash
kubectl apply -f manifests/rbac/
kubectl get sa,role,rolebinding -n lab-app
kubectl get clusterrolebinding security-reader-unsafe-demo 2>/dev/null || echo "unsafe binding absent (good)"
kubectl get all -n lab-app
```

## Evidence

See `evidence/phase-02/README.md`.

## Limitations

- Admin still uses default K3s kubeconfig (**cluster-admin**) for lab operations; Phase 2 teaches *workload* least privilege, not human SSO.
- No audit log sink configured yet (detection remains design-only).
- Single-node K3s; no multi-tenant isolation beyond namespace RBAC.
