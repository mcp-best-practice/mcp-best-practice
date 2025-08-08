# MCP Architecture

## System Architecture

The Model Context Protocol follows a client-server architecture with optional gateway intermediaries for enterprise deployments.

## Components Overview

### MCP Server
The foundation of the MCP ecosystem, servers expose:
- **Tools**: Executable functions with defined inputs/outputs
- **Resources**: Data and content providers
- **Prompts**: Pre-configured interaction templates

### MCP Client
Clients connect to servers and can:
- Discover available capabilities
- Execute tools with parameters
- Fetch resources for context
- Utilize prompts for interactions

### MCP Gateway
Enterprise-grade component that:
- Manages multiple MCP servers
- Provides centralized authentication
- Enables load balancing and failover
- Offers unified API access

## Communication Patterns

### Direct Connection
```
Client <---> MCP Server
```

### Gateway Pattern
```
Client <---> Gateway <---> MCP Server 1
                      <---> MCP Server 2
                      <---> MCP Server N
```

## Transport Protocols

MCP defines two standard transport mechanisms:

| Protocol | Use Case | Description |
|----------|----------|-------------|
| **stdio** | Local processes | Client launches server as subprocess, communicates via stdin/stdout |
| **Streamable HTTP** | Remote servers | HTTP POST for client-to-server, optional Server-Sent Events for streaming |

### stdio Transport
- Client launches MCP server as subprocess
- Server reads JSON-RPC messages from stdin
- Server writes JSON-RPC messages to stdout
- Messages delimited by newlines, no embedded newlines
- Server may log to stderr (captured by client)

### Streamable HTTP Transport  
- Client sends JSON-RPC via HTTP POST to MCP endpoint
- Server responds with either JSON or Server-Sent Events
- Supports standard HTTP authentication (Bearer tokens, API keys)
- Enables remote server deployment and scaling

## Data Flow

1. **Discovery**: Client queries available capabilities
2. **Invocation**: Client calls tools with parameters
3. **Processing**: Server executes requested operations
4. **Response**: Server returns results to client

## Security Layers

- **Authentication**: Bearer tokens, API keys
- **Authorization**: Role-based access control
- **Encryption**: TLS for network transport
- **Validation**: Input/output sanitization