# MCP Server Anatomy

Understand the core elements MCP servers expose and how to design them for clarity, safety, and maintainability.

## Capabilities

- Tools: Executable actions with typed input/output schemas, explicit constraints, and disclosed side effects.
- Resources: Read‑only data/context providers (files, records, documents) with stable identifiers and access controls.
- Prompts: Predefined interaction templates that standardize tasks and reduce prompt drift.

## Tool Contracts (High‑Level)

- Schemas: Define strict input/output contracts (types, ranges, formats) and provide examples in documentation.
- Errors: Enumerate error classes with machine‑actionable codes and human‑readable messages.
- Side effects: Declare mutations (create/update/delete) and external impacts; require idempotency for create/update.
- Authorization: Specify scopes/roles required per tool; require approvals for high‑impact operations.

## Discovery and Versioning

- Discovery: Provide names, descriptions, schemas, and versions for tools, resources, and prompts.
- Versioning: Use semantic versions and content hashes for contracts; prefer additive changes; publish deprecation timelines.
- Feature flags: Surface optional capabilities and environment/tenant differences in discovery responses.

## State and Long‑Running Work

- Stateless by default: Externalize state (cache/DB) with clear TTLs and PII handling; avoid hidden server memory.
- Asynchronous patterns: For long operations, return a handle and provide status/list tools; support callbacks/webhooks where appropriate.
- Compensations: Offer rollback tools for reversible side effects; log irreversible actions with approvals and change tickets.

## Documentation Essentials

- Purpose: One sentence describing the server’s domain and boundaries.
- Capabilities: List tools/resources/prompts with concise, specific descriptions and constraints.
- Contracts: Link to input/output definitions and error catalogs; include examples of typical calls and responses.
- Policies: State required scopes, approvals, rate limits, and environment/tenant differences.
- Operations: Provide SLOs, health endpoints, runbooks, and support contacts.
