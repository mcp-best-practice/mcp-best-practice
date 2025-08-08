# Advanced MCP Tutorial

## Building Production-Ready MCP Servers

This tutorial covers advanced concepts for building robust, scalable MCP servers suitable for production environments.

## Prerequisites

- Completed the [Quickstart Tutorial](./quickstart.md)
- Understanding of async programming
- Basic knowledge of databases and HTTP clients
- Familiarity with configuration management

## Overview

We'll enhance our basic MCP server with:
1. **Database Integration**: PostgreSQL tool for data queries
2. **HTTP Client**: Web API integration tool
3. **File System**: Secure file operations
4. **Configuration Management**: Environment-based settings
5. **Error Handling**: Comprehensive error management
6. **Logging & Monitoring**: Observability features
7. **Security**: Authentication and authorization
8. **Testing**: Unit and integration tests

## Project Structure

```
advanced-mcp-server/
├── src/
│   ├── config/
│   │   ├── __init__.py
│   │   └── settings.py
│   ├── core/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── errors.py
│   │   └── logging.py
│   ├── tools/
│   │   ├── __init__.py
│   │   ├── database.py
│   │   ├── http_client.py
│   │   ├── filesystem.py
│   │   └── base.py
│   ├── resources/
│   │   ├── __init__.py
│   │   └── config_resource.py
│   ├── middleware/
│   │   ├── __init__.py
│   │   ├── auth.py
│   │   ├── logging.py
│   │   └── rate_limiting.py
│   └── main.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── conftest.py
├── config/
│   ├── development.yaml
│   ├── production.yaml
│   └── test.yaml
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── requirements.txt
├── requirements-dev.txt
└── README.md
```

## Step 1: Configuration Management

### Settings Configuration
```python
# src/config/settings.py
from pydantic import BaseSettings, Field
from typing import List, Optional
import os

class DatabaseConfig(BaseSettings):
    url: str = Field(..., env="DATABASE_URL")
    pool_size: int = Field(5, env="DB_POOL_SIZE")
    pool_timeout: int = Field(30, env="DB_POOL_TIMEOUT")
    
    class Config:
        env_prefix = "DB_"

class SecurityConfig(BaseSettings):
    secret_key: str = Field(..., env="SECRET_KEY")
    allowed_origins: List[str] = Field(["*"], env="ALLOWED_ORIGINS")
    rate_limit_per_minute: int = Field(60, env="RATE_LIMIT_PER_MINUTE")
    require_auth: bool = Field(False, env="REQUIRE_AUTH")
    
    class Config:
        env_prefix = "SECURITY_"

class LoggingConfig(BaseSettings):
    level: str = Field("INFO", env="LOG_LEVEL")
    format: str = Field("json", env="LOG_FORMAT")
    file_path: Optional[str] = Field(None, env="LOG_FILE_PATH")
    
    class Config:
        env_prefix = "LOG_"

class ServerSettings(BaseSettings):
    name: str = Field("advanced-mcp-server", env="SERVER_NAME")
    version: str = Field("2.0.0", env="SERVER_VERSION")
    environment: str = Field("development", env="ENVIRONMENT")
    
    # Sub-configurations
    database: DatabaseConfig = DatabaseConfig()
    security: SecurityConfig = SecurityConfig()
    logging: LoggingConfig = LoggingConfig()
    
    # Tool-specific settings
    filesystem_allowed_paths: List[str] = Field(["/tmp"], env="FILESYSTEM_ALLOWED_PATHS")
    http_timeout: int = Field(30, env="HTTP_TIMEOUT")
    http_max_retries: int = Field(3, env="HTTP_MAX_RETRIES")
    
    @classmethod
    def load_from_yaml(cls, config_file: str) -> "ServerSettings":
        import yaml
        with open(config_file, 'r') as f:
            config_data = yaml.safe_load(f)
        return cls(**config_data)
    
    class Config:
        env_file = ".env"
        case_sensitive = False

# Global settings instance
settings = ServerSettings()
```

### YAML Configuration Files
```yaml
# config/development.yaml
server:
  name: "advanced-mcp-server"
  version: "2.0.0"
  environment: "development"

database:
  url: "postgresql://user:password@localhost:5432/mcp_dev"
  pool_size: 5
  pool_timeout: 30

security:
  secret_key: "dev-secret-key-change-in-production"
  allowed_origins: ["*"]
  rate_limit_per_minute: 100
  require_auth: false

logging:
  level: "DEBUG"
  format: "pretty"
  file_path: null

filesystem_allowed_paths:
  - "/tmp"
  - "/var/data"

http_timeout: 30
http_max_retries: 3
```

## Step 2: Enhanced Error Handling

```python
# src/core/errors.py
from typing import Optional, Dict, Any
import traceback
from enum import Enum

class ErrorCode(Enum):
    VALIDATION_ERROR = "VALIDATION_ERROR"
    TOOL_EXECUTION_ERROR = "TOOL_EXECUTION_ERROR"
    RESOURCE_NOT_FOUND = "RESOURCE_NOT_FOUND"
    PERMISSION_DENIED = "PERMISSION_DENIED"
    RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED"
    AUTHENTICATION_FAILED = "AUTHENTICATION_FAILED"
    CONFIGURATION_ERROR = "CONFIGURATION_ERROR"
    INTERNAL_ERROR = "INTERNAL_ERROR"

class MCPError(Exception):
    """Base MCP error with structured information."""
    
    def __init__(
        self,
        message: str,
        code: ErrorCode = ErrorCode.INTERNAL_ERROR,
        details: Optional[Dict[str, Any]] = None,
        cause: Optional[Exception] = None
    ):
        super().__init__(message)
        self.message = message
        self.code = code
        self.details = details or {}
        self.cause = cause
        
    def to_dict(self) -> Dict[str, Any]:
        """Convert error to dictionary for JSON serialization."""
        result = {
            "error": {
                "code": self.code.value,
                "message": self.message,
                "details": self.details
            }
        }
        
        if self.cause:
            result["error"]["cause"] = str(self.cause)
            result["error"]["traceback"] = traceback.format_exception(
                type(self.cause), self.cause, self.cause.__traceback__
            )
        
        return result

class ValidationError(MCPError):
    def __init__(self, message: str, field: Optional[str] = None, **kwargs):
        details = kwargs.get('details', {})
        if field:
            details['field'] = field
        super().__init__(message, ErrorCode.VALIDATION_ERROR, details, **kwargs)

class ToolExecutionError(MCPError):
    def __init__(self, tool_name: str, message: str, **kwargs):
        details = kwargs.get('details', {})
        details['tool_name'] = tool_name
        super().__init__(
            f"Tool '{tool_name}' execution failed: {message}",
            ErrorCode.TOOL_EXECUTION_ERROR,
            details,
            **kwargs
        )

class ResourceNotFoundError(MCPError):
    def __init__(self, resource_uri: str, **kwargs):
        details = kwargs.get('details', {})
        details['resource_uri'] = resource_uri
        super().__init__(
            f"Resource '{resource_uri}' not found",
            ErrorCode.RESOURCE_NOT_FOUND,
            details,
            **kwargs
        )

class PermissionDeniedError(MCPError):
    def __init__(self, operation: str, reason: Optional[str] = None, **kwargs):
        details = kwargs.get('details', {})
        details['operation'] = operation
        if reason:
            details['reason'] = reason
        
        message = f"Permission denied for operation: {operation}"
        if reason:
            message += f" ({reason})"
            
        super().__init__(
            message,
            ErrorCode.PERMISSION_DENIED,
            details,
            **kwargs
        )

# Error handling decorator
def handle_tool_errors(func):
    """Decorator to handle tool execution errors consistently."""
    import functools
    
    @functools.wraps(func)
    async def wrapper(self, *args, **kwargs):
        try:
            return await func(self, *args, **kwargs)
        except MCPError:
            raise  # Re-raise MCP errors as-is
        except Exception as e:
            raise ToolExecutionError(
                tool_name=getattr(self, 'name', 'unknown'),
                message=str(e),
                cause=e
            )
    
    return wrapper
```

## Step 3: Structured Logging

```python
# src/core/logging.py
import logging
import logging.config
import json
import sys
from datetime import datetime
from typing import Any, Dict
from src.config.settings import settings

class JSONFormatter(logging.Formatter):
    """Custom JSON formatter for structured logging."""
    
    def format(self, record: logging.LogRecord) -> str:
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName,
            "line": record.lineno,
        }
        
        # Add extra fields
        if hasattr(record, 'tool_name'):
            log_entry['tool_name'] = record.tool_name
        if hasattr(record, 'user_id'):
            log_entry['user_id'] = record.user_id
        if hasattr(record, 'request_id'):
            log_entry['request_id'] = record.request_id
        
        # Add exception info if present
        if record.exc_info:
            log_entry['exception'] = self.formatException(record.exc_info)
        
        return json.dumps(log_entry)

class PrettyFormatter(logging.Formatter):
    """Human-readable formatter for development."""
    
    def format(self, record: logging.LogRecord) -> str:
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        level_color = {
            'DEBUG': '\033[36m',    # Cyan
            'INFO': '\033[32m',     # Green
            'WARNING': '\033[33m',  # Yellow
            'ERROR': '\033[31m',    # Red
            'CRITICAL': '\033[35m', # Magenta
        }.get(record.levelname, '')
        
        reset_color = '\033[0m'
        
        return (
            f"{timestamp} | {level_color}{record.levelname:8}{reset_color} | "
            f"{record.name:20} | {record.getMessage()}"
        )

def setup_logging():
    """Configure logging based on settings."""
    
    # Choose formatter
    if settings.logging.format == "json":
        formatter_class = JSONFormatter
    else:
        formatter_class = PrettyFormatter
    
    # Configure handlers
    handlers = {
        'console': {
            'class': 'logging.StreamHandler',
            'formatter': 'default',
            'stream': sys.stdout,
        }
    }
    
    if settings.logging.file_path:
        handlers['file'] = {
            'class': 'logging.FileHandler',
            'formatter': 'default',
            'filename': settings.logging.file_path,
        }
    
    # Logging configuration
    config = {
        'version': 1,
        'disable_existing_loggers': False,
        'formatters': {
            'default': {
                '()': formatter_class,
            }
        },
        'handlers': handlers,
        'loggers': {
            'mcp_server': {
                'handlers': list(handlers.keys()),
                'level': settings.logging.level,
                'propagate': False,
            },
            'uvicorn': {
                'handlers': ['console'],
                'level': 'INFO',
                'propagate': False,
            },
        },
        'root': {
            'handlers': list(handlers.keys()),
            'level': settings.logging.level,
        }
    }
    
    logging.config.dictConfig(config)

# Context manager for request logging
import contextvars

request_id_var: contextvars.ContextVar[str] = contextvars.ContextVar('request_id')
user_id_var: contextvars.ContextVar[str] = contextvars.ContextVar('user_id')

class LoggingContext:
    """Context manager for adding request context to logs."""
    
    def __init__(self, request_id: str = None, user_id: str = None):
        self.request_id = request_id
        self.user_id = user_id
        self.tokens = []
    
    def __enter__(self):
        if self.request_id:
            self.tokens.append(request_id_var.set(self.request_id))
        if self.user_id:
            self.tokens.append(user_id_var.set(self.user_id))
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        for token in self.tokens:
            token.reset()

class ContextualLoggerAdapter(logging.LoggerAdapter):
    """Logger adapter that automatically includes context variables."""
    
    def process(self, msg, kwargs):
        extra = kwargs.get('extra', {})
        
        try:
            extra['request_id'] = request_id_var.get()
        except LookupError:
            pass
        
        try:
            extra['user_id'] = user_id_var.get()
        except LookupError:
            pass
        
        kwargs['extra'] = extra
        return msg, kwargs

def get_logger(name: str) -> ContextualLoggerAdapter:
    """Get a contextual logger instance."""
    base_logger = logging.getLogger(name)
    return ContextualLoggerAdapter(base_logger, {})
```

## Step 4: Advanced Database Tool

```python
# src/tools/database.py
import asyncpg
import json
from typing import Dict, List, Any, Optional
from src.tools.base import BaseTool
from src.core.errors import ValidationError, ToolExecutionError, handle_tool_errors
from src.core.logging import get_logger
from src.config.settings import settings

logger = get_logger(__name__)

class DatabaseTool(BaseTool):
    """Advanced database query tool with connection pooling and security."""
    
    def __init__(self):
        super().__init__(
            name="database_query",
            description="Execute SQL queries against PostgreSQL database with safety checks"
        )
        self.pool: Optional[asyncpg.Pool] = None
        self.allowed_tables: List[str] = []  # Configure based on your needs
        
    async def initialize(self):
        """Initialize database connection pool."""
        try:
            self.pool = await asyncpg.create_pool(
                dsn=settings.database.url,
                min_size=1,
                max_size=settings.database.pool_size,
                command_timeout=settings.database.pool_timeout
            )
            logger.info("Database connection pool initialized")
            
            # Load allowed tables (example)
            async with self.pool.acquire() as conn:
                rows = await conn.fetch("""
                    SELECT table_name 
                    FROM information_schema.tables 
                    WHERE table_schema = 'public'
                """)
                self.allowed_tables = [row['table_name'] for row in rows]
                logger.info(f"Loaded {len(self.allowed_tables)} allowed tables")
                
        except Exception as e:
            logger.error(f"Failed to initialize database pool: {e}")
            raise ToolExecutionError(self.name, f"Database initialization failed: {e}", cause=e)
    
    def get_schema(self) -> Dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "SQL query to execute (SELECT only)"
                },
                "parameters": {
                    "type": "object",
                    "description": "Parameters for parameterized queries",
                    "additionalProperties": True
                },
                "limit": {
                    "type": "integer",
                    "minimum": 1,
                    "maximum": 1000,
                    "default": 100,
                    "description": "Maximum number of rows to return"
                },
                "timeout": {
                    "type": "integer",
                    "minimum": 1,
                    "maximum": 300,
                    "default": 30,
                    "description": "Query timeout in seconds"
                }
            },
            "required": ["query"]
        }
    
    def _validate_query(self, query: str) -> None:
        """Validate SQL query for security."""
        query_upper = query.strip().upper()
        
        # Only allow SELECT statements
        if not query_upper.startswith('SELECT'):
            raise ValidationError(
                "Only SELECT queries are allowed",
                field="query",
                details={"query_type": "non_select"}
            )
        
        # Block dangerous keywords
        dangerous_keywords = [
            'DROP', 'DELETE', 'INSERT', 'UPDATE', 'ALTER',
            'CREATE', 'TRUNCATE', 'REPLACE', 'EXEC',
            'EXECUTE', 'MERGE', 'CALL'
        ]
        
        for keyword in dangerous_keywords:
            if keyword in query_upper:
                raise ValidationError(
                    f"Query contains forbidden keyword: {keyword}",
                    field="query",
                    details={"forbidden_keyword": keyword}
                )
        
        # Check for suspicious patterns
        suspicious_patterns = [
            'UNION', 'INFORMATION_SCHEMA', 'PG_', 'SYS'
        ]
        
        for pattern in suspicious_patterns:
            if pattern in query_upper:
                logger.warning(f"Query contains suspicious pattern: {pattern}", extra={
                    'query': query[:100] + "..." if len(query) > 100 else query
                })
    
    @handle_tool_errors
    async def execute(self, arguments: Dict[str, Any]) -> List[Any]:
        if not self.pool:
            await self.initialize()
        
        # Extract and validate arguments
        query = arguments.get("query", "").strip()
        parameters = arguments.get("parameters", {})
        limit = arguments.get("limit", 100)
        timeout = arguments.get("timeout", 30)
        
        if not query:
            raise ValidationError("Query cannot be empty", field="query")
        
        self._validate_query(query)
        
        # Add LIMIT if not present
        if "LIMIT" not in query.upper():
            query = f"{query} LIMIT ${len(parameters) + 1}"
            parameters[f"${len(parameters) + 1}"] = limit
        
        logger.info(f"Executing database query", extra={
            'query': query[:100] + "..." if len(query) > 100 else query,
            'parameter_count': len(parameters)
        })
        
        try:
            async with self.pool.acquire() as conn:
                # Set statement timeout
                await conn.execute(f'SET statement_timeout = {timeout * 1000}')
                
                # Execute query
                if parameters:
                    rows = await conn.fetch(query, *parameters.values())
                else:
                    rows = await conn.fetch(query)
                
                # Convert to JSON-serializable format
                results = []
                for row in rows:
                    row_dict = {}
                    for key, value in row.items():
                        # Handle special types
                        if hasattr(value, 'isoformat'):  # datetime
                            value = value.isoformat()
                        elif isinstance(value, bytes):
                            value = value.hex()
                        row_dict[key] = value
                    results.append(row_dict)
                
                logger.info(f"Query executed successfully, returned {len(results)} rows")
                
                return [{
                    "type": "text",
                    "text": json.dumps({
                        "query": query,
                        "row_count": len(results),
                        "results": results
                    }, indent=2, default=str)
                }]
                
        except asyncpg.exceptions.PostgresError as e:
            logger.error(f"Database query failed: {e}")
            raise ToolExecutionError(
                self.name,
                f"Query execution failed: {e}",
                details={
                    "error_code": e.sqlstate,
                    "error_message": str(e)
                },
                cause=e
            )
        except Exception as e:
            logger.error(f"Unexpected error during query execution: {e}")
            raise ToolExecutionError(self.name, f"Unexpected error: {e}", cause=e)
    
    async def cleanup(self):
        """Clean up database connections."""
        if self.pool:
            await self.pool.close()
            logger.info("Database connection pool closed")
```

## Step 5: HTTP Client Tool with Resilience

```python
# src/tools/http_client.py
import aiohttp
import asyncio
import json
from typing import Dict, List, Any, Optional
from urllib.parse import urlparse
import time
from src.tools.base import BaseTool
from src.core.errors import ValidationError, ToolExecutionError, handle_tool_errors
from src.core.logging import get_logger
from src.config.settings import settings

logger = get_logger(__name__)

class HTTPClientTool(BaseTool):
    """Advanced HTTP client with retry logic, circuit breaker, and caching."""
    
    def __init__(self):
        super().__init__(
            name="http_request",
            description="Make HTTP requests with automatic retries and error handling"
        )
        self.session: Optional[aiohttp.ClientSession] = None
        self.circuit_breaker_state: Dict[str, Dict] = {}
        self.cache: Dict[str, Dict] = {}
        
    async def initialize(self):
        """Initialize HTTP client session."""
        timeout = aiohttp.ClientTimeout(total=settings.http_timeout)
        self.session = aiohttp.ClientSession(
            timeout=timeout,
            headers={"User-Agent": f"{settings.name}/{settings.version}"}
        )
        logger.info("HTTP client session initialized")
    
    def get_schema(self) -> Dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "method": {
                    "type": "string",
                    "enum": ["GET", "POST", "PUT", "DELETE", "PATCH", "HEAD"],
                    "default": "GET",
                    "description": "HTTP method"
                },
                "url": {
                    "type": "string",
                    "format": "uri",
                    "description": "URL to request"
                },
                "headers": {
                    "type": "object",
                    "description": "HTTP headers",
                    "additionalProperties": {"type": "string"}
                },
                "data": {
                    "type": "object",
                    "description": "Request body data (JSON)"
                },
                "params": {
                    "type": "object",
                    "description": "URL query parameters",
                    "additionalProperties": {"type": "string"}
                },
                "timeout": {
                    "type": "integer",
                    "minimum": 1,
                    "maximum": 300,
                    "default": settings.http_timeout,
                    "description": "Request timeout in seconds"
                },
                "retries": {
                    "type": "integer",
                    "minimum": 0,
                    "maximum": 10,
                    "default": settings.http_max_retries,
                    "description": "Number of retry attempts"
                },
                "cache_ttl": {
                    "type": "integer",
                    "minimum": 0,
                    "default": 0,
                    "description": "Cache TTL in seconds (0 = no cache)"
                }
            },
            "required": ["url"]
        }
    
    def _validate_url(self, url: str) -> None:
        """Validate URL for security."""
        try:
            parsed = urlparse(url)
        except Exception:
            raise ValidationError("Invalid URL format", field="url")
        
        if parsed.scheme not in ['http', 'https']:
            raise ValidationError("Only HTTP and HTTPS URLs are allowed", field="url")
        
        # Block private/local addresses (optional security measure)
        hostname = parsed.hostname
        if hostname:
            import ipaddress
            try:
                ip = ipaddress.ip_address(hostname)
                if ip.is_private or ip.is_loopback or ip.is_link_local:
                    logger.warning(f"Request to private IP address blocked: {hostname}")
                    raise ValidationError("Requests to private IP addresses are not allowed", field="url")
            except ipaddress.AddressValueError:
                pass  # Hostname is not an IP address
    
    def _get_circuit_breaker_state(self, host: str) -> Dict:
        """Get circuit breaker state for a host."""
        if host not in self.circuit_breaker_state:
            self.circuit_breaker_state[host] = {
                "failures": 0,
                "last_failure": 0,
                "state": "closed"  # closed, open, half_open
            }
        return self.circuit_breaker_state[host]
    
    def _should_allow_request(self, host: str) -> bool:
        """Check if request should be allowed based on circuit breaker."""
        state = self._get_circuit_breaker_state(host)
        current_time = time.time()
        
        if state["state"] == "open":
            # Check if we should try again (half-open)
            if current_time - state["last_failure"] > 60:  # 1 minute timeout
                state["state"] = "half_open"
                return True
            return False
        
        return True
    
    def _record_success(self, host: str):
        """Record successful request."""
        state = self._get_circuit_breaker_state(host)
        state["failures"] = 0
        state["state"] = "closed"
    
    def _record_failure(self, host: str):
        """Record failed request."""
        state = self._get_circuit_breaker_state(host)
        state["failures"] += 1
        state["last_failure"] = time.time()
        
        if state["failures"] >= 5:  # Threshold
            state["state"] = "open"
            logger.warning(f"Circuit breaker opened for host: {host}")
    
    def _get_cache_key(self, method: str, url: str, headers: Dict, params: Dict) -> str:
        """Generate cache key for request."""
        import hashlib
        key_data = f"{method}:{url}:{json.dumps(headers, sort_keys=True)}:{json.dumps(params, sort_keys=True)}"
        return hashlib.md5(key_data.encode()).hexdigest()
    
    def _get_cached_response(self, cache_key: str, cache_ttl: int) -> Optional[Dict]:
        """Get cached response if valid."""
        if cache_key in self.cache:
            cached = self.cache[cache_key]
            if time.time() - cached["timestamp"] < cache_ttl:
                logger.info("Returning cached response")
                return cached["response"]
            else:
                del self.cache[cache_key]
        return None
    
    def _cache_response(self, cache_key: str, response: Dict):
        """Cache response."""
        self.cache[cache_key] = {
            "response": response,
            "timestamp": time.time()
        }
    
    @handle_tool_errors
    async def execute(self, arguments: Dict[str, Any]) -> List[Any]:
        if not self.session:
            await self.initialize()
        
        # Extract arguments
        method = arguments.get("method", "GET").upper()
        url = arguments["url"]
        headers = arguments.get("headers", {})
        data = arguments.get("data")
        params = arguments.get("params", {})
        timeout = arguments.get("timeout", settings.http_timeout)
        retries = arguments.get("retries", settings.http_max_retries)
        cache_ttl = arguments.get("cache_ttl", 0)
        
        self._validate_url(url)
        
        # Check circuit breaker
        parsed_url = urlparse(url)
        host = parsed_url.hostname
        if not self._should_allow_request(host):
            raise ToolExecutionError(
                self.name,
                f"Circuit breaker is open for host: {host}",
                details={"host": host, "circuit_breaker_state": "open"}
            )
        
        # Check cache for GET requests
        if method == "GET" and cache_ttl > 0:
            cache_key = self._get_cache_key(method, url, headers, params)
            cached_response = self._get_cached_response(cache_key, cache_ttl)
            if cached_response:
                return [{"type": "text", "text": json.dumps(cached_response, indent=2)}]
        
        logger.info(f"Making {method} request to {url}")
        
        # Make request with retries
        last_exception = None
        for attempt in range(retries + 1):
            try:
                async with self.session.request(
                    method=method,
                    url=url,
                    headers=headers,
                    json=data if data else None,
                    params=params,
                    timeout=aiohttp.ClientTimeout(total=timeout)
                ) as response:
                    # Read response
                    try:
                        response_text = await response.text()
                        try:
                            response_data = json.loads(response_text)
                        except json.JSONDecodeError:
                            response_data = response_text
                    except Exception:
                        response_data = None
                    
                    # Build result
                    result = {
                        "status_code": response.status,
                        "headers": dict(response.headers),
                        "data": response_data,
                        "url": str(response.url),
                        "method": method,
                        "success": 200 <= response.status < 400
                    }
                    
                    # Record success/failure for circuit breaker
                    if result["success"]:
                        self._record_success(host)
                    else:
                        self._record_failure(host)
                    
                    # Cache successful GET responses
                    if method == "GET" and cache_ttl > 0 and result["success"]:
                        self._cache_response(cache_key, result)
                    
                    logger.info(f"Request completed with status {response.status}")
                    
                    return [{
                        "type": "text",
                        "text": json.dumps(result, indent=2, default=str)
                    }]
                    
            except asyncio.TimeoutError:
                last_exception = ToolExecutionError(
                    self.name,
                    f"Request timed out after {timeout} seconds"
                )
                self._record_failure(host)
            except aiohttp.ClientError as e:
                last_exception = ToolExecutionError(
                    self.name,
                    f"HTTP client error: {str(e)}",
                    cause=e
                )
                self._record_failure(host)
            except Exception as e:
                last_exception = ToolExecutionError(
                    self.name,
                    f"Unexpected error: {str(e)}",
                    cause=e
                )
                self._record_failure(host)
            
            if attempt < retries:
                wait_time = 2 ** attempt  # Exponential backoff
                logger.warning(f"Request failed, retrying in {wait_time}s (attempt {attempt + 1}/{retries + 1})")
                await asyncio.sleep(wait_time)
        
        # All retries exhausted
        raise last_exception
    
    async def cleanup(self):
        """Clean up HTTP client session."""
        if self.session:
            await self.session.close()
            logger.info("HTTP client session closed")
```

This advanced tutorial demonstrates how to build production-ready MCP servers with proper error handling, logging, configuration management, and resilience patterns. The remaining sections would continue with middleware implementation, security features, comprehensive testing, and deployment strategies.