# Kubernetes Security Foundations Lab

Hands-on lab that teaches **Kubernetes through security engineering**: harden the host first, then build and observe a single-node cluster with security controls as first-class requirements.

## Current status

| Phase | Status |
|-------|--------|
| **Phase 0** — Host hardening | **Complete** |
| **Phase 1** — K3s + Kubernetes foundations | **Complete** |
| Phase 2+ — RBAC, Pod Security, NetworkPolicy, audit, detection | Not started |

Phase 1 installed single-node **K3s (v1.36.2+k3s1)**, deployed a declarative nginx workload in namespace `lab-app`, and validated scaling, reconciliation, rolling updates, and rollbacks. The Kubernetes API is **not** allowed through UFW (SSH only).

## Planned phases

| Phase | Focus |
|-------|--------|
| **0** | Host preparation and hardening |
| **1** | Single-node K3s install and core workload fundamentals |
| **2** | Cluster hardening and least-privilege access (RBAC) |
| **3** | Network policy and workload isolation |
| **4** | Observability, detection, and audit |
| **5** | Threat scenarios and remediation practice |

Exact later-phase names may evolve.

## Repository structure

```text
kubernetes-security-foundations-lab/
├── README.md
├── PROJECT_CONTEXT.md
├── .gitignore
├── LICENSE
├── docs/
│   ├── phase-00-host-hardening.md
│   ├── phase-01-kubernetes-foundations.md
│   └── lab-notes.md
├── manifests/
│   └── lab-app/
│       ├── 00-namespace.yaml
│       ├── 10-deployment.yaml
│       └── 20-service.yaml
└── evidence/
    ├── phase-00/
    └── phase-01/
```

## Lab safety notice

This environment is a **temporary DigitalOcean VPS**. Changes to SSH and the host firewall can lock you out.

- Keep an existing session open when changing SSH or UFW.
- Validate a second SSH session before reloading `sshd`.
- Prefer reloads over restarts when supported.
- Do not open Kubernetes API (6443) or application NodePorts/Ingress unless a later phase explicitly requires it and documents the risk.
- Never commit secrets, keys, tokens, kubeconfig, or raw public IPs into this repository.

## Disclaimer

This is an **educational single-node lab**, not a production reference architecture. Controls are intentionally modest and incomplete relative to a multi-node, multi-tenant production cluster. Treat findings as learning artifacts, not certification of production readiness.
