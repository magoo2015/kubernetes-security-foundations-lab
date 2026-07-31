# Evidence — Phase 3 (sanitized)

Outputs from Pod Security Admission (Restricted), workload SecurityContext hardening, resource-limit documentation, ServiceAccount automount disablement, temporary privileged Pod demo, and cleanup.

**Redacted:** IPv4 addresses (`<IP_REDACTED>`).
**Excluded:** kubeconfig, tokens, certificates, secrets, raw public IPs.
**Unsafe Pod:** created only under temporary `enforce=privileged` for demo `08`, deleted immediately; re-create under Restricted rejected (`08` footer / `08a`).

| File | Contents |
|------|----------|
| `01-before-restricted-violations.txt` | Pre-harden UID 0; Restricted dry-run violations |
| `02-apply-hardened-manifests.txt` | `kubectl apply -f manifests/lab-app/` |
| `03-rollout-status.txt` | Deployment rollout success |
| `04-namespace-psa-labels.txt` | enforce/warn/audit = restricted |
| `05-deployment-securitycontext.txt` | Pod/container SecurityContext + resources + automount |
| `06-runtime-hardening-checks.txt` | non-root, Cap*=0, RO rootfs, emptyDir writable |
| `07-app-reachability.txt` | In-cluster HTTP `200` via Restricted curl Pod |
| `08-unsafe-privileged-demo.txt` | Temporary privileged+hostPID+hostNetwork Pod; delete; restore Restricted |
| `08a-unsafe-privileged-rejected-by-psa.txt` | Direct create under Restricted → Forbidden |
| `09-final-state.txt` | Healthy hardened nginx; automount false; no privileged leftover |
| `10-automount-sa-token-disabled.txt` | Rollout + Ready + HTTP 200 + SA path absent + PSA still restricted |

Image after harden: `nginxinc/nginx-unprivileged:1.27.4` (UID 101, port 8080).
Workload identity: `automountServiceAccountToken: false` (PSA does not enforce this).
