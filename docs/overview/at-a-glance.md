# MCP at a Glance

A concise, high‑level summary for teams adopting the Model Context Protocol (MCP). Use this page as your orientation before diving into detailed sections.

## What MCP Is

- A standard client‑server protocol to expose capabilities as tools, resources, and prompts.
- Transports: `stdio` for local per‑user use; Streamable HTTP for shared, remote services.
- Optional gateway centralizes authN/Z, routing, catalogs, quotas, and policy enforcement across many servers.

## When to Use MCP

- You want a consistent, discoverable interface for tools and context across apps.
- You need security, auditability, and policy control for integrations.
- You operate multiple servers, tenants, or environments and need central governance.

If you just need a single bespoke integration with minimal control, direct integration might be simpler.

## Server Responsibilities

- Expose narrowly scoped tools with typed input/output schemas and explicit side effects.
- Provide resources for read‑only context; publish prompts for standardized interactions.
- Implement authentication/authorization, logging, and metrics.
- Keep execution stateless by default; externalize state with clear TTLs and PII handling.

## Transport Choice

- `stdio`: local, per‑user, strong isolation, minimal ops; limited for centralized operations.
- Streamable HTTP: shared, remote, scalable; requires full production hardening (TLS, auth, SLOs).

## Gateway Pattern (When Needed)

- Identity and scope brokering, routing/health, quotas/rate limits.
- Policy as code (allow/deny, environment gating, approvals, sensitive‑data rules).
- Catalog/registry, audit correlation, emergency kill switches.

## Tool Design Basics

- Specific and actionable descriptions; document constraints and side effects.
- Version tool schemas; prefer additive changes; publish deprecation timelines.
- Read‑only by default; enable write tools behind explicit roles and policies.
- Long‑running work: async handles and status; idempotency keys; compensating actions.

## Security Basics

- Least privilege per tool/action; fine‑grained authorization.
- Validate inputs/outputs; sanitize outputs to avoid downstream injection.
- Secrets in a manager; TLS everywhere; signed artifacts; restricted egress/ingress.

## Observability Essentials

- Structured audit logs: who/what/when/why; arguments redacted; outcomes captured.
- Metrics: success rate, latency, error classes, approvals, policy denials, cost/quotas.
- Traces where applicable; correlate via gateway or request IDs.

## Versioning & Promotion

- Pin SDKs, dependencies, tool schemas, and configurations; record environment/decoding.
- SBOMs and signatures for servers and dependencies.
- Canary/shadow with rollback; publish release notes and impact levels.

## SLOs and Runbooks

- Define p95 latency, success rate, and error budgets per critical tool.
- Runbooks for dependency failures, rate‑limit responses, and rollbacks; on‑call routing.

## Quick Checklists

- Readiness: authN/Z, schemas, redaction, logs/metrics, health endpoints, rate limits, runbooks.
- Security: secrets, TLS, least privilege, approvals for risky operations, egress policy.
- Operations: scaling plan, budgets/quotas, caching/batching, circuit breakers, load tests.

See also: Architecture, Core Concepts, Standards, and Getting Started.
