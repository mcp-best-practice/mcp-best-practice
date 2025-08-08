# Go Project Structure

## Recommended Project Layout for Go MCP Servers

Following Go community conventions and best practices for maintainable MCP server projects.

## Standard Go Project Layout

```
mcp-server-go/
├── cmd/
│   └── server/
│       └── main.go              # Application entrypoint
├── internal/
│   ├── server/
│   │   ├── server.go           # MCP server implementation
│   │   └── handlers.go         # Request handlers
│   ├── tools/
│   │   ├── tools.go           # Tool implementations
│   │   └── registry.go        # Tool registration
│   ├── resources/
│   │   ├── resources.go       # Resource providers
│   │   └── handlers.go        # Resource handlers
│   └── config/
│       └── config.go          # Configuration management
├── pkg/
│   ├── client/
│   │   └── client.go          # MCP client implementation (if needed)
│   └── types/
│       └── types.go           # Shared types
├── api/
│   └── mcp/
│       └── v1/
│           └── types.go       # API type definitions
├── build/
│   ├── Dockerfile
│   └── package/
├── scripts/
│   ├── build.sh
│   └── test.sh
├── test/
│   ├── integration/
│   └── testdata/
├── docs/
│   ├── api.md
│   └── deployment.md
├── go.mod                     # Go module definition
├── go.sum                     # Dependency checksums
├── Makefile                   # Build automation
└── README.md
```

## Core Components

### Main Entry Point
```go
// cmd/server/main.go
package main

import (
    "context"
    "flag"
    "log"
    "os"
    "os/signal"
    "syscall"

    "github.com/your-org/mcp-server-go/internal/config"
    "github.com/your-org/mcp-server-go/internal/server"
)

func main() {
    var configPath = flag.String("config", "", "Path to configuration file")
    flag.Parse()

    cfg, err := config.Load(*configPath)
    if err != nil {
        log.Fatalf("Failed to load configuration: %v", err)
    }

    srv, err := server.New(cfg)
    if err != nil {
        log.Fatalf("Failed to create server: %v", err)
    }

    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    // Handle graceful shutdown
    sigCh := make(chan os.Signal, 1)
    signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
    
    go func() {
        <-sigCh
        log.Println("Shutting down server...")
        cancel()
    }()

    if err := srv.Run(ctx); err != nil {
        log.Fatalf("Server error: %v", err)
    }
}
```

### Server Implementation
```go
// internal/server/server.go
package server

import (
    "context"
    "encoding/json"
    "fmt"
    "io"
    "os"

    "github.com/modelcontextprotocol/go-sdk/server"
    "github.com/your-org/mcp-server-go/internal/config"
    "github.com/your-org/mcp-server-go/internal/tools"
)

type Server struct {
    config *config.Config
    server *server.Server
}

func New(cfg *config.Config) (*Server, error) {
    mcpServer := server.NewServer(
        server.ServerInfo{
            Name:    cfg.ServerName,
            Version: cfg.Version,
        },
        server.ServerCapabilities{
            Tools: &server.ToolsCapability{},
        },
    )

    s := &Server{
        config: cfg,
        server: mcpServer,
    }

    if err := s.registerHandlers(); err != nil {
        return nil, fmt.Errorf("failed to register handlers: %w", err)
    }

    return s, nil
}

func (s *Server) registerHandlers() error {
    // Register tool handlers
    s.server.SetListToolsHandler(s.handleListTools)
    s.server.SetCallToolHandler(s.handleCallTool)

    return nil
}

func (s *Server) Run(ctx context.Context) error {
    switch s.config.Transport {
    case "stdio":
        return s.server.RunStdio(ctx, os.Stdin, os.Stdout)
    case "http":
        return s.server.RunHTTP(ctx, s.config.HTTPAddr)
    default:
        return fmt.Errorf("unsupported transport: %s", s.config.Transport)
    }
}
```

### Configuration Management
```go
// internal/config/config.go
package config

import (
    "encoding/json"
    "fmt"
    "os"
)

type Config struct {
    ServerName string `json:"server_name" env:"SERVER_NAME"`
    Version    string `json:"version" env:"VERSION"`
    Transport  string `json:"transport" env:"TRANSPORT"`
    HTTPAddr   string `json:"http_addr" env:"HTTP_ADDR"`
    
    // Database configuration
    Database DatabaseConfig `json:"database"`
    
    // API configuration
    APIs map[string]APIConfig `json:"apis"`
    
    // Logging
    LogLevel  string `json:"log_level" env:"LOG_LEVEL"`
    LogFormat string `json:"log_format" env:"LOG_FORMAT"`
}

type DatabaseConfig struct {
    URL         string `json:"url" env:"DATABASE_URL"`
    MaxConns    int    `json:"max_conns" env:"DATABASE_MAX_CONNS"`
    MaxIdle     int    `json:"max_idle" env:"DATABASE_MAX_IDLE"`
    ConnTimeout int    `json:"conn_timeout" env:"DATABASE_CONN_TIMEOUT"`
}

type APIConfig struct {
    BaseURL string `json:"base_url"`
    APIKey  string `json:"api_key"`
    Timeout int    `json:"timeout"`
}

func Load(configPath string) (*Config, error) {
    cfg := &Config{
        ServerName: "mcp-server-go",
        Version:    "1.0.0",
        Transport:  "stdio",
        HTTPAddr:   ":8000",
        LogLevel:   "info",
        LogFormat:  "json",
        Database: DatabaseConfig{
            MaxConns:    10,
            MaxIdle:     5,
            ConnTimeout: 30,
        },
    }

    // Load from file if provided
    if configPath != "" {
        if err := loadFromFile(cfg, configPath); err != nil {
            return nil, fmt.Errorf("failed to load config from file: %w", err)
        }
    }

    // Override with environment variables
    if err := loadFromEnv(cfg); err != nil {
        return nil, fmt.Errorf("failed to load config from environment: %w", err)
    }

    return cfg, nil
}

func loadFromFile(cfg *Config, path string) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return err
    }

    return json.Unmarshal(data, cfg)
}

func loadFromEnv(cfg *Config) error {
    if val := os.Getenv("SERVER_NAME"); val != "" {
        cfg.ServerName = val
    }
    if val := os.Getenv("VERSION"); val != "" {
        cfg.Version = val
    }
    if val := os.Getenv("TRANSPORT"); val != "" {
        cfg.Transport = val
    }
    if val := os.Getenv("HTTP_ADDR"); val != "" {
        cfg.HTTPAddr = val
    }
    if val := os.Getenv("DATABASE_URL"); val != "" {
        cfg.Database.URL = val
    }
    if val := os.Getenv("LOG_LEVEL"); val != "" {
        cfg.LogLevel = val
    }

    return nil
}
```

### Tool Implementation
```go
// internal/tools/tools.go
package tools

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/modelcontextprotocol/go-sdk/server"
)

type ToolRegistry struct {
    tools map[string]Tool
}

type Tool interface {
    Name() string
    Description() string
    InputSchema() map[string]interface{}
    Execute(ctx context.Context, arguments map[string]interface{}) ([]server.Content, error)
}

func NewRegistry() *ToolRegistry {
    return &ToolRegistry{
        tools: make(map[string]Tool),
    }
}

func (r *ToolRegistry) Register(tool Tool) {
    r.tools[tool.Name()] = tool
}

func (r *ToolRegistry) ListTools() []server.Tool {
    var tools []server.Tool
    
    for _, tool := range r.tools {
        tools = append(tools, server.Tool{
            Name:        tool.Name(),
            Description: tool.Description(),
            InputSchema: tool.InputSchema(),
        })
    }
    
    return tools
}

func (r *ToolRegistry) CallTool(ctx context.Context, name string, arguments map[string]interface{}) ([]server.Content, error) {
    tool, exists := r.tools[name]
    if !exists {
        return nil, fmt.Errorf("unknown tool: %s", name)
    }

    return tool.Execute(ctx, arguments)
}

// Example tool implementation
type EchoTool struct{}

func (t *EchoTool) Name() string {
    return "echo"
}

func (t *EchoTool) Description() string {
    return "Echo back the provided text"
}

func (t *EchoTool) InputSchema() map[string]interface{} {
    return map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "text": map[string]interface{}{
                "type":        "string",
                "description": "Text to echo back",
            },
        },
        "required": []string{"text"},
    }
}

func (t *EchoTool) Execute(ctx context.Context, arguments map[string]interface{}) ([]server.Content, error) {
    text, ok := arguments["text"].(string)
    if !ok {
        return nil, fmt.Errorf("text argument must be a string")
    }

    return []server.Content{
        {
            Type: "text",
            Text: fmt.Sprintf("Echo: %s", text),
        },
    }, nil
}
```

## Go Module Configuration

### go.mod
```go
module github.com/your-org/mcp-server-go

go 1.21

require (
    github.com/modelcontextprotocol/go-sdk v0.1.0
    github.com/stretchr/testify v1.8.4
)

require (
    github.com/davecgh/go-spew v1.1.1 // indirect
    github.com/pmezard/go-difflib v1.0.0 // indirect
    gopkg.in/yaml.v3 v3.0.1 // indirect
)
```

## Build Configuration

### Makefile
```makefile
.PHONY: build test clean lint fmt vet deps

# Go parameters
GOCMD=go
GOBUILD=$(GOCMD) build
GOCLEAN=$(GOCMD) clean
GOTEST=$(GOCMD) test
GOGET=$(GOCMD) get
GOMOD=$(GOCMD) mod
BINARY_NAME=mcp-server
BINARY_PATH=cmd/server

# Build flags
LDFLAGS=-ldflags "-w -s"
BUILD_FLAGS=-trimpath $(LDFLAGS)

# Default target
all: test build

# Build the binary
build:
	$(GOBUILD) $(BUILD_FLAGS) -o $(BINARY_NAME) ./$(BINARY_PATH)

# Build for different platforms
build-linux:
	GOOS=linux GOARCH=amd64 $(GOBUILD) $(BUILD_FLAGS) -o $(BINARY_NAME)-linux-amd64 ./$(BINARY_PATH)

build-windows:
	GOOS=windows GOARCH=amd64 $(GOBUILD) $(BUILD_FLAGS) -o $(BINARY_NAME)-windows-amd64.exe ./$(BINARY_PATH)

build-darwin:
	GOOS=darwin GOARCH=amd64 $(GOBUILD) $(BUILD_FLAGS) -o $(BINARY_NAME)-darwin-amd64 ./$(BINARY_PATH)

# Test
test:
	$(GOTEST) -v -race -coverprofile=coverage.out ./...

# Test with coverage
test-coverage:
	$(GOTEST) -race -coverprofile=coverage.out ./...
	$(GOCMD) tool cover -html=coverage.out -o coverage.html

# Clean
clean:
	$(GOCLEAN)
	rm -f $(BINARY_NAME)*
	rm -f coverage.out coverage.html

# Format code
fmt:
	$(GOCMD) fmt ./...

# Lint
lint:
	golangci-lint run

# Vet
vet:
	$(GOCMD) vet ./...

# Update dependencies
deps:
	$(GOMOD) download
	$(GOMOD) tidy

# Install development tools
tools:
	$(GOGET) -u github.com/golangci/golangci-lint/cmd/golangci-lint

# Run server
run:
	$(GOCMD) run ./$(BINARY_PATH)

# Run with config
run-config:
	$(GOCMD) run ./$(BINARY_PATH) -config=config.json

# Install binary
install:
	$(GOCMD) install $(BUILD_FLAGS) ./$(BINARY_PATH)
```

## Best Practices

### Project Organization
1. **cmd/**: Application entrypoints
2. **internal/**: Private application code
3. **pkg/**: Library code for external use
4. **api/**: API definitions and generated code
5. **build/**: Packaging and CI configuration

### Code Structure
1. **Interfaces**: Define clear interfaces for testability
2. **Dependency Injection**: Use constructor injection
3. **Error Handling**: Return errors, don't panic
4. **Context**: Pass context.Context for cancellation

### Naming Conventions
1. **Packages**: Short, lowercase, single word
2. **Files**: Snake_case for multi-word names
3. **Functions**: CamelCase, exported functions capitalized
4. **Constants**: CamelCase or ALL_CAPS for package-level

This structure provides a solid foundation for building scalable and maintainable MCP servers in Go.