# Development Tools

## MCP Development Tools and Utilities

Essential tools, libraries, and utilities for efficient MCP server development across different languages and platforms.

## Code Editors and IDEs

### VS Code Extensions
```json
// .vscode/extensions.json
{
  "recommendations": [
    "ms-vscode.vscode-json",
    "ms-python.python",
    "ms-python.pylint",
    "ms-python.black-formatter",
    "golang.go",
    "rust-lang.rust-analyzer",
    "bradlc.vscode-tailwindcss",
    "ms-vscode.vscode-typescript-next",
    "formulahendry.auto-rename-tag",
    "esbenp.prettier-vscode",
    "redhat.vscode-yaml"
  ]
}
```

### VS Code Settings
```json
// .vscode/settings.json
{
  "python.defaultInterpreterPath": "./venv/bin/python",
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": true,
  "python.formatting.provider": "black",
  "go.formatTool": "goimports",
  "go.lintTool": "golangci-lint",
  "rust-analyzer.checkOnSave.command": "clippy",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.organizeImports": true,
    "source.fixAll": true
  }
}
```

### IntelliJ/PyCharm Configuration
```xml
<!-- .idea/inspectionProfiles/Project_Default.xml -->
<component name="InspectionProjectProfileManager">
  <profile version="1.0">
    <option name="myName" value="Project Default" />
    <inspection_tool class="PyPackageRequirementsInspection" enabled="true" level="WARNING" enabled_by_default="true">
      <option name="ignoredPackages">
        <value>
          <list size="0" />
        </value>
      </option>
    </inspection_tool>
  </profile>
</component>
```

## Language-Specific Tools

### Python Development
```bash
# Install development tools
pip install black pylint mypy pytest pytest-cov
pip install pre-commit bandit safety

# pyproject.toml configuration
[tool.black]
line-length = 100
target-version = ['py38', 'py39', 'py310', 'py311']

[tool.pylint.messages_control]
disable = ["C0111", "R0903"]

[tool.mypy]
python_version = "3.8"
strict = true
ignore_missing_imports = true

[tool.pytest.ini_options]
minversion = "6.0"
addopts = "-ra -q --strict-markers --strict-config"
testpaths = ["tests"]
```

### Go Development Tools
```bash
# Install Go tools
go install golang.org/x/tools/cmd/goimports@latest
go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest
go install github.com/securecodewarrior/sast-scan@latest
go install honnef.co/go/tools/cmd/staticcheck@latest

# Install testing tools
go install github.com/golang/mock/mockgen@latest
go install gotest.tools/gotestsum@latest
```

### JavaScript/TypeScript Tools
```bash
# Install development dependencies
npm install -D typescript @types/node
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install -D prettier
npm install -D jest @types/jest ts-jest
npm install -D nodemon ts-node

# Package.json scripts
"scripts": {
  "dev": "nodemon src/index.ts",
  "build": "tsc",
  "test": "jest",
  "test:watch": "jest --watch",
  "lint": "eslint src/**/*.ts",
  "lint:fix": "eslint src/**/*.ts --fix",
  "format": "prettier --write src/**/*.ts"
}
```

### Rust Development
```bash
# Install Rust tools
rustup component add rustfmt clippy
cargo install cargo-watch
cargo install cargo-audit
cargo install cargo-outdated

# Cargo.toml configuration
[profile.dev]
debug = true

[profile.release]
lto = true
codegen-units = 1
panic = "abort"
```

## Testing Tools

### Unit Testing Frameworks
```python
# Python - pytest configuration
# tests/conftest.py
import pytest
from mcp_server import create_server

@pytest.fixture
def test_client():
    server = create_server(test_mode=True)
    return server.test_client()

@pytest.fixture
def mock_database():
    """Mock database for testing."""
    return MockDatabase()
```

### Integration Testing
```javascript
// JavaScript/TypeScript - Jest configuration
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src', '<rootDir>/tests'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
};
```

### Load Testing Tools
```yaml
# k6 load testing script
# scripts/load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 0 },
  ],
};

export default function() {
  let response = http.get('http://localhost:8000/health');
  check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

## Development Utilities

### Docker Development Environment
```dockerfile
# docker/dev.Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install development dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    vim \
    && rm -rf /var/lib/apt/lists/*

# Install Python dev tools
COPY requirements-dev.txt .
RUN pip install -r requirements-dev.txt

# Set up development environment
ENV PYTHONPATH=/app
ENV MCP_ENV=development

EXPOSE 8000
CMD ["python", "-m", "debugpy", "--listen", "0.0.0.0:5678", "--wait-for-client", "-m", "mcp_server"]
```

### Docker Compose for Development
```yaml
# docker-compose.dev.yml
version: '3.8'

services:
  mcp-server:
    build:
      context: .
      dockerfile: docker/dev.Dockerfile
    volumes:
      - .:/app
      - /app/__pycache__
    ports:
      - "8000:8000"
      - "5678:5678"  # debugpy
    environment:
      - MCP_LOG_LEVEL=debug
      - MCP_DATABASE_URL=postgresql://user:pass@postgres:5432/mcpdev
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: mcpdev
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

## Debugging Tools

### Python Debugging
```python
# Debug configuration
import logging
import debugpy

# Enable debugger
if os.getenv('MCP_DEBUG'):
    debugpy.listen(5678)
    debugpy.wait_for_client()

# Logging setup
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('debug.log'),
        logging.StreamHandler()
    ]
)
```

### Go Debugging with Delve
```bash
# Install delve
go install github.com/go-delve/delve/cmd/dlv@latest

# Debug with delve
dlv debug ./cmd/server -- --config config.yaml

# Remote debugging
dlv debug --headless --listen=:2345 --api-version=2 ./cmd/server
```

### JavaScript Debugging
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug MCP Server",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/src/index.ts",
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "runtimeArgs": ["--nolazy", "-r", "ts-node/register"],
      "env": {
        "NODE_ENV": "development",
        "MCP_LOG_LEVEL": "debug"
      },
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    }
  ]
}
```

## Code Quality Tools

### Pre-commit Hooks
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black

  - repo: https://github.com/PyCQA/pylint
    rev: v2.17.4
    hooks:
      - id: pylint

  - repo: https://github.com/PyCQA/bandit
    rev: 1.7.5
    hooks:
      - id: bandit
        args: ['-f', 'json', '-o', 'bandit-report.json']
```

### GitHub Actions Workflow
```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.8', '3.9', '3.10', '3.11']

    steps:
    - uses: actions/checkout@v4

    - name: Set up Python ${{ matrix.python-version }}
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements-dev.txt

    - name: Run tests
      run: |
        pytest --cov=mcp_server --cov-report=xml

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
```

## Monitoring and Profiling

### Performance Profiling
```python
# Python profiling
import cProfile
import pstats
from pstats import SortKey

def profile_tool_execution(tool_name, arguments):
    profiler = cProfile.Profile()
    profiler.enable()
    
    # Execute tool
    result = execute_tool(tool_name, arguments)
    
    profiler.disable()
    stats = pstats.Stats(profiler)
    stats.sort_stats(SortKey.TIME)
    stats.print_stats(10)
    
    return result
```

### Memory Profiling
```python
# Python memory profiling with memory_profiler
from memory_profiler import profile

@profile
def memory_intensive_tool():
    # Tool implementation
    pass
```

### Go Profiling
```go
// Go profiling
import (
    _ "net/http/pprof"
    "net/http"
    "log"
)

func init() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
}
```

## Database Tools

### Database Migrations
```python
# Python - Alembic migration
from alembic import op
import sqlalchemy as sa

def upgrade():
    op.create_table(
        'tool_executions',
        sa.Column('id', sa.Integer, primary_key=True),
        sa.Column('tool_name', sa.String(100), nullable=False),
        sa.Column('arguments', sa.JSON),
        sa.Column('result', sa.JSON),
        sa.Column('executed_at', sa.DateTime, default=sa.func.now())
    )

def downgrade():
    op.drop_table('tool_executions')
```

### Database Seeding
```sql
-- SQL seed data
INSERT INTO tool_configurations (name, config) VALUES
('github', '{"token": "${GITHUB_TOKEN}", "base_url": "https://api.github.com"}'),
('database', '{"connection_string": "${DATABASE_URL}", "read_only": true}'),
('filesystem', '{"allowed_paths": ["/tmp", "/var/data"], "max_file_size": 1048576}');
```

## Documentation Tools

### API Documentation Generation
```python
# Python - Generate OpenAPI spec
from apispec import APISpec
from apispec.ext.marshmallow import MarshmallowPlugin

spec = APISpec(
    title="MCP Server API",
    version="1.0.0",
    openapi_version="3.0.2",
    plugins=[MarshmallowPlugin()],
)

# Add tool schemas
for tool in registered_tools:
    spec.components.schema(tool.name, schema=tool.input_schema)
```

### Code Documentation
```go
// Go documentation with godoc
// Package tools provides MCP tool implementations.
//
// This package contains various tool implementations for the MCP server,
// including database queries, HTTP requests, and file system operations.
//
// Example usage:
//   registry := tools.NewRegistry()
//   registry.Register(&tools.DatabaseTool{})
//   result, err := registry.CallTool(ctx, "database", args)
package tools
```

## Security Tools

### Static Analysis
```bash
# Security scanning tools
pip install bandit safety semgrep

# Run security scans
bandit -r src/
safety check
semgrep --config=auto src/
```

### Dependency Scanning
```yaml
# GitHub Security Advisory scanning
name: Security Scan
on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
```

These development tools provide a comprehensive foundation for building, testing, debugging, and maintaining MCP servers across different programming languages and environments.