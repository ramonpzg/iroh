# Part 08: Cross-Platform Development

You've built CLI and TUI tools. Now let's build GUIs. We'll create the same file-sharing app for the web (with Leptos + WebAssembly) and desktop (with GPUI). One backend, multiple frontends.

## Why Cross-Platform with Rust

Python has Electron (run a browser in a window), Qt (mature but C++-based), or Tkinter (looks like 1995). Rust has native options that compile to efficient code.

**Web**: Compile Rust to WebAssembly, run in the browser. Full access to Rust libraries.
**Desktop**: Native UI frameworks (GPUI, egui, iced, Tauri) that render natively.

The key: share business logic, swap UI layer.

## Architecture: Business Logic Separate from UI

```rust
// src/core.rs - shared logic
pub struct FileShareCore {
    endpoint: Endpoint,
    shared_files: Vec<FileInfo>,
}

impl FileShareCore {
    pub async fn new() -> Result<Self> { /* ... */ }
    pub async fn share_file(&mut self, path: PathBuf) -> Result<()> { /* ... */ }
    pub async fn list_files(&self) -> Vec<FileInfo> { /* ... */ }
    pub async fn download(&self, share_code: String) -> Result<()> { /* ... */ }
}
```

This core is platform-agnostic. No UI code, no platform-specific I/O.

Then wrap it:

```rust
// src/web.rs - Leptos web UI
use leptos::*;

#[component]
fn App(core: FileShareCore) -> impl IntoView { /* ... */ }

// src/desktop.rs - GPUI desktop UI
use gpui::*;

struct FileShareApp {
    core: FileShareCore,
}
```

Same `core`, different UIs.

## Part 1: Web with Leptos

Leptos is a Rust web framework. Think React, but compiled to WebAssembly. Reactive state, components, all the modern web stuff.

### Setup

```bash
cargo new --bin iroh-share-web
cd iroh-share-web
```

Add dependencies:

```toml
[dependencies]
iroh = { version = "0.30", default-features = false, features = ["wasm"] }
leptos = { version = "0.7", features = ["csr"] }
wasm-bindgen = "0.2"
wasm-bindgen-futures = "0.4"
console_error_panic_hook = "0.1"
```

Note: `iroh` with `wasm` feature. Iroh can run in the browser (limited: no UDP, relay-only).

### The Core (Shared Logic)

```rust
// src/core.rs
use iroh::{Endpoint, EndpointId, EndpointAddr};
use std::sync::Arc;
use tokio::sync::RwLock;

#[derive(Clone)]
pub struct FileShareCore {
    endpoint: Endpoint,
    files: Arc<RwLock<Vec<FileInfo>>>,
}

#[derive(Clone, Debug)]
pub struct FileInfo {
    pub name: String,
    pub size: u64,
}

impl FileShareCore {
    pub async fn new() -> anyhow::Result<Self> {
        let endpoint = Endpoint::bind().await?;
        Ok(Self {
            endpoint,
            files: Arc::new(RwLock::new(Vec::new())),
        })
    }

    pub fn endpoint_id(&self) -> EndpointId {
        self.endpoint.id()
    }

    pub async fn add_file(&self, name: String, size: u64) {
        self.files.write().await.push(FileInfo { name, size });
    }

    pub async fn list_files(&self) -> Vec<FileInfo> {
        self.files.read().await.clone()
    }

    pub fn generate_share_code(&self) -> String {
        format!("iroh://{}/share", self.endpoint.id())
    }
}
```

Platform-agnostic. Works on native and web.

### The Web UI

```rust
// src/main.rs
use leptos::*;
use wasm_bindgen::prelude::*;

mod core;
use core::FileShareCore;

#[wasm_bindgen(start)]
pub fn main() {
    console_error_panic_hook::set_once();
    mount_to_body(|| view! { <App/> })
}

#[component]
fn App() -> impl IntoView {
    let (core, set_core) = create_signal(None::<FileShareCore>);
    let (files, set_files) = create_signal(Vec::<core::FileInfo>::new());
    let (share_code, set_share_code) = create_signal(String::new());

    // Initialize core on mount
    create_effect(move |_| {
        spawn_local(async move {
            if let Ok(c) = FileShareCore::new().await {
                set_share_code.set(c.generate_share_code());
                set_core.set(Some(c));
            }
        });
    });

    view! {
        <div class="app">
            <h1>"Iroh File Share"</h1>

            <div class="share-code">
                <h2>"Your Share Code"</h2>
                <input
                    type="text"
                    readonly
                    value=move || share_code.get()
                />
            </div>

            <div class="files">
                <h2>"Shared Files"</h2>
                <ul>
                    <For
                        each=move || files.get()
                        key=|f| f.name.clone()
                        children=move |file| {
                            view! {
                                <li>{file.name.clone()} " (" {file.size} " bytes)"</li>
                            }
                        }
                    />
                </ul>
            </div>

            <div class="actions">
                <button on:click=move |_| {
                    if let Some(c) = core.get() {
                        let c = c.clone();
                        spawn_local(async move {
                            c.add_file("example.txt".to_string(), 1024).await;
                            set_files.set(c.list_files().await);
                        });
                    }
                }>"Add File"</button>
            </div>
        </div>
    }
}
```

Leptos components look like React JSX. The `view!` macro generates code to render DOM nodes.

### Limitations in WASM

WebAssembly in the browser can't:
- Open arbitrary UDP sockets (security restriction)
- Access the filesystem directly (need File API)
- Spawn threads (single-threaded)

Iroh in WASM:
- Uses WebSocket to relay server (not UDP)
- Only relay connections (no direct)
- Single-threaded async (works fine)

For file access, use the browser's File API:

```rust
use web_sys::{File, FileReader};
use wasm_bindgen::JsCast;

async fn read_file(file: File) -> Result<Vec<u8>> {
    let reader = FileReader::new()?;
    // Set up promise-based reading
    // ...
}
```

### Building and Running

```bash
# Install trunk (WASM build tool)
cargo install trunk

# Add rust wasm target
rustup target add wasm32-unknown-unknown

# Build and serve
trunk serve --open
```

This compiles to WASM, bundles with HTML/CSS/JS, and serves locally. Open browser to `http://localhost:8080`.

You'll see your share code and can interact with the UI. File sharing works through relay (direct connections aren't possible in browser).

## Part 2: Desktop with GPUI

GPUI is Zed's UI framework. Fast, modern, GPU-accelerated. Still evolving (breaking changes happen), but powerful.

### Setup

```bash
cargo new --bin iroh-share-desktop
cd iroh-share-desktop
```

Add dependencies:

```toml
[dependencies]
iroh = "0.30"
gpui = { git = "https://github.com/zed-industries/zed", rev = "..." }  # Pin a commit
tokio = { version = "1", features = ["full"] }
anyhow = "1"
```

GPUI isn't on crates.io yet. Pin to a git commit.

### The Core (Same as Before)

Copy `src/core.rs` from the web version. It's identical.

### The Desktop UI

```rust
// src/main.rs
use gpui::*;
use std::sync::Arc;
use tokio::runtime::Runtime;

mod core;
use core::FileShareCore;

struct FileShareApp {
    core: Arc<FileShareCore>,
    files: Vec<core::FileInfo>,
}

impl FileShareApp {
    fn new(core: Arc<FileShareCore>) -> Self {
        Self {
            core,
            files: Vec::new(),
        }
    }

    fn refresh_files(&mut self, cx: &mut ViewContext<Self>) {
        let core = self.core.clone();
        cx.spawn(|this, mut cx| async move {
            let files = core.list_files().await;
            this.update(&mut cx, |this, cx| {
                this.files = files;
                cx.notify();
            }).ok();
        }).detach();
    }
}

impl Render for FileShareApp {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        div()
            .flex()
            .flex_col()
            .size_full()
            .child(
                div()
                    .p_4()
                    .child("Iroh File Share")
                    .text_size(px(24.))
            )
            .child(
                div()
                    .p_4()
                    .child(format!("Share Code: {}", self.core.generate_share_code()))
            )
            .child(
                div()
                    .p_4()
                    .flex()
                    .flex_col()
                    .children(self.files.iter().map(|f| {
                        div().child(format!("{} ({} bytes)", f.name, f.size))
                    }))
            )
            .child(
                div()
                    .p_4()
                    .child(
                        button()
                            .child("Add File")
                            .on_click(cx.listener(|this, _, cx| {
                                let core = this.core.clone();
                                cx.spawn(|this, mut cx| async move {
                                    core.add_file("example.txt".to_string(), 1024).await;
                                    this.update(&mut cx, |this, cx| {
                                        this.refresh_files(cx);
                                    }).ok();
                                }).detach();
                            }))
                    )
            )
    }
}

fn main() {
    let rt = Runtime::new().unwrap();
    let core = rt.block_on(async {
        FileShareCore::new().await.unwrap()
    });

    App::new().run(move |cx: &mut AppContext| {
        let bounds = Bounds::centered(None, size(px(600.), px(400.)), cx);
        cx.open_window(
            WindowOptions {
                window_bounds: Some(WindowBounds::Windowed(bounds)),
                ..Default::default()
            },
            |cx| {
                let core = Arc::new(core);
                cx.new_view(|cx| {
                    let mut app = FileShareApp::new(core);
                    app.refresh_files(cx);
                    app
                })
            },
        );
    });
}
```

GPUI's API is different from Leptos:
- `Render` trait instead of component functions
- Method chaining for styling (`.p_4()` for padding)
- `cx.spawn()` for async tasks

But the core logic is the same.

### Running

```bash
cargo run
```

A native window opens with your app. No browser, no Electron overhead. Just Rust code rendering UI.

## Comparison: Web vs Desktop

| Aspect | Leptos (Web) | GPUI (Desktop) |
|--------|--------------|----------------|
| Distribution | URL (instant access) | Binary (download/install) |
| Startup time | Fast (loads in browser) | Instant (native) |
| Memory | Browser overhead (~100MB+) | Minimal (~10MB) |
| File access | Limited (File API) | Full (OS APIs) |
| Networking | WebSocket only | Full UDP/TCP |
| Permissions | Sandboxed | Full system access |
| Updates | Instant (reload page) | Manual or auto-update |

Choose based on your needs. Web for accessibility, desktop for power.

## Shared Code Across Platforms

The pattern:

```
project/
├── core/          # Platform-agnostic business logic
│   ├── src/
│   │   └── lib.rs
│   └── Cargo.toml
├── web/           # Leptos web UI
│   ├── src/
│   │   └── main.rs
│   └── Cargo.toml
└── desktop/       # GPUI desktop UI
    ├── src/
    │   └── main.rs
    └── Cargo.toml
```

Each UI depends on `core`:

```toml
# web/Cargo.toml
[dependencies]
core = { path = "../core" }

# desktop/Cargo.toml
[dependencies]
core = { path = "../core" }
```

Changes to business logic propagate to all UIs automatically.

Python comparison: You'd use a shared module and different frontends (Flask for web, PyQt for desktop). But the UI frameworks would be completely different (HTML/CSS vs Qt widgets). In Rust, frameworks share more concepts (reactive state, components).

## Styling: Making It Not Ugly

Both frameworks support styling.

**Leptos**: Use CSS classes:

```rust
view! {
    <div class="card">
        <h2 class="card-title">"Title"</h2>
    </div>
}
```

Add a `style.css`:

```css
.card {
    border: 1px solid #ccc;
    border-radius: 8px;
    padding: 16px;
}
```

Or use Tailwind (via Trunk).

**GPUI**: Inline styling:

```rust
div()
    .border_1()
    .border_color(rgb(0xcccccc))
    .rounded(px(8.))
    .p_4()
```

Or define styles as functions:

```rust
fn card_style() -> Div {
    div().border_1().rounded(px(8.)).p_4()
}

// Usage:
card_style().child("Content")
```

## State Management

Both frameworks are reactive: when state changes, UI updates automatically.

**Leptos**: Signals

```rust
let (count, set_count) = create_signal(0);

view! {
    <button on:click=move |_| set_count.update(|n| *n += 1)>
        "Count: " {move || count.get()}
    </button>
}
```

When `set_count` is called, the `{move || count.get()}` automatically re-runs and updates the DOM.

**GPUI**: ViewContext + Notify

```rust
struct Counter {
    count: usize,
}

impl Render for Counter {
    fn render(&mut self, cx: &mut ViewContext<Self>) -> impl IntoElement {
        div().child(
            button()
                .child(format!("Count: {}", self.count))
                .on_click(cx.listener(|this, _, cx| {
                    this.count += 1;
                    cx.notify();  // Trigger re-render
                }))
        )
    }
}
```

When `cx.notify()` is called, GPUI re-renders the component.

## Exercise: Multi-Platform Chat App

Build a P2P chat app with:
- Web UI (Leptos): Join chat by entering endpoint ID
- Desktop UI (GPUI): Same functionality, native window
- Shared core: Connection management, message handling

**Hints**:
- Use a channel for message passing (core to UI)
- Store messages in core, expose via async method
- Use iroh's bidirectional streams for chat

**Bonus**: Add file attachments (works on desktop, limited on web).

## Mobile: React Native via Rust FFI

Iroh can run on mobile (iOS, Android). But building mobile UIs in Rust is less mature. The pragmatic approach: Rust core + React Native UI.

1. Expose Rust as a C API:

```rust
// lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::c_char;

#[no_mangle]
pub extern "C" fn iroh_create_endpoint() -> *mut Endpoint {
    // ...
}

#[no_mangle]
pub extern "C" fn iroh_connect(
    endpoint: *mut Endpoint,
    peer_id: *const c_char,
) -> i32 {
    // ...
}
```

2. Build as a shared library:

```toml
[lib]
crate-type = ["cdylib"]
```

3. Call from React Native (via FFI bridge like `react-native-rust`):

```javascript
import { IrohCore } from './native-bridge';

const endpoint = IrohCore.createEndpoint();
IrohCore.connect(endpoint, peerId);
```

This is more work than pure Rust, but gives you mature mobile UI frameworks.

## What You Learned

- Separate business logic from UI for code reuse
- Leptos compiles Rust to WebAssembly for browser apps
- GPUI builds native desktop apps with GPU acceleration
- WebAssembly has limitations (no UDP, sandboxed filesystem)
- Reactive state management in both frameworks
- Mobile apps need FFI bridges (Rust core + native UI)
- Platform choice depends on requirements (distribution, capabilities, UX)

Next: Embedded and mobile development. Running iroh on ESP32, Raspberry Pi, Android, and iOS. The challenges of resource-constrained and mobile environments.
