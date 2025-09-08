# Deploy

Deploy MCP servers behind a gateway on hardened container platforms with strong isolation and guardrails.

## Recommended Topology
- MCP Gateway as single pane of glass: centralized authN/Z, routing, quotas, catalogs, policy enforcement, and audit correlation.
- Separate environments (dev/stage/prod) with distinct credentials, routes, policies, and quotas; prevent cross‑environment leakage.

## Platform Guidance
- Kubernetes/OpenShift with:
  - Hardened images (minimal base, non‑root), signed artifacts, SBOMs, and vulnerability scanning.
  - Runtime sandboxing (gVisor, Kata Containers) and OS controls (seccomp, SELinux/AppArmor, cgroups).
  - Network policies (least privilege egress/ingress), mTLS service‑to‑service, and managed secrets.
  - Autoscaling based on SLOs; quotas per tenant; resource requests/limits enforced.

## Rollout Strategies
- Canary and blue‑green with fast rollback and kill switches; monitor p95 latency, error rates, and policy denials during rollout.
- Shadow traffic where feasible to compare behavior before promotion.

## Packaging and Supply Chain
- Package MCP servers as containers for production; sign and verify images; maintain SBOMs and provenance; use curated catalogs and verified registries only.

## Pre‑Deployment Readiness
- Tests passing; security scans complete; dependencies pinned and current; version tagged.
- Monitoring and alerting wired; dashboards ready; health/readiness endpoints validated.
- Configuration and secrets stored securely; network and DNS configured; change plan and rollback steps approved.
