# MCP Server Best Practices Overview

## Picking the Right SDK

### SDK Selection Criteria
Choose an SDK based on these attributes:

#### Read-Only Mode Support
- **Hosting providers**: Prefer SDKs with read-only mode restrictions
- **Security-first**: Choose SDKs that default to read-only operations
- **Granular control**: Select SDKs with fine-grained permission controls

#### Dynamic Toolset Selection
- **Runtime configuration**: SDKs that support tool registration at runtime
- **Conditional tools**: Ability to enable/disable tools based on permissions
- **Context-aware tools**: Tools that adapt based on user authentication

#### Authentication/Authorization Support
- **OAuth 2.0**: Built-in OAuth flow handling
- **API key management**: Secure credential storage and rotation
- **Scope management**: Granular permission scopes per tool
- **Token exchange**: Support for token refresh and exchange patterns

### SDK Comparison Matrix

| Feature | Python SDK | JavaScript SDK | Go SDK |
|---------|------------|----------------|--------|
| Read-only mode | ✅ | ✅ | ✅ |
| Dynamic tools | ✅ | ✅ | ⚠️ Limited |
| OAuth 2.0 | ✅ | ✅ | ✅ |
| Built-in auth | ✅ | ⚠️ Partial | ⚠️ Partial |
| Scope management | ✅ | ✅ | ✅ |

## Tool and Prompt Description Guidance

### Writing Effective Tool Descriptions
Follow these principles for tool descriptions:

#### Be Specific and Actionable
```json
// ❌ Bad: Vague description
{
  "name": "manage_files",
  "description": "Handle files"
}

// ✅ Good: Specific and clear
{
  "name": "read_text_file",
  "description": "Read the contents of a text file from the local filesystem. Returns the file content as plain text. Maximum file size: 10MB."
}
```

#### Include Context and Constraints
```json
{
  "name": "create_github_issue",
  "description": "Create a new issue in a GitHub repository. Requires 'issues:write' permission. The issue will be created with the authenticated user as the author.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "repository": {
        "type": "string",
        "description": "Repository in format 'owner/repo' (e.g., 'microsoft/vscode')",
        "pattern": "^[a-zA-Z0-9_.-]+/[a-zA-Z0-9_.-]+$"
      },
      "title": {
        "type": "string",
        "description": "Issue title (1-256 characters)",
        "minLength": 1,
        "maxLength": 256
      },
      "body": {
        "type": "string",
        "description": "Issue description in GitHub Markdown format",
        "maxLength": 65536
      }
    }
  }
}
```

#### Reference Existing Documentation
For tool descriptions, refer to:
- [Anthropic's MCP documentation](https://modelcontextprotocol.io/docs)
- [Tool design patterns](https://github.com/anthropics/mcp-examples)
- Industry-specific API documentation

### Managing State in MCP Servers

#### Stateless by Default
```python
# ✅ Good: Stateless operation
@server.call_tool()
async def get_user_profile(name: str, arguments: dict):
    user_id = arguments["user_id"]
    # Fetch from external system
    profile = await api_client.get_user(user_id)
    return [TextContent(type="text", text=json.dumps(profile))]
```

#### External State Management
```python
# ✅ Good: External state storage
class StatefulMCPServer:
    def __init__(self, redis_client):
        self.cache = redis_client
    
    async def process_with_cache(self, key: str, data: dict):
        # Check cache first
        cached = await self.cache.get(key)
        if cached:
            return json.loads(cached)
        
        # Process and cache result
        result = await process_data(data)
        await self.cache.setex(key, 300, json.dumps(result))  # 5 min TTL
        return result
```

## Testing Strategy

### Evaluation with Models
Test your MCP server with both hosted and local models:

```python
# Automated testing with different models
async def test_with_models():
    models = [
        ("claude-3-sonnet", "hosted"),
        ("llama2", "local"),
        ("gpt-4", "hosted")
    ]
    
    for model_name, deployment in models:
        client = get_model_client(model_name, deployment)
        
        # Test tool discovery
        tools = await client.list_tools()
        assert len(tools) > 0
        
        # Test tool execution
        result = await client.call_tool("echo", {"text": "test"})
        assert "test" in str(result)
```

### Linting and Static Analysis
Implement comprehensive code quality checks:

```yaml
# .github/workflows/quality.yml
name: Code Quality
on: [push, pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Static Analysis Security Testing (SAST)
      run: |
        # Python
        bandit -r src/
        semgrep --config=auto src/
        
        # JavaScript
        npm audit
        eslint --ext .js,.ts src/
        
        # Go
        gosec ./...
        staticcheck ./...
```

## Curated Catalog

### What is a Curated Catalog?
A curated catalog is a vetted collection of MCP servers approved for organizational use.

#### Internal Catalogs
- **Company-specific**: Internal MCP servers for organizational tools
- **Security reviewed**: All servers undergo security assessment
- **Compliance checked**: Servers meet regulatory requirements
- **Usage tracked**: Monitor adoption and performance

#### External Catalogs
- **Public repositories**: Community-maintained MCP server collections
- **Vendor catalogs**: Official servers from service providers
- **Industry-specific**: Specialized servers for specific domains

### Example Catalog Structure
```yaml
# catalog.yaml
servers:
  github-mcp:
    name: "GitHub Integration Server"
    repository: "https://github.com/company/github-mcp"
    version: "1.2.0"
    security_review: "2024-01-15"
    compliance: ["SOC2", "GDPR"]
    tags: ["git", "development", "issues"]
    
  database-mcp:
    name: "Database Query Server"
    repository: "https://github.com/company/database-mcp"
    version: "2.1.0"
    security_review: "2024-02-01"
    compliance: ["SOC2", "HIPAA"]
    tags: ["database", "sql", "reporting"]
```

## Tool Filtering and Guardrails

### Production Tool Filtering
Never provide agents with tools that can cause irreversible changes in production:

```python
class ProductionToolFilter:
    DANGEROUS_PATTERNS = [
        "delete_",
        "drop_",
        "truncate_",
        "destroy_",
        "_production"
    ]
    
    def filter_tools(self, tools: List[Tool], environment: str) -> List[Tool]:
        if environment == "production":
            return [
                tool for tool in tools 
                if not any(pattern in tool.name.lower() 
                          for pattern in self.DANGEROUS_PATTERNS)
            ]
        return tools
```

### Sequencing Tools with Guardrails
Implement approval workflows for sensitive operations:

```python
async def execute_with_guardrails(tool_name: str, arguments: dict):
    # Check if tool requires approval
    if tool_name in REQUIRES_APPROVAL:
        approval = await request_approval(tool_name, arguments)
        if not approval.granted:
            raise PermissionError(f"Tool execution not approved: {approval.reason}")
    
    # Execute with monitoring
    with execution_monitor(tool_name):
        result = await execute_tool(tool_name, arguments)
    
    return result
```

## Model Selection and Experience

### Model Performance by Task Type
Different models excel at different MCP operations:

| Task Type | Best Models | Notes |
|-----------|-------------|-------|
| Code generation | Claude-3.5 Sonnet, GPT-4 | Strong reasoning for complex tools |
| Data analysis | Claude-3 Sonnet, GPT-4 | Good at structured data manipulation |
| Text processing | Claude-3 Haiku, GPT-3.5 | Fast for simple text operations |
| API integration | Claude-3.5 Sonnet | Excellent at following API specifications |

### Context Size Considerations
- **Large context models**: Better for complex multi-step operations
- **Small context models**: Suitable for simple, focused tasks
- **Context management**: Implement context window optimization

```python
# Context optimization
def optimize_context(messages: List[Message], max_tokens: int) -> List[Message]:
    if total_tokens(messages) <= max_tokens:
        return messages
    
    # Keep system message and recent context
    system_msg = messages[0]
    recent_msgs = messages[-10:]  # Keep last 10 messages
    
    return [system_msg] + recent_msgs
```

## Product Support Lifecycle

### Managing Spec Evolution
The MCP specification evolves rapidly while product lifecycles are long:

#### Version Strategy
```yaml
mcp_compatibility:
  supported_versions: ["2024-11-05", "2025-06-18"]
  deprecated_versions: ["2024-06-01"]
  migration_timeline: "6 months"
  
product_lifecycle:
  support_duration: "3 years"
  update_frequency: "quarterly"
  lts_versions: ["1.0", "2.0"]
```

#### Adaptation Strategies
- **Graceful degradation**: Handle unknown protocol features
- **Feature detection**: Test for capability support at runtime
- **Backward compatibility**: Support multiple protocol versions
- **Regular updates**: Plan quarterly compatibility updates

### Developer Preview Recommendations
Given the evolving nature of MCP:

- **Community support only**: Don't provide enterprise SLA during preview
- **Regular updates required**: Plan for frequent protocol updates
- **Feedback collection**: Actively gather user experience feedback
- **Migration tooling**: Provide tools for spec updates

```python
# Feature detection pattern
async def check_server_capabilities():
    try:
        # Try new protocol feature
        result = await server.call("new_method", {})
        return {"supports_new_method": True}
    except MethodNotFoundError:
        # Fallback to older approach
        return {"supports_new_method": False}
```

## The MCP Host Application

### Key Security Features for Client Applications
Host applications must implement these security features:

#### Tool Description Transparency
Always show complete tool descriptions to users to prevent rug pulls and shadowing:

```python
# Show full tool schema before execution
def display_tool_info(tool: Tool):
    print(f"Tool: {tool.name}")
    print(f"Description: {tool.description}")
    print(f"Required permissions: {get_required_permissions(tool)}")
    print(f"Potential side effects: {analyze_side_effects(tool)}")
    
    # Show input schema clearly
    for param, schema in tool.inputSchema.get("properties", {}).items():
        required = param in tool.inputSchema.get("required", [])
        print(f"  {param} ({'required' if required else 'optional'}): {schema.get('description', '')}")
```

#### Request Approval for Tool Use
Implement user approval workflows for sensitive operations:

```python
class ToolApprovalSystem:
    def __init__(self):
        self.high_risk_patterns = ["delete", "drop", "remove", "destroy"]
        self.auto_approve_patterns = ["read", "get", "list", "show"]
    
    async def request_approval(self, tool_name: str, arguments: dict) -> bool:
        risk_level = self.assess_risk(tool_name, arguments)
        
        if risk_level == "low":
            return True  # Auto-approve safe operations
        
        # Show detailed information and request approval
        print(f"\n🔧 Tool Execution Request")
        print(f"Tool: {tool_name}")
        print(f"Arguments: {json.dumps(arguments, indent=2)}")
        print(f"Risk Level: {risk_level}")
        
        return await self.get_user_confirmation()
    
    def assess_risk(self, tool_name: str, arguments: dict) -> str:
        if any(pattern in tool_name.lower() for pattern in self.high_risk_patterns):
            return "high"
        
        # Check for production environment
        if any("production" in str(v).lower() for v in arguments.values()):
            return "high"
        
        return "low"
```

#### Tool Shadowing Prevention
Detect and prevent malicious tools that mimic safe operations:

```python
def detect_tool_shadowing(new_tool: Tool, existing_tools: List[Tool]) -> List[str]:
    warnings = []
    
    for existing in existing_tools:
        # Check for name similarity
        similarity = calculate_similarity(new_tool.name, existing.name)
        if similarity > 0.8 and new_tool.name != existing.name:
            warnings.append(f"Tool name '{new_tool.name}' is similar to existing '{existing.name}'")
        
        # Check for description overlap with different behavior
        if (similar_descriptions(new_tool.description, existing.description) and
            different_schemas(new_tool.inputSchema, existing.inputSchema)):
            warnings.append(f"Tool '{new_tool.name}' has similar description but different schema than '{existing.name}'")
    
    return warnings
```

## Input and Output Sanitization

Ensure your inputs and outputs are sanitized. In Python, we recommend using Pydantic V2.

## 📦 Self-Containment
Each MCP server must be a **standalone repository** that includes all necessary code and documentation.
Example: `git clone; make serve`

## 🛠 Makefile Requirements

All MCP repositories must include a `Makefile` with the following standard targets. These targets ensure consistency, enable automation, and support local development and containerization.

### ✅ Required Make Targets

Make targets are grouped by functionality. Use `make help` to see them all in your terminal.

#### 🌱 VIRTUAL ENVIRONMENT & INSTALLATION

| Target           | Description |
|------------------|-------------|
| `make venv`      | Create a new Python virtual environment in `~/.venv/<project>`. |
| `make activate`  | Output the command to activate the virtual environment. |
| `make install`   | Install all dependencies using `uv` from `pyproject.toml`. |
| `make clean`     | Remove virtualenv, Python artifacts, build files, and containers. |

#### ▶️ RUN SERVER & TESTING

| Target               | Description |
|----------------------|-------------|
| `make serve`         | Run the MCP server locally (e.g., `mcp-time-server`). |
| `make test`          | Run all unit and integration tests with `pytest`. |
| `make test-curl`     | Run public API integration tests using a `curl` script. |

#### 📚 DOCUMENTATION & SBOM

| Target         | Description |
|----------------|-------------|
| `make docs`    | Generate project documentation and SBOM using `handsdown`. |
| `make sbom`    | Create a software bill of materials (SBOM) and scan dependencies. |

#### 🔍 LINTING & STATIC ANALYSIS

| Target               | Description |
|----------------------|-------------|
| `make lint`          | Run all linters (e.g., `ruff check`, `ruff format`). |

#### 🐳 CONTAINER BUILD & RUN

| Target               | Description |
|----------------------|-------------|
| `make podman`        | Build a production-ready container image with Podman. |
| `make podman-run`    | Run the container locally and expose it on port 8080. |
| `make podman-stop`   | Stop and remove the running container. |
| `make podman-test`   | Test the container with a `curl` script. |

#### 🛡️ SECURITY & PACKAGE SCANNING

| Target         | Description |
|----------------|-------------|
| `make trivy`   | Scan the container image for vulnerabilities using [Trivy](https://aquasecurity.github.io/trivy/). |

> **Tip:** These commands should work out-of-the-box after cloning a repo and running `make venv install serve`.

## 🐳 Containerfile

Each repo must include a `Containerfile` (Podman-compatible, Docker-compatible) to support containerized execution.

### Containerfile Requirements:

- Must start from a secure base (e.g., latest Red Hat UBI9 minimal image `registry.access.redhat.com/ubi9-minimal:9.5-1741850109`)
- Should use `uv` or `pdm` to install dependencies via `pyproject.toml`
- Must run the server using the same entry point as `make serve`
- Should expose relevant ports (`EXPOSE 8080`)
- Should define a non-root user for runtime

## 📚 Dependency Management
- All Python projects must use `pyproject.toml` and follow PEP standards.
- Dependencies must either be:
  - Included in the repo
  - Pulled from PyPI (no external links)

## 🎯 Clear Role Definition
- State the **specific role** of the server (e.g., GitHub tools).
- Group related tools together.
- **Do not mix roles** (e.g., GitHub ≠ Jira ≠ GitLab).

## 🧰 Standardized Tools
Each MCP server should expose tools that follow the MCP conventions, e.g.:

- `create_ticket`
- `create_pr`
- `read_file`

## 📁 Consistent Structure
Repos must follow a common structure. For example, from the time_server

```
time_server/
├── Containerfile                  # Container build definition (Podman/Docker compatible)
├── Makefile                       # Build, run, test, and container automation targets
├── pyproject.toml                 # Python project and dependency configuration (PEP 621)
├── README.md                      # Main documentation: overview, setup, usage, env vars
├── CONTRIBUTING.md                # Guidelines for contributing, PRs, and issue management
├── .gitignore                     # Exclude venvs, artifacts, and secrets from Git
├── docs/                          # (Optional) Diagrams, specs, and additional documentation
├── tests/                         # Unit and integration tests
│   ├── __init__.py
│   ├── test_main.py               # Tests for main entrypoint behavior
│   └── test_tools.py              # Tests for core tool functionality
└── src/                           # Application source code
    └── mcp_time_server/           # Main package named after your server
        ├── __init__.py            # Marks this directory as a Python package
        ├── main.py                # Entrypoint that wires everything together
        ├── mcp_server_base.py     # Optional base class for shared server behavior
        ├── server.py              # Server logic (e.g., tool registration, lifecycle hooks)
        └── tools/                 # Directory for all MCP tool implementations
            ├── __init__.py
            ├── tools.py           # Tool business logic (e.g., `get_time`, `format_time`)
            └── tools_registration.py # Registers tools into the MCP framework
```

## 📝 Documentation
Each repo must include:

- A comprehensive `README.md`
- Setup and usage instructions
- Environment variable documentation

## 🧩 Modular Design
Code should be cleanly separated into modules for easier maintenance and scaling.

## ✅ Testing
Include **unit and integration tests** to validate functionality.

## 🤝 Contribution Guidelines
Add a `CONTRIBUTING.md` with:

- How to file issues
- How to submit pull requests
- Review and merge process

## 🏷 Versioning and Releases
Use **semantic versioning**.
Include **release notes** for all changes.

## 🔄 Pull Request Process
Submit new MCP servers via **pull request** to the org's main repo.
PR must:

- Follow all standards
- Include all documentation

## 🔐 Environment Variables and Secrets
- Use environment variables for secrets
- Use a clear, role-based prefix (e.g., `MCP_GITHUB_`)

**Example:**

```env
MCP_GITHUB_ACCESS_TOKEN=...
MCP_GITHUB_BASE_URL=...
```

## 🏷 Required Capabilities (README Metadata Tags)

Add tags at the top of `README.md` between YAML markers to declare your server's required capabilities.

### Available Tags:

- **`needs_filesystem_access`**
  Indicates the server requires access to the local filesystem (e.g., for reading/writing files).

- **`needs_api_key_user`**
  Requires a user-specific API key to interact with external services on behalf of the user.

- **`needs_api_key_central`**
  Requires a centrally managed API key, typically provisioned and stored by the platform.

- **`needs_database`**
  The server interacts with a persistent database (e.g., PostgreSQL, MongoDB).

- **`needs_network_access_inbound`**
  The server expects to receive inbound network requests (e.g., runs a web server or webhook listener).

- **`needs_network_access_outbound`**
  The server needs to make outbound network requests (e.g., calling external APIs or services).

### Example:

```markdown
---
tags:
  - needs_filesystem_access
  - needs_api_key_user
---
```
