# MCP Overview

## What MCP Is

The Model Context Protocol (MCP) standardizes how applications expose capabilities (tools, resources, and prompts) and how clients discover and invoke them over well‑defined transports. It creates a consistent contract between providers and consumers of capabilities so teams can scale integrations without bespoke adapters, hidden side effects, or inconsistent security models.

## Why MCP

Adopting MCP gives organizations a durable interface for connecting AI applications to systems of record and action.


- Standardization: consistent, discoverable contracts across integrations
- Security and control: built-in patterns for authN/Z and policy enforcement
- Scale and flexibility: works locally (stdio) and remotely (streamable HTTP)
- Observability: structured telemetry for discovery and invocation flows

## Core Building Blocks

At its core, MCP is a simple client–server protocol with optional gateway mediation.


- Servers: expose tools (actions), resources (read-only context), and prompts (templates)
- Clients/Hosts: discover capabilities and invoke them safely
- Gateway (optional): centralizes authN/Z, routing, quotas, catalogs, and policy
- Transports: stdio for local per-user; streamable HTTP for shared, remote services

## When to Use MCP

Choose MCP when you need repeatable, governed integrations rather than one‑off connectors.


- You need a consistent, policy-enforced interface for tools and context
- You operate many servers/tenants and require centralized governance
- You want portable integrations decoupled from specific runtime implementations

## Success Factors

- Clear server responsibility and narrow tool scopes
- Contracts-first design (schemas, side effects, errors)
- Centralized platform controls where appropriate (gateway)
- Observability and SLOs established from day one

## What MCP Is Not

- Not an agent framework: MCP defines the interface between clients and capability providers; it doesn’t prescribe planning or reasoning.
- Not a data lake or message bus: it’s a control plane for capability discovery and invocation, not a storage or streaming substrate.
- Not a silver bullet: it won’t fix unclear ownership, missing contracts, or absent governance by itself.

## Adoption Path (Typical)

1. Define a small, single‑purpose server with strict contracts.
2. Add observability and SLOs; validate with tests and evals.
3. Package as a container; sign artifacts and publish to a trusted catalog.
4. Deploy behind a gateway; enforce policy and quotas; iterate with canaries.
5. Expand capabilities gradually, keeping contracts and ownership clear.
