# Lab notes

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
- Temporary passwordless sudo used for agent automation; remove `/etc/sudoers.d/99-lab-agent-temp` after Phase 1 validation.

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
- What ServiceAccount and Role bindings should `lab-app` use instead of default?
- Which NetworkPolicies should deny egress/ingress except probes and intentional clients?
- When to enable audit logging and runtime detection without drowning in noise on a tiny lab node?
