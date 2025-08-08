# JavaScript Packaging

## Packaging MCP Applications for Distribution

Proper packaging makes your MCP servers and clients easy to install, distribute, and maintain across different environments.

## Package Structure

### Node.js Package Setup
```json
{
  "name": "@your-org/mcp-server",
  "version": "1.0.0",
  "description": "MCP server for [specific purpose]",
  "type": "module",
  "main": "dist/index.js",
  "module": "src/index.js",
  "types": "dist/index.d.ts",
  "exports": {
    ".": {
      "import": "./dist/index.js",
      "require": "./dist/index.cjs",
      "types": "./dist/index.d.ts"
    },
    "./client": {
      "import": "./dist/client.js",
      "types": "./dist/client.d.ts"
    }
  },
  "bin": {
    "mcp-server": "./bin/server.js"
  },
  "files": [
    "dist/",
    "bin/",
    "README.md",
    "LICENSE"
  ],
  "scripts": {
    "build": "npm run clean && npm run compile",
    "clean": "rm -rf dist/",
    "compile": "tsc && npm run bundle",
    "bundle": "rollup -c rollup.config.js",
    "dev": "nodemon --exec node --loader=tsx src/index.ts",
    "start": "node dist/index.js",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "lint": "eslint src/**/*.{js,ts}",
    "lint:fix": "eslint src/**/*.{js,ts} --fix",
    "prepublishOnly": "npm run build && npm test",
    "postversion": "git push && git push --tags"
  },
  "keywords": [
    "mcp",
    "model-context-protocol",
    "ai",
    "assistant",
    "server",
    "tools"
  ],
  "author": "Your Name <you@example.com>",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/your-org/mcp-server.git"
  },
  "bugs": {
    "url": "https://github.com/your-org/mcp-server/issues"
  },
  "homepage": "https://github.com/your-org/mcp-server#readme",
  "engines": {
    "node": ">=18.0.0",
    "npm": ">=9.0.0"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^0.1.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.56.0",
    "jest": "^29.7.0",
    "nodemon": "^3.0.0",
    "rollup": "^4.0.0",
    "typescript": "^5.3.0",
    "tsx": "^4.0.0"
  }
}
```

## Build Configuration

### TypeScript Compilation
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
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "removeComments": false,
    "noEmitOnError": true,
    "resolveJsonModule": true,
    "allowImportingTsExtensions": false,
    "noEmit": false
  },
  "include": ["src/**/*"],
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts",
    "**/*.test.js"
  ]
}
```

### Rollup Bundling
```javascript
// rollup.config.js
import { nodeResolve } from '@rollup/plugin-node-resolve';
import { terser } from 'rollup-plugin-terser';
import commonjs from '@rollup/plugin-commonjs';
import json from '@rollup/plugin-json';

const isProduction = process.env.NODE_ENV === 'production';

export default [
  // ESM build
  {
    input: 'dist/index.js',
    output: {
      file: 'dist/index.esm.js',
      format: 'esm',
      sourcemap: true
    },
    plugins: [
      nodeResolve({ preferBuiltins: true }),
      json(),
      isProduction && terser()
    ].filter(Boolean),
    external: [
      '@modelcontextprotocol/sdk',
      'node:fs',
      'node:path',
      'node:process'
    ]
  },
  
  // CommonJS build
  {
    input: 'dist/index.js', 
    output: {
      file: 'dist/index.cjs',
      format: 'cjs',
      sourcemap: true
    },
    plugins: [
      nodeResolve({ preferBuiltins: true }),
      commonjs(),
      json(),
      isProduction && terser()
    ].filter(Boolean),
    external: [
      '@modelcontextprotocol/sdk'
    ]
  },

  // Binary executable
  {
    input: 'dist/server.js',
    output: {
      file: 'bin/server.js',
      format: 'cjs',
      banner: '#!/usr/bin/env node',
      sourcemap: false
    },
    plugins: [
      nodeResolve({ preferBuiltins: true }),
      commonjs(),
      json(),
      terser()
    ],
    external: []
  }
];
```

## CLI Executable

### Binary Script
```javascript
#!/usr/bin/env node
// bin/server.js

import { fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';
import { readFileSync } from 'node:fs';

const __filename = fileURLToPath(import.meta.url);
const __dirname = dirname(__filename);

// Load package.json for version info
const packageJson = JSON.parse(
  readFileSync(join(__dirname, '../package.json'), 'utf8')
);

// CLI argument parsing
function parseArgs() {
  const args = process.argv.slice(2);
  const options = {
    version: false,
    help: false,
    config: null,
    port: process.env.PORT || 8000,
    transport: 'stdio'
  };

  for (let i = 0; i < args.length; i++) {
    switch (args[i]) {
      case '--version':
      case '-v':
        options.version = true;
        break;
      case '--help':
      case '-h':
        options.help = true;
        break;
      case '--config':
      case '-c':
        options.config = args[++i];
        break;
      case '--port':
      case '-p':
        options.port = parseInt(args[++i]);
        break;
      case '--transport':
      case '-t':
        options.transport = args[++i];
        break;
      default:
        console.error(`Unknown option: ${args[i]}`);
        process.exit(1);
    }
  }

  return options;
}

// Display help text
function showHelp() {
  console.log(`
${packageJson.name} v${packageJson.version}

Usage: mcp-server [options]

Options:
  -v, --version              Show version number
  -h, --help                 Show help
  -c, --config <file>        Configuration file path
  -p, --port <number>        Port number (default: 8000)
  -t, --transport <type>     Transport type: stdio|http (default: stdio)

Examples:
  mcp-server                 Start with stdio transport
  mcp-server -t http -p 8000 Start HTTP server on port 8000
  mcp-server -c config.json  Start with custom configuration
`);
}

// Main execution
async function main() {
  const options = parseArgs();

  if (options.version) {
    console.log(packageJson.version);
    process.exit(0);
  }

  if (options.help) {
    showHelp();
    process.exit(0);
  }

  try {
    const { MCPServer } = await import('../dist/index.js');
    
    const server = new MCPServer({
      name: packageJson.name,
      version: packageJson.version,
      configFile: options.config
    });

    console.log(`Starting ${packageJson.name} v${packageJson.version}`);
    console.log(`Transport: ${options.transport}`);
    
    if (options.transport === 'http') {
      console.log(`Port: ${options.port}`);
      await server.listen(options.port);
    } else {
      console.log('Using stdio transport');
      await server.run();
    }
    
  } catch (error) {
    console.error('Failed to start server:', error.message);
    process.exit(1);
  }
}

// Handle process signals
process.on('SIGINT', () => {
  console.log('\nReceived SIGINT, shutting down gracefully...');
  process.exit(0);
});

process.on('SIGTERM', () => {
  console.log('\nReceived SIGTERM, shutting down gracefully...');
  process.exit(0);
});

main().catch((error) => {
  console.error('Unhandled error:', error);
  process.exit(1);
});
```

## Browser Distribution

### Webpack Configuration
```javascript
// webpack.config.js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const { CleanWebpackPlugin } = require('clean-webpack-plugin');

module.exports = (env, argv) => {
  const isProduction = argv.mode === 'production';
  
  return {
    entry: {
      main: './src/browser/index.js',
      worker: './src/browser/worker.js'
    },
    
    output: {
      filename: isProduction ? '[name].[contenthash].js' : '[name].js',
      path: path.resolve(__dirname, 'dist/browser'),
      clean: true,
      library: {
        name: 'MCPClient',
        type: 'umd'
      }
    },
    
    module: {
      rules: [
        {
          test: /\.js$/,
          exclude: /node_modules/,
          use: {
            loader: 'babel-loader',
            options: {
              presets: [
                ['@babel/preset-env', {
                  targets: {
                    browsers: ['> 1%', 'last 2 versions']
                  }
                }]
              ]
            }
          }
        },
        {
          test: /\.css$/i,
          use: ['style-loader', 'css-loader']
        },
        {
          test: /\.(png|svg|jpg|jpeg|gif)$/i,
          type: 'asset/resource'
        }
      ]
    },
    
    plugins: [
      new CleanWebpackPlugin(),
      new HtmlWebpackPlugin({
        title: 'MCP Client',
        template: './src/browser/index.html',
        chunks: ['main']
      })
    ],
    
    optimization: {
      splitChunks: {
        chunks: 'all',
        cacheGroups: {
          vendor: {
            test: /[\\/]node_modules[\\/]/,
            name: 'vendors',
            chunks: 'all',
          }
        }
      }
    },
    
    devServer: {
      static: './dist/browser',
      port: 3000,
      hot: true,
      open: true
    },
    
    resolve: {
      fallback: {
        "buffer": require.resolve("buffer/"),
        "process": require.resolve("process/browser"),
        "stream": require.resolve("stream-browserify"),
        "util": require.resolve("util/")
      }
    }
  };
};
```

## Docker Packaging

### Multi-stage Dockerfile
```dockerfile
# Build stage
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
COPY tsconfig.json ./
COPY rollup.config.js ./

# Install dependencies
RUN npm ci --only=production

# Copy source code
COPY src/ ./src/

# Build application
RUN npm run build

# Production stage
FROM node:20-alpine AS production

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S mcp -u 1001

WORKDIR /app

# Copy built application
COPY --from=builder --chown=mcp:nodejs /app/dist ./dist
COPY --from=builder --chown=mcp:nodejs /app/package*.json ./
COPY --from=builder --chown=mcp:nodejs /app/bin ./bin

# Install production dependencies only
RUN npm ci --omit=dev --cache /tmp/empty-cache && \
    rm -rf /tmp/empty-cache

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "console.log('OK')" || exit 1

# Switch to non-root user
USER mcp

# Expose port
EXPOSE 8000

# Default command
CMD ["node", "bin/server.js", "--transport", "http", "--port", "8000"]
```

### Docker Compose for Development
```yaml
# docker-compose.yml
version: '3.8'

services:
  mcp-server:
    build: .
    ports:
      - "8000:8000"
    environment:
      - NODE_ENV=production
      - PORT=8000
      - LOG_LEVEL=info
    volumes:
      - ./config:/app/config:ro
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  mcp-client:
    build:
      context: .
      dockerfile: Dockerfile.client
    ports:
      - "3000:3000"
    environment:
      - REACT_APP_MCP_SERVER_URL=http://mcp-server:8000
    depends_on:
      - mcp-server
```

## Publishing

### NPM Publishing Workflow
```javascript
// scripts/publish.js
#!/usr/bin/env node

import { execSync } from 'child_process';
import { readFileSync, writeFileSync } from 'fs';

// Validate environment
if (!process.env.NPM_TOKEN) {
  console.error('NPM_TOKEN environment variable is required');
  process.exit(1);
}

// Read package.json
const packageJson = JSON.parse(readFileSync('package.json', 'utf8'));
const currentVersion = packageJson.version;

console.log(`Publishing ${packageJson.name} v${currentVersion}`);

try {
  // Run pre-publish checks
  console.log('Running pre-publish checks...');
  execSync('npm run lint', { stdio: 'inherit' });
  execSync('npm test', { stdio: 'inherit' });
  execSync('npm run build', { stdio: 'inherit' });
  
  // Publish to npm
  console.log('Publishing to npm...');
  execSync('npm publish --access public', { stdio: 'inherit' });
  
  // Create git tag
  console.log('Creating git tag...');
  execSync(`git tag v${currentVersion}`, { stdio: 'inherit' });
  execSync('git push origin --tags', { stdio: 'inherit' });
  
  console.log(`✅ Successfully published ${packageJson.name} v${currentVersion}`);
  
} catch (error) {
  console.error('❌ Publish failed:', error.message);
  process.exit(1);
}
```

### GitHub Actions Publishing
```yaml
# .github/workflows/publish.yml
name: Publish Package

on:
  release:
    types: [published]

jobs:
  publish-npm:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          registry-url: 'https://registry.npmjs.org'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run tests
        run: npm test
      
      - name: Build package
        run: npm run build
      
      - name: Publish to NPM
        run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}

  publish-docker:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: your-org/mcp-server
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

## Distribution Formats

### Multiple Export Formats
```javascript
// scripts/build-all.js
import { execSync } from 'child_process';
import { mkdirSync, existsSync } from 'fs';

// Ensure dist directory exists
if (!existsSync('dist')) {
  mkdirSync('dist', { recursive: true });
}

console.log('Building all distribution formats...');

// Build TypeScript
console.log('📦 Building TypeScript...');
execSync('tsc', { stdio: 'inherit' });

// Build ES modules
console.log('📦 Building ES modules...');
execSync('rollup -c --format es --file dist/index.esm.js', { stdio: 'inherit' });

// Build CommonJS
console.log('📦 Building CommonJS...');
execSync('rollup -c --format cjs --file dist/index.cjs', { stdio: 'inherit' });

// Build UMD for browsers
console.log('📦 Building UMD...');
execSync('rollup -c --format umd --name MCPServer --file dist/index.umd.js', { stdio: 'inherit' });

// Build minified version
console.log('📦 Building minified version...');
execSync('rollup -c --format umd --name MCPServer --file dist/index.umd.min.js --environment NODE_ENV:production', { stdio: 'inherit' });

// Build CLI executable
console.log('📦 Building CLI executable...');
execSync('rollup -c rollup.cli.config.js', { stdio: 'inherit' });

console.log('✅ All builds completed successfully!');
```

## Version Management

### Semantic Release Configuration
```json
// .releaserc.json
{
  "branches": [
    "main",
    {
      "name": "beta",
      "prerelease": true
    }
  ],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    [
      "@semantic-release/changelog",
      {
        "changelogFile": "CHANGELOG.md"
      }
    ],
    [
      "@semantic-release/npm",
      {
        "npmPublish": true
      }
    ],
    [
      "@semantic-release/github",
      {
        "assets": [
          {
            "path": "dist/*.js",
            "label": "Distribution files"
          }
        ]
      }
    ],
    [
      "@semantic-release/git",
      {
        "assets": ["package.json", "CHANGELOG.md"],
        "message": "chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}"
      }
    ]
  ]
}
```

## Best Practices

### Package Optimization
1. **Tree Shaking**: Structure code to support tree shaking
2. **Bundle Size**: Monitor and optimize bundle size
3. **Dependencies**: Minimize runtime dependencies
4. **Polyfills**: Include necessary polyfills for browser compatibility

### Security
1. **Dependency Scanning**: Regularly scan for vulnerable dependencies
2. **Package Signing**: Sign packages when publishing
3. **Access Control**: Use proper NPM access controls
4. **Secrets Management**: Never commit API keys or secrets

### Distribution
1. **Multiple Formats**: Support ESM, CommonJS, and UMD formats
2. **Browser Compatibility**: Test across different browsers
3. **Node.js Versions**: Support multiple Node.js versions
4. **Documentation**: Include comprehensive installation and usage docs

Proper packaging ensures your MCP applications are accessible, maintainable, and easy to deploy across different environments and platforms.