# Interview notes — Kubernetes Security Foundations Lab

Concise Q&A for security / platform interviews. Prefer real commands and failure modes over buzzwords.

---

### 1. Where does RBAC sit in the API request path?

**Answer:** After authentication, before admission. AuthN establishes identity; RBAC allow/denies the verb/resource; admission mutates/validates the object; then etcd persists.

```text
kubectl → API Server → Authentication → Authorization (RBAC) → Admission → etcd
```

---

### 2. What is the difference between a Role and a ClusterRole?

**Answer:** A **Role** is namespaced. A **ClusterRole** is cluster-scoped (and can also be referenced by a RoleBinding for namespaced grants of cluster-defined rules). Prefer Roles for app least privilege; reserve ClusterRoles for node/system or truly cluster-wide needs.

---

### 3. Why does creating a Role alone change nothing?

**Answer:** Permissions apply only when a **RoleBinding** or **ClusterRoleBinding** attaches the role to subjects (User, Group, ServiceAccount). Untied Roles are inert policy documents.

```bash
kubectl get role,rolebinding -n lab-app
kubectl describe rolebinding security-reader -n lab-app
```

---

### 4. Why avoid the `default` ServiceAccount for applications?

**Answer:** Every Pod without an explicit SA uses `default`. Binding powers to `default` grants them to *all* such Pods, expanding blast radius and muddying audit attribution. Use a dedicated SA per workload/permission set.

---

### 5. How do you prove a ServiceAccount can list Pods but not delete them?

**Answer:** Impersonate and use `kubectl auth can-i`, then attempt the action.

```bash
AS=--as=system:serviceaccount:lab-app:security-reader
kubectl auth can-i get pods -n lab-app $AS      # yes
kubectl auth can-i delete pods -n lab-app $AS   # no
kubectl delete pod -n lab-app -l app=nginx $AS  # Forbidden
```

---

### 6. What is dangerous about binding `cluster-admin` to a workload SA?

**Answer:** Any process with that SA token can manage all resources (Secrets, RBAC, privileged Pods)—full cluster takeover after a single Pod compromise. Use break-glass only, time-box, and alert on such bindings.

---

### 7. Wildcard verbs or resources—what goes wrong?

**Answer:** `verbs: ["*"]` includes delete and often escalate paths. `resources: ["*"]` frequently includes Secrets and RBAC objects. Enumerate the minimum verbs and resources.

---

### 8. How would you troubleshoot “Forbidden” for a CI ServiceAccount?

**Answer:** Confirm identity, binding scope, and rule match.

```bash
kubectl auth can-i create deployments.apps -n lab-app \
  --as=system:serviceaccount:lab-app:ci-deployer -v=8
kubectl get rolebinding,clusterrolebinding -A | grep ci-deployer
kubectl describe role ci-deployer -n lab-app
```

Check namespace mismatch (Role in wrong NS) and apiGroup mistakes (`deployments` need `apps`).

---

### 9. Role vs ClusterRoleBinding—common confusion?

**Answer:** A **ClusterRoleBinding** grants a ClusterRole *cluster-wide* to subjects. A **RoleBinding** can reference a Role *or* a ClusterRole but grants only in that RoleBinding’s namespace. Binding `cluster-admin` via ClusterRoleBinding is global admin.

---

### 10. What audit/SIEM signals indicate RBAC abuse?

**Answer:** Creates/updates of ClusterRoleBindings; subjects bound to `cluster-admin`; spikes of HTTP 403 for one SA; unexpected `impersonate`; verbs `bind`/`escalate` on `rbac.authorization.k8s.io`. Correlate with pod exec / new privileged pods.

---

### 11. How are ServiceAccount tokens mounted today?

**Answer:** Prefer projected, short-lived tokens via the TokenRequest API (path under `/var/run/secrets/kubernetes.io/serviceaccount/`). Long-lived Secret-backed tokens are legacy. Disable automount when the Pod never calls the API (`automountServiceAccountToken: false`).

---

### 12. Lab command set worth memorizing

```bash
kubectl apply -f manifests/rbac/
kubectl get sa,role,rolebinding -n lab-app
kubectl auth can-i --list -n lab-app \
  --as=system:serviceaccount:lab-app:security-reader
kubectl delete clusterrolebinding security-reader-unsafe-demo  # cleanup demo
```

---

## Phase 3 — Pod Security Admission & workload hardening

### 13. What replaced PodSecurityPolicy, and why was PSP removed?

**Answer:** **Pod Security Admission (PSA)** replaced PSP. PSP was complex (policy objects + RBAC `use` bindings + ordering), deprecated in 1.21, and **removed in 1.25**. PSA applies built-in Pod Security Standards per namespace via labels (`enforce` / `audit` / `warn`).

---

### 14. Privileged vs Baseline vs Restricted—how do you choose?

**Answer:** **Privileged** = unrestricted (effectively off). **Baseline** = block known dangerous escalations but still allow root. **Restricted** = hardened defaults (non-root, drop all caps, no privilege escalation, seccomp required). Use Restricted for app namespaces when workloads can comply; keep system namespaces carefully exempted/labeled.

---

### 15. What do the PSA namespace labels do?

**Answer:**

| Label | Effect on non-compliant Pods |
|-------|------------------------------|
| `pod-security.kubernetes.io/enforce` | **Reject** |
| `pod-security.kubernetes.io/warn` | Admit + client **warning** |
| `pod-security.kubernetes.io/audit` | Admit + **audit** annotation/log |

Production pattern: enforce Restricted where possible; use warn/audit while migrating.

---

### 16. Name SecurityContext fields you set for Restricted and why.

**Answer:** `runAsNonRoot` / `runAsUser` (no UID 0); `allowPrivilegeEscalation: false` (no_new_privs); `capabilities.drop: [ALL]`; `seccompProfile: RuntimeDefault`; often `readOnlyRootFilesystem: true` (not strictly all PSA levels, but strong hardening). Pod-level `fsGroup` helps volume writability for non-root.

---

### 17. Why drop all capabilities?

**Answer:** Linux capabilities split root powers (e.g., `NET_ADMIN`, `SYS_ADMIN`). Leaving the default set expands what a compromised process can do. Dropping `ALL` (and adding back only what is required) shrinks the kernel attack surface.

---

### 18. What does seccomp RuntimeDefault buy you?

**Answer:** Filters syscalls to the runtime’s default allow-list, blocking many obscure/dangerous calls used in exploits. It is not a custom profile—but it is the Restricted baseline and far better than unconfined.

---

### 19. Root vs non-root in containers—interview framing?

**Answer:** Root in a container is not “root on the host,” but with privileged mode, hostPath, or capability-rich configs it can become a breakout path. Non-root + RO rootfs + dropped caps raises the cost of escape and limits write/credential abuse inside the container.

---

### 20. Why are privileged, hostPID, and hostNetwork dangerous?

**Answer:**

- **privileged: true** — near-host privileges (devices, full caps, weak isolation).
- **hostPID** — see/signal host processes; aids escape and secret scraping.
- **hostNetwork** — share host network stack; sniff/bind host ports; weaken network policy assumptions.

---

### 21. Explain resource requests vs limits for security.

**Answer:** **Requests** influence scheduling and fair share; **limits** cap usage. Without limits, a compromised or buggy Pod can cause **resource exhaustion**, **noisy neighbor** impact, and node-level **DoS** (CPU starve / memory pressure / eviction cascades).

---

### 22. Scenario: “Our Restricted namespace broke nginx—what do you do?”

**Answer:** Read the PSA admission error; check UID, capabilities, privilege escalation, seccomp, and volume types. For nginx: use an unprivileged image or custom config on :8080, set SecurityContext, and mount emptyDirs for cache/PID/tmp if using read-only rootfs. Do not weaken the whole namespace to Privileged to “make it work.”

---

### 23. Scenario: How would you detect privileged Pods in production?

**Answer:** Audit-log / SIEM rules on Pod create/update where `securityContext.privileged=true`, or `hostPath` / `hostNetwork` / `hostPID` appear; alert on namespaces that should be Restricted; Gatekeeper/Kyverno deny + notify; periodic `kubectl`/API inventory for privileged workloads.

---

### 24. Does Pod Security Admission prevent ServiceAccount token mounting?

**Answer:** **No.** PSA enforces Pod Security Standards (privilege, host namespaces, capabilities, seccomp, etc.). It does **not** control whether a projected ServiceAccount token is mounted. Disable unused tokens explicitly with `automountServiceAccountToken: false` on the Pod (or SA). In this lab, nginx never calls the API, so the Deployment sets that field as a separate workload-identity hardening control.

---

### 25. Lab command set (Phase 3)

```bash
kubectl apply -f manifests/lab-app/
kubectl get ns lab-app --show-labels
kubectl exec -n lab-app deploy/nginx -- id
kubectl get deploy nginx -n lab-app -o jsonpath='{.spec.template.spec.containers[0].securityContext}'
kubectl get pods -n lab-app -l app=nginx -o jsonpath='{.items[0].spec.automountServiceAccountToken}{"\n"}'
```
