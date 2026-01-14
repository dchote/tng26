# Links

A **Link** is an encrypted connection between two endpoints. It represents the fundamental unit of connectivity in TNG, built on top of an [E3X exchange](e3x.md) and providing the foundation for [channels](channels.md).

## Overview

```
        Endpoint A                           Endpoint B
    ┌──────────────┐                     ┌──────────────┐
    │   Hashname   │                     │   Hashname   │
    │   kw3a...kkq │                     │   xy9b...mnp │
    └──────┬───────┘                     └──────┬───────┘
           │                                    │
           │            ┌────────┐              │
           └────────────│  Link  │──────────────┘
                        │        │
                        │ E3X    │
                        │Exchange│
                        └────────┘
```

## Link Components

| Component | Description |
|-----------|-------------|
| **Hashname** | Remote endpoint's identity |
| **Keys** | Remote endpoint's public keys (CSKs) |
| **Paths** | Network connectivity information |
| **Exchange** | E3X session state |
| **Channels** | Active data streams |

## Link States

A link progresses through three states:

```
    ┌──────────────┐
    │  Unresolved  │  Hashname known, but missing keys or paths
    └──────┬───────┘
           │ resolve()
           ▼
    ┌──────────────┐
    │     Down     │  Keys validated, paths available, not connected
    └──────┬───────┘
           │ connect()
           ▼
    ┌──────────────┐
    │      Up      │  Handshake complete, channels available
    └──────────────┘
```

### Unresolved

- Only hashname is known
- Missing keys and/or paths
- Must resolve before connecting

### Down

- Keys are validated (hashname verified)
- At least one path is known
- No active connection

### Up

- Handshake completed
- Session keys established
- Channels can be opened

## Link Keys

Link keys are the public keys required to establish an E3X exchange:

```json
{
  "keys": {
    "4a": "base32-encoded-cs4a-csk",
    "4b": "base32-encoded-cs4b-csk",
    "3a": "base32-encoded-cs3a-csk"
  }
}
```

The highest common cipher suite is selected for the link.

## Link Paths

Paths describe how to reach the endpoint over various transports:

```json
{
  "paths": [
    {
      "type": "quic",
      "ip": "192.168.1.100",
      "port": 42424
    },
    {
      "type": "webtransport",
      "url": "https://example.com/tng"
    },
    {
      "type": "bluetooth",
      "address": "AA:BB:CC:DD:EE:FF"
    },
    {
      "type": "peer",
      "router": "router-hashname"
    }
  ]
}
```

### Path Types

| Type | Transport | Required Fields |
|------|-----------|-----------------|
| `quic` | QUIC over IP | `ip`, `port` |
| `webtransport` | WebTransport | `url` |
| `bluetooth` | Bluetooth LE | `address` |
| `peer` | Via router | `router` |
| `lora` | LoRa radio | `address`, `spreading_factor` |
| `802.15.4` | Zigbee/Thread | `pan_id`, `short_addr` |

### Path Priority

When multiple paths exist:
1. Try direct paths first (quic, webtransport, bluetooth)
2. Fall back to routed paths (peer)
3. Prefer lower-latency transports
4. Maintain multiple active paths for redundancy

## JSON Representation

A complete link can be serialized as JSON:

```json
{
  "hashname": "kw3akwcypoedvfdquuppofpujbu7rplhj3vjvmvbkvf7z3do7kkq",
  "keys": {
    "4a": "aiw4cwmhicwp4lfbsecbdwyr6ymx6xqsli..."
  },
  "paths": [
    {
      "type": "quic",
      "ip": "192.168.0.55",
      "port": 42424
    }
  ]
}
```

This format can be:
- Stored for persistence
- Shared with other endpoints
- Embedded in URIs

## Link Handshake

To establish a link, endpoints exchange handshakes:

### Handshake Packet

```json
HEAD: {
  "type": "link",
  "at": 1234567890,
  "csid": "4a"
}
BODY: [attached packet]
```

The attached packet contains:

```json
HEAD: {
  "3a": "intermediate-hash-base32",
  "4b": "intermediate-hash-base32"
}
BODY: [CSK for negotiated cipher suite]
```

### Handshake Flow

```
    Initiator                              Responder
        │                                      │
        │  ── Handshake (at=T1) ────────────► │
        │                                      │
        │  ◄── Handshake (at=T2) ────────────  │
        │                                      │
        │        (if T2 > T1, initiator wins) │
        │                                      │
       ════════════ Link Up ═══════════════════
```

### `at` Resolution

When both endpoints initiate simultaneously:
- Higher `at` value determines the initiator
- Ensures deterministic role selection

## Multi-Path Support

Links can operate over multiple paths simultaneously:

```
                    ┌─────────────┐
              ┌─────│    Link     │─────┐
              │     └─────────────┘     │
              │                         │
       ┌──────▼──────┐          ┌───────▼─────┐
       │  Path: QUIC │          │ Path: BLE   │
       │  1.2.3.4    │          │ AA:BB:CC... │
       └─────────────┘          └─────────────┘
```

### Path Selection

```python
class Link:
    def select_path(self, packet_size: int) -> Path:
        # Filter by capability
        valid = [p for p in self.paths if p.mtu >= packet_size]
        
        # Prefer by latency
        valid.sort(key=lambda p: p.latency)
        
        return valid[0] if valid else None
```

### Path Failover

If a path fails:
1. Mark path as unavailable
2. Switch to next best path
3. Attempt reconnection on failed path
4. Notify application of path change

## Link Lifecycle

```python
class Link:
    def __init__(self, hashname: str):
        self.hashname = hashname
        self.state = "unresolved"
        self.keys = {}
        self.paths = []
        self.exchange = None
        self.channels = {}
    
    def add_keys(self, keys: dict):
        """Add public keys, verify hashname"""
        self.keys.update(keys)
        if self._verify_hashname():
            self.state = "down"
    
    def add_path(self, path: dict):
        """Add connectivity path"""
        self.paths.append(path)
        if self.keys and self.state == "unresolved":
            self.state = "down"
    
    def connect(self):
        """Initiate connection"""
        if self.state != "down":
            raise InvalidState()
        
        self.exchange = Exchange(self.local_keys, self.keys)
        handshake = self.exchange.handshake(time.time())
        self._send_to_paths(handshake)
    
    def receive_handshake(self, handshake: bytes, path: Path):
        """Process incoming handshake"""
        if self.exchange.sync(handshake):
            self.state = "up"
            self.active_path = path
            self._on_link_up()
```

## DID-Based Identity (Optional)

Links can optionally bind to W3C Decentralized Identifiers:

```json
{
  "hashname": "kw3a...kkq",
  "did": "did:th:kw3a...kkq",
  "keys": {...},
  "paths": [...]
}
```

This enables:
- Integration with DID ecosystems
- Verifiable Credential presentation over links
- Higher-level identity associations

## Link Events

Applications receive link events:

| Event | Description |
|-------|-------------|
| `link_up` | Handshake complete, link active |
| `link_down` | Connection lost |
| `path_added` | New path discovered |
| `path_removed` | Path no longer available |
| `channel_open` | New channel requested |

## URI Representation

Links can be shared via URIs:

```
tng://kw3a...kkq?cs4a=base32key&ip=1.2.3.4&port=42424
```

See [URI specification](../transports/README.md#uri-format) for details.

## Security Considerations

### Key Verification

Always verify that keys match the claimed hashname:

```python
def verify_link(hashname: str, keys: dict) -> bool:
    computed = generate_hashname(keys)
    return computed == hashname
```

### Path Trust

- Direct paths reveal your IP to the peer
- Routed paths reveal your hashname to the router
- Consider privacy implications when sharing paths

### Link Acceptance

- Never auto-accept links from unknown hashnames
- Application must authorize new links
- Consider rate limiting connection attempts

---

*Related: [E3X](e3x.md) | [Channels](channels.md) | [Mesh](mesh.md) | [Routing](routing.md)*
