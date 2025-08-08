# Rust Testing Guide

## Testing Strategies for Rust MCP Servers

Comprehensive testing ensures your Rust MCP server is reliable, maintainable, and performs well under various conditions using Rust's powerful testing ecosystem.

## Testing Configuration

### Cargo.toml Test Dependencies
```toml
[dev-dependencies]
# Core testing framework
tokio-test = "0.4"
assert_matches = "1.5"
tempfile = "3.0"

# Mocking and test doubles
mockall = "0.11"
wiremock = "0.5"

# Property-based testing
quickcheck = "1.0"
quickcheck_macros = "1.0"

# Benchmarking
criterion = { version = "0.5", features = ["html_reports"] }

# Test utilities
rstest = "0.18"
test-case = "3.0"
fake = { version = "2.8", features = ["derive"] }

# HTTP testing
httpmock = "0.6"
reqwest = { version = "0.11", features = ["json"] }

# Database testing
sqlx = { version = "0.7", features = ["testing"] }

[features]
testing = []
```

### Test Utilities Module
```rust
// src/testing/mod.rs
use std::sync::Once;
use tracing_subscriber;

static INIT: Once = Once::new();

/// Initialize test logging (call once per test run)
pub fn init_test_logging() {
    INIT.call_once(|| {
        tracing_subscriber::fmt()
            .with_env_filter("debug")
            .with_test_writer()
            .init();
    });
}

/// Create temporary directory for tests
pub fn temp_dir() -> tempfile::TempDir {
    tempfile::tempdir().expect("Failed to create temp dir")
}

/// Create test configuration
pub fn test_config() -> crate::config::Settings {
    crate::config::Settings {
        server: crate::config::ServerConfig {
            name: "test-server".to_string(),
            version: "test".to_string(),
            host: "127.0.0.1".to_string(),
            port: 0, // Random available port
            transport: crate::config::TransportType::Http,
        },
        database: None,
        tools: crate::config::ToolsConfig {
            enabled: vec!["http".to_string(), "filesystem".to_string()],
            database_tool: None,
            http_tool: Some(crate::config::HttpToolConfig {
                allowed_domains: None,
                blocked_domains: None,
                timeout: 30,
                max_response_size: 1024 * 1024,
            }),
            filesystem_tool: Some(crate::config::FilesystemToolConfig {
                allowed_paths: vec!["/tmp".to_string()],
                max_file_size: 1024 * 1024,
                read_only: true,
            }),
        },
        logging: crate::config::LoggingConfig {
            level: "debug".to_string(),
            format: crate::config::LogFormat::Pretty,
        },
    }
}
```

## Unit Testing

### Tool Unit Tests
```rust
// src/tools/http/tests.rs
use super::*;
use crate::testing::*;
use httpmock::prelude::*;
use serde_json::json;
use std::collections::HashMap;
use tokio_test;

#[tokio::test]
async fn test_http_tool_get_request() {
    init_test_logging();
    
    // Setup mock server
    let server = MockServer::start();
    let mock = server.mock(|when, then| {
        when.method(GET)
            .path("/api/users");
        then.status(200)
            .header("content-type", "application/json")
            .body(r#"{"users": ["alice", "bob"]}"#);
    });
    
    // Create HTTP tool
    let config = crate::config::HttpToolConfig {
        allowed_domains: None,
        blocked_domains: None,
        timeout: 30,
        max_response_size: 1024 * 1024,
    };
    
    let tool = HttpTool::new(config).unwrap();
    
    // Prepare arguments
    let mut arguments = HashMap::new();
    arguments.insert("url".to_string(), json!(server.url("/api/users")));
    arguments.insert("method".to_string(), json!("GET"));
    
    // Execute tool
    let result = tool.execute(arguments).await.unwrap();
    
    // Verify result
    assert_eq!(result.content.len(), 1);
    
    if let Content::Text { text } = &result.content[0] {
        let response: serde_json::Value = serde_json::from_str(text).unwrap();
        assert_eq!(response["status_code"], 200);
        assert!(response["body"].as_str().unwrap().contains("alice"));
    } else {
        panic!("Expected text content");
    }
    
    mock.assert();
}

#[tokio::test]
async fn test_http_tool_post_request() {
    init_test_logging();
    
    let server = MockServer::start();
    let mock = server.mock(|when, then| {
        when.method(POST)
            .path("/api/users")
            .header("content-type", "application/json")
            .json_body(json!({"name": "charlie"}));
        then.status(201)
            .header("content-type", "application/json")
            .body(r#"{"id": 123, "name": "charlie"}"#);
    });
    
    let config = crate::config::HttpToolConfig {
        allowed_domains: None,
        blocked_domains: None,
        timeout: 30,
        max_response_size: 1024 * 1024,
    };
    
    let tool = HttpTool::new(config).unwrap();
    
    let mut arguments = HashMap::new();
    arguments.insert("url".to_string(), json!(server.url("/api/users")));
    arguments.insert("method".to_string(), json!("POST"));
    arguments.insert("body".to_string(), json!(r#"{"name": "charlie"}"#));
    
    let mut headers = HashMap::new();
    headers.insert("content-type".to_string(), json!("application/json"));
    arguments.insert("headers".to_string(), json!(headers));
    
    let result = tool.execute(arguments).await.unwrap();
    
    assert_eq!(result.content.len(), 1);
    if let Content::Text { text } = &result.content[0] {
        let response: serde_json::Value = serde_json::from_str(text).unwrap();
        assert_eq!(response["status_code"], 201);
    }
    
    mock.assert();
}

#[tokio::test]
async fn test_http_tool_blocked_domain() {
    init_test_logging();
    
    let config = crate::config::HttpToolConfig {
        allowed_domains: None,
        blocked_domains: Some(vec!["blocked.com".to_string()]),
        timeout: 30,
        max_response_size: 1024 * 1024,
    };
    
    let tool = HttpTool::new(config).unwrap();
    
    let mut arguments = HashMap::new();
    arguments.insert("url".to_string(), json!("https://blocked.com/api"));
    
    let result = tool.validate_arguments(&arguments);
    
    assert!(result.is_err());
    assert!(result.unwrap_err().to_string().contains("blocked"));
}

// Property-based tests
#[quickcheck]
fn test_http_tool_url_validation(url: String) -> bool {
    let config = crate::config::HttpToolConfig {
        allowed_domains: None,
        blocked_domains: None,
        timeout: 30,
        max_response_size: 1024 * 1024,
    };
    
    let tool = HttpTool::new(config).unwrap();
    
    // Valid URLs should not cause panics
    let _ = tool.validate_url(&url);
    true
}

// Parametrized tests using rstest
#[rstest]
#[case("GET", "https://api.example.com/users")]
#[case("POST", "https://api.example.com/users")]
#[case("PUT", "https://api.example.com/users/123")]
#[case("DELETE", "https://api.example.com/users/123")]
#[tokio::test]
async fn test_http_methods(#[case] method: &str, #[case] url: &str) {
    let config = crate::config::HttpToolConfig {
        allowed_domains: None,
        blocked_domains: None,
        timeout: 30,
        max_response_size: 1024 * 1024,
    };
    
    let tool = HttpTool::new(config).unwrap();
    
    let mut arguments = HashMap::new();
    arguments.insert("method".to_string(), json!(method));
    arguments.insert("url".to_string(), json!(url));
    
    let validation_result = tool.validate_arguments(&arguments);
    assert!(validation_result.is_ok());
}
```

### Database Tool Tests with Mocking
```rust
// src/tools/database/tests.rs
use super::*;
use crate::testing::*;
use mockall::predicate::*;
use sqlx::testing::TestPool;
use std::collections::HashMap;

// Mock trait for database operations
mockall::mock! {
    DatabasePool {
        async fn query(&self, query: &str) -> Result<Vec<sqlx::Row>, sqlx::Error>;
    }
}

#[tokio::test]
async fn test_database_tool_select_query() {
    init_test_logging();
    
    // This would typically use sqlx::testing::TestPool in real scenarios
    let config = crate::config::DatabaseToolConfig {
        read_only: true,
        max_rows: 100,
        timeout: 30,
    };
    
    // For this test, we'll test validation logic
    let tool_result = std::panic::catch_unwind(|| {
        // Tool creation would happen here with real database
        // For now, we test validation logic directly
        let query = "SELECT id, name FROM users";
        validate_query(query, &config)
    });
    
    assert!(tool_result.is_ok());
}

#[tokio::test]
async fn test_database_tool_prevents_dangerous_queries() {
    let config = crate::config::DatabaseToolConfig {
        read_only: true,
        max_rows: 100,
        timeout: 30,
    };
    
    let dangerous_queries = vec![
        "DROP TABLE users",
        "DELETE FROM users",
        "UPDATE users SET password = 'hacked'",
        "INSERT INTO users VALUES ('hacker', 'password')",
    ];
    
    for query in dangerous_queries {
        let result = validate_query(query, &config);
        assert!(result.is_err(), "Query should be rejected: {}", query);
    }
}

fn validate_query(query: &str, config: &crate::config::DatabaseToolConfig) -> Result<()> {
    let query_upper = query.trim().to_uppercase();
    
    if config.read_only && !query_upper.starts_with("SELECT") {
        return Err(crate::Error::permission_denied(
            "Only SELECT statements are allowed in read-only mode"
        ));
    }
    
    let dangerous_keywords = [
        "DROP", "DELETE", "INSERT", "UPDATE", "ALTER", 
        "CREATE", "TRUNCATE", "REPLACE", "MERGE"
    ];
    
    for keyword in &dangerous_keywords {
        if query_upper.contains(keyword) && config.read_only {
            return Err(crate::Error::permission_denied(
                format!("Query contains forbidden keyword: {}", keyword)
            ));
        }
    }
    
    Ok(())
}

// Integration test with real database (requires test database)
#[cfg(feature = "testing")]
#[sqlx::test]
async fn test_database_tool_integration(pool: sqlx::PgPool) {
    // Setup test data
    sqlx::query("CREATE TEMPORARY TABLE test_users (id SERIAL PRIMARY KEY, name TEXT)")
        .execute(&pool)
        .await
        .unwrap();
    
    sqlx::query("INSERT INTO test_users (name) VALUES ('Alice'), ('Bob')")
        .execute(&pool)
        .await
        .unwrap();
    
    // Create tool
    let config = crate::config::DatabaseToolConfig {
        read_only: true,
        max_rows: 100,
        timeout: 30,
    };
    
    let tool = DatabaseTool::new("postgresql://test", config).await.unwrap();
    
    // Test query
    let mut arguments = HashMap::new();
    arguments.insert("query".to_string(), json!("SELECT * FROM test_users"));
    
    let result = tool.execute(arguments).await.unwrap();
    
    assert_eq!(result.content.len(), 1);
    if let Content::Text { text } = &result.content[0] {
        assert!(text.contains("Alice"));
        assert!(text.contains("Bob"));
    }
}
```

### Filesystem Tool Tests
```rust
// src/tools/filesystem/tests.rs
use super::*;
use crate::testing::*;
use std::fs;
use std::path::Path;
use tempfile::TempDir;
use tokio_test;

#[tokio::test]
async fn test_filesystem_tool_read_file() {
    init_test_logging();
    
    let temp_dir = temp_dir();
    let test_file = temp_dir.path().join("test.txt");
    let test_content = "Hello, World!";
    
    // Create test file
    fs::write(&test_file, test_content).unwrap();
    
    let config = crate::config::FilesystemToolConfig {
        allowed_paths: vec![temp_dir.path().to_string_lossy().into_owned()],
        max_file_size: 1024 * 1024,
        read_only: true,
    };
    
    let tool = FilesystemTool::new(config);
    
    let mut arguments = HashMap::new();
    arguments.insert("operation".to_string(), json!("read_file"));
    arguments.insert("path".to_string(), json!(test_file.to_string_lossy()));
    
    let result = tool.execute(arguments).await.unwrap();
    
    assert_eq!(result.content.len(), 1);
    if let Content::Text { text } = &result.content[0] {
        let response: serde_json::Value = serde_json::from_str(text).unwrap();
        assert_eq!(response["content"], test_content);
    }
}

#[tokio::test]
async fn test_filesystem_tool_list_directory() {
    init_test_logging();
    
    let temp_dir = temp_dir();
    
    // Create test files
    fs::write(temp_dir.path().join("file1.txt"), "content1").unwrap();
    fs::write(temp_dir.path().join("file2.txt"), "content2").unwrap();
    fs::create_dir(temp_dir.path().join("subdir")).unwrap();
    
    let config = crate::config::FilesystemToolConfig {
        allowed_paths: vec![temp_dir.path().to_string_lossy().into_owned()],
        max_file_size: 1024 * 1024,
        read_only: true,
    };
    
    let tool = FilesystemTool::new(config);
    
    let mut arguments = HashMap::new();
    arguments.insert("operation".to_string(), json!("list_directory"));
    arguments.insert("path".to_string(), json!(temp_dir.path().to_string_lossy()));
    
    let result = tool.execute(arguments).await.unwrap();
    
    assert_eq!(result.content.len(), 1);
    if let Content::Text { text } = &result.content[0] {
        let response: serde_json::Value = serde_json::from_str(text).unwrap();
        let entries = response["entries"].as_array().unwrap();
        assert_eq!(entries.len(), 3); // 2 files + 1 directory
    }
}

#[tokio::test]
async fn test_filesystem_tool_path_traversal_protection() {
    let config = crate::config::FilesystemToolConfig {
        allowed_paths: vec!["/tmp".to_string()],
        max_file_size: 1024 * 1024,
        read_only: true,
    };
    
    let tool = FilesystemTool::new(config);
    
    let mut arguments = HashMap::new();
    arguments.insert("operation".to_string(), json!("read_file"));
    arguments.insert("path".to_string(), json!("../../../etc/passwd"));
    
    let result = tool.validate_arguments(&arguments);
    assert!(result.is_err());
    assert!(result.unwrap_err().to_string().contains("traversal"));
}

#[test_case("/tmp/allowed.txt"; "allowed path")]
#[test_case("/tmp/subdir/allowed.txt"; "allowed subdirectory")]
fn test_filesystem_path_validation_allowed(path: &str) {
    let config = crate::config::FilesystemToolConfig {
        allowed_paths: vec!["/tmp".to_string()],
        max_file_size: 1024 * 1024,
        read_only: true,
    };
    
    let tool = FilesystemTool::new(config);
    
    // Note: This test would need actual file creation in real scenario
    // For unit testing, we'd extract the validation logic
    let validation_result = validate_path_logic(path, &["/tmp"]);
    assert!(validation_result.is_ok());
}

// Helper function for testing path validation logic
fn validate_path_logic(path: &str, allowed_paths: &[&str]) -> Result<()> {
    if path.contains("..") {
        return Err(crate::Error::permission_denied("Path traversal not allowed"));
    }
    
    let path_obj = Path::new(path);
    
    for allowed_path in allowed_paths {
        let allowed_obj = Path::new(allowed_path);
        if path_obj.starts_with(allowed_obj) {
            return Ok(());
        }
    }
    
    Err(crate::Error::permission_denied(
        format!("Path not in allowed directories: {}", path)
    ))
}
```

## Integration Testing

### Server Integration Tests
```rust
// tests/integration/server_tests.rs
use mcp_server_rust::{Server, config::Settings, testing::*};
use reqwest;
use serde_json::json;
use std::time::Duration;
use tokio::time::sleep;

#[tokio::test]
async fn test_server_startup_and_health() {
    init_test_logging();
    
    let config = test_config();
    let server = Server::new(config).await.unwrap();
    
    // Start server in background
    let server_handle = tokio::spawn(async move {
        server.run().await
    });
    
    // Give server time to start
    sleep(Duration::from_millis(100)).await;
    
    // Test health endpoint
    let client = reqwest::Client::new();
    let response = client
        .get("http://127.0.0.1:8000/health")
        .send()
        .await
        .unwrap();
    
    assert!(response.status().is_success());
    
    // Cleanup
    server_handle.abort();
}

#[tokio::test]
async fn test_tool_listing_endpoint() {
    init_test_logging();
    
    let mut config = test_config();
    config.server.port = 0; // Use random port
    
    let server = Server::new(config).await.unwrap();
    let server_handle = tokio::spawn(async move {
        server.run().await
    });
    
    sleep(Duration::from_millis(100)).await;
    
    let client = reqwest::Client::new();
    let response = client
        .post("http://127.0.0.1:8000/")
        .json(&json!({
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/list",
            "params": {}
        }))
        .send()
        .await
        .unwrap();
    
    assert!(response.status().is_success());
    
    let body: serde_json::Value = response.json().await.unwrap();
    assert_eq!(body["jsonrpc"], "2.0");
    assert_eq!(body["id"], 1);
    assert!(body["result"]["tools"].is_array());
    
    server_handle.abort();
}
```

## Performance Testing and Benchmarks

### Benchmark Configuration
```rust
// benches/tool_benchmarks.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use mcp_server_rust::{tools::*, config::*};
use std::collections::HashMap;
use serde_json::json;
use tokio::runtime::Runtime;

fn benchmark_http_tool(c: &mut Criterion) {
    let rt = Runtime::new().unwrap();
    
    let config = HttpToolConfig {
        allowed_domains: None,
        blocked_domains: None,
        timeout: 30,
        max_response_size: 1024 * 1024,
    };
    
    let tool = HttpTool::new(config).unwrap();
    
    c.bench_function("http_tool_validation", |b| {
        b.iter(|| {
            let mut arguments = HashMap::new();
            arguments.insert("url".to_string(), json!("https://httpbin.org/get"));
            arguments.insert("method".to_string(), json!("GET"));
            
            black_box(tool.validate_arguments(&arguments))
        });
    });
    
    c.bench_function("http_tool_execute", |b| {
        b.to_async(&rt).iter(|| async {
            let mut arguments = HashMap::new();
            arguments.insert("url".to_string(), json!("https://httpbin.org/get"));
            arguments.insert("method".to_string(), json!("GET"));
            
            black_box(tool.execute(arguments).await)
        });
    });
}

fn benchmark_filesystem_tool(c: &mut Criterion) {
    let rt = Runtime::new().unwrap();
    
    let temp_dir = tempfile::tempdir().unwrap();
    let test_file = temp_dir.path().join("benchmark.txt");
    std::fs::write(&test_file, "benchmark content".repeat(1000)).unwrap();
    
    let config = FilesystemToolConfig {
        allowed_paths: vec![temp_dir.path().to_string_lossy().into_owned()],
        max_file_size: 1024 * 1024,
        read_only: true,
    };
    
    let tool = FilesystemTool::new(config);
    
    c.bench_function("filesystem_tool_read", |b| {
        b.to_async(&rt).iter(|| async {
            let mut arguments = HashMap::new();
            arguments.insert("operation".to_string(), json!("read_file"));
            arguments.insert("path".to_string(), json!(test_file.to_string_lossy()));
            
            black_box(tool.execute(arguments).await)
        });
    });
}

criterion_group!(benches, benchmark_http_tool, benchmark_filesystem_tool);
criterion_main!(benches);
```

## Test Organization

### Test Directory Structure
```
tests/
├── integration/
│   ├── mod.rs
│   ├── server_tests.rs
│   ├── tool_integration_tests.rs
│   └── resource_tests.rs
├── common/
│   ├── mod.rs
│   ├── fixtures.rs
│   └── test_data.rs
└── e2e/
    ├── mod.rs
    └── full_workflow_tests.rs
```

### Test Makefile Targets
```makefile
# Makefile
.PHONY: test test-unit test-integration test-bench test-coverage

# Run all tests
test: test-unit test-integration

# Run unit tests only
test-unit:
	cargo test --lib

# Run integration tests
test-integration:
	cargo test --test '*'

# Run benchmarks
test-bench:
	cargo bench

# Generate coverage report
test-coverage:
	cargo tarpaulin --out html --output-dir target/coverage

# Run tests with specific features
test-database:
	cargo test --features database

# Run property-based tests
test-property:
	cargo test --features quickcheck

# Clean test artifacts
clean-test:
	cargo clean
	rm -rf target/coverage
```

## Test Configuration Files

### CI/CD Testing
```yaml
# .github/workflows/test.yml
name: Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    name: Test Suite
    runs-on: ubuntu-latest
    strategy:
      matrix:
        rust: [stable, beta, nightly]

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
    - uses: actions/checkout@v4

    - name: Install Rust
      uses: dtolnay/rust-toolchain@master
      with:
        toolchain: ${{ matrix.rust }}
        components: rustfmt, clippy

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/registry
          ~/.cargo/git
          target
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

    - name: Run tests
      run: cargo test --all-features
      env:
        DATABASE_URL: postgres://postgres:postgres@localhost/test_db

    - name: Run clippy
      run: cargo clippy -- -D warnings

    - name: Run rustfmt
      run: cargo fmt -- --check

    - name: Generate coverage
      run: |
        cargo install tarpaulin
        cargo tarpaulin --all-features --out xml

    - name: Upload coverage
      uses: codecov/codecov-action@v3
      with:
        file: cobertura.xml
```

This comprehensive Rust testing guide ensures your MCP server is thoroughly tested with unit tests, integration tests, property-based tests, and performance benchmarks, following Rust best practices and leveraging the ecosystem's powerful testing tools.