# Frequently Asked Questions

## General Questions

### What is MCP?
The Model Context Protocol (MCP) is a standardized protocol for enabling AI models to interact with external tools, resources, and services through a well-defined interface.

### Why should I use MCP?
- **Standardization**: Consistent interface across all integrations
- **Security**: Built-in authentication and authorization
- **Scalability**: Designed for distributed systems
- **Flexibility**: Multiple transport protocols and languages
- **Ecosystem**: Growing library of pre-built servers

### What languages are supported?
Official SDKs:
- Python (recommended)
- JavaScript/TypeScript
- Go

Community SDKs:
- Rust
- Java
- Ruby
- C#

## Getting Started

### How do I create my first MCP server?
```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my_server")

@mcp.tool()
def hello(name: str) -> str:
    return f"Hello, {name}!"

if __name__ == "__main__":
    mcp.run()
```

### How do I test my MCP server?
```bash
# Use MCP Inspector
uv run mcp dev my_server.py

# Or test with curl
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

### How do I deploy my MCP server?
1. Package your server (Python wheel, NPM package, or container)
2. Choose deployment platform (Kubernetes, Cloud Run, Lambda)
3. Configure environment variables
4. Set up monitoring and logging
5. Deploy and verify health checks

## Architecture

### What's the difference between tools and resources?
- **Tools**: Active operations that perform actions (functions)
- **Resources**: Passive data providers (configuration, state)

### Should I create one server with many tools or many servers with few tools?
Follow the single responsibility principle:
- ✅ One server per service/domain (GitHub server, Database server)
- ❌ One monolithic server with all tools

### How do I handle authentication?
MCP supports multiple authentication methods:
- API Keys (simplest)
- Bearer tokens (JWT)
- OAuth 2.0 (enterprise)
- mTLS (highest security)

## Performance

### How many concurrent requests can an MCP server handle?
Depends on:
- Server implementation (async vs sync)
- Hardware resources
- Tool complexity
- Network latency

Typical benchmarks:
- Python async: 1000-5000 req/s
- Go: 5000-20000 req/s
- Node.js: 2000-8000 req/s

### How can I improve performance?
1. Use async/await for I/O operations
2. Implement connection pooling
3. Add caching layers
4. Use efficient serialization
5. Optimize database queries
6. Scale horizontally

### What's the recommended timeout for MCP requests?
- Default: 30 seconds
- Long operations: 5 minutes
- Real-time operations: 5 seconds

Configure based on your use case.

## Security

### How do I secure my MCP server?
1. Always use HTTPS in production
2. Implement authentication
3. Validate and sanitize all inputs
4. Use environment variables for secrets
5. Enable rate limiting
6. Log security events
7. Regular security scans

### Can I restrict which tools a client can access?
Yes, implement role-based access control (RBAC):
```python
@mcp.tool(roles=["admin"])
def dangerous_operation():
    # Only admins can access
    pass
```

### How do I handle sensitive data?
- Never log sensitive information
- Encrypt data in transit (TLS)
- Encrypt data at rest
- Use secure secret management
- Implement data retention policies
- Follow compliance requirements (GDPR, HIPAA)

## Troubleshooting

### My server won't start
Check:
1. Port availability: `lsof -i :8000`
2. Dependencies installed: `pip list`
3. Environment variables set
4. Syntax errors: `python -m py_compile my_server.py`
5. Permissions: file and network access

### Tools aren't showing up
Verify:
1. Tools are decorated with `@mcp.tool()`
2. Server is running: check health endpoint
3. No startup errors in logs
4. Correct MCP protocol version

### Connection refused errors
Troubleshoot:
1. Server is running: `ps aux | grep mcp`
2. Correct URL and port
3. Firewall rules allow connection
4. Network connectivity: `ping` and `traceroute`
5. DNS resolution: `nslookup`

## Best Practices

### How should I structure my MCP server project?
```
my-mcp-server/
├── src/
│   └── server/
│       ├── __init__.py
│       ├── main.py
│       ├── tools.py
│       └── resources.py
├── tests/
├── docs/
├── Containerfile
├── Makefile
├── pyproject.toml
└── README.md
```

### What should I include in my README?
- Overview and purpose
- Installation instructions
- Configuration options
- Available tools and resources
- Usage examples
- Troubleshooting guide
- Contributing guidelines

### How do I version my MCP server?
Use semantic versioning:
- MAJOR: Breaking changes
- MINOR: New features (backward compatible)
- PATCH: Bug fixes

Example: `1.2.3`

## Integration

### How do I integrate with Claude Desktop?
```bash
uv run mcp install my_server.py --name "My Server"
```

### How do I connect to MCP Gateway?
```bash
curl -X POST http://gateway:4444/servers \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"my-server","url":"http://localhost:8000/mcp"}'
```

### Can I use MCP with other AI models?
Yes, MCP is model-agnostic. Any system that can make HTTP requests and handle JSON can use MCP.

## Advanced Topics

### How do I implement streaming responses?
```python
@mcp.tool()
async def stream_data():
    async def generate():
        for i in range(10):
            yield f"Data chunk {i}\n"
            await asyncio.sleep(1)
    
    return StreamingResponse(generate())
```

### Can I chain multiple MCP servers?
Yes, servers can call other servers:
```python
@mcp.tool()
async def orchestrate():
    result1 = await call_server("server1", "tool1")
    result2 = await call_server("server2", "tool2", result1)
    return result2
```

### How do I implement custom transports?
Extend the base transport class:
```python
class CustomTransport(MCPTransport):
    def send(self, message):
        # Custom send logic
        pass
    
    def receive(self):
        # Custom receive logic
        pass
```

## Common Errors

### "Method not found"
The requested method doesn't exist. Check:
- Spelling of method name
- Tool is registered
- Correct server URL

### "Invalid parameters"
Parameters don't match schema. Verify:
- Parameter names
- Data types
- Required fields
- JSON formatting

### "Internal server error"
Server encountered an error. Check:
- Server logs for stack trace
- Database connectivity
- External service availability
- Resource limits

## Migration

### Migrating from REST API to MCP
1. Map endpoints to tools
2. Convert request/response to MCP format
3. Add MCP server alongside REST
4. Gradually migrate clients
5. Deprecate REST endpoints

### Migrating between MCP versions
1. Review breaking changes
2. Update SDK version
3. Modify code for new APIs
4. Test thoroughly
5. Deploy with version compatibility

## Community

### Where can I get help?
- GitHub Discussions
- Discord community
- Stack Overflow (#mcp tag)
- Official documentation

### How can I contribute?
- Report bugs
- Submit pull requests
- Write documentation
- Create example servers
- Help others in community

### Are there example MCP servers?
Yes, check the official repositories:
- Time server (simple example)
- GitHub tools (complex example)
- File system (resource example)
- Database connector (integration example)

## Next Steps

- 🔧 [Troubleshooting Guide](troubleshooting.md)
- 💡 [Common Patterns](patterns.md)
- 🚀 [Migration Guide](migration.md)