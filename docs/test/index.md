# Test

Validate MCP servers for correctness, safety, and performance. Focus on behavior and contracts.

## Testing Pyramid (Guidance)
- Unit (majority): fast, isolated validations of tool logic and contracts.
- Integration: protocol flows, discovery, and interactions with dependencies.
- End-to-end: gateway + host/client workflows with real configurations.

## What to Test
- Critical paths: tool execution, resource access, error handling, input validation, authN/Z.
- Edge cases: empty/large inputs, invalid types, concurrency, dependency/network failures.
- Non-functional: latency and throughput targets, quotas, and backpressure.

## Coverage Targets (Tune to Context)
- Overall coverage around 80%+; 100% for critical paths; high coverage for error handling and edge cases.

## Test Data and Dependencies
- Use realistic fixtures; avoid brittle tests.
- Mock external services where appropriate; include selective live-integration tests for critical dependencies.

## Continuous Integration
- Lint, test, and measure coverage on every change; publish artifacts and test reports.
- Track regressions via dashboards; block promotion if SLOs or quality gates fail.

## Pre-Release Checklist
- Unit/integration/E2E tests pass; security scans clean; performance benchmarks within SLOs.
- Documentation current; manual smoke tests done; rollback plan defined.
