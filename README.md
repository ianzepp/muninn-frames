# frames

Shared realtime frame model and protobuf codec for WebSocket transport.

All client-server communication flows through a single WebSocket carrying binary protobuf frames. Each frame has a `syscall` name (e.g. `board:join`, `object:create`, `ai:prompt`) and the server routes it to a handler function. Handlers never touch the socket directly — they return an outcome enum and a single dispatch layer decides who receives what.

## Wire format

On the wire, frames are binary protobuf via [Prost](https://docs.rs/prost). The `data` field round-trips through a recursive `serde_json::Value ↔ prost_types::Value` conversion, keeping payloads flexible while the envelope is compact and typed.

### Frame fields

| Field | Type | Description |
|---|---|---|
| `id` | `String` | Unique frame identifier (UUID) |
| `parent_id` | `Option<String>` | Links response frames to the originating request (enables tree-structured traces) |
| `ts` | `i64` | Milliseconds since Unix epoch |
| `from` | `Option<String>` | Sender identifier (user ID or system label) |
| `syscall` | `String` | Namespaced operation name, e.g. `"object:update"` |
| `status` | `Status` | Lifecycle position of the frame |
| `trace` | `Option<Value>` | Optional trace metadata, carried separately from business payload |
| `data` | `Value` | Arbitrary JSON payload (`serde_json::Value`) |

### Status lifecycle

```
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

`Item` enables streaming responses — for example, a `board:join` can stream all existing objects as individual items before a final `Done` with the count. `Bulk` works the same way but carries a batch of items per frame.

### Request → Done span pairing

Two frames form a logical span when:

1. They share the same `parent_id`
2. They have the same `syscall`
3. One has status `Request`, the other has status `Done` or `Error`

This makes frames suitable for structured tracing and observability without additional infrastructure.

## Public API

```rust
// Encode a frame into protobuf bytes (for sending over WebSocket)
pub fn encode_frame(frame: &Frame) -> Vec<u8>;

// Decode protobuf bytes back into a frame
pub fn decode_frame(bytes: &[u8]) -> Result<Frame, CodecError>;
```

That's the entire surface. This crate is transport-focused and contains no business logic.

## Usage

```rust
use frames::{Frame, Status, encode_frame, decode_frame};

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

// Encode to binary for WebSocket transport
let bytes = encode_frame(&frame);

// Decode on the other side
let decoded = decode_frame(&bytes).expect("valid frame");
assert_eq!(decoded, frame);
```

## Notes

- Protobuf numeric values are decoded as JSON floats, so integer consumers should accept whole-number floats (e.g. `2` becomes `2.0` after a round trip).
- HTTP/auth endpoints can continue using JSON outside this crate — frames are specifically for the WebSocket transport layer.
- The `data` field accepts arbitrary JSON. Schema enforcement, if needed, belongs in the handler layer, not here.
