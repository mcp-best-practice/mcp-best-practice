# Core Concepts

## Fundamental MCP Concepts

Understanding these core concepts is essential for working with the Model Context Protocol.

## Tools

Tools are the primary mechanism for exposing functionality through MCP.

### Tool Definition
- **Name**: Unique identifier for the tool
- **Description**: Human-readable explanation
- **Parameters**: Typed input schema (JSON Schema)
- **Return Type**: Expected output format

### Protocol Methods
- **Discovery**: `tools/list` - Lists available tools
- **Execution**: `tools/call` - Executes a specific tool

### Example Tool Definition
```json
{
  "name": "calculate_sum",
  "description": "Add two numbers together",
  "inputSchema": {
    "type": "object",
    "properties": {
      "a": {"type": "number"},
      "b": {"type": "number"}
    },
    "required": ["a", "b"]
  }
}
```

## Resources

Resources provide contextual data to AI models without requiring explicit tool calls.

### Resource Types
- **Static**: Fixed content (configuration, documentation)
- **Dynamic**: Real-time data (status, metrics)
- **Streaming**: Continuous updates (logs, events)

### Resource URIs
Resources use URI schemes for identification:
- `file://path/to/resource`
- `config://settings/app`
- `status://health/check`

### Protocol Methods
- **Discovery**: `resources/list` - Lists available resources
- **Access**: `resources/read` - Retrieves resource content
- **Templates**: `resources/templates/list` - Lists resource templates
- **Monitoring**: `resources/subscribe` - Monitors resource changes

## Prompts

Pre-configured interaction templates that guide AI behavior.

### Prompt Components
- **Name**: Identifier for the prompt
- **Description**: Purpose and usage
- **Arguments**: Dynamic values to inject
- **Template**: The actual prompt text

### Protocol Methods
- **Discovery**: `prompts/list` - Lists available prompts
- **Retrieval**: `prompts/get` - Retrieves a specific prompt

## Contexts

Execution contexts provide:
- **Logging**: Structured logging capabilities
- **Progress**: Status updates for long operations
- **Cancellation**: Graceful operation termination
- **Metadata**: Request-specific information

## Messages

MCP uses JSON-RPC 2.0 for message formatting.

### Message Types
1. **Request**: Client-initiated operations
2. **Response**: Server results
3. **Notification**: Async events
4. **Error**: Structured error responses

## Capabilities

Servers declare their capabilities during initialization:
- Supported tools
- Available resources
- Protocol version
- Transport options

## Sessions

Stateful connections between clients and servers:
- Session establishment
- Capability negotiation
- State management
- Graceful termination