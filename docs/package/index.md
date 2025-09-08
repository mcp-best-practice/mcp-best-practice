# Package

Package MCP servers for reliable distribution and deployment. Emphasize supply chain integrity and operational readiness.

## Packaging Strategies
- Language packages (PyPI/NPM/etc.) can support developer distribution; for production, prefer containers.
- Containers: OCI images via multi-stage with minimal base images; run as non-root.
- Binary distributions: only when required; ensure signing and update channels.

## Package Structure
- Recommended layout: src (code), tests, docs, configs, scripts, Containerfile, README, CHANGELOG.

## Production Containers (Guidance)
- Use minimal, well-maintained images; drop unnecessary capabilities; read-only file systems.
- Sign images; maintain SBOMs and provenance; enforce verification in clusters.
- Externally managed secrets; health and readiness probes; resource requests/limits.

## Supply Chain Controls
- Verified registries and curated catalogs; block untrusted sources.
- Reproducible builds where feasible; store artifacts with signatures.

## Distribution
- Publish through trusted channels; include release notes and impact levels.
- Keep compatibility/support matrices; document upgrade paths and deprecations.

## Versioning Strategy
- Semantic versioning for servers, SDKs, and contracts; document breaking changes clearly.
- Maintain a simple compatibility matrix and update with each release.

## Next Steps
- Validate packaging against organizational policies (signing, SBOMs, provenance).
- Prefer containers for production distribution; enforce non-root and minimal images.
