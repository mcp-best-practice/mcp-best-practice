# Development Patterns

## Common MCP Development Patterns and Practices

This guide covers reusable patterns, architectural approaches, and design strategies for building robust MCP servers and clients.

## Architectural Patterns

### Layered Architecture
```
┌─────────────────────────────────┐
│          Presentation Layer     │  MCP Protocol Handlers
├─────────────────────────────────┤
│          Business Logic Layer   │  Tool Implementations
├─────────────────────────────────┤
│          Service Layer          │  External API Integrations  
├─────────────────────────────────┤
│          Data Access Layer      │  Database, Cache, Storage
└─────────────────────────────────┘
```

#### Implementation Example
```python
# Presentation Layer - MCP Protocol Handlers
class MCPHandlers:
    def __init__(self, tool_service: ToolService):
        self.tool_service = tool_service
    
    async def handle_list_tools(self) -> List[Tool]:
        return await self.tool_service.list_available_tools()
    
    async def handle_call_tool(self, name: str, args: dict) -> List[TextContent]:
        return await self.tool_service.execute_tool(name, args)

# Business Logic Layer - Tool Orchestration
class ToolService:
    def __init__(self, tool_registry: ToolRegistry, validator: InputValidator):
        self.registry = tool_registry
        self.validator = validator
    
    async def execute_tool(self, name: str, args: dict) -> List[TextContent]:
        # Validate inputs
        await self.validator.validate(name, args)
        
        # Get tool implementation
        tool = await self.registry.get_tool(name)
        
        # Execute with error handling
        return await self.safe_execute(tool, args)

# Service Layer - External Integrations
class GitHubService:
    def __init__(self, client: HTTPClient, auth: AuthProvider):
        self.client = client
        self.auth = auth
    
    async def create_issue(self, repo: str, title: str, body: str) -> dict:
        headers = await self.auth.get_headers()
        return await self.client.post(f"/repos/{repo}/issues", {
            "title": title,
            "body": body
        }, headers=headers)

# Data Access Layer - Storage Abstraction
class IssueRepository:
    def __init__(self, db: Database):
        self.db = db
    
    async def store_issue_ref(self, tool_call_id: str, issue_url: str):
        await self.db.execute(
            "INSERT INTO issue_refs (tool_call_id, issue_url) VALUES (?, ?)",
            tool_call_id, issue_url
        )
```

### Plugin Architecture
```python
# Plugin interface
class MCPPlugin(ABC):
    @abstractmethod
    def get_name(self) -> str:
        pass
    
    @abstractmethod
    def get_tools(self) -> List[Tool]:
        pass
    
    @abstractmethod
    async def initialize(self, config: dict) -> None:
        pass
    
    @abstractmethod
    async def cleanup(self) -> None:
        pass

# Plugin manager
class PluginManager:
    def __init__(self):
        self.plugins: Dict[str, MCPPlugin] = {}
        self.tools: Dict[str, Tool] = {}
    
    async def load_plugin(self, plugin_class: Type[MCPPlugin], config: dict):
        plugin = plugin_class()
        await plugin.initialize(config)
        
        self.plugins[plugin.get_name()] = plugin
        
        # Register plugin tools
        for tool in plugin.get_tools():
            self.tools[tool.name] = tool
    
    async def unload_plugin(self, name: str):
        if name in self.plugins:
            plugin = self.plugins[name]
            
            # Unregister tools
            for tool in plugin.get_tools():
                self.tools.pop(tool.name, None)
            
            await plugin.cleanup()
            del self.plugins[name]
    
    def get_available_tools(self) -> List[Tool]:
        return list(self.tools.values())

# Example plugin implementation
class GitHubPlugin(MCPPlugin):
    def get_name(self) -> str:
        return "github"
    
    def get_tools(self) -> List[Tool]:
        return [
            CreateIssueTool(self.github_client),
            SearchRepositoriesTool(self.github_client),
            GetPullRequestTool(self.github_client)
        ]
    
    async def initialize(self, config: dict) -> None:
        self.github_client = GitHubClient(
            token=config["token"],
            base_url=config.get("base_url", "https://api.github.com")
        )
        await self.github_client.verify_auth()
    
    async def cleanup(self) -> None:
        await self.github_client.close()
```

## Tool Design Patterns

### Command Pattern
```python
class Command(ABC):
    @abstractmethod
    async def execute(self) -> Any:
        pass
    
    @abstractmethod
    async def undo(self) -> Any:
        pass
    
    @abstractmethod
    def get_description(self) -> str:
        pass

class CreateFileCommand(Command):
    def __init__(self, file_path: str, content: str):
        self.file_path = file_path
        self.content = content
        self.existed_before = False
        self.previous_content = None
    
    async def execute(self) -> Any:
        self.existed_before = Path(self.file_path).exists()
        if self.existed_before:
            self.previous_content = Path(self.file_path).read_text()
        
        Path(self.file_path).write_text(self.content)
        return f"File created: {self.file_path}"
    
    async def undo(self) -> Any:
        if self.existed_before and self.previous_content is not None:
            Path(self.file_path).write_text(self.previous_content)
            return f"File restored: {self.file_path}"
        elif Path(self.file_path).exists():
            Path(self.file_path).unlink()
            return f"File deleted: {self.file_path}"
    
    def get_description(self) -> str:
        return f"Create file {self.file_path}"

class CommandExecutor:
    def __init__(self):
        self.history: List[Command] = []
    
    async def execute(self, command: Command) -> Any:
        result = await command.execute()
        self.history.append(command)
        return result
    
    async def undo_last(self) -> Any:
        if not self.history:
            raise ValueError("No commands to undo")
        
        command = self.history.pop()
        return await command.undo()
```

### Factory Pattern for Tool Creation
```python
class ToolFactory:
    _registry: Dict[str, Callable[..., Tool]] = {}
    
    @classmethod
    def register(cls, tool_type: str):
        def decorator(tool_class: Type[Tool]):
            cls._registry[tool_type] = tool_class
            return tool_class
        return decorator
    
    @classmethod
    def create_tool(cls, tool_type: str, **kwargs) -> Tool:
        if tool_type not in cls._registry:
            raise ValueError(f"Unknown tool type: {tool_type}")
        
        tool_class = cls._registry[tool_type]
        return tool_class(**kwargs)
    
    @classmethod
    def list_available_types(cls) -> List[str]:
        return list(cls._registry.keys())

# Usage
@ToolFactory.register("github_issue")
class GitHubIssueTool(Tool):
    def __init__(self, github_client: GitHubClient):
        self.github_client = github_client

@ToolFactory.register("file_read")
class FileReadTool(Tool):
    def __init__(self, allowed_paths: List[str]):
        self.allowed_paths = allowed_paths

# Create tools dynamically
github_tool = ToolFactory.create_tool("github_issue", github_client=client)
file_tool = ToolFactory.create_tool("file_read", allowed_paths=["/tmp", "/var/data"])
```

### Strategy Pattern for Different Execution Modes
```python
class ExecutionStrategy(ABC):
    @abstractmethod
    async def execute(self, tool: Tool, arguments: dict) -> List[Content]:
        pass

class SynchronousExecution(ExecutionStrategy):
    async def execute(self, tool: Tool, arguments: dict) -> List[Content]:
        return await tool.execute(arguments)

class AsynchronousExecution(ExecutionStrategy):
    def __init__(self, task_queue: TaskQueue):
        self.task_queue = task_queue
    
    async def execute(self, tool: Tool, arguments: dict) -> List[Content]:
        task_id = await self.task_queue.enqueue(tool, arguments)
        return [TextContent(
            type="text",
            text=f"Task queued with ID: {task_id}"
        )]

class RetryExecution(ExecutionStrategy):
    def __init__(self, base_strategy: ExecutionStrategy, max_retries: int = 3):
        self.base_strategy = base_strategy
        self.max_retries = max_retries
    
    async def execute(self, tool: Tool, arguments: dict) -> List[Content]:
        last_error = None
        
        for attempt in range(self.max_retries + 1):
            try:
                return await self.base_strategy.execute(tool, arguments)
            except Exception as e:
                last_error = e
                if attempt < self.max_retries:
                    await asyncio.sleep(2 ** attempt)  # Exponential backoff
                continue
        
        raise last_error

class ToolExecutor:
    def __init__(self, strategy: ExecutionStrategy):
        self.strategy = strategy
    
    async def execute_tool(self, tool: Tool, arguments: dict) -> List[Content]:
        return await self.strategy.execute(tool, arguments)
    
    def set_strategy(self, strategy: ExecutionStrategy):
        self.strategy = strategy
```

## Resource Management Patterns

### Resource Pool Pattern
```python
class ResourcePool(Generic[T]):
    def __init__(self, factory: Callable[[], T], max_size: int = 10):
        self.factory = factory
        self.max_size = max_size
        self._pool: List[T] = []
        self._in_use: Set[T] = set()
        self._lock = asyncio.Lock()
    
    async def acquire(self) -> T:
        async with self._lock:
            if self._pool:
                resource = self._pool.pop()
            elif len(self._in_use) < self.max_size:
                resource = self.factory()
            else:
                # Wait for resource to become available
                while not self._pool:
                    await asyncio.sleep(0.1)
                resource = self._pool.pop()
            
            self._in_use.add(resource)
            return resource
    
    async def release(self, resource: T):
        async with self._lock:
            if resource in self._in_use:
                self._in_use.remove(resource)
                self._pool.append(resource)
    
    @asynccontextmanager
    async def get_resource(self) -> AsyncGenerator[T, None]:
        resource = await self.acquire()
        try:
            yield resource
        finally:
            await self.release(resource)

# Usage with database connections
class DatabaseConnectionPool(ResourcePool[Database]):
    def __init__(self, connection_string: str, max_size: int = 10):
        super().__init__(
            factory=lambda: Database(connection_string),
            max_size=max_size
        )

# Tool using connection pool
class DatabaseQueryTool(Tool):
    def __init__(self, db_pool: DatabaseConnectionPool):
        self.db_pool = db_pool
    
    async def execute(self, arguments: dict) -> List[Content]:
        query = arguments["query"]
        
        async with self.db_pool.get_resource() as db:
            results = await db.execute(query)
            return [TextContent(
                type="text",
                text=json.dumps(results, indent=2)
            )]
```

### Circuit Breaker Pattern
```python
from enum import Enum
from datetime import datetime, timedelta

class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"

class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 60,
        expected_exception: Type[Exception] = Exception
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.expected_exception = expected_exception
        
        self.failure_count = 0
        self.last_failure_time: Optional[datetime] = None
        self.state = CircuitState.CLOSED
    
    async def call(self, func: Callable, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except self.expected_exception as e:
            self._on_failure()
            raise e
    
    def _should_attempt_reset(self) -> bool:
        return (
            self.last_failure_time is not None and
            datetime.now() - self.last_failure_time >= timedelta(seconds=self.recovery_timeout)
        )
    
    def _on_success(self):
        self.failure_count = 0
        self.state = CircuitState.CLOSED
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = datetime.now()
        
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN

# Tool with circuit breaker
class ResilientAPITool(Tool):
    def __init__(self, api_client: APIClient):
        self.api_client = api_client
        self.circuit_breaker = CircuitBreaker(
            failure_threshold=3,
            recovery_timeout=30,
            expected_exception=APIError
        )
    
    async def execute(self, arguments: dict) -> List[Content]:
        try:
            result = await self.circuit_breaker.call(
                self.api_client.make_request,
                arguments["endpoint"],
                arguments.get("data")
            )
            return [TextContent(type="text", text=str(result))]
        except Exception as e:
            return [TextContent(
                type="text",
                text=f"Service temporarily unavailable: {str(e)}"
            )]
```

## Event-Driven Patterns

### Observer Pattern for Tool Execution Events
```python
class ToolExecutionEvent:
    def __init__(self, tool_name: str, arguments: dict, result: Optional[Any] = None, error: Optional[Exception] = None):
        self.tool_name = tool_name
        self.arguments = arguments
        self.result = result
        self.error = error
        self.timestamp = datetime.utcnow()

class ToolEventObserver(ABC):
    @abstractmethod
    async def on_tool_started(self, event: ToolExecutionEvent):
        pass
    
    @abstractmethod
    async def on_tool_completed(self, event: ToolExecutionEvent):
        pass
    
    @abstractmethod
    async def on_tool_failed(self, event: ToolExecutionEvent):
        pass

class MetricsObserver(ToolEventObserver):
    def __init__(self):
        self.execution_counts: Dict[str, int] = defaultdict(int)
        self.error_counts: Dict[str, int] = defaultdict(int)
    
    async def on_tool_started(self, event: ToolExecutionEvent):
        self.execution_counts[event.tool_name] += 1
    
    async def on_tool_completed(self, event: ToolExecutionEvent):
        # Log successful completion metrics
        pass
    
    async def on_tool_failed(self, event: ToolExecutionEvent):
        self.error_counts[event.tool_name] += 1

class LoggingObserver(ToolEventObserver):
    def __init__(self, logger: logging.Logger):
        self.logger = logger
    
    async def on_tool_started(self, event: ToolExecutionEvent):
        self.logger.info(f"Tool {event.tool_name} started with args: {event.arguments}")
    
    async def on_tool_completed(self, event: ToolExecutionEvent):
        self.logger.info(f"Tool {event.tool_name} completed successfully")
    
    async def on_tool_failed(self, event: ToolExecutionEvent):
        self.logger.error(f"Tool {event.tool_name} failed: {event.error}")

class ObservableToolExecutor:
    def __init__(self):
        self.observers: List[ToolEventObserver] = []
    
    def add_observer(self, observer: ToolEventObserver):
        self.observers.append(observer)
    
    def remove_observer(self, observer: ToolEventObserver):
        self.observers.remove(observer)
    
    async def execute_tool(self, tool: Tool, arguments: dict) -> List[Content]:
        event = ToolExecutionEvent(tool.name, arguments)
        
        # Notify start
        await self._notify_observers('on_tool_started', event)
        
        try:
            result = await tool.execute(arguments)
            event.result = result
            
            # Notify completion
            await self._notify_observers('on_tool_completed', event)
            
            return result
        except Exception as e:
            event.error = e
            
            # Notify failure
            await self._notify_observers('on_tool_failed', event)
            
            raise e
    
    async def _notify_observers(self, method_name: str, event: ToolExecutionEvent):
        for observer in self.observers:
            try:
                method = getattr(observer, method_name)
                await method(event)
            except Exception as e:
                # Don't let observer errors break tool execution
                logging.error(f"Observer error: {e}")
```

## Configuration Patterns

### Configuration Builder Pattern
```python
class ServerConfigBuilder:
    def __init__(self):
        self.config = {
            "server": {},
            "tools": {},
            "logging": {},
            "security": {}
        }
    
    def with_server_name(self, name: str):
        self.config["server"]["name"] = name
        return self
    
    def with_transport(self, transport: str, **options):
        self.config["server"]["transport"] = {
            "type": transport,
            **options
        }
        return self
    
    def with_tool(self, tool_name: str, **tool_config):
        self.config["tools"][tool_name] = tool_config
        return self
    
    def with_logging(self, level: str = "INFO", format: str = "json"):
        self.config["logging"] = {
            "level": level,
            "format": format
        }
        return self
    
    def with_security(self, **security_config):
        self.config["security"].update(security_config)
        return self
    
    def build(self) -> dict:
        # Validate required fields
        if "name" not in self.config["server"]:
            raise ValueError("Server name is required")
        
        return self.config.copy()

# Usage
config = (ServerConfigBuilder()
    .with_server_name("my-mcp-server")
    .with_transport("http", port=8000, host="0.0.0.0")
    .with_tool("github", token="ghp_xxx", base_url="https://api.github.com")
    .with_logging("DEBUG", "text")
    .with_security(require_auth=True, max_requests_per_minute=60)
    .build())
```

These patterns provide proven solutions for common challenges in MCP server development, promoting code reusability, maintainability, and scalability.