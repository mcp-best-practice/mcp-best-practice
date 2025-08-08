# Go Implementation Guide

## Building MCP Servers in Go

This guide covers implementation patterns, best practices, and examples for building robust MCP servers using Go.

## Core Implementation Patterns

### Server Implementation
```go
// internal/server/handlers.go
package server

import (
    "context"
    "fmt"
    "log"

    "github.com/modelcontextprotocol/go-sdk/server"
    "github.com/your-org/mcp-server-go/internal/tools"
)

func (s *Server) handleListTools(ctx context.Context) (*server.ListToolsResponse, error) {
    toolList := s.toolRegistry.ListTools()
    
    return &server.ListToolsResponse{
        Tools: toolList,
    }, nil
}

func (s *Server) handleCallTool(ctx context.Context, req *server.CallToolRequest) (*server.CallToolResponse, error) {
    log.Printf("Calling tool: %s with arguments: %v", req.Params.Name, req.Params.Arguments)
    
    content, err := s.toolRegistry.CallTool(ctx, req.Params.Name, req.Params.Arguments)
    if err != nil {
        return nil, fmt.Errorf("tool execution failed: %w", err)
    }

    return &server.CallToolResponse{
        Content: content,
    }, nil
}

func (s *Server) handleListResources(ctx context.Context) (*server.ListResourcesResponse, error) {
    resources := s.resourceRegistry.ListResources()
    
    return &server.ListResourcesResponse{
        Resources: resources,
    }, nil
}

func (s *Server) handleReadResource(ctx context.Context, req *server.ReadResourceRequest) (*server.ReadResourceResponse, error) {
    content, err := s.resourceRegistry.ReadResource(ctx, req.Params.URI)
    if err != nil {
        return nil, fmt.Errorf("resource read failed: %w", err)
    }

    return &server.ReadResourceResponse{
        Contents: content,
    }, nil
}
```

### Advanced Tool Implementation
```go
// internal/tools/database.go
package tools

import (
    "context"
    "database/sql"
    "encoding/json"
    "fmt"
    "strings"

    "github.com/modelcontextprotocol/go-sdk/server"
    _ "github.com/lib/pq" // PostgreSQL driver
)

type DatabaseTool struct {
    db *sql.DB
}

func NewDatabaseTool(databaseURL string) (*DatabaseTool, error) {
    db, err := sql.Open("postgres", databaseURL)
    if err != nil {
        return nil, fmt.Errorf("failed to connect to database: %w", err)
    }

    if err := db.Ping(); err != nil {
        return nil, fmt.Errorf("failed to ping database: %w", err)
    }

    return &DatabaseTool{db: db}, nil
}

func (t *DatabaseTool) Name() string {
    return "query_database"
}

func (t *DatabaseTool) Description() string {
    return "Execute SQL queries against the database. Only SELECT statements are allowed."
}

func (t *DatabaseTool) InputSchema() map[string]interface{} {
    return map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "query": map[string]interface{}{
                "type":        "string",
                "description": "SQL query to execute (SELECT only)",
            },
            "limit": map[string]interface{}{
                "type":        "integer",
                "description": "Maximum number of rows to return",
                "default":     100,
                "minimum":     1,
                "maximum":     1000,
            },
        },
        "required": []string{"query"},
    }
}

func (t *DatabaseTool) Execute(ctx context.Context, arguments map[string]interface{}) ([]server.Content, error) {
    query, ok := arguments["query"].(string)
    if !ok {
        return nil, fmt.Errorf("query argument must be a string")
    }

    // Security: Only allow SELECT statements
    if err := t.validateQuery(query); err != nil {
        return nil, err
    }

    limit := 100
    if l, ok := arguments["limit"].(float64); ok {
        limit = int(l)
    }

    // Add LIMIT clause if not present
    if !strings.Contains(strings.ToUpper(query), "LIMIT") {
        query = fmt.Sprintf("%s LIMIT %d", query, limit)
    }

    rows, err := t.db.QueryContext(ctx, query)
    if err != nil {
        return nil, fmt.Errorf("query execution failed: %w", err)
    }
    defer rows.Close()

    results, err := t.scanRows(rows)
    if err != nil {
        return nil, fmt.Errorf("failed to scan results: %w", err)
    }

    resultJSON, err := json.MarshalIndent(results, "", "  ")
    if err != nil {
        return nil, fmt.Errorf("failed to marshal results: %w", err)
    }

    return []server.Content{
        {
            Type: "text",
            Text: string(resultJSON),
        },
    }, nil
}

func (t *DatabaseTool) validateQuery(query string) error {
    upperQuery := strings.ToUpper(strings.TrimSpace(query))
    
    // Only allow SELECT statements
    if !strings.HasPrefix(upperQuery, "SELECT") {
        return fmt.Errorf("only SELECT statements are allowed")
    }

    // Block dangerous keywords
    dangerousKeywords := []string{
        "DROP", "DELETE", "INSERT", "UPDATE", "ALTER", 
        "CREATE", "TRUNCATE", "REPLACE", "MERGE",
    }

    for _, keyword := range dangerousKeywords {
        if strings.Contains(upperQuery, keyword) {
            return fmt.Errorf("query contains forbidden keyword: %s", keyword)
        }
    }

    return nil
}

func (t *DatabaseTool) scanRows(rows *sql.Rows) ([]map[string]interface{}, error) {
    columns, err := rows.Columns()
    if err != nil {
        return nil, err
    }

    var results []map[string]interface{}

    for rows.Next() {
        values := make([]interface{}, len(columns))
        valuePtrs := make([]interface{}, len(columns))
        
        for i := range columns {
            valuePtrs[i] = &values[i]
        }

        if err := rows.Scan(valuePtrs...); err != nil {
            return nil, err
        }

        row := make(map[string]interface{})
        for i, col := range columns {
            row[col] = t.convertValue(values[i])
        }

        results = append(results, row)
    }

    return results, rows.Err()
}

func (t *DatabaseTool) convertValue(value interface{}) interface{} {
    if value == nil {
        return nil
    }

    switch v := value.(type) {
    case []byte:
        return string(v)
    default:
        return v
    }
}

func (t *DatabaseTool) Close() error {
    return t.db.Close()
}
```

### HTTP Client Tool
```go
// internal/tools/http.go
package tools

import (
    "bytes"
    "context"
    "encoding/json"
    "fmt"
    "io"
    "net/http"
    "net/url"
    "time"

    "github.com/modelcontextprotocol/go-sdk/server"
)

type HTTPTool struct {
    client *http.Client
}

func NewHTTPTool() *HTTPTool {
    return &HTTPTool{
        client: &http.Client{
            Timeout: 30 * time.Second,
        },
    }
}

func (t *HTTPTool) Name() string {
    return "http_request"
}

func (t *HTTPTool) Description() string {
    return "Make HTTP requests to external APIs. Supports GET, POST, PUT, DELETE methods."
}

func (t *HTTPTool) InputSchema() map[string]interface{} {
    return map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "method": map[string]interface{}{
                "type":        "string",
                "enum":        []string{"GET", "POST", "PUT", "DELETE"},
                "description": "HTTP method to use",
                "default":     "GET",
            },
            "url": map[string]interface{}{
                "type":        "string",
                "description": "URL to make the request to",
                "format":      "uri",
            },
            "headers": map[string]interface{}{
                "type":        "object",
                "description": "HTTP headers to include",
            },
            "body": map[string]interface{}{
                "type":        "string",
                "description": "Request body (for POST/PUT requests)",
            },
            "timeout": map[string]interface{}{
                "type":        "integer",
                "description": "Request timeout in seconds",
                "default":     30,
                "minimum":     1,
                "maximum":     300,
            },
        },
        "required": []string{"url"},
    }
}

func (t *HTTPTool) Execute(ctx context.Context, arguments map[string]interface{}) ([]server.Content, error) {
    reqURL, ok := arguments["url"].(string)
    if !ok {
        return nil, fmt.Errorf("url argument must be a string")
    }

    // Validate URL
    if _, err := url.Parse(reqURL); err != nil {
        return nil, fmt.Errorf("invalid URL: %w", err)
    }

    method := "GET"
    if m, ok := arguments["method"].(string); ok {
        method = m
    }

    // Set timeout
    timeout := 30 * time.Second
    if t, ok := arguments["timeout"].(float64); ok {
        timeout = time.Duration(t) * time.Second
    }

    client := &http.Client{Timeout: timeout}

    // Prepare request body
    var body io.Reader
    if b, ok := arguments["body"].(string); ok && b != "" {
        body = bytes.NewReader([]byte(b))
    }

    req, err := http.NewRequestWithContext(ctx, method, reqURL, body)
    if err != nil {
        return nil, fmt.Errorf("failed to create request: %w", err)
    }

    // Add headers
    if headers, ok := arguments["headers"].(map[string]interface{}); ok {
        for key, value := range headers {
            if strValue, ok := value.(string); ok {
                req.Header.Set(key, strValue)
            }
        }
    }

    // Set default content type for POST/PUT
    if (method == "POST" || method == "PUT") && req.Header.Get("Content-Type") == "" {
        req.Header.Set("Content-Type", "application/json")
    }

    resp, err := client.Do(req)
    if err != nil {
        return nil, fmt.Errorf("request failed: %w", err)
    }
    defer resp.Body.Close()

    responseBody, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("failed to read response body: %w", err)
    }

    result := map[string]interface{}{
        "status_code": resp.StatusCode,
        "status":      resp.Status,
        "headers":     resp.Header,
        "body":        string(responseBody),
    }

    resultJSON, err := json.MarshalIndent(result, "", "  ")
    if err != nil {
        return nil, fmt.Errorf("failed to marshal response: %w", err)
    }

    return []server.Content{
        {
            Type: "text",
            Text: string(resultJSON),
        },
    }, nil
}
```

### File System Tool
```go
// internal/tools/filesystem.go
package tools

import (
    "context"
    "fmt"
    "os"
    "path/filepath"
    "strings"

    "github.com/modelcontextprotocol/go-sdk/server"
)

type FileSystemTool struct {
    allowedPaths []string
}

func NewFileSystemTool(allowedPaths []string) *FileSystemTool {
    return &FileSystemTool{
        allowedPaths: allowedPaths,
    }
}

func (t *FileSystemTool) Name() string {
    return "read_file"
}

func (t *FileSystemTool) Description() string {
    return "Read the contents of a text file from the filesystem."
}

func (t *FileSystemTool) InputSchema() map[string]interface{} {
    return map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "path": map[string]interface{}{
                "type":        "string",
                "description": "Path to the file to read",
            },
        },
        "required": []string{"path"},
    }
}

func (t *FileSystemTool) Execute(ctx context.Context, arguments map[string]interface{}) ([]server.Content, error) {
    filePath, ok := arguments["path"].(string)
    if !ok {
        return nil, fmt.Errorf("path argument must be a string")
    }

    // Security: Validate file path
    if err := t.validatePath(filePath); err != nil {
        return nil, err
    }

    content, err := os.ReadFile(filePath)
    if err != nil {
        if os.IsNotExist(err) {
            return nil, fmt.Errorf("file not found: %s", filePath)
        }
        if os.IsPermission(err) {
            return nil, fmt.Errorf("permission denied: %s", filePath)
        }
        return nil, fmt.Errorf("failed to read file: %w", err)
    }

    return []server.Content{
        {
            Type: "text",
            Text: string(content),
        },
    }, nil
}

func (t *FileSystemTool) validatePath(filePath string) error {
    // Prevent path traversal attacks
    if strings.Contains(filePath, "..") {
        return fmt.Errorf("path traversal not allowed")
    }

    // Check if path is within allowed directories
    if len(t.allowedPaths) > 0 {
        absPath, err := filepath.Abs(filePath)
        if err != nil {
            return fmt.Errorf("invalid path: %w", err)
        }

        allowed := false
        for _, allowedPath := range t.allowedPaths {
            absAllowed, err := filepath.Abs(allowedPath)
            if err != nil {
                continue
            }

            if strings.HasPrefix(absPath, absAllowed) {
                allowed = true
                break
            }
        }

        if !allowed {
            return fmt.Errorf("path not in allowed directories: %s", filePath)
        }
    }

    return nil
}
```

### Resource Implementation
```go
// internal/resources/config.go
package resources

import (
    "context"
    "encoding/json"
    "fmt"

    "github.com/modelcontextprotocol/go-sdk/server"
    "github.com/your-org/mcp-server-go/internal/config"
)

type ConfigResource struct {
    config *config.Config
}

func NewConfigResource(cfg *config.Config) *ConfigResource {
    return &ConfigResource{config: cfg}
}

func (r *ConfigResource) ListResources() []server.Resource {
    return []server.Resource{
        {
            URI:         "config://server",
            Name:        "Server Configuration",
            Description: "Current server configuration settings",
            MimeType:    "application/json",
        },
        {
            URI:         "config://database",
            Name:        "Database Configuration", 
            Description: "Database connection settings",
            MimeType:    "application/json",
        },
    }
}

func (r *ConfigResource) ReadResource(ctx context.Context, uri string) ([]server.ResourceContent, error) {
    switch uri {
    case "config://server":
        return r.getServerConfig()
    case "config://database":
        return r.getDatabaseConfig()
    default:
        return nil, fmt.Errorf("unknown resource URI: %s", uri)
    }
}

func (r *ConfigResource) getServerConfig() ([]server.ResourceContent, error) {
    configData := map[string]interface{}{
        "server_name": r.config.ServerName,
        "version":     r.config.Version,
        "transport":   r.config.Transport,
        "http_addr":   r.config.HTTPAddr,
        "log_level":   r.config.LogLevel,
        "log_format":  r.config.LogFormat,
    }

    jsonData, err := json.MarshalIndent(configData, "", "  ")
    if err != nil {
        return nil, fmt.Errorf("failed to marshal server config: %w", err)
    }

    return []server.ResourceContent{
        {
            URI:      "config://server",
            MimeType: "application/json",
            Text:     string(jsonData),
        },
    }, nil
}

func (r *ConfigResource) getDatabaseConfig() ([]server.ResourceContent, error) {
    // Don't expose sensitive information like passwords
    configData := map[string]interface{}{
        "max_conns":    r.config.Database.MaxConns,
        "max_idle":     r.config.Database.MaxIdle,
        "conn_timeout": r.config.Database.ConnTimeout,
        "url_scheme":   "postgresql", // Only show scheme, not full URL
    }

    jsonData, err := json.MarshalIndent(configData, "", "  ")
    if err != nil {
        return nil, fmt.Errorf("failed to marshal database config: %w", err)
    }

    return []server.ResourceContent{
        {
            URI:      "config://database",
            MimeType: "application/json",
            Text:     string(jsonData),
        },
    }, nil
}
```

## Error Handling Patterns

### Custom Error Types
```go
// internal/errors/errors.go
package errors

import "fmt"

type MCPError struct {
    Code    string
    Message string
    Err     error
}

func (e *MCPError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("%s: %s: %v", e.Code, e.Message, e.Err)
    }
    return fmt.Sprintf("%s: %s", e.Code, e.Message)
}

func (e *MCPError) Unwrap() error {
    return e.Err
}

// Common error types
func NewValidationError(message string, err error) *MCPError {
    return &MCPError{
        Code:    "VALIDATION_ERROR",
        Message: message,
        Err:     err,
    }
}

func NewExecutionError(message string, err error) *MCPError {
    return &MCPError{
        Code:    "EXECUTION_ERROR",
        Message: message,
        Err:     err,
    }
}

func NewPermissionError(message string) *MCPError {
    return &MCPError{
        Code:    "PERMISSION_ERROR",
        Message: message,
    }
}
```

## Logging Implementation
```go
// internal/logging/logger.go
package logging

import (
    "context"
    "log/slog"
    "os"
)

type Logger struct {
    *slog.Logger
}

func New(level string, format string) *Logger {
    var handler slog.Handler

    opts := &slog.HandlerOptions{
        Level: parseLevel(level),
    }

    switch format {
    case "json":
        handler = slog.NewJSONHandler(os.Stdout, opts)
    default:
        handler = slog.NewTextHandler(os.Stdout, opts)
    }

    return &Logger{
        Logger: slog.New(handler),
    }
}

func parseLevel(level string) slog.Level {
    switch level {
    case "debug":
        return slog.LevelDebug
    case "info":
        return slog.LevelInfo
    case "warn":
        return slog.LevelWarn
    case "error":
        return slog.LevelError
    default:
        return slog.LevelInfo
    }
}

func (l *Logger) WithRequest(ctx context.Context, requestID string) *slog.Logger {
    return l.With("request_id", requestID)
}
```

## Testing Helpers
```go
// internal/testutil/testutil.go
package testutil

import (
    "context"
    "testing"

    "github.com/stretchr/testify/assert"
    "github.com/your-org/mcp-server-go/internal/config"
    "github.com/your-org/mcp-server-go/internal/server"
)

func NewTestServer(t *testing.T) *server.Server {
    cfg := &config.Config{
        ServerName: "test-server",
        Version:    "test",
        Transport:  "stdio",
        LogLevel:   "debug",
    }

    srv, err := server.New(cfg)
    assert.NoError(t, err)

    return srv
}

func NewTestContext() context.Context {
    return context.Background()
}

type MockTool struct {
    NameValue        string
    DescriptionValue string
    SchemaValue      map[string]interface{}
    ExecuteFunc      func(context.Context, map[string]interface{}) ([]server.Content, error)
}

func (m *MockTool) Name() string {
    return m.NameValue
}

func (m *MockTool) Description() string {
    return m.DescriptionValue
}

func (m *MockTool) InputSchema() map[string]interface{} {
    return m.SchemaValue
}

func (m *MockTool) Execute(ctx context.Context, args map[string]interface{}) ([]server.Content, error) {
    if m.ExecuteFunc != nil {
        return m.ExecuteFunc(ctx, args)
    }
    return nil, nil
}
```

This implementation guide provides robust patterns for building production-ready MCP servers in Go with proper error handling, logging, and testing support.