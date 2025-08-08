# Packaging Guide

## Packaging and Distribution for MCP Servers

Proper packaging ensures MCP servers can be easily installed, deployed, and maintained across different environments.

## Packaging Strategies

### Language-Specific Packages
- **Python**: PyPI packages with pip/uv
- **JavaScript**: NPM packages
- **Go**: Go modules
- **Rust**: Cargo crates

### Container Images
- Docker/Podman containers
- OCI-compliant images
- Multi-stage builds
- Minimal base images

### Binary Distributions
- Platform-specific executables
- Static binaries
- Cross-compilation

## Package Structure

### Standard Layout
```
my-mcp-server/
├── src/                 # Source code
├── tests/              # Test files
├── docs/               # Documentation
├── configs/            # Configuration files
├── scripts/            # Build/deploy scripts
├── Containerfile       # Container definition
├── Makefile           # Build automation
├── pyproject.toml     # Python package config
├── package.json       # Node.js package config
├── go.mod             # Go module config
├── LICENSE            # License file
├── README.md          # Documentation
└── CHANGELOG.md       # Version history
```

## Python Packaging

### pyproject.toml Configuration
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mcp-server-example"
version = "1.0.0"
description = "Example MCP Server"
readme = "README.md"
license = "MIT"
authors = [
    {name = "Your Name", email = "you@example.com"}
]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.11",
]
dependencies = [
    "mcp[cli]>=1.0.0",
    "pydantic>=2.0",
    "aiohttp>=3.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "ruff>=0.1",
    "mypy>=1.0",
]

[project.scripts]
mcp-example = "mcp_server_example.main:cli"

[project.entry-points."mcp.server"]
example = "mcp_server_example:server"
```

### Building Python Package
```bash
# Install build tools
pip install build twine

# Build package
python -m build

# Check package
twine check dist/*

# Upload to PyPI
twine upload dist/*
```

## JavaScript/NPM Packaging

### package.json Configuration
```json
{
  "name": "@yourorg/mcp-server-example",
  "version": "1.0.0",
  "description": "Example MCP Server",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc",
    "prepublishOnly": "npm run build && npm test",
    "test": "jest"
  },
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "keywords": ["mcp", "server", "ai"],
  "author": "Your Name",
  "license": "MIT",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "jest": "^29.0.0"
  },
  "engines": {
    "node": ">=18.0.0"
  },
  "publishConfig": {
    "access": "public",
    "registry": "https://registry.npmjs.org/"
  }
}
```

### Publishing to NPM
```bash
# Login to NPM
npm login

# Build and test
npm run build
npm test

# Publish
npm publish --access public
```

## Container Packaging

### Multi-Stage Dockerfile
```dockerfile
# Build stage
FROM python:3.11-slim as builder

WORKDIR /build

# Install build dependencies
RUN pip install --no-cache-dir build

# Copy source
COPY pyproject.toml README.md ./
COPY src ./src

# Build wheel
RUN python -m build --wheel

# Runtime stage
FROM python:3.11-slim

WORKDIR /app

# Install runtime dependencies
COPY --from=builder /build/dist/*.whl /tmp/
RUN pip install --no-cache-dir /tmp/*.whl && rm /tmp/*.whl

# Create non-root user
RUN useradd -m -u 1000 mcp && chown -R mcp:mcp /app
USER mcp

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import requests; requests.get('http://localhost:8000/health')"

EXPOSE 8000

CMD ["python", "-m", "mcp_server_example"]
```

### Building Container Images
```bash
# Build with Docker
docker build -t mcp-server-example:latest .

# Build with Podman
podman build -t mcp-server-example:latest .

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 \
  -t mcp-server-example:latest .
```

## Binary Distribution

### Go Binary Building
```bash
# Build for current platform
go build -o mcp-server cmd/server/main.go

# Cross-compilation
GOOS=linux GOARCH=amd64 go build -o mcp-server-linux-amd64
GOOS=darwin GOARCH=arm64 go build -o mcp-server-darwin-arm64
GOOS=windows GOARCH=amd64 go build -o mcp-server-windows.exe

# Static binary
CGO_ENABLED=0 go build -ldflags="-s -w" -o mcp-server
```

### Rust Binary Building
```bash
# Build release binary
cargo build --release

# Cross-compilation with cross
cross build --target x86_64-unknown-linux-musl --release

# Optimize for size
cargo build --release --features minimal
strip target/release/mcp-server
```

## Package Metadata

### Version Information
```python
# src/mcp_server_example/__version__.py
__version__ = "1.0.0"
__author__ = "Your Name"
__email__ = "you@example.com"
__description__ = "Example MCP Server"

def get_version_info():
    return {
        "version": __version__,
        "author": __author__,
        "description": __description__,
        "build_date": BUILD_DATE,
        "git_commit": GIT_COMMIT
    }
```

### License Selection
```
Common Open Source Licenses:
- MIT: Simple, permissive
- Apache 2.0: Patent protection
- GPL v3: Copyleft
- BSD 3-Clause: Similar to MIT
- MPL 2.0: File-level copyleft
```

## Distribution Channels

### Package Registries
- **PyPI**: Python packages
- **NPM**: JavaScript packages
- **Docker Hub**: Container images
- **GitHub Packages**: Multi-format registry
- **Artifactory**: Enterprise registry

### GitHub Releases
```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Build packages
      run: |
        make build
        make package
    
    - name: Create Release
      uses: softprops/action-gh-release@v1
      with:
        files: |
          dist/*
          binaries/*
        generate_release_notes: true
```

## Package Security

### Signing Packages
```bash
# GPG signing for Python packages
gpg --detach-sign -a dist/package-1.0.0.tar.gz

# NPM package signing
npm pack --sign

# Container image signing with cosign
cosign sign docker.io/myorg/mcp-server:latest
```

### SBOM Generation
```bash
# Generate SBOM with syft
syft packages dir:. -o spdx-json > sbom.json

# Include in container
COPY sbom.json /usr/share/doc/sbom.json
```

## Installation Methods

### Quick Install Scripts
```bash
#!/bin/bash
# install.sh

set -e

# Detect OS and architecture
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)

# Download appropriate binary
URL="https://github.com/org/mcp-server/releases/latest/download/mcp-server-${OS}-${ARCH}"
curl -L "$URL" -o mcp-server

# Make executable and install
chmod +x mcp-server
sudo mv mcp-server /usr/local/bin/

echo "MCP Server installed successfully!"
```

### Package Managers
```bash
# Homebrew (macOS/Linux)
brew tap yourorg/mcp
brew install mcp-server

# APT (Debian/Ubuntu)
sudo add-apt-repository ppa:yourorg/mcp
sudo apt update
sudo apt install mcp-server

# YUM/DNF (RHEL/Fedora)
sudo dnf config-manager --add-repo https://repo.example.com/mcp.repo
sudo dnf install mcp-server
```

## Versioning Strategy

### Semantic Versioning
```
1.0.0 - Initial stable release
1.0.1 - Bug fixes only
1.1.0 - New features (backward compatible)
2.0.0 - Breaking changes

Pre-release versions:
1.0.0-alpha.1
1.0.0-beta.1
1.0.0-rc.1
```

### Compatibility Matrix
```markdown
| MCP Server | MCP SDK | Python | Node.js |
|------------|---------|--------|---------|
| 1.0.x      | >=1.0   | >=3.10 | >=16    |
| 1.1.x      | >=1.1   | >=3.11 | >=18    |
| 2.0.x      | >=2.0   | >=3.11 | >=20    |
```

## Next Steps

- 🐍 [Python Packaging Details](python.md)
- 📦 [NPM Packaging](npm.md)
- 🐳 [Container Packaging](containers.md)
- 📚 [Distribution Strategies](distribution.md)