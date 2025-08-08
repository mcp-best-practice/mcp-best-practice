# Rust Project Structure

## Recommended Structure for MCP Servers in Rust

This guide outlines the optimal project structure for building MCP servers using Rust, focusing on modularity, testability, and maintainability.

## Basic Project Layout

```
mcp-server-rust/
├── Cargo.toml              # Project manifest
├── Cargo.lock              # Dependency lockfile
├── README.md               # Project documentation
├── LICENSE                 # License file
├── .gitignore              # Git ignore rules
├── .rustfmt.toml           # Code formatting config
├── clippy.toml             # Linter configuration
├── src/                    # Source code
│   ├── main.rs             # Application entry point
│   ├── lib.rs              # Library root
│   ├── config/             # Configuration handling
│   │   ├── mod.rs
│   │   └── settings.rs
│   ├── server/             # MCP server implementation
│   │   ├── mod.rs
│   │   ├── handlers.rs
│   │   └── transport.rs
│   ├── tools/              # Tool implementations
│   │   ├── mod.rs
│   │   ├── database.rs
│   │   ├── http.rs
│   │   └── filesystem.rs
│   ├── resources/          # Resource implementations
│   │   ├── mod.rs
│   │   └── config.rs
│   ├── error/              # Error handling
│   │   ├── mod.rs
│   │   └── types.rs
│   └── utils/              # Utility functions
│       ├── mod.rs
│       └── validation.rs
├── tests/                  # Integration tests
│   ├── integration/
│   │   ├── mod.rs
│   │   ├── server_tests.rs
│   │   └── tool_tests.rs
│   └── common/
│       ├── mod.rs
│       └── fixtures.rs
├── benches/                # Benchmarks
│   └── tool_benchmarks.rs
├── examples/               # Example code
│   ├── basic_server.rs
│   └── custom_tools.rs
└── docs/                   # Documentation
    ├── api.md
    └── deployment.md
```

## Cargo.toml Configuration

```toml
[package]
name = "mcp-server-rust"
version = "0.1.0"
edition = "2021"
authors = ["Your Name <your.email@example.com>"]
description = "A Model Context Protocol server implementation in Rust"
repository = "https://github.com/your-org/mcp-server-rust"
license = "MIT"
keywords = ["mcp", "ai", "protocol", "server"]
categories = ["web-programming", "api-bindings"]

[[bin]]
name = "mcp-server"
path = "src/main.rs"

[lib]
name = "mcp_server_rust"
path = "src/lib.rs"

[dependencies]
# Async runtime
tokio = { version = "1.0", features = ["full"] }
tokio-util = "0.7"

# HTTP and networking
reqwest = { version = "0.11", features = ["json"] }
hyper = { version = "0.14", features = ["full"] }
tower = "0.4"
tower-http = { version = "0.4", features = ["cors", "trace"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_yaml = "0.9"

# Database (optional)
sqlx = { version = "0.7", features = ["runtime-tokio-rustls", "postgres", "uuid", "chrono"], optional = true }

# Logging and tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

# Error handling
anyhow = "1.0"
thiserror = "1.0"

# Configuration
config = "0.13"
clap = { version = "4.0", features = ["derive"] }

# Utilities
uuid = { version = "1.0", features = ["v4"] }
chrono = { version = "0.4", features = ["serde"] }
url = "2.0"

[dev-dependencies]
# Testing
tokio-test = "0.4"
assert_matches = "1.5"
tempfile = "3.0"
mockall = "0.11"

# Benchmarking
criterion = { version = "0.5", features = ["html_reports"] }

[features]
default = ["database"]
database = ["sqlx"]
metrics = []

[profile.release]
lto = true
codegen-units = 1
panic = "abort"
strip = true

[profile.dev]
debug = true
opt-level = 0

[[example]]
name = "basic_server"
path = "examples/basic_server.rs"

[[bench]]
name = "tool_benchmarks"
harness = false
```

## Main Entry Point

```rust
// src/main.rs
use std::process;

use clap::Parser;
use tracing::{info, error};
use tracing_subscriber;

use mcp_server_rust::{
    config::Settings,
    server::Server,
    error::Result,
};

#[derive(Parser)]
#[command(name = "mcp-server")]
#[command(about = "A Model Context Protocol server implementation")]
struct Cli {
    /// Configuration file path
    #[arg(short, long, default_value = "config.yaml")]
    config: String,
    
    /// Log level
    #[arg(long, default_value = "info")]
    log_level: String,
    
    /// Server port (overrides config)
    #[arg(short, long)]
    port: Option<u16>,
}

#[tokio::main]
async fn main() -> Result<()> {
    let cli = Cli::parse();
    
    // Initialize tracing
    tracing_subscriber::fmt()
        .with_env_filter(&cli.log_level)
        .init();
    
    info!("Starting MCP server");
    
    // Load configuration
    let mut settings = Settings::load(&cli.config)?;
    
    // Override port if provided
    if let Some(port) = cli.port {
        settings.server.port = port;
    }
    
    // Create and run server
    let server = Server::new(settings).await?;
    
    if let Err(e) = server.run().await {
        error!("Server error: {}", e);
        process::exit(1);
    }
    
    Ok(())
}
```

## Library Root

```rust
// src/lib.rs
//! # MCP Server Rust
//! 
//! A Model Context Protocol server implementation in Rust.
//! 
//! ## Features
//! 
//! - Asynchronous processing with Tokio
//! - Type-safe configuration management
//! - Comprehensive error handling
//! - Built-in tools for common operations
//! - Resource management
//! - Extensible architecture

pub mod config;
pub mod error;
pub mod resources;
pub mod server;
pub mod tools;
pub mod utils;

// Re-export commonly used types
pub use error::{Error, Result};
pub use server::Server;
pub use config::Settings;

/// MCP protocol version supported by this implementation
pub const MCP_VERSION: &str = "2024-11-05";

/// Server information
#[derive(Debug, Clone)]
pub struct ServerInfo {
    pub name: String,
    pub version: String,
}

impl Default for ServerInfo {
    fn default() -> Self {
        Self {
            name: env!("CARGO_PKG_NAME").to_string(),
            version: env!("CARGO_PKG_VERSION").to_string(),
        }
    }
}
```

## Configuration Module

```rust
// src/config/mod.rs
mod settings;

pub use settings::{Settings, ServerConfig, DatabaseConfig};
```

```rust
// src/config/settings.rs
use std::fs;
use serde::{Deserialize, Serialize};
use config::{Config, ConfigError, Environment, File};

/// Application settings
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Settings {
    pub server: ServerConfig,
    pub database: Option<DatabaseConfig>,
    pub tools: ToolsConfig,
    pub logging: LoggingConfig,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ServerConfig {
    pub name: String,
    pub version: String,
    pub host: String,
    pub port: u16,
    pub transport: TransportType,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DatabaseConfig {
    pub url: String,
    pub max_connections: u32,
    pub min_connections: u32,
    pub acquire_timeout: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ToolsConfig {
    pub enabled: Vec<String>,
    pub database_tool: Option<DatabaseToolConfig>,
    pub http_tool: Option<HttpToolConfig>,
    pub filesystem_tool: Option<FilesystemToolConfig>,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DatabaseToolConfig {
    pub read_only: bool,
    pub max_rows: u32,
    pub timeout: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct HttpToolConfig {
    pub allowed_domains: Option<Vec<String>>,
    pub blocked_domains: Option<Vec<String>>,
    pub timeout: u64,
    pub max_response_size: u64,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FilesystemToolConfig {
    pub allowed_paths: Vec<String>,
    pub max_file_size: u64,
    pub read_only: bool,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LoggingConfig {
    pub level: String,
    pub format: LogFormat,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum TransportType {
    Stdio,
    Http,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum LogFormat {
    Json,
    Pretty,
    Compact,
}

impl Settings {
    /// Load settings from configuration file
    pub fn load(config_path: &str) -> Result<Self, ConfigError> {
        let mut config = Config::builder()
            .add_source(File::with_name("config/default"))
            .add_source(File::with_name(config_path).required(false))
            .add_source(Environment::with_prefix("MCP").separator("_"))
            .build()?;
            
        // Set defaults
        config.set_default("server.name", env!("CARGO_PKG_NAME"))?;
        config.set_default("server.version", env!("CARGO_PKG_VERSION"))?;
        config.set_default("server.host", "0.0.0.0")?;
        config.set_default("server.port", 8000)?;
        config.set_default("server.transport", "http")?;
        config.set_default("logging.level", "info")?;
        config.set_default("logging.format", "pretty")?;
        config.set_default("tools.enabled", Vec::<String>::new())?;
        
        config.try_deserialize()
    }
    
    /// Validate configuration
    pub fn validate(&self) -> Result<(), String> {
        if self.server.port == 0 {
            return Err("Server port cannot be 0".to_string());
        }
        
        if let Some(db_config) = &self.database {
            if db_config.max_connections == 0 {
                return Err("Database max_connections cannot be 0".to_string());
            }
        }
        
        // Validate enabled tools exist
        for tool_name in &self.tools.enabled {
            match tool_name.as_str() {
                "database" | "http" | "filesystem" => {},
                _ => return Err(format!("Unknown tool: {}", tool_name)),
            }
        }
        
        Ok(())
    }
}

impl Default for Settings {
    fn default() -> Self {
        Self {
            server: ServerConfig {
                name: env!("CARGO_PKG_NAME").to_string(),
                version: env!("CARGO_PKG_VERSION").to_string(),
                host: "0.0.0.0".to_string(),
                port: 8000,
                transport: TransportType::Http,
            },
            database: None,
            tools: ToolsConfig {
                enabled: vec!["http".to_string(), "filesystem".to_string()],
                database_tool: None,
                http_tool: Some(HttpToolConfig {
                    allowed_domains: None,
                    blocked_domains: None,
                    timeout: 30,
                    max_response_size: 1024 * 1024, // 1MB
                }),
                filesystem_tool: Some(FilesystemToolConfig {
                    allowed_paths: vec!["/tmp".to_string()],
                    max_file_size: 1024 * 1024, // 1MB
                    read_only: true,
                }),
            },
            logging: LoggingConfig {
                level: "info".to_string(),
                format: LogFormat::Pretty,
            },
        }
    }
}
```

## Error Handling Module

```rust
// src/error/mod.rs
mod types;

pub use types::{Error, Result};
```

```rust
// src/error/types.rs
use std::fmt;
use thiserror::Error;

/// Application result type
pub type Result<T> = std::result::Result<T, Error>;

/// Application error types
#[derive(Error, Debug)]
pub enum Error {
    #[error("Configuration error: {0}")]
    Config(#[from] config::ConfigError),
    
    #[error("I/O error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("JSON error: {0}")]
    Json(#[from] serde_json::Error),
    
    #[error("HTTP error: {0}")]
    Http(#[from] reqwest::Error),
    
    #[cfg(feature = "database")]
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Validation error: {message}")]
    Validation { message: String },
    
    #[error("Tool error: {tool_name}: {message}")]
    Tool { tool_name: String, message: String },
    
    #[error("Resource error: {uri}: {message}")]
    Resource { uri: String, message: String },
    
    #[error("Permission denied: {message}")]
    Permission { message: String },
    
    #[error("Not found: {message}")]
    NotFound { message: String },
    
    #[error("Internal server error: {message}")]
    Internal { message: String },
}

impl Error {
    pub fn validation(message: impl Into<String>) -> Self {
        Self::Validation { message: message.into() }
    }
    
    pub fn tool_error(tool_name: impl Into<String>, message: impl Into<String>) -> Self {
        Self::Tool {
            tool_name: tool_name.into(),
            message: message.into(),
        }
    }
    
    pub fn resource_error(uri: impl Into<String>, message: impl Into<String>) -> Self {
        Self::Resource {
            uri: uri.into(),
            message: message.into(),
        }
    }
    
    pub fn permission_denied(message: impl Into<String>) -> Self {
        Self::Permission { message: message.into() }
    }
    
    pub fn not_found(message: impl Into<String>) -> Self {
        Self::NotFound { message: message.into() }
    }
    
    pub fn internal(message: impl Into<String>) -> Self {
        Self::Internal { message: message.into() }
    }
}
```

## Server Module Structure

```rust
// src/server/mod.rs
mod handlers;
mod transport;

pub use handlers::Handlers;
use transport::Transport;

use crate::{config::Settings, error::Result, tools::ToolRegistry, resources::ResourceRegistry};
use std::sync::Arc;
use tokio::signal;
use tracing::{info, error};

/// MCP Server
#[derive(Clone)]
pub struct Server {
    settings: Arc<Settings>,
    tool_registry: Arc<ToolRegistry>,
    resource_registry: Arc<ResourceRegistry>,
    handlers: Arc<Handlers>,
}

impl Server {
    /// Create a new server instance
    pub async fn new(settings: Settings) -> Result<Self> {
        // Validate settings
        settings.validate().map_err(|e| crate::Error::validation(e))?;
        
        let settings = Arc::new(settings);
        
        // Initialize registries
        let tool_registry = Arc::new(ToolRegistry::new(settings.clone()).await?);
        let resource_registry = Arc::new(ResourceRegistry::new(settings.clone()));
        
        // Initialize handlers
        let handlers = Arc::new(Handlers::new(
            tool_registry.clone(),
            resource_registry.clone(),
        ));
        
        Ok(Self {
            settings,
            tool_registry,
            resource_registry,
            handlers,
        })
    }
    
    /// Run the server
    pub async fn run(self) -> Result<()> {
        info!("Starting MCP server on {}:{}", 
               self.settings.server.host, 
               self.settings.server.port);
        
        // Create transport
        let transport = Transport::new(self.settings.clone(), self.handlers.clone()).await?;
        
        // Run server with graceful shutdown
        tokio::select! {
            result = transport.serve() => {
                match result {
                    Ok(_) => info!("Server stopped normally"),
                    Err(e) => {
                        error!("Server error: {}", e);
                        return Err(e);
                    }
                }
            },
            _ = signal::ctrl_c() => {
                info!("Received shutdown signal");
            }
        }
        
        // Cleanup
        self.tool_registry.shutdown().await;
        info!("Server shutdown complete");
        
        Ok(())
    }
}
```

This Rust project structure provides a solid foundation for building scalable and maintainable MCP servers with proper separation of concerns, configuration management, and error handling.