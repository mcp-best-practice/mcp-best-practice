# Getting Started

## Quick Start Guide

Get up and running with MCP in minutes.

## Prerequisites

- Python 3.11+ or Node.js 18+
- Git for version control
- Text editor or IDE
- Terminal/Command line access

## Installation

### Python
```bash
# Using uv (recommended)
uv add mcp

# Using pip
pip install mcp
```

### Node.js
```bash
npm install @modelcontextprotocol/sdk
```

## Your First MCP Server

### Step 1: Create Server File

```python
# hello_mcp.py
import asyncio
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

# Create server instance
server = Server("hello-server")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="say_hello",
            description="Greet someone by name",
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
    if name == "say_hello":
        name_arg = arguments.get("name", "World")
        return [TextContent(type="text", text=f"Hello, {name_arg}!")]
    
    raise ValueError(f"Unknown tool: {name}")

async def main():
    async with stdio_server() as (read_stream, write_stream):
        await server.run(read_stream, write_stream)

if __name__ == "__main__":
    asyncio.run(main())
```

### Step 2: Run the Server

```bash
# STDIO mode (for development and testing)
python hello_mcp.py
```

### Step 3: Test Your Server

You can test your MCP server using:

1. **MCP Inspector** (development tool):
```bash
npx @modelcontextprotocol/inspector python hello_mcp.py
```

2. **Claude Desktop**: Add to your configuration file

3. **Manual JSON-RPC testing**: The server communicates via stdio using JSON-RPC messages

## Next Steps

### Essential Reading
- 📋 [Standards](standards.md) - Protocol specifications
- 🏗️ [Architecture](architecture.md) - System design
- 💻 [Development Guide](../develop/index.md) - Language-specific guides

### Hands-On Tutorials
1. [Building a GitHub Integration](../develop/tutorials/github-integration.md)
2. [Creating a Database Connector](../develop/tutorials/database-connector.md)
3. [Implementing Authentication](../secure/authentication.md)

### Sample Projects
- [Time Server](https://github.com/mcp-servers/time-server)
- [File System Server](https://github.com/mcp-servers/filesystem)
- [GitHub Tools](https://github.com/mcp-servers/github)

## Common Patterns

### Tool Registration
```python
@mcp.tool()
def my_tool(param: str) -> str:
    """Tool description"""
    return result
```

### Resource Exposure
```python
@mcp.resource("config://settings")
def get_settings() -> str:
    return json.dumps(config)
```

### Error Handling
```python
@mcp.tool()
def safe_operation(data: str) -> str:
    try:
        return process(data)
    except Exception as e:
        raise McpError(f"Operation failed: {e}")
```

## Troubleshooting

### Common Issues

**Port Already in Use**
```bash
# Find process using port
lsof -i :8000
# Kill process or use different port
```

**Module Not Found**
```bash
# Ensure MCP is installed
pip list | grep mcp
# Reinstall if needed
pip install --upgrade "mcp[cli]"
```

**Connection Refused**
- Check server is running
- Verify correct port
- Check firewall settings

## Getting Help

- 📚 [Documentation](../index.md)
- 💬 [Community Forum](https://github.com/modelcontextprotocol/discussions)
- 🐛 [Issue Tracker](https://github.com/modelcontextprotocol/issues)
- ❓ [FAQ](../faq/index.md)