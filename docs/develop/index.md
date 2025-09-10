# Develop — Building Production-Ready MCP Servers

Build MCP servers using clear contracts and disciplined patterns. Keep implementations simple, observable, and maintainable while following enterprise-grade development practices.

## Development Principles

### Foundation Principles

- **Single responsibility**: One domain and authentication boundary per server
- **Contracts first**: Strict input/output schemas, explicit side effects, documented errors
- **Additive change**: Version contracts; prefer backward-compatible evolution with deprecations
- **Stateless by default**: Externalize state with TTLs and PII handling; provide async status for long-running work
- **Security by design**: Integrate authentication, authorization, and audit from the start
- **Observability first**: Instrument from day one with structured logs, metrics, and traces

### Development Mindset

- **Evaluation-driven development**: Treat MCP behavior as a product; define success metrics before coding
- **Fail-fast validation**: Validate inputs rigorously; reject invalid requests immediately
- **Least-privilege integration**: Default to read-only capabilities; require explicit elevation for write operations
- **Dependency awareness**: Understand and document external service dependencies and failure modes

## Project Structure and Standards

### Standardized Repository Structure

Every MCP server should follow a consistent project layout for maintainability and operational clarity:

```
mcp-server-name/
├── Containerfile                  # Container build definition
├── Makefile                       # Build, test, and deployment automation
├── pyproject.toml                 # Project and dependency configuration
├── README.md                      # Comprehensive documentation
├── CONTRIBUTING.md                # Contribution guidelines
├── .gitignore                     # Version control exclusions
├── docs/                          # Additional documentation and specs
├── tests/                         # Unit and integration tests
│   ├── test_main.py              # Entry point tests
│   └── test_tools.py             # Tool functionality tests
└── src/                          # Application source code
    └── mcp_server_name/          # Main package
        ├── main.py               # Entry point and server initialization
        ├── server.py             # Server logic and tool registration
        └── tools/                # Tool implementations
            ├── tools.py          # Business logic
            └── registry.py       # MCP tool registration
```

### Self-Containment Requirements

- **Standalone repositories**: Each server must include all necessary code and documentation
- **One-command setup**: `git clone; make install serve` should work out-of-the-box
- **Dependency management**: Use `pyproject.toml` for Python projects; pin all dependencies
- **Clear role definition**: State the specific purpose and boundaries of your server

## Development Workflow

### 1. Planning and Design Phase

**Define outcomes first:**
- Identify the specific problem domain and user needs
- Define success metrics and evaluation criteria
- Document the API contract before implementation
- Plan for error scenarios and edge cases

**Schema-driven development:**
- Define tool schemas with strong typing (use Pydantic models)
- Specify input validation rules and constraints
- Document side effects and state changes
- Plan for backward compatibility and versioning

### 2. Implementation Phase

**Start with the contract:**
```python
from pydantic import BaseModel, Field

class ToolRequest(BaseModel):
    """Well-defined input schema"""
    text: str = Field(max_length=1000, description="Input text to process")
    options: dict = Field(default_factory=dict, description="Optional parameters")

class ToolResponse(BaseModel):
    """Structured output schema"""
    result: str = Field(description="Processed result")
    metadata: dict = Field(description="Operation metadata")
```

**Implement with validation:**
```python
@mcp.tool()
def process_text(request: ToolRequest) -> ToolResponse:
    """Process text with full validation and error handling"""
    try:
        # Input validation (beyond schema)
        if not request.text.strip():
            raise ValueError("Text cannot be empty")
        
        # Business logic
        result = perform_processing(request.text, request.options)
        
        # Return structured response
        return ToolResponse(
            result=result,
            metadata={"processed_at": datetime.now().isoformat()}
        )
    except Exception as e:
        logger.error(f"Processing failed: {e}")
        raise
```

### 3. Testing Strategy

**Multi-level testing approach:**
- Unit tests for business logic
- Integration tests for MCP protocol compliance
- End-to-end tests for complete workflows
- Contract tests for schema validation

**Testing infrastructure:**
```python
# Unit testing example
def test_tool_logic():
    """Test core business logic independently"""
    result = process_text_logic("test input", {})
    assert result == expected_output

# Integration testing with MCP framework
@pytest.mark.asyncio
async def test_mcp_tool_integration():
    """Test MCP protocol integration"""
    # Test tool discovery, schema validation, execution
    pass
```

### 4. Quality Assurance

**Code quality standards:**
- Static analysis with linters (ruff, mypy, bandit)
- Code formatting consistency
- Security vulnerability scanning
- Dependency license and vulnerability checks

**Runtime quality:**
- Performance benchmarking with realistic workloads
- Memory usage profiling
- Error rate and latency monitoring
- Cross-platform compatibility testing

## Language and Technology Choices

### Python Development

**Recommended stack:**
- **FastMCP** or official MCP SDK for rapid development
- **Pydantic** for data validation and serialization
- **asyncio** for concurrent operations
- **structured logging** with correlation IDs

**Performance considerations:**
- Use async I/O for network-bound operations
- Implement connection pooling for external services
- Consider process pools for CPU-intensive tasks
- Profile memory usage and implement caching strategically

### Go Development

**When to choose Go:**
- High-throughput, low-latency requirements
- Minimal resource footprint needed
- Strong concurrency patterns required
- Single binary deployment preferred

**Go patterns:**
- Structured error handling with clear error types
- Context propagation for cancellation and timeouts
- Graceful shutdown handling
- Comprehensive testing with table-driven tests

### TypeScript/Node.js Development

**Optimal use cases:**
- API integration and webhook handling
- Event-driven architectures
- Rapid prototyping and iteration
- JavaScript ecosystem integration

**Node.js patterns:**
- Promise-based async patterns
- Stream processing for large data
- Event emitter patterns for notifications
- Proper error boundary handling

## Configuration and Environment Management

### Environment-Driven Configuration

```python
from pydantic_settings import BaseSettings

class ServerConfig(BaseSettings):
    """Type-safe configuration management"""
    server_name: str = "mcp-server"
    server_port: int = 8000
    log_level: str = "INFO"
    debug_mode: bool = False
    
    # External service configuration
    api_key: str = Field(..., description="Required API key")
    api_base_url: str = "https://api.example.com"
    
    class Config:
        env_prefix = "MCP_"  # Environment variable prefix
```

### Secrets Management

- **Never inline secrets** in code or configuration files
- Use environment variables for local development
- Integrate with enterprise secret managers for production
- Implement secret rotation patterns
- Log secret access for audit trails (but never log the secrets themselves)

## Error Handling and Resilience

### Comprehensive Error Strategy

**Error classification:**
- **Client errors**: Invalid input, authentication failures, authorization denials
- **Server errors**: Internal failures, dependency outages, resource exhaustion
- **Business errors**: Domain-specific validation failures, state conflicts

**Error handling patterns:**
```python
from enum import Enum

class ErrorCategory(str, Enum):
    CLIENT_ERROR = "client_error"
    SERVER_ERROR = "server_error"
    BUSINESS_ERROR = "business_error"

class MCPError(Exception):
    """Base MCP error with structured information"""
    def __init__(self, message: str, category: ErrorCategory, details: dict = None):
        super().__init__(message)
        self.category = category
        self.details = details or {}
```

### Resilience Patterns

- **Circuit breakers**: Protect against cascading failures
- **Retry with backoff**: Handle transient failures gracefully  
- **Timeout management**: Set appropriate timeouts for all operations
- **Bulkhead isolation**: Isolate failure domains to prevent total outages

## Instrumentation and Observability

### Structured Logging

```python
import structlog

logger = structlog.get_logger()

@mcp.tool()
async def instrumented_tool(text: str) -> str:
    """Tool with comprehensive instrumentation"""
    correlation_id = generate_correlation_id()
    
    await logger.info(
        "Tool execution started",
        tool_name="instrumented_tool",
        correlation_id=correlation_id,
        input_length=len(text)
    )
    
    try:
        result = await process_with_timeout(text)
        
        await logger.info(
            "Tool execution completed",
            correlation_id=correlation_id,
            success=True,
            output_length=len(result)
        )
        
        return result
        
    except Exception as e:
        await logger.error(
            "Tool execution failed",
            correlation_id=correlation_id,
            error=str(e),
            error_type=type(e).__name__
        )
        raise
```

### Metrics and Monitoring

**Key metrics to track:**
- Tool success/failure rates
- Request latency percentiles (p50, p95, p99)
- Concurrent request counts
- External dependency response times
- Resource utilization (CPU, memory, connections)

### Health and Readiness Checks

```python
@app.get("/health")
async def health_check():
    """Health endpoint for monitoring"""
    checks = {
        "server": "healthy",
        "database": await check_database_health(),
        "external_api": await check_api_health(),
    }
    
    status = "healthy" if all(v == "healthy" for v in checks.values()) else "degraded"
    
    return {"status": status, "checks": checks}
```

## Security in Development

### Authentication and Authorization

```python
from functools import wraps

def require_scope(required_scope: str):
    """Decorator for tool-level authorization"""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Extract user context from request
            user_scopes = get_user_scopes_from_context()
            
            if required_scope not in user_scopes:
                raise MCPError(
                    f"Insufficient permissions: {required_scope} required",
                    ErrorCategory.CLIENT_ERROR
                )
            
            return await func(*args, **kwargs)
        return wrapper
    return decorator

@mcp.tool()
@require_scope("data:read")
async def read_sensitive_data(query: str) -> dict:
    """Tool with explicit authorization requirements"""
    pass
```

### Input Sanitization and Validation

- **Validate all inputs** against strict schemas
- **Sanitize outputs** to prevent injection attacks
- **Rate limit** requests to prevent abuse
- **Audit all operations** with sufficient detail for compliance

## Advanced Development Patterns

### Resource Management

```python
@mcp.resource("config://server-settings")
async def get_server_config() -> str:
    """Provide server configuration as a resource"""
    config = {
        "server_name": SERVER_CONFIG.server_name,
        "capabilities": ["tool_execution", "resource_access"],
        "version": "1.0.0"
    }
    return json.dumps(config, indent=2)
```

### Long-Running Operations

```python
@mcp.tool()
async def async_operation(params: dict) -> dict:
    """Handle long-running operations with progress tracking"""
    operation_id = generate_operation_id()
    
    # Start async task
    asyncio.create_task(perform_long_operation(operation_id, params))
    
    return {
        "operation_id": operation_id,
        "status": "started",
        "status_url": f"/operations/{operation_id}/status"
    }

@mcp.tool()
async def check_operation_status(operation_id: str) -> dict:
    """Check status of long-running operation"""
    status = await get_operation_status(operation_id)
    return {
        "operation_id": operation_id,
        "status": status.state,
        "progress": status.progress,
        "result": status.result if status.is_complete else None
    }
```

## Development Tools and Automation

### Essential Make Targets

Every MCP project should include a `Makefile` with standardized targets:

```makefile
# Development environment
venv:           # Create virtual environment
install:        # Install dependencies
activate:       # Show activation command

# Development workflow  
serve:          # Run server locally
test:           # Run all tests
lint:           # Run linters and formatters
docs:           # Generate documentation

# Quality assurance
test-integration: # Run integration tests
security-scan:    # Security vulnerability scan
performance-test: # Performance benchmarking

# Packaging and deployment
build:          # Build distributable package
container:      # Build container image
deploy:         # Deploy to staging/production
```

### Testing Infrastructure

```bash
# Comprehensive testing workflow
make test           # Unit and integration tests
make test-contract  # Schema and contract validation
make test-security  # Security testing
make test-perf      # Performance testing
make test-e2e       # End-to-end testing
```

## Deployment Preparation

### Containerization Best Practices

```dockerfile
# Multi-stage build for optimal image size
FROM python:3.11-slim as builder
WORKDIR /app
COPY pyproject.toml ./
RUN pip install uv && uv venv && uv pip install -e .

FROM python:3.11-slim as runtime
# Create non-root user
RUN useradd --create-home --shell /bin/bash mcp
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY src/ ./src/
USER mcp
EXPOSE 8000
CMD ["python", "-m", "src.mcp_server_name.main"]
```

### Production Readiness Checklist

✅ **Configuration**: Environment-driven, no hardcoded values  
✅ **Security**: Authentication, authorization, input validation implemented  
✅ **Observability**: Logging, metrics, health checks configured  
✅ **Error handling**: Comprehensive error handling with proper categorization  
✅ **Testing**: Unit, integration, and contract tests passing  
✅ **Documentation**: API documentation, runbooks, and troubleshooting guides  
✅ **Performance**: Latency and throughput requirements validated  
✅ **Resilience**: Circuit breakers, timeouts, and retry logic implemented  

## Development Anti-Patterns to Avoid

### Common Mistakes

❌ **Monolithic servers**: Mixing multiple domains in one server  
❌ **Hardcoded configuration**: Embedding secrets or config in code  
❌ **Poor error handling**: Generic errors without context or categorization  
❌ **Lack of instrumentation**: No logging, metrics, or observability  
❌ **Synchronous I/O**: Blocking operations in async contexts  
❌ **Version lock-in**: Tight coupling to specific framework versions  
❌ **Security afterthoughts**: Adding security as a later concern  

### Technical Debt Prevention

- **Regular dependency updates** with automated vulnerability scanning
- **Refactoring sprints** to address code quality issues
- **Performance monitoring** to catch degradation early
- **Documentation maintenance** as part of development process

---

**Next Steps**: After development, proceed to [Test](../test/) for comprehensive validation strategies, then [Package](../package/) for distribution preparation.

**See Also**: Best Practices, Package, Deploy, Operate, Secure, Use.