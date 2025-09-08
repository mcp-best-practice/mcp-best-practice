# Operate (Lifecycle & Management)

Operate MCP servers with consistent lifecycle management, observability, and governance. This page combines operations and management into one high‑level guide.

## Operational Excellence
- Observability: standardized logs, metrics, and tracing; correlate via request IDs and gateway.
- Automation: automate rollouts, scaling, and remediation; codify runbooks.
- Reliability: design for failure; use circuit breakers and graceful degradation.
- Performance: set SLOs; monitor and optimize continuously.
- Security: defense in depth at runtime (policies, approvals, least privilege).

## Operational Readiness (Pre‑Prod)
- Dashboards and alerting in place; on‑call rotation and incident response defined.
- Logging, tracing, and metrics standardized (OpenTelemetry); correlation IDs across components.
- Backups and recovery tested for any externalized state.
- Capacity planning done; load/burst and dependency failure modes tested.

## Monitoring & Observability
- Four signals: latency, traffic, errors, saturation; plus policy denials and approval deferrals.
- Key metrics: tool success rate, p95 latency, error classes, cost/quotas, dependency health.
- Health/readiness endpoints; dependency checks with timeouts and fallbacks.

## Lifecycle Management
- Curated catalog: only deploy approved servers; require SBOMs, provenance, signatures, and scans.
- Approval workflow: registration, promotion, ownership, SLOs recorded; audit trails by default.
- Multi‑tenancy: per‑tenant isolation for configs, logs, metrics, and limits; tenant‑aware authZ.
- Remote/external servers: govern through a gateway/proxy where possible; document constraints and SLAs.

## Tooling & Automation
- One operator/controller framework for all servers; avoid bespoke operators per team.
- Use Helm or operator frameworks but standardize org‑wide; version your ops configs.
- Runbooks: dependency outages, rate‑limit responses, rollback steps, and escalation mapped.

## SLOs and Incident Response
- Example baselines to tune: tool success rate ≥ 99.0% monthly; p95 ≤ 400 ms.
- Taxonomy: availability, latency, correctness (contract failures), security, quota/budget.
- Flow: detect → triage (blast radius) → mitigate (degrade/circuit break) → RCA → fix/rollback → catalog learnings.
