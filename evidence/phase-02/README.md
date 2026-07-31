# Evidence — Phase 2 (sanitized)

Outputs from RBAC least-privilege setup, `kubectl auth can-i` validation, temporary unsafe `cluster-admin` demo, and cleanup.

**Redacted:** IPv4 addresses (`<IP_REDACTED>`).  
**Excluded:** kubeconfig, tokens, certificates, secrets, raw public IPs.  
**Unsafe binding:** applied only for demos `07`–`08`, deleted in `09`, confirmed absent in `10`–`11`.

| File | Contents |
|------|----------|
| `01-serviceaccounts-before.txt` | SAs in `lab-app` before apply (default only) |
| `02-apply-rbac.txt` | `kubectl apply -f manifests/rbac/` |
| `03-rbac-objects.txt` | SA / Role / RoleBinding inventory + describe |
| `04-auth-can-i-least-privilege.txt` | Expected yes/no for `security-reader` |
| `05-list-as-security-reader.txt` | Successful list pods/svc/deploy as SA |
| `06-denied-actions.txt` | Forbidden delete pods / get secrets / list nodes |
| `07-unsafe-bind-apply.txt` | Temporary ClusterRoleBinding to `cluster-admin` |
| `08-unsafe-can-i.txt` | Excessive permissions while unsafe binding existed |
| `09-unsafe-bind-delete.txt` | Removal of unsafe ClusterRoleBinding |
| `10-revalidate-after-cleanup.txt` | Permissions restored to least privilege |
| `11-final-state.txt` | App healthy; RBAC inventory; no leftover CRB |

Impersonation subject used throughout: `system:serviceaccount:lab-app:security-reader`
