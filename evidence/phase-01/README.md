# Evidence — Phase 1 (sanitized)

Outputs captured during K3s install and `lab-app` workload exercises.

**Redacted:** IPv4 addresses (`<IP_REDACTED>`).  
**Excluded:** kubeconfig, tokens, certificates, secrets, raw public IPs.

| File | Contents |
|------|----------|
| `01-get-nodes.txt` | Node Ready / version / runtime |
| `02-get-pods.txt` | nginx Pod(s) in `lab-app` |
| `03-get-deployments.txt` | Deployment status |
| `04-get-services.txt` | ClusterIP Service |
| `05-get-replicasets.txt` | ReplicaSet(s) |
| `06-get-all-lab-app.txt` | `kubectl get all -n lab-app` |
| `07-scale-to-3.txt` | Scale 1→3 |
| `08-scale-to-1.txt` | Scale 3→1 |
| `09-pod-delete-recreate.txt` | Pod delete → reconciliation |
| `10-rollout.txt` | Rolling update to `nginx:1.28.0` |
| `11-rollback.txt` | `kubectl rollout undo` |
| `12-troubleshooting.txt` | get / describe / logs / explain / rollout |
| `13-api-not-public.txt` | UFW + API listen posture |

K3s version at capture: **v1.36.2+k3s1**
