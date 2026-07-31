# Lab notes

## Phase 3 — important commands

```bash
# Apply PSA labels + hardened workload
kubectl apply -f manifests/lab-app/
kubectl get ns lab-app --show-labels
kubectl rollout status deployment/nginx -n lab-app

# Validate hardening
kubectl exec -n lab-app deploy/nginx -- id
kubectl exec -n lab-app deploy/nginx -- cat /proc/1/status | grep -E '^(Uid|Cap)'
kubectl exec -n lab-app deploy/nginx -- sh -c 'touch /ro-test 2>&1 || true'

# App still reachable (probe Pod must also satisfy Restricted)
# See evidence/phase-03/07-app-reachability.txt

# Confirm no privileged leftover
kubectl get pods -n lab-app -o jsonpath='{range .items[*]}{.metadata.name} privileged={.spec.containers[0].securityContext.privileged} hostNetwork={.spec.hostNetwork}{"\n"}{end}'

# SA token not automounted (nginx never calls the API)
kubectl get pods -n lab-app -l app=nginx -o jsonpath='{range .items[*]}{.metadata.name} automount={.spec.automountServiceAccountToken}{"\n"}{end}'
kubectl exec -n lab-app deploy/nginx -- sh -c 'ls /var/run/secrets/kubernetes.io/serviceaccount 2>&1 || true'
```

## Phase 3 decisions

- Enforce/warn/audit all set to **restricted** on `lab-app`.
- Stock `nginx:1.27.4` could not satisfy Restricted (root, caps, no seccomp, privilege-escalation defaults, port 80 / writable rootfs assumptions).
- Switched to `nginxinc/nginx-unprivileged:1.27.4` (UID 101, listen 8080) + emptyDir mounts for RO rootfs.
- Resource requests/limits retained and documented (CPU 50m/250m, memory 64Mi/128Mi).
- Set `automountServiceAccountToken: false` on the nginx Pod template — PSA does **not** prevent SA token mounting; this is a separate workload-identity control because nginx does not call the API.
- Privileged+hostPID+hostNetwork Pod demo was temporary; deleted; Restricted restored and re-proven.
- Detection / production policy (Kyverno, Gatekeeper, signing) documented only.

## Phase 3 problems / notes

- After Restricted enforce, ad-hoc `kubectl run` probe Pods are rejected unless SecurityContext is PSA-compliant.
- Unprivileged nginx image has no `wget`; use a compliant curl Pod for Service HTTP checks.
- Existing Pods are not mutated when enforce is enabled—only new creates/updates are gated (warnings noted on label apply).

## Phase 3 — final layered security posture

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

Disabling the ServiceAccount token is a separate workload-identity hardening control and is **not** enforced by Pod Security Admission.

## Troubleshooting — K3s kubeconfig permission denied

Symptom:

```text
WARN Unable to read /etc/rancher/k3s/k3s.yaml
error loading config file: permission denied
```

Cause: K3s may default `kubectl` to the root-owned kubeconfig at `/etc/rancher/k3s/k3s.yaml`. This repository’s normal `sysadmin` workflow uses the user-owned copy at `~/.kube/config`. The root-owned file should remain permission-restricted.

Do **not**:

- Loosen permissions on `/etc/rancher/k3s/k3s.yaml`
- Routinely use `sudo kubectl`

Session-level fix:

```bash
export KUBECONFIG="$HOME/.kube/config"
kubectl get nodes
```

Optional persistent host-level fix: add the same `export` to `~/.bashrc`. `.bashrc` is host configuration and must **not** be added to or committed in this repository.

Validated:

```text
/home/sysadmin/.kube/config
Node k8s-security-lab returned Ready
```

## Phase 2 — important commands

```bash
# Apply least-privilege RBAC
kubectl apply -f manifests/rbac/
kubectl get sa,role,rolebinding -n lab-app
kubectl describe role security-reader -n lab-app
kubectl describe rolebinding security-reader -n lab-app

# Authorization tests (impersonate workload SA)
AS='--as=system:serviceaccount:lab-app:security-reader'
kubectl auth can-i get pods -n lab-app $AS
kubectl auth can-i get services -n lab-app $AS
kubectl auth can-i get deployments.apps -n lab-app $AS
kubectl auth can-i delete pods -n lab-app $AS          # no
kubectl auth can-i create deployments.apps -n lab-app $AS  # no
kubectl auth can-i get secrets -n lab-app $AS          # no
kubectl auth can-i list nodes $AS                      # no

# Confirm unsafe demo binding is absent
kubectl get clusterrolebinding security-reader-unsafe-demo 2>&1 || true
```

## Phase 2 decisions

- Dedicated SA `security-reader` instead of granting to `default`.
- Role limited to get/list/watch on pods, services, deployments in `lab-app` only.
- Deployments rules use `apiGroups: ["apps"]`.
- Unsafe `cluster-admin` ClusterRoleBinding used only as a live teaching demo; deleted immediately; not stored under `manifests/`.
- Detection/audit guidance documented only (implementation in Phase 4+).

## Phase 2 problems / notes

- `kubectl auth can-i list nodes` prints a warning that nodes are not namespace-scoped; result is still correctly `no` for the SA.
- Impersonation requires the calling kubeconfig user to have `impersonate` permission (default K3s admin does).

## Phase 1 — important commands

```bash
# Cluster
sudo systemctl status k3s
kubectl get nodes
kubectl version
kubectl cluster-info

# Workload
kubectl apply -f manifests/lab-app/
kubectl get all -n lab-app
kubectl describe deployment nginx -n lab-app
kubectl logs -n lab-app -l app=nginx --tail=50
kubectl explain deployment.spec.replicas

# Desired-state exercises
kubectl scale deployment/nginx -n lab-app --replicas=3
kubectl scale deployment/nginx -n lab-app --replicas=1
kubectl delete pod -n lab-app -l app=nginx   # RS recreates
kubectl set image deployment/nginx nginx=nginx:1.28.0 -n lab-app
kubectl rollout status deployment/nginx -n lab-app
kubectl rollout undo deployment/nginx -n lab-app
kubectl rollout history deployment/nginx -n lab-app
```

## Phase 1 decisions

- Official K3s stable install (`v1.36.2+k3s1` at execution time).
- Leave UFW without 6443 allow; API reachable only from the host / SSH session.
- ClusterIP Service only — no Ingress / LB / NodePort for nginx.
- Manifest filenames prefixed (`00-`, `10-`, `20-`) so `kubectl apply -f dir` creates Namespace first.
- Baseline image in Git: `nginx:1.27.4`; exercises temporarily used `1.28.0` then rolled back.
- kubeconfig lives in `~/.kube/config` (from `/etc/rancher/k3s/k3s.yaml`); never commit.

## Phase 1 problems / notes

- First `kubectl apply -f manifests/lab-app/` failed when files were named `deployment.yaml` / `namespace.yaml` / `service.yaml` because alphabetical order applied Deployment before Namespace existed. Fixed with numeric prefixes.
- UFW `Default: routed` showed **deny** after K3s (was disabled/routed in Phase 0 capture). Inbound still deny-except-SSH; 6443 remains unallowed.
- Temporary passwordless sudo was removed after Phase 3 validation.

## Phase 0 — important commands (unchanged)

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication)'
sudo ufw status verbose
sudo fail2ban-client status sshd
timedatectl
systemctl is-active unattended-upgrades
```

## Questions for later phases

- How should admin API access be limited (SSH tunnel only vs UFW allow from `<TRUSTED_ADMIN_IP>`)?
- Which NetworkPolicies should deny egress/ingress except probes and intentional clients?
- When to enable audit logging and runtime detection without drowning in noise on a tiny lab node?
- How to replace standing cluster-admin kubeconfig with break-glass + SSO for humans?
