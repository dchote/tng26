# LOB Packets

**LOB (Length-Object-Binary)** is the fundamental packet encoding format used throughout TNG. It provides a simple, efficient way to combine JSON metadata with binary payloads in a single packet.

## Format

Every LOB packet consists of three parts:

```
┌──────────────────────────────────────────────────────────────────┐
│  LENGTH (2 bytes)  │  HEAD (0-N bytes)  │  BODY (remaining)      │
└──────────────────────────────────────────────────────────────────┘
```

| Field | Size | Description |
|-------|------|-------------|
| **LENGTH** | 2 bytes | Big-endian unsigned integer, size of HEAD |
| **HEAD** | 0 to LENGTH bytes | JSON object or binary data |
| **BODY** | Remaining bytes | Binary payload |

## LENGTH Field

The LENGTH field is a 16-bit big-endian unsigned integer that specifies the size of the HEAD section:

| LENGTH Value | HEAD Interpretation |
|--------------|---------------------|
| `0` | No HEAD; packet is pure BODY |
| `1-6` | HEAD is raw binary (too short for valid JSON object) |
| `7+` | HEAD should be parsed as UTF-8 JSON object |

### Maximum Sizes

- Maximum HEAD size: 65,535 bytes (2^16 - 1)
- Maximum packet size: Determined by transport MTU
- Typical packet size: 1,200-1,400 bytes for IP transports

## HEAD Section

When LENGTH ≥ 7, the HEAD is interpreted as a UTF-8 encoded JSON object:

```json
{
  "type": "link",
  "at": 1234567890,
  "c": 42
}
```

### JSON Requirements

- Must be a JSON **object** (not array, string, number, or null)
- Must start with `{` and end with `}`
- Should follow [I-JSON](https://datatracker.ietf.org/doc/html/rfc7493) guidelines
- Keys should be unique within the object

### Binary HEAD

When LENGTH is 1-6, the HEAD contains raw binary data. This is used for:
- Compact binary headers in constrained environments
- Protocol-specific optimizations

## BODY Section

The BODY contains arbitrary binary data. Common uses:

- Encrypted payloads
- Attached packets (nested LOB)
- Raw data streams
- Public keys

The BODY size is implicit: `BODY_SIZE = PACKET_SIZE - 2 - LENGTH`

## Encoding

### Pseudocode

```python
def encode_lob(head: dict | bytes | None, body: bytes | None) -> bytes:
    if head is None:
        head_bytes = b''
    elif isinstance(head, dict):
        head_bytes = json.dumps(head, separators=(',', ':')).encode('utf-8')
    else:
        head_bytes = head
    
    body_bytes = body if body else b''
    
    length = len(head_bytes)
    if length > 65535:
        raise ValueError("HEAD too large")
    
    return struct.pack('>H', length) + head_bytes + body_bytes
```

### Example

```python
# JSON head with binary body
packet = encode_lob(
    head={"type": "message", "c": 1},
    body=b'\x00\x01\x02\x03'
)
# Result: b'\x00\x18{"type":"message","c":1}\x00\x01\x02\x03'

# Pure binary (no head)
packet = encode_lob(head=None, body=encrypted_data)
# Result: b'\x00\x00' + encrypted_data

# Head only (no body)
packet = encode_lob(head={"type": "ping"}, body=None)
# Result: b'\x00\x0f{"type":"ping"}'
```

## Decoding

### Pseudocode

```python
def decode_lob(packet: bytes) -> tuple[int, bytes | None, dict | None, bytes | None]:
    if len(packet) < 2:
        raise ValueError("Packet too short")
    
    length = struct.unpack('>H', packet[:2])[0]
    
    if length > len(packet) - 2:
        raise ValueError("LENGTH exceeds packet size")
    
    head_bytes = packet[2:2+length] if length > 0 else None
    body_bytes = packet[2+length:] if len(packet) > 2+length else None
    
    json_obj = None
    if head_bytes and length >= 7:
        try:
            json_obj = json.loads(head_bytes.decode('utf-8'))
        except (json.JSONDecodeError, UnicodeDecodeError):
            pass  # HEAD is binary, not JSON
    
    return length, head_bytes, json_obj, body_bytes
```

### Return Values

A decoder should return:

| Value | Description |
|-------|-------------|
| `head_length` | 0 to packet length - 2 |
| `head_bytes` | Raw HEAD bytes, or None if LENGTH=0 |
| `json` | Parsed JSON object, or None if not valid JSON |
| `body_bytes` | BODY bytes, or None if no BODY |

## Packet Nesting

LOB packets can be nested by placing one packet as the BODY of another:

```
Outer Packet:
┌─────────┬──────────────────────────────────────────────┐
│ LENGTH  │  HEAD: {"type":"wrapper"}                    │
├─────────┴──────────────────────────────────────────────┤
│  BODY: Inner Packet                                    │
│  ┌─────────┬───────────────────────────────────────┐   │
│  │ LENGTH  │  HEAD: {"type":"inner"}               │   │
│  ├─────────┴───────────────────────────────────────┤   │
│  │  BODY: actual payload                           │   │
│  └─────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

This pattern is used for:
- Handshake packets (encrypted inner packet)
- Routing (wrapped packets for relay)
- Protocol extensions

## CBOR Alternative

For constrained devices with limited memory, CBOR can be used instead of JSON for the HEAD section. When using CBOR:

1. Set a flag in the outer protocol layer indicating CBOR mode
2. HEAD contains CBOR-encoded map instead of JSON object
3. Parsing follows [RFC 8949](https://datatracker.ietf.org/doc/html/rfc8949)

### CBOR Benefits

- More compact encoding (especially for binary data)
- Native binary string support
- Faster parsing on embedded systems
- Deterministic encoding available

### CBOR Example

```python
import cbor2

# JSON: {"type":"message","c":1,"data":"base64..."} = ~45 bytes
# CBOR equivalent: ~25 bytes

head_cbor = cbor2.dumps({
    "type": "message",
    "c": 1,
    "data": b'\x00\x01\x02\x03'  # Native binary, no base64
})
```

## Common Packet Types

### Channel Packet

```json
HEAD: {
  "c": 42,           // Channel ID
  "seq": 100,        // Sequence number (reliable)
  "ack": 99          // Acknowledgment (reliable)
}
BODY: [channel data]
```

### Handshake Packet

```json
HEAD: {
  "type": "link",
  "at": 1234567890,  // Timestamp
  "csid": "4a"       // Cipher suite
}
BODY: [attached packet with keys]
```

### Message Packet

```json
HEAD: {
  "type": "message"
}
BODY: [encrypted payload]
```

## Wire Format Summary

```
Byte:  0     1     2     3     4     ...   N     N+1   ...
     ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
     │ LEN │ LEN │  H  │  E  │  A  │  D  │  B  │  O  │  D  │
     │ HI  │ LO  │     │     │     │     │     │     │  Y  │
     └─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
           │           │                 │
           └─ LENGTH ──┴─── HEAD ────────┴─── BODY ───────
              (2 bytes)   (LENGTH bytes)   (remaining)
```

## Error Handling

| Condition | Behavior |
|-----------|----------|
| Packet < 2 bytes | Parse error |
| LENGTH > packet size - 2 | Parse error |
| JSON parse fails (LENGTH ≥ 7) | Return raw HEAD bytes, JSON = null |
| Empty BODY | Return null/empty for BODY |

## Best Practices

1. **Minimize HEAD size**: Keep JSON keys short, omit default values
2. **Use binary BODY**: Don't base64-encode in JSON; put binary in BODY
3. **Validate early**: Check LENGTH before allocating memory
4. **Handle partial packets**: Transport layer handles framing/chunking

---

*Related: [E3X](e3x.md) | [Channels](channels.md) | [Technical Overview](../technical-overview.md)*
