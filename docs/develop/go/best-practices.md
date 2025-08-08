# Go Best Practices for MCP Servers

## Go-Specific Best Practices

This guide covers Go-specific best practices, patterns, and conventions for building robust, maintainable, and performant MCP servers.

## Code Organization and Structure

### Package Design Principles
```go
// ✅ Good: Clear, focused packages
package tools      // Contains tool implementations
package server     // Contains server logic  
package config     // Contains configuration management
package storage    // Contains data storage abstractions

// ❌ Bad: Generic, unfocused packages
package utils      // Too generic, unclear purpose
package helpers    // Vague, could contain anything
package stuff      // Meaningless name
```

### Interface Design
```go
// ✅ Good: Small, focused interfaces
type Tool interface {
    Name() string
    Description() string
    Execute(context.Context, map[string]interface{}) ([]Content, error)
}

type Storage interface {
    Store(ctx context.Context, key string, value []byte) error
    Retrieve(ctx context.Context, key string) ([]byte, error)
}

// ❌ Bad: Large, monolithic interface
type Everything interface {
    Name() string
    Description() string
    Execute(context.Context, map[string]interface{}) ([]Content, error)
    Store(ctx context.Context, key string, value []byte) error
    Retrieve(ctx context.Context, key string) ([]byte, error)
    Connect() error
    Disconnect() error
    Validate() error
}
```

### Dependency Injection
```go
// ✅ Good: Constructor injection with interfaces
type Server struct {
    tools    ToolRegistry
    storage  Storage
    logger   Logger
    config   *Config
}

func NewServer(tools ToolRegistry, storage Storage, logger Logger, config *Config) *Server {
    return &Server{
        tools:   tools,
        storage: storage,
        logger:  logger,
        config:  config,
    }
}

// ❌ Bad: Global variables and tight coupling
var (
    globalDB     *sql.DB
    globalLogger *log.Logger
    globalConfig *Config
)

func (s *Server) doSomething() {
    // Using global variables makes testing difficult
    result := globalDB.Query("SELECT * FROM users")
    globalLogger.Println("Query executed")
}
```

## Error Handling

### Structured Error Types
```go
// Define custom error types with context
type ValidationError struct {
    Field   string
    Value   interface{}
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed for field %s: %s", e.Field, e.Message)
}

type ToolExecutionError struct {
    ToolName string
    Cause    error
    Context  map[string]interface{}
}

func (e *ToolExecutionError) Error() string {
    return fmt.Sprintf("tool %s execution failed: %v", e.ToolName, e.Cause)
}

func (e *ToolExecutionError) Unwrap() error {
    return e.Cause
}
```

### Error Wrapping and Context
```go
// ✅ Good: Wrap errors with context
func (t *DatabaseTool) Execute(ctx context.Context, args map[string]interface{}) ([]Content, error) {
    query, err := t.validateQuery(args)
    if err != nil {
        return nil, fmt.Errorf("query validation failed: %w", err)
    }

    results, err := t.executeQuery(ctx, query)
    if err != nil {
        return nil, fmt.Errorf("query execution failed for tool %s: %w", t.Name(), err)
    }

    return results, nil
}

// ❌ Bad: Losing error context
func (t *DatabaseTool) Execute(ctx context.Context, args map[string]interface{}) ([]Content, error) {
    query, err := t.validateQuery(args)
    if err != nil {
        return nil, err // Lost context about where this failed
    }

    results, err := t.executeQuery(ctx, query)
    if err != nil {
        return nil, errors.New("query failed") // Lost original error information
    }

    return results, nil
}
```

### Sentinel Errors and Error Checking
```go
// Define sentinel errors for common conditions
var (
    ErrToolNotFound      = errors.New("tool not found")
    ErrInvalidArguments  = errors.New("invalid arguments")
    ErrResourceNotFound  = errors.New("resource not found")
    ErrUnauthorized      = errors.New("unauthorized access")
)

// ✅ Good: Use errors.Is for checking
func (r *ToolRegistry) GetTool(name string) (Tool, error) {
    tool, exists := r.tools[name]
    if !exists {
        return nil, fmt.Errorf("tool %s: %w", name, ErrToolNotFound)
    }
    return tool, nil
}

func handleToolCall(registry *ToolRegistry, name string) {
    tool, err := registry.GetTool(name)
    if errors.Is(err, ErrToolNotFound) {
        // Handle tool not found specifically
        log.Printf("Tool %s not available", name)
        return
    }
    if err != nil {
        // Handle other errors
        log.Printf("Error getting tool: %v", err)
        return
    }
    // Use tool...
}
```

## Concurrency Patterns

### Context Usage
```go
// ✅ Good: Proper context usage
func (s *Server) handleRequest(ctx context.Context, req *Request) (*Response, error) {
    // Set timeout for the entire operation
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    // Pass context through all operations
    result, err := s.processRequest(ctx, req)
    if err != nil {
        if errors.Is(err, context.Canceled) {
            return nil, fmt.Errorf("request canceled: %w", err)
        }
        if errors.Is(err, context.DeadlineExceeded) {
            return nil, fmt.Errorf("request timed out: %w", err)
        }
        return nil, fmt.Errorf("request processing failed: %w", err)
    }

    return result, nil
}

func (s *Server) processRequest(ctx context.Context, req *Request) (*Response, error) {
    // Check context before expensive operations
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }

    // Pass context to database operations
    return s.db.QueryContext(ctx, req.Query)
}
```

### Graceful Shutdown
```go
func (s *Server) Run(ctx context.Context) error {
    // Create server
    server := &http.Server{
        Addr:    s.config.HTTPAddr,
        Handler: s.router,
    }

    // Start server in goroutine
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            s.logger.Error("Server failed to start", "error", err)
        }
    }()

    s.logger.Info("Server started", "addr", s.config.HTTPAddr)

    // Wait for context cancellation
    <-ctx.Done()

    s.logger.Info("Shutting down server...")

    // Graceful shutdown with timeout
    shutdownCtx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()

    if err := server.Shutdown(shutdownCtx); err != nil {
        s.logger.Error("Server shutdown failed", "error", err)
        return err
    }

    s.logger.Info("Server stopped")
    return nil
}
```

### Worker Pools
```go
// Worker pool for concurrent tool execution
type ToolWorkerPool struct {
    workers    int
    jobChan    chan *ToolJob
    resultChan chan *ToolResult
    ctx        context.Context
    cancel     context.CancelFunc
    wg         sync.WaitGroup
}

type ToolJob struct {
    ID        string
    Tool      Tool
    Arguments map[string]interface{}
    Context   context.Context
}

type ToolResult struct {
    ID      string
    Content []Content
    Error   error
}

func NewToolWorkerPool(workers int) *ToolWorkerPool {
    ctx, cancel := context.WithCancel(context.Background())
    
    pool := &ToolWorkerPool{
        workers:    workers,
        jobChan:    make(chan *ToolJob, workers*2),
        resultChan: make(chan *ToolResult, workers*2),
        ctx:        ctx,
        cancel:     cancel,
    }

    // Start workers
    for i := 0; i < workers; i++ {
        pool.wg.Add(1)
        go pool.worker(i)
    }

    return pool
}

func (p *ToolWorkerPool) worker(id int) {
    defer p.wg.Done()
    
    for {
        select {
        case job := <-p.jobChan:
            content, err := job.Tool.Execute(job.Context, job.Arguments)
            
            result := &ToolResult{
                ID:      job.ID,
                Content: content,
                Error:   err,
            }
            
            select {
            case p.resultChan <- result:
            case <-p.ctx.Done():
                return
            case <-job.Context.Done():
                result.Error = job.Context.Err()
                select {
                case p.resultChan <- result:
                case <-p.ctx.Done():
                    return
                }
            }
            
        case <-p.ctx.Done():
            return
        }
    }
}

func (p *ToolWorkerPool) Submit(job *ToolJob) {
    select {
    case p.jobChan <- job:
    case <-p.ctx.Done():
    case <-job.Context.Done():
    }
}

func (p *ToolWorkerPool) GetResult() *ToolResult {
    select {
    case result := <-p.resultChan:
        return result
    case <-p.ctx.Done():
        return nil
    }
}

func (p *ToolWorkerPool) Close() {
    p.cancel()
    p.wg.Wait()
    close(p.jobChan)
    close(p.resultChan)
}
```

## Performance Optimization

### Memory Management
```go
// ✅ Good: Reuse buffers and minimize allocations
type BufferPool struct {
    pool sync.Pool
}

func NewBufferPool() *BufferPool {
    return &BufferPool{
        pool: sync.Pool{
            New: func() interface{} {
                return make([]byte, 0, 1024)
            },
        },
    }
}

func (p *BufferPool) Get() []byte {
    return p.pool.Get().([]byte)
}

func (p *BufferPool) Put(buf []byte) {
    if cap(buf) > 64*1024 { // Don't pool very large buffers
        return
    }
    p.pool.Put(buf[:0]) // Reset length but keep capacity
}

// Usage in tool execution
func (t *JSONTool) Execute(ctx context.Context, args map[string]interface{}) ([]Content, error) {
    buf := t.bufferPool.Get()
    defer t.bufferPool.Put(buf)

    // Use buf for JSON marshaling...
    result := json.Marshal(data) // This would use the pooled buffer
    
    return []Content{{Type: "text", Text: string(result)}}, nil
}
```

### Efficient String Building
```go
// ✅ Good: Use strings.Builder for string concatenation
func buildResponse(data []Record) string {
    var builder strings.Builder
    builder.Grow(len(data) * 100) // Pre-allocate capacity

    builder.WriteString("Results:\n")
    for i, record := range data {
        builder.WriteString(fmt.Sprintf("%d. %s: %s\n", i+1, record.Key, record.Value))
    }

    return builder.String()
}

// ❌ Bad: String concatenation in loops
func buildResponseBad(data []Record) string {
    result := "Results:\n"
    for i, record := range data {
        result += fmt.Sprintf("%d. %s: %s\n", i+1, record.Key, record.Value) // Creates new string each iteration
    }
    return result
}
```

### Caching Patterns
```go
// LRU Cache with expiration
type CacheItem struct {
    Value      interface{}
    Expiration time.Time
}

type LRUCache struct {
    mu       sync.RWMutex
    items    map[string]*list.Element
    lru      *list.List
    maxItems int
    ttl      time.Duration
}

type cacheEntry struct {
    key   string
    value CacheItem
}

func NewLRUCache(maxItems int, ttl time.Duration) *LRUCache {
    return &LRUCache{
        items:    make(map[string]*list.Element),
        lru:      list.New(),
        maxItems: maxItems,
        ttl:      ttl,
    }
}

func (c *LRUCache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()

    elem, exists := c.items[key]
    if !exists {
        return nil, false
    }

    entry := elem.Value.(*cacheEntry)
    
    // Check expiration
    if time.Now().After(entry.value.Expiration) {
        c.mu.RUnlock()
        c.mu.Lock()
        c.removeElement(elem)
        c.mu.Unlock()
        c.mu.RLock()
        return nil, false
    }

    // Move to front (most recently used)
    c.lru.MoveToFront(elem)
    
    return entry.value.Value, true
}

func (c *LRUCache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()

    now := time.Now()
    expiration := now.Add(c.ttl)

    if elem, exists := c.items[key]; exists {
        // Update existing item
        entry := elem.Value.(*cacheEntry)
        entry.value.Value = value
        entry.value.Expiration = expiration
        c.lru.MoveToFront(elem)
        return
    }

    // Add new item
    entry := &cacheEntry{
        key: key,
        value: CacheItem{
            Value:      value,
            Expiration: expiration,
        },
    }

    elem := c.lru.PushFront(entry)
    c.items[key] = elem

    // Evict if over capacity
    if len(c.items) > c.maxItems {
        oldest := c.lru.Back()
        if oldest != nil {
            c.removeElement(oldest)
        }
    }
}

func (c *LRUCache) removeElement(elem *list.Element) {
    entry := elem.Value.(*cacheEntry)
    delete(c.items, entry.key)
    c.lru.Remove(elem)
}
```

## Security Best Practices

### Input Validation
```go
import (
    "net/url"
    "regexp"
    "unicode/utf8"
)

// Validation functions
func validateEmail(email string) error {
    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    if !emailRegex.MatchString(email) {
        return fmt.Errorf("invalid email format")
    }
    return nil
}

func validateURL(urlStr string) error {
    u, err := url.Parse(urlStr)
    if err != nil {
        return fmt.Errorf("invalid URL: %w", err)
    }
    
    if u.Scheme != "http" && u.Scheme != "https" {
        return fmt.Errorf("unsupported URL scheme: %s", u.Scheme)
    }
    
    return nil
}

func validateStringLength(s string, maxLength int) error {
    if !utf8.ValidString(s) {
        return fmt.Errorf("invalid UTF-8 string")
    }
    
    if utf8.RuneCountInString(s) > maxLength {
        return fmt.Errorf("string too long: max %d characters", maxLength)
    }
    
    return nil
}

// Tool with comprehensive validation
type ValidatedTool struct {
    name string
}

func (t *ValidatedTool) Execute(ctx context.Context, args map[string]interface{}) ([]Content, error) {
    // Validate each argument type and content
    email, ok := args["email"].(string)
    if !ok {
        return nil, &ValidationError{Field: "email", Message: "must be a string"}
    }
    
    if err := validateEmail(email); err != nil {
        return nil, &ValidationError{Field: "email", Message: err.Error()}
    }
    
    urlStr, ok := args["url"].(string)
    if !ok {
        return nil, &ValidationError{Field: "url", Message: "must be a string"}
    }
    
    if err := validateURL(urlStr); err != nil {
        return nil, &ValidationError{Field: "url", Message: err.Error()}
    }
    
    // Continue with validated inputs...
    return nil, nil
}
```

### Secure File Operations
```go
import (
    "path/filepath"
    "strings"
)

type SecureFileHandler struct {
    allowedDirs []string
    maxSize     int64
}

func (h *SecureFileHandler) validatePath(filePath string) error {
    // Prevent path traversal
    if strings.Contains(filePath, "..") {
        return fmt.Errorf("path traversal not allowed")
    }
    
    // Convert to absolute path
    absPath, err := filepath.Abs(filePath)
    if err != nil {
        return fmt.Errorf("invalid path: %w", err)
    }
    
    // Check if path is within allowed directories
    allowed := false
    for _, allowedDir := range h.allowedDirs {
        absAllowed, err := filepath.Abs(allowedDir)
        if err != nil {
            continue
        }
        
        if strings.HasPrefix(absPath, absAllowed+string(filepath.Separator)) {
            allowed = true
            break
        }
    }
    
    if !allowed {
        return fmt.Errorf("path not in allowed directories")
    }
    
    return nil
}

func (h *SecureFileHandler) readFile(filePath string) ([]byte, error) {
    if err := h.validatePath(filePath); err != nil {
        return nil, err
    }
    
    // Check file size before reading
    info, err := os.Stat(filePath)
    if err != nil {
        return nil, fmt.Errorf("file stat failed: %w", err)
    }
    
    if info.Size() > h.maxSize {
        return nil, fmt.Errorf("file too large: %d bytes (max: %d)", info.Size(), h.maxSize)
    }
    
    return os.ReadFile(filePath)
}
```

## Testing Patterns

### Table-Driven Tests
```go
func TestToolExecution(t *testing.T) {
    tests := []struct {
        name      string
        tool      Tool
        args      map[string]interface{}
        wantErr   bool
        wantCount int
        validate  func(t *testing.T, content []Content)
    }{
        {
            name: "valid echo",
            tool: &EchoTool{},
            args: map[string]interface{}{"text": "hello"},
            wantErr: false,
            wantCount: 1,
            validate: func(t *testing.T, content []Content) {
                assert.Equal(t, "text", content[0].Type)
                assert.Contains(t, content[0].Text, "hello")
            },
        },
        {
            name: "missing required argument",
            tool: &EchoTool{},
            args: map[string]interface{}{},
            wantErr: true,
        },
        {
            name: "invalid argument type",
            tool: &EchoTool{},
            args: map[string]interface{}{"text": 123},
            wantErr: true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctx := context.Background()
            content, err := tt.tool.Execute(ctx, tt.args)
            
            if tt.wantErr {
                assert.Error(t, err)
                return
            }
            
            assert.NoError(t, err)
            assert.Len(t, content, tt.wantCount)
            
            if tt.validate != nil {
                tt.validate(t, content)
            }
        })
    }
}
```

### Mock Generation and Testing
```go
//go:generate go run github.com/golang/mock/mockgen -source=tool.go -destination=mocks/mock_tool.go

func TestServerWithMocks(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    mockTool := mocks.NewMockTool(ctrl)
    mockTool.EXPECT().Name().Return("mock-tool").AnyTimes()
    mockTool.EXPECT().Execute(gomock.Any(), gomock.Any()).Return(
        []Content{{Type: "text", Text: "mock response"}}, nil,
    ).Times(1)

    registry := NewRegistry()
    registry.Register(mockTool)

    server := NewServer(registry, nil, nil)
    
    ctx := context.Background()
    result, err := server.CallTool(ctx, "mock-tool", map[string]interface{}{})
    
    assert.NoError(t, err)
    assert.Len(t, result, 1)
    assert.Equal(t, "mock response", result[0].Text)
}
```

## Documentation and Code Quality

### Effective Comments and Documentation
```go
// Package tools provides implementations of MCP tools for various operations.
// 
// Each tool implements the Tool interface and should provide:
//   - A unique name that describes its function
//   - A clear description of what it does
//   - A JSON schema for input validation
//   - Thread-safe execution logic
//
// Example usage:
//   registry := tools.NewRegistry()
//   registry.Register(&tools.EchoTool{})
//   
//   result, err := registry.CallTool(ctx, "echo", map[string]interface{}{
//       "text": "Hello, World!",
//   })
package tools

// EchoTool implements a simple echo functionality for testing and demonstration.
// It accepts a text input and returns it unchanged, prefixed with "Echo: ".
//
// This tool is primarily useful for:
//   - Testing MCP client implementations
//   - Demonstrating tool execution patterns
//   - Validating connection and protocol functionality
type EchoTool struct {
    // No fields needed for stateless echo operation
}

// Execute processes the echo request and returns the input text with a prefix.
// 
// The context parameter is respected for cancellation, though echo operations
// are typically fast enough that cancellation is rare.
//
// Arguments expected:
//   - text (string): The text to echo back (required)
//
// Returns content with type "text" containing the echoed message.
func (t *EchoTool) Execute(ctx context.Context, arguments map[string]interface{}) ([]Content, error) {
    // Implementation details...
}
```

### Linting Configuration
```yaml
# .golangci.yml
linters-settings:
  revive:
    rules:
      - name: exported
        arguments: [checkPrivateReceivers, sayRepetitiveInsteadOfStutters]
  govet:
    check-shadowing: true
  misspell:
    locale: US
  gofmt:
    simplify: true

linters:
  enable:
    - revive
    - govet
    - misspell
    - gofmt
    - goimports
    - gosec
    - ineffassign
    - staticcheck
    - typecheck
    - unused
    - errcheck
    - goconst
    - gocyclo
    - dupl

issues:
  exclude-rules:
    - path: _test\.go
      linters:
        - gosec
        - dupl
```

Following these Go-specific best practices will result in MCP servers that are efficient, maintainable, secure, and idiomatic to the Go ecosystem.