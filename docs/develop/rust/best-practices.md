# Rust Best Practices for MCP Servers

## Rust-Specific Best Practices

This guide covers Rust-specific best practices, patterns, and idioms for building robust, efficient, and maintainable MCP servers that leverage Rust's unique strengths.

## Code Organization and Architecture

### Module Structure and Visibility

```rust
// ✅ Good: Clear module hierarchy with appropriate visibility
pub mod server {
    pub mod handlers;
    pub mod transport;
    
    // Re-export public API
    pub use handlers::Handlers;
    pub use transport::Transport;
}

pub mod tools {
    mod database;   // Private implementation
    mod http;       // Private implementation
    
    pub use database::DatabaseTool;
    pub use http::HttpTool;
    
    // Trait for tool implementations
    pub trait ToolExecutor: Send + Sync {
        // Method definitions
    }
}

// ❌ Bad: Everything public or unclear hierarchy
pub mod everything {
    pub mod database;
    pub mod http;
    pub mod utils;
    pub mod helpers;
    pub mod stuff;
}
```

### Error Handling with thiserror and anyhow

```rust
// ✅ Good: Structured error types with context
use thiserror::Error;

#[derive(Error, Debug)]
pub enum McpError {
    #[error("Configuration error: {message}")]
    Config { message: String },
    
    #[error("Tool '{tool_name}' execution failed: {source}")]
    ToolExecution {
        tool_name: String,
        #[source]
        source: Box<dyn std::error::Error + Send + Sync>,
    },
    
    #[error("Validation failed for field '{field}': {message}")]
    Validation { field: String, message: String },
    
    #[error("Resource '{uri}' not found")]
    ResourceNotFound { uri: String },
    
    #[error("Permission denied: {operation}")]
    PermissionDenied { operation: String },
    
    #[error("I/O error")]
    Io(#[from] std::io::Error),
    
    #[error("HTTP request failed")]
    Http(#[from] reqwest::Error),
    
    #[error("JSON serialization error")]
    Json(#[from] serde_json::Error),
}

// Extension trait for better error context
pub trait McpErrorExt<T> {
    fn with_tool_context(self, tool_name: &str) -> Result<T, McpError>;
    fn with_validation_context(self, field: &str) -> Result<T, McpError>;
}

impl<T, E> McpErrorExt<T> for Result<T, E>
where
    E: std::error::Error + Send + Sync + 'static,
{
    fn with_tool_context(self, tool_name: &str) -> Result<T, McpError> {
        self.map_err(|e| McpError::ToolExecution {
            tool_name: tool_name.to_string(),
            source: Box::new(e),
        })
    }
    
    fn with_validation_context(self, field: &str) -> Result<T, McpError> {
        self.map_err(|e| McpError::Validation {
            field: field.to_string(),
            message: e.to_string(),
        })
    }
}

// Usage
fn execute_tool(name: &str) -> Result<String, McpError> {
    some_operation()
        .with_tool_context(name)?
        .into_string()
        .with_validation_context("result")
}
```

### Type Safety with Newtypes

```rust
// ✅ Good: Strong typing with newtypes
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
pub struct ToolName(String);

impl ToolName {
    pub fn new(name: impl Into<String>) -> Result<Self, McpError> {
        let name = name.into();
        if name.is_empty() {
            return Err(McpError::Validation {
                field: "tool_name".to_string(),
                message: "Tool name cannot be empty".to_string(),
            });
        }
        if name.len() > 100 {
            return Err(McpError::Validation {
                field: "tool_name".to_string(),
                message: "Tool name too long".to_string(),
            });
        }
        Ok(Self(name))
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl std::fmt::Display for ToolName {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{}", self.0)
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceUri(String);

impl ResourceUri {
    pub fn new(uri: impl Into<String>) -> Result<Self, McpError> {
        let uri = uri.into();
        url::Url::parse(&uri)
            .map_err(|e| McpError::Validation {
                field: "resource_uri".to_string(),
                message: format!("Invalid URI: {}", e),
            })?;
        Ok(Self(uri))
    }
}

// ❌ Bad: Stringly typed parameters
fn execute_tool(name: String, uri: String) -> Result<String, McpError> {
    // No compile-time guarantees about validity
    unimplemented!()
}

// ✅ Good: Type-safe parameters
fn execute_tool(name: ToolName, uri: ResourceUri) -> Result<String, McpError> {
    // Guaranteed valid at compile time
    unimplemented!()
}
```

## Async Programming Best Practices

### Structured Concurrency

```rust
// ✅ Good: Structured concurrency with proper error handling
use tokio::select;
use tokio::sync::mpsc;
use tokio::time::{timeout, Duration};

pub struct ToolExecutorPool {
    workers: Vec<tokio::task::JoinHandle<()>>,
    task_tx: mpsc::UnboundedSender<ToolTask>,
    shutdown_tx: mpsc::Sender<()>,
}

impl ToolExecutorPool {
    pub async fn new(worker_count: usize) -> Self {
        let (task_tx, task_rx) = mpsc::unbounded_channel();
        let (shutdown_tx, shutdown_rx) = mpsc::channel(1);
        let task_rx = Arc::new(Mutex::new(task_rx));
        let shutdown_rx = Arc::new(Mutex::new(shutdown_rx));
        
        let mut workers = Vec::new();
        
        for worker_id in 0..worker_count {
            let task_rx = task_rx.clone();
            let shutdown_rx = shutdown_rx.clone();
            
            let handle = tokio::spawn(async move {
                Self::worker_loop(worker_id, task_rx, shutdown_rx).await;
            });
            
            workers.push(handle);
        }
        
        Self {
            workers,
            task_tx,
            shutdown_tx,
        }
    }
    
    async fn worker_loop(
        worker_id: usize,
        task_rx: Arc<Mutex<mpsc::UnboundedReceiver<ToolTask>>>,
        shutdown_rx: Arc<Mutex<mpsc::Receiver<()>>>,
    ) {
        tracing::info!("Worker {} started", worker_id);
        
        loop {
            select! {
                // Receive shutdown signal
                _ = async {
                    let mut rx = shutdown_rx.lock().await;
                    rx.recv().await
                } => {
                    tracing::info!("Worker {} shutting down", worker_id);
                    break;
                }
                
                // Process task
                task = async {
                    let mut rx = task_rx.lock().await;
                    rx.recv().await
                } => {
                    match task {
                        Some(task) => {
                            if let Err(e) = Self::process_task(task).await {
                                tracing::error!("Worker {} task failed: {}", worker_id, e);
                            }
                        }
                        None => break, // Channel closed
                    }
                }
            }
        }
        
        tracing::info!("Worker {} stopped", worker_id);
    }
    
    async fn process_task(task: ToolTask) -> Result<(), McpError> {
        // Add timeout to prevent hanging
        let result = timeout(Duration::from_secs(30), async {
            task.tool.execute(task.arguments).await
        }).await;
        
        match result {
            Ok(Ok(result)) => {
                let _ = task.response_tx.send(Ok(result));
            }
            Ok(Err(e)) => {
                let _ = task.response_tx.send(Err(e));
            }
            Err(_) => {
                let _ = task.response_tx.send(Err(McpError::ToolExecution {
                    tool_name: "unknown".to_string(),
                    source: Box::new(std::io::Error::new(
                        std::io::ErrorKind::TimedOut,
                        "Task execution timed out"
                    )),
                }));
            }
        }
        
        Ok(())
    }
    
    pub async fn execute_tool(
        &self,
        tool: Arc<dyn ToolExecutor>,
        arguments: HashMap<String, Value>,
    ) -> Result<ToolResult, McpError> {
        let (response_tx, mut response_rx) = mpsc::channel(1);
        
        let task = ToolTask {
            tool,
            arguments,
            response_tx,
        };
        
        self.task_tx.send(task)
            .map_err(|_| McpError::Config {
                message: "Tool executor pool is shut down".to_string(),
            })?;
        
        response_rx.recv().await
            .ok_or_else(|| McpError::Config {
                message: "Failed to receive task result".to_string(),
            })?
    }
    
    pub async fn shutdown(self) {
        // Signal shutdown
        let _ = self.shutdown_tx.send(()).await;
        
        // Wait for all workers to finish
        for handle in self.workers {
            let _ = handle.await;
        }
    }
}

struct ToolTask {
    tool: Arc<dyn ToolExecutor>,
    arguments: HashMap<String, Value>,
    response_tx: mpsc::Sender<Result<ToolResult, McpError>>,
}
```

### Resource Management with RAII

```rust
// ✅ Good: RAII pattern with proper cleanup
pub struct DatabaseConnection {
    pool: sqlx::PgPool,
    connection_id: uuid::Uuid,
    acquired_at: std::time::Instant,
}

impl DatabaseConnection {
    pub async fn acquire(pool: &sqlx::PgPool) -> Result<Self, McpError> {
        let connection_id = uuid::Uuid::new_v4();
        let acquired_at = std::time::Instant::now();
        
        tracing::debug!("Acquiring database connection {}", connection_id);
        
        Ok(Self {
            pool: pool.clone(),
            connection_id,
            acquired_at,
        })
    }
    
    pub async fn execute_query(&self, query: &str) -> Result<Vec<serde_json::Value>, McpError> {
        tracing::debug!("Executing query on connection {}", self.connection_id);
        
        let rows = sqlx::query(query)
            .fetch_all(&self.pool)
            .await
            .map_err(|e| McpError::ToolExecution {
                tool_name: "database".to_string(),
                source: Box::new(e),
            })?;
        
        // Convert rows to JSON...
        Ok(vec![])
    }
}

impl Drop for DatabaseConnection {
    fn drop(&mut self) {
        let duration = self.acquired_at.elapsed();
        tracing::debug!(
            "Releasing database connection {} after {:?}",
            self.connection_id,
            duration
        );
    }
}

// Usage ensures automatic cleanup
async fn use_database() -> Result<(), McpError> {
    let pool = get_pool().await?;
    let conn = DatabaseConnection::acquire(&pool).await?;
    
    conn.execute_query("SELECT 1").await?;
    // Connection automatically cleaned up when `conn` goes out of scope
    
    Ok(())
}
```

## Memory Management and Performance

### Zero-Copy Operations

```rust
// ✅ Good: Zero-copy string handling
use std::borrow::Cow;

pub fn process_content<'a>(input: &'a str, should_transform: bool) -> Cow<'a, str> {
    if should_transform {
        // Only allocate when transformation is needed
        Cow::Owned(input.to_uppercase())
    } else {
        // Return borrowed reference
        Cow::Borrowed(input)
    }
}

// ✅ Good: Efficient JSON handling with serde_json::RawValue
use serde_json::value::RawValue;

pub struct ToolArguments<'a> {
    // Store raw JSON to avoid unnecessary parsing
    raw: &'a RawValue,
}

impl<'a> ToolArguments<'a> {
    pub fn get_string(&self, key: &str) -> Result<Option<&str>, McpError> {
        let obj: serde_json::Map<String, serde_json::Value> = 
            serde_json::from_str(self.raw.get())?;
        
        Ok(obj.get(key).and_then(|v| v.as_str()))
    }
    
    pub fn parse_into<T: serde::de::DeserializeOwned>(&self) -> Result<T, McpError> {
        serde_json::from_str(self.raw.get())
            .map_err(|e| McpError::Json(e))
    }
}
```

### Custom Allocators and Memory Pools

```rust
// ✅ Good: Object pooling for frequently allocated objects
use std::sync::{Arc, Mutex};

pub struct BufferPool {
    buffers: Arc<Mutex<Vec<Vec<u8>>>>,
    max_capacity: usize,
    buffer_size: usize,
}

impl BufferPool {
    pub fn new(max_capacity: usize, buffer_size: usize) -> Self {
        Self {
            buffers: Arc::new(Mutex::new(Vec::new())),
            max_capacity,
            buffer_size,
        }
    }
    
    pub fn get(&self) -> PooledBuffer {
        let mut pool = self.buffers.lock().unwrap();
        let buffer = pool.pop().unwrap_or_else(|| Vec::with_capacity(self.buffer_size));
        
        PooledBuffer {
            buffer: Some(buffer),
            pool: self.buffers.clone(),
        }
    }
}

pub struct PooledBuffer {
    buffer: Option<Vec<u8>>,
    pool: Arc<Mutex<Vec<Vec<u8>>>>,
}

impl PooledBuffer {
    pub fn as_mut(&mut self) -> &mut Vec<u8> {
        self.buffer.as_mut().unwrap()
    }
}

impl Drop for PooledBuffer {
    fn drop(&mut self) {
        if let Some(mut buffer) = self.buffer.take() {
            buffer.clear(); // Reset but keep capacity
            
            let mut pool = self.pool.lock().unwrap();
            if pool.len() < pool.capacity() {
                pool.push(buffer);
            }
        }
    }
}
```

## Security Best Practices

### Input Validation with Type System

```rust
// ✅ Good: Validation at type level
use serde::{Deserialize, Deserializer};
use std::fmt;

#[derive(Debug, Clone)]
pub struct SafeFilePath(std::path::PathBuf);

impl SafeFilePath {
    pub fn new(path: impl AsRef<std::path::Path>) -> Result<Self, McpError> {
        let path = path.as_ref();
        
        // Prevent path traversal
        if path.to_string_lossy().contains("..") {
            return Err(McpError::ValidationError {
                field: "path".to_string(),
                message: "Path traversal not allowed".to_string(),
            });
        }
        
        // Ensure absolute path
        let canonical = path.canonicalize()
            .map_err(|e| McpError::ValidationError {
                field: "path".to_string(),
                message: format!("Invalid path: {}", e),
            })?;
        
        Ok(Self(canonical))
    }
    
    pub fn as_path(&self) -> &std::path::Path {
        &self.0
    }
}

impl<'de> Deserialize<'de> for SafeFilePath {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        let s = String::deserialize(deserializer)?;
        SafeFilePath::new(s).map_err(serde::de::Error::custom)
    }
}

#[derive(Debug, Clone)]
pub struct SafeUrl(url::Url);

impl SafeUrl {
    pub fn new(url_str: &str, allowed_schemes: &[&str]) -> Result<Self, McpError> {
        let url = url::Url::parse(url_str)
            .map_err(|e| McpError::ValidationError {
                field: "url".to_string(),
                message: format!("Invalid URL: {}", e),
            })?;
        
        if !allowed_schemes.contains(&url.scheme()) {
            return Err(McpError::ValidationError {
                field: "url".to_string(),
                message: format!("Scheme '{}' not allowed", url.scheme()),
            });
        }
        
        Ok(Self(url))
    }
    
    pub fn as_url(&self) -> &url::Url {
        &self.0
    }
}

// Usage in tool arguments
#[derive(Deserialize)]
pub struct FileReadArgs {
    pub path: SafeFilePath,
}

#[derive(Deserialize)]
pub struct HttpRequestArgs {
    pub url: SafeUrl,
    pub method: Option<HttpMethod>,
}
```

### Secure Default Configurations

```rust
// ✅ Good: Secure defaults with explicit opt-in for permissive settings
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SecurityConfig {
    /// Allow dangerous operations (default: false)
    #[serde(default)]
    pub allow_dangerous_operations: bool,
    
    /// Maximum request size in bytes (default: 1MB)
    #[serde(default = "default_max_request_size")]
    pub max_request_size: u64,
    
    /// Request timeout in seconds (default: 30)
    #[serde(default = "default_request_timeout")]
    pub request_timeout: u64,
    
    /// Allowed file extensions (default: empty = all allowed)
    #[serde(default)]
    pub allowed_file_extensions: Vec<String>,
    
    /// Rate limiting (requests per minute, default: 60)
    #[serde(default = "default_rate_limit")]
    pub rate_limit: u32,
}

fn default_max_request_size() -> u64 { 1024 * 1024 } // 1MB
fn default_request_timeout() -> u64 { 30 }
fn default_rate_limit() -> u32 { 60 }

impl Default for SecurityConfig {
    fn default() -> Self {
        Self {
            allow_dangerous_operations: false,
            max_request_size: default_max_request_size(),
            request_timeout: default_request_timeout(),
            allowed_file_extensions: Vec::new(), // Empty = all allowed
            rate_limit: default_rate_limit(),
        }
    }
}

impl SecurityConfig {
    /// Validate security configuration
    pub fn validate(&self) -> Result<(), McpError> {
        if self.max_request_size > 100 * 1024 * 1024 {
            return Err(McpError::ValidationError {
                field: "max_request_size".to_string(),
                message: "Maximum request size too large (>100MB)".to_string(),
            });
        }
        
        if self.request_timeout > 300 {
            return Err(McpError::ValidationError {
                field: "request_timeout".to_string(),
                message: "Request timeout too long (>5 minutes)".to_string(),
            });
        }
        
        Ok(())
    }
    
    /// Check if file extension is allowed
    pub fn is_file_extension_allowed(&self, path: &std::path::Path) -> bool {
        if self.allowed_file_extensions.is_empty() {
            return true; // All extensions allowed
        }
        
        path.extension()
            .and_then(|ext| ext.to_str())
            .map(|ext| self.allowed_file_extensions.contains(&ext.to_string()))
            .unwrap_or(false)
    }
}
```

## Testing Patterns

### Property-Based Testing

```rust
// ✅ Good: Property-based testing for robust validation
use quickcheck::{quickcheck, TestResult};
use quickcheck_macros::quickcheck;

#[quickcheck]
fn prop_safe_file_path_prevents_traversal(input: String) -> TestResult {
    // Skip inputs that are valid paths to focus on traversal attempts
    if !input.contains("..") {
        return TestResult::discard();
    }
    
    let result = SafeFilePath::new(&input);
    TestResult::from_bool(result.is_err())
}

#[quickcheck]
fn prop_tool_name_length_limits(input: String) -> TestResult {
    let result = ToolName::new(input.clone());
    
    if input.is_empty() || input.len() > 100 {
        TestResult::from_bool(result.is_err())
    } else {
        TestResult::from_bool(result.is_ok())
    }
}

// Test tool execution properties
#[quickcheck]
fn prop_tool_execution_timeout(name: String, args: HashMap<String, Value>) -> TestResult {
    // Create a slow tool for testing
    struct SlowTool;
    
    #[async_trait::async_trait]
    impl ToolExecutor for SlowTool {
        fn name(&self) -> &str { "slow_tool" }
        
        async fn execute(&self, _args: HashMap<String, Value>) -> Result<ToolResult, McpError> {
            tokio::time::sleep(Duration::from_secs(10)).await;
            Ok(ToolResult { content: vec![], is_error: None })
        }
    }
    
    // Test should complete within reasonable time due to timeout
    let rt = tokio::runtime::Runtime::new().unwrap();
    let result = rt.block_on(async {
        let tool = Arc::new(SlowTool);
        
        tokio::time::timeout(
            Duration::from_secs(1),
            execute_tool_with_timeout(tool, args, Duration::from_millis(500))
        ).await
    });
    
    TestResult::from_bool(result.is_ok())
}

async fn execute_tool_with_timeout(
    tool: Arc<dyn ToolExecutor>,
    args: HashMap<String, Value>,
    timeout: Duration,
) -> Result<ToolResult, McpError> {
    tokio::time::timeout(timeout, tool.execute(args))
        .await
        .map_err(|_| McpError::ToolExecution {
            tool_name: tool.name().to_string(),
            source: Box::new(std::io::Error::new(
                std::io::ErrorKind::TimedOut,
                "Tool execution timed out"
            )),
        })?
}
```

### Fuzzing Integration

```rust
// ✅ Good: Fuzz testing for security-critical components
#[cfg(fuzzing)]
pub mod fuzz {
    use super::*;
    
    pub fn fuzz_json_parsing(data: &[u8]) {
        if let Ok(s) = std::str::from_utf8(data) {
            let _ = serde_json::from_str::<serde_json::Value>(s);
        }
    }
    
    pub fn fuzz_tool_arguments(data: &[u8]) {
        if let Ok(s) = std::str::from_utf8(data) {
            if let Ok(value) = serde_json::from_str::<serde_json::Value>(s) {
                if let Ok(map) = serde_json::from_value::<HashMap<String, serde_json::Value>>(value) {
                    // Test validation doesn't panic
                    for tool in get_all_tools() {
                        let _ = tool.validate_arguments(&map);
                    }
                }
            }
        }
    }
    
    pub fn fuzz_url_validation(data: &[u8]) {
        if let Ok(s) = std::str::from_utf8(data) {
            let _ = SafeUrl::new(s, &["http", "https"]);
        }
    }
}

// Cargo.toml should include:
// [dependencies]
// afl = { version = "0.12", optional = true }
// 
// [features]
// fuzzing = ["afl"]
```

## Performance Optimization

### Compile-Time Optimizations

```rust
// ✅ Good: Compile-time string interning
use std::sync::LazyLock;
use std::collections::HashMap;

static TOOL_SCHEMAS: LazyLock<HashMap<&'static str, serde_json::Value>> = LazyLock::new(|| {
    let mut schemas = HashMap::new();
    
    schemas.insert("http_request", serde_json::json!({
        "type": "object",
        "properties": {
            "url": {"type": "string", "format": "uri"},
            "method": {"type": "string", "enum": ["GET", "POST", "PUT", "DELETE"]}
        },
        "required": ["url"]
    }));
    
    schemas.insert("read_file", serde_json::json!({
        "type": "object",
        "properties": {
            "path": {"type": "string"}
        },
        "required": ["path"]
    }));
    
    schemas
});

// ✅ Good: Const generics for compile-time validation
pub struct BoundedString<const MAX_LEN: usize> {
    value: String,
}

impl<const MAX_LEN: usize> BoundedString<MAX_LEN> {
    pub fn new(value: String) -> Result<Self, McpError> {
        if value.len() > MAX_LEN {
            return Err(McpError::ValidationError {
                field: "bounded_string".to_string(),
                message: format!("String too long: {} > {}", value.len(), MAX_LEN),
            });
        }
        Ok(Self { value })
    }
    
    pub fn as_str(&self) -> &str {
        &self.value
    }
}

// Usage with different bounds
type ToolName = BoundedString<100>;
type Description = BoundedString<1000>;
```

These Rust-specific best practices ensure your MCP server leverages Rust's unique strengths in memory safety, performance, and concurrent programming while maintaining code clarity and maintainability.