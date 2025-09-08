# Develop

Build MCP servers using clear contracts and disciplined patterns. Keep implementations simple and observable; avoid framework lock-in where possible.

## Principles

- Single responsibility: one domain and authentication boundary per server.
- Contracts first: strict input/output schemas, explicit side effects, documented errors.
- Additive change: version contracts; prefer backward-compatible evolution with deprecations.
- Stateless by default: externalize state with TTLs and PII handling; provide async status for long-running work.

## Workflow

- Define outcomes and success metrics before coding; write tool/resource contracts first.
- Instrument from day one (structured logs, metrics, traces); add correlation IDs.
- Validate inputs rigorously; enforce least-privilege for tools and resources.
- Integrate with secrets managers; never embed credentials in code or configs.

## Quality

- Unit tests for tool logic and contract enforcement; integration tests for protocol flows and discovery.
- Track regressions with evals on key tasks; compare candidate vs. baseline before promotion.
- Keep a changelog and release notes with impact levels (breaking, behavior-shifting, non-functional).

## Handover

- Document server purpose, boundaries, capabilities, contracts, and error catalog.
- Provide operational notes: SLOs, health/readiness, limits/quotas, and runbooks.
- Publish version identifiers and SBOMs; store signed artifacts in trusted catalogs.

## See Also

Test, Package, Deploy, Operate, Secure.
