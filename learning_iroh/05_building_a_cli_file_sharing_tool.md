# Part 05: Building a CLI File Sharing Tool

Stop reading. Start building. You're going to create `iroh-send`, a command-line tool for peer-to-peer file sharing. Think `scp` but without needing SSH access or knowing the remote IP address.

## What You're Building

```bash
# On machine A
$ iroh-send serve /path/to/files
Listening...
Endpoint ID: abc123...
Share code: iroh://abc123.../share/xyz789

# On machine B
$ iroh-send get iroh://abc123.../share/xyz789
Downloading...
Received: file1.txt (1.2 MB)
Received: file2.jpg (450 KB)
Done!
```

Two machines, different networks, no configuration. Just works.

## Project Setup

```bash
cargo new --bin iroh-send
cd iroh-send
```

Add dependencies to `Cargo.toml`:

```toml
[dependencies]
iroh = "0.30"
tokio = { version = "1", features = ["full"] }
anyhow = "1"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
blake3 = "1"
indicatif = "0.17"
tokio-util = { version = "0.7", features = ["codec"] }
futures = "0.3"
```

What each does:
- `iroh`: P2P networking
- `tokio`: Async runtime
- `anyhow`: Error handling (simpler than `Result<T, E>` for applications)
- `clap`: Command-line argument parsing
- `serde`: Serialization (for the protocol)
- `blake3`: Hashing (for content verification)
- `indicatif`: Progress bars
- `tokio-util`: Utilities for async I/O
- `futures`: Stream combinators

## The Protocol

Before code, define the protocol. What messages do peers exchange?

```rust
// src/protocol.rs
use serde::{Deserialize, Serialize};

const ALPN: &[u8] = b"iroh-send/0";

#[derive(Debug, Serialize, Deserialize)]
enum Request {
    ListFiles,
    GetFile { name: String },
}

#[derive(Debug, Serialize, Deserialize)]
enum Response {
    FileList { files: Vec<FileInfo> },
    FileData { name: String, size: u64 },
    Error { message: String },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
struct FileInfo {
    name: String,
    size: u64,
    hash: [u8; 32],  // BLAKE3 hash
}
```

Simple request/response. Could be more sophisticated (resumable transfers, compression), but this works.

Python comparison: You'd use `json` or `pickle`. But JSON is slower to parse and pickle isn't cross-language. We're using `bincode` (implied by serde):

```rust
use serde::{Serialize, Deserialize};

let request = Request::ListFiles;
let bytes = bincode::serialize(&request)?;  // Fast binary encoding
let decoded: Request = bincode::deserialize(&bytes)?;
```

## The Server Side

```rust
// src/server.rs
use anyhow::{Context, Result};
use iroh::{Endpoint, endpoint::Connection};
use std::path::{Path, PathBuf};
use tokio::fs::{self, File};
use tokio::io::AsyncReadExt;

use crate::protocol::*;

pub async fn serve(path: PathBuf) -> Result<()> {
    let endpoint = Endpoint::bind().await
        .context("Failed to create endpoint")?;

    println!("Serving: {}", path.display());
    println!("Endpoint ID: {}", endpoint.id());
    println!("Share code: iroh://{}/share", endpoint.id());

    endpoint.online().await;

    // Accept connections forever
    while let Some(incoming) = endpoint.accept().await {
        let conn = match incoming.accept() {
            Ok(accepting) => accepting,
            Err(err) => {
                eprintln!("Failed to accept: {}", err);
                continue;
            }
        };

        let path = path.clone();
        tokio::spawn(async move {
            if let Err(e) = handle_connection(conn.await?, path).await {
                eprintln!("Connection error: {}", e);
            }
            Ok::<(), anyhow::Error>(())
        });
    }

    Ok(())
}

async fn handle_connection(conn: Connection, base_path: PathBuf) -> Result<()> {
    let endpoint_id = conn.remote_id();
    println!("[{}] Connected", endpoint_id.fmt_short());

    // Accept the bi-directional stream
    let (mut send, mut recv) = conn.accept_bi().await?;

    // Read request
    let mut buf = vec![0u8; 4096];
    let n = recv.read(&mut buf).await?;
    let request: Request = bincode::deserialize(&buf[..n])?;

    match request {
        Request::ListFiles => {
            let files = scan_directory(&base_path).await?;
            let response = Response::FileList { files };
            let data = bincode::serialize(&response)?;
            send.write_all(&data).await?;
            send.finish()?;
        }
        Request::GetFile { name } => {
            let file_path = base_path.join(&name);
            // Security: prevent path traversal
            if !file_path.starts_with(&base_path) {
                let response = Response::Error {
                    message: "Invalid path".to_string(),
                };
                let data = bincode::serialize(&response)?;
                send.write_all(&data).await?;
                send.finish()?;
                return Ok(());
            }

            let mut file = File::open(&file_path).await?;
            let size = file.metadata().await?.len();

            // Send response header
            let response = Response::FileData {
                name: name.clone(),
                size,
            };
            let data = bincode::serialize(&response)?;
            send.write_all(&data).await?;

            // Stream file content
            tokio::io::copy(&mut file, &mut send).await?;
            send.finish()?;

            println!("[{}] Sent: {} ({} bytes)", endpoint_id.fmt_short(), name, size);
        }
    }

    conn.closed().await;
    Ok(())
}

async fn scan_directory(path: &Path) -> Result<Vec<FileInfo>> {
    let mut files = Vec::new();
    let mut entries = fs::read_dir(path).await?;

    while let Some(entry) = entries.next_entry().await? {
        let metadata = entry.metadata().await?;
        if metadata.is_file() {
            let name = entry.file_name().to_string_lossy().to_string();
            let size = metadata.len();
            let hash = hash_file(entry.path()).await?;

            files.push(FileInfo { name, size, hash });
        }
    }

    Ok(files)
}

async fn hash_file(path: impl AsRef<Path>) -> Result<[u8; 32]> {
    let mut file = File::open(path).await?;
    let mut hasher = blake3::Hasher::new();
    let mut buf = vec![0u8; 65536];  // 64KB buffer

    loop {
        let n = file.read(&mut buf).await?;
        if n == 0 {
            break;
        }
        hasher.update(&buf[..n]);
    }

    Ok(*hasher.finalize().as_bytes())
}
```

### Security Note

Notice this check:

```rust
if !file_path.starts_with(&base_path) {
    // Reject
}
```

This prevents path traversal attacks. If a malicious client sends:

```rust
Request::GetFile { name: "../../etc/passwd".to_string() }
```

The check fails because `/some/path/../../etc/passwd` doesn't start with `/some/path/`.

Python comparison: You'd need the same check:

```python
import os

file_path = os.path.join(base_path, name)
if not file_path.startswith(base_path):
    raise ValueError("Invalid path")
```

But Python's `os.path.join` has subtle gotchas (absolute paths override the base). Rust's `Path::join` is more predictable.

## The Client Side

```rust
// src/client.rs
use anyhow::{Context, Result, bail};
use iroh::{Endpoint, EndpointAddr, EndpointId};
use indicatif::{ProgressBar, ProgressStyle};
use std::path::PathBuf;
use tokio::fs::File;
use tokio::io::AsyncWriteExt;

use crate::protocol::*;

pub async fn get(share_code: String) -> Result<()> {
    // Parse share code: iroh://<endpoint_id>/share
    let endpoint_id = parse_share_code(&share_code)?;

    let endpoint = Endpoint::bind().await
        .context("Failed to create endpoint")?;

    println!("Connecting to {}...", endpoint_id.fmt_short());

    // Try to connect
    let addr = EndpointAddr::from_parts(endpoint_id, vec![]);
    let conn = endpoint.connect(addr, ALPN).await
        .context("Failed to connect")?;

    println!("Connected!");

    // List files
    let files = list_files(&conn).await?;

    println!("Available files:");
    for file in &files {
        println!("  {} ({} bytes)", file.name, file.size);
    }

    // Download all files
    for file in &files {
        download_file(&conn, &file.name, file.size).await?;
    }

    conn.close(0u32.into(), b"done");
    endpoint.close().await;

    println!("Done!");
    Ok(())
}

async fn list_files(conn: &iroh::endpoint::Connection) -> Result<Vec<FileInfo>> {
    let (mut send, mut recv) = conn.open_bi().await?;

    // Send request
    let request = Request::ListFiles;
    let data = bincode::serialize(&request)?;
    send.write_all(&data).await?;
    send.finish()?;

    // Read response
    let response_data = recv.read_to_end(1024 * 1024).await?;  // Max 1MB for file list
    let response: Response = bincode::deserialize(&response_data)?;

    match response {
        Response::FileList { files } => Ok(files),
        Response::Error { message } => bail!("Server error: {}", message),
        _ => bail!("Unexpected response"),
    }
}

async fn download_file(
    conn: &iroh::endpoint::Connection,
    name: &str,
    expected_size: u64,
) -> Result<()> {
    let (mut send, mut recv) = conn.open_bi().await?;

    // Send request
    let request = Request::GetFile {
        name: name.to_string(),
    };
    let data = bincode::serialize(&request)?;
    send.write_all(&data).await?;
    send.finish()?;

    // Read response header
    let mut header_buf = vec![0u8; 1024];
    let n = recv.read(&mut header_buf).await?;
    let response: Response = bincode::deserialize(&header_buf[..n])?;

    let size = match response {
        Response::FileData { size, .. } => size,
        Response::Error { message } => bail!("Server error: {}", message),
        _ => bail!("Unexpected response"),
    };

    // Setup progress bar
    let pb = ProgressBar::new(size);
    pb.set_style(
        ProgressStyle::default_bar()
            .template("{msg} [{bar:40}] {bytes}/{total_bytes} ({eta})")?
            .progress_chars("#>-"),
    );
    pb.set_message(format!("Downloading {}", name));

    // Download file
    let mut file = File::create(name).await?;
    let mut downloaded = 0u64;
    let mut buf = vec![0u8; 65536];  // 64KB buffer

    loop {
        let n = recv.read(&mut buf).await?;
        if n == 0 {
            break;
        }
        file.write_all(&buf[..n]).await?;
        downloaded += n as u64;
        pb.set_position(downloaded);
    }

    pb.finish_with_message(format!("Downloaded {}", name));

    if downloaded != size {
        bail!("Size mismatch: expected {}, got {}", size, downloaded);
    }

    Ok(())
}

fn parse_share_code(code: &str) -> Result<EndpointId> {
    // Format: iroh://<endpoint_id>/share
    let code = code.strip_prefix("iroh://")
        .context("Invalid share code format")?;
    let endpoint_id_str = code.split('/').next()
        .context("Invalid share code format")?;

    endpoint_id_str.parse()
        .context("Invalid endpoint ID")
}
```

Progress bars! This is what separates toy projects from usable tools.

## The Main Function and CLI

```rust
// src/main.rs
mod protocol;
mod server;
mod client;

use clap::{Parser, Subcommand};
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "iroh-send")]
#[command(about = "P2P file sharing via iroh", long_about = None)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
}

#[derive(Subcommand)]
enum Commands {
    /// Serve files from a directory
    Serve {
        /// Directory to serve
        path: PathBuf,
    },
    /// Download files from a share code
    Get {
        /// Share code (iroh://...)
        share_code: String,
    },
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();

    match cli.command {
        Commands::Serve { path } => {
            server::serve(path).await?;
        }
        Commands::Get { share_code } => {
            client::get(share_code).await?;
        }
    }

    Ok(())
}
```

`clap` derives the CLI from structs. This is better than manual argument parsing:

```rust
// No need for this:
if args.len() < 2 {
    eprintln!("Usage: ...");
    std::process::exit(1);
}
```

Python comparison with `argparse`:

```python
import argparse

parser = argparse.ArgumentParser(description='P2P file sharing')
subparsers = parser.add_subparsers(dest='command')

serve_parser = subparsers.add_parser('serve')
serve_parser.add_argument('path', type=str)

get_parser = subparsers.add_parser('get')
get_parser.add_argument('share_code', type=str)

args = parser.parse_args()
```

Rust's version is more type-safe (the enum guarantees valid commands) and generates help text automatically.

## Running It

Terminal 1:

```bash
cargo run -- serve /path/to/files
```

Terminal 2:

```bash
cargo run -- get iroh://abc123.../share
```

If both terminals are on the same machine, they'll connect via localhost. Try different machines (different WiFi networks, mobile hotspot, etc.) to see real P2P.

## What Could Go Wrong

### Discovery Failures

If you only pass the endpoint ID (no relay URL or direct addresses), the connection might fail. Fix:

```rust
// In client.rs
let addr = EndpointAddr::from_parts(endpoint_id, vec![
    TransportAddr::Relay(default_relay_url()),
]);
```

Or configure discovery:

```rust
use iroh::discovery::dns::DnsDiscovery;

let endpoint = Endpoint::builder()
    .discovery(DnsDiscovery::builder("iroh.example".to_string()))
    .bind()
    .await?;
```

### Firewall Blocks

If direct connection fails and relay is blocked by firewall, you're stuck. Test with:

```bash
# Check if you can reach the default relay
curl https://use1-1.relay.iroh.network/
```

If not, you'll need to configure a relay you can reach or disable firewalls (not recommended).

### Large File Memory Usage

The file list response is loaded entirely into memory:

```rust
let response_data = recv.read_to_end(1024 * 1024).await?;
```

If the file list is huge (10,000 files), this might OOM. Fix: stream the response or paginate.

## Improvements to Make

This is a working tool, but far from polished. Here are exercises:

### 1. Resumable Downloads

If download fails mid-way, start over. Add resume support:

```rust
Request::GetFileRange {
    name: String,
    start: u64,
    end: u64,
}
```

Check if partial file exists, resume from that offset.

### 2. Encryption

Files are encrypted in transit (QUIC) but not at rest. Add an optional passphrase:

```bash
iroh-send serve /files --password=secret
iroh-send get iroh://... --password=secret
```

Derive a key from the password, encrypt files before sending.

### 3. Directory Support

Currently only serves files in one directory. Add recursive directory traversal and preserve structure on download.

### 4. Compression

Compress files before transfer:

```rust
use async_compression::tokio::write::GzipEncoder;

let file = File::open(path).await?;
let mut encoder = GzipEncoder::new(stream);
tokio::io::copy(&mut file, &mut encoder).await?;
```

Trade CPU for bandwidth.

### 5. Multi-Source Downloads

BitTorrent-style: download different parts from different peers simultaneously.

This requires:
- Chunk the file into pieces
- Request different pieces from different peers
- Verify each piece with its hash
- Reassemble

Significantly more complex, but much faster for large files with many seeders.

## Python Comparison: How You'd Build This

Python with `asyncio`:

```python
import asyncio
import json

async def serve(path):
    reader, writer = await asyncio.start_server(
        lambda r, w: handle_client(r, w, path),
        '0.0.0.0', 8080
    )
    async with reader:
        await reader.serve_forever()

async def handle_client(reader, writer, path):
    data = await reader.read(4096)
    request = json.loads(data)
    # ... handle request
```

Problems:
- No P2P (need manual port forwarding or NAT traversal)
- No encryption (need to add TLS manually)
- No identity (client doesn't know who they're connecting to)
- Single-threaded (Python's GIL)

You'd need to add:
- A STUN/TURN library for NAT traversal
- TLS for encryption
- Some identity mechanism (certificates?)
- Threading or multiprocessing for parallelism

Iroh gives you all of this out of the box.

## What You Learned

- Building a real CLI tool with `clap`
- Designing a binary protocol with `serde`
- Handling file I/O with `tokio::fs`
- Progress bars with `indicatif`
- Security considerations (path traversal)
- Error handling with `anyhow`
- Real-world P2P file transfer

Next: Building a TUI (terminal user interface) with `ratatui`. We'll make `iroh-send` interactive with real-time progress, connection status, and controls.
