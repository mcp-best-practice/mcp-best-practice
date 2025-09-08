# Use

High-level guidance for how MCP servers are consumed by clients and hosts.

## Hosts and Clients
- Use supported MCP hosts (desktop, headless, IDEs) appropriate for your environment.
- Prefer gateway-mediated access for centralized authN/Z, routing, quotas, and policy.

## Integration Patterns
- Direct connection (stdio/http) for local, trusted use cases.
- Through a gateway for enterprise scenarios with many servers/tenants and shared controls.

## Authentication
- Follow the spec for supported flows; avoid bespoke schemes. Keep tokens scoped, short‑lived, and auditable.

## Operational Considerations
- Provide clear discovery; document contracts and side effects.
- Publish SLOs and error catalogs; provide support contacts and runbooks.

## Good Citizen Guidance
- Avoid long‑running client calls without async patterns.
- Respect rate limits and quotas; handle policy denials gracefully.
- Surface provenance and citations where appropriate for auditability.
