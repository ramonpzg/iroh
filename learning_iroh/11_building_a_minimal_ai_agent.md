# Part 11: Building a Minimal AI Agent

Stop installing SDKs. You're going to build an AI agent with direct HTTP calls to the Anthropic API. Zero abstraction layers. Five dependencies. 300 lines of code. It will search the web, execute Python, and respond intelligently to your commands.

## Why Minimal Dependencies

The typical Python AI agent:

```bash
$ pip install anthropic openai langchain chromadb numpy pandas requests beautifulsoup4
$ du -sh venv/
156M    venv/
```

156MB of dependencies to make HTTP calls and parse JSON. Plus the startup time (1-2 seconds on a fast machine), the security surface (every dependency is a potential vulnerability), and the compile-time cost (for Rust, every dependency adds to build time).

Our Rust agent:

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json", "stream"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
```

Five dependencies. Binary size: ~8MB (release mode, stripped). Startup time: ~10ms. All dependencies are widely used, well-audited crates.

Python comparison: You could use just `requests` and `json`, but you'd lose type safety, async, and the benefits of Rust's ownership model for long-running agents.

## The Anthropic API (Unfiltered)

Anthropic's Messages API is HTTP POST with JSON. No magic.

```bash
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-3-5-haiku-20241022",
    "max_tokens": 1024,
    "messages": [
      {"role": "user", "content": "Hello, Claude"}
    ]
  }'
```

Response:

```json
{
  "id": "msg_01XYZ...",
  "type": "message",
  "role": "assistant",
  "content": [
    {"type": "text", "text": "Hello! How can I help you today?"}
  ],
  "model": "claude-3-5-haiku-20241022",
  "stop_reason": "end_turn",
  "usage": {"input_tokens": 10, "output_tokens": 12}
}
```

That's it. The SDK is just a wrapper around this.

## Project Setup

```bash
cargo new --bin ai-agent
cd ai-agent
```

Add to `Cargo.toml`:

```toml
[dependencies]
reqwest = { version = "0.12", features = ["json", "stream"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
```

## The Core Types

```rust
// src/api.rs
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize, Clone)]
pub struct Message {
    pub role: String,
    pub content: String,
}

#[derive(Debug, Serialize)]
pub struct ApiRequest {
    pub model: String,
    pub max_tokens: u32,
    pub messages: Vec<Message>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub tools: Option<Vec<Tool>>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub stream: Option<bool>,
}

#[derive(Debug, Deserialize)]
pub struct ApiResponse {
    pub id: String,
    pub content: Vec<ContentBlock>,
    pub stop_reason: Option<String>,
    pub usage: Usage,
}

#[derive(Debug, Deserialize, Clone)]
#[serde(tag = "type")]
pub enum ContentBlock {
    #[serde(rename = "text")]
    Text { text: String },
    #[serde(rename = "tool_use")]
    ToolUse {
        id: String,
        name: String,
        input: serde_json::Value,
    },
}

#[derive(Debug, Serialize)]
pub struct Tool {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,
}

#[derive(Debug, Deserialize)]
pub struct Usage {
    pub input_tokens: u32,
    pub output_tokens: u32,
}
```

These types map exactly to Anthropic's JSON schema. `#[serde(tag = "type")]` handles the discriminated union for content blocks (text vs tool_use).

Python comparison:

```python
from typing import Literal, Union
from dataclasses import dataclass

@dataclass
class Message:
    role: str
    content: str

# But at runtime, nothing enforces this.
# You can pass {"role": 123, "content": None} and Python won't complain until it fails.
```

Rust's types are checked at compile time. If you construct an invalid `ApiRequest`, the code won't compile.

## Making the API Call

```rust
// src/api.rs (continued)
use anyhow::{Context, Result};
use reqwest::Client;

const API_URL: &str = "https://api.anthropic.com/v1/messages";
const API_VERSION: &str = "2023-06-01";

pub struct AnthropicClient {
    client: Client,
    api_key: String,
}

impl AnthropicClient {
    pub fn new(api_key: String) -> Self {
        Self {
            client: Client::new(),
            api_key,
        }
    }

    pub async fn send_message(&self, request: ApiRequest) -> Result<ApiResponse> {
        let response = self
            .client
            .post(API_URL)
            .header("x-api-key", &self.api_key)
            .header("anthropic-version", API_VERSION)
            .header("content-type", "application/json")
            .json(&request)
            .send()
            .await
            .context("Failed to send request to Anthropic API")?;

        if !response.status().is_success() {
            let status = response.status();
            let body = response.text().await?;
            anyhow::bail!("API error {}: {}", status, body);
        }

        let api_response: ApiResponse = response
            .json()
            .await
            .context("Failed to parse API response")?;

        Ok(api_response)
    }
}
```

Error handling: If the API returns non-2xx, we bail with the status and body. If JSON parsing fails, we bail with context.

## The First Agent Loop

```rust
// src/main.rs
mod api;

use api::{AnthropicClient, ApiRequest, Message};
use anyhow::Result;
use std::io::{self, Write};

#[tokio::main]
async fn main() -> Result<()> {
    let api_key = std::env::var("ANTHROPIC_API_KEY")
        .expect("ANTHROPIC_API_KEY environment variable not set");

    let client = AnthropicClient::new(api_key);
    let mut conversation: Vec<Message> = Vec::new();

    println!("AI Agent (type 'exit' to quit)");
    println!();

    loop {
        print!("> ");
        io::stdout().flush()?;

        let mut input = String::new();
        io::stdin().read_line(&mut input)?;
        let input = input.trim();

        if input.is_empty() {
            continue;
        }

        if input == "exit" {
            break;
        }

        conversation.push(Message {
            role: "user".to_string(),
            content: input.to_string(),
        });

        let request = ApiRequest {
            model: "claude-3-5-haiku-20241022".to_string(),
            max_tokens: 1024,
            messages: conversation.clone(),
            tools: None,
            stream: None,
        };

        let response = client.send_message(request).await?;

        let assistant_message = response
            .content
            .iter()
            .filter_map(|block| match block {
                api::ContentBlock::Text { text } => Some(text.clone()),
                _ => None,
            })
            .collect::<Vec<_>>()
            .join("\n");

        println!("\n{}\n", assistant_message);

        conversation.push(Message {
            role: "assistant".to_string(),
            content: assistant_message,
        });

        println!(
            "(tokens: {} in, {} out)",
            response.usage.input_tokens, response.usage.output_tokens
        );
    }

    Ok(())
}
```

Run it:

```bash
export ANTHROPIC_API_KEY=your-key-here
cargo run
```

```
AI Agent (type 'exit' to quit)

> What is the capital of France?

Paris is the capital of France.

(tokens: 15 in, 8 out)
>
```

You just built a conversational AI agent. No SDK. 87 lines.

## Adding Streaming (Real-Time Responses)

The API supports Server-Sent Events (SSE) for streaming. Set `"stream": true` in the request, then parse the event stream.

```rust
// src/api.rs (add this method to AnthropicClient)
use futures::stream::StreamExt;

pub async fn send_message_stream(
    &self,
    request: ApiRequest,
    mut on_chunk: impl FnMut(&str),
) -> Result<ApiResponse> {
    let mut request = request;
    request.stream = Some(true);

    let response = self
        .client
        .post(API_URL)
        .header("x-api-key", &self.api_key)
        .header("anthropic-version", API_VERSION)
        .header("content-type", "application/json")
        .json(&request)
        .send()
        .await
        .context("Failed to send request")?;

    if !response.status().is_success() {
        let status = response.status();
        let body = response.text().await?;
        anyhow::bail!("API error {}: {}", status, body);
    }

    let mut stream = response.bytes_stream();
    let mut buffer = String::new();
    let mut full_text = String::new();
    let mut message_id = String::new();

    while let Some(chunk) = stream.next().await {
        let chunk = chunk.context("Failed to read chunk")?;
        buffer.push_str(&String::from_utf8_lossy(&chunk));

        // Parse SSE format: "data: {json}\n\n"
        while let Some(pos) = buffer.find("\n\n") {
            let event = &buffer[..pos];
            buffer = buffer[pos + 2..].to_string();

            if let Some(data) = event.strip_prefix("data: ") {
                if data == "[DONE]" {
                    break;
                }

                if let Ok(event_data) = serde_json::from_str::<serde_json::Value>(data) {
                    if event_data["type"] == "content_block_delta" {
                        if let Some(text) = event_data["delta"]["text"].as_str() {
                            full_text.push_str(text);
                            on_chunk(text);
                        }
                    }
                    if event_data["type"] == "message_start" {
                        if let Some(id) = event_data["message"]["id"].as_str() {
                            message_id = id.to_string();
                        }
                    }
                }
            }
        }
    }

    // Construct a response compatible with non-streaming
    Ok(ApiResponse {
        id: message_id,
        content: vec![api::ContentBlock::Text { text: full_text }],
        stop_reason: Some("end_turn".to_string()),
        usage: Usage {
            input_tokens: 0,
            output_tokens: 0,
        },
    })
}
```

Update main to use streaming:

```rust
// In the main loop:
print!("\n");
io::stdout().flush()?;

let response = client
    .send_message_stream(request, |chunk| {
        print!("{}", chunk);
        io::stdout().flush().ok();
    })
    .await?;

println!("\n");
```

Now responses appear word-by-word as Claude generates them. This feels more responsive than waiting for the full response.

## Tool Calling: Web Search

Define a tool that Claude can call:

```rust
// src/tools.rs
use anyhow::Result;
use serde_json::json;

pub fn get_tools() -> Vec<api::Tool> {
    vec![
        api::Tool {
            name: "web_search".to_string(),
            description: "Search the web for current information".to_string(),
            input_schema: json!({
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "The search query"
                    }
                },
                "required": ["query"]
            }),
        },
        api::Tool {
            name: "python_exec".to_string(),
            description: "Execute Python code and return the output".to_string(),
            input_schema: json!({
                "type": "object",
                "properties": {
                    "code": {
                        "type": "string",
                        "description": "The Python code to execute"
                    }
                },
                "required": ["code"]
            }),
        },
    ]
}

pub async fn execute_tool(name: &str, input: serde_json::Value) -> Result<String> {
    match name {
        "web_search" => web_search(input).await,
        "python_exec" => python_exec(input).await,
        _ => anyhow::bail!("Unknown tool: {}", name),
    }
}

async fn web_search(input: serde_json::Value) -> Result<String> {
    let query = input["query"]
        .as_str()
        .ok_or_else(|| anyhow::anyhow!("Missing query"))?;

    // Simple DuckDuckGo HTML scraping (for demonstration)
    // In production, use a proper search API
    let url = format!(
        "https://html.duckduckgo.com/html/?q={}",
        urlencoding::encode(query)
    );

    let client = reqwest::Client::new();
    let response = client
        .get(&url)
        .header("User-Agent", "Mozilla/5.0")
        .send()
        .await?
        .text()
        .await?;

    // Very crude parsing: extract first few results
    // In production, use a proper HTML parser or search API
    let results: Vec<&str> = response
        .lines()
        .filter(|line| line.contains("result__snippet"))
        .take(3)
        .collect();

    if results.is_empty() {
        Ok(format!("No results found for: {}", query))
    } else {
        Ok(format!(
            "Search results for '{}':\n{}",
            query,
            results.join("\n")
        ))
    }
}

async fn python_exec(input: serde_json::Value) -> Result<String> {
    let code = input["code"]
        .as_str()
        .ok_or_else(|| anyhow::anyhow!("Missing code"))?;

    // WARNING: This is unsafe! Only for testing.
    // In production, use containers (Docker) or a sandboxing library.
    let output = tokio::process::Command::new("python3")
        .arg("-c")
        .arg(code)
        .output()
        .await?;

    if output.status.success() {
        Ok(String::from_utf8_lossy(&output.stdout).to_string())
    } else {
        Ok(format!(
            "Error:\n{}",
            String::from_utf8_lossy(&output.stderr)
        ))
    }
}
```

Add `urlencoding = "2"` to `Cargo.toml` for the web search.

**Security note**: The Python execution uses `std::process::Command`, which runs code directly on your system. This is fine for personal testing but catastrophic in production. For production, use:
- Docker containers (spawn a container, run code, capture output, kill container)
- WebAssembly sandbox (compile Python to WASM, run in isolated environment)
- VM-based sandboxes (Firecracker, gVisor)

Python comparison: Same issue. `exec()` in Python is equally dangerous. The solution is the same (containers, sandboxing).

## The Tool-Calling Agent Loop

```rust
// src/main.rs (updated loop)
mod api;
mod tools;

use api::{AnthropicClient, ApiRequest, ContentBlock, Message};

// ... (setup code same as before)

loop {
    // ... (get user input)

    conversation.push(Message {
        role: "user".to_string(),
        content: input.to_string(),
    });

    // Agent loop: might need multiple turns for tool use
    loop {
        let request = ApiRequest {
            model: "claude-3-5-haiku-20241022".to_string(),
            max_tokens: 2048,
            messages: conversation.clone(),
            tools: Some(tools::get_tools()),
            stream: None,
        };

        let response = client.send_message(request).await?;

        let mut tool_uses = Vec::new();
        let mut text_parts = Vec::new();

        for block in &response.content {
            match block {
                ContentBlock::Text { text } => {
                    text_parts.push(text.clone());
                }
                ContentBlock::ToolUse { id, name, input } => {
                    println!("[Calling tool: {}]", name);
                    tool_uses.push((id.clone(), name.clone(), input.clone()));
                }
            }
        }

        if !text_parts.is_empty() {
            println!("\n{}\n", text_parts.join("\n"));
        }

        // If no tool uses, we're done with this turn
        if tool_uses.is_empty() {
            let assistant_text = text_parts.join("\n");
            conversation.push(Message {
                role: "assistant".to_string(),
                content: assistant_text,
            });
            break;
        }

        // Execute tools and add results to conversation
        conversation.push(Message {
            role: "assistant".to_string(),
            content: serde_json::to_string(&response.content)?,
        });

        for (tool_id, tool_name, tool_input) in tool_uses {
            let result = tools::execute_tool(&tool_name, tool_input).await?;
            println!("[Tool result: {}]\n", result.lines().take(3).collect::<Vec<_>>().join("\n"));

            conversation.push(Message {
                role: "user".to_string(),
                content: serde_json::to_string(&serde_json::json!({
                    "type": "tool_result",
                    "tool_use_id": tool_id,
                    "content": result,
                }))?,
            });
        }

        // Loop again to let Claude process tool results
    }
}
```

Now when you ask "What's the weather in Paris?", the agent:
1. Realizes it needs current information
2. Calls the `web_search` tool
3. Gets results
4. Formulates an answer based on the results

Try it:

```
> What's the current price of Bitcoin?

[Calling tool: web_search]
[Tool result: Search results for 'current price of Bitcoin'...]

Based on the search results, Bitcoin is currently trading at approximately $43,500 USD.

(tokens: 234 in, 67 out)
```

## Why This Beats SDKs

**Advantages of minimal dependencies:**

1. **Control**: You see exactly what's happening. No magic.
2. **Debugging**: When something breaks, you read 300 lines, not 3000.
3. **Customization**: Want to add request signing? Custom retry logic? Just do it.
4. **Binary size**: 8MB vs 50MB+ with full SDKs.
5. **Compile time**: Sub-10 seconds vs 30+ seconds with heavy dependencies.
6. **Security surface**: Five dependencies to audit vs dozens.

**Disadvantages:**

1. **API changes**: You handle them manually (though Anthropic's API is stable).
2. **Missing convenience**: No built-in retry, rate limiting, etc. (but you can add exactly what you need).
3. **Type coverage**: Official SDKs might have more complete types.

For most use cases, the advantages outweigh the disadvantages.

## Python Comparison

Equivalent Python code with the official SDK:

```python
import anthropic

client = anthropic.Anthropic(api_key="...")

conversation = []
while True:
    user_input = input("> ")
    if user_input == "exit":
        break

    conversation.append({"role": "user", "content": user_input})

    response = client.messages.create(
        model="claude-3-5-haiku-20241022",
        max_tokens=1024,
        messages=conversation,
    )

    print(response.content[0].text)
    conversation.append({"role": "assistant", "content": response.content[0].text})
```

This is simpler (Python SDK handles serialization, errors, etc.). But:
- Startup time: 1-2 seconds (import overhead)
- Memory: ~50MB base + dependencies
- Type safety: Runtime errors only
- Async: Requires `asyncio` boilerplate

Our Rust version:
- Startup time: 10ms
- Memory: ~5MB
- Type safety: Compile-time guarantees
- Async: Built into the language

Choose based on your priorities. For long-running agents, deployed services, or embedded systems, Rust wins. For quick scripts or Jupyter notebooks, Python is faster to write.

## Exercise: Add More Tools

Extend the agent with:

1. **File system tool**: Read/write files (with path restrictions for safety)
2. **HTTP fetch tool**: GET arbitrary URLs, return content
3. **Calculator tool**: For precise math (Claude sometimes gets arithmetic wrong)
4. **Database query tool**: Query a SQLite database

Hints:
- Define the tool schema in `get_tools()`
- Implement the logic in `execute_tool()`
- Test with prompts that need that tool

## What You Learned

- Making direct HTTP calls to LLM APIs (no SDK needed)
- Parsing JSON with serde
- Implementing tool calling (function calling)
- Building an agent loop (user input → API → tool execution → API → response)
- Streaming responses with Server-Sent Events
- Security considerations for code execution
- Why minimal dependencies matter (size, compile time, security)
- Rust vs Python for AI agents

Next: Distributed AI agents. We'll make this agent communicate with other agents via iroh, enabling cross-device tool execution. Your laptop's agent can delegate tasks to your Pi's agent, which has GPIO access and Docker control.
