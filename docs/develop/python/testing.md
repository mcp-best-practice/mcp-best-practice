# Python Testing

## Testing Strategy for MCP Servers

Comprehensive testing ensures your MCP server is reliable and maintainable.

## Testing Stack

### Core Testing Tools
```bash
# Install testing dependencies
uv add --dev pytest pytest-asyncio pytest-cov pytest-mock
```

### Testing Configuration
```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = ["test_*.py", "*_test.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]
addopts = "-v --tb=short --cov=src --cov-report=term-missing"
asyncio_mode = "auto"
```

## Unit Testing

### Testing Tools
```python
# tests/test_tools.py
import pytest
from unittest.mock import AsyncMock, patch
from mcp.types import TextContent

from my_mcp_server.tools.database import _execute_query

@pytest.mark.asyncio
async def test_execute_query():
    """Test database query execution."""
    args = {"query": "SELECT * FROM users", "limit": 10}
    
    with patch("my_mcp_server.tools.database.db_client") as mock_db:
        mock_db.execute.return_value = [{"id": 1, "name": "Alice"}]
        
        result = await _execute_query(args)
        
        assert len(result) == 1
        assert isinstance(result[0], TextContent)
        assert "Alice" in result[0].text
```

### Testing Server Handlers
```python
# tests/test_server.py
import pytest
from mcp.server import Server
from my_mcp_server.server import MCPServer

@pytest.fixture
def server():
    """Create test server instance."""
    return MCPServer("test-server")

@pytest.mark.asyncio
async def test_server_initialization(server):
    """Test server initializes correctly."""
    assert server.server.name == "test-server"
    # Test that handlers are registered
    tools = await server.server._list_tools_handler()
    assert len(tools) > 0
```

## Integration Testing

### Testing with Real MCP Protocol
```python
# tests/test_integration.py
import asyncio
import json
from io import StringIO
from mcp.server.stdio import stdio_server

@pytest.mark.asyncio
async def test_mcp_protocol_integration():
    """Test full MCP protocol integration."""
    from my_mcp_server.server import MCPServer
    
    server = MCPServer("test-server")
    
    # Simulate stdin/stdout
    stdin_data = json.dumps({
        "jsonrpc": "2.0",
        "method": "tools/list",
        "id": 1
    }) + "\n"
    
    stdin = StringIO(stdin_data)
    stdout = StringIO()
    
    # This would require custom stdio handling for testing
    # In practice, use MCP test utilities when available
```

## Mock External Dependencies

### Database Mocking
```python
# tests/conftest.py
import pytest
from unittest.mock import AsyncMock

@pytest.fixture
def mock_database():
    """Mock database client."""
    mock_client = AsyncMock()
    mock_client.execute.return_value = [{"id": 1, "name": "test"}]
    mock_client.connect.return_value = True
    return mock_client

@pytest.fixture(autouse=True)
def patch_database(mock_database, monkeypatch):
    """Automatically patch database in all tests."""
    monkeypatch.setattr(
        "my_mcp_server.tools.database.get_db_client",
        lambda: mock_database
    )
```

### API Client Mocking
```python
# tests/test_api_tools.py
import pytest
import aiohttp
from aioresponses import aioresponses

@pytest.mark.asyncio
async def test_api_call():
    """Test external API calls."""
    with aioresponses() as m:
        m.get(
            "https://api.example.com/users/1",
            payload={"id": 1, "name": "Alice"}
        )
        
        result = await fetch_user_data(1)
        assert result["name"] == "Alice"
```

## Testing Utilities

### Test Fixtures
```python
# tests/fixtures.py
import pytest
from pathlib import Path

@pytest.fixture
def sample_data():
    """Load sample test data."""
    fixture_path = Path(__file__).parent / "fixtures" / "sample_data.json"
    with open(fixture_path) as f:
        return json.load(f)

@pytest.fixture
def temp_file(tmp_path):
    """Create temporary test file."""
    test_file = tmp_path / "test.txt"
    test_file.write_text("test content")
    return test_file
```

### Custom Assertions
```python
# tests/assertions.py
def assert_valid_mcp_response(response):
    """Assert response is valid MCP format."""
    assert isinstance(response, list)
    for item in response:
        assert hasattr(item, 'type')
        if item.type == 'text':
            assert hasattr(item, 'text')
            assert isinstance(item.text, str)
```

## Error Testing

### Testing Error Conditions
```python
@pytest.mark.asyncio
async def test_invalid_tool_name():
    """Test error handling for invalid tool names."""
    server = MCPServer("test-server")
    
    with pytest.raises(ValueError, match="Unknown tool"):
        await server.server._call_tool_handler(
            "nonexistent_tool",
            {"arg": "value"}
        )

@pytest.mark.asyncio
async def test_missing_required_argument():
    """Test error handling for missing arguments."""
    with pytest.raises(ValueError, match="required"):
        await call_tool_with_validation("query_database", {})
```

### Testing Input Validation
```python
@pytest.mark.parametrize("invalid_input,expected_error", [
    ("", "Query cannot be empty"),
    (None, "Query is required"),
    ("SELECT * FROM users; DROP TABLE users;", "Invalid query"),
])
@pytest.mark.asyncio
async def test_query_validation(invalid_input, expected_error):
    """Test query validation."""
    with pytest.raises(ValueError, match=expected_error):
        await validate_query(invalid_input)
```

## Performance Testing

### Timing Tests
```python
import time

@pytest.mark.asyncio
async def test_response_time():
    """Test response time requirements."""
    start_time = time.time()
    
    result = await execute_expensive_operation()
    
    elapsed = time.time() - start_time
    assert elapsed < 1.0  # Should complete within 1 second
```

### Load Testing
```python
@pytest.mark.asyncio
async def test_concurrent_requests():
    """Test handling multiple concurrent requests."""
    server = MCPServer("test-server")
    
    # Create 10 concurrent requests
    tasks = [
        server.call_tool("simple_tool", {"id": i})
        for i in range(10)
    ]
    
    results = await asyncio.gather(*tasks)
    assert len(results) == 10
```

## Test Coverage

### Running Coverage
```bash
# Run tests with coverage
pytest --cov=src --cov-report=html --cov-report=term-missing

# View HTML coverage report
open htmlcov/index.html
```

### Coverage Configuration
```toml
# pyproject.toml
[tool.coverage.run]
source = ["src"]
omit = [
    "tests/*",
    "*/test_*.py",
    "src/*/__main__.py",
]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "if __name__ == .__main__.:",
]
```

## Continuous Integration

### GitHub Actions Testing
```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.11", "3.12"]
    
    steps:
    - uses: actions/checkout@v4
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: ${{ matrix.python-version }}
    
    - name: Install dependencies
      run: |
        pip install -e ".[dev]"
    
    - name: Run tests
      run: |
        pytest --cov=src --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
```

## Best Practices

### Test Organization
1. **Mirror Structure**: Test files mirror source structure
2. **Clear Names**: Descriptive test function names
3. **Single Responsibility**: One test per behavior
4. **Setup/Teardown**: Use fixtures for common setup

### Test Quality
1. **Fast Tests**: Unit tests should run quickly
2. **Isolated Tests**: Tests don't depend on each other
3. **Deterministic**: Tests produce same results every time
4. **Comprehensive**: Cover happy path, edge cases, and errors

### Debugging Tests
```bash
# Run specific test with verbose output
pytest tests/test_specific.py::test_function -v -s

# Run with debugging on first failure
pytest --pdb

# Run with coverage and open report
pytest --cov=src --cov-report=html && open htmlcov/index.html
```

Testing is crucial for maintaining reliable MCP servers. Focus on testing your business logic, error handling, and integration points.