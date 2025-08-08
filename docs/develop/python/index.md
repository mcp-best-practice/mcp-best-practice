# Python Development

## MCP Server Development with Python

Python is the recommended language for MCP server development, offering a mature SDK and extensive ecosystem.

## Quick Start

### Installation
```bash
# Using uv (recommended)
uv add mcp

# Using pip
pip install mcp
```

### Minimal Server
```python
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# Create server instance
server = Server("my-server")

@server.list_tools()
async def list_tools():
    """List available tools."""
    return [
        Tool(
            name="hello",
            description="Say hello to someone",
            inputSchema={
                "type": "object",
                "properties": {
                    "name": {"type": "string"}
                },
                "required": ["name"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    """Execute a tool."""
    if name == "hello":
        name_arg = arguments.get("name", "World")
        return [TextContent(type="text", text=f"Hello, {name_arg}!")]
    
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream)

if __name__ == "__main__":
    asyncio.run(main())
```

## Python MCP Features

### Type Hints
Python's type hints automatically generate JSON schemas:
```python
def process(
    text: str,
    count: int = 10,
    enabled: bool = True
) -> dict[str, Any]:
    """Type hints define the tool interface"""
```

### Async Support
Full async/await support for I/O operations:
```python
import aiohttp

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "fetch_data":
        url = arguments.get("url")
        async with aiohttp.ClientSession() as session:
            async with session.get(url) as response:
                text = await response.text()
                return [TextContent(type="text", text=text)]
```

### Pydantic Integration
Use Pydantic for complex data validation:
```python
from pydantic import BaseModel, Field

class TaskInput(BaseModel):
    title: str = Field(..., min_length=1, max_length=100)
    priority: int = Field(default=1, ge=1, le=5)
    tags: list[str] = Field(default_factory=list)

@mcp.tool()
def create_task(task: TaskInput) -> str:
    """Create a task with validation"""
    return f"Created: {task.title}"
```

## Development Setup

### Virtual Environment
```bash
# Create venv
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
# .venv\Scripts\activate   # Windows

# Install dependencies
pip install -e ".[dev]"
```

### Project Structure
```
my-python-server/
├── src/
│   └── my_server/
│       ├── __init__.py
│       ├── main.py
│       ├── tools.py
│       └── resources.py
├── tests/
│   ├── test_tools.py
│   └── test_integration.py
├── pyproject.toml
├── Makefile
└── README.md
```

## Testing

### Unit Tests with pytest
```python
# tests/test_tools.py
import pytest
from my_server.tools import process_data

def test_process_data():
    result = process_data("test input")
    assert result == "expected output"

@pytest.mark.asyncio
async def test_async_tool():
    result = await fetch_data("http://example.com")
    assert result is not None
```

### Running Tests
```bash
# Run all tests
pytest

# With coverage
pytest --cov=my_server --cov-report=html

# Specific test file
pytest tests/test_tools.py
```

## Dependency Management

### Using pyproject.toml
```toml
[project]
name = "my-mcp-server"
version = "0.1.0"
dependencies = [
    "mcp[cli]>=0.1.0",
    "pydantic>=2.0",
    "aiohttp>=3.9",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "pytest-asyncio>=0.21",
    "pytest-cov>=4.0",
    "ruff>=0.1",
]
```

## Best Practices

### 1. Use Type Hints
Always provide type hints for better IDE support and automatic validation.

### 2. Implement Logging
```python
import logging

logger = logging.getLogger(__name__)

@mcp.tool()
def my_tool(input: str) -> str:
    logger.info(f"Processing: {input}")
    try:
        result = process(input)
        logger.debug(f"Result: {result}")
        return result
    except Exception as e:
        logger.error(f"Error: {e}")
        raise
```

### 3. Handle Errors Gracefully
```python
from mcp.server.exceptions import McpError

@mcp.tool()
def safe_tool(data: str) -> str:
    if not data:
        raise McpError("Data cannot be empty")
    
    try:
        return process(data)
    except ProcessingError as e:
        logger.error(f"Processing failed: {e}")
        raise McpError("Failed to process data")
```

### 4. Use Environment Variables
```python
import os
from dotenv import load_dotenv

load_dotenv()

API_KEY = os.getenv("MCP_API_KEY")
if not API_KEY:
    raise ValueError("MCP_API_KEY not set")
```

## Performance Tips

1. **Use async for I/O**: Network requests, file operations
2. **Cache expensive operations**: Use `functools.lru_cache`
3. **Connection pooling**: Reuse database/HTTP connections
4. **Batch operations**: Process multiple items together
5. **Profile your code**: Use `cProfile` for bottlenecks

## Common Patterns

### Singleton Pattern for Clients
```python
class DatabaseClient:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.initialize()
        return cls._instance
```

### Context Manager for Resources
```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def get_connection():
    conn = await create_connection()
    try:
        yield conn
    finally:
        await conn.close()
```

## Next Steps

- ⚡ [FastMCP Framework](fastmcp.md)
- 🏗️ [Project Structure](structure.md)
- 🧪 [Testing Guide](testing.md)
- 📦 [Packaging](packaging.md)
- 🎯 [Best Practices](best-practices.md)