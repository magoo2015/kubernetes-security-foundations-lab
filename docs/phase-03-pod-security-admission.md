# Phase 3 — Pod Security Admission & workload hardening

## Objectives

Teach **workload hardening**: reduce the impact of a compromised container using Pod Security Admission (PSA), SecurityContext, dropped capabilities, seccomp, read-only root filesystems, and resource limits.

**Explicit non-goals (later phases):** NetworkPolicies, Secrets management, audit logging configuration, Falco.

## Part 1 — Pod Security Admission

### History: PodSecurityPolicy (PSP)

Before PSA, clusters used **PodSecurityPolicy** — a cluster-scoped admission resource that gated Pod creation based on security fields (privileged, hostNetwork, capabilities, volumes, etc.). Policies were bound to users/groups/SAs via RBAC (`use` verb on `podsecuritypolicies`).

### Why PSP was removed

- Complex and easy to misconfigure (RBAC on policies + policy selection order).
- Hard to reason about which policy applied.
- Deprecated in Kubernetes 1.21, **removed in 1.25**.

### What Pod Security Admission is

**Pod Security Admission** is a built-in admission controller (GA since 1.25) that enforces one of three **Pod Security Standards** on a **per-namespace** basis via labels. No separate policy objects or RBAC `use` bindings are required for the built-in standards.

```text
kubectl / client
        |
        v
   API Server
        |
        v
  Authentication
        |
        v
  Authorization (RBAC)
        |
        v
  Admission
     ├── ... other plugins ...
     └── Pod Security Admission  ← enforce / audit / warn by NS labels
        |
        v
  etcd / datastore
```

### Privileged / Baseline / Restricted

| Level | Intent | Typical allowances / denials |
|-------|--------|------------------------------|
| **Privileged** | Unrestricted (policy effectively off) | Anything allowed by the API |
| **Baseline** | Prevent known privilege escalations | Blocks privileged, host namespaces, hostPath, dangerous capabilities additions, etc. Still allows running as root |
| **Restricted** | Hardened best practices | Baseline + must run as non-root, drop all capabilities, no privilege escalation, RuntimeDefault/Localhost seccomp, tighter volume types |

### Namespace labels

| Label | Mode | Behavior |
|-------|------|----------|
| `pod-security.kubernetes.io/enforce` | Enforce | Non-compliant Pods are **rejected** |
| `pod-security.kubernetes.io/audit` | Audit | Non-compliant Pods are accepted; violations annotated / audit-logged |
| `pod-security.kubernetes.io/warn` | Warn | Non-compliant Pods are accepted; API returns **warnings** to the client |

Optional version labels (`*-version: latest` or a specific Kubernetes minor) pin the standard revision.

### Diagram — enforce vs warn vs audit

```text
                    Pod create/update
                            |
              +-------------+-------------+
              |             |             |
           enforce        warn         audit
              |             |             |
         compliant?    compliant?    compliant?
           / \           / \           / \
         yes  no       yes  no       yes  no
          |    |        |    |        |    |
       admit reject  admit +warn   admit +audit annotation
```

### Diagram — Restricted controls on a Pod

```text
                    ┌─────────────────────────────────┐
                    │           Pod (lab-app)         │
                    │  runAsNonRoot / runAsUser≠0     │
                    │  seccomp: RuntimeDefault        │
                    │  fsGroup (volume ownership)     │
                    │  automountServiceAccountToken   │
                    │    = false (identity; not PSA)  │
                    │                                 │
                    │  ┌───────────────────────────┐  │
                    │  │ Container                 │  │
                    │  │  allowPrivilegeEscalation │  │
                    │  │    = false                │  │
                    │  │  capabilities.drop: ALL   │  │
                    │  │  readOnlyRootFilesystem   │  │
                    │  │  resources req+lim        │  │
                    │  └───────────────────────────┘  │
                    └─────────────────────────────────┘
```

## Part 2 — Enable PSA on `lab-app`

Labels applied in `manifests/lab-app/00-namespace.yaml`:

```yaml
pod-security.kubernetes.io/enforce: restricted
pod-security.kubernetes.io/warn: restricted
pod-security.kubernetes.io/audit: restricted
# plus matching *-version: latest
```

### Why stock `nginx:1.27.4` failed Restricted immediately

Existing Phase 1 Pods violated Restricted with:

- `allowPrivilegeEscalation != false` (default allows escalation)
- unrestricted capabilities (default bounding set)
- `runAsNonRoot != true` (process ran as UID 0)
- missing `seccompProfile`

Stock `nginx:1.27.4` also listens on privileged port **80** and expects a writable root filesystem for PID/cache/logs — incompatible with non-root + `readOnlyRootFilesystem` without further customization.

### Manifest changes to pass Restricted

1. Switch image to `nginxinc/nginx-unprivileged:1.27.4` (UID **101**, listen **8080**).
2. Set pod + container `securityContext` fields required by Restricted.
3. Mount `emptyDir` volumes at `/tmp`, `/var/cache/nginx`, `/var/run` so nginx can write with a read-only root FS.
4. Keep Service `port: 80` → `targetPort: http` (container 8080).
5. Set `automountServiceAccountToken: false` (separate workload-identity control — see below).

## Part 3 — SecurityContext field reference

| Field | Where | Purpose |
|-------|-------|---------|
| `runAsNonRoot: true` | Pod + container | Reject start if UID would be 0 |
| `runAsUser: 101` | Pod + container | Fixed non-root UID (nginx unprivileged user) |
| `runAsGroup: 101` | Pod | Primary GID for processes |
| `fsGroup: 101` | Pod | Volume group ownership / writable by GID 101 |
| `allowPrivilegeEscalation: false` | Container | Blocks `no_new_privs` bypass / setuid escalation paths |
| `readOnlyRootFilesystem: true` | Container | Immutable container root; writable only via mounts |
| `capabilities.drop: [ALL]` | Container | Remove Linux capabilities from the bounding set |
| `seccompProfile.type: RuntimeDefault` | Pod + container | Apply container runtime’s default syscall filter |
| `automountServiceAccountToken: false` | Pod | Do not mount API credentials (not a PSA control) |

### Workload identity: `automountServiceAccountToken`

**Pod Security Admission does not itself prevent ServiceAccount token mounting.** Restricted/Baseline/Privileged gate privilege, host namespaces, capabilities, seccomp, and related PodSpec fields — they do **not** require or forbid projected SA tokens.

nginx in this lab never calls the Kubernetes API, so the Pod template sets:

```yaml
spec:
  automountServiceAccountToken: false
```

That is a **separate least-privilege / workload-identity** control: a compromised container has no in-Pod API bearer token to abuse. Validate with the live PodSpec (`automountServiceAccountToken: false`) and by confirming `/var/run/secrets/kubernetes.io/serviceaccount` is absent.

## Part 4 — Resource limits

Requests/limits on the nginx container:

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | `50m` | `250m` |
| Memory | `64Mi` | `128Mi` |

### Why they matter

| Concept | Explanation |
|---------|-------------|
| **Resource exhaustion** | Unbounded Pods can consume node CPU/RAM until kubelet eviction or node failure |
| **Noisy neighbor** | One workload steals CPU/memory from co-located Pods on the same node |
| **Denial of service** | Accidental loops or intentional abuse without limits amplify impact; limits contain blast radius (with requests enabling scheduling fairness) |

Requests also drive the scheduler’s bin-packing decisions; limits cap runtime usage (CPU throttling; memory OOMKill when over limit).

## Part 5 — Unsafe privileged Pod (temporary)

A temporary Pod demonstrated:

| Setting | Why dangerous |
|---------|----------------|
| `privileged: true` | Near-host privileges inside the container (devices, capabilities, weaker isolation) |
| `hostPID: true` | See/signal host processes; aids escape and credential scraping |
| `hostNetwork: true` | Share host network namespace; sniff/bind host interfaces; bypass NetworkPolicy paths |

**Not left deployed.** Applied only for evidence, then deleted. Enforce=restricted rejects such Pods in `lab-app` after cleanup validation.

## Part 6 — Validation checklist

| Check | Expected |
|-------|----------|
| PSA labels on `lab-app` | enforce/warn/audit = `restricted` |
| Deployment Available | `1/1` Ready |
| Process UID | non-root (`101`) |
| Capabilities | dropped (effective set empty / minimal) |
| Root filesystem | read-only (`/` not writable) |
| Restricted admit | Deployment/Pod accepted |
| App reachable | `wget`/`curl` via ClusterIP or `kubectl exec` probe to Service |
| SA automount | `automountServiceAccountToken: false`; SA mount path absent |
| Privileged demo | deleted; absent from inventory |

## Part 7 — Threat model

| Threat | Impact | Mitigation (this phase) | Residual risk |
|--------|--------|-------------------------|---------------|
| **Container breakout** | Escape to node | Restricted PSA; no privileged; drop caps; seccomp; non-root; RO rootfs | Kernel/runtime bugs; misconfig elsewhere |
| **Privilege escalation** | Gain root / caps / host access | `allowPrivilegeEscalation: false`; drop ALL; non-root | Privileged Pods in other namespaces; node admin |
| **Host filesystem access** | Read host secrets/kubelet creds | No hostPath; Restricted volume rules; RO rootfs | hostPath in unconstrained NS; node compromise |
| **Credential theft** | Steal SA tokens / kubeconfig material | `automountServiceAccountToken: false` on nginx; RO rootfs; NetworkPolicy later | Tokens still mount by default on other Pods; human kubeconfig |
| **Kernel attack surface** | Exploit syscalls / privileged ops | seccomp RuntimeDefault; drop capabilities | Custom profiles not yet tuned; shared kernel |

## Part 8 — Detection engineering (docs only)

Map unsafe PodSpecs to **audit log** / SIEM ideas (implement sinks in Phase 4+):

| Signal | What to look for |
|--------|------------------|
| Privileged Pods | `securityContext.privileged=true` on create/update |
| hostPath | `volumes[].hostPath` present |
| hostNetwork / hostPID | `spec.hostNetwork` / `spec.hostPID` true |
| Root containers | missing `runAsNonRoot` or `runAsUser: 0` |
| Privilege escalation allowed | `allowPrivilegeEscalation: true` or field omitted where policy expects false |
| Missing seccomp | no `seccompProfile` / not RuntimeDefault\|Localhost |
| Missing resource limits | empty `resources.limits` |

Correlate with namespace PSA audit annotations and RBAC identity that created the Pod.

## Part 9 — Production considerations

Built-in PSA is necessary but not sufficient for production multi-tenant clusters:

| Control | Role |
|---------|------|
| **Kyverno** | Policy-as-code (mutate/validate/generate); image policy, required labels, SecurityContext defaults |
| **OPA Gatekeeper** | ConstraintTemplates + Constraints for custom admission rules |
| **Image signing** | Cosign/Sigstore signatures so only signed images run |
| **Image provenance** | SLSA / attestations linking image → source commit → builder |
| **Admission webhooks** | Org-specific validate/mutate beyond PSA levels |
| **GitOps validation** | Reject drifts; policy checks in Argo CD / Flux sync windows |
| **CI/CD policy enforcement** | Fail PRs that open privileged fields before cluster apply |

Prefer **fail closed** in production namespaces: enforce Restricted (or stricter custom policy), signed images, and no standing privileged workloads.

## Apply / verify

```bash
export KUBECONFIG=~/.kube/config
kubectl apply -f manifests/lab-app/
kubectl get ns lab-app --show-labels
kubectl get deploy,pods -n lab-app
kubectl exec -n lab-app deploy/nginx -- id
kubectl exec -n lab-app deploy/nginx -- sh -c 'touch /writable-test 2>&1 || true'
kubectl get svc nginx -n lab-app
kubectl run curl-test -n lab-app --rm -i --restart=Never \
  --image=curlimages/curl:8.5.0 -- \
  curl -sS -o /dev/null -w '%{http_code}\n' http://nginx.lab-app.svc.cluster.local/
```

## Evidence

See `evidence/phase-03/README.md`.

## Final layered security posture

The NGINX workload in `lab-app` now has:

- Restricted Pod Security Admission (enforce / warn / audit)
- Non-root execution as UID/GID 101
- `allowPrivilegeEscalation: false`
- All Linux capabilities dropped
- `RuntimeDefault` seccomp
- Read-only root filesystem
- CPU and memory requests and limits
- `automountServiceAccountToken: false`
- No automatically mounted Kubernetes API token
- No privileged, `hostPID`, or `hostNetwork` workloads remaining

Disabling the ServiceAccount token is a separate workload-identity hardening control and is **not** enforced by Pod Security Admission. PSA gates privilege, host namespaces, capabilities, seccomp, and related PodSpec fields; SA token automount must be set explicitly on the Pod (or ServiceAccount).

## Limitations

- PSA is namespace-scoped; `kube-system` and other system namespaces are not Restricted here.
- Default seccomp profile is not a custom tight filter.
- No NetworkPolicies yet — east-west traffic still unrestricted at L3/L4.
- PSA does not enforce SA automount policy — that remains an explicit PodSpec/SA setting (done for nginx here).
- Single-node K3s lab — not multi-tenant production.
