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
