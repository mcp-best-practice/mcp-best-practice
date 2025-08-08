# Python Best Practices

## Python-Specific Best Practices for MCP Servers

Following Python best practices ensures your MCP server is maintainable, performant, and follows community standards.

## Code Style and Formatting

### Use Black and Ruff
```bash
# Install formatting tools
uv add --dev black ruff

# Format code
black src tests

# Lint and fix issues
ruff check --fix src tests
```

### Configuration
```toml
# pyproject.toml
[tool.black]
line-length = 88
target-version = ['py311']
include = '\.pyi?$'
extend-exclude = '''
/(
  \.eggs
  | \.git
  | \.venv
  | build
  | dist
)/
'''

[tool.ruff]
target-version = "py311"
line-length = 88
select = [
    "E",  # pycodestyle errors
    "W",  # pycodestyle warnings
    "F",  # pyflakes
    "I",  # isort
    "B",  # flake8-bugbear
    "C4", # flake8-comprehensions
    "UP", # pyupgrade
]
ignore = [
    "E501",  # line too long (handled by black)
    "B008",  # do not perform function calls in argument defaults
]

[tool.ruff.per-file-ignores]
"__init__.py" = ["F401"]
"tests/**/*.py" = ["S101"]  # assert allowed in tests
```

## Type Hints and MyPy

### Comprehensive Type Hints
```python
from typing import Any, Dict, List, Optional, Union
from mcp.types import Tool, TextContent
import asyncio

class DatabaseTool:
    def __init__(self, connection_string: str) -> None:
        self.connection_string = connection_string
        self._connection: Optional[Any] = None
    
    async def execute_query(
        self, 
        query: str, 
        params: Optional[Dict[str, Any]] = None
    ) -> List[TextContent]:
        """Execute database query with type safety."""
        results = await self._execute(query, params or {})
        return [TextContent(type="text", text=str(result)) for result in results]
    
    async def _execute(
        self, 
        query: str, 
        params: Dict[str, Any]
    ) -> List[Dict[str, Any]]:
        # Implementation details
        return []
```

### MyPy Configuration
```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
strict = true
warn_return_any = true
warn_unused_configs = true
warn_redundant_casts = true
warn_unused_ignores = true
warn_no_return = true
warn_unreachable = true
show_error_codes = true

[[tool.mypy.overrides]]
module = "tests.*"
ignore_errors = true
```

## Error Handling

### Custom Exception Hierarchy
```python
# src/my_mcp_server/exceptions.py
class MCPServerError(Exception):
    """Base exception for MCP server errors."""
    pass

class ValidationError(MCPServerError):
    """Raised when input validation fails."""
    def __init__(self, message: str, field: Optional[str] = None):
        super().__init__(message)
        self.field = field

class ExternalServiceError(MCPServerError):
    """Raised when external service calls fail."""
    def __init__(self, message: str, service: str, status_code: Optional[int] = None):
        super().__init__(message)
        self.service = service
        self.status_code = status_code

class ConfigurationError(MCPServerError):
    """Raised when configuration is invalid."""
    pass
```

### Error Handling Patterns
```python
import logging
from typing import NoReturn

logger = logging.getLogger(__name__)

async def safe_tool_execution(
    tool_name: str, 
    arguments: Dict[str, Any]
) -> List[TextContent]:
    """Execute tool with comprehensive error handling."""
    try:
        # Validate inputs first
        validated_args = validate_tool_arguments(tool_name, arguments)
        
        # Execute with timeout
        result = await asyncio.wait_for(
            execute_tool_impl(tool_name, validated_args),
            timeout=30.0
        )
        
        return result
        
    except ValidationError as e:
        logger.warning(f"Validation failed for {tool_name}: {e}")
        raise  # Re-raise for proper MCP error response
        
    except ExternalServiceError as e:
        logger.error(f"External service error in {tool_name}: {e}")
        # Convert to user-friendly message
        raise MCPServerError(f"Service temporarily unavailable: {e.service}")
        
    except asyncio.TimeoutError:
        logger.error(f"Timeout executing {tool_name}")
        raise MCPServerError("Operation timed out")
        
    except Exception as e:
        logger.exception(f"Unexpected error in {tool_name}")
        raise MCPServerError("Internal server error")

def validate_tool_arguments(tool_name: str, arguments: Dict[str, Any]) -> Dict[str, Any]:
    """Validate and sanitize tool arguments."""
    if not isinstance(arguments, dict):
        raise ValidationError("Arguments must be a dictionary")
    
    # Tool-specific validation
    validators = {
        "database_query": validate_database_query,
        "file_operation": validate_file_operation,
    }
    
    validator = validators.get(tool_name)
    if validator:
        return validator(arguments)
    
    return arguments
```

## Async Programming

### Proper Async Patterns
```python
import asyncio
import aiohttp
import aiofiles
from contextlib import asynccontextmanager
from typing import AsyncGenerator

class AsyncResourceManager:
    """Manage async resources properly."""
    
    def __init__(self):
        self._session: Optional[aiohttp.ClientSession] = None
        self._db_pool: Optional[Any] = None
    
    async def __aenter__(self):
        await self.initialize()
        return self
    
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        await self.cleanup()
    
    async def initialize(self):
        """Initialize async resources."""
        self._session = aiohttp.ClientSession(
            timeout=aiohttp.ClientTimeout(total=30)
        )
        # Initialize database pool
        
    async def cleanup(self):
        """Clean up async resources."""
        if self._session:
            await self._session.close()
        # Clean up database pool

@asynccontextmanager
async def get_file_content(file_path: str) -> AsyncGenerator[str, None]:
    """Async context manager for file operations."""
    async with aiofiles.open(file_path, 'r') as f:
        content = await f.read()
        yield content

# Concurrent operations
async def process_multiple_requests(requests: List[Dict[str, Any]]) -> List[Any]:
    """Process multiple requests concurrently."""
    semaphore = asyncio.Semaphore(10)  # Limit concurrent operations
    
    async def process_single(request: Dict[str, Any]) -> Any:
        async with semaphore:
            return await process_request(request)
    
    # Use gather for concurrent execution
    results = await asyncio.gather(
        *[process_single(req) for req in requests],
        return_exceptions=True  # Handle exceptions gracefully
    )
    
    return results
```

## Configuration Management

### Pydantic Settings
```python
# src/my_mcp_server/config.py
from pydantic import BaseSettings, Field, validator
from typing import Optional, List
import os

class Settings(BaseSettings):
    """Application settings with validation."""
    
    # Server settings
    server_name: str = Field(default="my-mcp-server", description="Server name")
    debug: bool = Field(default=False, description="Debug mode")
    log_level: str = Field(default="INFO", description="Log level")
    
    # Database settings
    database_url: Optional[str] = Field(None, description="Database connection URL")
    database_pool_size: int = Field(default=10, ge=1, le=50)
    database_timeout: int = Field(default=30, ge=1, le=300)
    
    # API settings
    api_key: Optional[str] = Field(None, description="External API key")
    api_base_url: str = Field(default="https://api.example.com")
    api_timeout: int = Field(default=30, ge=1, le=300)
    
    # Feature flags
    enable_caching: bool = Field(default=True)
    enable_metrics: bool = Field(default=False)
    allowed_operations: List[str] = Field(default_factory=lambda: ["read", "write"])
    
    @validator('log_level')
    def validate_log_level(cls, v):
        valid_levels = ['DEBUG', 'INFO', 'WARNING', 'ERROR', 'CRITICAL']
        if v.upper() not in valid_levels:
            raise ValueError(f'Invalid log level. Must be one of: {valid_levels}')
        return v.upper()
    
    @validator('database_url')
    def validate_database_url(cls, v):
        if v and not v.startswith(('postgresql://', 'sqlite:///', 'mysql://')):
            raise ValueError('Invalid database URL scheme')
        return v
    
    class Config:
        env_prefix = 'MCP_'
        env_file = '.env'
        case_sensitive = False

# Global settings instance
settings = Settings()
```

## Logging

### Structured Logging
```python
# src/my_mcp_server/logging_config.py
import logging
import sys
from typing import Any, Dict
import json
from datetime import datetime

class StructuredFormatter(logging.Formatter):
    """JSON formatter for structured logging."""
    
    def format(self, record: logging.LogRecord) -> str:
        log_data: Dict[str, Any] = {
            'timestamp': datetime.utcnow().isoformat() + 'Z',
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno,
        }
        
        # Add exception info if present
        if record.exc_info:
            log_data['exception'] = self.formatException(record.exc_info)
        
        # Add extra fields
        if hasattr(record, 'extra_fields'):
            log_data.update(record.extra_fields)
        
        return json.dumps(log_data, separators=(',', ':'))

def setup_logging(level: str = 'INFO') -> None:
    """Configure application logging."""
    root_logger = logging.getLogger()
    root_logger.setLevel(level)
    
    # Remove existing handlers
    for handler in root_logger.handlers[:]:
        root_logger.removeHandler(handler)
    
    # Console handler with structured format
    handler = logging.StreamHandler(sys.stderr)
    handler.setFormatter(StructuredFormatter())
    root_logger.addHandler(handler)
    
    # Suppress noisy third-party loggers
    logging.getLogger('aiohttp').setLevel(logging.WARNING)
    logging.getLogger('asyncio').setLevel(logging.WARNING)

# Usage
logger = logging.getLogger(__name__)

async def example_with_logging():
    """Example function with structured logging."""
    logger.info(
        "Processing request", 
        extra={'extra_fields': {
            'tool_name': 'database_query',
            'user_id': 'user123',
            'request_id': 'req-456'
        }}
    )
```

## Testing Best Practices

### Fixtures and Test Organization
```python
# tests/conftest.py
import pytest
import asyncio
from unittest.mock import AsyncMock
from my_mcp_server.config import Settings
from my_mcp_server.server import MCPServer

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for async tests."""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
def test_settings():
    """Test configuration settings."""
    return Settings(
        server_name="test-server",
        debug=True,
        database_url="sqlite:///:memory:",
        api_key="test-key"
    )

@pytest.fixture
async def mock_external_service():
    """Mock external service dependencies."""
    mock = AsyncMock()
    mock.get_data.return_value = {"test": "data"}
    return mock

@pytest.fixture
async def test_server(test_settings, mock_external_service):
    """Create test server instance."""
    server = MCPServer(test_settings.server_name)
    # Inject mocked dependencies
    server._external_service = mock_external_service
    return server
```

### Test Categories
```python
# tests/test_tools.py
import pytest
from my_mcp_server.tools import DatabaseTool

class TestDatabaseTool:
    """Test database tool functionality."""
    
    @pytest.mark.asyncio
    async def test_valid_query(self, test_server):
        """Test successful query execution."""
        result = await test_server.execute_query("SELECT 1")
        assert len(result) > 0
    
    @pytest.mark.asyncio
    async def test_invalid_query_raises_error(self, test_server):
        """Test error handling for invalid queries."""
        with pytest.raises(ValidationError):
            await test_server.execute_query("INVALID SQL")
    
    @pytest.mark.parametrize("query,expected_error", [
        ("", "Query cannot be empty"),
        ("DROP TABLE users", "DROP statements not allowed"),
        ("SELECT * FROM users; DELETE FROM users", "Multiple statements not allowed"),
    ])
    @pytest.mark.asyncio
    async def test_query_validation(self, test_server, query, expected_error):
        """Test various query validation scenarios."""
        with pytest.raises(ValidationError, match=expected_error):
            await test_server.execute_query(query)
```

## Performance Optimization

### Caching
```python
from functools import lru_cache
from typing import Dict, Any
import asyncio
import time

# Simple in-memory cache
class TTLCache:
    """Time-to-live cache implementation."""
    
    def __init__(self, ttl: int = 300):
        self.ttl = ttl
        self._cache: Dict[str, tuple] = {}
    
    def get(self, key: str) -> Any:
        if key in self._cache:
            value, timestamp = self._cache[key]
            if time.time() - timestamp < self.ttl:
                return value
            else:
                del self._cache[key]
        return None
    
    def set(self, key: str, value: Any) -> None:
        self._cache[key] = (value, time.time())
    
    def clear(self) -> None:
        self._cache.clear()

# Usage with async functions
cache = TTLCache(ttl=300)  # 5 minutes

async def cached_api_call(endpoint: str) -> Dict[str, Any]:
    """API call with caching."""
    cached_result = cache.get(endpoint)
    if cached_result is not None:
        return cached_result
    
    result = await make_api_call(endpoint)
    cache.set(endpoint, result)
    return result
```

### Connection Pooling
```python
import aiohttp
from contextlib import asynccontextmanager

class ConnectionManager:
    """Manage HTTP connection pools."""
    
    def __init__(self):
        self._session: Optional[aiohttp.ClientSession] = None
    
    async def get_session(self) -> aiohttp.ClientSession:
        if self._session is None or self._session.closed:
            connector = aiohttp.TCPConnector(
                limit=100,  # Total connection pool size
                limit_per_host=30,  # Per-host connection limit
                keepalive_timeout=30,
                enable_cleanup_closed=True
            )
            
            timeout = aiohttp.ClientTimeout(
                total=30,
                connect=5,
                sock_read=10
            )
            
            self._session = aiohttp.ClientSession(
                connector=connector,
                timeout=timeout
            )
        
        return self._session
    
    async def close(self):
        if self._session and not self._session.closed:
            await self._session.close()

# Global connection manager
connection_manager = ConnectionManager()
```

## Security Best Practices

### Input Sanitization
```python
import re
from pathlib import Path
from typing import Any

def sanitize_file_path(file_path: str, allowed_dirs: List[str]) -> Path:
    """Sanitize file path to prevent directory traversal."""
    # Remove any path traversal attempts
    clean_path = re.sub(r'\.\./+', '', file_path)
    
    # Convert to absolute path
    abs_path = Path(clean_path).resolve()
    
    # Check if path is within allowed directories
    for allowed_dir in allowed_dirs:
        allowed_path = Path(allowed_dir).resolve()
        try:
            abs_path.relative_to(allowed_path)
            return abs_path
        except ValueError:
            continue
    
    raise ValidationError(f"File path not allowed: {file_path}")

def sanitize_sql_identifier(identifier: str) -> str:
    """Sanitize SQL identifier to prevent injection."""
    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', identifier):
        raise ValidationError(f"Invalid SQL identifier: {identifier}")
    return identifier
```

### Environment Variable Handling
```python
# Never log sensitive data
def safe_log_config(config: Dict[str, Any]) -> Dict[str, Any]:
    """Log configuration with sensitive data masked."""
    sensitive_keys = {'password', 'token', 'key', 'secret'}
    safe_config = {}
    
    for key, value in config.items():
        if any(sensitive in key.lower() for sensitive in sensitive_keys):
            safe_config[key] = "***MASKED***"
        else:
            safe_config[key] = value
    
    return safe_config
```

Following these Python-specific best practices will help you create robust, maintainable, and secure MCP servers that integrate well with the Python ecosystem.