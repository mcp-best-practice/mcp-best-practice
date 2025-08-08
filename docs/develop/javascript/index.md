# JavaScript/TypeScript Development

## MCP Server Development with JavaScript

Build MCP servers using JavaScript or TypeScript with the official SDK.

## Quick Start

### Installation
```bash
# npm
npm install @modelcontextprotocol/sdk

# yarn
yarn add @modelcontextprotocol/sdk

# pnpm
pnpm add @modelcontextprotocol/sdk
```

### Basic Server (JavaScript)
```javascript
const { Server } = require('@modelcontextprotocol/sdk');

const server = new Server({
  name: 'my-js-server',
  version: '1.0.0'
});

server.tool('greet', {
  description: 'Greet someone',
  parameters: {
    type: 'object',
    properties: {
      name: { type: 'string' }
    },
    required: ['name']
  }
}, async ({ name }) => {
  return `Hello, ${name}!`;
});

server.start();
```

### TypeScript Server
```typescript
import { Server, Tool } from '@modelcontextprotocol/sdk';

interface GreetParams {
  name: string;
  greeting?: string;
}

const server = new Server({
  name: 'my-ts-server',
  version: '1.0.0'
});

const greetTool: Tool<GreetParams> = {
  name: 'greet',
  description: 'Greet someone',
  parameters: {
    type: 'object',
    properties: {
      name: { type: 'string' },
      greeting: { type: 'string', default: 'Hello' }
    },
    required: ['name']
  },
  handler: async ({ name, greeting = 'Hello' }) => {
    return `${greeting}, ${name}!`;
  }
};

server.registerTool(greetTool);
server.start();
```

## Project Setup

### Package.json
```json
{
  "name": "my-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "test": "jest",
    "lint": "eslint src/**/*.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.0.0",
    "tsx": "^4.0.0",
    "jest": "^29.0.0",
    "@types/jest": "^29.0.0",
    "eslint": "^8.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0"
  }
}
```

### TypeScript Configuration
```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

## Advanced Features

### Async Operations
```typescript
server.tool('fetchData', {
  description: 'Fetch data from API',
  parameters: {
    type: 'object',
    properties: {
      url: { type: 'string', format: 'uri' }
    },
    required: ['url']
  }
}, async ({ url }) => {
  const response = await fetch(url);
  const data = await response.json();
  return JSON.stringify(data, null, 2);
});
```

### Resource Providers
```typescript
server.resource('config', {
  description: 'Server configuration',
  handler: async () => {
    return {
      version: '1.0.0',
      features: ['tools', 'resources'],
      environment: process.env.NODE_ENV
    };
  }
});
```

### Error Handling
```typescript
class McpError extends Error {
  constructor(message: string, public code: number = -32000) {
    super(message);
    this.name = 'McpError';
  }
}

server.tool('riskyOperation', {
  // ... parameters
}, async (params) => {
  try {
    return await performOperation(params);
  } catch (error) {
    if (error instanceof ValidationError) {
      throw new McpError('Invalid input', -32602);
    }
    throw new McpError('Operation failed', -32000);
  }
});
```

## Testing

### Jest Configuration
```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  roots: ['<rootDir>/src'],
  testMatch: ['**/__tests__/**/*.ts', '**/?(*.)+(spec|test).ts'],
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/*.test.ts'
  ]
};
```

### Unit Tests
```typescript
// src/__tests__/tools.test.ts
import { greetTool } from '../tools';

describe('greet tool', () => {
  it('should greet with name', async () => {
    const result = await greetTool.handler({ name: 'Alice' });
    expect(result).toBe('Hello, Alice!');
  });

  it('should use custom greeting', async () => {
    const result = await greetTool.handler({ 
      name: 'Bob', 
      greeting: 'Hi' 
    });
    expect(result).toBe('Hi, Bob!');
  });
});
```

## Deployment

### Docker
```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY dist ./dist

EXPOSE 8000

CMD ["node", "dist/index.js"]
```

### Environment Variables
```typescript
const config = {
  port: process.env.MCP_PORT || 8000,
  apiKey: process.env.MCP_API_KEY,
  debug: process.env.MCP_DEBUG === 'true'
};

if (!config.apiKey) {
  throw new Error('MCP_API_KEY is required');
}
```

## Performance Optimization

### Connection Pooling
```typescript
import { Pool } from 'pg';

const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

server.tool('queryDatabase', {
  // parameters...
}, async ({ query }) => {
  const client = await pool.connect();
  try {
    const result = await client.query(query);
    return result.rows;
  } finally {
    client.release();
  }
});
```

### Caching
```typescript
const cache = new Map<string, { data: any; expires: number }>();

function cached<T>(key: string, ttl: number, fn: () => Promise<T>): Promise<T> {
  const now = Date.now();
  const cached = cache.get(key);
  
  if (cached && cached.expires > now) {
    return Promise.resolve(cached.data);
  }
  
  return fn().then(data => {
    cache.set(key, { data, expires: now + ttl });
    return data;
  });
}
```

## Common Patterns

### Middleware Pattern
```typescript
type Middleware = (context: any, next: () => Promise<any>) => Promise<any>;

class MiddlewareServer extends Server {
  private middlewares: Middleware[] = [];
  
  use(middleware: Middleware) {
    this.middlewares.push(middleware);
  }
  
  async execute(context: any, handler: () => Promise<any>) {
    let index = 0;
    
    const next = async (): Promise<any> => {
      if (index >= this.middlewares.length) {
        return handler();
      }
      
      const middleware = this.middlewares[index++];
      return middleware(context, next);
    };
    
    return next();
  }
}
```

## Next Steps

- 🔷 [TypeScript Best Practices](typescript.md)
- 🚀 [Node.js Specifics](nodejs.md)
- 🌐 [Browser Integration](browser.md)
- 🧪 [Testing Guide](testing.md)
- 📦 [Packaging & Distribution](packaging.md)