# Kubernetes Security Foundations Lab

Hands-on lab that teaches **Kubernetes through security engineering**: harden the host first, then build and observe a single-node cluster with security controls as first-class requirements.

## Current status

| Phase | Status |
|-------|--------|
| **Phase 0** — Host hardening | **Complete** |
| **Phase 1** — K3s + Kubernetes foundations | **Complete** |
| **Phase 2** — RBAC & least privilege | **Complete** |
| **Phase 3** — Pod Security Admission & workload hardening | **Complete** |
| Phase 4+ — NetworkPolicy, audit, detection, threat scenarios | Not started |

Phase 3 labeled `lab-app` for **Restricted** PSA, switched nginx to an unprivileged image with full SecurityContext (non-root, drop ALL capabilities, read-only rootfs, RuntimeDefault seccomp), retained CPU/memory requests and limits, and demonstrated a temporary privileged Pod that was deleted so the lab ends secure.

## Planned phases

| Phase | Focus |
|-------|--------|
| **0** | Host preparation and hardening |
| **1** | Single-node K3s install and core workload fundamentals |
| **2** | Cluster hardening and least-privilege access (RBAC) |
| **3** | Pod Security Admission and workload hardening |
| **4** | Network policy and workload isolation |
| **5** | Observability, detection, and audit |
| **6** | Threat scenarios and remediation practice |

Exact later-phase names may evolve.

## Repository structure

```text
kubernetes-security-foundations-lab/
├── README.md
├── PROJECT_CONTEXT.md
├── PORTFOLIO.md
├── CHAT_CONTEXT.md
├── CHANGELOG.md
├── .gitignore
├── LICENSE
├── docs/
│   ├── phase-00-host-hardening.md
│   ├── phase-01-kubernetes-foundations.md
│   ├── phase-02-rbac-least-privilege.md
│   ├── phase-03-pod-security-admission.md
│   ├── interview-notes.md
│   └── lab-notes.md
├── manifests/
│   ├── lab-app/
│   │   ├── 00-namespace.yaml
│   │   ├── 10-deployment.yaml
│   │   └── 20-service.yaml
│   └── rbac/
│       ├── 00-serviceaccount.yaml
│       ├── 10-role.yaml
│       └── 20-rolebinding.yaml
└── evidence/
    ├── phase-00/
    ├── phase-01/
    ├── phase-02/
    └── phase-03/
```

## Quick start (workload + RBAC)

```bash
kubectl apply -f manifests/lab-app/
kubectl apply -f manifests/rbac/
kubectl get ns lab-app --show-labels
kubectl auth can-i get pods -n lab-app \
  --as=system:serviceaccount:lab-app:security-reader
```

## Lab safety notice

This environment is a **temporary DigitalOcean VPS**. Changes to SSH and the host firewall can lock you out.

- Keep an existing session open when changing SSH or UFW.
- Validate a second SSH session before reloading `sshd`.
- Prefer reloads over restarts when supported.
- Do not open Kubernetes API (6443) or application NodePorts/Ingress unless a later phase explicitly requires it and documents the risk.
- Never commit secrets, keys, tokens, kubeconfig, or raw public IPs into this repository.
- Never leave standing `cluster-admin` bindings on workload ServiceAccounts.
- Never leave privileged / hostNetwork / hostPID demo Pods deployed.

## Disclaimer

This is an **educational single-node lab**, not a production reference architecture. Controls are intentionally modest and incomplete relative to a multi-node, multi-tenant production cluster. Treat findings as learning artifacts, not certification of production readiness.
