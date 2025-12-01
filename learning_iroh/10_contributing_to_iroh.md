# Part 10: Contributing to Iroh

You've learned Rust, built applications, and understand P2P networking. Now let's contribute to iroh itself. Reading unfamiliar code, finding bugs, writing tests, and submitting pull requests.

## Why Contribute

Selfish reasons:
- Learn from experienced Rust developers
- Get your name in the git history
- Fix bugs that affect your applications
- Add features you need

Altruistic reasons:
- Improve software others rely on
- Help grow the Rust P2P ecosystem
- Pay it forward (you've learned from this codebase)

## Understanding the Codebase

### Repository Structure

```
iroh/
├── iroh/               # Main library
│   ├── src/
│   │   ├── endpoint.rs   # High-level endpoint API
│   │   ├── magicsock.rs  # NAT traversal, path selection
│   │   ├── disco.rs      # Path discovery protocol
│   │   └── ...
│   ├── examples/        # Example applications
│   └── tests/           # Integration tests
├── iroh-relay/         # Relay server implementation
├── iroh-base/          # Common types (keys, addresses)
├── iroh-dns-server/    # DNS discovery server
└── Cargo.toml          # Workspace manifest
```

Start with examples. They're simpler than the library itself but demonstrate usage patterns.

### Reading Code: The Layered Approach

Don't try to understand everything at once. Start from your use case and work inward.

**Example: You want to understand how connection establishment works.**

1. Find the entry point: `Endpoint::connect()` in `iroh/src/endpoint.rs`
2. Read the function. It calls `self.msock.connect()`
3. Follow to MagicSock: `iroh/src/magicsock.rs`
4. See it sends disco messages (discovery protocol)
5. Follow to `disco.rs` for the protocol details

Each layer reveals more. You don't need to understand disco protocol to use `Endpoint::connect()`. But if you're fixing a connection bug, you'll eventually need to.

### Using rust-analyzer

Install rust-analyzer in your editor (VSCode, Neovim, Emacs). It provides:
- Go to definition (jump to source of a function)
- Find references (see where a function is called)
- Type hints (see inferred types)
- Auto-complete

This is essential for navigating large codebases.

```rust
// Hover over `connect` to see its signature
endpoint.connect(addr, ALPN).await?;

// Ctrl+Click to jump to definition
```

### Reading Tests

Tests show expected behavior:

```rust
// From iroh/tests/endpoint.rs
#[tokio::test]
async fn test_basic_connect() {
    let ep1 = Endpoint::bind().await.unwrap();
    let ep2 = Endpoint::bind().await.unwrap();

    let addr = ep2.addr();
    let conn = ep1.connect(addr, ALPN).await.unwrap();

    // Test sends and receives data
    // ...
}
```

Reading tests tells you:
- How to use the API
- What edge cases are handled
- What assumptions the code makes

If you find a bug, write a failing test first. Then fix it. This ensures:
- The bug is actually fixed
- It won't regress in the future

## Finding Your First Issue

### Good First Issues

Look for issues labeled "good first issue" or "help wanted" on GitHub. These are explicitly curated for new contributors.

Example: "Add configurable timeout for XYZ" is easier than "Refactor the entire endpoint state machine."

### Documentation Improvements

The lowest-risk contribution: improve documentation.

- Found a confusing doc comment? Clarify it.
- Missing example? Add one.
- Typo? Fix it.

```rust
/// Connects to a remote endpoint.
///
/// This method attempts to establish a connection using the provided address.
/// It will try direct connection first, falling back to relay if necessary.
///
/// # Example
///
/// ```no_run
/// # use iroh::{Endpoint, EndpointAddr};
/// # async fn example() -> anyhow::Result<()> {
/// let endpoint = Endpoint::bind().await?;
/// let addr = /* ... */;
/// let conn = endpoint.connect(addr, b"my-alpn").await?;
/// # Ok(())
/// # }
/// ```
pub async fn connect(&self, addr: EndpointAddr, alpn: &[u8]) -> Result<Connection> {
    // ...
}
```

Even small clarifications help.

### Testing Edge Cases

Found a case that isn't tested? Add a test.

```rust
#[tokio::test]
async fn test_connect_with_invalid_alpn() {
    let ep1 = Endpoint::bind().await.unwrap();
    let ep2 = Endpoint::builder()
        .alpns(vec![b"other-alpn".to_vec()])
        .bind()
        .await
        .unwrap();

    let addr = ep2.addr();
    let result = ep1.connect(addr, b"wrong-alpn").await;

    // Should fail due to ALPN mismatch
    assert!(result.is_err());
}
```

This test documents expected behavior and catches regressions.

## Writing a Bug Fix

Let's walk through a hypothetical bug fix.

### Step 1: Reproduce

You notice: "Connections fail after 10 minutes of inactivity."

Write a test that reproduces it:

```rust
#[tokio::test]
async fn test_connection_survives_inactivity() {
    let ep1 = Endpoint::bind().await.unwrap();
    let ep2 = Endpoint::bind().await.unwrap();

    let addr = ep2.addr();
    let conn = ep1.connect(addr, ALPN).await.unwrap();

    // Wait 10 minutes (or use a shorter timeout for testing)
    tokio::time::sleep(Duration::from_secs(600)).await;

    // Connection should still work
    let mut stream = conn.open_uni().await.unwrap();
    stream.write_all(b"still alive").await.unwrap();
    stream.finish().unwrap();

    // This test currently fails (hypothetically)
}
```

Run the test: `cargo test test_connection_survives_inactivity`. It fails. Good.

### Step 2: Investigate

Why does it fail? Enable tracing:

```bash
RUST_LOG=iroh=debug cargo test test_connection_survives_inactivity
```

Logs show:

```
DEBUG iroh::magicsock: No recent activity, closing connection to <peer>
```

Ah. MagicSock is closing idle connections.

### Step 3: Find the Code

Search the codebase for that log message:

```bash
grep -r "No recent activity" iroh/src/
```

Found in `magicsock.rs`:

```rust
if last_activity.elapsed() > IDLE_TIMEOUT {
    debug!("No recent activity, closing connection to {}", endpoint_id);
    self.close_connection(endpoint_id);
}
```

The problem: QUIC connections can be idle (no data) but still alive (keepalives). MagicSock is prematurely closing them.

### Step 4: Fix

Instead of closing, send a keepalive:

```rust
if last_activity.elapsed() > KEEPALIVE_INTERVAL {
    debug!("Sending keepalive to {}", endpoint_id);
    self.send_keepalive(endpoint_id)?;
}
```

### Step 5: Test

Run the test again. It passes. Run all tests:

```bash
cargo test
```

All pass. Good.

### Step 6: Document

Add a comment explaining the fix:

```rust
// Send periodic keepalives for idle connections to prevent NAT
// mapping expiration. Without this, connections idle for more
// than ~30s may fail when resumed due to NAT state loss.
if last_activity.elapsed() > KEEPALIVE_INTERVAL {
    debug!("Sending keepalive to {}", endpoint_id);
    self.send_keepalive(endpoint_id)?;
}
```

### Step 7: Submit PR

1. Commit your changes:

```bash
git checkout -b fix-idle-connection
git add .
git commit -m "Fix: Send keepalives for idle connections

Previously, connections idle for >10 minutes would fail due to
MagicSock closing them. Now we send periodic keepalives to
maintain NAT mappings and QUIC connection state.

Fixes #123"
```

Reference the issue number if there is one.

2. Push to your fork:

```bash
git push origin fix-idle-connection
```

3. Open a pull request on GitHub.

4. Respond to review feedback.

## Code Review: What Reviewers Look For

### Correctness

Does it fix the bug without introducing new ones?

**Bad**:

```rust
// Might panic if endpoint_id doesn't exist
self.connections.get(endpoint_id).unwrap().send_keepalive();
```

**Good**:

```rust
if let Some(conn) = self.connections.get(endpoint_id) {
    conn.send_keepalive()?;
}
```

### Tests

Are there tests that cover the change?

If you're fixing a bug, the test should fail without your fix and pass with it.

If you're adding a feature, the test should exercise the new functionality.

### Documentation

Are public APIs documented? Are doc comments updated if behavior changes?

```rust
/// Connects to a remote endpoint.
///
/// This method maintains the connection with periodic keepalives,
/// so idle connections won't timeout due to NAT expiration.
pub async fn connect(&self, addr: EndpointAddr, alpn: &[u8]) -> Result<Connection> {
    // ...
}
```

### Code Style

Follow the existing style. Use `rustfmt`:

```bash
cargo fmt
```

And `clippy` for lints:

```bash
cargo clippy
```

Fix any warnings (or explain why they're false positives).

### Commit Messages

Write clear commit messages:

**Bad**:

```
fix bug
```

**Good**:

```
Fix: Send keepalives for idle connections

Previously, connections idle for >10 minutes would fail due to
MagicSock closing them. Now we send periodic keepalives to
maintain NAT mappings and QUIC connection state.

The keepalive interval is set to 30 seconds, well within typical
NAT mapping timeouts (60-120 seconds).

Fixes #123
```

Explain what, why, and how.

## Performance Improvements

Optimizations need benchmarks. Don't optimize without measuring.

### Step 1: Benchmark

Create a benchmark:

```rust
// benches/connection.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use iroh::Endpoint;

fn bench_connection_setup(c: &mut Criterion) {
    let rt = tokio::runtime::Runtime::new().unwrap();
    c.bench_function("connection_setup", |b| {
        b.iter(|| {
            rt.block_on(async {
                let ep1 = Endpoint::bind().await.unwrap();
                let ep2 = Endpoint::bind().await.unwrap();
                let addr = ep2.addr();
                let _conn = ep1.connect(addr, b"bench").await.unwrap();
            });
        });
    });
}

criterion_group!(benches, bench_connection_setup);
criterion_main!(benches);
```

Run it:

```bash
cargo bench
```

Output:

```
connection_setup        time:   [52.123 ms 52.456 ms 52.789 ms]
```

### Step 2: Optimize

You notice: endpoint binds to both IPv4 and IPv6, but the benchmark only uses IPv4. Binding to IPv6 is wasted.

Add a configuration option:

```rust
let ep = Endpoint::builder()
    .ipv6(false)  // Disable IPv6
    .bind()
    .await?;
```

### Step 3: Re-benchmark

```bash
cargo bench
```

Output:

```
connection_setup        time:   [48.234 ms 48.567 ms 48.900 ms]
                        change: [-8.12% -7.41% -6.70%] (p = 0.00 < 0.05)
                        Performance has improved.
```

A 7% improvement. Document it in the PR.

### Step 4: Ensure Correctness

Run all tests:

```bash
cargo test
```

Ensure the optimization doesn't break anything.

## Adding Features

Features are harder than bug fixes. They require design decisions.

### Step 1: Discussion

Open an issue describing the feature:

```markdown
## Feature Request: Connection Priority

I'd like to prioritize certain connections (e.g., high-priority data should
preempt low-priority). QUIC supports stream priorities, but iroh doesn't expose
this.

Proposed API:

```rust
let conn = endpoint.connect(addr, alpn).await?;
let high_priority_stream = conn.open_bi_with_priority(Priority::High).await?;
```

This would map to QUIC's stream priority mechanism.

Thoughts?
```

Wait for feedback. Maybe the maintainers have opinions:
- "We've been thinking about this, here's our design."
- "This doesn't fit our architecture, but here's an alternative."
- "We don't want this feature because X."

Don't write code until there's consensus on the design.

### Step 2: Implement

Once approved, implement it.

### Step 3: Document

Features need examples:

```rust
/// Opens a bidirectional stream with a specified priority.
///
/// Higher priority streams will be scheduled before lower priority streams
/// when bandwidth is limited.
///
/// # Example
///
/// ```no_run
/// # use iroh::{Endpoint, Priority};
/// # async fn example(conn: iroh::endpoint::Connection) -> anyhow::Result<()> {
/// let stream = conn.open_bi_with_priority(Priority::High).await?;
/// # Ok(())
/// # }
/// ```
pub async fn open_bi_with_priority(&self, priority: Priority) -> Result<(SendStream, RecvStream)> {
    // ...
}
```

### Step 4: Test

Test the feature thoroughly:

```rust
#[tokio::test]
async fn test_stream_priority() {
    let ep1 = Endpoint::bind().await.unwrap();
    let ep2 = Endpoint::bind().await.unwrap();

    let conn = ep1.connect(ep2.addr(), ALPN).await.unwrap();

    let high = conn.open_bi_with_priority(Priority::High).await.unwrap();
    let low = conn.open_bi_with_priority(Priority::Low).await.unwrap();

    // Test that high priority stream gets bandwidth first
    // ...
}
```

## Reviewing Others' PRs

Contributing isn't just code. Review others' pull requests:

1. Read the code. Does it make sense?
2. Run the tests. Do they pass?
3. Try the feature. Does it work as described?
4. Leave constructive feedback:

**Bad**:

```
This is wrong.
```

**Good**:

```
I think this could cause a panic if the connection is closed while this is running.
Consider using `if let Some(conn) = self.connections.get(id)` instead of `.unwrap()`.
```

Explain why, suggest alternatives.

## Iroh's Development Practices

(These are general good practices; check iroh's actual CONTRIBUTING.md for specifics.)

- **Tests required**: All bug fixes and features need tests.
- **Clippy clean**: Run `cargo clippy` and fix warnings.
- **Formatted**: Run `cargo fmt`.
- **Documented**: Public APIs must have doc comments.
- **Changelog**: Update CHANGELOG.md with your changes.
- **Sign commits**: (If required by the project)

## Common Pitfalls

### Trying to Change Too Much

Start small. A 10-line PR is easier to review than a 1000-line refactoring.

### Not Reading the Code First

Understand the existing architecture before proposing changes. Your "better way" might break assumptions elsewhere.

### Bikeshedding

Don't argue about style (tabs vs spaces, naming conventions). Follow the existing style.

### Taking Feedback Personally

Code review is about the code, not you. "This function could be clearer" doesn't mean "You're a bad programmer."

## The Bigger Picture: Open Source Community

Contributing to iroh connects you to:
- Other Rust developers
- The P2P networking community
- People building interesting applications

Join the Discord/IRC/forum. Ask questions. Share what you're building. Help others.

Open source isn't just code. It's people solving problems together.

## Exercise: Your First Contribution

1. Clone iroh: `git clone https://github.com/n0-computer/iroh`
2. Read CONTRIBUTING.md
3. Browse open issues
4. Find something small (documentation, test, minor bug)
5. Fix it
6. Submit a PR

Don't aim for perfection. Aim for progress.

## What You've Learned (This Entire Series)

1. **Part 01**: Building your first P2P connection
2. **Part 02**: Rust fundamentals (ownership, traits, types)
3. **Part 03**: Async Rust and QUIC streams
4. **Part 04**: Iroh's architecture (MagicSock, QUIC, relay)
5. **Part 05**: Building a CLI file sharing tool
6. **Part 06**: TUI development with ratatui
7. **Part 07**: Deep P2P concepts (NAT traversal, DHT, gossip)
8. **Part 08**: Cross-platform (web and desktop)
9. **Part 09**: Embedded and mobile
10. **Part 10**: Contributing to iroh

You've gone from Rust novice to capable iroh contributor. Build something interesting. Break it. Fix it. Share it.

Welcome to the community.

## Resources

- **Iroh Repo**: https://github.com/n0-computer/iroh
- **Iroh Docs**: https://iroh.computer/docs
- **Rust Book**: https://doc.rust-lang.org/book/
- **Async Book**: https://rust-lang.github.io/async-book/
- **Quinn (QUIC)**: https://github.com/quinn-rs/quinn
- **Discord/Community**: (Check iroh's README for links)

Now go build.
