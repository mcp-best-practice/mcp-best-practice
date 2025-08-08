# Testing Guide

## Comprehensive Testing for MCP Servers

Testing is crucial for ensuring MCP servers are reliable, secure, and performant. This guide covers all testing levels from unit to end-to-end.

## Testing Pyramid

```
        /\
       /E2E\
      /------\
     /  Integ \
    /----------\
   /    Unit    \
  /--------------\
```

1. **Unit Tests** (70%) - Fast, isolated component tests
2. **Integration Tests** (20%) - Component interaction tests
3. **End-to-End Tests** (10%) - Full system workflow tests

## Testing Strategy

### What to Test

#### Critical Paths
- Tool execution and responses
- Resource access and permissions
- Error handling and recovery
- Input validation and sanitization
- Authentication and authorization

#### Edge Cases
- Empty inputs
- Maximum size inputs
- Invalid data types
- Concurrent requests
- Network failures

## Quick Start Testing

### Python Example
```python
import pytest
from my_server import mcp

@pytest.fixture
def client():
    return mcp.test_client()

def test_tool_execution(client):
    response = client.call_tool('my_tool', {'input': 'test'})
    assert response['status'] == 'success'
    assert 'result' in response
```

### JavaScript Example
```javascript
const { TestClient } = require('@mcp/testing');
const server = require('./server');

describe('MCP Server', () => {
  let client;
  
  beforeAll(() => {
    client = new TestClient(server);
  });
  
  test('tool execution', async () => {
    const result = await client.callTool('my_tool', { input: 'test' });
    expect(result.status).toBe('success');
  });
});
```

### Go Example
```go
func TestToolExecution(t *testing.T) {
    server := NewTestServer()
    defer server.Close()
    
    result, err := server.CallTool("my_tool", map[string]interface{}{
        "input": "test",
    })
    
    assert.NoError(t, err)
    assert.Equal(t, "expected", result)
}
```

## Test Coverage Requirements

### Minimum Coverage
- **Overall**: 80% code coverage
- **Critical paths**: 100% coverage
- **Error handling**: 90% coverage
- **Edge cases**: 85% coverage

### Coverage Tools
| Language | Tool | Command |
|----------|------|---------|
| Python | pytest-cov | `pytest --cov=my_server --cov-report=html` |
| JavaScript | Jest | `jest --coverage` |
| Go | Built-in | `go test -cover ./...` |

## Test Data Management

### Fixtures
```python
# Python fixtures
@pytest.fixture
def sample_data():
    return {
        'valid_input': {'name': 'test', 'value': 123},
        'invalid_input': {'name': '', 'value': 'not_a_number'},
        'edge_case': {'name': 'x' * 1000, 'value': float('inf')}
    }
```

### Mocking External Services
```javascript
// JavaScript mocking
jest.mock('./external-api', () => ({
  fetchData: jest.fn(() => Promise.resolve({ data: 'mocked' }))
}));
```

## Continuous Integration

### GitHub Actions Example
```yaml
name: Test MCP Server

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install -e ".[test]"
    
    - name: Run tests
      run: |
        pytest --cov=my_server --cov-report=xml
    
    - name: Upload coverage
      uses: codecov/codecov-action@v3
```

## Testing Checklist

### Pre-Release Testing
- [ ] All unit tests pass
- [ ] Integration tests complete
- [ ] Security scan performed
- [ ] Performance benchmarks met
- [ ] Documentation updated
- [ ] Manual smoke tests done

### Test Categories
1. **Functional Testing**
   - Tool functionality
   - Resource access
   - Protocol compliance

2. **Non-Functional Testing**
   - Performance
   - Security
   - Usability
   - Reliability

3. **Regression Testing**
   - Previous bugs fixed
   - Features still working
   - Backward compatibility

## Common Testing Patterns

### Table-Driven Tests
```go
// Go example
tests := []struct {
    name    string
    input   string
    want    string
    wantErr bool
}{
    {"valid", "test", "TEST", false},
    {"empty", "", "", true},
    {"special", "!@#", "!@#", false},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got, err := process(tt.input)
        if (err != nil) != tt.wantErr {
            t.Errorf("process() error = %v, wantErr %v", err, tt.wantErr)
        }
        if got != tt.want {
            t.Errorf("process() = %v, want %v", got, tt.want)
        }
    })
}
```

### Property-Based Testing
```python
# Python hypothesis example
from hypothesis import given, strategies as st

@given(st.text())
def test_tool_handles_any_string(s):
    result = my_tool(s)
    assert isinstance(result, str)
    assert len(result) <= 1000  # Max length constraint
```

## Testing Tools

### Recommended Tools by Language

#### Python
- **pytest** - Testing framework
- **pytest-asyncio** - Async test support
- **pytest-cov** - Coverage reporting
- **hypothesis** - Property-based testing
- **responses** - HTTP mocking

#### JavaScript/TypeScript
- **Jest** - Testing framework
- **Supertest** - HTTP testing
- **Sinon** - Mocking library
- **Playwright** - E2E testing

#### Go
- **testify** - Assertions and mocks
- **gomock** - Mocking framework
- **httptest** - HTTP testing
- **race detector** - Concurrency testing

## Next Steps

- ✅ [Unit Testing](unit-testing.md)
- 🔄 [Integration Testing](integration-testing.md)
- 🚀 [End-to-End Testing](e2e-testing.md)
- 📊 [Performance Testing](performance-testing.md)
- 🔒 [Security Testing](security-testing.md)