# Part 09: Embedded and Mobile

Your laptop is not the universe. Iroh can run on devices with 4MB of RAM (ESP32), in your pocket (mobile phones), and on tiny Linux computers (Raspberry Pi). Each environment has unique constraints and opportunities.

## The Spectrum of Hardware

```
┌─────────────┬──────────┬──────────┬──────────┬─────────┐
│ ESP32       │ Pi Zero  │ Pi 4     │ Phone    │ Laptop  │
├─────────────┼──────────┼──────────┼──────────┼─────────┤
│ 4MB RAM     │ 512MB    │ 4GB      │ 6GB      │ 16GB    │
│ 160MHz CPU  │ 1GHz     │ 1.5GHz   │ 2GHz     │ 3GHz    │
│ No OS       │ Linux    │ Linux    │ Android  │ Linux   │
│ WiFi only   │ WiFi     │ WiFi/Eth │ 4G/5G    │ WiFi    │
└─────────────┴──────────┴──────────┴──────────┴─────────┘
```

Iroh's full feature set works on Pi and above. On ESP32, you'll need to trim features. On mobile, network transitions (WiFi to cellular) are common.

## Part 1: Raspberry Pi (Easy Mode)

The Pi runs Linux. If your code works on your laptop, it probably works on Pi. But there are gotchas.

### Cross-Compilation

Pi uses ARM architecture. Your laptop probably uses x86_64. Cross-compile:

```bash
# Install cross-compilation toolchain
rustup target add armv7-unknown-linux-gnueabihf  # Pi 2/3/4 (32-bit)
# Or: aarch64-unknown-linux-gnu                   # Pi 3/4 (64-bit)

# Install linker
sudo apt install gcc-arm-linux-gnueabihf

# Configure cargo
cat >> ~/.cargo/config.toml <<EOF
[target.armv7-unknown-linux-gnueabihf]
linker = "arm-linux-gnueabihf-gcc"
EOF

# Build
cargo build --target armv7-unknown-linux-gnueabihf --release

# Copy to Pi
scp target/armv7-unknown-linux-gnueabihf/release/your-app pi@raspberrypi.local:/home/pi/
```

Or use [`cross`](https://github.com/cross-rs/cross) (Docker-based cross-compilation):

```bash
cargo install cross
cross build --target armv7-unknown-linux-gnueabihf --release
```

Easier, but requires Docker.

### Performance Considerations

Pi 4 is ~1/10th the speed of a laptop. Pi Zero is ~1/100th. Profile your code:

```rust
use std::time::Instant;

let start = Instant::now();
// Expensive operation
let elapsed = start.elapsed();
println!("Took {:?}", elapsed);
```

Things that are instant on your laptop might take seconds on Pi Zero.

**QUIC crypto**: Ed25519 signing/verification is fast even on Pi. ChaCha20-Poly1305 encryption is efficient. You're fine.

**Hash computation**: BLAKE3 is SIMD-optimized. On Pi without SIMD, it's slower but acceptable. Hashing a 1GB file: ~20s on Pi 4, ~200s on Pi Zero.

**Network throughput**: Pi 4 has gigabit Ethernet, fast WiFi. Pi Zero has 2.4GHz WiFi only (max ~50 Mbps). Design for your Pi's network.

### Real-World Pi Application: IoT Sensor Network

```rust
// Sensor node (Pi Zero) - collects data, sends to hub
use iroh::{Endpoint, EndpointAddr};
use serde::{Serialize, Deserialize};
use tokio::time::{interval, Duration};

#[derive(Serialize, Deserialize)]
struct SensorData {
    temperature: f32,
    humidity: f32,
    timestamp: u64,
}

async fn sensor_node(hub_addr: EndpointAddr) -> anyhow::Result<()> {
    let endpoint = Endpoint::bind().await?;
    let conn = endpoint.connect(hub_addr, b"sensor/0").await?;

    let mut ticker = interval(Duration::from_secs(60));
    loop {
        ticker.tick().await;

        let data = SensorData {
            temperature: read_temperature(),
            humidity: read_humidity(),
            timestamp: std::time::SystemTime::now()
                .duration_since(std::time::UNIX_EPOCH)?
                .as_secs(),
        };

        let mut stream = conn.open_uni().await?;
        let bytes = bincode::serialize(&data)?;
        stream.write_all(&bytes).await?;
        stream.finish()?;
    }
}

fn read_temperature() -> f32 {
    // Read from GPIO sensor (DHT22, BME280, etc.)
    // Placeholder
    22.5
}

fn read_humidity() -> f32 {
    50.0
}
```

This runs continuously, sending data to a hub every minute. QUIC connection stays alive (or reconnects if it drops).

Python comparison: You'd use `aiohttp` or plain sockets. But you'd need to handle reconnection yourself, add TLS manually, and deal with Python's startup time (~1s on Pi Zero, vs ~10ms for Rust).

## Part 2: ESP32 (Hard Mode)

ESP32 is a microcontroller: no OS, limited memory, bare metal. Rust works via [`esp-idf`](https://github.com/esp-rs/esp-idf-hal) (using FreeRTOS under the hood) or [`esp-hal`](https://github.com/esp-rs/esp-hal) (bare metal).

### The Constraints

- **RAM**: 4MB total, ~200KB available for your app (rest used by WiFi stack, FreeRTOS)
- **Storage**: 4MB flash
- **No standard library**: `no_std` environment (or limited `std` via esp-idf)
- **Single core**: Or dual-core with limited threading

Iroh's full stack won't fit. You'll need to:
1. Disable features (no relay, no discovery)
2. Use a minimal QUIC implementation or raw UDP
3. Offload heavy work to a gateway

### Minimal Iroh on ESP32

This is experimental. Iroh isn't officially tested on ESP32, but here's the approach:

```bash
# Install esp toolchain
cargo install espup
espup install
. ~/export-esp.sh

# Create project
cargo generate esp-rs/esp-idf-template cargo
cd your-project

# Add minimal dependencies
# Note: You'll need to use a stripped-down version of iroh or just quinn
```

```rust
// src/main.rs
#![no_std]
#![no_main]

use esp_idf_hal::prelude::*;
use esp_idf_svc::wifi::*;
use esp_idf_sys as _;

#[no_mangle]
fn main() {
    esp_idf_sys::link_patches();

    // Initialize WiFi
    let peripherals = Peripherals::take().unwrap();
    let sys_loop = EspSystemEventLoop::take().unwrap();

    let mut wifi = EspWifi::new(
        peripherals.modem,
        sys_loop.clone(),
        None,
    ).unwrap();

    wifi.set_configuration(&Configuration::Client(ClientConfiguration {
        ssid: "your-ssid".into(),
        password: "your-password".into(),
        ..Default::default()
    })).unwrap();

    wifi.start().unwrap();
    wifi.connect().unwrap();

    // Now you have network access
    // Use quinn or a minimal UDP socket to communicate
    // Full iroh might be too heavy
}
```

For a real application, consider:
- ESP32 as data collector only
- Gateway (Pi or server) runs full iroh
- ESP32 sends data to gateway via MQTT or HTTP
- Gateway bridges to iroh network

This is more practical than squeezing iroh onto ESP32.

### Example: ESP32 as Edge Device

```rust
// ESP32 sends temperature data to a gateway
use embedded_svc::wifi::*;
use esp_idf_svc::{eventloop::EspSystemEventLoop, nvs::EspDefaultNvsPartition, wifi::*};
use std::time::Duration;

// Note: This uses esp-idf which gives you a limited std
fn main() -> anyhow::Result<()> {
    esp_idf_sys::link_patches();

    let peripherals = Peripherals::take().unwrap();
    let sys_loop = EspSystemEventLoop::take()?;
    let nvs = EspDefaultNvsPartition::take()?;

    let mut wifi = BlockingWifi::wrap(
        EspWifi::new(peripherals.modem, sys_loop.clone(), Some(nvs))?,
        sys_loop,
    )?;

    wifi.set_configuration(&Configuration::Client(ClientConfiguration::default()))?;
    wifi.start()?;
    wifi.connect()?;
    wifi.wait_netif_up()?;

    // Connected to WiFi, send data to gateway
    loop {
        let temp = read_sensor();
        send_to_gateway(temp)?;
        std::thread::sleep(Duration::from_secs(60));
    }
}

fn send_to_gateway(temp: f32) -> anyhow::Result<()> {
    // Use HTTP POST or UDP to send to a gateway running iroh
    // Gateway bridges to P2P network
    Ok(())
}
```

The gateway acts as a bridge:

```rust
// Gateway (Pi or server)
async fn gateway() -> anyhow::Result<()> {
    let endpoint = Endpoint::bind().await?;

    // Accept HTTP/MQTT from ESP32 devices
    let listener = tokio::net::TcpListener::bind("0.0.0.0:8080").await?;

    while let Ok((stream, _)) = listener.accept().await {
        let endpoint = endpoint.clone();
        tokio::spawn(async move {
            // Read data from ESP32
            let data = read_data_from_stream(stream).await?;

            // Forward to iroh peer
            let conn = endpoint.connect(peer_addr, b"sensor/0").await?;
            let mut s = conn.open_uni().await?;
            s.write_all(&data).await?;
            s.finish()?;

            Ok::<(), anyhow::Error>(())
        });
    }

    Ok(())
}
```

This architecture is more practical than running full iroh on ESP32.

## Part 3: Mobile (iOS and Android)

Mobile adds complexity: frequent network changes, background restrictions, battery concerns.

### Iroh on Mobile: Approach

Rust core + native UI (Swift for iOS, Kotlin for Android). Use FFI to call Rust from native code.

### Building Rust for Mobile

**iOS**:

```bash
# Install targets
rustup target add aarch64-apple-ios      # iPhone/iPad (ARM)
rustup target add x86_64-apple-ios       # Simulator

# Build
cargo build --target aarch64-apple-ios --release

# Create static library
# In Cargo.toml:
[lib]
crate-type = ["staticlib"]

# This produces: target/aarch64-apple-ios/release/libyour_app.a
```

**Android**:

```bash
# Install targets
rustup target add aarch64-linux-android  # ARM64
rustup target add armv7-linux-androideabi  # ARMv7
rustup target add i686-linux-android  # x86 (emulator)
rustup target add x86_64-linux-android  # x86_64 (emulator)

# Install NDK and configure linker
# See: https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-21-rust-on-android.html

# Build
cargo build --target aarch64-linux-android --release

# For easier Android integration, use: https://github.com/mozilla/rust-android-gradle
```

### FFI Interface

Expose Rust functions to native code:

```rust
// src/lib.rs
use std::ffi::{CStr, CString};
use std::os::raw::c_char;
use iroh::{Endpoint, EndpointId};
use std::sync::Arc;
use tokio::runtime::Runtime;

// Global runtime (required for async)
static mut RUNTIME: Option<Runtime> = None;

fn get_runtime() -> &'static Runtime {
    unsafe {
        RUNTIME.get_or_insert_with(|| Runtime::new().unwrap())
    }
}

#[repr(C)]
pub struct IrohEndpoint {
    inner: *mut Endpoint,
}

#[no_mangle]
pub extern "C" fn iroh_create_endpoint() -> *mut IrohEndpoint {
    let rt = get_runtime();
    let endpoint = rt.block_on(async {
        Endpoint::bind().await.ok()
    });

    if let Some(ep) = endpoint {
        let boxed = Box::new(IrohEndpoint {
            inner: Box::into_raw(Box::new(ep)),
        });
        Box::into_raw(boxed)
    } else {
        std::ptr::null_mut()
    }
}

#[no_mangle]
pub extern "C" fn iroh_endpoint_id(endpoint: *const IrohEndpoint) -> *mut c_char {
    if endpoint.is_null() {
        return std::ptr::null_mut();
    }

    unsafe {
        let ep = &*(*endpoint).inner;
        let id = ep.id().to_string();
        CString::new(id).unwrap().into_raw()
    }
}

#[no_mangle]
pub extern "C" fn iroh_free_string(s: *mut c_char) {
    if !s.is_null() {
        unsafe { CString::from_raw(s) };
    }
}

#[no_mangle]
pub extern "C" fn iroh_destroy_endpoint(endpoint: *mut IrohEndpoint) {
    if !endpoint.is_null() {
        unsafe {
            Box::from_raw((*endpoint).inner);
            Box::from_raw(endpoint);
        }
    }
}
```

This gives you a C API callable from Swift or Kotlin.

### Swift Integration (iOS)

```swift
// IrohBridge.swift
import Foundation

class IrohEndpoint {
    private var handle: OpaquePointer?

    init() {
        handle = iroh_create_endpoint()
    }

    deinit {
        if let h = handle {
            iroh_destroy_endpoint(h)
        }
    }

    var endpointId: String {
        guard let h = handle else { return "" }
        guard let cStr = iroh_endpoint_id(h) else { return "" }
        let str = String(cString: cStr)
        iroh_free_string(UnsafeMutablePointer(mutating: cStr))
        return str
    }
}

// Usage in SwiftUI
struct ContentView: View {
    @State private var endpoint = IrohEndpoint()

    var body: some View {
        VStack {
            Text("Endpoint ID:")
            Text(endpoint.endpointId)
                .font(.caption)
        }
    }
}
```

### Kotlin Integration (Android)

```kotlin
// IrohBridge.kt
class IrohEndpoint {
    private var handle: Long = 0

    init {
        System.loadLibrary("your_app")
        handle = createEndpoint()
    }

    fun endpointId(): String {
        val cStr = getEndpointId(handle)
        val str = cStringToString(cStr)
        freeString(cStr)
        return str
    }

    protected fun finalize() {
        destroyEndpoint(handle)
    }

    private external fun createEndpoint(): Long
    private external fun getEndpointId(handle: Long): Long
    private external fun freeString(ptr: Long)
    private external fun destroyEndpoint(handle: Long)

    private fun cStringToString(ptr: Long): String {
        // JNI helper to convert C string to Java String
        // ...
    }
}

// Usage in Compose
@Composable
fun IrohScreen() {
    val endpoint = remember { IrohEndpoint() }
    Column {
        Text("Endpoint ID:")
        Text(endpoint.endpointId(), fontSize = 12.sp)
    }
}
```

### Mobile-Specific Challenges

#### Background Restrictions

Mobile OSes suspend apps in the background. Your iroh endpoint can't maintain connections when suspended.

**Solution**: Use push notifications to wake the app when data arrives. Or keep connections in a background service (Android) or background task (iOS, time-limited).

#### Network Transitions

WiFi to cellular is common. QUIC's connection migration helps, but you need to handle network state changes:

```rust
// Monitor network state (platform-specific APIs)
// When network changes:
endpoint.rebind_to_new_network().await?;
```

On mobile, you'd call this from native code when the OS notifies you of network changes.

#### Battery Life

Maintaining persistent connections drains battery. Consider:
- Only connect when needed
- Use push notifications for incoming messages (relay can queue)
- Batch operations (send multiple files at once, not one-by-one)

#### Mobile Data Costs

Users pay for cellular data. Prefer WiFi, or let users configure data usage:

```rust
if on_cellular_network() && !user_allowed_cellular {
    return Err("Waiting for WiFi");
}
```

## Part 4: Performance on Constrained Devices

### Memory Profiling

Use `heaptrack` (Linux) or `instruments` (macOS) to profile heap usage:

```bash
heaptrack ./your-app
heaptrack --analyze heaptrack.your-app.*.gz
```

Iroh's baseline: ~10MB heap for an endpoint with a few connections. Each connection adds ~100KB. Each active stream adds ~32KB (buffer size).

On a Pi with 1GB RAM, this is fine. On ESP32 with 200KB available, it's impossible (you'd need 50x more RAM).

### Binary Size

Rust binaries are large by default. For embedded:

```toml
[profile.release]
opt-level = "z"     # Optimize for size
lto = true          # Link-time optimization
codegen-units = 1   # Better optimization
panic = "abort"     # No unwinding
strip = true        # Remove symbols
```

This reduces binary from ~10MB to ~2MB. Still too big for ESP32 (which has 4MB flash total), but fine for Pi.

### CPU Usage

QUIC crypto is the main CPU cost. On Pi 4, encrypting at 100 Mbps uses ~20% CPU. On Pi Zero, same throughput uses ~80% CPU.

If you need higher throughput on weak hardware, consider:
- Hardware crypto acceleration (some ARM chips have it)
- Compression (trade CPU for bandwidth)
- Lower encryption strength (not recommended, but possible)

## Exercise: Build an IoT Dashboard

Create a system:
- Multiple Pis (or simulated nodes) collect sensor data
- One Pi acts as hub, runs iroh endpoint
- Web dashboard (from Part 08) displays real-time data
- Mobile app (iOS or Android) can view data on the go

**Bonus**: Add alerting (if temperature > threshold, push notification to mobile).

## What You Learned

- Raspberry Pi runs iroh with minor adjustments (cross-compilation, performance awareness)
- ESP32 requires significant cuts (use as edge device with gateway)
- Mobile requires FFI bridge (Rust core + native UI)
- Mobile challenges: background restrictions, network transitions, battery life
- Memory and binary size optimizations for constrained devices
- Architecture pattern: thin edge devices + gateway + full P2P network

Next: Contributing to iroh. Reading the codebase, finding bugs, writing good PRs, and joining the community.
