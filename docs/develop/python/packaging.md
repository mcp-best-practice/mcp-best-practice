# Python Packaging

## Packaging MCP Servers for Distribution

Properly packaged MCP servers are easy to install, distribute, and maintain.

## Package Structure

### Modern Python Packaging
Use `pyproject.toml` for modern Python packaging:

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-mcp-server"
version = "0.1.0"
description = "An MCP server for [specific purpose]"
readme = "README.md"
license = {text = "MIT"}
authors = [
    {name = "Your Name", email = "you@example.com"},
]
maintainers = [
    {name = "Your Name", email = "you@example.com"},
]
keywords = ["mcp", "model-context-protocol", "ai", "assistant"]
classifiers = [
    "Development Status :: 4 - Beta",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
    "Topic :: Software Development :: Libraries :: Python Modules",
    "Topic :: Scientific/Engineering :: Artificial Intelligence",
]
requires-python = ">=3.11"
dependencies = [
    "mcp>=0.1.0",
    "pydantic>=2.0.0",
    "aiofiles>=23.0.0",
    "aiohttp>=3.9.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.0.0",
    "black>=23.0.0",
    "ruff>=0.1.0",
    "mypy>=1.0.0",
    "build>=0.10.0",
]
test = [
    "pytest>=7.0.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.0.0",
]
lint = [
    "black>=23.0.0",
    "ruff>=0.1.0",
    "mypy>=1.0.0",
]

[project.urls]
"Homepage" = "https://github.com/yourusername/my-mcp-server"
"Bug Reports" = "https://github.com/yourusername/my-mcp-server/issues"
"Source" = "https://github.com/yourusername/my-mcp-server"
"Documentation" = "https://my-mcp-server.readthedocs.io/"

[project.scripts]
my-mcp-server = "my_mcp_server.server:main"

[project.entry-points."mcp.servers"]
my-server = "my_mcp_server.server:create_server"

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-data]
"my_mcp_server" = ["py.typed", "*.json", "*.yaml"]
```

## Entry Points

### Script Entry Points
```python
# src/my_mcp_server/server.py
def main():
    """Main entry point for the CLI command."""
    import asyncio
    from .server import MCPServer
    
    server = MCPServer("my-mcp-server")
    asyncio.run(server.run())

def create_server():
    """Factory function for programmatic server creation."""
    return MCPServer("my-mcp-server")

if __name__ == "__main__":
    main()
```

## Building Packages

### Build Tools Setup
```bash
# Install build tools
uv add --dev build twine

# Or with pip
pip install build twine
```

### Building the Package
```bash
# Clean previous builds
rm -rf dist/ build/ *.egg-info/

# Build the package
python -m build

# This creates:
# dist/my_mcp_server-0.1.0.tar.gz (source distribution)
# dist/my_mcp_server-0.1.0-py3-none-any.whl (wheel)
```

### Package Validation
```bash
# Check package contents
tar -tzf dist/my_mcp_server-0.1.0.tar.gz
unzip -l dist/my_mcp_server-0.1.0-py3-none-any.whl

# Validate package metadata
twine check dist/*

# Test installation
pip install dist/my_mcp_server-0.1.0-py3-none-any.whl
```

## Publishing

### PyPI Publishing
```bash
# Configure PyPI token
# Create .pypirc or use environment variables
export TWINE_USERNAME=__token__
export TWINE_PASSWORD=pypi-your-api-token-here

# Upload to Test PyPI first
twine upload --repository testpypi dist/*

# Test install from Test PyPI
pip install --index-url https://test.pypi.org/simple/ my-mcp-server

# Upload to production PyPI
twine upload dist/*
```

### Automated Publishing
```yaml
# .github/workflows/publish.yml
name: Publish Python Package

on:
  release:
    types: [published]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install build twine
    
    - name: Build package
      run: python -m build
    
    - name: Publish to PyPI
      env:
        TWINE_USERNAME: __token__
        TWINE_PASSWORD: ${{ secrets.PYPI_API_TOKEN }}
      run: |
        twine upload dist/*
```

## Version Management

### Semantic Versioning
Follow semantic versioning (semver):
- `MAJOR.MINOR.PATCH`
- `1.0.0` - Initial stable release
- `1.0.1` - Bug fixes
- `1.1.0` - New features (backward compatible)
- `2.0.0` - Breaking changes

### Dynamic Versioning
```python
# src/my_mcp_server/__init__.py
__version__ = "0.1.0"
__all__ = ["__version__"]
```

```toml
# pyproject.toml - dynamic version
[project]
dynamic = ["version"]

[tool.setuptools.dynamic]
version = {attr = "my_mcp_server.__version__"}
```

## Documentation Packaging

### Include Documentation
```toml
# Include documentation in package
[tool.setuptools]
include-package-data = true

[tool.setuptools.package-data]
"my_mcp_server" = ["docs/*.md", "examples/*.py"]
```

### README and Changelog
```python
# Use README as long description
[project]
readme = "README.md"

# Or dynamically read README
[tool.setuptools.dynamic]
readme = {file = ["README.md"], content-type = "text/markdown"}
```

## Type Information

### Include Type Information
```python
# src/my_mcp_server/py.typed
# Empty file to indicate package includes type information
```

```toml
# Include py.typed in package
[tool.setuptools.package-data]
"my_mcp_server" = ["py.typed"]
```

## Development Installation

### Editable Installation
```bash
# Install in development mode
pip install -e .

# With development dependencies
pip install -e ".[dev]"

# Using uv
uv pip install -e ".[dev]"
```

### Development Workflow
```bash
# Setup development environment
git clone https://github.com/yourusername/my-mcp-server
cd my-mcp-server
python -m venv .venv
source .venv/bin/activate  # On Windows: .venv\Scripts\activate
pip install -e ".[dev]"

# Run tests
pytest

# Format code
black src tests
ruff check src tests

# Type checking
mypy src

# Build package
python -m build
```

## Makefile Automation

```makefile
# Makefile
.PHONY: install dev test lint format build clean publish

install:
	pip install .

dev:
	pip install -e ".[dev]"

test:
	pytest --cov=src --cov-report=term-missing

lint:
	ruff check src tests
	mypy src

format:
	black src tests
	ruff check --fix src tests

build: clean
	python -m build

clean:
	rm -rf dist/ build/ *.egg-info/ .coverage htmlcov/

publish: build
	twine upload dist/*

test-publish: build
	twine upload --repository testpypi dist/*
```

## Docker Packaging

### Dockerfile for Distribution
```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Copy package files
COPY dist/my_mcp_server-*.whl .

# Install package
RUN pip install --no-cache-dir my_mcp_server-*.whl

# Create non-root user
RUN useradd -m -u 1000 mcp
USER mcp

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import my_mcp_server; print('OK')"

CMD ["my-mcp-server"]
```

## Best Practices

### Package Metadata
1. **Descriptive Name**: Clear, searchable package name
2. **Good Description**: Explain what the server does
3. **Proper Classifiers**: Help users find your package
4. **Version Pinning**: Pin major versions, allow minor updates

### Dependencies
1. **Minimal Dependencies**: Only include necessary packages
2. **Version Constraints**: Use `>=` for minimum versions
3. **Optional Dependencies**: Group related optional features
4. **Security**: Regularly update dependencies

### Distribution
1. **Test First**: Always test on Test PyPI first
2. **Automated Builds**: Use CI/CD for consistent builds
3. **Signed Packages**: Sign packages for security
4. **Documentation**: Include comprehensive documentation

### Maintenance
1. **Semantic Versioning**: Follow semver strictly
2. **Changelog**: Maintain detailed changelog
3. **Backward Compatibility**: Avoid breaking changes in minor versions
4. **Deprecation Warnings**: Warn before removing features

Proper packaging makes your MCP server accessible to the community and easy to integrate into various environments.