# muninn-frames

Shared frame model and protobuf codec for realtime transport boundaries.

`muninn-frames` is the wire half of the Muninn messaging stack:

- **`muninn-frames`** — transport-friendly `Frame` type, protobuf encoding and decoding
- **`muninn-kernel`** — in-memory routing, handler registration, cancellation, backpressure

This crate is intentionally minimal. It handles protobuf bytes and flexible JSON payloads, but contains no routing, handler registration, or transport session logic.

## Installation

Add to your `Cargo.toml` as a git dependency:

```toml
[dependencies]
frames = { package = "muninn-frames", git = "https://github.com/ianzepp/muninn-frames.git" }
```

Or with the full package name in `use` statements:

```toml
[dependencies]
muninn-frames = { git = "https://github.com/ianzepp/muninn-frames.git" }
```

Pin to a specific tag or commit:

```toml
[dependencies]
frames = { package = "muninn-frames", git = "https://github.com/ianzepp/muninn-frames.git", tag = "v0.1.0" }
```

## Public API

The entire public surface is two functions and two types:

```rust
pub fn encode_frame(frame: &Frame) -> Vec<u8>;
pub fn decode_frame(bytes: &[u8]) -> Result<Frame, CodecError>;

pub struct Frame { ... }
pub enum Status { Request, Item, Bulk, Done, Error, Cancel }
pub enum CodecError { Decode(prost::DecodeError), InvalidStatus(i32) }
```

## Frame

`Frame` is the wire envelope for every request and response:

```rust
pub struct Frame {
    pub id: String,                    // Unique identifier (typically a UUID string)
    pub parent_id: Option<String>,     // Links responses to their originating request
    pub ts: i64,                       // Milliseconds since Unix epoch
    pub from: Option<String>,          // Sender identifier (user ID, system label, etc.)
    pub syscall: String,               // Namespaced operation, e.g. "object:create"
    pub status: Status,                // Lifecycle position (see below)
    pub trace: Option<serde_json::Value>,  // Optional observability metadata
    pub data: serde_json::Value,       // Arbitrary JSON payload
}
```

`Frame` derives `Clone`, `Debug`, `PartialEq`, `Serialize`, and `Deserialize`, so it can be used with serde directly (e.g. for logging or REST adapters) as well as with the protobuf codec.

### Status Lifecycle

```
Request  →  Item* / Bulk*  →  Done | Error | Cancel
```

| Status | Wire value | Meaning |
|---|---|---|
| `Request` | 0 | Initial frame sent by the client |
| `Item` | 4 | Intermediate streaming result (non-terminal) |
| `Bulk` | 5 | Intermediate streaming batch (non-terminal) |
| `Done` | 1 | Successful terminal response |
| `Error` | 2 | Error terminal response |
| `Cancel` | 3 | Cancellation signal |

`Item` and `Bulk` frames are non-terminal — more frames follow. `Done`, `Error`, and `Cancel` are terminal — the stream is closed after one of these.

In JSON serialization, `Status` is a lowercase string: `"request"`, `"item"`, `"bulk"`, `"done"`, `"error"`, `"cancel"`.

## Wire Format

Frames are encoded as binary protobuf using [Prost](https://docs.rs/prost). The `data` and `trace` fields round-trip through a recursive `serde_json::Value ↔ prost_types::Value` conversion, keeping payloads flexible and schema-free while the envelope stays compact and typed.

**Encoding is infallible.** `encode_frame` always returns a `Vec<u8>`.

**Decoding returns errors only for protocol violations:**
- `CodecError::Decode` — the bytes are not valid protobuf
- `CodecError::InvalidStatus` — the status integer is outside the valid range (0–5)

No validation is performed on business fields (ids, syscalls, etc.) — that is the caller's responsibility.

## Usage

### Basic Encode/Decode

```rust
use frames::{decode_frame, encode_frame, Frame, Status};
use serde_json::json;

let frame = Frame {
    id: "550e8400-e29b-41d4-a716-446655440000".to_owned(),
    parent_id: None,
    ts: 1709913600000,
    from: Some("user-42".to_owned()),
    syscall: "object:create".to_owned(),
    status: Status::Request,
    trace: None,
    data: json!({
        "type": "sticky",
        "x": 100.0,
        "y": 200.0,
        "text": "Hello"
    }),
};

// Encode to bytes for transmission over WebSocket or TCP
let bytes = encode_frame(&frame);

// Decode received bytes back to a Frame
let decoded = decode_frame(&bytes).expect("valid frame");
assert_eq!(decoded, frame);
```

### WebSocket Gateway

```rust
use frames::{decode_frame, encode_frame, Frame, Status};

// Incoming bytes from WebSocket → decode to Frame
async fn on_message(raw: Vec<u8>) -> Result<(), Box<dyn std::error::Error>> {
    let frame = decode_frame(&raw)?;
    // forward to muninn-kernel or handle locally
    Ok(())
}

// Outgoing Frame → encode to bytes for WebSocket
async fn send_frame(ws: &mut WebSocket, frame: &Frame) {
    let bytes = encode_frame(frame);
    ws.send(bytes.into()).await.ok();
}
```

### JSON Logging

Because `Frame` derives `Serialize`/`Deserialize`, it can be logged as JSON directly:

```rust
let frame = decode_frame(&raw)?;
tracing::debug!(frame = %serde_json::to_string(&frame)?, "received frame");
```

### Constructing Response Frames

Response frames set `parent_id` to link them to the originating request:

```rust
use frames::{Frame, Status};
use serde_json::json;

let request = decode_frame(&incoming_bytes)?;

// Build a streaming item response
let item = Frame {
    id: uuid::Uuid::new_v4().to_string(),
    parent_id: Some(request.id.clone()),
    ts: now_millis(),
    from: Some("server".to_owned()),
    syscall: request.syscall.clone(),
    status: Status::Item,
    trace: request.trace.clone(),
    data: json!({ "row": 1, "value": "foo" }),
};

// Build a terminal done response
let done = Frame {
    id: uuid::Uuid::new_v4().to_string(),
    parent_id: Some(request.id.clone()),
    ts: now_millis(),
    from: Some("server".to_owned()),
    syscall: request.syscall.clone(),
    status: Status::Done,
    trace: request.trace.clone(),
    data: json!({}),
};
```

## Relationship to muninn-kernel

The two crates use different `Frame` types optimized for their respective concerns:

| | `muninn-frames::Frame` | `muninn-kernel::Frame` |
|---|---|---|
| IDs | `String` | `Uuid` |
| Payload | `serde_json::Value` | `HashMap<String, serde_json::Value>` |
| Purpose | Compact wire transport | Fast in-memory routing |

Keep the crates separate and convert between them at the gateway boundary. If multiple projects share the same conversion logic, a small bridge crate is the right home for it — neither library should depend on the other.

A typical bridge looks like:

```rust
// In your gateway or bridge crate:

fn kernel_frame_from_wire(wire: frames::Frame) -> kernel::Frame {
    kernel::Frame {
        id: uuid::Uuid::parse_str(&wire.id).unwrap_or_else(|_| Uuid::new_v4()),
        parent_id: wire.parent_id.and_then(|s| Uuid::parse_str(&s).ok()),
        ts: wire.ts,
        from: wire.from,
        syscall: wire.syscall,
        status: map_status(wire.status),
        trace: wire.trace.unwrap_or(serde_json::Value::Null),
        data: wire.data.as_object()
            .cloned()
            .unwrap_or_default()
            .into_iter()
            .collect(),
    }
}
```

## Notes

- **Integer precision:** Protobuf encodes all numbers as `f64`. Integer JSON values round-trip as whole-number floats (`2` becomes `2.0`). Consumers should accept whole-number floats wherever integers are expected.
- **No schema enforcement:** The `data` field accepts any JSON value. Validation belongs in the handler layer, not here.
- **Non-object payloads:** The `data` field is a `serde_json::Value`, so it can hold any JSON (string, number, array, object, null). When bridging into `muninn-kernel`, non-object payloads require explicit conversion since the kernel expects `HashMap<String, Value>`.
- **Encoding never panics:** `encode_frame` is infallible. Decoding is the only fallible operation.
- **`trace` is separate from `data`:** Keep observability metadata in `trace` rather than `data` to avoid polluting business payloads. The kernel automatically propagates `trace` from request to response frames.

## Status

The API is small and early-stage. Pin to a tag or revision rather than tracking a moving branch.
