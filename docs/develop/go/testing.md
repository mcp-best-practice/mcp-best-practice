# Go Testing Guide

## Testing Strategies for Go MCP Servers

Comprehensive testing ensures your Go MCP server is reliable, maintainable, and performs well under various conditions.

## Testing Stack

### Core Testing Dependencies
```go
// go.mod
module github.com/your-org/mcp-server-go

go 1.21

require (
    github.com/modelcontextprotocol/go-sdk v0.1.0
    github.com/stretchr/testify v1.8.4
    github.com/DATA-DOG/go-sqlmock v1.5.0
    github.com/jarcoal/httpmock v1.3.1
    github.com/golang/mock v1.6.0
)

require (
    github.com/davecgh/go-spew v1.1.1 // indirect
    github.com/pmezard/go-difflib v1.0.0 // indirect
    gopkg.in/yaml.v3 v3.0.1 // indirect
)
```

### Test Configuration
```go
// internal/testutil/config.go
package testutil

import (
    "os"
    "path/filepath"
    "testing"

    "github.com/stretchr/testify/require"
    "github.com/your-org/mcp-server-go/internal/config"
)

// NewTestConfig creates a configuration suitable for testing
func NewTestConfig(t *testing.T) *config.Config {
    tmpDir := t.TempDir()
    
    return &config.Config{
        ServerName:  "test-server",
        Version:     "test",
        Transport:   "stdio",
        HTTPAddr:    ":0", // Random available port
        LogLevel:    "debug",
        LogFormat:   "text",
        Database: config.DatabaseConfig{
            URL:         "sqlite://file:test.db?mode=memory&cache=shared",
            MaxConns:    1,
            MaxIdle:     1,
            ConnTimeout: 5,
        },
    }
}

// SetupTestDir creates a temporary directory for testing
func SetupTestDir(t *testing.T, files map[string]string) string {
    tmpDir := t.TempDir()
    
    for filename, content := range files {
        filePath := filepath.Join(tmpDir, filename)
        require.NoError(t, os.MkdirAll(filepath.Dir(filePath), 0755))
        require.NoError(t, os.WriteFile(filePath, []byte(content), 0644))
    }
    
    return tmpDir
}
```

## Unit Testing

### Testing Tool Implementations
```go
// internal/tools/echo_test.go
package tools

import (
    "context"
    "testing"

    "github.com/modelcontextprotocol/go-sdk/server"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestEchoTool_Name(t *testing.T) {
    tool := &EchoTool{}
    assert.Equal(t, "echo", tool.Name())
}

func TestEchoTool_Description(t *testing.T) {
    tool := &EchoTool{}
    assert.NotEmpty(t, tool.Description())
}

func TestEchoTool_InputSchema(t *testing.T) {
    tool := &EchoTool{}
    schema := tool.InputSchema()
    
    assert.Equal(t, "object", schema["type"])
    
    properties, ok := schema["properties"].(map[string]interface{})
    require.True(t, ok)
    
    textProp, ok := properties["text"].(map[string]interface{})
    require.True(t, ok)
    assert.Equal(t, "string", textProp["type"])
    
    required, ok := schema["required"].([]string)
    require.True(t, ok)
    assert.Contains(t, required, "text")
}

func TestEchoTool_Execute(t *testing.T) {
    tests := []struct {
        name      string
        arguments map[string]interface{}
        wantErr   bool
        wantText  string
    }{
        {
            name:      "valid text",
            arguments: map[string]interface{}{"text": "Hello, World!"},
            wantErr:   false,
            wantText:  "Echo: Hello, World!",
        },
        {
            name:      "empty text",
            arguments: map[string]interface{}{"text": ""},
            wantErr:   false,
            wantText:  "Echo: ",
        },
        {
            name:      "missing text argument",
            arguments: map[string]interface{}{},
            wantErr:   true,
        },
        {
            name:      "invalid text type",
            arguments: map[string]interface{}{"text": 123},
            wantErr:   true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tool := &EchoTool{}
            ctx := context.Background()
            
            result, err := tool.Execute(ctx, tt.arguments)
            
            if tt.wantErr {
                assert.Error(t, err)
                assert.Nil(t, result)
            } else {
                assert.NoError(t, err)
                require.Len(t, result, 1)
                assert.Equal(t, "text", result[0].Type)
                assert.Equal(t, tt.wantText, result[0].Text)
            }
        })
    }
}

func TestEchoTool_ExecuteWithContext(t *testing.T) {
    tool := &EchoTool{}
    
    // Test context cancellation
    ctx, cancel := context.WithCancel(context.Background())
    cancel()
    
    arguments := map[string]interface{}{"text": "test"}
    result, err := tool.Execute(ctx, arguments)
    
    // Echo tool doesn't check context, so it should still work
    assert.NoError(t, err)
    assert.Len(t, result, 1)
}
```

### Testing Database Tools with Mocks
```go
// internal/tools/database_test.go
package tools

import (
    "context"
    "database/sql"
    "testing"

    "github.com/DATA-DOG/go-sqlmock"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestDatabaseTool_Execute(t *testing.T) {
    // Create mock database
    db, mock, err := sqlmock.New()
    require.NoError(t, err)
    defer db.Close()

    tool := &DatabaseTool{db: db}

    tests := []struct {
        name        string
        arguments   map[string]interface{}
        setupMock   func(sqlmock.Sqlmock)
        wantErr     bool
        wantResults int
    }{
        {
            name:      "valid select query",
            arguments: map[string]interface{}{"query": "SELECT id, name FROM users"},
            setupMock: func(m sqlmock.Sqlmock) {
                rows := sqlmock.NewRows([]string{"id", "name"}).
                    AddRow(1, "Alice").
                    AddRow(2, "Bob")
                m.ExpectQuery("SELECT id, name FROM users LIMIT 100").WillReturnRows(rows)
            },
            wantErr:     false,
            wantResults: 1,
        },
        {
            name:      "invalid query - not SELECT",
            arguments: map[string]interface{}{"query": "DELETE FROM users"},
            setupMock: func(m sqlmock.Sqlmock) {
                // No mock setup needed as validation should fail first
            },
            wantErr: true,
        },
        {
            name:      "database error",
            arguments: map[string]interface{}{"query": "SELECT * FROM nonexistent"},
            setupMock: func(m sqlmock.Sqlmock) {
                m.ExpectQuery("SELECT \\* FROM nonexistent LIMIT 100").
                    WillReturnError(sql.ErrNoRows)
            },
            wantErr: true,
        },
        {
            name:      "query with custom limit",
            arguments: map[string]interface{}{"query": "SELECT * FROM users", "limit": float64(50)},
            setupMock: func(m sqlmock.Sqlmock) {
                rows := sqlmock.NewRows([]string{"id"}).AddRow(1)
                m.ExpectQuery("SELECT \\* FROM users LIMIT 50").WillReturnRows(rows)
            },
            wantErr:     false,
            wantResults: 1,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            tt.setupMock(mock)
            
            ctx := context.Background()
            result, err := tool.Execute(ctx, tt.arguments)
            
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                assert.Len(t, result, tt.wantResults)
            }
            
            // Verify all expectations were met
            assert.NoError(t, mock.ExpectationsWereMet())
        })
    }
}

func TestDatabaseTool_validateQuery(t *testing.T) {
    tool := &DatabaseTool{}

    tests := []struct {
        name    string
        query   string
        wantErr bool
    }{
        {
            name:    "valid select",
            query:   "SELECT * FROM users",
            wantErr: false,
        },
        {
            name:    "valid select with whitespace",
            query:   "  SELECT id FROM users  ",
            wantErr: false,
        },
        {
            name:    "invalid - delete",
            query:   "DELETE FROM users",
            wantErr: true,
        },
        {
            name:    "invalid - insert",
            query:   "INSERT INTO users VALUES (1, 'test')",
            wantErr: true,
        },
        {
            name:    "invalid - drop hidden in select",
            query:   "SELECT * FROM users; DROP TABLE users;",
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := tool.validateQuery(tt.query)
            
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

### Testing HTTP Tools
```go
// internal/tools/http_test.go
package tools

import (
    "context"
    "encoding/json"
    "net/http"
    "testing"

    "github.com/jarcoal/httpmock"
    "github.com/modelcontextprotocol/go-sdk/server"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestHTTPTool_Execute(t *testing.T) {
    // Setup HTTP mock
    httpmock.Activate()
    defer httpmock.DeactivateAndReset()

    tool := NewHTTPTool()

    tests := []struct {
        name        string
        arguments   map[string]interface{}
        setupMock   func()
        wantErr     bool
        wantStatus  int
    }{
        {
            name:      "successful GET request",
            arguments: map[string]interface{}{"url": "http://example.com/api/users"},
            setupMock: func() {
                httpmock.RegisterResponder("GET", "http://example.com/api/users",
                    httpmock.NewStringResponder(200, `{"users": ["alice", "bob"]}`))
            },
            wantErr:    false,
            wantStatus: 200,
        },
        {
            name: "successful POST request",
            arguments: map[string]interface{}{
                "method": "POST",
                "url":    "http://example.com/api/users",
                "body":   `{"name": "charlie"}`,
                "headers": map[string]interface{}{
                    "Content-Type": "application/json",
                },
            },
            setupMock: func() {
                httpmock.RegisterResponder("POST", "http://example.com/api/users",
                    httpmock.NewStringResponder(201, `{"id": 123, "name": "charlie"}`))
            },
            wantErr:    false,
            wantStatus: 201,
        },
        {
            name:      "404 error",
            arguments: map[string]interface{}{"url": "http://example.com/api/notfound"},
            setupMock: func() {
                httpmock.RegisterResponder("GET", "http://example.com/api/notfound",
                    httpmock.NewStringResponder(404, `{"error": "Not found"}`))
            },
            wantErr:    false,
            wantStatus: 404,
        },
        {
            name:      "invalid URL",
            arguments: map[string]interface{}{"url": "not-a-url"},
            setupMock: func() {},
            wantErr:   true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            httpmock.Reset()
            tt.setupMock()
            
            ctx := context.Background()
            result, err := tool.Execute(ctx, tt.arguments)
            
            if tt.wantErr {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
                require.Len(t, result, 1)
                
                var response map[string]interface{}
                err := json.Unmarshal([]byte(result[0].Text), &response)
                require.NoError(t, err)
                
                statusCode, ok := response["status_code"].(float64)
                require.True(t, ok)
                assert.Equal(t, float64(tt.wantStatus), statusCode)
            }
        })
    }
}

func TestHTTPTool_ExecuteWithTimeout(t *testing.T) {
    httpmock.Activate()
    defer httpmock.DeactivateAndReset()

    // Setup slow response
    httpmock.RegisterResponder("GET", "http://example.com/slow",
        httpmock.NewStringResponder(200, "slow response").Delay(100))

    tool := NewHTTPTool()
    ctx := context.Background()

    arguments := map[string]interface{}{
        "url":     "http://example.com/slow",
        "timeout": float64(1), // 1 second timeout
    }

    result, err := tool.Execute(ctx, arguments)
    
    // Should succeed as 100ms delay is within 1 second timeout
    assert.NoError(t, err)
    assert.Len(t, result, 1)
}
```

## Integration Testing

### Testing Complete Server
```go
// test/integration/server_test.go
package integration

import (
    "context"
    "encoding/json"
    "testing"

    "github.com/modelcontextprotocol/go-sdk/server"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    
    "github.com/your-org/mcp-server-go/internal/config"
    mcpserver "github.com/your-org/mcp-server-go/internal/server"
    "github.com/your-org/mcp-server-go/internal/testutil"
)

func TestServer_Integration(t *testing.T) {
    // Setup test configuration
    cfg := testutil.NewTestConfig(t)
    
    // Create server
    srv, err := mcpserver.New(cfg)
    require.NoError(t, err)

    ctx := context.Background()

    // Test tool listing
    toolsResponse, err := srv.HandleListTools(ctx)
    require.NoError(t, err)
    assert.NotEmpty(t, toolsResponse.Tools)

    // Test tool execution
    callRequest := &server.CallToolRequest{
        Params: server.CallToolParams{
            Name: "echo",
            Arguments: map[string]interface{}{
                "text": "integration test",
            },
        },
    }

    callResponse, err := srv.HandleCallTool(ctx, callRequest)
    require.NoError(t, err)
    require.Len(t, callResponse.Content, 1)
    assert.Contains(t, callResponse.Content[0].Text, "integration test")
}

func TestServer_ErrorHandling(t *testing.T) {
    cfg := testutil.NewTestConfig(t)
    srv, err := mcpserver.New(cfg)
    require.NoError(t, err)

    ctx := context.Background()

    // Test unknown tool
    callRequest := &server.CallToolRequest{
        Params: server.CallToolParams{
            Name:      "nonexistent_tool",
            Arguments: map[string]interface{}{},
        },
    }

    _, err = srv.HandleCallTool(ctx, callRequest)
    assert.Error(t, err)
    assert.Contains(t, err.Error(), "unknown tool")
}

func TestServer_ResourceHandling(t *testing.T) {
    cfg := testutil.NewTestConfig(t)
    srv, err := mcpserver.New(cfg)
    require.NoError(t, err)

    ctx := context.Background()

    // Test resource listing
    resourcesResponse, err := srv.HandleListResources(ctx)
    require.NoError(t, err)
    
    if len(resourcesResponse.Resources) > 0 {
        // Test reading first resource
        readRequest := &server.ReadResourceRequest{
            Params: server.ReadResourceParams{
                URI: resourcesResponse.Resources[0].URI,
            },
        }

        readResponse, err := srv.HandleReadResource(ctx, readRequest)
        require.NoError(t, err)
        assert.NotEmpty(t, readResponse.Contents)
    }
}
```

### Testing with External Services
```go
// test/integration/external_test.go
package integration

import (
    "context"
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    
    "github.com/your-org/mcp-server-go/internal/tools"
)

func TestHTTPTool_RealServer(t *testing.T) {
    // Create test HTTP server
    testServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.Header().Set("Content-Type", "application/json")
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"message": "test response"}`))
    }))
    defer testServer.Close()

    // Test HTTP tool against real server
    tool := tools.NewHTTPTool()
    ctx := context.Background()

    arguments := map[string]interface{}{
        "url": testServer.URL,
    }

    result, err := tool.Execute(ctx, arguments)
    require.NoError(t, err)
    require.Len(t, result, 1)
    
    assert.Contains(t, result[0].Text, "test response")
}
```

## Benchmark Tests

### Performance Testing
```go
// internal/tools/echo_bench_test.go
package tools

import (
    "context"
    "testing"
)

func BenchmarkEchoTool_Execute(b *testing.B) {
    tool := &EchoTool{}
    ctx := context.Background()
    arguments := map[string]interface{}{
        "text": "benchmark test message",
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, err := tool.Execute(ctx, arguments)
        if err != nil {
            b.Fatal(err)
        }
    }
}

func BenchmarkEchoTool_ExecuteParallel(b *testing.B) {
    tool := &EchoTool{}
    arguments := map[string]interface{}{
        "text": "benchmark test message",
    }

    b.RunParallel(func(pb *testing.PB) {
        ctx := context.Background()
        for pb.Next() {
            _, err := tool.Execute(ctx, arguments)
            if err != nil {
                b.Fatal(err)
            }
        }
    })
}

func BenchmarkToolRegistry_CallTool(b *testing.B) {
    registry := NewRegistry()
    registry.Register(&EchoTool{})

    ctx := context.Background()
    arguments := map[string]interface{}{
        "text": "benchmark test",
    }

    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _, err := registry.CallTool(ctx, "echo", arguments)
        if err != nil {
            b.Fatal(err)
        }
    }
}
```

## Test Utilities and Helpers

### Mock Interfaces
```go
// internal/testutil/mocks.go
//go:generate mockgen -source=../tools/tools.go -destination=mocks.go

package testutil

import (
    "context"

    "github.com/golang/mock/gomock"
    "github.com/modelcontextprotocol/go-sdk/server"
)

// MockTool is a mock implementation of the Tool interface
type MockTool struct {
    ctrl     *gomock.Controller
    recorder *MockToolMockRecorder
}

type MockToolMockRecorder struct {
    mock *MockTool
}

func NewMockTool(ctrl *gomock.Controller) *MockTool {
    mock := &MockTool{ctrl: ctrl}
    mock.recorder = &MockToolMockRecorder{mock}
    return mock
}

func (m *MockTool) EXPECT() *MockToolMockRecorder {
    return m.recorder
}

func (m *MockTool) Name() string {
    m.ctrl.T.Helper()
    ret := m.ctrl.Call(m, "Name")
    ret0, _ := ret[0].(string)
    return ret0
}

func (mr *MockToolMockRecorder) Name() *gomock.Call {
    mr.mock.ctrl.T.Helper()
    return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "Name", reflect.TypeOf((*MockTool)(nil).Name))
}

func (m *MockTool) Description() string {
    m.ctrl.T.Helper()
    ret := m.ctrl.Call(m, "Description")
    ret0, _ := ret[0].(string)
    return ret0
}

func (mr *MockToolMockRecorder) Description() *gomock.Call {
    mr.mock.ctrl.T.Helper()
    return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "Description", reflect.TypeOf((*MockTool)(nil).Description))
}

func (m *MockTool) InputSchema() map[string]interface{} {
    m.ctrl.T.Helper()
    ret := m.ctrl.Call(m, "InputSchema")
    ret0, _ := ret[0].(map[string]interface{})
    return ret0
}

func (mr *MockToolMockRecorder) InputSchema() *gomock.Call {
    mr.mock.ctrl.T.Helper()
    return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "InputSchema", reflect.TypeOf((*MockTool)(nil).InputSchema))
}

func (m *MockTool) Execute(ctx context.Context, arguments map[string]interface{}) ([]server.Content, error) {
    m.ctrl.T.Helper()
    ret := m.ctrl.Call(m, "Execute", ctx, arguments)
    ret0, _ := ret[0].([]server.Content)
    ret1, _ := ret[1].(error)
    return ret0, ret1
}

func (mr *MockToolMockRecorder) Execute(ctx, arguments interface{}) *gomock.Call {
    mr.mock.ctrl.T.Helper()
    return mr.mock.ctrl.RecordCallWithMethodType(mr.mock, "Execute", reflect.TypeOf((*MockTool)(nil).Execute), ctx, arguments)
}
```

## Test Configuration

### Makefile Test Targets
```makefile
# Test targets
.PHONY: test test-unit test-integration test-bench test-race test-coverage

# Run all tests
test: test-unit test-integration

# Run unit tests only
test-unit:
	$(GOTEST) -short ./...

# Run integration tests
test-integration:
	$(GOTEST) -tags=integration ./test/integration/...

# Run benchmark tests
test-bench:
	$(GOTEST) -bench=. -benchmem ./...

# Run tests with race detection
test-race:
	$(GOTEST) -race ./...

# Generate test coverage report
test-coverage:
	$(GOTEST) -race -coverprofile=coverage.out ./...
	$(GOCMD) tool cover -html=coverage.out -o coverage.html
	$(GOCMD) tool cover -func=coverage.out

# Run tests with verbose output
test-verbose:
	$(GOTEST) -v ./...

# Run tests for specific package
test-pkg:
	$(GOTEST) -v ./$(PKG)/...
```

## Best Practices

### Test Organization
1. **Package-level tests**: Place tests in the same package as the code being tested
2. **Integration tests**: Separate integration tests in their own package
3. **Test data**: Use `testdata/` directories for test fixtures
4. **Parallel tests**: Use `t.Parallel()` for independent tests

### Assertions and Validation
1. **Use testify**: Leverage testify for cleaner assertions
2. **Descriptive names**: Write clear, descriptive test function names
3. **Table-driven tests**: Use table-driven tests for multiple scenarios
4. **Error checking**: Always check and assert on errors appropriately

### Performance and Reliability
1. **Benchmark critical paths**: Benchmark performance-critical code
2. **Race detection**: Always run tests with race detection enabled
3. **Context usage**: Test context cancellation and timeouts
4. **Resource cleanup**: Ensure proper cleanup of resources in tests

This comprehensive testing approach ensures your Go MCP server is robust, performant, and maintainable.