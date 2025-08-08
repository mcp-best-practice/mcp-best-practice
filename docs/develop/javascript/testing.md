# JavaScript Testing

## Testing Strategy for JavaScript MCP Applications

Comprehensive testing ensures your MCP servers and clients work reliably across different environments and scenarios.

## Testing Stack

### Core Testing Tools
```bash
# Install testing dependencies
npm install --save-dev \
  jest \
  @jest/globals \
  jest-environment-node \
  supertest \
  nock \
  msw
```

### Jest Configuration
```javascript
// jest.config.js
export default {
  testEnvironment: 'node',
  extensionsToTreatAsEsm: ['.js'],
  globals: {
    'ts-jest': {
      useESM: true
    }
  },
  moduleNameMapping: {
    '^(\\.{1,2}/.*)\\.js$': '$1'
  },
  transform: {
    '^.+\\.js$': ['babel-jest', { presets: [['@babel/preset-env', { targets: { node: 'current' } }]] }]
  },
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/test-helpers/**'
  ],
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  testMatch: [
    '**/src/**/__tests__/**/*.js',
    '**/src/**/*.test.js'
  ],
  setupFilesAfterEnv: ['<rootDir>/src/test-setup.js']
};
```

## Unit Testing MCP Servers

### Testing Tool Handlers
```javascript
// src/tools/__tests__/database-tools.test.js
import { jest } from '@jest/globals';
import { DatabaseTools } from '../database-tools.js';

describe('DatabaseTools', () => {
  let dbTools;
  let mockDatabase;

  beforeEach(() => {
    mockDatabase = {
      query: jest.fn(),
      connect: jest.fn(),
      close: jest.fn()
    };
    dbTools = new DatabaseTools(mockDatabase);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('executeQuery', () => {
    it('should execute valid SELECT query', async () => {
      const mockResults = [
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' }
      ];
      
      mockDatabase.query.mockResolvedValue({ rows: mockResults });

      const result = await dbTools.executeQuery({
        query: 'SELECT * FROM users',
        limit: 10
      });

      expect(mockDatabase.query).toHaveBeenCalledWith(
        'SELECT * FROM users LIMIT $1',
        [10]
      );
      
      expect(result).toEqual({
        content: [{
          type: 'text',
          text: JSON.stringify(mockResults, null, 2)
        }]
      });
    });

    it('should reject dangerous queries', async () => {
      await expect(dbTools.executeQuery({
        query: 'DROP TABLE users'
      })).rejects.toThrow('Dangerous query detected');

      expect(mockDatabase.query).not.toHaveBeenCalled();
    });

    it('should handle database connection errors', async () => {
      mockDatabase.query.mockRejectedValue(new Error('Connection failed'));

      await expect(dbTools.executeQuery({
        query: 'SELECT 1'
      })).rejects.toThrow('Database error: Connection failed');
    });

    it('should validate input parameters', async () => {
      await expect(dbTools.executeQuery({})).rejects.toThrow('Query is required');
      
      await expect(dbTools.executeQuery({
        query: ''
      })).rejects.toThrow('Query cannot be empty');
      
      await expect(dbTools.executeQuery({
        query: 'SELECT * FROM users',
        limit: -1
      })).rejects.toThrow('Limit must be positive');
    });
  });

  describe('listTables', () => {
    it('should return available tables', async () => {
      const mockTables = ['users', 'products', 'orders'];
      mockDatabase.query.mockResolvedValue({
        rows: mockTables.map(name => ({ table_name: name }))
      });

      const result = await dbTools.listTables();

      expect(result.content[0].text).toContain('users');
      expect(result.content[0].text).toContain('products');
      expect(result.content[0].text).toContain('orders');
    });
  });
});
```

### Testing HTTP Tools
```javascript
// src/tools/__tests__/http-tools.test.js
import nock from 'nock';
import { HTTPTools } from '../http-tools.js';

describe('HTTPTools', () => {
  let httpTools;

  beforeEach(() => {
    httpTools = new HTTPTools();
    nock.cleanAll();
  });

  afterEach(() => {
    nock.cleanAll();
  });

  describe('fetchUrl', () => {
    it('should fetch URL successfully', async () => {
      const mockData = { message: 'Hello, World!' };
      
      nock('https://api.example.com')
        .get('/test')
        .reply(200, mockData);

      const result = await httpTools.fetchUrl({
        url: 'https://api.example.com/test'
      });

      expect(result.content[0].text).toContain(JSON.stringify(mockData));
    });

    it('should handle HTTP errors', async () => {
      nock('https://api.example.com')
        .get('/error')
        .reply(404, { error: 'Not found' });

      await expect(httpTools.fetchUrl({
        url: 'https://api.example.com/error'
      })).rejects.toThrow('HTTP 404');
    });

    it('should handle network timeouts', async () => {
      nock('https://api.example.com')
        .get('/slow')
        .delay(6000)
        .reply(200, 'OK');

      await expect(httpTools.fetchUrl({
        url: 'https://api.example.com/slow',
        timeout: 1000
      })).rejects.toThrow('timeout');
    });

    it('should validate URLs', async () => {
      await expect(httpTools.fetchUrl({
        url: 'not-a-url'
      })).rejects.toThrow('Invalid URL');

      await expect(httpTools.fetchUrl({
        url: 'javascript:alert("xss")'
      })).rejects.toThrow('Invalid protocol');
    });
  });

  describe('postData', () => {
    it('should send POST requests with JSON data', async () => {
      const requestData = { name: 'John', email: 'john@example.com' };
      const responseData = { id: 123, status: 'created' };

      nock('https://api.example.com')
        .post('/users', requestData)
        .reply(201, responseData);

      const result = await httpTools.postData({
        url: 'https://api.example.com/users',
        data: requestData
      });

      expect(result.content[0].text).toContain(JSON.stringify(responseData));
    });
  });
});
```

## Integration Testing

### Testing Complete MCP Server
```javascript
// src/__tests__/server.integration.test.js
import { jest } from '@jest/globals';
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { MCPServer } from '../server.js';

describe('MCP Server Integration', () => {
  let server;
  let mockTransport;

  beforeEach(async () => {
    mockTransport = {
      start: jest.fn(),
      close: jest.fn(),
      send: jest.fn(),
      onmessage: null,
      onerror: null,
      onclose: null
    };

    server = new MCPServer('test-server', '1.0.0');
  });

  afterEach(async () => {
    if (server) {
      await server.close();
    }
  });

  describe('Tool Discovery', () => {
    it('should list all registered tools', async () => {
      const response = await simulateRequest({
        jsonrpc: '2.0',
        method: 'tools/list',
        id: 1
      });

      expect(response.result.tools).toBeInstanceOf(Array);
      expect(response.result.tools.length).toBeGreaterThan(0);
      
      const toolNames = response.result.tools.map(t => t.name);
      expect(toolNames).toContain('database_query');
      expect(toolNames).toContain('http_fetch');
    });

    it('should provide complete tool schemas', async () => {
      const response = await simulateRequest({
        jsonrpc: '2.0',
        method: 'tools/list', 
        id: 1
      });

      const dbTool = response.result.tools.find(t => t.name === 'database_query');
      expect(dbTool).toBeDefined();
      expect(dbTool.description).toBeTruthy();
      expect(dbTool.inputSchema).toBeDefined();
      expect(dbTool.inputSchema.properties).toBeDefined();
    });
  });

  describe('Tool Execution', () => {
    it('should execute tools with valid parameters', async () => {
      const response = await simulateRequest({
        jsonrpc: '2.0',
        method: 'tools/call',
        params: {
          name: 'echo',
          arguments: { text: 'Hello, World!' }
        },
        id: 1
      });

      expect(response.result.content).toBeInstanceOf(Array);
      expect(response.result.content[0].text).toContain('Hello, World!');
    });

    it('should handle tool errors gracefully', async () => {
      const response = await simulateRequest({
        jsonrpc: '2.0',
        method: 'tools/call',
        params: {
          name: 'database_query',
          arguments: { query: 'INVALID SQL' }
        },
        id: 1
      });

      expect(response.error).toBeDefined();
      expect(response.error.message).toContain('syntax error');
    });

    it('should validate tool arguments', async () => {
      const response = await simulateRequest({
        jsonrpc: '2.0',
        method: 'tools/call',
        params: {
          name: 'echo',
          arguments: {} // Missing required 'text' parameter
        },
        id: 1
      });

      expect(response.error).toBeDefined();
      expect(response.error.message).toContain('required');
    });
  });

  // Helper function to simulate JSON-RPC requests
  async function simulateRequest(request) {
    return new Promise((resolve) => {
      const handler = server.server.getRequestHandler(request.method);
      handler(request).then(result => {
        resolve({
          jsonrpc: '2.0',
          id: request.id,
          result
        });
      }).catch(error => {
        resolve({
          jsonrpc: '2.0',
          id: request.id,
          error: {
            code: -32603,
            message: error.message
          }
        });
      });
    });
  }
});
```

## Testing Browser Clients

### Mock Service Worker Setup
```javascript
// src/test-helpers/msw-setup.js
import { setupServer } from 'msw/node';
import { rest } from 'msw';

// Mock MCP server responses
export const server = setupServer(
  rest.post('http://localhost:8000/mcp', (req, res, ctx) => {
    const { method, params } = req.body;
    
    if (method === 'tools/list') {
      return res(
        ctx.json({
          jsonrpc: '2.0',
          id: req.body.id,
          result: {
            tools: [
              {
                name: 'echo',
                description: 'Echo back text',
                inputSchema: {
                  type: 'object',
                  properties: {
                    text: { type: 'string' }
                  },
                  required: ['text']
                }
              }
            ]
          }
        })
      );
    }
    
    if (method === 'tools/call' && params.name === 'echo') {
      return res(
        ctx.json({
          jsonrpc: '2.0',
          id: req.body.id,
          result: {
            content: [{
              type: 'text',
              text: `Echo: ${params.arguments.text}`
            }]
          }
        })
      );
    }
    
    return res(
      ctx.status(404),
      ctx.json({
        jsonrpc: '2.0',
        id: req.body.id,
        error: {
          code: -32601,
          message: 'Method not found'
        }
      })
    );
  })
);

// Start server before all tests
beforeAll(() => server.listen());

// Reset handlers after each test
afterEach(() => server.resetHandlers());

// Clean up after all tests
afterAll(() => server.close());
```

### Browser Client Testing
```javascript
// src/__tests__/browser-client.test.js
import { screen, fireEvent, waitFor } from '@testing-library/dom';
import '@testing-library/jest-dom';
import { MCPWebClient } from '../browser-client.js';

// Mock fetch globally
global.fetch = jest.fn();

describe('Browser MCP Client', () => {
  let client;
  let mockContainer;

  beforeEach(() => {
    // Create DOM container for testing
    mockContainer = document.createElement('div');
    mockContainer.innerHTML = `
      <input id="serverUrl" value="http://localhost:8000/mcp">
      <button id="connectBtn">Connect</button>
      <div id="connectionStatus"></div>
      <div id="toolsList"></div>
      <div id="results"></div>
    `;
    document.body.appendChild(mockContainer);

    client = new MCPWebClient();
    
    // Reset fetch mock
    fetch.mockClear();
  });

  afterEach(() => {
    document.body.removeChild(mockContainer);
  });

  describe('Connection', () => {
    it('should connect to MCP server successfully', async () => {
      fetch.mockResolvedValueOnce({
        ok: true,
        json: () => Promise.resolve({
          jsonrpc: '2.0',
          id: 1,
          result: {}
        })
      });

      const connectBtn = screen.getByText('Connect');
      fireEvent.click(connectBtn);

      await waitFor(() => {
        expect(fetch).toHaveBeenCalledWith(
          'http://localhost:8000/mcp',
          expect.objectContaining({
            method: 'POST',
            headers: { 'Content-Type': 'application/json' }
          })
        );
      });
    });

    it('should handle connection errors', async () => {
      fetch.mockRejectedValueOnce(new Error('Network error'));

      const connectBtn = screen.getByText('Connect');
      fireEvent.click(connectBtn);

      await waitFor(() => {
        const status = screen.getByText(/Connection failed/);
        expect(status).toBeInTheDocument();
      });
    });
  });

  describe('Tool Management', () => {
    beforeEach(async () => {
      // Mock successful connection
      fetch.mockResolvedValue({
        ok: true,
        json: () => Promise.resolve({
          jsonrpc: '2.0',
          id: 1,
          result: {
            tools: [{
              name: 'echo',
              description: 'Echo text',
              inputSchema: {
                type: 'object',
                properties: {
                  text: { type: 'string' }
                },
                required: ['text']
              }
            }]
          }
        })
      });

      await client.connect();
    });

    it('should load and display tools', async () => {
      await waitFor(() => {
        expect(screen.getByText('echo')).toBeInTheDocument();
        expect(screen.getByText('Echo text')).toBeInTheDocument();
      });
    });

    it('should generate input fields from schema', async () => {
      await waitFor(() => {
        const textInput = screen.getByPlaceholderText(/text/i);
        expect(textInput).toBeInTheDocument();
        expect(textInput.type).toBe('text');
      });
    });

    it('should call tools with user input', async () => {
      fetch.mockResolvedValueOnce({
        ok: true,
        json: () => Promise.resolve({
          jsonrpc: '2.0',
          id: 2,
          result: {
            content: [{ type: 'text', text: 'Echo: Hello' }]
          }
        })
      });

      await waitFor(() => {
        const textInput = screen.getByPlaceholderText(/text/i);
        fireEvent.change(textInput, { target: { value: 'Hello' } });
        
        const callBtn = screen.getByText('Call Tool');
        fireEvent.click(callBtn);
      });

      await waitFor(() => {
        expect(fetch).toHaveBeenCalledWith(
          'http://localhost:8000/mcp',
          expect.objectContaining({
            body: expect.stringContaining('"arguments":{"text":"Hello"}')
          })
        );
      });
    });
  });
});
```

## Performance Testing

### Load Testing Tools
```javascript
// src/__tests__/performance.test.js
import { performance } from 'perf_hooks';
import { MCPServer } from '../server.js';

describe('Performance Tests', () => {
  let server;

  beforeEach(() => {
    server = new MCPServer('perf-test-server', '1.0.0');
  });

  describe('Response Times', () => {
    it('should respond to tool calls within acceptable time', async () => {
      const startTime = performance.now();
      
      await server.handleToolCall('echo', { text: 'performance test' });
      
      const endTime = performance.now();
      const responseTime = endTime - startTime;
      
      expect(responseTime).toBeLessThan(100); // 100ms threshold
    });

    it('should handle concurrent requests efficiently', async () => {
      const numberOfRequests = 100;
      const requests = Array.from({ length: numberOfRequests }, (_, i) => 
        server.handleToolCall('echo', { text: `request ${i}` })
      );

      const startTime = performance.now();
      await Promise.all(requests);
      const endTime = performance.now();

      const totalTime = endTime - startTime;
      const avgResponseTime = totalTime / numberOfRequests;

      expect(avgResponseTime).toBeLessThan(50); // Average under 50ms
    });
  });

  describe('Memory Usage', () => {
    it('should not leak memory during repeated operations', async () => {
      const initialMemory = process.memoryUsage().heapUsed;
      
      // Perform many operations
      for (let i = 0; i < 1000; i++) {
        await server.handleToolCall('echo', { text: `iteration ${i}` });
      }
      
      // Force garbage collection if available
      if (global.gc) {
        global.gc();
      }
      
      const finalMemory = process.memoryUsage().heapUsed;
      const memoryIncrease = finalMemory - initialMemory;
      
      // Memory increase should be reasonable (less than 10MB)
      expect(memoryIncrease).toBeLessThan(10 * 1024 * 1024);
    });
  });
});
```

## Test Coverage and Reporting

### Coverage Configuration
```javascript
// jest.config.js (coverage section)
export default {
  // ... other config
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/**/*.test.js',
    '!src/**/index.js',
    '!src/test-helpers/**'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    },
    './src/tools/': {
      branches: 90,
      functions: 90,
      lines: 90,
      statements: 90
    }
  }
};
```

### Custom Test Reporters
```javascript
// src/test-helpers/custom-reporter.js
export default class CustomReporter {
  constructor(globalConfig, options) {
    this._globalConfig = globalConfig;
    this._options = options;
  }

  onRunComplete(contexts, results) {
    const { numTotalTests, numPassedTests, numFailedTests, testResults } = results;
    
    console.log('\n=== Test Summary ===');
    console.log(`Total Tests: ${numTotalTests}`);
    console.log(`Passed: ${numPassedTests}`);
    console.log(`Failed: ${numFailedTests}`);
    
    if (numFailedTests > 0) {
      console.log('\n=== Failed Tests ===');
      testResults.forEach(suite => {
        suite.testResults.forEach(test => {
          if (test.status === 'failed') {
            console.log(`❌ ${suite.testFilePath}: ${test.fullName}`);
            test.failureMessages.forEach(msg => {
              console.log(`   ${msg}`);
            });
          }
        });
      });
    }
    
    console.log(`\nTest run ${numFailedTests === 0 ? '✅ PASSED' : '❌ FAILED'}`);
  }
}
```

## CI/CD Integration

### GitHub Actions Testing
```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node-version: [18, 20, 21]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node-version }}
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run type check
      run: npm run type-check
    
    - name: Run tests
      run: npm test -- --coverage
    
    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage/lcov.info
```

## Best Practices

### Test Organization
1. **Describe Blocks**: Group related tests logically
2. **Clear Names**: Use descriptive test names that explain the behavior
3. **Arrange-Act-Assert**: Structure tests with clear setup, execution, and verification
4. **One Assertion**: Focus each test on a single behavior

### Mocking Strategy
1. **External Dependencies**: Mock all external services and APIs
2. **Database**: Use in-memory databases or mocks for testing
3. **File System**: Mock file operations to avoid test pollution
4. **Time**: Mock time-dependent functions for predictable tests

### Performance
1. **Parallel Execution**: Run tests in parallel when possible
2. **Test Isolation**: Ensure tests don't depend on each other
3. **Fast Feedback**: Prioritize unit tests for quick feedback
4. **Resource Cleanup**: Clean up resources after tests

Testing is essential for building reliable MCP applications. Focus on comprehensive coverage, realistic scenarios, and maintainable test code.