# TypeScript Development

## Using TypeScript for MCP Servers

TypeScript provides excellent type safety and developer experience for building MCP servers, with strong support for JSON Schema validation and async programming patterns.

## TypeScript Setup

### Project Initialization
```bash
# Initialize new TypeScript project
npm init -y
npm install -D typescript @types/node ts-node

# Create TypeScript configuration
npx tsc --init
```

### TypeScript Configuration
```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "node",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noEmitOnError": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

## Basic TypeScript MCP Server

### Type-Safe Server Implementation
```typescript
// src/server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
  Tool,
  TextContent
} from '@modelcontextprotocol/sdk/types.js';
import { z } from 'zod';

// Input validation schemas
const EchoToolInputSchema = z.object({
  text: z.string().min(1, "Text cannot be empty")
});

const CalculatorToolInputSchema = z.object({
  operation: z.enum(['add', 'subtract', 'multiply', 'divide']),
  a: z.number(),
  b: z.number()
});

// Type definitions
type EchoToolInput = z.infer<typeof EchoToolInputSchema>;
type CalculatorToolInput = z.infer<typeof CalculatorToolInputSchema>;

class MCPServer {
  private server: Server;

  constructor(name: string, version: string) {
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

  private setupToolHandlers(): void {
    // List available tools
    this.server.setRequestHandler(
      ListToolsRequestSchema,
      async (): Promise<{ tools: Tool[] }> => {
        return {
          tools: [
            {
              name: 'echo',
              description: 'Echo back the provided text',
              inputSchema: {
                type: 'object',
                properties: {
                  text: {
                    type: 'string',
                    description: 'Text to echo back'
                  }
                },
                required: ['text']
              }
            },
            {
              name: 'calculator',
              description: 'Perform basic arithmetic operations',
              inputSchema: {
                type: 'object',
                properties: {
                  operation: {
                    type: 'string',
                    enum: ['add', 'subtract', 'multiply', 'divide'],
                    description: 'The operation to perform'
                  },
                  a: {
                    type: 'number',
                    description: 'First number'
                  },
                  b: {
                    type: 'number', 
                    description: 'Second number'
                  }
                },
                required: ['operation', 'a', 'b']
              }
            }
          ]
        };
      }
    );

    // Handle tool calls
    this.server.setRequestHandler(
      CallToolRequestSchema,
      async (request): Promise<{ content: TextContent[] }> => {
        const { name, arguments: args } = request.params;

        try {
          switch (name) {
            case 'echo':
              return await this.handleEchoTool(args);
            case 'calculator':
              return await this.handleCalculatorTool(args);
            default:
              throw new Error(`Unknown tool: ${name}`);
          }
        } catch (error) {
          const errorMessage = error instanceof Error 
            ? error.message 
            : 'An unknown error occurred';
          
          return {
            content: [{
              type: 'text',
              text: `Error: ${errorMessage}`
            }]
          };
        }
      }
    );
  }

  private async handleEchoTool(args: unknown): Promise<{ content: TextContent[] }> {
    const input = EchoToolInputSchema.parse(args);
    
    return {
      content: [{
        type: 'text',
        text: `Echo: ${input.text}`
      }]
    };
  }

  private async handleCalculatorTool(args: unknown): Promise<{ content: TextContent[] }> {
    const input = CalculatorToolInputSchema.parse(args);
    
    let result: number;
    
    switch (input.operation) {
      case 'add':
        result = input.a + input.b;
        break;
      case 'subtract':
        result = input.a - input.b;
        break;
      case 'multiply':
        result = input.a * input.b;
        break;
      case 'divide':
        if (input.b === 0) {
          throw new Error('Division by zero is not allowed');
        }
        result = input.a / input.b;
        break;
    }

    return {
      content: [{
        type: 'text',
        text: `Result: ${input.a} ${input.operation} ${input.b} = ${result}`
      }]
    };
  }

  async run(): Promise<void> {
    const transport = new StdioServerTransport();
    await this.server.connect(transport);
  }
}

// Main execution
async function main(): Promise<void> {
  const server = new MCPServer('typescript-mcp-server', '1.0.0');
  await server.run();
}

if (import.meta.url === `file://${process.argv[1]}`) {
  main().catch(console.error);
}
```

## Advanced TypeScript Patterns

### Generic Tool Handler
```typescript
// src/types.ts
import { z } from 'zod';
import { TextContent } from '@modelcontextprotocol/sdk/types.js';

// Generic tool interface
export interface ToolHandler<TInput, TOutput = TextContent[]> {
  name: string;
  description: string;
  inputSchema: z.ZodSchema<TInput>;
  handler: (input: TInput) => Promise<{ content: TOutput }>;
}

// Tool factory function
export function createTool<TInput>(
  config: {
    name: string;
    description: string;
    inputSchema: z.ZodSchema<TInput>;
    handler: (input: TInput) => Promise<{ content: TextContent[] }>;
  }
): ToolHandler<TInput> {
  return config;
}

// Example usage
const echoTool = createTool({
  name: 'echo',
  description: 'Echo back text',
  inputSchema: z.object({
    text: z.string(),
    uppercase: z.boolean().optional()
  }),
  handler: async (input) => {
    const text = input.uppercase ? input.text.toUpperCase() : input.text;
    return {
      content: [{
        type: 'text',
        text: `Echo: ${text}`
      }]
    };
  }
});
```

### Database Integration with Type Safety
```typescript
// src/database.ts
import { z } from 'zod';
import { createTool } from './types.js';

// Database schema validation
const UserSchema = z.object({
  id: z.number(),
  name: z.string(),
  email: z.string().email(),
  created_at: z.string().datetime()
});

type User = z.infer<typeof UserSchema>;

// Mock database interface
interface Database {
  users: {
    findMany(where?: Partial<User>): Promise<User[]>;
    findById(id: number): Promise<User | null>;
    create(data: Omit<User, 'id' | 'created_at'>): Promise<User>;
  };
}

// Database tool implementation
export const queryUsersTool = createTool({
  name: 'query_users',
  description: 'Query users from database',
  inputSchema: z.object({
    search: z.string().optional(),
    limit: z.number().int().positive().max(100).default(10)
  }),
  handler: async (input) => {
    // Type-safe database interaction
    const db: Database = getDatabase();
    
    const users = await db.users.findMany({
      ...(input.search && { name: input.search })
    });

    const limitedUsers = users.slice(0, input.limit);
    
    return {
      content: [{
        type: 'text',
        text: JSON.stringify(limitedUsers, null, 2)
      }]
    };
  }
});

// Mock database implementation
function getDatabase(): Database {
  return {
    users: {
      async findMany(where) {
        // Implementation would connect to real database
        return [];
      },
      async findById(id) {
        return null;
      },
      async create(data) {
        return {
          id: Date.now(),
          created_at: new Date().toISOString(),
          ...data
        };
      }
    }
  };
}
```

## Configuration Management

### Type-Safe Configuration
```typescript
// src/config.ts
import { z } from 'zod';

const ConfigSchema = z.object({
  server: z.object({
    name: z.string().default('typescript-mcp-server'),
    version: z.string().default('1.0.0'),
    debug: z.boolean().default(false)
  }),
  database: z.object({
    url: z.string().url().optional(),
    timeout: z.number().int().positive().default(30000),
    pool_size: z.number().int().positive().default(10)
  }),
  api: z.object({
    base_url: z.string().url().default('https://api.example.com'),
    timeout: z.number().int().positive().default(5000),
    retries: z.number().int().non-negative().default(3)
  })
});

export type Config = z.infer<typeof ConfigSchema>;

export function loadConfig(): Config {
  const rawConfig = {
    server: {
      name: process.env.MCP_SERVER_NAME,
      version: process.env.MCP_SERVER_VERSION,
      debug: process.env.MCP_DEBUG === 'true'
    },
    database: {
      url: process.env.DATABASE_URL,
      timeout: process.env.DATABASE_TIMEOUT ? 
        parseInt(process.env.DATABASE_TIMEOUT) : undefined,
      pool_size: process.env.DATABASE_POOL_SIZE ? 
        parseInt(process.env.DATABASE_POOL_SIZE) : undefined
    },
    api: {
      base_url: process.env.API_BASE_URL,
      timeout: process.env.API_TIMEOUT ? 
        parseInt(process.env.API_TIMEOUT) : undefined,
      retries: process.env.API_RETRIES ? 
        parseInt(process.env.API_RETRIES) : undefined
    }
  };

  return ConfigSchema.parse(rawConfig);
}
```

## Error Handling

### Custom Error Types
```typescript
// src/errors.ts
export abstract class MCPError extends Error {
  abstract readonly code: string;
  
  constructor(message: string, public readonly details?: unknown) {
    super(message);
    this.name = this.constructor.name;
  }
}

export class ValidationError extends MCPError {
  readonly code = 'VALIDATION_ERROR';
  
  constructor(message: string, public readonly field?: string) {
    super(message, { field });
  }
}

export class DatabaseError extends MCPError {
  readonly code = 'DATABASE_ERROR';
  
  constructor(message: string, public readonly query?: string) {
    super(message, { query });
  }
}

export class ExternalAPIError extends MCPError {
  readonly code = 'EXTERNAL_API_ERROR';
  
  constructor(
    message: string, 
    public readonly statusCode?: number,
    public readonly service?: string
  ) {
    super(message, { statusCode, service });
  }
}

// Error handling utility
export function handleError(error: unknown): { content: TextContent[] } {
  if (error instanceof MCPError) {
    return {
      content: [{
        type: 'text',
        text: `Error (${error.code}): ${error.message}`
      }]
    };
  }
  
  if (error instanceof z.ZodError) {
    const issues = error.errors.map(err => 
      `${err.path.join('.')}: ${err.message}`
    ).join(', ');
    
    return {
      content: [{
        type: 'text',
        text: `Validation Error: ${issues}`
      }]
    };
  }
  
  return {
    content: [{
      type: 'text',
      text: 'An unexpected error occurred'
    }]
  };
}
```

## Testing Setup

### TypeScript Test Configuration
```typescript
// src/__tests__/server.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { MCPServer } from '../server.js';

describe('MCPServer', () => {
  let server: MCPServer;
  
  beforeEach(() => {
    server = new MCPServer('test-server', '1.0.0');
  });
  
  it('should handle echo tool correctly', async () => {
    const result = await server.handleEchoTool({ text: 'Hello, World!' });
    
    expect(result.content).toHaveLength(1);
    expect(result.content[0].text).toBe('Echo: Hello, World!');
  });
  
  it('should validate input schemas', async () => {
    await expect(
      server.handleEchoTool({ text: '' })
    ).rejects.toThrow('Text cannot be empty');
  });
  
  it('should handle calculator operations', async () => {
    const result = await server.handleCalculatorTool({
      operation: 'add',
      a: 5,
      b: 3
    });
    
    expect(result.content[0].text).toBe('Result: 5 add 3 = 8');
  });
});
```

## Build and Development Scripts

### Package.json Configuration
```json
{
  "name": "typescript-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "dev": "ts-node --esm src/server.ts",
    "start": "node dist/server.js",
    "test": "vitest",
    "test:coverage": "vitest --coverage",
    "lint": "eslint src/**/*.ts",
    "lint:fix": "eslint src/**/*.ts --fix",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.1.0",
    "zod": "^3.22.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.56.0",
    "ts-node": "^10.9.0",
    "typescript": "^5.3.0",
    "vitest": "^1.0.0",
    "@vitest/coverage-v8": "^1.0.0"
  }
}
```

## Best Practices

### Code Organization
1. **Strict TypeScript**: Use strict mode and exact optional properties
2. **Schema Validation**: Validate all inputs with Zod schemas
3. **Error Handling**: Use custom error types with proper context
4. **Type Safety**: Leverage TypeScript's type system fully

### Performance
1. **Compilation**: Use incremental compilation for faster builds
2. **Tree Shaking**: Structure code for optimal bundling
3. **Memory Management**: Avoid memory leaks in long-running processes

### Development Experience
1. **IDE Support**: Configure proper TypeScript language server settings
2. **Debugging**: Use source maps for better debugging experience
3. **Hot Reload**: Use ts-node or similar for development
4. **Code Quality**: Integrate ESLint and Prettier

TypeScript provides excellent tooling and type safety for building robust MCP servers with confidence in your code's correctness.