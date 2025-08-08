# MCP Standards

## Protocol Standards

The Model Context Protocol follows established standards and conventions to ensure interoperability and consistency.

## Protocol Version

Current version: **2025-06-18**

### Version Format
MCP uses date-based versioning format: `YYYY-MM-DD`
- Each revision represents the protocol state at that date
- Breaking changes are indicated in the changelog
- Implementations must support capability negotiation

## Message Format

MCP uses JSON-RPC 2.0 specification.

### Request Structure
```json
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "tool_name",
    "arguments": {}
  },
  "id": 1
}
```

### Response Structure
```json
{
  "jsonrpc": "2.0",
  "result": {
    "content": []
  },
  "id": 1
}
```

## Naming Conventions

### Tools
- **Format**: `snake_case`
- **Examples**: `get_user`, `create_ticket`, `fetch_data`

### Resources
- **Format**: URI scheme
- **Examples**: `file://docs/readme`, `config://app/settings`

### Environment Variables
- **Format**: `SCREAMING_SNAKE_CASE` with role prefix
- **Examples**: `MCP_GITHUB_TOKEN`, `MCP_SERVER_PORT`

## Error Codes

Standard JSON-RPC error codes plus MCP-specific ranges:

| Code Range | Description |
|------------|-------------|
| -32700 to -32600 | JSON-RPC protocol errors |
| -32000 to -32099 | MCP server errors |
| -31000 to -31999 | Tool execution errors |
| -30000 to -30999 | Resource access errors |

## Data Types

### Primitive Types
- `string`: UTF-8 text
- `number`: JSON number (float/int)
- `boolean`: true/false
- `null`: Null value

### Complex Types
- `object`: JSON object with schema
- `array`: Ordered list of values

## Content Types

MCP supports multiple content types in responses:

| Type | Description | Example |
|------|-------------|---------|
| `text` | Plain text content | Error messages, simple responses |
| `image` | Base64 encoded images | Screenshots, diagrams |
| `resource` | Resource references | File paths, URIs |

## Transport Standards

### HTTP
- **Endpoint**: `/mcp`
- **Methods**: POST only
- **Content-Type**: `application/json`
- **Authentication**: Bearer token in Authorization header

### WebSocket
- **Subprotocol**: `mcp.v1`
- **Ping/Pong**: Every 30 seconds
- **Message framing**: Text frames with JSON

## Compliance Requirements

All MCP implementations must:
1. Support JSON-RPC 2.0
2. Implement capability discovery
3. Provide error handling
4. Support graceful shutdown
5. Include health checks