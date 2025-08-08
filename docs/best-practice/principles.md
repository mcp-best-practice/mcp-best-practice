# MCP Design Principles

## Core Principles

These fundamental principles guide the design and implementation of effective MCP servers and should inform all development decisions.

## 1. Single Responsibility Principle

### Definition
Each MCP server should have one clear, well-defined purpose and focus on a specific domain or service.

### Guidelines
- **One Domain Per Server**: GitHub server, Database server, Email server
- **Cohesive Functionality**: All tools within a server should be related
- **Clear Boundaries**: Easy to explain what the server does in one sentence

### ✅ Good Examples
```
✅ GitHub MCP Server
   Tools: create_issue, get_pull_request, merge_branch, list_repositories

✅ Database MCP Server  
   Tools: query_table, insert_record, update_record, get_schema

✅ Email MCP Server
   Tools: send_email, list_inbox, search_messages, create_draft
```

### ❌ Anti-Patterns
```
❌ "Everything" MCP Server
   Tools: create_github_issue, send_email, query_database, upload_file
   Problem: Mixed responsibilities, hard to maintain and test

❌ Over-Specific Server
   Tools: create_urgent_bug_report_for_frontend_team
   Problem: Too narrow, not reusable
```

### Benefits
- Easier testing and debugging
- Clearer ownership and responsibilities  
- Better reusability across applications
- Simpler deployment and scaling

## 2. Stateless Design

### Definition
MCP servers should avoid storing state between requests and be designed for stateless operation.

### Guidelines
- **No Persistent State**: Don't store request data between calls
- **Idempotent Operations**: Same input produces same output
- **External State Only**: Use databases, APIs, or files for persistence
- **Session Independence**: Each request is self-contained

### Implementation
```python
# ✅ Good: Stateless operation
@server.call_tool()
async def get_user_data(name: str, arguments: dict):
    user_id = arguments["user_id"]
    # Fetch from external system
    user = await database.get_user(user_id)
    return [TextContent(type="text", text=str(user))]

# ❌ Bad: Storing state
class BadServer:
    def __init__(self):
        self.user_cache = {}  # Problematic state storage
    
    async def get_user_data(self, user_id):
        if user_id in self.user_cache:  # Relies on internal state
            return self.user_cache[user_id]
```

### Benefits
- Better scalability and load balancing
- Easier debugging and testing
- More reliable under high load
- Simpler deployment patterns

## 3. Fail Fast and Explicit

### Definition
Detect and report errors as early as possible with clear, actionable error messages.

### Guidelines
- **Input Validation**: Validate parameters before processing
- **Clear Error Messages**: Specific, actionable error descriptions  
- **Structured Errors**: Use consistent error formats
- **Early Detection**: Check preconditions before expensive operations

### Implementation
```python
@server.call_tool()
async def process_file(name: str, arguments: dict):
    file_path = arguments.get("file_path")
    
    # ✅ Validate early and explicitly
    if not file_path:
        raise ValueError("file_path parameter is required")
    
    if not file_path.endswith('.json'):
        raise ValueError("Only JSON files are supported")
    
    if not os.path.exists(file_path):
        raise FileNotFoundError(f"File not found: {file_path}")
    
    # Continue with processing...
```

### Error Response Format
```python
# Use structured error responses
{
    "error": {
        "code": -32602,  # JSON-RPC error code
        "message": "Invalid file format",
        "data": {
            "parameter": "file_path",
            "expected": "*.json",
            "received": "document.txt"
        }
    }
}
```

## 4. Self-Documenting APIs

### Definition
Tools, resources, and prompts should be self-describing through comprehensive metadata.

### Guidelines
- **Descriptive Names**: Clear, unambiguous tool and parameter names
- **Comprehensive Descriptions**: Explain purpose, behavior, and side effects
- **Rich Schemas**: Detailed input/output specifications
- **Usage Examples**: Include examples in descriptions when helpful

### Implementation
```python
Tool(
    name="search_github_issues",
    description="Search for GitHub issues using flexible query criteria. Supports text search, labels, assignees, and state filters.",
    inputSchema={
        "type": "object",
        "properties": {
            "repository": {
                "type": "string", 
                "description": "Repository in format 'owner/repo' (e.g., 'microsoft/vscode')"
            },
            "query": {
                "type": "string",
                "description": "Search query text. Supports GitHub search syntax."
            },
            "state": {
                "type": "string",
                "enum": ["open", "closed", "all"],
                "default": "open",
                "description": "Issue state to filter by"
            },
            "labels": {
                "type": "array",
                "items": {"type": "string"},
                "description": "Array of label names to filter by (AND logic)"
            }
        },
        "required": ["repository"]
    }
)
```

## 5. Graceful Degradation

### Definition
Servers should handle errors and edge cases gracefully, providing partial results when possible.

### Guidelines
- **Partial Success**: Return available data even if some operations fail
- **Timeout Handling**: Set reasonable timeouts and handle them gracefully
- **Rate Limiting**: Respect API limits and retry appropriately
- **Circuit Breaker**: Fail fast when external services are unavailable

### Implementation
```python
@server.call_tool()
async def get_user_profiles(name: str, arguments: dict):
    user_ids = arguments["user_ids"]
    results = []
    errors = []
    
    for user_id in user_ids:
        try:
            profile = await api_client.get_profile(user_id)
            results.append(profile)
        except APIError as e:
            errors.append(f"Failed to fetch user {user_id}: {e}")
            continue  # Continue with other users
    
    # Return partial results with error summary
    response = {
        "profiles": results,
        "successful_count": len(results),
        "total_count": len(user_ids)
    }
    
    if errors:
        response["errors"] = errors
    
    return [TextContent(type="text", text=json.dumps(response))]
```

## 6. Security by Default

### Definition
Implement security best practices as the default behavior, not as an optional feature.

### Guidelines
- **Input Sanitization**: Always validate and sanitize inputs
- **Least Privilege**: Request minimal necessary permissions
- **No Secrets in Logs**: Never log sensitive information
- **Secure Defaults**: Err on the side of being more restrictive

### Implementation
```python
import re
from pathlib import Path

@server.call_tool()
async def read_file(name: str, arguments: dict):
    file_path = arguments["file_path"]
    
    # ✅ Security validations
    # Prevent path traversal
    if ".." in file_path or file_path.startswith("/"):
        raise ValueError("Invalid file path: path traversal not allowed")
    
    # Restrict to allowed directories
    allowed_dirs = ["/app/data", "/app/uploads"]
    abs_path = Path(file_path).resolve()
    
    if not any(str(abs_path).startswith(allowed) for allowed in allowed_dirs):
        raise ValueError("File access outside allowed directories")
    
    # Check file size before reading
    if abs_path.stat().st_size > 10_000_000:  # 10MB limit
        raise ValueError("File too large to process")
```

## 7. Observable Operations

### Definition
Provide visibility into server operations through structured logging, metrics, and health checks.

### Guidelines
- **Structured Logging**: Use consistent, machine-readable log formats
- **Operation Tracing**: Log request lifecycle events
- **Health Endpoints**: Provide health and readiness checks
- **Performance Metrics**: Track response times and error rates

### Implementation
```python
import logging
import time
from contextvars import ContextVar

# Set up structured logging
logger = logging.getLogger(__name__)
request_id: ContextVar[str] = ContextVar('request_id')

@server.call_tool()
async def process_request(name: str, arguments: dict):
    req_id = f"req_{int(time.time() * 1000)}"
    request_id.set(req_id)
    
    logger.info("tool_execution_started", extra={
        "tool": name,
        "request_id": req_id,
        "args_size": len(str(arguments))
    })
    
    start_time = time.time()
    try:
        result = await execute_tool(name, arguments)
        
        logger.info("tool_execution_completed", extra={
            "tool": name,
            "request_id": req_id,
            "duration_ms": int((time.time() - start_time) * 1000),
            "success": True
        })
        
        return result
        
    except Exception as e:
        logger.error("tool_execution_failed", extra={
            "tool": name,
            "request_id": req_id,
            "duration_ms": int((time.time() - start_time) * 1000),
            "error": str(e),
            "error_type": type(e).__name__
        })
        raise
```

## 8. Resource Efficiency

### Definition
Use system resources responsibly and implement appropriate limits and cleanup.

### Guidelines
- **Memory Limits**: Set bounds on memory usage
- **Connection Pooling**: Reuse connections to external services
- **Cleanup Resources**: Properly close files, connections, and handles
- **Timeout Operations**: Don't let operations run indefinitely

### Implementation
```python
import asyncio
import aiofiles
from contextlib import asynccontextmanager

class ResourceManager:
    def __init__(self):
        self.connection_pool = None
        self.max_concurrent = 10
        self.semaphore = asyncio.Semaphore(self.max_concurrent)
    
    @asynccontextmanager
    async def get_connection(self):
        async with self.semaphore:  # Limit concurrent connections
            conn = await self.connection_pool.acquire()
            try:
                yield conn
            finally:
                await self.connection_pool.release(conn)
    
    async def process_file(self, file_path: str):
        # Use async file handling with automatic cleanup
        async with aiofiles.open(file_path, 'r') as f:
            # Process file with memory-efficient streaming
            async for line in f:
                if len(line) > 10000:  # Skip overly long lines
                    continue
                yield line.strip()
```

## Applying These Principles

### During Design
- **Principle-First Design**: Consider these principles when designing your MCP server
- **Trade-off Decisions**: When principles conflict, document your reasoning
- **Regular Review**: Revisit designs against these principles as requirements evolve

### During Implementation
- **Code Reviews**: Check implementations against these principles
- **Testing Strategy**: Test for principle adherence (security, error handling, etc.)
- **Documentation**: Document how your server follows these principles

### During Operation
- **Monitoring**: Track metrics that validate principle adherence
- **Incident Analysis**: Review incidents for principle violations
- **Continuous Improvement**: Refactor when principles are violated

## Principle Trade-offs

Sometimes principles conflict. Here's how to handle common tensions:

### Performance vs. Security
- **Default**: Choose security, optimize later
- **Exception**: When performance is critical and risks are well understood

### Simplicity vs. Flexibility  
- **Default**: Start simple, add flexibility when needed
- **Exception**: When future requirements are well known

### Stateless vs. Performance
- **Default**: Choose stateless design
- **Exception**: When caching provides significant benefits and complexity is manageable

Remember: principles guide decisions but shouldn't be followed blindly. Use judgment and document your reasoning when deviating from these principles.