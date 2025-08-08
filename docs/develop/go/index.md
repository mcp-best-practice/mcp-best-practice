# Go Development

## MCP Server Development with Go

Go provides excellent performance and concurrency for MCP servers, making it ideal for high-throughput applications.

## Quick Start

### Installation
```bash
go get github.com/modelcontextprotocol/go-mcp
```

### Basic Server
```go
package main

import (
    "context"
    "log"
    
    "github.com/modelcontextprotocol/go-mcp/server"
    "github.com/modelcontextprotocol/go-mcp/types"
)

func main() {
    // Create server
    srv := server.New("my-go-server", "1.0.0")
    
    // Register tool
    srv.RegisterTool("greet", types.Tool{
        Description: "Greet someone",
        Parameters: types.Parameters{
            Type: "object",
            Properties: map[string]types.Property{
                "name": {Type: "string", Description: "Name to greet"},
            },
            Required: []string{"name"},
        },
    }, greetHandler)
    
    // Start server
    if err := srv.Start(":8000"); err != nil {
        log.Fatal(err)
    }
}

func greetHandler(ctx context.Context, params map[string]interface{}) (interface{}, error) {
    name, ok := params["name"].(string)
    if !ok {
        return nil, fmt.Errorf("name must be a string")
    }
    
    return fmt.Sprintf("Hello, %s!", name), nil
}
```

## Project Structure

### Standard Layout
```
my-go-server/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handlers/
│   │   └── tools.go
│   ├── models/
│   │   └── types.go
│   └── service/
│       └── service.go
├── pkg/
│   └── client/
│       └── client.go
├── configs/
│   └── config.yaml
├── scripts/
│   └── build.sh
├── go.mod
├── go.sum
├── Makefile
├── Dockerfile
└── README.md
```

### Go Module
```go
// go.mod
module github.com/yourusername/my-mcp-server

go 1.21

require (
    github.com/modelcontextprotocol/go-mcp v1.0.0
    github.com/gorilla/mux v1.8.0
    github.com/spf13/viper v1.16.0
)
```

## Advanced Implementation

### Structured Tools
```go
type GreetRequest struct {
    Name     string `json:"name" validate:"required"`
    Greeting string `json:"greeting,omitempty"`
}

type GreetResponse struct {
    Message   string    `json:"message"`
    Timestamp time.Time `json:"timestamp"`
}

func (s *Server) RegisterTools() {
    s.mcp.RegisterTool("greet", types.Tool{
        Description: "Greet someone with timestamp",
        Parameters:  generateSchema(GreetRequest{}),
    }, s.handleGreet)
}

func (s *Server) handleGreet(ctx context.Context, params json.RawMessage) (interface{}, error) {
    var req GreetRequest
    if err := json.Unmarshal(params, &req); err != nil {
        return nil, fmt.Errorf("invalid parameters: %w", err)
    }
    
    if err := s.validator.Struct(req); err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }
    
    greeting := "Hello"
    if req.Greeting != "" {
        greeting = req.Greeting
    }
    
    return GreetResponse{
        Message:   fmt.Sprintf("%s, %s!", greeting, req.Name),
        Timestamp: time.Now(),
    }, nil
}
```

### Concurrent Operations
```go
func (s *Server) handleBatchProcess(ctx context.Context, items []string) ([]Result, error) {
    results := make([]Result, len(items))
    errChan := make(chan error, len(items))
    
    var wg sync.WaitGroup
    for i, item := range items {
        wg.Add(1)
        go func(index int, data string) {
            defer wg.Done()
            
            result, err := s.processItem(ctx, data)
            if err != nil {
                errChan <- err
                return
            }
            results[index] = result
        }(i, item)
    }
    
    wg.Wait()
    close(errChan)
    
    // Check for errors
    for err := range errChan {
        if err != nil {
            return nil, fmt.Errorf("batch processing failed: %w", err)
        }
    }
    
    return results, nil
}
```

### Resource Management
```go
type ResourceManager struct {
    pool *sql.DB
    cache *cache.Cache
    mu    sync.RWMutex
}

func NewResourceManager(config *Config) (*ResourceManager, error) {
    db, err := sql.Open("postgres", config.DatabaseURL)
    if err != nil {
        return nil, err
    }
    
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    
    return &ResourceManager{
        pool:  db,
        cache: cache.New(5*time.Minute, 10*time.Minute),
    }, nil
}

func (rm *ResourceManager) Get(ctx context.Context, key string) (interface{}, error) {
    // Check cache first
    if val, found := rm.cache.Get(key); found {
        return val, nil
    }
    
    // Query database
    var result interface{}
    err := rm.pool.QueryRowContext(ctx, 
        "SELECT data FROM resources WHERE key = $1", key).Scan(&result)
    
    if err == nil {
        rm.cache.Set(key, result, cache.DefaultExpiration)
    }
    
    return result, err
}
```

## Error Handling

### Custom Error Types
```go
type McpError struct {
    Code    int    `json:"code"`
    Message string `json:"message"`
    Details any    `json:"details,omitempty"`
}

func (e *McpError) Error() string {
    return fmt.Sprintf("MCP Error %d: %s", e.Code, e.Message)
}

func NewMcpError(code int, message string, details ...any) *McpError {
    err := &McpError{
        Code:    code,
        Message: message,
    }
    if len(details) > 0 {
        err.Details = details[0]
    }
    return err
}

// Error codes
const (
    ErrInvalidParams = -32602
    ErrInternal      = -32603
    ErrTimeout       = -32001
    ErrRateLimit     = -32002
)
```

### Error Middleware
```go
func ErrorMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic recovered: %v", err)
                
                mcpErr := NewMcpError(ErrInternal, "Internal server error")
                respondWithError(w, mcpErr)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}
```

## Testing

### Unit Tests
```go
func TestGreetHandler(t *testing.T) {
    tests := []struct {
        name    string
        params  map[string]interface{}
        want    string
        wantErr bool
    }{
        {
            name:    "valid name",
            params:  map[string]interface{}{"name": "Alice"},
            want:    "Hello, Alice!",
            wantErr: false,
        },
        {
            name:    "missing name",
            params:  map[string]interface{}{},
            want:    "",
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := greetHandler(context.Background(), tt.params)
            
            if (err != nil) != tt.wantErr {
                t.Errorf("greetHandler() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            
            if !tt.wantErr && got != tt.want {
                t.Errorf("greetHandler() = %v, want %v", got, tt.want)
            }
        })
    }
}
```

### Integration Tests
```go
func TestServerIntegration(t *testing.T) {
    // Start test server
    srv := NewTestServer(t)
    defer srv.Close()
    
    // Test tool invocation
    resp, err := srv.CallTool("greet", map[string]interface{}{
        "name": "Test",
    })
    
    require.NoError(t, err)
    assert.Equal(t, "Hello, Test!", resp)
}
```

## Performance

### Benchmarking
```go
func BenchmarkGreetHandler(b *testing.B) {
    params := map[string]interface{}{"name": "Benchmark"}
    ctx := context.Background()
    
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, _ = greetHandler(ctx, params)
    }
}
```

### Profiling
```go
import _ "net/http/pprof"

func main() {
    // Enable profiling
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // Start MCP server
    startServer()
}
```

## Configuration

### Using Viper
```go
type Config struct {
    Server   ServerConfig   `mapstructure:"server"`
    Database DatabaseConfig `mapstructure:"database"`
    MCP      MCPConfig      `mapstructure:"mcp"`
}

func LoadConfig(path string) (*Config, error) {
    viper.SetConfigFile(path)
    viper.SetEnvPrefix("MCP")
    viper.AutomaticEnv()
    
    if err := viper.ReadInConfig(); err != nil {
        return nil, err
    }
    
    var config Config
    if err := viper.Unmarshal(&config); err != nil {
        return nil, err
    }
    
    return &config, nil
}
```

## Build and Deploy

### Makefile
```makefile
.PHONY: build test run

build:
	go build -o bin/server cmd/server/main.go

test:
	go test -v -cover ./...

run:
	go run cmd/server/main.go

docker:
	docker build -t my-mcp-server .

clean:
	rm -rf bin/
```

## Next Steps

- 🏗️ [Project Structure](structure.md)
- 🔧 [Implementation Details](implementation.md)
- 🧪 [Testing Strategies](testing.md)
- 📦 [Building & Packaging](building.md)
- 🎯 [Go Best Practices](best-practices.md)