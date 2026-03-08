# muninn-frames

Shared frame model and protobuf codec for realtime transport boundaries.

`muninn-frames` is the wire half of the Muninn messaging stack:

- `muninn-frames` owns transport-friendly `Frame` encoding and decoding.
- `muninn-kernel` owns in-memory routing, cancellation, and backpressure.

This crate is intentionally small. It handles protobuf bytes and flexible JSON payloads, but it does not implement routing, handler registration, or transport sessions.

## Installation

```toml
[dependencies]
frames = { package = "muninn-frames", git = "https://github.com/ianzepp/muninn-frames-rs.git", tag = "v0.1.0" }
```

## Wire format

On the wire, frames are binary protobuf via [Prost](https://docs.rs/prost). The `data` field round-trips through a recursive `serde_json::Value ↔ prost_types::Value` conversion, keeping payloads flexible while the envelope is compact and typed.

### Frame fields

| Field | Type | Description |
|---|---|---|
| `id` | `String` | Unique frame identifier (UUID) |
| `parent_id` | `Option<String>` | Links response frames to the originating request |
| `ts` | `i64` | Milliseconds since Unix epoch |
| `from` | `Option<String>` | Sender identifier (user ID or system label) |
| `syscall` | `String` | Namespaced operation name, e.g. `"object:update"` |
| `status` | `Status` | Lifecycle position of the frame |
| `trace` | `Option<Value>` | Optional trace metadata, separate from business payload |
| `data` | `Value` | Arbitrary JSON payload |

### Status lifecycle

```text
Request → Item* / Bulk* → Done | Error | Cancel
```

| Status | Meaning |
|---|---|
| `Request` | Initial frame sent by the client |
| `Item` | Intermediate streaming item (non-terminal) |
| `Bulk` | Intermediate streaming batch (non-terminal) |
| `Done` | Successful terminal response |
| `Error` | Error terminal response |
| `Cancel` | Cancellation frame |

## Public API

```rust
pub fn encode_frame(frame: &Frame) -> Vec<u8>;
pub fn decode_frame(bytes: &[u8]) -> Result<Frame, CodecError>;
```

That is the whole surface. This crate is transport-focused and contains no business logic.

## Usage

```rust
use frames::{decode_frame, encode_frame, Frame, Status};

let frame = Frame {
    id: "msg-001".to_owned(),
    parent_id: None,
    ts: 1709913600000,
    from: Some("user-42".to_owned()),
    syscall: "object:create".to_owned(),
    status: Status::Request,
    trace: None,
    data: serde_json::json!({
        "type": "sticky",
        "x": 100.0,
        "y": 200.0,
        "text": "Hello"
    }),
};

let bytes = encode_frame(&frame);
let decoded = decode_frame(&bytes).expect("valid frame");
assert_eq!(decoded, frame);
```

## Relationship to muninn-kernel

`muninn-kernel::Frame` is optimized for in-memory routing:

- `Uuid` identifiers
- `HashMap<String, serde_json::Value>` payloads

`muninn-frames::Frame` is optimized for transport boundaries:

- `String` identifiers
- `serde_json::Value` payloads

Keep the crates separate. Shared conversion logic should live in a bridge crate or boundary module rather than making either crate depend directly on the other.

## Notes

- Protobuf numeric values are decoded as JSON floats, so integer consumers should accept whole-number floats (for example `2` becomes `2.0` after a round trip).
- `data` accepts arbitrary JSON. Schema enforcement belongs in the handler layer, not here.
- When bridging into `muninn-kernel`, treat non-object `data` payloads as a conversion concern rather than assuming the routing layer will accept them unchanged.
