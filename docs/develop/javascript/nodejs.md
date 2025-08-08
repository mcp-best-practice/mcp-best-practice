# Node.js Development

## Building MCP Servers with Node.js

Node.js provides an excellent runtime for MCP servers with its strong async capabilities, rich ecosystem, and native JSON handling.

## Node.js Setup

### Project Initialization
```bash
# Initialize new Node.js project
npm init -y

# Install MCP SDK
npm install @modelcontextprotocol/sdk

# Install development dependencies
npm install --save-dev nodemon dotenv
```

### Package Configuration
```json
{
  "name": "nodejs-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "main": "src/server.js",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "test": "node --test src/**/*.test.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.1.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0",
    "dotenv": "^16.3.0"
  }
}
```

## Basic Node.js MCP Server

### Simple Server Implementation
```javascript
// src/server.js
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema
} from '@modelcontextprotocol/sdk/types.js';

class NodeMCPServer {
  constructor(name, version) {
    this.server = new Server(
      { name, version },
      {
        capabilities: {
          tools: {}
        }
      }
    );

    this.setupToolHandlers();
  }

  setupToolHandlers() {
    // List available tools
    this.server.setRequestHandler(ListToolsRequestSchema, async () => {
      return {
        tools: [
          {
            name: 'file_read',
            description: 'Read contents of a file',
            inputSchema: {
              type: 'object',
              properties: {
                path: {
                  type: 'string',
                  description: 'Path to the file to read'
                },
                encoding: {
                  type: 'string',
                  enum: ['utf8', 'base64', 'hex'],
                  default: 'utf8',
                  description: 'File encoding'
                }
              },
              required: ['path']
            }
          },
          {
            name: 'system_info',
            description: 'Get system information',
            inputSchema: {
              type: 'object',
              properties: {
                info_type: {
                  type: 'string',
                  enum: ['platform', 'memory', 'cpu', 'network'],
                  description: 'Type of system information to retrieve'
                }
              },
              required: ['info_type']
            }
          }
        ]
      };
    });

    // Handle tool calls
    this.server.setRequestHandler(CallToolRequestSchema, async (request) => {
      const { name, arguments: args } = request.params;

      try {
        switch (name) {
          case 'file_read':
            return await this.handleFileRead(args);
          case 'system_info':
            return await this.handleSystemInfo(args);
          default:
            throw new Error(`Unknown tool: ${name}`);
        }
      } catch (error) {
        return {
          content: [{
            type: 'text',
            text: `Error: ${error.message}`
          }]
        };
      }
    });
  }

  async handleFileRead(args) {
    const { path, encoding = 'utf8' } = args;
    
    // Input validation
    if (!path || typeof path !== 'string') {
      throw new Error('Path is required and must be a string');
    }

    try {
      const fs = await import('fs/promises');
      const content = await fs.readFile(path, encoding);
      
      return {
        content: [{
          type: 'text',
          text: `File contents (${path}):\n${content}`
        }]
      };
    } catch (error) {
      if (error.code === 'ENOENT') {
        throw new Error(`File not found: ${path}`);
      } else if (error.code === 'EACCES') {
        throw new Error(`Permission denied: ${path}`);
      }
      throw error;
    }
  }

  async handleSystemInfo(args) {
    const { info_type } = args;
    const os = await import('os');
    
    let info;
    
    switch (info_type) {
      case 'platform':
        info = {
          platform: os.platform(),
          arch: os.arch(),
          release: os.release(),
          hostname: os.hostname()
        };
        break;
        
      case 'memory':
        info = {
          total_memory: os.totalmem(),
          free_memory: os.freemem(),
          memory_usage: process.memoryUsage()
        };
        break;
        
      case 'cpu':
        info = {
          cpu_count: os.cpus().length,
          cpu_model: os.cpus()[0]?.model || 'Unknown',
          load_average: os.loadavg(),
          uptime: os.uptime()
        };
        break;
        
      case 'network':
        info = {
          network_interfaces: os.networkInterfaces()
        };
        break;
        
      default:
        throw new Error(`Unknown info type: ${info_type}`);
    }

    return {
      content: [{
        type: 'text',
        text: JSON.stringify(info, null, 2)
      }]
    };
  }

  async run() {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
  }
}

// Main execution
async function main() {
  const server = new NodeMCPServer('nodejs-mcp-server', '1.0.0');
  await server.run();
}

// Handle process signals
process.on('SIGINT', () => {
  console.log('Received SIGINT, shutting down gracefully...');
  process.exit(0);
});

process.on('SIGTERM', () => {
  console.log('Received SIGTERM, shutting down gracefully...');
  process.exit(0);
});

if (import.meta.url === `file://${process.argv[1]}`) {
  main().catch(console.error);
}
```

## File System Operations

### Advanced File Handling
```javascript
// src/tools/filesystem.js
import { promises as fs } from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';

export class FileSystemTools {
  constructor(allowedPaths = []) {
    this.allowedPaths = allowedPaths;
  }

  validatePath(filePath) {
    const resolvedPath = path.resolve(filePath);
    
    // Check if path is within allowed directories
    if (this.allowedPaths.length > 0) {
      const isAllowed = this.allowedPaths.some(allowedPath => 
        resolvedPath.startsWith(path.resolve(allowedPath))
      );
      
      if (!isAllowed) {
        throw new Error(`Access denied: Path not in allowed directories`);
      }
    }
    
    return resolvedPath;
  }

  async readFile(filePath, encoding = 'utf8') {
    const validPath = this.validatePath(filePath);
    
    try {
      const content = await fs.readFile(validPath, encoding);
      return {
        path: validPath,
        size: (await fs.stat(validPath)).size,
        content: content
      };
    } catch (error) {
      this.handleFileError(error, validPath);
    }
  }

  async writeFile(filePath, content, options = {}) {
    const validPath = this.validatePath(filePath);
    
    try {
      await fs.writeFile(validPath, content, {
        encoding: 'utf8',
        ...options
      });
      
      const stats = await fs.stat(validPath);
      return {
        path: validPath,
        size: stats.size,
        modified: stats.mtime
      };
    } catch (error) {
      this.handleFileError(error, validPath);
    }
  }

  async listDirectory(dirPath) {
    const validPath = this.validatePath(dirPath);
    
    try {
      const entries = await fs.readdir(validPath, { withFileTypes: true });
      
      return entries.map(entry => ({
        name: entry.name,
        type: entry.isDirectory() ? 'directory' : 'file',
        path: path.join(validPath, entry.name)
      }));
    } catch (error) {
      this.handleFileError(error, validPath);
    }
  }

  async getFileStats(filePath) {
    const validPath = this.validatePath(filePath);
    
    try {
      const stats = await fs.stat(validPath);
      
      return {
        path: validPath,
        size: stats.size,
        created: stats.birthtime,
        modified: stats.mtime,
        accessed: stats.atime,
        isDirectory: stats.isDirectory(),
        isFile: stats.isFile(),
        permissions: stats.mode
      };
    } catch (error) {
      this.handleFileError(error, validPath);
    }
  }

  handleFileError(error, filePath) {
    switch (error.code) {
      case 'ENOENT':
        throw new Error(`File or directory not found: ${filePath}`);
      case 'EACCES':
        throw new Error(`Permission denied: ${filePath}`);
      case 'EISDIR':
        throw new Error(`Expected file but found directory: ${filePath}`);
      case 'ENOTDIR':
        throw new Error(`Expected directory but found file: ${filePath}`);
      default:
        throw error;
    }
  }
}
```

## HTTP Client Integration

### REST API Tools
```javascript
// src/tools/http-client.js
export class HttpClient {
  constructor(options = {}) {
    this.defaultTimeout = options.timeout || 30000;
    this.defaultHeaders = options.headers || {};
    this.retryCount = options.retryCount || 3;
    this.retryDelay = options.retryDelay || 1000;
  }

  async request(url, options = {}) {
    const requestOptions = {
      method: 'GET',
      timeout: this.defaultTimeout,
      headers: { ...this.defaultHeaders },
      ...options
    };

    let lastError;
    
    for (let attempt = 1; attempt <= this.retryCount; attempt++) {
      try {
        const response = await this.performRequest(url, requestOptions);
        return response;
      } catch (error) {
        lastError = error;
        
        if (attempt < this.retryCount && this.shouldRetry(error)) {
          await this.delay(this.retryDelay * attempt);
          continue;
        }
        
        break;
      }
    }
    
    throw lastError;
  }

  async performRequest(url, options) {
    // Use dynamic import for better compatibility
    const { default: fetch } = await import('node-fetch');
    
    const controller = new AbortController();
    const timeoutId = setTimeout(() => controller.abort(), options.timeout);
    
    try {
      const response = await fetch(url, {
        ...options,
        signal: controller.signal
      });
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }
      
      const contentType = response.headers.get('content-type');
      let data;
      
      if (contentType?.includes('application/json')) {
        data = await response.json();
      } else {
        data = await response.text();
      }
      
      return {
        status: response.status,
        statusText: response.statusText,
        headers: Object.fromEntries(response.headers.entries()),
        data: data
      };
    } finally {
      clearTimeout(timeoutId);
    }
  }

  shouldRetry(error) {
    // Retry on network errors, timeouts, and 5xx status codes
    return (
      error.name === 'AbortError' ||
      error.code === 'ECONNRESET' ||
      error.code === 'ETIMEDOUT' ||
      (error.message && error.message.includes('HTTP 5'))
    );
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  async get(url, options = {}) {
    return this.request(url, { ...options, method: 'GET' });
  }

  async post(url, data, options = {}) {
    return this.request(url, {
      ...options,
      method: 'POST',
      body: JSON.stringify(data),
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      }
    });
  }

  async put(url, data, options = {}) {
    return this.request(url, {
      ...options,
      method: 'PUT',
      body: JSON.stringify(data),
      headers: {
        'Content-Type': 'application/json',
        ...options.headers
      }
    });
  }

  async delete(url, options = {}) {
    return this.request(url, { ...options, method: 'DELETE' });
  }
}
```

## Configuration Management

### Environment-Based Configuration
```javascript
// src/config.js
import { config } from 'dotenv';
import path from 'path';

// Load environment variables
config();

export class Config {
  constructor() {
    this.server = {
      name: process.env.MCP_SERVER_NAME || 'nodejs-mcp-server',
      version: process.env.MCP_SERVER_VERSION || '1.0.0',
      debug: process.env.MCP_DEBUG === 'true'
    };

    this.filesystem = {
      allowedPaths: process.env.ALLOWED_PATHS 
        ? process.env.ALLOWED_PATHS.split(',').map(p => p.trim())
        : []
    };

    this.http = {
      timeout: parseInt(process.env.HTTP_TIMEOUT) || 30000,
      retryCount: parseInt(process.env.HTTP_RETRY_COUNT) || 3,
      retryDelay: parseInt(process.env.HTTP_RETRY_DELAY) || 1000,
      userAgent: process.env.HTTP_USER_AGENT || 'MCP-Server/1.0'
    };

    this.database = {
      url: process.env.DATABASE_URL,
      poolSize: parseInt(process.env.DB_POOL_SIZE) || 10,
      timeout: parseInt(process.env.DB_TIMEOUT) || 30000
    };

    this.logging = {
      level: process.env.LOG_LEVEL || 'info',
      format: process.env.LOG_FORMAT || 'json'
    };
  }

  validate() {
    const errors = [];

    // Validate required configuration
    if (this.filesystem.allowedPaths.length === 0) {
      console.warn('Warning: No allowed paths configured for filesystem access');
    }

    if (this.database.url && !this.isValidUrl(this.database.url)) {
      errors.push('Invalid database URL');
    }

    if (errors.length > 0) {
      throw new Error(`Configuration validation failed: ${errors.join(', ')}`);
    }
  }

  isValidUrl(string) {
    try {
      new URL(string);
      return true;
    } catch {
      return false;
    }
  }
}

export const config = new Config();
```

## Database Integration

### Database Connection Pool
```javascript
// src/database.js
export class DatabaseConnection {
  constructor(config) {
    this.config = config;
    this.pool = null;
  }

  async connect() {
    if (this.pool) {
      return this.pool;
    }

    // Example with a hypothetical database driver
    try {
      const { createPool } = await import('generic-database-driver');
      
      this.pool = createPool({
        connectionString: this.config.database.url,
        max: this.config.database.poolSize,
        idleTimeoutMillis: this.config.database.timeout,
        ssl: process.env.NODE_ENV === 'production'
      });

      // Test connection
      const client = await this.pool.connect();
      await client.query('SELECT 1');
      client.release();

      console.log('Database connected successfully');
      return this.pool;
    } catch (error) {
      console.error('Failed to connect to database:', error.message);
      throw error;
    }
  }

  async query(text, params = []) {
    if (!this.pool) {
      await this.connect();
    }

    const client = await this.pool.connect();
    
    try {
      const result = await client.query(text, params);
      return result;
    } finally {
      client.release();
    }
  }

  async transaction(callback) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      const result = await callback(client);
      await client.query('COMMIT');
      return result;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async close() {
    if (this.pool) {
      await this.pool.end();
      this.pool = null;
    }
  }
}
```

## Logging and Monitoring

### Structured Logging
```javascript
// src/logger.js
export class Logger {
  constructor(options = {}) {
    this.level = options.level || 'info';
    this.format = options.format || 'json';
    this.levels = {
      debug: 0,
      info: 1,
      warn: 2,
      error: 3
    };
  }

  shouldLog(level) {
    return this.levels[level] >= this.levels[this.level];
  }

  formatMessage(level, message, meta = {}) {
    const timestamp = new Date().toISOString();
    
    if (this.format === 'json') {
      return JSON.stringify({
        timestamp,
        level: level.toUpperCase(),
        message,
        ...meta,
        pid: process.pid,
        hostname: require('os').hostname()
      });
    } else {
      const metaStr = Object.keys(meta).length > 0 
        ? ` ${JSON.stringify(meta)}` 
        : '';
      return `${timestamp} [${level.toUpperCase()}] ${message}${metaStr}`;
    }
  }

  log(level, message, meta) {
    if (!this.shouldLog(level)) {
      return;
    }

    const formatted = this.formatMessage(level, message, meta);
    
    if (level === 'error') {
      console.error(formatted);
    } else {
      console.log(formatted);
    }
  }

  debug(message, meta) {
    this.log('debug', message, meta);
  }

  info(message, meta) {
    this.log('info', message, meta);
  }

  warn(message, meta) {
    this.log('warn', message, meta);
  }

  error(message, meta) {
    this.log('error', message, meta);
  }
}

export const logger = new Logger({
  level: process.env.LOG_LEVEL || 'info',
  format: process.env.LOG_FORMAT || 'json'
});
```

## Testing

### Unit Tests with Node.js Test Runner
```javascript
// src/server.test.js
import { describe, it, before, after } from 'node:test';
import assert from 'node:assert';
import { NodeMCPServer } from './server.js';

describe('NodeMCPServer', () => {
  let server;

  before(async () => {
    server = new NodeMCPServer('test-server', '1.0.0');
  });

  it('should handle file read correctly', async () => {
    const result = await server.handleFileRead({
      path: './package.json',
      encoding: 'utf8'
    });

    assert(result.content);
    assert(Array.isArray(result.content));
    assert(result.content[0].type === 'text');
    assert(result.content[0].text.includes('package.json'));
  });

  it('should handle system info requests', async () => {
    const result = await server.handleSystemInfo({
      info_type: 'platform'
    });

    assert(result.content);
    assert(result.content[0].type === 'text');
    
    const info = JSON.parse(result.content[0].text);
    assert(typeof info.platform === 'string');
    assert(typeof info.arch === 'string');
  });

  it('should throw error for invalid file paths', async () => {
    await assert.rejects(
      server.handleFileRead({ path: '/nonexistent/file.txt' }),
      /File not found/
    );
  });
});
```

## Production Deployment

### Process Management
```javascript
// src/cluster.js
import cluster from 'cluster';
import os from 'os';
import { logger } from './logger.js';

const numCPUs = os.cpus().length;

if (cluster.isPrimary) {
  logger.info(`Master ${process.pid} is running`);

  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    logger.warn(`Worker ${worker.process.pid} died`, { code, signal });
    logger.info('Starting a new worker');
    cluster.fork();
  });
} else {
  // Workers can share any TCP port
  const { NodeMCPServer } = await import('./server.js');
  const server = new NodeMCPServer('nodejs-mcp-server', '1.0.0');
  
  await server.run();
  logger.info(`Worker ${process.pid} started`);
}
```

## Best Practices

### Performance Optimization
1. **Event Loop**: Keep the event loop responsive
2. **Memory Management**: Monitor and prevent memory leaks  
3. **Streaming**: Use streams for large data processing
4. **Connection Pooling**: Reuse database and HTTP connections

### Error Handling
1. **Graceful Degradation**: Handle errors without crashing
2. **Process Signals**: Handle SIGINT and SIGTERM properly
3. **Uncaught Exceptions**: Log and handle uncaught errors
4. **Promise Rejections**: Always handle promise rejections

### Security
1. **Input Validation**: Validate all inputs thoroughly
2. **Path Traversal**: Prevent directory traversal attacks
3. **Environment Variables**: Never log sensitive data
4. **Dependencies**: Regularly update dependencies

Node.js provides excellent performance and developer experience for building scalable MCP servers with rich ecosystem support.