# Part 06: TUI Development with Ratatui

Command-line tools are fine. But sometimes you want more than scrolling text. Enter TUIs: terminal user interfaces. Think `htop`, `btm`, or `k9s`. Fully interactive applications in your terminal.

We're going to build `iroh-monitor`, a real-time dashboard for your iroh endpoint. It will show:
- Active connections
- Transfer speeds
- Connection types (direct vs relay)
- Network events

And you'll control it with keyboard shortcuts.

## Why TUIs

GUIs are heavy. Web interfaces need a browser. CLIs are limited to linear output. TUIs sit in the middle: rich interaction without leaving the terminal.

In Python, you might use `curses` (low-level) or `rich` (high-level). In Rust, `ratatui` is the standard. It's a fork of `tui-rs` with active maintenance.

## Project Setup

```bash
cargo new --bin iroh-monitor
cd iroh-monitor
```

Add to `Cargo.toml`:

```toml
[dependencies]
iroh = "0.30"
tokio = { version = "1", features = ["full"] }
anyhow = "1"
ratatui = "0.29"
crossterm = "0.28"
```

`ratatui` handles the UI rendering. `crossterm` handles terminal control (raw mode, events). They work together.

## The Event Loop

TUIs have an event loop:

1. Read terminal input (keyboard, mouse)
2. Update application state
3. Render UI
4. Repeat

This is similar to GUI frameworks but in the terminal.

```rust
// src/main.rs
use anyhow::Result;
use crossterm::{
    event::{self, Event, KeyCode, KeyEvent, KeyEventKind},
    execute,
    terminal::{disable_raw_mode, enable_raw_mode, EnterAlternateScreen, LeaveAlternateScreen},
};
use ratatui::{
    backend::CrosstermBackend,
    Terminal,
};
use std::io;

#[tokio::main]
async fn main() -> Result<()> {
    // Setup terminal
    enable_raw_mode()?;
    let mut stdout = io::stdout();
    execute!(stdout, EnterAlternateScreen)?;
    let backend = CrosstermBackend::new(stdout);
    let mut terminal = Terminal::new(backend)?;

    // Run app
    let res = run_app(&mut terminal).await;

    // Restore terminal
    disable_raw_mode()?;
    execute!(terminal.backend_mut(), LeaveAlternateScreen)?;
    terminal.show_cursor()?;

    if let Err(err) = res {
        eprintln!("Error: {:?}", err);
    }

    Ok(())
}

async fn run_app<B: ratatui::backend::Backend>(terminal: &mut Terminal<B>) -> Result<()> {
    loop {
        terminal.draw(|f| {
            ui(f);
        })?;

        if event::poll(std::time::Duration::from_millis(100))? {
            if let Event::Key(key) = event::read()? {
                if key.kind == KeyEventKind::Press {
                    match key.code {
                        KeyCode::Char('q') => return Ok(()),
                        _ => {}
                    }
                }
            }
        }
    }
}

fn ui(f: &mut ratatui::Frame) {
    use ratatui::{
        layout::{Constraint, Direction, Layout},
        widgets::{Block, Borders, Paragraph},
    };

    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Length(3),
            Constraint::Min(0),
        ])
        .split(f.area());

    let title = Paragraph::new("Iroh Monitor - Press 'q' to quit")
        .block(Block::default().borders(Borders::ALL));
    f.render_widget(title, chunks[0]);

    let content = Paragraph::new("Hello, TUI!")
        .block(Block::default().borders(Borders::ALL));
    f.render_widget(content, chunks[1]);
}
```

Run it:

```bash
cargo run
```

You should see a bordered box with "Iroh Monitor". Press 'q' to quit.

### What's Happening

```rust
enable_raw_mode()?;
```

Puts the terminal in raw mode: no line buffering, no echo. Every keypress is sent to your app immediately.

```rust
execute!(stdout, EnterAlternateScreen)?;
```

Switches to an alternate screen buffer. Your app draws here. When you exit, the alternate screen disappears and your shell prompt is unchanged. This is how `vim` and `htop` don't leave artifacts.

```rust
terminal.draw(|f| { ui(f); })?;
```

Renders the UI. The closure receives a `Frame`, you draw widgets into it.

```rust
event::poll(std::time::Duration::from_millis(100))?
```

Checks if input is available, waiting up to 100ms. This makes the loop responsive without busy-waiting.

Python comparison with `curses`:

```python
import curses

def main(stdscr):
    stdscr.clear()
    while True:
        stdscr.addstr(0, 0, "Iroh Monitor")
        stdscr.refresh()

        key = stdscr.getch()
        if key == ord('q'):
            break

curses.wrapper(main)
```

Similar structure, but `ratatui` gives you higher-level widgets and layout management.

## Application State

We need to track connections and stats:

```rust
// src/app.rs
use iroh::EndpointId;
use std::collections::HashMap;
use std::time::Instant;

pub struct App {
    pub endpoint_id: EndpointId,
    pub connections: HashMap<EndpointId, ConnectionInfo>,
    pub total_sent: u64,
    pub total_received: u64,
    pub start_time: Instant,
}

pub struct ConnectionInfo {
    pub endpoint_id: EndpointId,
    pub connection_type: String,
    pub bytes_sent: u64,
    pub bytes_received: u64,
    pub last_activity: Instant,
}

impl App {
    pub fn new(endpoint_id: EndpointId) -> Self {
        Self {
            endpoint_id,
            connections: HashMap::new(),
            total_sent: 0,
            total_received: 0,
            start_time: Instant::now(),
        }
    }

    pub fn update_connection(&mut self, endpoint_id: EndpointId, info: ConnectionInfo) {
        self.connections.insert(endpoint_id, info);
    }

    pub fn remove_connection(&mut self, endpoint_id: &EndpointId) {
        self.connections.remove(endpoint_id);
    }

    pub fn uptime(&self) -> std::time::Duration {
        self.start_time.elapsed()
    }
}
```

## Rendering the UI

Now we build a real UI:

```rust
// src/ui.rs
use ratatui::{
    layout::{Constraint, Direction, Layout, Rect},
    style::{Color, Modifier, Style},
    text::{Line, Span, Text},
    widgets::{Block, Borders, List, ListItem, Paragraph, Row, Table},
    Frame,
};

use crate::app::App;

pub fn ui(f: &mut Frame, app: &App) {
    let chunks = Layout::default()
        .direction(Direction::Vertical)
        .constraints([
            Constraint::Length(7),   // Header
            Constraint::Min(10),     // Connections table
            Constraint::Length(3),   // Footer
        ])
        .split(f.area());

    render_header(f, chunks[0], app);
    render_connections(f, chunks[1], app);
    render_footer(f, chunks[2]);
}

fn render_header(f: &mut Frame, area: Rect, app: &App) {
    let uptime = app.uptime();
    let uptime_str = format!("{}h {}m {}s",
        uptime.as_secs() / 3600,
        (uptime.as_secs() % 3600) / 60,
        uptime.as_secs() % 60
    );

    let text = vec![
        Line::from(vec![
            Span::styled("Endpoint ID: ", Style::default().fg(Color::Cyan)),
            Span::raw(app.endpoint_id.to_string()),
        ]),
        Line::from(vec![
            Span::styled("Uptime: ", Style::default().fg(Color::Cyan)),
            Span::raw(uptime_str),
        ]),
        Line::from(vec![
            Span::styled("Connections: ", Style::default().fg(Color::Cyan)),
            Span::raw(format!("{}", app.connections.len())),
        ]),
        Line::from(vec![
            Span::styled("Total Sent: ", Style::default().fg(Color::Cyan)),
            Span::raw(format_bytes(app.total_sent)),
            Span::raw(" | "),
            Span::styled("Received: ", Style::default().fg(Color::Cyan)),
            Span::raw(format_bytes(app.total_received)),
        ]),
    ];

    let paragraph = Paragraph::new(text)
        .block(Block::default().borders(Borders::ALL).title("Status"));

    f.render_widget(paragraph, area);
}

fn render_connections(f: &mut Frame, area: Rect, app: &App) {
    let header = Row::new(vec!["Peer", "Type", "Sent", "Received", "Last Activity"])
        .style(Style::default().fg(Color::Yellow))
        .bottom_margin(1);

    let rows: Vec<Row> = app.connections.values().map(|conn| {
        let elapsed = conn.last_activity.elapsed().as_secs();
        Row::new(vec![
            conn.endpoint_id.fmt_short().to_string(),
            conn.connection_type.clone(),
            format_bytes(conn.bytes_sent),
            format_bytes(conn.bytes_received),
            format!("{}s ago", elapsed),
        ])
    }).collect();

    let table = Table::new(
        rows,
        &[
            Constraint::Length(15),
            Constraint::Length(10),
            Constraint::Length(12),
            Constraint::Length(12),
            Constraint::Length(15),
        ],
    )
    .header(header)
    .block(Block::default().borders(Borders::ALL).title("Connections"));

    f.render_widget(table, area);
}

fn render_footer(f: &mut Frame, area: Rect) {
    let text = Paragraph::new("Press 'q' to quit | 'r' to refresh")
        .style(Style::default().fg(Color::DarkGray))
        .block(Block::default().borders(Borders::ALL));

    f.render_widget(text, area);
}

fn format_bytes(bytes: u64) -> String {
    const KB: u64 = 1024;
    const MB: u64 = KB * 1024;
    const GB: u64 = MB * 1024;

    if bytes >= GB {
        format!("{:.2} GB", bytes as f64 / GB as f64)
    } else if bytes >= MB {
        format!("{:.2} MB", bytes as f64 / MB as f64)
    } else if bytes >= KB {
        format!("{:.2} KB", bytes as f64 / KB as f64)
    } else {
        format!("{} B", bytes)
    }
}
```

This creates a layout with three sections: header (stats), connections table, footer (help).

## Integrating Iroh

Now connect it to a real iroh endpoint:

```rust
// src/main.rs (updated)
use iroh::Endpoint;
use tokio::sync::mpsc;
use std::sync::{Arc, Mutex};

mod app;
mod ui;

use app::App;

#[tokio::main]
async fn main() -> Result<()> {
    // Create endpoint
    let endpoint = Endpoint::bind().await?;
    let endpoint_id = endpoint.id();

    // Create app state
    let app = Arc::new(Mutex::new(App::new(endpoint_id)));

    // Spawn background task to monitor endpoint
    let app_clone = app.clone();
    tokio::spawn(async move {
        monitor_endpoint(endpoint, app_clone).await;
    });

    // Setup terminal
    enable_raw_mode()?;
    let mut stdout = io::stdout();
    execute!(stdout, EnterAlternateScreen)?;
    let backend = CrosstermBackend::new(stdout);
    let mut terminal = Terminal::new(backend)?;

    // Run UI loop
    let res = run_app(&mut terminal, app).await;

    // Restore terminal
    disable_raw_mode()?;
    execute!(terminal.backend_mut(), LeaveAlternateScreen)?;
    terminal.show_cursor()?;

    if let Err(err) = res {
        eprintln!("Error: {:?}", err);
    }

    Ok(())
}

async fn run_app<B: ratatui::backend::Backend>(
    terminal: &mut Terminal<B>,
    app: Arc<Mutex<App>>,
) -> Result<()> {
    loop {
        // Draw UI
        {
            let app = app.lock().unwrap();
            terminal.draw(|f| {
                ui::ui(f, &app);
            })?;
        }

        // Handle input
        if event::poll(std::time::Duration::from_millis(100))? {
            if let Event::Key(key) = event::read()? {
                if key.kind == KeyEventKind::Press {
                    match key.code {
                        KeyCode::Char('q') => return Ok(()),
                        _ => {}
                    }
                }
            }
        }
    }
}

async fn monitor_endpoint(endpoint: Endpoint, app: Arc<Mutex<App>>) {
    use iroh::endpoint::ConnectionTypeStream;

    // Accept connections
    let endpoint_clone = endpoint.clone();
    tokio::spawn(async move {
        while let Some(incoming) = endpoint_clone.accept().await {
            if let Ok(accepting) = incoming.accept() {
                tokio::spawn(async move {
                    if let Ok(conn) = accepting.await {
                        // Connection established, but we just drop it for monitoring
                        conn.closed().await;
                    }
                });
            }
        }
    });

    // Monitor all endpoint metrics
    // In a real app, you'd hook into iroh's metrics system
    // For simplicity, we'll just periodically update
    loop {
        tokio::time::sleep(tokio::time::Duration::from_secs(1)).await;

        // Update app state with metrics
        let mut app = app.lock().unwrap();
        // TODO: Actually get metrics from endpoint
        // This is a placeholder
        app.total_sent += 1000;
        app.total_received += 2000;
    }
}
```

### The Threading Model

Notice the pattern:

```rust
let app = Arc::new(Mutex::new(App::new(endpoint_id)));
```

`Arc` for shared ownership (UI thread and monitor thread both access it).
`Mutex` for mutual exclusion (only one thread modifies at a time).

When drawing:

```rust
let app = app.lock().unwrap();
terminal.draw(|f| { ui::ui(f, &app); })?;
```

Lock, read, render, unlock. If the monitor thread is updating, the UI blocks briefly. This is fine for a TUI (humans won't notice 1ms).

Python comparison with `threading.Lock`:

```python
import threading

app = App()
lock = threading.Lock()

def ui_thread():
    while True:
        with lock:
            draw(app)

def monitor_thread():
    while True:
        with lock:
            app.update()
```

Same concept. Rust enforces it at compile time (you can't access `app` without locking). Python trusts you to remember.

## Real-Time Updates

The current version updates every 100ms (the `poll` timeout). For smoother updates, use channels:

```rust
use tokio::sync::mpsc;

enum AppEvent {
    ConnectionEstablished { endpoint_id: EndpointId },
    ConnectionClosed { endpoint_id: EndpointId },
    BytesTransferred { endpoint_id: EndpointId, sent: u64, received: u64 },
}

async fn monitor_endpoint(endpoint: Endpoint, tx: mpsc::Sender<AppEvent>) {
    // When something happens:
    tx.send(AppEvent::ConnectionEstablished { endpoint_id }).await.ok();
}

async fn run_app(terminal: &mut Terminal<B>, mut rx: mpsc::Receiver<AppEvent>, app: Arc<Mutex<App>>) {
    loop {
        // Draw UI
        // ...

        // Handle events from monitor
        while let Ok(event) = rx.try_recv() {
            let mut app = app.lock().unwrap();
            match event {
                AppEvent::ConnectionEstablished { endpoint_id } => {
                    app.update_connection(endpoint_id, /* ... */);
                }
                // ...
            }
        }

        // Handle keyboard input
        // ...
    }
}
```

Now updates are immediate, not polled.

## Advanced UI Features

### Scrollable Lists

If you have many connections, they won't fit on screen. Add scrolling:

```rust
use ratatui::widgets::TableState;

pub struct App {
    // ...
    pub table_state: TableState,
}

// In render:
let mut state = app.table_state.clone();
f.render_stateful_widget(table, area, &mut state);

// In input handling:
KeyCode::Down => {
    let i = app.table_state.selected().map(|i| i + 1).unwrap_or(0);
    app.table_state.select(Some(i.min(app.connections.len() - 1)));
}
KeyCode::Up => {
    let i = app.table_state.selected().map(|i| i.saturating_sub(1)).unwrap_or(0);
    app.table_state.select(Some(i));
}
```

### Charts

Show bandwidth over time:

```rust
use ratatui::widgets::{Chart, Axis, Dataset, GraphType};
use ratatui::symbols;

// Store bandwidth samples
pub struct App {
    pub bandwidth_history: Vec<(f64, f64)>,  // (time, bytes/sec)
}

fn render_chart(f: &mut Frame, area: Rect, app: &App) {
    let datasets = vec![
        Dataset::default()
            .name("Bandwidth")
            .marker(symbols::Marker::Braille)
            .graph_type(GraphType::Line)
            .data(&app.bandwidth_history),
    ];

    let chart = Chart::new(datasets)
        .block(Block::default().title("Bandwidth").borders(Borders::ALL))
        .x_axis(Axis::default().title("Time").bounds([0.0, 60.0]))
        .y_axis(Axis::default().title("MB/s").bounds([0.0, 100.0]));

    f.render_widget(chart, area);
}
```

### Mouse Support

```rust
use crossterm::event::{EnableMouseCapture, DisableMouseCapture};

// In setup:
execute!(stdout, EnableMouseCapture)?;

// In event loop:
if let Event::Mouse(mouse) = event::read()? {
    match mouse.kind {
        MouseEventKind::Down(MouseButton::Left) => {
            // Handle click
        }
        _ => {}
    }
}
```

## Performance Considerations

TUIs can be slow if you're not careful:

### Diff-Based Rendering

`ratatui` only redraws what changed. But if you redraw the entire UI every frame, it's wasteful. Cache unchanged parts:

```rust
// Don't:
fn ui(f: &mut Frame, app: &App) {
    // Rebuild entire UI every frame
}

// Do:
fn ui(f: &mut Frame, app: &App, last_draw: &Instant) {
    // Only rebuild if data changed
    if app.last_update > *last_draw {
        // Rebuild
    }
}
```

### Avoid Locking During Render

Minimize time holding the lock:

```rust
// Don't:
let app = app.lock().unwrap();
terminal.draw(|f| {
    // app is locked during entire render
    ui(f, &app);
})?;

// Do:
let snapshot = {
    let app = app.lock().unwrap();
    app.clone()  // Quick clone
};
// Lock released
terminal.draw(|f| {
    ui(f, &snapshot);
})?;
```

If `App` is cheap to clone, this is faster. If not, use `Arc` for shared fields.

## Exercise: Add Interactive Controls

Extend `iroh-monitor`:

1. **Connect to Peer**: Press 'c', enter endpoint ID, connect
2. **Send Test Data**: Select a connection, press 's', send test data
3. **Disconnect**: Select a connection, press 'd', close it
4. **Export Logs**: Press 'e', write connection history to file

Hints:
- Use a modal dialog for input (there are ratatui examples)
- Add a `Mode` enum: `Normal`, `Input`, `Confirm`
- Store logs in `App` as `Vec<LogEntry>`

## Python Comparison: Building a TUI

Python's `rich` library:

```python
from rich.live import Live
from rich.table import Table
import time

def generate_table():
    table = Table(title="Iroh Monitor")
    table.add_column("Peer")
    table.add_column("Type")
    # ...
    return table

with Live(generate_table(), refresh_per_second=4) as live:
    while True:
        time.sleep(0.25)
        live.update(generate_table())
```

This is simpler but less flexible. You don't get fine-grained control over rendering or input handling. For complex TUIs, Rust + ratatui is more powerful.

## What You Learned

- TUI event loop: render, read input, update state
- Layout management with constraints
- Styling with colors and modifiers
- Real-time updates with channels
- Concurrent state management with Arc<Mutex>
- Performance optimization (diff rendering, lock minimization)
- Building interactive terminal applications

Next: Deep P2P concepts. We'll understand NAT traversal, hole punching, DHT, and gossip protocols. The theory behind why iroh works.
