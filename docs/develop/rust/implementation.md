# Rust Implementation Guide

## Building MCP Servers in Rust

This guide covers implementation patterns, best practices, and examples for building robust MCP servers using Rust's type system and async capabilities.

## Core MCP Types

```rust
// src/mcp/types.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

/// MCP protocol version
pub const MCP_VERSION: &str = "2024-11-05";

/// JSON-RPC 2.0 request
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Request {
    pub jsonrpc: String,
    pub id: Option<serde_json::Value>,
    pub method: String,
    pub params: Option<serde_json::Value>,
}

/// JSON-RPC 2.0 response
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Response {
    pub jsonrpc: String,
    pub id: Option<serde_json::Value>,
    #[serde(flatten)]
    pub result: ResponseResult,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(untagged)]
pub enum ResponseResult {
    Success { result: serde_json::Value },
    Error { error: ErrorObject },
}

/// JSON-RPC error object
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ErrorObject {
    pub code: i32,
    pub message: String,
    pub data: Option<serde_json::Value>,
}

/// Tool definition
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Tool {
    pub name: String,
    pub description: String,
    #[serde(rename = "inputSchema")]
    pub input_schema: serde_json::Value,
}

/// Tool execution result
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolResult {
    pub content: Vec<Content>,
    #[serde(rename = "isError", skip_serializing_if = "Option::is_none")]
    pub is_error: Option<bool>,
}

/// Content types
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]
pub enum Content {
    #[serde(rename = "text")]
    Text { text: String },
    #[serde(rename = "image")]
    Image { data: String, mime_type: String },
    #[serde(rename = "resource")]
    Resource { resource: ResourceContent },
}

/// Resource definition
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Resource {
    pub uri: String,
    pub name: String,
    pub description: Option<String>,
    #[serde(rename = "mimeType", skip_serializing_if = "Option::is_none")]
    pub mime_type: Option<String>,
}

/// Resource content
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ResourceContent {
    pub uri: String,
    #[serde(rename = "mimeType")]
    pub mime_type: String,
    pub text: Option<String>,
    pub blob: Option<String>,
}
```

## Tool Implementation Framework

```rust
// src/tools/mod.rs
use async_trait::async_trait;
use serde_json::Value;
use std::collections::HashMap;
use std::sync::Arc;

use crate::{
    mcp::types::{Content, Tool, ToolResult},
    error::Result,
};

/// Tool execution trait
#[async_trait]
pub trait ToolExecutor: Send + Sync {
    /// Get tool definition
    fn definition(&self) -> Tool;
    
    /// Execute the tool with given arguments
    async fn execute(
        &self,
        arguments: HashMap<String, Value>,
    ) -> Result<ToolResult>;
    
    /// Get tool name
    fn name(&self) -> &str;
    
    /// Validate arguments before execution
    fn validate_arguments(&self, arguments: &HashMap<String, Value>) -> Result<()> {
        // Default implementation - tools can override
        Ok(())
    }
}

/// Tool registry for managing available tools
pub struct ToolRegistry {
    tools: HashMap<String, Arc<dyn ToolExecutor>>,
}

impl ToolRegistry {
    pub fn new() -> Self {
        Self {
            tools: HashMap::new(),
        }
    }
    
    /// Register a tool
    pub fn register<T>(&mut self, tool: T) 
    where 
        T: ToolExecutor + 'static 
    {
        let name = tool.name().to_string();
        self.tools.insert(name, Arc::new(tool));
    }
    
    /// Get all available tools
    pub fn list_tools(&self) -> Vec<Tool> {
        self.tools.values()
            .map(|tool| tool.definition())
            .collect()
    }
    
    /// Execute a tool by name
    pub async fn execute_tool(
        &self,
        name: &str,
        arguments: HashMap<String, Value>,
    ) -> Result<ToolResult> {
        let tool = self.tools.get(name)
            .ok_or_else(|| crate::Error::not_found(format!("Tool not found: {}", name)))?;
        
        // Validate arguments
        tool.validate_arguments(&arguments)?;
        
        // Execute tool
        tool.execute(arguments).await
    }
    
    /// Check if tool exists
    pub fn has_tool(&self, name: &str) -> bool {
        self.tools.contains_key(name)
    }
}
```

## Database Tool Implementation

```rust
// src/tools/database.rs
use async_trait::async_trait;
use serde_json::{json, Value};
use sqlx::{PgPool, Row};
use std::collections::HashMap;
use tracing::{info, warn};

use crate::{
    mcp::types::{Content, Tool, ToolResult},
    tools::ToolExecutor,
    error::{Result, Error},
    config::DatabaseToolConfig,
};

/// Database query tool
pub struct DatabaseTool {
    pool: PgPool,
    config: DatabaseToolConfig,
}

impl DatabaseTool {
    pub async fn new(database_url: &str, config: DatabaseToolConfig) -> Result<Self> {
        let pool = PgPool::connect(database_url).await?;
        
        Ok(Self { pool, config })
    }
    
    fn validate_query(&self, query: &str) -> Result<()> {
        let query_upper = query.trim().to_uppercase();
        
        // Only allow SELECT statements if read_only is true
        if self.config.read_only && !query_upper.starts_with("SELECT") {
            return Err(Error::permission_denied(
                "Only SELECT statements are allowed in read-only mode"
            ));
        }
        
        // Block dangerous operations
        let dangerous_keywords = [
            "DROP", "DELETE", "INSERT", "UPDATE", "ALTER", 
            "CREATE", "TRUNCATE", "REPLACE", "MERGE"
        ];
        
        for keyword in &dangerous_keywords {
            if query_upper.contains(keyword) && self.config.read_only {
                return Err(Error::permission_denied(
                    format!("Query contains forbidden keyword: {}", keyword)
                ));
            }
        }
        
        Ok(())
    }
}

#[async_trait]
impl ToolExecutor for DatabaseTool {
    fn definition(&self) -> Tool {
        Tool {
            name: "query_database".to_string(),
            description: "Execute SQL queries against the database".to_string(),
            input_schema: json!({
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "SQL query to execute"
                    },
                    "limit": {
                        "type": "integer",
                        "description": "Maximum number of rows to return",
                        "default": self.config.max_rows,
                        "minimum": 1,
                        "maximum": self.config.max_rows
                    }
                },
                "required": ["query"]
            }),
        }
    }
    
    fn name(&self) -> &str {
        "query_database"
    }
    
    fn validate_arguments(&self, arguments: &HashMap<String, Value>) -> Result<()> {
        let query = arguments.get("query")
            .and_then(|v| v.as_str())
            .ok_or_else(|| Error::validation("query parameter is required and must be a string"))?;
        
        self.validate_query(query)?;
        
        if let Some(limit) = arguments.get("limit") {
            let limit = limit.as_u64()
                .ok_or_else(|| Error::validation("limit must be a positive integer"))?;
                
            if limit > self.config.max_rows as u64 {
                return Err(Error::validation(
                    format!("limit cannot exceed {}", self.config.max_rows)
                ));
            }
        }
        
        Ok(())
    }
    
    async fn execute(&self, arguments: HashMap<String, Value>) -> Result<ToolResult> {
        let query = arguments["query"].as_str().unwrap();
        let limit = arguments.get("limit")
            .and_then(|v| v.as_u64())
            .unwrap_or(self.config.max_rows as u64) as i64;
        
        info!("Executing database query: {}", query);
        
        // Add LIMIT clause if not present
        let final_query = if query.to_uppercase().contains("LIMIT") {
            query.to_string()
        } else {
            format!("{} LIMIT {}", query, limit)
        };
        
        // Execute query with timeout
        let rows = tokio::time::timeout(
            std::time::Duration::from_secs(self.config.timeout),
            sqlx::query(&final_query).fetch_all(&self.pool)
        ).await
        .map_err(|_| Error::internal("Query timeout"))?
        .map_err(|e| Error::tool_error("query_database", e.to_string()))?;
        
        // Convert rows to JSON
        let mut results = Vec::new();
        for row in rows {
            let mut row_map = serde_json::Map::new();
            
            for (i, column) in row.columns().iter().enumerate() {
                let column_name = column.name();
                let value: Value = match column.type_info().name() {
                    "TEXT" | "VARCHAR" | "CHAR" => {
                        row.try_get::<Option<String>, _>(i)
                            .map(|v| v.map(Value::String).unwrap_or(Value::Null))
                            .unwrap_or(Value::Null)
                    }
                    "INTEGER" | "INT4" | "INT8" | "BIGINT" => {
                        row.try_get::<Option<i64>, _>(i)
                            .map(|v| v.map(|n| Value::Number(n.into())).unwrap_or(Value::Null))
                            .unwrap_or(Value::Null)
                    }
                    "REAL" | "FLOAT4" | "FLOAT8" | "DOUBLE" => {
                        row.try_get::<Option<f64>, _>(i)
                            .map(|v| v.and_then(|n| serde_json::Number::from_f64(n).map(Value::Number)).unwrap_or(Value::Null))
                            .unwrap_or(Value::Null)
                    }
                    "BOOLEAN" | "BOOL" => {
                        row.try_get::<Option<bool>, _>(i)
                            .map(|v| v.map(Value::Bool).unwrap_or(Value::Null))
                            .unwrap_or(Value::Null)
                    }
                    _ => Value::Null, // Fallback for unsupported types
                };
                
                row_map.insert(column_name.to_string(), value);
            }
            
            results.push(Value::Object(row_map));
        }
        
        let results_json = serde_json::to_string_pretty(&results)
            .map_err(|e| Error::internal(format!("Failed to serialize results: {}", e)))?;
        
        Ok(ToolResult {
            content: vec![Content::Text { text: results_json }],
            is_error: None,
        })
    }
}
```

## HTTP Client Tool

```rust
// src/tools/http.rs
use async_trait::async_trait;
use reqwest::{Client, Method, Url};
use serde_json::{json, Value};
use std::collections::HashMap;
use std::time::Duration;
use tracing::{info, warn};

use crate::{
    mcp::types::{Content, Tool, ToolResult},
    tools::ToolExecutor,
    error::{Result, Error},
    config::HttpToolConfig,
};

/// HTTP request tool
pub struct HttpTool {
    client: Client,
    config: HttpToolConfig,
}

impl HttpTool {
    pub fn new(config: HttpToolConfig) -> Result<Self> {
        let client = Client::builder()
            .timeout(Duration::from_secs(config.timeout))
            .build()
            .map_err(|e| Error::internal(format!("Failed to create HTTP client: {}", e)))?;
        
        Ok(Self { client, config })
    }
    
    fn validate_url(&self, url: &str) -> Result<Url> {
        let parsed_url = Url::parse(url)
            .map_err(|e| Error::validation(format!("Invalid URL: {}", e)))?;
        
        // Check scheme
        if !matches!(parsed_url.scheme(), "http" | "https") {
            return Err(Error::validation("URL must use http or https scheme"));
        }
        
        // Check allowed/blocked domains
        if let Some(host) = parsed_url.host_str() {
            if let Some(blocked) = &self.config.blocked_domains {
                if blocked.iter().any(|domain| host.contains(domain)) {
                    return Err(Error::permission_denied(
                        format!("Domain {} is blocked", host)
                    ));
                }
            }
            
            if let Some(allowed) = &self.config.allowed_domains {
                if !allowed.iter().any(|domain| host.contains(domain)) {
                    return Err(Error::permission_denied(
                        format!("Domain {} is not allowed", host)
                    ));
                }
            }
        }
        
        Ok(parsed_url)
    }
}

#[async_trait]
impl ToolExecutor for HttpTool {
    fn definition(&self) -> Tool {
        Tool {
            name: "http_request".to_string(),
            description: "Make HTTP requests to external APIs".to_string(),
            input_schema: json!({
                "type": "object",
                "properties": {
                    "method": {
                        "type": "string",
                        "enum": ["GET", "POST", "PUT", "DELETE", "PATCH"],
                        "description": "HTTP method to use",
                        "default": "GET"
                    },
                    "url": {
                        "type": "string",
                        "description": "URL to make the request to",
                        "format": "uri"
                    },
                    "headers": {
                        "type": "object",
                        "description": "HTTP headers to include",
                        "additionalProperties": {"type": "string"}
                    },
                    "body": {
                        "type": "string",
                        "description": "Request body (for POST/PUT/PATCH requests)"
                    },
                    "timeout": {
                        "type": "integer",
                        "description": "Request timeout in seconds",
                        "default": self.config.timeout,
                        "minimum": 1,
                        "maximum": 300
                    }
                },
                "required": ["url"]
            }),
        }
    }
    
    fn name(&self) -> &str {
        "http_request"
    }
    
    fn validate_arguments(&self, arguments: &HashMap<String, Value>) -> Result<()> {
        let url = arguments.get("url")
            .and_then(|v| v.as_str())
            .ok_or_else(|| Error::validation("url parameter is required and must be a string"))?;
        
        self.validate_url(url)?;
        
        if let Some(method) = arguments.get("method") {
            let method_str = method.as_str()
                .ok_or_else(|| Error::validation("method must be a string"))?;
                
            if !matches!(method_str, "GET" | "POST" | "PUT" | "DELETE" | "PATCH") {
                return Err(Error::validation("Invalid HTTP method"));
            }
        }
        
        if let Some(timeout) = arguments.get("timeout") {
            let timeout = timeout.as_u64()
                .ok_or_else(|| Error::validation("timeout must be a positive integer"))?;
                
            if timeout > 300 {
                return Err(Error::validation("timeout cannot exceed 300 seconds"));
            }
        }
        
        Ok(())
    }
    
    async fn execute(&self, arguments: HashMap<String, Value>) -> Result<ToolResult> {
        let url = self.validate_url(arguments["url"].as_str().unwrap())?;
        
        let method_str = arguments.get("method")
            .and_then(|v| v.as_str())
            .unwrap_or("GET");
            
        let method = method_str.parse::<Method>()
            .map_err(|e| Error::validation(format!("Invalid method: {}", e)))?;
        
        info!("Making {} request to {}", method, url);
        
        let mut request = self.client.request(method, url.clone());
        
        // Add headers
        if let Some(headers) = arguments.get("headers").and_then(|v| v.as_object()) {
            for (key, value) in headers {
                if let Some(value_str) = value.as_str() {
                    request = request.header(key, value_str);
                }
            }
        }
        
        // Add body if present
        if let Some(body) = arguments.get("body").and_then(|v| v.as_str()) {
            request = request.body(body.to_string());
            
            // Set content-type if not already set
            if !arguments.get("headers")
                .and_then(|v| v.as_object())
                .map(|h| h.contains_key("content-type") || h.contains_key("Content-Type"))
                .unwrap_or(false)
            {
                request = request.header("content-type", "application/json");
            }
        }
        
        // Set timeout if specified
        if let Some(timeout) = arguments.get("timeout").and_then(|v| v.as_u64()) {
            request = request.timeout(Duration::from_secs(timeout));
        }
        
        // Execute request
        let response = request.send().await
            .map_err(|e| Error::tool_error("http_request", e.to_string()))?;
        
        let status = response.status();
        let headers: HashMap<String, String> = response.headers()
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_str().unwrap_or("").to_string()))
            .collect();
        
        // Read response body with size limit
        let body_bytes = response.bytes().await
            .map_err(|e| Error::tool_error("http_request", e.to_string()))?;
        
        if body_bytes.len() > self.config.max_response_size as usize {
            warn!("Response body too large: {} bytes", body_bytes.len());
            return Err(Error::tool_error("http_request", 
                format!("Response body too large: {} bytes", body_bytes.len())));
        }
        
        let body = String::from_utf8_lossy(&body_bytes).into_owned();
        
        let result = json!({
            "status_code": status.as_u16(),
            "status": status.to_string(),
            "headers": headers,
            "body": body,
            "url": url.to_string()
        });
        
        let result_text = serde_json::to_string_pretty(&result)
            .map_err(|e| Error::internal(format!("Failed to serialize response: {}", e)))?;
        
        Ok(ToolResult {
            content: vec![Content::Text { text: result_text }],
            is_error: Some(status.is_client_error() || status.is_server_error()),
        })
    }
}
```

## File System Tool

```rust
// src/tools/filesystem.rs
use async_trait::async_trait;
use serde_json::{json, Value};
use std::collections::HashMap;
use std::path::{Path, PathBuf};
use tokio::fs;
use tracing::{info, warn};

use crate::{
    mcp::types::{Content, Tool, ToolResult},
    tools::ToolExecutor,
    error::{Result, Error},
    config::FilesystemToolConfig,
};

/// Filesystem operations tool
pub struct FilesystemTool {
    config: FilesystemToolConfig,
}

impl FilesystemTool {
    pub fn new(config: FilesystemToolConfig) -> Self {
        Self { config }
    }
    
    fn validate_path(&self, path: &str) -> Result<PathBuf> {
        // Prevent path traversal
        if path.contains("..") {
            return Err(Error::permission_denied("Path traversal not allowed"));
        }
        
        let path_buf = PathBuf::from(path);
        let canonical_path = path_buf.canonicalize()
            .map_err(|_| Error::not_found(format!("Path does not exist: {}", path)))?;
        
        // Check if path is within allowed directories
        let mut allowed = false;
        for allowed_path in &self.config.allowed_paths {
            let allowed_canonical = PathBuf::from(allowed_path).canonicalize()
                .map_err(|_| Error::internal(format!("Invalid allowed path: {}", allowed_path)))?;
            
            if canonical_path.starts_with(&allowed_canonical) {
                allowed = true;
                break;
            }
        }
        
        if !allowed {
            return Err(Error::permission_denied(
                format!("Path not in allowed directories: {}", path)
            ));
        }
        
        Ok(canonical_path)
    }
}

#[async_trait]
impl ToolExecutor for FilesystemTool {
    fn definition(&self) -> Tool {
        let mut operations = vec!["read_file", "list_directory"];
        if !self.config.read_only {
            operations.extend_from_slice(&["write_file", "create_directory", "delete_file"]);
        }
        
        Tool {
            name: "filesystem".to_string(),
            description: "Perform filesystem operations".to_string(),
            input_schema: json!({
                "type": "object",
                "properties": {
                    "operation": {
                        "type": "string",
                        "enum": operations,
                        "description": "Filesystem operation to perform"
                    },
                    "path": {
                        "type": "string",
                        "description": "File or directory path"
                    },
                    "content": {
                        "type": "string",
                        "description": "Content to write (for write_file operation)"
                    }
                },
                "required": ["operation", "path"]
            }),
        }
    }
    
    fn name(&self) -> &str {
        "filesystem"
    }
    
    fn validate_arguments(&self, arguments: &HashMap<String, Value>) -> Result<()> {
        let operation = arguments.get("operation")
            .and_then(|v| v.as_str())
            .ok_or_else(|| Error::validation("operation parameter is required"))?;
        
        let path = arguments.get("path")
            .and_then(|v| v.as_str())
            .ok_or_else(|| Error::validation("path parameter is required"))?;
        
        self.validate_path(path)?;
        
        // Check if write operations are allowed
        if self.config.read_only && matches!(operation, "write_file" | "create_directory" | "delete_file") {
            return Err(Error::permission_denied(
                format!("Operation {} not allowed in read-only mode", operation)
            ));
        }
        
        // Validate content for write operations
        if operation == "write_file" && !arguments.contains_key("content") {
            return Err(Error::validation("content parameter is required for write_file operation"));
        }
        
        Ok(())
    }
    
    async fn execute(&self, arguments: HashMap<String, Value>) -> Result<ToolResult> {
        let operation = arguments["operation"].as_str().unwrap();
        let path = self.validate_path(arguments["path"].as_str().unwrap())?;
        
        info!("Performing filesystem operation: {} on {}", operation, path.display());
        
        let result_text = match operation {
            "read_file" => {
                let metadata = fs::metadata(&path).await
                    .map_err(|e| Error::tool_error("filesystem", e.to_string()))?;
                
                if metadata.len() > self.config.max_file_size {
                    return Err(Error::permission_denied(
                        format!("File too large: {} bytes", metadata.len())
                    ));
                }
                
                let content = fs::read_to_string(&path).await
                    .map_err(|e| Error::tool_error("filesystem", e.to_string()))?;
                
                json!({
                    "operation": "read_file",
                    "path": path.to_string_lossy(),
                    "content": content,
                    "size": metadata.len()
                }).to_string()
            },
            
            "list_directory" => {
                let mut entries = Vec::new();
                let mut dir = fs::read_dir(&path).await
                    .map_err(|e| Error::tool_error("filesystem", e.to_string()))?;
                
                while let Some(entry) = dir.next_entry().await
                    .map_err(|e| Error::tool_error("filesystem", e.to_string()))? {
                    
                    let metadata = entry.metadata().await
                        .map_err(|e| Error::tool_error("filesystem", e.to_string()))?;
                    
                    entries.push(json!({
                        "name": entry.file_name().to_string_lossy(),
                        "path": entry.path().to_string_lossy(),
                        "is_file": metadata.is_file(),
                        "is_dir": metadata.is_dir(),
                        "size": metadata.len()
                    }));
                }
                
                json!({
                    "operation": "list_directory",
                    "path": path.to_string_lossy(),
                    "entries": entries
                }).to_string()
            },
            
            "write_file" => {
                if self.config.read_only {
                    return Err(Error::permission_denied("Write operations not allowed"));
                }
                
                let content = arguments["content"].as_str()
                    .ok_or_else(|| Error::validation("content parameter is required"))?;
                
                if content.len() > self.config.max_file_size as usize {
                    return Err(Error::permission_denied(
                        format!("Content too large: {} bytes", content.len())
                    ));
                }
                
                fs::write(&path, content).await
                    .map_err(|e| Error::tool_error("filesystem", e.to_string()))?;
                
                json!({
                    "operation": "write_file",
                    "path": path.to_string_lossy(),
                    "bytes_written": content.len()
                }).to_string()
            },
            
            _ => {
                return Err(Error::validation(format!("Unknown operation: {}", operation)));
            }
        };
        
        Ok(ToolResult {
            content: vec![Content::Text { text: result_text }],
            is_error: None,
        })
    }
}
```

This Rust implementation provides a robust foundation for building MCP servers with strong type safety, comprehensive error handling, and async support for high-performance operations.