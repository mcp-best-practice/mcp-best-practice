# Python Project Structure

## Recommended Project Layout

A well-structured MCP server project follows Python packaging standards and best practices.

## Standard Structure

```
my-mcp-server/
├── src/
│   └── my_mcp_server/
│       ├── __init__.py
│       ├── server.py          # Main server implementation
│       ├── tools/             # Tool implementations
│       │   ├── __init__.py
│       │   ├── database.py
│       │   └── file_ops.py
│       ├── resources/         # Resource providers
│       │   ├── __init__.py
│       │   └── config.py
│       └── utils/            # Shared utilities
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── test_server.py
│   ├── test_tools/
│   │   ├── __init__.py
│   │   ├── test_database.py
│   │   └── test_file_ops.py
│   └── fixtures/
│       └── sample_data.json
├── docs/
│   ├── api.md
│   └── usage.md
├── scripts/
│   ├── run_server.py
│   └── setup_dev.py
├── pyproject.toml
├── README.md
├── CHANGELOG.md
├── LICENSE
└── .gitignore
```

## Core Components

### Server Module (`server.py`)
```python
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from .tools import register_tools
from .resources import register_resources

class MCPServer:
    def __init__(self, name: str):
        self.server = Server(name)
        self._register_handlers()
    
    def _register_handlers(self):
        register_tools(self.server)
        register_resources(self.server)
    
    async def run(self):
        async with stdio_server() as (read_stream, write_stream):
            await self.server.run(read_stream, write_stream)

def main():
    server = MCPServer("my-mcp-server")
    asyncio.run(server.run())

if __name__ == "__main__":
    main()
```

### Tool Organization (`tools/__init__.py`)
```python
from mcp.server import Server
from .database import register_database_tools
from .file_ops import register_file_tools

def register_tools(server: Server):
    """Register all tools with the server."""
    register_database_tools(server)
    register_file_tools(server)
```

### Individual Tool Modules (`tools/database.py`)
```python
from mcp.server import Server
from mcp.types import Tool, TextContent
import json

def register_database_tools(server: Server):
    """Register database-related tools."""
    
    @server.list_tools()
    async def list_tools():
        return [
            Tool(
                name="query_database",
                description="Execute a SQL query",
                inputSchema={
                    "type": "object",
                    "properties": {
                        "query": {"type": "string"},
                        "limit": {"type": "integer", "default": 100}
                    },
                    "required": ["query"]
                }
            )
        ]
    
    @server.call_tool()
    async def call_tool(name: str, arguments: dict):
        if name == "query_database":
            return await _execute_query(arguments)
        raise ValueError(f"Unknown tool: {name}")

async def _execute_query(args: dict):
    """Execute database query (implementation)."""
    query = args["query"]
    limit = args.get("limit", 100)
    
    # Your database logic here
    result = {"query": query, "limit": limit, "results": []}
    
    return [TextContent(type="text", text=json.dumps(result))]
```

## Configuration Management

### Settings Module (`utils/config.py`)
```python
import os
from dataclasses import dataclass
from typing import Optional

@dataclass
class Settings:
    """Application settings from environment variables."""
    
    # Server settings
    server_name: str = "my-mcp-server"
    debug: bool = False
    
    # Database settings
    database_url: Optional[str] = None
    database_timeout: int = 30
    
    # API settings
    api_key: Optional[str] = None
    api_base_url: str = "https://api.example.com"
    
    @classmethod
    def from_env(cls) -> 'Settings':
        """Load settings from environment variables."""
        return cls(
            server_name=os.getenv("MCP_SERVER_NAME", cls.server_name),
            debug=os.getenv("MCP_DEBUG", "false").lower() == "true",
            database_url=os.getenv("DATABASE_URL"),
            database_timeout=int(os.getenv("DATABASE_TIMEOUT", "30")),
            api_key=os.getenv("API_KEY"),
            api_base_url=os.getenv("API_BASE_URL", cls.api_base_url)
        )

# Global settings instance
settings = Settings.from_env()
```

## Package Configuration

### pyproject.toml
```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-mcp-server"
version = "0.1.0"
description = "My MCP Server implementation"
readme = "README.md"
license = {text = "MIT"}
authors = [{name = "Your Name", email = "you@example.com"}]
requires-python = ">=3.11"
dependencies = [
    "mcp>=0.1.0",
    "aiofiles>=23.0.0",
    "pydantic>=2.0.0",
]
classifiers = [
    "Development Status :: 3 - Alpha",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Programming Language :: Python :: 3.11",
    "Programming Language :: Python :: 3.12",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-asyncio>=0.21.0",
    "pytest-cov>=4.0.0",
    "black>=23.0.0",
    "ruff>=0.1.0",
    "mypy>=1.0.0",
]

[project.scripts]
my-mcp-server = "my_mcp_server.server:main"

[tool.setuptools.packages.find]
where = ["src"]

[tool.black]
line-length = 88
target-version = ['py311']

[tool.ruff]
target-version = "py311"
line-length = 88

[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
```

## Development Scripts

### Development Runner (`scripts/run_server.py`)
```python
#!/usr/bin/env python3
"""Development server runner with hot reload."""

import sys
import os
from pathlib import Path

# Add src to path
src_path = Path(__file__).parent.parent / "src"
sys.path.insert(0, str(src_path))

from my_mcp_server.server import main

if __name__ == "__main__":
    # Set development environment
    os.environ.setdefault("MCP_DEBUG", "true")
    main()
```

## Best Practices

### Code Organization
1. **Separation of Concerns**: Keep tools, resources, and utilities in separate modules
2. **Type Hints**: Use comprehensive type annotations
3. **Error Handling**: Implement proper exception handling
4. **Logging**: Use structured logging throughout

### Dependencies
1. **Minimal Dependencies**: Only include necessary packages
2. **Version Pinning**: Pin major versions, allow minor updates
3. **Development Dependencies**: Separate dev dependencies from runtime

### Testing Structure
1. **Mirror Source Structure**: Tests should mirror the src/ structure
2. **Fixtures**: Use pytest fixtures for common test data
3. **Integration Tests**: Test the full server integration

### Documentation
1. **Docstrings**: Document all public functions and classes
2. **Type Hints**: Use as documentation for function signatures
3. **README**: Include setup, usage, and configuration instructions

This structure provides a solid foundation for maintainable, testable MCP servers in Python.