# MCP Architecture, Security, and Operations Best Practices

This page summarizes practical, high‑value guidelines for building, securing, and operating MCP servers. It focuses on transports, gateway patterns, tool discipline, security controls, observability, and SLO‑driven operations.

## Architecture and Deployment

- Transports: use `stdio` for local per‑user integrations; use Streamable HTTP for shared, remote services and scale‑out. Harden HTTP services as you would any production API.
- Topology: prefer simple direct connections for single‑server use; introduce a gateway when you need centralized authN/Z, routing, catalogs, rate limits/quotas, and policy enforcement across many servers/tenants.
- Gateway responsibilities: identity and scope brokering, routing and health checks, rate limiting/quotas, policy enforcement, audit/metrics correlation, and emergency kill switches.

## Security Best Practices

- Identity and access: enforce least privilege per tool and per action (read vs. write). Use fine‑grained authorization and require approvals for high‑risk operations.
- Input safety: validate inputs with strict schemas, types, ranges, and patterns; reject on first failure.
- Output safety: sanitize outputs to avoid injection in downstream systems; clearly label side effects and references.
- Secrets and transport: keep credentials in a secret manager, never inline. Use TLS everywhere, sign/verify containers and artifacts, and only trust vetted registries.
- Network policy: restrict egress/ingress; apply service‑to‑service authentication and explicit allowlists.

## Tooling Discipline

- Tool design: provide specific, actionable descriptions including constraints and side effects. Keep tools bounded (avoid "kitchen‑sink"). Version tool schemas; prefer additive changes; announce deprecations.
- Toolset management: support read‑only deployments; enable/disable tools by tenant, role, environment, or risk level. Filter destructive operations out of production or gate behind approvals/policies.
- State management: keep execution stateless by default. Externalize state (cache/DB) with clear TTLs and PII handling; avoid hidden server memory.
- Long‑running operations: use async patterns for operations that exceed request timeouts; return a handle + status tools. Require idempotency keys for create/update; provide compensating (rollback) actions where feasible.
- Async and concurrency: use async I/O for API‑bound tools; bound concurrency with semaphores; use worker/process pools for CPU tasks; reuse HTTP clients with keep‑alive, timeouts, retries with jitter.

## Scalability and Reliability

- Horizontal scale: prefer concurrent, short‑lived requests; design idempotent operations and safe retries.
- Limits and backpressure: apply per‑tenant and per‑tool rate limits; surface "try later" semantics; protect upstream systems.
- Health and circuit breakers: provide health/readiness endpoints; trip on dependency failures; shed load gracefully.
- Caching and batching: cache read‑heavy operations with appropriate TTLs; batch compatible requests to reduce API usage and cost.
- Compatibility and versioning: version the server, tool schemas, and side‑effect contracts; support feature detection and safe fallbacks.
- Performance practices: pre‑warm connections, cache tokens, and reuse clients; track token/API/compute budgets per tenant and shed non‑critical work under pressure.
- Load testing: establish budgets (p50/p95/p99 latency, throughput, error budgets). Test steady‑state, burst, and dependency failure modes; run soak tests to catch leaks.

## Observability and Governance

- Audit trails: capture who/what/when/why, including tool arguments (with redaction), decisions, and outcomes using structured logs.
- Metrics that matter: track tool success rate, latency, error classes, approval deferrals, and policy denials; correlate with cost and quota consumption.
- Policy as code: centralize guardrails and enforce consistently across environments (allow/deny lists, environment gating, approval requirements, sensitive‑data rules).
- Curated catalogs: maintain an approved server catalog with ownership, versions, capabilities, security review dates, and compliance tags.
- Runbooks and SLOs: define success rate, p95 latency, and error budgets per critical tool; document common failures, dependency outages, rate‑limit responses, and rollback steps with on‑call routing and escalation.

## Versioning and Promotion

- Pin versions: pin SDKs, dependencies, tool schemas, and configuration. Record environment and decoding profiles used in validation.
- SBOMs and signing: publish SBOMs for servers and dependencies; sign artifacts; store alongside releases.
- Promotion gates: require passing tests and budget checks; support canary/shadow rollout with clear rollback plans.
- Deprecations: announce timelines; support dual‑run or feature detection where feasible.

## Testing and Quality

- Cross‑model validation: test tool discovery/execution across hosted and local models; monitor behavioral drift.
- Golden sets: maintain deterministic test fixtures for tool contracts and side‑effects; validate schema changes.
- CI quality gates: static analysis, security scanning, unit/integration tests, and contract tests on every change.

---

For a deeper dive into design principles and when to use MCP, see:
- Best Practices Guide
- Design Principles
- When to Use MCP
