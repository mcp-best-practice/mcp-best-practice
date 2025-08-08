# Integration Guide

## Using MCP Servers

Learn how to integrate MCP servers with various clients, platforms, and services.

## Integration Patterns

### Direct Integration
Client connects directly to MCP server:
```
Client <---> MCP Server
```

### Gateway Integration
Client connects through MCP Gateway:
```
Client <---> Gateway <---> MCP Servers
```

### Federated Integration
Multiple gateways with shared servers:
```
Clients <---> Gateway A <---> Shared Servers
         <---> Gateway B <--->
```

## Client Libraries

### Python Client
```python
from mcp import Client

# Initialize client
client = Client("http://localhost:8000/mcp")

# List available tools
tools = client.list_tools()
for tool in tools:
    print(f"Tool: {tool.name} - {tool.description}")

# Call a tool
result = client.call_tool(
    "process_data",
    {"input": "test data", "format": "json"}
)
print(f"Result: {result}")

# Access resources
resource = client.get_resource("config://settings")
print(f"Settings: {resource}")
```

### JavaScript Client
```javascript
import { MCPClient } from '@mcp/client';

// Initialize client
const client = new MCPClient({
  url: 'http://localhost:8000/mcp',
  apiKey: process.env.MCP_API_KEY
});

// List tools
const tools = await client.listTools();
tools.forEach(tool => {
  console.log(`Tool: ${tool.name} - ${tool.description}`);
});

// Call tool
const result = await client.callTool('process_data', {
  input: 'test data',
  format: 'json'
});
console.log('Result:', result);

// Get resource
const resource = await client.getResource('config://settings');
console.log('Settings:', resource);
```

### Go Client
```go
package main

import (
    "github.com/mcp/go-client"
)

func main() {
    // Initialize client
    client := mcp.NewClient("http://localhost:8000/mcp")
    
    // List tools
    tools, err := client.ListTools()
    if err != nil {
        log.Fatal(err)
    }
    
    for _, tool := range tools {
        fmt.Printf("Tool: %s - %s\n", tool.Name, tool.Description)
    }
    
    // Call tool
    result, err := client.CallTool("process_data", map[string]interface{}{
        "input": "test data",
        "format": "json",
    })
    
    if err != nil {
        log.Fatal(err)
    }
    
    fmt.Printf("Result: %v\n", result)
}
```

## Claude Desktop Integration

### Configuration
```json
// claude_desktop_config.json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["-m", "my_mcp_server"],
      "env": {
        "MCP_API_KEY": "${MCP_API_KEY}"
      }
    }
  }
}
```

### Installation
```bash
# Install server for Claude Desktop
uv run mcp install my_server.py --name "My Server"

# With configuration
uv run mcp install my_server.py \
  --name "My Server" \
  -v API_KEY="${API_KEY}" \
  -v DEBUG=true
```

## Gateway Integration

### Register with Gateway
```bash
# Register server
curl -X POST http://gateway.example.com/servers \
  -H "Authorization: Bearer ${GATEWAY_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-server",
    "url": "http://my-server:8000/mcp",
    "transport": "http",
    "authentication": {
      "type": "bearer",
      "token": "${SERVER_TOKEN}"
    }
  }'
```

### Gateway Client Usage
```python
from mcp_gateway import GatewayClient

# Connect to gateway
gateway = GatewayClient(
    url="http://gateway.example.com",
    token=os.getenv("GATEWAY_TOKEN")
)

# List available servers
servers = gateway.list_servers()
print(f"Available servers: {servers}")

# Call tool on specific server
result = gateway.call_tool(
    server="my-server",
    tool="process_data",
    params={"input": "test"}
)
```

## REST API Integration

### OpenAPI Specification
```yaml
openapi: 3.0.0
info:
  title: MCP Server API
  version: 1.0.0
paths:
  /mcp:
    post:
      summary: Execute MCP request
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/MCPRequest'
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/MCPResponse'

components:
  schemas:
    MCPRequest:
      type: object
      properties:
        jsonrpc:
          type: string
          enum: ["2.0"]
        method:
          type: string
        params:
          type: object
        id:
          type: integer
```

### REST Client Example
```python
import requests

class MCPRestClient:
    def __init__(self, base_url, api_key=None):
        self.base_url = base_url
        self.headers = {"Content-Type": "application/json"}
        if api_key:
            self.headers["Authorization"] = f"Bearer {api_key}"
    
    def call_tool(self, tool_name, params):
        payload = {
            "jsonrpc": "2.0",
            "method": "tools/call",
            "params": {
                "name": tool_name,
                "arguments": params
            },
            "id": 1
        }
        
        response = requests.post(
            f"{self.base_url}/mcp",
            json=payload,
            headers=self.headers
        )
        
        response.raise_for_status()
        return response.json()["result"]
```

## WebSocket Integration

### WebSocket Client
```javascript
class MCPWebSocketClient {
  constructor(url) {
    this.ws = new WebSocket(url);
    this.requestId = 0;
    this.pending = new Map();
    
    this.ws.on('message', (data) => {
      const response = JSON.parse(data);
      const handler = this.pending.get(response.id);
      if (handler) {
        handler(response);
        this.pending.delete(response.id);
      }
    });
  }
  
  async callTool(name, params) {
    return new Promise((resolve, reject) => {
      const id = ++this.requestId;
      
      this.pending.set(id, (response) => {
        if (response.error) {
          reject(new Error(response.error.message));
        } else {
          resolve(response.result);
        }
      });
      
      this.ws.send(JSON.stringify({
        jsonrpc: "2.0",
        method: "tools/call",
        params: { name, arguments: params },
        id
      }));
    });
  }
}
```

## Event Streaming

### Server-Sent Events (SSE)
```python
from flask import Response, Flask
import json

app = Flask(__name__)

@app.route('/mcp/stream')
def stream_events():
    def generate():
        # Subscribe to events
        for event in mcp.subscribe_events():
            yield f"data: {json.dumps(event)}\n\n"
    
    return Response(
        generate(),
        mimetype="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no"
        }
    )
```

### Client-Side SSE
```javascript
const eventSource = new EventSource('/mcp/stream');

eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log('Received event:', data);
};

eventSource.onerror = (error) => {
  console.error('SSE error:', error);
  eventSource.close();
};
```

## Webhook Integration

### Webhook Configuration
```python
@mcp.on_tool_execution
def webhook_handler(tool_name, params, result):
    """Send webhook on tool execution"""
    webhook_url = os.getenv("WEBHOOK_URL")
    
    payload = {
        "event": "tool_executed",
        "tool": tool_name,
        "params": params,
        "result": result,
        "timestamp": datetime.utcnow().isoformat()
    }
    
    requests.post(webhook_url, json=payload)
```

### Webhook Receiver
```python
from flask import Flask, request

app = Flask(__name__)

@app.route('/webhook/mcp', methods=['POST'])
def receive_webhook():
    data = request.json
    
    # Verify webhook signature
    signature = request.headers.get('X-MCP-Signature')
    if not verify_signature(request.data, signature):
        return {'error': 'Invalid signature'}, 401
    
    # Process webhook
    event_type = data.get('event')
    if event_type == 'tool_executed':
        process_tool_execution(data)
    
    return {'status': 'received'}, 200
```

## Authentication Methods

### API Key Authentication
```python
def authenticate_api_key(api_key):
    client = MCPClient(
        url="http://localhost:8000/mcp",
        headers={"X-API-Key": api_key}
    )
    return client
```

### OAuth 2.0 Integration
```python
from authlib.integrations.requests_client import OAuth2Session

oauth = OAuth2Session(
    client_id=CLIENT_ID,
    client_secret=CLIENT_SECRET,
    redirect_uri=REDIRECT_URI
)

# Get token
token = oauth.fetch_token(
    TOKEN_URL,
    authorization_response=request.url
)

# Use token with MCP
client = MCPClient(
    url="http://localhost:8000/mcp",
    headers={"Authorization": f"Bearer {token['access_token']}"}
)
```

## Error Handling

### Client-Side Error Handling
```python
class MCPClientWithRetry:
    def __init__(self, url, max_retries=3):
        self.url = url
        self.max_retries = max_retries
    
    def call_tool_with_retry(self, tool_name, params):
        for attempt in range(self.max_retries):
            try:
                return self.call_tool(tool_name, params)
            
            except MCPError as e:
                if e.code == -32000:  # Server error
                    if attempt < self.max_retries - 1:
                        time.sleep(2 ** attempt)  # Exponential backoff
                        continue
                raise
            
            except ConnectionError as e:
                if attempt < self.max_retries - 1:
                    time.sleep(1)
                    continue
                raise
```

## Usage Examples

### Data Processing Pipeline
```python
# Chain multiple MCP tools
async def process_pipeline(data):
    # Step 1: Validate data
    validated = await client.call_tool("validate_data", {"data": data})
    
    # Step 2: Transform data
    transformed = await client.call_tool("transform_data", {
        "data": validated,
        "format": "normalized"
    })
    
    # Step 3: Analyze data
    analysis = await client.call_tool("analyze_data", {
        "data": transformed,
        "metrics": ["mean", "median", "std"]
    })
    
    # Step 4: Generate report
    report = await client.call_tool("generate_report", {
        "analysis": analysis,
        "format": "pdf"
    })
    
    return report
```

### Multi-Server Orchestration
```python
async def orchestrate_servers():
    # Connect to multiple servers
    github_client = MCPClient("http://github-server:8000/mcp")
    jira_client = MCPClient("http://jira-server:8000/mcp")
    slack_client = MCPClient("http://slack-server:8000/mcp")
    
    # Get GitHub issues
    issues = await github_client.call_tool("list_issues", {
        "repo": "myorg/myrepo",
        "state": "open"
    })
    
    # Create Jira tickets
    for issue in issues:
        ticket = await jira_client.call_tool("create_ticket", {
            "title": issue["title"],
            "description": issue["body"],
            "labels": issue["labels"]
        })
        
        # Notify via Slack
        await slack_client.call_tool("send_message", {
            "channel": "#dev-team",
            "text": f"Created Jira ticket {ticket['key']} for GitHub issue #{issue['number']}"
        })
```

## Next Steps

- 🤖 [Claude Integration](claude.md)
- 🌐 [Gateway Setup](gateway.md)
- 📦 [API Client Libraries](api-clients.md)
- 🔗 [Webhook Configuration](webhooks.md)