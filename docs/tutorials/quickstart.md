# MCP Quickstart Tutorial

## Build Your First MCP Server in 10 Minutes

This tutorial will guide you through creating a basic MCP server from scratch, helping you understand the core concepts and get up and running quickly.

## Prerequisites

- Choose your language: Python 3.8+, Node.js 16+, Go 1.21+, or Rust 1.75+
- Basic understanding of your chosen programming language
- Text editor or IDE
- Terminal/command line access

## Overview

We'll build a simple MCP server that provides two tools:
1. **Echo Tool**: Returns the input text with a prefix
2. **Math Tool**: Performs basic arithmetic operations

## Step 1: Project Setup

### Python Setup
```bash
# Create project directory
mkdir my-mcp-server && cd my-mcp-server

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Create project structure
mkdir -p src/tools
touch src/__init__.py
touch src/main.py
touch src/tools/__init__.py
touch src/tools/echo.py
touch src/tools/math.py

# Install MCP SDK
pip install mcp-sdk
```

### Node.js Setup
```bash
# Create project directory
mkdir my-mcp-server && cd my-mcp-server

# Initialize npm project
npm init -y

# Install dependencies
npm install @modelcontextprotocol/sdk

# Create project structure
mkdir -p src/tools
touch src/index.js
touch src/tools/echo.js
touch src/tools/math.js
```

### Go Setup
```bash
# Create project directory
mkdir my-mcp-server && cd my-mcp-server

# Initialize Go module
go mod init my-mcp-server

# Create project structure
mkdir -p cmd/server internal/tools
touch cmd/server/main.go
touch internal/tools/echo.go
touch internal/tools/math.go

# Add MCP SDK dependency
go get github.com/modelcontextprotocol/go-sdk
```

### Rust Setup
```bash
# Create project directory
cargo new my-mcp-server --bin
cd my-mcp-server

# Create project structure
mkdir -p src/tools
touch src/tools/mod.rs
touch src/tools/echo.rs
touch src/tools/math.rs

# Add dependencies to Cargo.toml
cat >> Cargo.toml << EOF

[dependencies]
mcp-sdk = "0.1"
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
anyhow = "1.0"
EOF
```

## Step 2: Implement the Echo Tool

### Python Implementation
```python
# src/tools/echo.py
from typing import Dict, List, Any
from mcp_sdk import Tool, Content, TextContent

class EchoTool(Tool):
    def __init__(self):
        super().__init__(
            name="echo",
            description="Echo the input text with a prefix"
        )
    
    def get_schema(self) -> Dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "text": {
                    "type": "string",
                    "description": "Text to echo"
                },
                "prefix": {
                    "type": "string",
                    "description": "Prefix to add to the text",
                    "default": "Echo: "
                }
            },
            "required": ["text"]
        }
    
    async def execute(self, arguments: Dict[str, Any]) -> List[Content]:
        text = arguments.get("text", "")
        prefix = arguments.get("prefix", "Echo: ")
        
        result = f"{prefix}{text}"
        
        return [TextContent(text=result)]
```

### Node.js Implementation
```javascript
// src/tools/echo.js
class EchoTool {
    constructor() {
        this.name = "echo";
        this.description = "Echo the input text with a prefix";
    }

    getSchema() {
        return {
            type: "object",
            properties: {
                text: {
                    type: "string",
                    description: "Text to echo"
                },
                prefix: {
                    type: "string",
                    description: "Prefix to add to the text",
                    default: "Echo: "
                }
            },
            required: ["text"]
        };
    }

    async execute(arguments) {
        const text = arguments.text || "";
        const prefix = arguments.prefix || "Echo: ";
        
        const result = `${prefix}${text}`;
        
        return [{ type: "text", text: result }];
    }
}

module.exports = EchoTool;
```

### Go Implementation
```go
// internal/tools/echo.go
package tools

import (
    "context"
    "fmt"
    
    "github.com/modelcontextprotocol/go-sdk/server"
)

type EchoTool struct{}

func NewEchoTool() *EchoTool {
    return &EchoTool{}
}

func (t *EchoTool) Name() string {
    return "echo"
}

func (t *EchoTool) Description() string {
    return "Echo the input text with a prefix"
}

func (t *EchoTool) InputSchema() map[string]interface{} {
    return map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "text": map[string]interface{}{
                "type":        "string",
                "description": "Text to echo",
            },
            "prefix": map[string]interface{}{
                "type":        "string",
                "description": "Prefix to add to the text",
                "default":     "Echo: ",
            },
        },
        "required": []string{"text"},
    }
}

func (t *EchoTool) Execute(ctx context.Context, arguments map[string]interface{}) ([]server.Content, error) {
    text, ok := arguments["text"].(string)
    if !ok {
        return nil, fmt.Errorf("text argument is required and must be a string")
    }
    
    prefix := "Echo: "
    if p, ok := arguments["prefix"].(string); ok {
        prefix = p
    }
    
    result := fmt.Sprintf("%s%s", prefix, text)
    
    return []server.Content{
        {
            Type: "text",
            Text: result,
        },
    }, nil
}
```

### Rust Implementation
```rust
// src/tools/echo.rs
use async_trait::async_trait;
use serde_json::{json, Value};
use std::collections::HashMap;

use crate::tools::Tool;

pub struct EchoTool;

#[async_trait]
impl Tool for EchoTool {
    fn name(&self) -> &str {
        "echo"
    }
    
    fn description(&self) -> &str {
        "Echo the input text with a prefix"
    }
    
    fn schema(&self) -> Value {
        json!({
            "type": "object",
            "properties": {
                "text": {
                    "type": "string",
                    "description": "Text to echo"
                },
                "prefix": {
                    "type": "string",
                    "description": "Prefix to add to the text",
                    "default": "Echo: "
                }
            },
            "required": ["text"]
        })
    }
    
    async fn execute(&self, arguments: HashMap<String, Value>) -> Result<Vec<Content>, Box<dyn std::error::Error + Send + Sync>> {
        let text = arguments.get("text")
            .and_then(|v| v.as_str())
            .ok_or("text argument is required and must be a string")?;
        
        let prefix = arguments.get("prefix")
            .and_then(|v| v.as_str())
            .unwrap_or("Echo: ");
        
        let result = format!("{}{}", prefix, text);
        
        Ok(vec![Content {
            content_type: "text".to_string(),
            text: Some(result),
            data: None,
        }])
    }
}
```

## Step 3: Implement the Math Tool

### Python Implementation
```python
# src/tools/math.py
from typing import Dict, List, Any
import operator
from mcp_sdk import Tool, Content, TextContent

class MathTool(Tool):
    def __init__(self):
        super().__init__(
            name="math",
            description="Perform basic arithmetic operations"
        )
        
        self.operations = {
            "add": operator.add,
            "subtract": operator.sub,
            "multiply": operator.mul,
            "divide": operator.truediv,
        }
    
    def get_schema(self) -> Dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "operation": {
                    "type": "string",
                    "enum": list(self.operations.keys()),
                    "description": "Mathematical operation to perform"
                },
                "a": {
                    "type": "number",
                    "description": "First number"
                },
                "b": {
                    "type": "number",
                    "description": "Second number"
                }
            },
            "required": ["operation", "a", "b"]
        }
    
    async def execute(self, arguments: Dict[str, Any]) -> List[Content]:
        operation = arguments.get("operation")
        a = arguments.get("a")
        b = arguments.get("b")
        
        if operation not in self.operations:
            raise ValueError(f"Unsupported operation: {operation}")
        
        if operation == "divide" and b == 0:
            raise ValueError("Cannot divide by zero")
        
        try:
            result = self.operations[operation](a, b)
            return [TextContent(text=f"{a} {operation} {b} = {result}")]
        except Exception as e:
            raise ValueError(f"Calculation error: {str(e)}")
```

### Node.js Implementation
```javascript
// src/tools/math.js
class MathTool {
    constructor() {
        this.name = "math";
        this.description = "Perform basic arithmetic operations";
        
        this.operations = {
            add: (a, b) => a + b,
            subtract: (a, b) => a - b,
            multiply: (a, b) => a * b,
            divide: (a, b) => {
                if (b === 0) throw new Error("Cannot divide by zero");
                return a / b;
            }
        };
    }

    getSchema() {
        return {
            type: "object",
            properties: {
                operation: {
                    type: "string",
                    enum: Object.keys(this.operations),
                    description: "Mathematical operation to perform"
                },
                a: {
                    type: "number",
                    description: "First number"
                },
                b: {
                    type: "number",
                    description: "Second number"
                }
            },
            required: ["operation", "a", "b"]
        };
    }

    async execute(arguments) {
        const { operation, a, b } = arguments;
        
        if (!this.operations[operation]) {
            throw new Error(`Unsupported operation: ${operation}`);
        }
        
        try {
            const result = this.operations[operation](a, b);
            return [{ 
                type: "text", 
                text: `${a} ${operation} ${b} = ${result}` 
            }];
        } catch (error) {
            throw new Error(`Calculation error: ${error.message}`);
        }
    }
}

module.exports = MathTool;
```

## Step 4: Create the Main Server

### Python Server
```python
# src/main.py
import asyncio
from mcp_sdk import Server
from tools.echo import EchoTool
from tools.math import MathTool

async def main():
    # Create server instance
    server = Server("my-mcp-server", "1.0.0")
    
    # Register tools
    server.add_tool(EchoTool())
    server.add_tool(MathTool())
    
    # Start server
    print("Starting MCP server...")
    await server.start()

if __name__ == "__main__":
    asyncio.run(main())
```

### Node.js Server
```javascript
// src/index.js
const { Server } = require('@modelcontextprotocol/sdk');
const EchoTool = require('./tools/echo');
const MathTool = require('./tools/math');

async function main() {
    // Create server instance
    const server = new Server({
        name: "my-mcp-server",
        version: "1.0.0"
    });
    
    // Register tools
    server.addTool(new EchoTool());
    server.addTool(new MathTool());
    
    // Start server
    console.log("Starting MCP server...");
    await server.start();
}

main().catch(console.error);
```

### Go Server
```go
// cmd/server/main.go
package main

import (
    "context"
    "log"
    
    "github.com/modelcontextprotocol/go-sdk/server"
    "my-mcp-server/internal/tools"
)

func main() {
    // Create server
    srv := server.New("my-mcp-server", "1.0.0")
    
    // Register tools
    srv.AddTool(tools.NewEchoTool())
    srv.AddTool(tools.NewMathTool())
    
    // Start server
    log.Println("Starting MCP server...")
    if err := srv.Start(context.Background()); err != nil {
        log.Fatal(err)
    }
}
```

### Rust Server
```rust
// src/main.rs
use tokio;
use mcp_sdk::Server;
mod tools;
use tools::{EchoTool, MathTool};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create server
    let mut server = Server::new("my-mcp-server", "1.0.0");
    
    // Register tools
    server.add_tool(Box::new(EchoTool));
    server.add_tool(Box::new(MathTool));
    
    // Start server
    println!("Starting MCP server...");
    server.start().await?;
    
    Ok(())
}
```

## Step 5: Test Your Server

### Create a Test Script

#### Python Test
```python
# test_server.py
import asyncio
import json
from mcp_sdk.client import Client

async def test_server():
    client = Client()
    
    try:
        # Connect to server
        await client.connect("stdio", ["python", "src/main.py"])
        
        # Test echo tool
        echo_result = await client.call_tool("echo", {
            "text": "Hello, World!",
            "prefix": "Server says: "
        })
        print("Echo result:", echo_result)
        
        # Test math tool
        math_result = await client.call_tool("math", {
            "operation": "add",
            "a": 5,
            "b": 3
        })
        print("Math result:", math_result)
        
    finally:
        await client.disconnect()

if __name__ == "__main__":
    asyncio.run(test_server())
```

#### Manual Testing with curl
```bash
# Test the server manually (if running HTTP transport)
echo '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
}' | curl -X POST \
    -H "Content-Type: application/json" \
    -d @- \
    http://localhost:8000
```

## Step 6: Run and Test

### Start Your Server
```bash
# Python
python src/main.py

# Node.js
node src/index.js

# Go
go run cmd/server/main.go

# Rust
cargo run
```

### Expected Output
When you start your server, you should see:
```
Starting MCP server...
Server listening on stdio transport...
```

## Step 7: Connect with Claude Desktop (Optional)

If you have Claude Desktop installed, you can connect your server:

1. Open Claude Desktop settings
2. Add your server configuration:

```json
{
  "servers": {
    "my-mcp-server": {
      "command": "python",
      "args": ["/path/to/your/project/src/main.py"]
    }
  }
}
```

3. Restart Claude Desktop
4. Try asking Claude to use your tools:
   - "Echo 'Hello from MCP!'"
   - "Calculate 15 + 27 using the math tool"

## Next Steps

Congratulations! You've built your first MCP server. Here's what you can do next:

### Enhance Your Tools
- Add input validation
- Implement error handling
- Add more mathematical operations
- Create tools for file operations

### Add More Features
- Implement resources (data sources)
- Add prompts (predefined interactions)
- Include logging and monitoring
- Add configuration management

### Deploy Your Server
- Package as a container
- Deploy to cloud platforms
- Set up CI/CD pipelines
- Add health checks and metrics

### Advanced Topics
- Explore the [Advanced Tutorial](./advanced.md)
- Learn about [Security Best Practices](../secure/index.md)
- Study [Deployment Patterns](../deploy/index.md)

## Troubleshooting

### Common Issues

1. **"Module not found" errors**: Ensure all dependencies are installed and paths are correct
2. **Server won't start**: Check port availability and permissions
3. **Tools not responding**: Verify tool registration and schema definitions
4. **Connection issues**: Ensure transport configuration matches client expectations

### Getting Help

- Check the [FAQ](../faq/troubleshooting.md)
- Visit the MCP documentation
- Join the community discussions
- Review example implementations

You now have a working MCP server! This foundation will help you understand the core concepts as you build more sophisticated integrations.