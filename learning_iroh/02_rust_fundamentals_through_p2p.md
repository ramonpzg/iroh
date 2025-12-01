# Part 02: Rust Fundamentals Through P2P

You've written some Rust. Now let's understand what it actually means by reading iroh's source code. We're going to learn Rust's core concepts (ownership, borrowing, types, traits) by examining how iroh uses them to build a P2P network stack.

## The Type System: More Than Pedantry

Open `iroh-base/src/key.rs`. Look at these lines:

```rust
#[derive(Clone, Copy, PartialEq, Eq)]
#[repr(transparent)]
pub struct PublicKey(CompressedEdwardsY);

pub type EndpointId = PublicKey;
```

Two types that are actually the same type. Why?

In Python, you'd just use the bytes everywhere:

```python
endpoint_id = b"some_32_byte_key..."
public_key = b"some_32_byte_key..."

# These are the same type
type(endpoint_id) == type(public_key)  # True
```

This compiles but tells you nothing. Is this key for encryption? Signing? Identification? The type is `bytes`, which means "arbitrary data."

Rust's `PublicKey` and `EndpointId` are the same underlying type but different semantic types. One is for cryptography, the other for network identity. The compiler treats them as convertible but distinct. You can't accidentally encrypt with an endpoint ID without explicitly converting it.

```rust
fn encrypt(key: PublicKey, data: &[u8]) -> Vec<u8> { /* ... */ }
fn send_to(endpoint: EndpointId, data: &[u8]) -> Result<()> { /* ... */ }

let id = endpoint.remote_id();  // Returns EndpointId
encrypt(id, b"secret");  // Works! EndpointId is PublicKey
send_to(id, b"hello");   // Obviously works
```

The documentation tells you when to use which:

```rust
/// - `encrypt(key: PublicKey)`
/// - `send_to(endpoint: EndpointId)`
```

This is called the "newtype pattern." Wrap one type in another to give it semantic meaning. The `#[repr(transparent)]` attribute ensures zero runtime cost. The wrapper is free.

Python comparison: You could use `typing.NewType`, but it's only checked by mypy, not at runtime:

```python
from typing import NewType

EndpointId = NewType('EndpointId', bytes)
PublicKey = NewType('PublicKey', bytes)

def send_to(endpoint: EndpointId, data: bytes): ...

# This will pass type checking even though it shouldn't
send_to(b"raw_bytes", b"data")
```

## Ownership: The Thing Everyone Warns You About

Look at this function from `endpoint.rs`:

```rust
pub async fn connect(&self, addr: EndpointAddr, alpn: &[u8]) -> Result<Connection> {
    // Connection establishment logic
}
```

Three parameters, three different ownership models:

1. `&self`: Borrowed reference to the endpoint
2. `addr: EndpointAddr`: Owned value
3. `alpn: &[u8]`: Borrowed slice

### Owned Values

```rust
addr: EndpointAddr
```

The function takes ownership of `addr`. After you call `connect`, you can't use `addr` anymore (unless the type implements `Copy`).

```rust
let addr = endpoint.addr();
endpoint.connect(addr, ALPN).await?;
// addr is moved, can't use it here
println!("{:?}", addr);  // Compile error!
```

In Python, everything is a reference:

```python
addr = endpoint.addr()
endpoint.connect(addr, ALPN)
print(addr)  # Works fine, addr still exists
```

Python's model is simpler but hides costs. You never know if a function will keep a reference to your data, modify it, or pass it to another thread. Rust makes you explicit.

Why does `connect` take ownership? Because `EndpointAddr` contains the relay URLs and direct addresses. The connection needs to keep this data around. Taking ownership avoids copying and makes the lifetime clear: the connection owns this address.

### Borrowed References

```rust
&self
```

The method borrows the endpoint immutably. You can have many immutable borrows:

```rust
let conn1 = endpoint.connect(addr1, ALPN);
let conn2 = endpoint.connect(addr2, ALPN);
let conn3 = endpoint.connect(addr3, ALPN);
```

All three borrow `endpoint` simultaneously. This is safe because none of them modify it.

```rust
alpn: &[u8]
```

The function borrows the ALPN bytes. It reads them to establish the connection but doesn't need to keep them around. After the handshake, the ALPN is irrelevant.

Python comparison: Everything is borrowed by default:

```python
def connect(self, addr, alpn):
    # self, addr, and alpn are all references
    # No distinction between owned and borrowed
```

### Mutable Borrows

Look at this from the echo example:

```rust
async fn accept(&self, connection: Connection) -> Result<(), AcceptError> {
    let (mut send, mut recv) = connection.accept_bi().await?;
    tokio::io::copy(&mut recv, &mut send).await?;
    send.finish()?;
    Ok(())
}
```

`mut send` and `mut recv` are mutable variables, but `tokio::io::copy` takes `&mut recv` and `&mut send` (mutable borrows).

Why mutable? Because reading from `recv` changes its internal state (advances the read position). Writing to `send` changes its state (buffers data).

Key rule: You can have many immutable borrows OR one mutable borrow, but not both simultaneously.

```rust
let mut stream = connection.open_uni().await?;
let borrow1 = &mut stream;
let borrow2 = &mut stream;  // Compile error! Can't have two mutable borrows

borrow1.write_all(b"data").await?;
borrow2.write_all(b"more").await?;  // Would be racy
```

This prevents data races at compile time. Python has no such protection:

```python
stream = connection.open_uni()
borrow1 = stream  # Just another reference
borrow2 = stream  # Also just a reference

# If stream is used from two threads, you get a data race
# No compile-time protection
```

## Traits: Interfaces With Superpowers

Remember this from Part 01?

```rust
impl ProtocolHandler for Echo {
    async fn accept(&self, connection: Connection) -> Result<(), AcceptError> {
        // ...
    }
}
```

`ProtocolHandler` is a trait (Rust's version of interfaces). Let's look at its definition (simplified from `protocol.rs`):

```rust
pub trait ProtocolHandler: Send + Sync + 'static {
    async fn accept(&self, connection: Connection) -> Result<(), AcceptError>;
}
```

Three bounds after the trait name: `Send + Sync + 'static`. These are marker traits that enable safe concurrency.

### Send and Sync

- `Send`: The type can be transferred between threads
- `Sync`: The type can be accessed from multiple threads simultaneously (via `&T`)

Most types are `Send` and `Sync` automatically. But types containing raw pointers, thread-local data, or non-atomic shared state are not.

Why does `ProtocolHandler` require these? Because your handler might be called from any thread in the tokio runtime. The router doesn't know or care which thread handles which connection.

Python comparison: Everything is `Send` and `Sync` by default because of the GIL, but you pay for it with single-threaded performance:

```python
class ProtocolHandler:
    async def accept(self, connection):
        pass

# Python doesn't prevent you from having non-thread-safe state
# You just get crashes or data corruption at runtime
```

### The 'static Lifetime

```rust
'static
```

This doesn't mean the value lives forever. It means the type doesn't contain any borrowed references (or all references live for the entire program).

Your `Echo` struct:

```rust
struct Echo;
```

Contains no data, so it's trivially `'static`. If it contained a reference, it wouldn't be:

```rust
struct Echo<'a> {
    config: &'a Config,  // Borrowed reference
}

// This would fail: Echo is not 'static because it borrows data
impl ProtocolHandler for Echo<'_> { /* ... */ }
```

Why the restriction? Because the handler might outlive the thing that created it. The router spawns a task for each connection, and those tasks might run long after your code has moved on.

## Error Handling: Result Is Not Optional

Look at error handling in iroh:

```rust
pub async fn connect(&self, addr: EndpointAddr, alpn: &[u8]) -> Result<Connection> {
    // ...
}
```

Returns `Result<Connection>`, which is really `Result<Connection, Error>`. Either you get a `Connection` or an `Error`. No exceptions.

Python comparison:

```python
def connect(self, addr: bytes, alpn: bytes) -> Connection:
    # Might raise an exception, who knows?
    # The type signature doesn't tell you
```

In Python, any function might raise any exception at any time. The type system doesn't track this. You have to read the documentation (if it exists) or the source code (if you're diligent) or wait for it to crash in production (if you're normal).

Rust makes errors explicit. The `?` operator propagates errors:

```rust
let conn = endpoint.connect(addr, ALPN).await?;
```

If `connect` returns an error, `?` returns early from the current function with that error. If it succeeds, `conn` gets the unwrapped `Connection`.

Equivalent Python:

```python
conn = endpoint.connect(addr, ALPN)  # Might raise, might not
```

But Rust's version is checked. If you forget to handle the error, your code won't compile:

```rust
let conn = endpoint.connect(addr, ALPN).await;  // Compile error!
// You must either:
// - Use ? to propagate: endpoint.connect(addr, ALPN).await?
// - Match: match endpoint.connect(addr, ALPN).await { Ok(c) => ..., Err(e) => ... }
// - Unwrap (panic on error): endpoint.connect(addr, ALPN).await.unwrap()
```

### Context and Error Conversion

Notice this pattern:

```rust
.await.anyerr()?
.await.std_context("failed to connect")?
```

These are extension methods from `n0_error` that add context to errors. `anyerr()` converts any error type into the function's error type. `std_context()` adds a string describing what failed.

Python comparison:

```python
try:
    conn = await endpoint.connect(addr, ALPN)
except Exception as e:
    raise RuntimeError("failed to connect") from e
```

But Rust's version is zero-cost (errors are return values, not exceptions) and composable (you can chain context).

## The Clone and Copy Distinction

Look at this trait bound:

```rust
#[derive(Debug, Clone)]
struct Echo;
```

`Clone` means you can explicitly copy it with `.clone()`:

```rust
let echo1 = Echo;
let echo2 = echo1.clone();
```

Now look at `PublicKey`:

```rust
#[derive(Clone, Copy, PartialEq, Eq)]
pub struct PublicKey(CompressedEdwardsY);
```

It derives both `Clone` and `Copy`. `Copy` means it's implicitly copied on assignment:

```rust
let key1 = public_key;
let key2 = public_key;  // key1 is copied, not moved
// Both key1 and key2 are valid
```

Why the distinction? `Copy` types must be cheap to copy (think: integers, small structs of copyable data). `Clone` types might be expensive (think: `Vec`, `String`, `Connection`).

```rust
let conn1 = connection;
let conn2 = connection;  // Compile error! Connection is not Copy
// You must explicitly clone
let conn2 = connection.clone();
```

Python comparison: Everything is a reference, so "copying" is always cheap (just copies the pointer). But you have to worry about unexpected aliasing:

```python
list1 = [1, 2, 3]
list2 = list1  # Just a reference
list2.append(4)
print(list1)  # [1, 2, 3, 4] - surprise!
```

## Smart Pointers: Arc and Shared Ownership

Iroh uses `Arc` (atomic reference counted) extensively for shared ownership:

```rust
let router = Router::builder(endpoint).accept(ALPN, Echo).spawn();
```

Internally, the router wraps your handler in an `Arc`:

```rust
Arc::new(handler)
```

Why? Because the handler is shared between the router and all active connections. When a connection comes in, it clones the `Arc` (cheap: just increments a counter) rather than cloning the handler itself.

```rust
let handler = Arc::new(Echo);
let handler_clone = handler.clone();  // Just increments refcount
// handler and handler_clone point to the same Echo instance
```

Python comparison: Everything is already reference counted:

```python
handler = Echo()
handler_clone = handler  # Same object, refcount incremented
```

But Python's refcounting is slower (needs GIL for thread-safety) and invisible (you can't tell when something is shared). Rust's `Arc` is explicit and lock-free.

## Exercise: Reading Real Code

Open `iroh/src/endpoint.rs`. Find the `Endpoint::connect` method. Read through it. You'll see:

1. Ownership: What does it take ownership of? What does it borrow?
2. Error handling: How many different errors can occur?
3. Async: Where does it wait? Why?
4. Traits: What trait bounds are on generic parameters?
5. Lifetimes: Are there any explicit lifetime annotations?

Don't worry if you don't understand everything. The goal is to start recognizing patterns:

- `&self` means borrowing
- `mut` means mutation
- `?` means error propagation
- `.await` means waiting
- `Result<T>` means fallible
- `Arc<T>` means shared ownership

## Python to Rust Mental Model

If you're comfortable with Python, here's the translation table:

| Python | Rust | Notes |
|--------|------|-------|
| All values are references | Most values are owned | Makes mutation and aliasing explicit |
| `def func(x)` | `fn func(x: T)` | Types required, explicit ownership |
| `x = y` | `let x = y` | Might move or copy, depending on type |
| `x.method()` | `x.method()` | Might borrow `&self` or `&mut self` |
| `class MyClass` | `struct MyStruct` | Data only, methods in `impl` blocks |
| `class Protocol` | `trait Protocol` | Interfaces/protocols |
| `try: ... except: ...` | `Result<T, E>` + `?` | Errors are values, not exceptions |
| `async def / await` | `async fn / .await` | Same concept, different syntax |
| `threading.Thread` | `std::thread::spawn` | True parallelism, no GIL |
| `list[int]` | `Vec<i32>` | Generic syntax is `<T>` not `[T]` |
| `dict[str, int]` | `HashMap<String, i32>` | More cumbersome, but faster |

The biggest mental shift: In Python, you pass references and hope nobody mutates them unexpectedly. In Rust, the compiler guarantees nobody can mutate them without your permission.

## What You Learned

- Rust's type system encodes semantic meaning, not just data layout
- Ownership makes resource management explicit and safe
- Borrowing enables concurrent access without data races
- Traits are interfaces with compile-time guarantees
- Errors are values, not exceptions
- `Clone` and `Copy` make copying explicit
- `Arc` enables shared ownership when needed

Next: Async Rust and QUIC streams. We'll understand why everything is `async`, how futures work, and how QUIC's stream model maps to Rust's async I/O.
