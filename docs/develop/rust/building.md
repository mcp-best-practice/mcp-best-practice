# Rust Building and Deployment

## Building Rust MCP Servers for Production

This guide covers building, optimizing, and deploying Rust MCP servers with proper cross-compilation, optimization, and containerization strategies.

## Build Configuration

### Cargo.toml Optimization
```toml
[package]
name = "mcp-server-rust"
version = "0.1.0"
edition = "2021"

# Build profiles
[profile.release]
# Enable link-time optimization
lto = "fat"
# Use only one codegen unit for maximum optimization
codegen-units = 1
# Abort on panic instead of unwinding (smaller binary)
panic = "abort"
# Strip debug info and symbols
strip = true
# Optimize for size
opt-level = "z"

[profile.dev]
# Enable debug info for development
debug = true
# Don't optimize for faster builds
opt-level = 0
# Enable overflow checks
overflow-checks = true

# Custom profile for size-optimized release
[profile.release-size]
inherits = "release"
opt-level = "z"
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"

# Custom profile for performance-optimized release
[profile.release-perf]
inherits = "release"
opt-level = 3
lto = "fat"
codegen-units = 1
```

### Build Scripts and Automation

```bash
#!/bin/bash
# scripts/build.sh

set -e

# Configuration
BINARY_NAME="mcp-server"
TARGET_DIR="target"
DIST_DIR="dist"

# Parse command line arguments
PROFILE="release"
TARGET=""
FEATURES=""

while [[ $# -gt 0 ]]; do
    case $1 in
        --profile)
            PROFILE="$2"
            shift 2
            ;;
        --target)
            TARGET="$2"
            shift 2
            ;;
        --features)
            FEATURES="$2"
            shift 2
            ;;
        *)
            echo "Unknown option $1"
            exit 1
            ;;
    esac
done

# Set build info
export BUILD_TIME=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
export GIT_COMMIT=$(git rev-parse HEAD 2>/dev/null || echo "unknown")
export GIT_TAG=$(git describe --tags --exact-match 2>/dev/null || echo "")
export VERSION=${GIT_TAG:-"dev"}

echo "Building ${BINARY_NAME} (profile: ${PROFILE}, version: ${VERSION})"

# Build command
CARGO_CMD="cargo build --profile ${PROFILE}"

if [[ -n "$TARGET" ]]; then
    CARGO_CMD="${CARGO_CMD} --target ${TARGET}"
fi

if [[ -n "$FEATURES" ]]; then
    CARGO_CMD="${CARGO_CMD} --features ${FEATURES}"
fi

# Execute build
eval $CARGO_CMD

# Create distribution directory
mkdir -p "$DIST_DIR"

# Copy binary to dist
if [[ -n "$TARGET" ]]; then
    BINARY_PATH="${TARGET_DIR}/${TARGET}/${PROFILE}/${BINARY_NAME}"
    DIST_BINARY="${DIST_DIR}/${BINARY_NAME}-${TARGET}"
else
    BINARY_PATH="${TARGET_DIR}/${PROFILE}/${BINARY_NAME}"
    DIST_BINARY="${DIST_DIR}/${BINARY_NAME}"
fi

# Add .exe extension for Windows
if [[ "$TARGET" == *"windows"* ]]; then
    BINARY_PATH="${BINARY_PATH}.exe"
    DIST_BINARY="${DIST_BINARY}.exe"
fi

cp "$BINARY_PATH" "$DIST_BINARY"

echo "Binary built: $DIST_BINARY"

# Print binary info
echo "Binary size: $(du -h $DIST_BINARY | cut -f1)"
echo "Binary info:"
file "$DIST_BINARY" || true
```

### Cross-Compilation Configuration

```toml
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
linker = "x86_64-linux-gnu-gcc"

[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"

[target.aarch64-unknown-linux-gnu]
linker = "aarch64-linux-gnu-gcc"

[target.x86_64-pc-windows-gnu]
linker = "x86_64-w64-mingw32-gcc"

[target.x86_64-apple-darwin]
linker = "x86_64-apple-darwin-clang"

# Environment variables for build
[env]
RUSTFLAGS = "-C target-cpu=native"
```

### Multi-Target Build Script

```bash
#!/bin/bash
# scripts/build-all.sh

set -e

# Define targets
TARGETS=(
    "x86_64-unknown-linux-gnu"
    "x86_64-unknown-linux-musl" 
    "aarch64-unknown-linux-gnu"
    "x86_64-pc-windows-gnu"
    "x86_64-apple-darwin"
    "aarch64-apple-darwin"
)

BINARY_NAME="mcp-server"
FEATURES="database,metrics"

# Install targets if not already installed
echo "Installing Rust targets..."
for target in "${TARGETS[@]}"; do
    rustup target add "$target" || true
done

# Build for each target
for target in "${TARGETS[@]}"; do
    echo "Building for $target..."
    
    # Skip targets that require special setup
    if [[ "$target" == *"windows"* ]] && ! command -v x86_64-w64-mingw32-gcc &> /dev/null; then
        echo "Skipping $target (mingw-w64 not installed)"
        continue
    fi
    
    if [[ "$target" == *"apple"* ]] && [[ "$OSTYPE" != "darwin"* ]]; then
        echo "Skipping $target (requires macOS)"
        continue
    fi
    
    # Static linking for musl
    if [[ "$target" == *"musl"* ]]; then
        export RUSTFLAGS="-C target-feature=+crt-static"
    else
        export RUSTFLAGS=""
    fi
    
    cargo build \
        --profile release-size \
        --target "$target" \
        --features "$FEATURES" \
        --locked
    
    # Copy to dist directory
    mkdir -p "dist/$target"
    
    BINARY_SRC="target/$target/release-size/$BINARY_NAME"
    BINARY_DEST="dist/$target/$BINARY_NAME"
    
    if [[ "$target" == *"windows"* ]]; then
        BINARY_SRC="${BINARY_SRC}.exe"
        BINARY_DEST="${BINARY_DEST}.exe"
    fi
    
    cp "$BINARY_SRC" "$BINARY_DEST"
    
    echo "Built: $BINARY_DEST ($(du -h $BINARY_DEST | cut -f1))"
done

echo "All builds completed!"
ls -la dist/*/
```

## Container Building

### Multi-Stage Dockerfile

```dockerfile
# Multi-stage Dockerfile for Rust MCP server
FROM rust:1.75-slim as builder

# Install build dependencies
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Create app user
RUN useradd -m -u 1001 appuser

WORKDIR /app

# Copy manifests
COPY Cargo.toml Cargo.lock ./

# Create dummy main to cache dependencies
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    echo '' > src/lib.rs

# Build dependencies (this layer will be cached)
RUN cargo build --profile release-size --locked
RUN rm src/*.rs

# Copy source code
COPY src ./src

# Build arguments for versioning
ARG VERSION=dev
ARG BUILD_TIME
ARG GIT_COMMIT

# Build the actual application
RUN touch src/main.rs && \
    cargo build --profile release-size --locked

# Runtime stage
FROM debian:bookworm-slim

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

# Create app user
RUN useradd -m -u 1001 appuser

# Copy the binary from builder stage
COPY --from=builder /app/target/release-size/mcp-server /usr/local/bin/mcp-server

# Set ownership
RUN chown appuser:appuser /usr/local/bin/mcp-server

# Switch to non-root user
USER appuser

# Create directories
WORKDIR /home/appuser
RUN mkdir -p logs data config

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD /usr/local/bin/mcp-server --version || exit 1

EXPOSE 8000

# Default command
CMD ["mcp-server", "--config", "/home/appuser/config/config.yaml"]
```

### Optimized Dockerfile with Static Binary

```dockerfile
# Static binary Dockerfile
FROM rust:1.75-alpine as builder

# Install musl development tools
RUN apk add --no-cache musl-dev pkgconfig openssl-dev openssl-libs-static

WORKDIR /app

# Copy manifests
COPY Cargo.toml Cargo.lock ./

# Copy source
COPY src ./src

# Build static binary
ENV RUSTFLAGS="-C target-feature=+crt-static"
RUN cargo build \
    --profile release-size \
    --target x86_64-unknown-linux-musl \
    --locked

# Runtime stage - using scratch for minimal image
FROM scratch

# Copy CA certificates
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# Copy the static binary
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release-size/mcp-server /mcp-server

# Create minimal filesystem
COPY --from=builder /etc/passwd /etc/passwd
COPY --from=builder /etc/group /etc/group

# Use non-root user (create entry in builder stage if needed)
USER 1001:1001

EXPOSE 8000

ENTRYPOINT ["/mcp-server"]
```

### Docker Compose for Development

```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  mcp-server:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8000:8000"
    environment:
      - RUST_LOG=debug
      - DATABASE_URL=postgres://postgres:password@postgres:5432/mcp_dev
    volumes:
      - .:/app
      - cargo-cache:/usr/local/cargo/registry
      - target-cache:/app/target
    depends_on:
      - postgres
    command: cargo watch -x 'run --bin mcp-server'

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: mcp_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  cargo-cache:
  target-cache:
  postgres_data:
```

### Development Dockerfile

```dockerfile
# Dockerfile.dev - for development with hot reload
FROM rust:1.75-slim

# Install development dependencies
RUN apt-get update && apt-get install -y \
    pkg-config \
    libssl-dev \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Install cargo-watch for hot reloading
RUN cargo install cargo-watch

WORKDIR /app

# Copy manifests
COPY Cargo.toml Cargo.lock ./

# Pre-build dependencies
RUN mkdir src && \
    echo "fn main() {}" > src/main.rs && \
    cargo build && \
    rm -rf src

# Expose port
EXPOSE 8000

# Default command for development
CMD ["cargo", "watch", "-x", "run"]
```

## GitHub Actions CI/CD

### Complete Build Pipeline

```yaml
# .github/workflows/build.yml
name: Build and Release

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always

jobs:
  test:
    name: Test Suite
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Install Rust
      uses: dtolnay/rust-toolchain@stable
      with:
        components: rustfmt, clippy
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/registry
          ~/.cargo/git
          target
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Check formatting
      run: cargo fmt -- --check
    
    - name: Run clippy
      run: cargo clippy -- -D warnings
    
    - name: Run tests
      run: cargo test --all-features

  build:
    name: Build Release
    needs: test
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            target: x86_64-unknown-linux-gnu
          - os: ubuntu-latest
            target: x86_64-unknown-linux-musl
          - os: ubuntu-latest
            target: aarch64-unknown-linux-gnu
          - os: windows-latest
            target: x86_64-pc-windows-msvc
          - os: macos-latest
            target: x86_64-apple-darwin
          - os: macos-latest
            target: aarch64-apple-darwin

    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0
    
    - name: Install Rust
      uses: dtolnay/rust-toolchain@stable
      with:
        targets: ${{ matrix.target }}
    
    - name: Install cross-compilation tools
      if: matrix.target == 'aarch64-unknown-linux-gnu'
      run: |
        sudo apt-get update
        sudo apt-get install -y gcc-aarch64-linux-gnu
    
    - name: Install musl tools
      if: matrix.target == 'x86_64-unknown-linux-musl'
      run: |
        sudo apt-get update
        sudo apt-get install -y musl-tools
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/registry
          ~/.cargo/git
          target
        key: ${{ runner.os }}-${{ matrix.target }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Build binary
      run: |
        cargo build --profile release-size --target ${{ matrix.target }} --locked
    
    - name: Package binary
      shell: bash
      run: |
        VERSION=${GITHUB_REF#refs/tags/}
        if [[ "$VERSION" == "refs/heads/main" ]]; then
          VERSION="main-$(git rev-parse --short HEAD)"
        fi
        
        BINARY_NAME="mcp-server"
        if [[ "${{ matrix.target }}" == *"windows"* ]]; then
          BINARY_NAME="${BINARY_NAME}.exe"
        fi
        
        ARCHIVE_NAME="mcp-server-${VERSION}-${{ matrix.target }}"
        
        mkdir -p dist
        cp target/${{ matrix.target }}/release-size/${BINARY_NAME} dist/
        
        cd dist
        if [[ "${{ matrix.target }}" == *"windows"* ]]; then
          7z a ${ARCHIVE_NAME}.zip ${BINARY_NAME}
        else
          tar czf ${ARCHIVE_NAME}.tar.gz ${BINARY_NAME}
        fi
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: binaries-${{ matrix.target }}
        path: dist/*

  docker:
    name: Build Docker Image
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Login to DockerHub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: your-org/mcp-server-rust
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=semver,pattern={{version}}
          type=semver,pattern={{major}}.{{minor}}
    
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        build-args: |
          VERSION=${{ steps.meta.outputs.version }}
          BUILD_TIME=${{ steps.meta.outputs.date }}
          GIT_COMMIT=${{ github.sha }}

  release:
    name: Create Release
    needs: [test, build]
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    
    steps:
    - name: Download artifacts
      uses: actions/download-artifact@v3
    
    - name: Create release
      uses: softprops/action-gh-release@v1
      with:
        files: |
          binaries-*/mcp-server-*
        generate_release_notes: true
        draft: false
        prerelease: false
```

## Binary Optimization Techniques

### Size Optimization Script

```bash
#!/bin/bash
# scripts/optimize.sh

set -e

BINARY_NAME="mcp-server"
BINARY_PATH="target/release-size/$BINARY_NAME"

echo "Original binary size:"
ls -lh "$BINARY_PATH"

# Strip additional symbols
echo "Stripping symbols..."
strip "$BINARY_PATH"

echo "Stripped binary size:"
ls -lh "$BINARY_PATH"

# Compress with UPX (optional)
if command -v upx &> /dev/null; then
    echo "Compressing with UPX..."
    cp "$BINARY_PATH" "${BINARY_PATH}.backup"
    upx --best "$BINARY_PATH"
    
    echo "Compressed binary size:"
    ls -lh "$BINARY_PATH"
    
    echo "Testing compressed binary..."
    if ! "$BINARY_PATH" --version; then
        echo "Compressed binary is corrupted, restoring backup"
        mv "${BINARY_PATH}.backup" "$BINARY_PATH"
    else
        rm -f "${BINARY_PATH}.backup"
        echo "Compression successful"
    fi
fi

# Analyze binary
echo "Binary analysis:"
file "$BINARY_PATH"

if command -v bloaty &> /dev/null; then
    echo "Size breakdown:"
    bloaty "$BINARY_PATH"
fi
```

### Performance Profiling

```bash
#!/bin/bash
# scripts/profile.sh

BINARY_NAME="mcp-server"

# Build with profiling enabled
echo "Building with profiling..."
RUSTFLAGS="-C profile-generate=/tmp/pgo-data" \
    cargo build --release --target-dir target/pgo

# Run training workload
echo "Running training workload..."
./target/pgo/release/$BINARY_NAME --config config/profile.yaml &
SERVER_PID=$!

# Generate some load
sleep 2
curl http://localhost:8000/health
# Add more representative workload here

kill $SERVER_PID
wait

# Build optimized binary
echo "Building optimized binary..."
RUSTFLAGS="-C profile-use=/tmp/pgo-data -C llvm-args=-pgo-warn-missing-function" \
    cargo build --release --target-dir target/pgo-optimized

echo "PGO optimization complete"
ls -lh target/pgo-optimized/release/$BINARY_NAME
```

This comprehensive Rust building guide ensures your MCP server is optimized, cross-platform compatible, and ready for production deployment with minimal resource usage and maximum performance.