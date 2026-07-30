# Kubernetes Security Foundations Lab

Hands-on lab that teaches **Kubernetes through security engineering**: harden the host first, then build and observe a single-node cluster with security controls as first-class requirements.

## Current status

**Phase 0 complete** — Ubuntu host inspected, patched, and hardened (SSH, UFW, Fail2ban, time sync, unattended security updates). Documentation and a sanitized baseline are in this repository.

**Kubernetes has not been installed.** K3s, kubectl, container runtimes, and cluster workloads are out of scope until Phase 1.

## Planned phases

| Phase | Focus |
|-------|--------|
| **0** | Host preparation and hardening (this phase) |
| **1** | Single-node K3s install and baseline cluster access |
| **2** | Cluster hardening and least-privilege access |
| **3** | Network policy and workload isolation |
| **4** | Observability, detection, and audit |
| **5** | Threat scenarios and remediation practice |

Exact later-phase names may evolve; Phase 0 deliberately stops before any cluster software.

## Repository structure

```text
kubernetes-security-foundations-lab/
├── README.md
├── PROJECT_CONTEXT.md
├── .gitignore
├── LICENSE
├── docs/
│   ├── phase-00-host-hardening.md
│   └── lab-notes.md
└── evidence/
    └── phase-00/
        └── README.md
```

## Lab safety notice

This environment is a **temporary DigitalOcean VPS**. Changes to SSH and the host firewall can lock you out.

- Keep an existing session open when changing SSH or UFW.
- Validate a second SSH session before reloading `sshd`.
- Prefer reloads over restarts when supported.
- Do not open Kubernetes or application ports during Phase 0.
- Never commit secrets, keys, tokens, or raw public IPs into this repository.

## Disclaimer

This is an **educational single-node lab**, not a production reference architecture. Controls are intentionally modest and incomplete relative to a multi-node, multi-tenant production cluster. Treat findings as learning artifacts, not certification of production readiness.
