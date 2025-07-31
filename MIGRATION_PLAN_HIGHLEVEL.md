# Claude Code SDK - Highlevel Migration Plan

## 🚀 Phase 0: Project Setup & Foundation

This initial phase establishes the project structure, dependencies, and core data types, which are the building blocks for the entire SDK.

### 1\. Initialize Cargo Project

Create a new Rust library project:

```bash
cargo new claude-code-sdk-rs --lib
```

### 2\. Configure `Cargo.toml`

Add the necessary dependencies. The initial setup should include:

```toml
[package]
name = "claude-code-sdk"
version = "0.1.0"
edition = "2021"
authors = ["Anthropic <support@anthropic.com>"]
description = "Rust SDK for Claude Code"
license = "MIT"
repository = "https://github.com/your-repo/claude-code-sdk-rs" # Placeholder
keywords = ["claude", "ai", "sdk", "anthropic"]
categories = ["api-bindings", "development-tools"]

[dependencies]
# Async runtime and utilities
tokio = { version = "1", features = ["full"] }
tokio-stream = "0.1"
futures = "0.3"

# Serialization and Deserialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

# Error Handling
thiserror = "1.0"

# Logging and Diagnostics
tracing = "0.1"

# Utility for finding the CLI executable
which = "6.0"

[dev-dependencies]
# For running tests and examples
anyhow = "1.0"

# A testing framework similar to pytest
rstest = "0.21"
```

### 3\. Define Core Types (`src/types.rs`)

Translate the Python dataclasses from `types.py` into Rust structs. Use `serde` to make them serializable and deserializable.

  * **Message Types:** `UserMessage`, `AssistantMessage`, `SystemMessage`, `ResultMessage`.
  * **Content Blocks:** `TextBlock`, `ToolUseBlock`, `ToolResultBlock`. Create an enum `ContentBlock` to represent these variants.
  * **Configuration:** Create a `ClaudeCodeOptions` struct. Implement the **builder pattern** for ergonomic construction, as it's more idiomatic in Rust than a struct with many public optional fields.

**Example: `ResultMessage`**

```rust
// src/types.rs
use serde::{Deserialize, Serialize};
use std::collections::HashMap;

#[derive(Debug, Clone, PartialEq, Deserialize, Serialize)]
pub struct ResultMessage {
    pub subtype: String,
    pub duration_ms: i64,
    pub duration_api_ms: i64,
    pub is_error: bool,
    pub num_turns: i32,
    pub session_id: String,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub total_cost_usd: Option<f64>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub usage: Option<HashMap<String, serde_json::Value>>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub result: Option<String>,
}
```

### 4\. Define Custom Errors (`src/errors.rs`)

Create a dedicated error enum using `thiserror` to represent all possible failure modes, mirroring `_errors.py`.

**Example: `SdkError`**

```rust
// src/errors.rs
use thiserror::Error;

#[derive(Error, Debug)]
pub enum SdkError {
    #[error("Claude Code CLI not found. Please ensure it is installed and in your PATH.")]
    CliNotFound(#[from] which::Error),

    #[error("Failed to connect to or start the Claude Code CLI process.")]
    CliConnection(#[from] std::io::Error),

    #[error("The CLI process failed with exit code {exit_code:?}: {stderr}")]
    Process {
        exit_code: Option<i32>,
        stderr: String,
    },

    #[error("Failed to decode JSON from CLI output: {0}")]
    JsonDecode(#[from] serde_json::Error),

    #[error("Failed to parse message from data: {message}")]
    MessageParse {
        message: String,
        data: serde_json::Value,
    },
}
```

-----

## ⚙️ Phase 1: Core Transport Layer

This phase focuses on the low-level interaction with the `claude-code` Node.js subprocess. The goal is to create a reliable mechanism for spawning, managing, and communicating with the CLI.

### 1\. Create Transport Module (`src/transport.rs`)

This module will house the `SubprocessCliTransport` struct.

### 2\. Implement CLI Discovery

In the `SubprocessCliTransport::new()`, replicate the logic from `_find_cli` using the `which` crate to locate the `claude` executable.

### 3\. Implement Process Management

  * **Spawning:** Use `tokio::process::Command` to spawn the `claude` CLI.
  * **Command Building:** Create a private `build_command` method that constructs the command arguments from the `ClaudeCodeOptions` builder.
  * **I/O Handling:** Configure the command's `stdin`, `stdout`, and `stderr` to be `piped`. Store the resulting `ChildStdin`, `ChildStdout`, and `ChildStderr` handles.

### 4\. Implement `connect` and `disconnect`

  * `async fn connect(&mut self)` will execute the `Command` and set up the I/O stream readers/writers.
  * `async fn disconnect(&mut self)` will terminate the child process. Implement the `Drop` trait on the transport to ensure the child process is cleaned up automatically if the transport goes out of scope.

-----

## 📨 Phase 2: Message Handling & Parsing

With the transport layer in place, this phase handles the stream of data coming from the CLI.

### 1\. Implement Message Streaming

The `SubprocessCliTransport` will expose an `async fn receive_messages(&mut self)` method.

  * **Return Type:** This function will return a `impl Stream<Item = Result<Message, SdkError>>`. The `tokio-stream` crate will be essential here.
  * **Parsing Logic:**
    1.  Read from the `ChildStdout` stream line by line.
    2.  Implement robust buffering to handle JSON objects that are split across multiple reads or concatenated into a single read. This is a critical feature from the Python SDK (`test_subprocess_buffering.py`).
    3.  For each complete JSON string, parse it using `serde_json::from_str`.
    4.  Map the raw `serde_json::Value` to the strongly-typed Rust `Message` structs defined in Phase 0.
  * **Error Handling:** Any I/O or parsing errors should be propagated as variants of the `SdkError` enum.

### 2\. Implement Stderr Handling

Read the `stderr` stream concurrently to capture any error messages from the CLI. If the process exits with a non-zero code, include the captured `stderr` content in the `SdkError::Process` error.

-----

## 🧑‍💻 Phase 3: High-Level Client & API

This phase builds the public-facing API that developers will use. The goal is to create an ergonomic and idiomatic Rust interface.

### 1\. One-Shot `query` Function (`src/lib.rs`)

Create a top-level async function similar to the Python version for simple, stateless use cases.

```rust
// src/lib.rs
use crate::types::{ClaudeCodeOptions, Message};
use crate::errors::SdkError;
use tokio_stream::Stream;

pub async fn query(
    prompt: String,
    options: ClaudeCodeOptions,
) -> impl Stream<Item = Result<Message, SdkError>> {
    // 1. Create and configure transport
    // 2. Connect
    // 3. Return the message stream
    // The transport will be dropped and cleaned up when the stream is fully consumed.
    // ... implementation ...
}
```

### 2\. Interactive `ClaudeSdkClient` (`src/client.rs`)

Create the `ClaudeSdkClient` struct for stateful, bidirectional conversations.

  * **State:** The client will own the `SubprocessCliTransport`.
  * **Methods:**
      * `new(options: ClaudeCodeOptions) -> Self`
      * `connect(&mut self, prompt: Option<impl Stream<Item = Message>>)`
      * `disconnect(&mut self)`
      * `query(&mut self, prompt: String)`: Sends a message to the running process's `stdin`.
      * `interrupt(&mut self)`: Sends a special control message to `stdin`.
      * `receive_messages(&mut self) -> impl Stream<Item = Result<Message, SdkError>>`: Streams all incoming messages.
      * `receive_response(&mut self) -> impl Stream<Item = Result<Message, SdkError>>`: A convenience wrapper that streams messages until a `ResultMessage` is received.

-----

## ✅ Phase 4: Testing

A robust test suite is crucial for a reliable SDK. The goal is to port the logic from the Python tests to ensure all features and edge cases are covered.

### 1\. Unit Tests

Place unit tests within their respective modules (e.g., `#[cfg(test)] mod tests { ... }` at the bottom of `types.rs`, `transport.rs`, etc.).

  * Test the `ClaudeCodeOptions` builder.
  * Test the serialization and deserialization of all `Message` types.

### 2\. Integration Tests (`tests/`)

Create a `tests` directory next to `src/`.

  * **Mock CLI:** Create a simple script (Python or shell script) that mimics the `claude-code` CLI's behavior. This script can be configured to produce specific stdout/stderr output, including split JSON and error conditions.
  * **Test Cases:**
      * `test_client_lifecycle.rs`: Verify `connect`, `disconnect`, and `query` behavior.
      * `test_streaming.rs`: Test `receive_messages` and `receive_response`, especially the buffering logic for split/merged JSON.
      * `test_errors.rs`: Test all `SdkError` variants by having the mock CLI produce corresponding error conditions.
      * `test_query_function.rs`: Test the top-level `query` function.

-----

## 📚 Phase 5: Documentation & Examples

Clear documentation and examples are essential for adoption.

### 1\. Code Documentation

Add comprehensive doc comments (`///`) to all public structs, enums, functions, and methods. Explain what each item does, its parameters, and what it returns. Use `#[doc(hidden)]` for internal-only components.

### 2\. `README.md`

Adapt the Python `README.md` for a Rust audience. Include:

  * Installation instructions (`cargo add claude-code-sdk`).
  * Prerequisites (Node.js and `claude-code` CLI).
  * A "Quick Start" example.
  * Detailed usage examples for both the `query` function and the `ClaudeSdkClient`.
  * A section on error handling using `Result` and `match`.

### 3\. Examples (`examples/`)

Create an `examples` directory and add runnable examples that mirror the Python SDK's examples, such as `quick_start.rs` and `streaming_mode.rs`. Use `tokio`'s `main` macro.

**Example: `quick_start.rs`**

```rust
// examples/quick_start.rs
use claude_code_sdk::{query, ClaudeCodeOptions};
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let options = ClaudeCodeOptions::builder().build(); // Using the builder
    let mut stream = query("What is 2 + 2?".to_string(), options).await;

    while let Some(message_result) = stream.next().await {
        match message_result {
            Ok(message) => println!("{:?}", message),
            Err(e) => eprintln!("Error: {}", e),
        }
    }

    Ok(())
}
```

-----

\#\#📦 Phase 6: Packaging & Release

The final step is to prepare the crate for publishing.

### 1\. Finalize `Cargo.toml`

Ensure all metadata fields are correct and complete. Set a `0.1.0` version for the initial release.

### 2\. Run Final Checks

  * `cargo fmt` to ensure consistent formatting.
  * `cargo clippy` to catch common mistakes and improve the code.
  * `cargo test` to run all tests.
  * `cargo doc --open` to review the generated documentation.

### 3\. Publish to Crates.io

```bash
# Perform a dry run first to check for any packaging issues
cargo publish --dry-run

# If the dry run is successful, publish the crate
cargo publish
```