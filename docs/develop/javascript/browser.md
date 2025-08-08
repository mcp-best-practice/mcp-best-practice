# Browser Development

## MCP Clients in the Browser

Building browser-based MCP clients enables rich web applications to interact with MCP servers through various transport mechanisms.

## Browser Transport Options

### HTTP Transport
The primary transport for browser-based MCP clients, using the Streamable HTTP specification.

```javascript
// src/client.js
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { HTTPTransport } from '@modelcontextprotocol/sdk/client/http.js';

class BrowserMCPClient {
  constructor(serverUrl) {
    this.transport = new HTTPTransport(serverUrl);
    this.client = new Client(
      {
        name: 'browser-mcp-client',
        version: '1.0.0'
      },
      {
        capabilities: {}
      }
    );
  }

  async connect() {
    await this.client.connect(this.transport);
    console.log('Connected to MCP server');
  }

  async listTools() {
    const response = await this.client.request(
      { method: 'tools/list' },
      { method: 'tools/list' }
    );
    return response.tools;
  }

  async callTool(name, arguments = {}) {
    const response = await this.client.request(
      { 
        method: 'tools/call',
        params: { name, arguments }
      },
      { 
        method: 'tools/call',
        params: { name, arguments }
      }
    );
    return response.content;
  }

  async disconnect() {
    await this.client.close();
    console.log('Disconnected from MCP server');
  }
}

// Usage
const client = new BrowserMCPClient('http://localhost:8000/mcp');
await client.connect();
```

### WebSocket Transport
For real-time communication with MCP servers (when supported):

```javascript
// src/websocket-client.js
class WebSocketMCPClient {
  constructor(wsUrl) {
    this.wsUrl = wsUrl;
    this.ws = null;
    this.requestId = 1;
    this.pendingRequests = new Map();
  }

  async connect() {
    return new Promise((resolve, reject) => {
      this.ws = new WebSocket(this.wsUrl);
      
      this.ws.onopen = () => {
        console.log('WebSocket connected');
        resolve();
      };
      
      this.ws.onmessage = (event) => {
        this.handleMessage(JSON.parse(event.data));
      };
      
      this.ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        reject(error);
      };
      
      this.ws.onclose = () => {
        console.log('WebSocket disconnected');
      };
    });
  }

  handleMessage(message) {
    if (message.id && this.pendingRequests.has(message.id)) {
      const { resolve, reject } = this.pendingRequests.get(message.id);
      this.pendingRequests.delete(message.id);
      
      if (message.error) {
        reject(new Error(message.error.message));
      } else {
        resolve(message.result);
      }
    }
  }

  async sendRequest(method, params = {}) {
    const id = this.requestId++;
    const message = {
      jsonrpc: '2.0',
      id,
      method,
      params
    };

    return new Promise((resolve, reject) => {
      this.pendingRequests.set(id, { resolve, reject });
      
      if (this.ws.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify(message));
      } else {
        this.pendingRequests.delete(id);
        reject(new Error('WebSocket not connected'));
      }
    });
  }

  async listTools() {
    return await this.sendRequest('tools/list');
  }

  async callTool(name, arguments) {
    return await this.sendRequest('tools/call', { name, arguments });
  }

  disconnect() {
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }
}
```

## Building a Web Interface

### HTML Structure
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MCP Web Client</title>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        
        .container {
            background: white;
            border-radius: 8px;
            padding: 24px;
            box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
        }
        
        .tool-card {
            border: 1px solid #e1e5e9;
            border-radius: 6px;
            padding: 16px;
            margin: 12px 0;
            background: #fafbfc;
        }
        
        .tool-input {
            width: 100%;
            padding: 8px 12px;
            border: 1px solid #d0d7de;
            border-radius: 4px;
            margin: 8px 0;
        }
        
        .btn {
            background-color: #0969da;
            color: white;
            border: none;
            padding: 8px 16px;
            border-radius: 4px;
            cursor: pointer;
            font-size: 14px;
        }
        
        .btn:hover {
            background-color: #0550ae;
        }
        
        .result {
            background: #f6f8fa;
            border: 1px solid #d0d7de;
            border-radius: 4px;
            padding: 12px;
            margin: 12px 0;
            white-space: pre-wrap;
            font-family: 'SF Mono', Monaco, 'Cascadia Code', monospace;
        }
        
        .status {
            padding: 8px 12px;
            border-radius: 4px;
            margin: 8px 0;
            font-weight: 500;
        }
        
        .status.connected {
            background-color: #dafbe1;
            color: #1a7f37;
        }
        
        .status.disconnected {
            background-color: #ffebe9;
            color: #cf222e;
        }
        
        .loading {
            opacity: 0.6;
            pointer-events: none;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>MCP Web Client</h1>
        
        <div class="connection-section">
            <h2>Connection</h2>
            <input type="text" id="serverUrl" placeholder="http://localhost:8000/mcp" class="tool-input">
            <button id="connectBtn" class="btn">Connect</button>
            <button id="disconnectBtn" class="btn" style="display: none;">Disconnect</button>
            <div id="connectionStatus" class="status disconnected">Disconnected</div>
        </div>
        
        <div class="tools-section" style="display: none;">
            <h2>Available Tools</h2>
            <div id="toolsList"></div>
        </div>
        
        <div class="results-section">
            <h2>Results</h2>
            <div id="results"></div>
        </div>
    </div>

    <script type="module" src="app.js"></script>
</body>
</html>
```

### Application Logic
```javascript
// app.js
class MCPWebClient {
  constructor() {
    this.client = null;
    this.isConnected = false;
    this.tools = [];
    
    this.initializeEventListeners();
  }

  initializeEventListeners() {
    document.getElementById('connectBtn').addEventListener('click', () => {
      this.connect();
    });
    
    document.getElementById('disconnectBtn').addEventListener('click', () => {
      this.disconnect();
    });
  }

  async connect() {
    const serverUrl = document.getElementById('serverUrl').value.trim();
    if (!serverUrl) {
      this.addResult('Error: Please enter a server URL', 'error');
      return;
    }

    try {
      this.setLoading(true);
      
      // Use fetch-based HTTP transport for browser compatibility
      this.client = new HTTPMCPClient(serverUrl);
      await this.client.connect();
      
      this.isConnected = true;
      this.updateConnectionStatus(true);
      
      // Load available tools
      await this.loadTools();
      
      this.addResult('Connected successfully!', 'success');
    } catch (error) {
      this.addResult(`Connection failed: ${error.message}`, 'error');
      this.updateConnectionStatus(false);
    } finally {
      this.setLoading(false);
    }
  }

  async disconnect() {
    if (this.client) {
      await this.client.disconnect();
      this.client = null;
    }
    
    this.isConnected = false;
    this.updateConnectionStatus(false);
    this.clearTools();
    this.addResult('Disconnected', 'info');
  }

  async loadTools() {
    try {
      this.tools = await this.client.listTools();
      this.renderTools();
    } catch (error) {
      this.addResult(`Failed to load tools: ${error.message}`, 'error');
    }
  }

  renderTools() {
    const toolsList = document.getElementById('toolsList');
    toolsList.innerHTML = '';

    if (this.tools.length === 0) {
      toolsList.innerHTML = '<p>No tools available</p>';
      return;
    }

    this.tools.forEach(tool => {
      const toolCard = this.createToolCard(tool);
      toolsList.appendChild(toolCard);
    });

    document.querySelector('.tools-section').style.display = 'block';
  }

  createToolCard(tool) {
    const card = document.createElement('div');
    card.className = 'tool-card';
    
    card.innerHTML = `
      <h3>${tool.name}</h3>
      <p>${tool.description}</p>
      <div class="tool-inputs" id="inputs-${tool.name}"></div>
      <button class="btn" onclick="mcpClient.callTool('${tool.name}')">
        Call Tool
      </button>
    `;

    // Generate input fields based on schema
    const inputsContainer = card.querySelector(`#inputs-${tool.name}`);
    this.generateInputFields(tool.inputSchema, inputsContainer, tool.name);

    return card;
  }

  generateInputFields(schema, container, toolName) {
    if (!schema || !schema.properties) {
      return;
    }

    Object.entries(schema.properties).forEach(([propName, propSchema]) => {
      const inputGroup = document.createElement('div');
      
      const label = document.createElement('label');
      label.textContent = propName + (schema.required?.includes(propName) ? ' *' : '');
      label.style.display = 'block';
      label.style.marginBottom = '4px';
      label.style.fontWeight = '500';
      
      const input = document.createElement('input');
      input.className = 'tool-input';
      input.id = `${toolName}-${propName}`;
      input.placeholder = propSchema.description || '';
      
      // Set input type based on schema
      if (propSchema.type === 'number') {
        input.type = 'number';
      } else if (propSchema.type === 'boolean') {
        input.type = 'checkbox';
      } else if (propSchema.enum) {
        const select = document.createElement('select');
        select.className = 'tool-input';
        select.id = `${toolName}-${propName}`;
        
        propSchema.enum.forEach(value => {
          const option = document.createElement('option');
          option.value = value;
          option.textContent = value;
          select.appendChild(option);
        });
        
        inputGroup.appendChild(label);
        inputGroup.appendChild(select);
        container.appendChild(inputGroup);
        return;
      } else {
        input.type = 'text';
      }

      inputGroup.appendChild(label);
      inputGroup.appendChild(input);
      container.appendChild(inputGroup);
    });
  }

  async callTool(toolName) {
    const tool = this.tools.find(t => t.name === toolName);
    if (!tool) {
      this.addResult(`Tool not found: ${toolName}`, 'error');
      return;
    }

    // Collect input values
    const arguments = {};
    const schema = tool.inputSchema;
    
    if (schema && schema.properties) {
      Object.keys(schema.properties).forEach(propName => {
        const input = document.getElementById(`${toolName}-${propName}`);
        if (input) {
          if (input.type === 'checkbox') {
            arguments[propName] = input.checked;
          } else if (input.type === 'number') {
            arguments[propName] = parseFloat(input.value) || 0;
          } else {
            arguments[propName] = input.value;
          }
        }
      });
    }

    try {
      this.setLoading(true);
      this.addResult(`Calling tool: ${toolName}`, 'info');
      
      const result = await this.client.callTool(toolName, arguments);
      
      // Display result
      const resultText = result.map(item => {
        if (item.type === 'text') {
          return item.text;
        } else {
          return JSON.stringify(item, null, 2);
        }
      }).join('\n\n');
      
      this.addResult(resultText, 'success');
    } catch (error) {
      this.addResult(`Tool call failed: ${error.message}`, 'error');
    } finally {
      this.setLoading(false);
    }
  }

  updateConnectionStatus(connected) {
    const status = document.getElementById('connectionStatus');
    const connectBtn = document.getElementById('connectBtn');
    const disconnectBtn = document.getElementById('disconnectBtn');
    
    if (connected) {
      status.textContent = 'Connected';
      status.className = 'status connected';
      connectBtn.style.display = 'none';
      disconnectBtn.style.display = 'inline-block';
    } else {
      status.textContent = 'Disconnected';
      status.className = 'status disconnected';
      connectBtn.style.display = 'inline-block';
      disconnectBtn.style.display = 'none';
    }
  }

  clearTools() {
    document.getElementById('toolsList').innerHTML = '';
    document.querySelector('.tools-section').style.display = 'none';
    this.tools = [];
  }

  addResult(message, type = 'info') {
    const results = document.getElementById('results');
    const resultDiv = document.createElement('div');
    resultDiv.className = 'result';
    
    const timestamp = new Date().toLocaleTimeString();
    resultDiv.innerHTML = `
      <strong>[${timestamp}] ${type.toUpperCase()}:</strong><br>
      ${message}
    `;
    
    // Color code by type
    if (type === 'error') {
      resultDiv.style.borderColor = '#d73a49';
      resultDiv.style.backgroundColor = '#ffeaea';
    } else if (type === 'success') {
      resultDiv.style.borderColor = '#28a745';
      resultDiv.style.backgroundColor = '#eaffea';
    } else if (type === 'info') {
      resultDiv.style.borderColor = '#0366d6';
      resultDiv.style.backgroundColor = '#eaf3ff';
    }
    
    results.prepend(resultDiv);
    
    // Limit number of results displayed
    while (results.children.length > 10) {
      results.removeChild(results.lastChild);
    }
  }

  setLoading(loading) {
    document.body.classList.toggle('loading', loading);
  }
}

// HTTP-based MCP client for browsers
class HTTPMCPClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl.replace(/\/$/, '');
    this.connected = false;
  }

  async connect() {
    // Test connection with a simple request
    try {
      const response = await fetch(`${this.baseUrl}`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          jsonrpc: '2.0',
          method: 'initialize',
          id: 1,
          params: {
            protocolVersion: '2025-06-18',
            capabilities: {},
            clientInfo: {
              name: 'browser-mcp-client',
              version: '1.0.0'
            }
          }
        })
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      this.connected = true;
    } catch (error) {
      throw new Error(`Connection failed: ${error.message}`);
    }
  }

  async sendRequest(method, params = {}) {
    if (!this.connected) {
      throw new Error('Not connected to server');
    }

    const response = await fetch(`${this.baseUrl}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method,
        params,
        id: Date.now()
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const data = await response.json();
    
    if (data.error) {
      throw new Error(data.error.message || 'Server error');
    }

    return data.result;
  }

  async listTools() {
    const result = await this.sendRequest('tools/list');
    return result.tools || [];
  }

  async callTool(name, arguments) {
    const result = await this.sendRequest('tools/call', { name, arguments });
    return result.content || [];
  }

  async disconnect() {
    this.connected = false;
  }
}

// Initialize the application
const mcpClient = new MCPWebClient();
window.mcpClient = mcpClient; // Make available globally for button onclick handlers
```

## Error Handling and User Experience

### Robust Error Handling
```javascript
// error-handler.js
class ErrorHandler {
  static handle(error, context = '') {
    console.error(`Error in ${context}:`, error);
    
    let userMessage = 'An unexpected error occurred';
    
    // Categorize errors for better user experience
    if (error instanceof TypeError && error.message.includes('fetch')) {
      userMessage = 'Network error: Unable to connect to server. Please check the URL and try again.';
    } else if (error.message.includes('HTTP 404')) {
      userMessage = 'Server not found: The MCP server URL appears to be incorrect.';
    } else if (error.message.includes('HTTP 500')) {
      userMessage = 'Server error: The MCP server encountered an internal error.';
    } else if (error.message.includes('timeout')) {
      userMessage = 'Request timeout: The server took too long to respond.';
    } else if (error.message.includes('CORS')) {
      userMessage = 'Cross-origin error: The server needs to allow browser access.';
    } else if (error.message) {
      userMessage = error.message;
    }
    
    return userMessage;
  }

  static showUserError(message, container) {
    const errorDiv = document.createElement('div');
    errorDiv.className = 'error-message';
    errorDiv.style.cssText = `
      background-color: #ffebee;
      color: #c62828;
      padding: 12px 16px;
      border-radius: 4px;
      margin: 8px 0;
      border: 1px solid #ef5350;
    `;
    errorDiv.textContent = message;
    
    container.prepend(errorDiv);
    
    // Auto-remove after 5 seconds
    setTimeout(() => {
      if (errorDiv.parentNode) {
        errorDiv.parentNode.removeChild(errorDiv);
      }
    }, 5000);
  }
}
```

## Progressive Web App Features

### Service Worker for Offline Capability
```javascript
// sw.js (Service Worker)
const CACHE_NAME = 'mcp-client-v1';
const urlsToCache = [
  '/',
  '/app.js',
  '/style.css'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then((cache) => cache.addAll(urlsToCache))
  );
});

self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request)
      .then((response) => {
        // Return cached version or fetch from network
        return response || fetch(event.request);
      })
  );
});
```

### Web App Manifest
```json
{
  "name": "MCP Web Client",
  "short_name": "MCP Client",
  "description": "Browser-based Model Context Protocol client",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#0969da",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icon-512.png", 
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

## Build Process

### Webpack Configuration
```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');

module.exports = {
  entry: './src/app.js',
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'dist'),
    clean: true,
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
          options: {
            presets: ['@babel/preset-env']
          }
        }
      },
      {
        test: /\.css$/i,
        use: ['style-loader', 'css-loader'],
      }
    ],
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './src/index.html',
    }),
  ],
  devServer: {
    static: './dist',
    port: 3000,
    open: true,
  },
  resolve: {
    fallback: {
      "buffer": require.resolve("buffer/"),
      "process": require.resolve("process/browser")
    }
  }
};
```

## Security Considerations

### Content Security Policy
```html
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; 
               connect-src 'self' http://localhost:8000 https://your-mcp-server.com;
               script-src 'self' 'unsafe-inline';
               style-src 'self' 'unsafe-inline';">
```

### CORS Handling
```javascript
// For MCP servers that need to support browser clients
// Add CORS headers to server responses:

// Access-Control-Allow-Origin: https://your-client-domain.com
// Access-Control-Allow-Methods: POST, OPTIONS
// Access-Control-Allow-Headers: Content-Type, Authorization
// Access-Control-Max-Age: 86400
```

## Best Practices

### Performance
1. **Lazy Loading**: Load tools and features on demand
2. **Caching**: Cache server responses when appropriate
3. **Debouncing**: Debounce user inputs to reduce server requests
4. **Virtual Scrolling**: For large lists of tools or results

### User Experience  
1. **Loading States**: Show loading indicators for all async operations
2. **Error Recovery**: Provide clear error messages and recovery options
3. **Offline Support**: Cache resources for offline functionality
4. **Responsive Design**: Support mobile and desktop interfaces

### Development
1. **Code Splitting**: Split code into chunks for better loading performance
2. **Testing**: Write unit tests for client logic
3. **Documentation**: Document API endpoints and tool schemas
4. **Monitoring**: Track errors and performance metrics

Browser-based MCP clients enable rich, interactive web applications that can leverage the full power of MCP servers while providing excellent user experiences.