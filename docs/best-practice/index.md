# MCP Best Practices

This page summarizes high‑level, vendor‑neutral principles for building, securing, and operating MCP servers. It focuses on what to do (and why), not code or SDK details.

## Design Principles

- Single responsibility: one clear domain and authentication boundary per server.
- Bounded toolsets: focused tools with specific contracts; avoid “kitchen‑sink” servers.
- Contracts first: strict input/output schemas, explicit side effects, documented errors.
- Additive change: version schemas; prefer additive evolution; publish deprecations.

## Architecture & Deployment

- Transport: `stdio` for local per‑user; Streamable HTTP for shared remote services (hardened like any API).
- Gateway: centralize authN/Z, routing, catalogs, quotas, and policy enforcement across many servers/tenants.
- Stateless by default: externalize state with clear TTLs and PII handling.
- Long‑running work: async handles, status tools, idempotency keys, compensations.

## Security

- Least privilege: per tool and action (read vs. write); approvals for high‑impact ops.
- Input/output safety: validate inputs; sanitize outputs to avoid downstream injection.
- Secrets & transport: managed secrets; TLS everywhere; signed artifacts; restricted egress/ingress.
- Policy as code: centralized allow/deny, environment gating, approvals, sensitive‑data rules.

## Observability & Governance

- Structured audit: who/what/when/why; arguments redacted; outcomes captured.
- Metrics that matter: tool success rate, latency, error classes, approvals, policy denials, cost/quotas.
- Catalogs: ownership, versions, capabilities, security review dates, compliance tags.
- SLOs & runbooks: success rates, p95 latency, error budgets; on‑call playbooks and escalation.

## Versioning & Promotion

- Pin SDKs/deps/schemas/configs; record environment/decoding profiles.
- SBOMs & signatures for servers and dependencies; store with releases.
- Promotion gates: tests and budget checks; canary/shadow with rollback; deprecations.

## Operations & Tenancy

- Environments: separate dev/stage/prod credentials, routes, policies, quotas; avoid cross‑env leakage.
- Tenancy: per‑tenant configs/logs/metrics/limits; tenant‑aware authZ; discovery reflects tenant/env differences.
- Quotas & budgets: per‑tenant/tool limits; alerting; auto‑degrade non‑critical work when over budget.
- Dependency protection: bounded concurrency, backoff with jitter; strict validation of third‑party responses.

## Testing & Quality

- Cross‑model validation; golden sets for tool contracts; CI gates for static analysis and contracts.
- Evals: measure success rate, latency, correctness; track regressions across versions.

See also: Overview, Develop, Test, Package, Deploy, Operate, Secure, Use.
