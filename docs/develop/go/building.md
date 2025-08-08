# Go Building and Deployment

## Building Go MCP Servers for Production

This guide covers building, optimizing, and deploying Go MCP servers with proper cross-compilation, optimization, and containerization.

## Build Configuration

### Basic Build Commands
```bash
# Development build
go build -o mcp-server ./cmd/server

# Production build with optimizations
go build -ldflags "-w -s" -trimpath -o mcp-server ./cmd/server

# Build with version information
VERSION=$(git describe --tags --always --dirty)
BUILD_TIME=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
go build -ldflags "-w -s -X main.Version=$VERSION -X main.BuildTime=$BUILD_TIME" -trimpath -o mcp-server ./cmd/server
```

### Advanced Build Configuration
```go
// cmd/server/version.go
package main

import (
    "fmt"
    "runtime"
)

var (
    Version   = "dev"
    BuildTime = "unknown"
    GitCommit = "unknown"
)

func printVersion() {
    fmt.Printf("MCP Server %s\n", Version)
    fmt.Printf("Build time: %s\n", BuildTime)
    fmt.Printf("Git commit: %s\n", GitCommit)
    fmt.Printf("Go version: %s\n", runtime.Version())
    fmt.Printf("OS/Arch: %s/%s\n", runtime.GOOS, runtime.GOARCH)
}
```

## Cross-Platform Compilation

### Multi-Platform Makefile
```makefile
# Build configuration
BINARY_NAME=mcp-server
MAIN_PATH=./cmd/server
VERSION ?= $(shell git describe --tags --always --dirty)
BUILD_TIME ?= $(shell date -u '+%Y-%m-%dT%H:%M:%SZ')
GIT_COMMIT ?= $(shell git rev-parse HEAD)

# Build flags
LDFLAGS=-ldflags "-w -s -X main.Version=$(VERSION) -X main.BuildTime=$(BUILD_TIME) -X main.GitCommit=$(GIT_COMMIT)"
BUILD_FLAGS=-trimpath $(LDFLAGS)

# Supported platforms
PLATFORMS := windows/amd64 linux/amd64 linux/arm64 darwin/amd64 darwin/arm64

# Default build
build:
	go build $(BUILD_FLAGS) -o $(BINARY_NAME) $(MAIN_PATH)

# Cross-compile for all platforms
build-all: $(PLATFORMS)

$(PLATFORMS):
	$(eval GOOS := $(word 1,$(subst /, ,$@)))
	$(eval GOARCH := $(word 2,$(subst /, ,$@)))
	$(eval BINARY := $(BINARY_NAME)-$(GOOS)-$(GOARCH))
	$(eval BINARY := $(if $(filter windows,$(GOOS)),$(BINARY).exe,$(BINARY)))
	
	@echo "Building $(BINARY)..."
	GOOS=$(GOOS) GOARCH=$(GOARCH) go build $(BUILD_FLAGS) -o dist/$(BINARY) $(MAIN_PATH)

# Build for current platform with race detection (development)
build-dev:
	go build -race -o $(BINARY_NAME)-dev $(MAIN_PATH)

# Static binary for containers
build-static:
	CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build $(BUILD_FLAGS) -a -installsuffix cgo -o $(BINARY_NAME)-static $(MAIN_PATH)

# Clean build artifacts
clean:
	rm -f $(BINARY_NAME)*
	rm -rf dist/

.PHONY: build build-all build-dev build-static clean $(PLATFORMS)
```

### GitHub Actions Build Pipeline
```yaml
# .github/workflows/build.yml
name: Build and Release

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Run tests
      run: |
        go test -race -coverprofile=coverage.out ./...
        go tool cover -func=coverage.out
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3

  build:
    needs: test
    runs-on: ubuntu-latest
    strategy:
      matrix:
        goos: [linux, windows, darwin]
        goarch: [amd64, arm64]
        exclude:
          - goos: windows
            goarch: arm64
    
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Set up Go
      uses: actions/setup-go@v4
      with:
        go-version: '1.21'
    
    - name: Build binary
      env:
        GOOS: ${{ matrix.goos }}
        GOARCH: ${{ matrix.goarch }}
      run: |
        VERSION=$(git describe --tags --always --dirty)
        BUILD_TIME=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
        GIT_COMMIT=$(git rev-parse HEAD)
        
        BINARY_NAME=mcp-server-${{ matrix.goos }}-${{ matrix.goarch }}
        if [ "${{ matrix.goos }}" = "windows" ]; then
          BINARY_NAME="${BINARY_NAME}.exe"
        fi
        
        CGO_ENABLED=0 go build \
          -ldflags "-w -s -X main.Version=$VERSION -X main.BuildTime=$BUILD_TIME -X main.GitCommit=$GIT_COMMIT" \
          -trimpath \
          -o $BINARY_NAME \
          ./cmd/server
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: binaries
        path: mcp-server-*

  release:
    if: startsWith(github.ref, 'refs/tags/v')
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Download artifacts
      uses: actions/download-artifact@v3
      with:
        name: binaries
    
    - name: Create release
      uses: softprops/action-gh-release@v1
      with:
        files: mcp-server-*
        generate_release_notes: true
```

## Container Building

### Multi-Stage Dockerfile
```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git ca-certificates tzdata

# Set working directory
WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Build arguments
ARG VERSION=dev
ARG BUILD_TIME
ARG GIT_COMMIT

# Build the binary
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags "-w -s -X main.Version=$VERSION -X main.BuildTime=$BUILD_TIME -X main.GitCommit=$GIT_COMMIT" \
    -trimpath \
    -o mcp-server \
    ./cmd/server

# Final stage
FROM scratch

# Copy CA certificates
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy timezone data
COPY --from=builder /usr/share/zoneinfo /usr/share/zoneinfo

# Copy the binary
COPY --from=builder /app/mcp-server /mcp-server

# Create non-root user (numeric for scratch image)
USER 10001:10001

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/mcp-server", "health"]

# Run the binary
ENTRYPOINT ["/mcp-server"]
```

### Optimized Dockerfile with UPX
```dockerfile
# Build stage with UPX compression
FROM golang:1.21-alpine AS builder

# Install dependencies including UPX
RUN apk add --no-cache git ca-certificates tzdata upx

WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

ARG VERSION=dev
ARG BUILD_TIME
ARG GIT_COMMIT

# Build with static linking
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags "-w -s -X main.Version=$VERSION -X main.BuildTime=$BUILD_TIME -X main.GitCommit=$GIT_COMMIT" \
    -trimpath \
    -a -installsuffix cgo \
    -o mcp-server \
    ./cmd/server

# Compress binary with UPX
RUN upx --best --lzma mcp-server

# Verify the compressed binary works
RUN ./mcp-server version

# Final stage
FROM scratch

COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/mcp-server /mcp-server

USER 10001:10001
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/mcp-server", "health"]

ENTRYPOINT ["/mcp-server"]
```

### Docker Compose for Development
```yaml
# docker-compose.yml
version: '3.8'

services:
  mcp-server:
    build:
      context: .
      dockerfile: Dockerfile.dev
      args:
        VERSION: dev
    ports:
      - "8000:8000"
    environment:
      - MCP_LOG_LEVEL=debug
      - MCP_TRANSPORT=http
      - MCP_HTTP_ADDR=:8000
    volumes:
      - .:/app
      - go-mod-cache:/go/pkg/mod
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mcp_dev
      POSTGRES_USER: mcp
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"

volumes:
  go-mod-cache:
  postgres_data:
  redis_data:
```

### Development Dockerfile
```dockerfile
# Dockerfile.dev - for development with hot reload
FROM golang:1.21-alpine

RUN apk add --no-cache git ca-certificates tzdata

# Install air for hot reloading
RUN go install github.com/cosmtrek/air@latest

WORKDIR /app

# Copy go mod files and download deps
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Expose port
EXPOSE 8000

# Command for development
CMD ["air", "-c", ".air.toml"]
```

## Build Optimization

### Compiler Optimizations
```bash
# Size optimization
go build -ldflags "-w -s" -trimpath

# Performance optimization  
go build -ldflags "-w -s" -trimpath -gcflags "-B"

# Link-time optimization (experimental)
go build -ldflags "-w -s" -trimpath -buildmode=pie

# Profile-guided optimization (Go 1.21+)
go build -pgo=auto -ldflags "-w -s" -trimpath
```

### Build Caching
```bash
# Enable build cache
export GOCACHE=$(go env GOCACHE)

# Module cache
export GOMODCACHE=$(go env GOMODCACHE)

# Docker build cache
docker build --build-arg BUILDKIT_INLINE_CACHE=1 .
```

## Binary Analysis

### Size Analysis Script
```bash
#!/bin/bash
# scripts/analyze-binary.sh

BINARY=${1:-mcp-server}

if [ ! -f "$BINARY" ]; then
    echo "Binary $BINARY not found"
    exit 1
fi

echo "=== Binary Analysis for $BINARY ==="
echo

echo "File size:"
ls -lh "$BINARY" | awk '{print $5}'
echo

echo "Symbols (top 20):"
go tool nm "$BINARY" | grep -E ' T | D | B ' | sort -k3 -nr | head -20
echo

echo "Build info:"
go version -m "$BINARY"
echo

echo "Dependencies:"
go tool nm "$BINARY" | grep -o 'github.com/[^/]*/[^/]*' | sort | uniq -c | sort -nr | head -10
```

### Security Scanning
```bash
#!/bin/bash
# scripts/security-scan.sh

echo "=== Security Analysis ==="

# Check for hardcoded secrets
echo "Checking for hardcoded secrets..."
truffleHog --regex --entropy=False .

# Scan dependencies for vulnerabilities
echo "Scanning dependencies..."
go list -json -m all | nancy sleuth

# Static analysis
echo "Running static analysis..."
gosec ./...

# Container security scan (if image built)
if [ "$1" = "container" ]; then
    echo "Scanning container image..."
    trivy image mcp-server:latest
fi
```

## Performance Optimization

### Build-Time Profiling
```go
// build_profile.go - build with profiling enabled
//go:build profile

package main

import (
    _ "net/http/pprof"
    "net/http"
    "log"
)

func init() {
    go func() {
        log.Println("Starting profile server on :6060")
        log.Println(http.ListenAndServe(":6060", nil))
    }()
}
```

### Memory Optimization
```bash
# Build with memory optimization
go build -gcflags "-l=4" -ldflags "-w -s" -trimpath

# Runtime memory tuning
export GOGC=100
export GOMEMLIMIT=1GiB
export GOMAXPROCS=4
```

## Deployment Scripts

### Systemd Service
```ini
# /etc/systemd/system/mcp-server.service
[Unit]
Description=MCP Server
After=network.target
Wants=network.target

[Service]
Type=simple
User=mcp
Group=mcp
WorkingDirectory=/opt/mcp-server
ExecStart=/opt/mcp-server/mcp-server
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal
SyslogIdentifier=mcp-server

# Security settings
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/mcp-server

# Environment
Environment=MCP_LOG_LEVEL=info
Environment=MCP_TRANSPORT=http
Environment=MCP_HTTP_ADDR=:8000

[Install]
WantedBy=multi-user.target
```

### Installation Script
```bash
#!/bin/bash
# scripts/install.sh

set -e

# Configuration
USER="mcp"
GROUP="mcp"
INSTALL_DIR="/opt/mcp-server"
SERVICE_NAME="mcp-server"

# Check if running as root
if [ "$EUID" -ne 0 ]; then
    echo "Please run as root"
    exit 1
fi

# Create user and group
if ! id "$USER" &>/dev/null; then
    useradd -r -s /bin/false -d "$INSTALL_DIR" "$USER"
fi

# Create installation directory
mkdir -p "$INSTALL_DIR"
mkdir -p "/var/log/$SERVICE_NAME"

# Copy binary
cp mcp-server "$INSTALL_DIR/"
chmod +x "$INSTALL_DIR/mcp-server"

# Set ownership
chown -R "$USER:$GROUP" "$INSTALL_DIR"
chown -R "$USER:$GROUP" "/var/log/$SERVICE_NAME"

# Install systemd service
cp scripts/mcp-server.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable "$SERVICE_NAME"

echo "Installation complete!"
echo "Start the service with: systemctl start $SERVICE_NAME"
echo "Check status with: systemctl status $SERVICE_NAME"
echo "View logs with: journalctl -u $SERVICE_NAME -f"
```

## Best Practices

### Build Performance
1. **Module caching**: Use persistent module cache in CI/CD
2. **Layer caching**: Optimize Docker layer caching
3. **Parallel builds**: Use `GOMAXPROCS` for faster builds
4. **Incremental builds**: Structure code to minimize rebuild scope

### Security
1. **Static linking**: Use `CGO_ENABLED=0` for static binaries
2. **Strip symbols**: Use `-ldflags "-w -s"` to reduce size
3. **Minimal containers**: Use scratch or distroless base images
4. **Vulnerability scanning**: Regularly scan dependencies and containers

### Optimization
1. **Profile-guided optimization**: Use PGO for performance-critical code
2. **Binary compression**: Consider UPX for size-constrained deployments
3. **Runtime tuning**: Configure Go runtime parameters for deployment environment
4. **Health checks**: Include health check endpoints in production builds

This comprehensive building guide ensures your Go MCP server is optimized, secure, and ready for production deployment.