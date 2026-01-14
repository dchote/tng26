# Channels

**Channels** are multiplexed, bidirectional data streams within an E3X session. They allow multiple concurrent conversations over a single encrypted link, each with its own reliability and ordering guarantees.

## Overview

```
┌─────────────────────────────────────────────────────────────┐
│                          Link                               │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │
│  │Channel 1 │  │Channel 2 │  │Channel 3 │  │Channel N │    │
│  │(reliable)│  │(unreliab)│  │(reliable)│  │   ...    │    │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │
├─────────────────────────────────────────────────────────────┤
│                     E3X Session                             │
└─────────────────────────────────────────────────────────────┘
```

## Channel Types

| Type | Ordering | Delivery | Use Case |
|------|----------|----------|----------|
| **Reliable** | Ordered | Guaranteed | File transfer, streams, RPC |
| **Unreliable** | Unordered | Best-effort | Real-time data, gaming, voice |

## Channel Identifier

Each channel has a unique **Channel ID (CID)**, a 32-bit unsigned integer:

| Initiator | Responder |
|-----------|-----------|
| Odd CIDs (1, 3, 5, ...) | Even CIDs (2, 4, 6, ...) |

This allocation prevents ID collisions without coordination.

### CID Assignment

```python
class ChannelManager:
    def __init__(self, is_initiator: bool):
        self.next_cid = 1 if is_initiator else 2
        self.step = 2
    
    def allocate(self) -> int:
        cid = self.next_cid
        self.next_cid += self.step
        return cid
```

## Channel Packet Format

Channel packets are [LOB encoded](lob.md) with specific header fields:

### Reliable Channel Packet

```json
HEAD: {
  "c": 42,          // Channel ID (required)
  "seq": 100,       // Sequence number
  "ack": 99,        // Acknowledgment
  "miss": [97, 98], // Missing sequences (optional)
  "end": true       // End of channel (optional)
}
BODY: [payload data]
```

### Unreliable Channel Packet

```json
HEAD: {
  "c": 43,          // Channel ID (required)
  "seq": 50         // Sequence for dedup (optional)
}
BODY: [payload data]
```

## Reliable Channels

Reliable channels guarantee ordered, complete delivery using a sliding window protocol.

### Sequence Numbers

- Start at 0 for each channel
- Increment for each packet sent
- 32-bit unsigned, wrap at 2^32

### Acknowledgments

The `ack` field indicates the highest contiguous sequence received:

```
Sender                              Receiver
   │                                    │
   │ ──── seq=0 ─────────────────────► │
   │ ──── seq=1 ─────────────────────► │
   │ ──── seq=2 ─────────────────────► │
   │                                    │
   │ ◄──── ack=2 ────────────────────  │  All received up to 2
```

### Missing Packets

The `miss` array lists sequences that were skipped:

```
Sender                              Receiver
   │                                    │
   │ ──── seq=0 ─────────────────────► │
   │ ──── seq=1 ─────X (lost)          │
   │ ──── seq=2 ─────────────────────► │
   │                                    │
   │ ◄──── ack=0, miss=[1] ──────────  │  Need seq 1
   │                                    │
   │ ──── seq=1 (retransmit) ────────► │
   │                                    │
   │ ◄──── ack=2 ────────────────────  │  All caught up
```

### Flow Control

A sliding window limits outstanding packets:

```python
class ReliableChannel:
    WINDOW_SIZE = 32  # Max unacked packets
    
    def __init__(self):
        self.send_seq = 0
        self.send_acked = 0
        self.recv_ack = 0
        self.recv_buffer = {}
    
    def can_send(self) -> bool:
        return self.send_seq - self.send_acked < self.WINDOW_SIZE
    
    def send(self, data: bytes) -> dict:
        if not self.can_send():
            raise WindowFull()
        
        packet = {
            "c": self.cid,
            "seq": self.send_seq,
            "ack": self.recv_ack
        }
        self.send_seq += 1
        return packet, data
```

### Retransmission

Packets are retransmitted when:
1. Listed in `miss` array
2. Timeout expires without acknowledgment

```python
RETRANSMIT_TIMEOUT = 1.0  # seconds
MAX_RETRIES = 5

def check_retransmit(self, now: float):
    for seq, (packet, sent_at, retries) in self.unacked.items():
        if now - sent_at > RETRANSMIT_TIMEOUT:
            if retries >= MAX_RETRIES:
                self.fail_channel()
            else:
                self.retransmit(seq)
```

## Unreliable Channels

Unreliable channels provide best-effort delivery without retransmission.

### Properties

- No acknowledgments
- No ordering guarantees
- Duplicates possible (use `seq` for deduplication)
- Lower latency than reliable

### Optional Sequence Numbers

Even unreliable channels can include `seq` for:
- Duplicate detection
- Out-of-order detection (application decides what to do)
- Statistics/debugging

```python
class UnreliableChannel:
    def __init__(self):
        self.send_seq = 0
        self.seen_seqs = set()  # For dedup
    
    def send(self, data: bytes) -> dict:
        packet = {"c": self.cid, "seq": self.send_seq}
        self.send_seq += 1
        return packet, data
    
    def receive(self, packet: dict, data: bytes):
        seq = packet.get("seq")
        if seq is not None:
            if seq in self.seen_seqs:
                return None  # Duplicate
            self.seen_seqs.add(seq)
        return data
```

## Channel Lifecycle

```
    ┌──────────────┐
    │   Opening    │  Open packet sent
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │     Open     │  Both sides exchanging data
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │   Closing    │  End packet sent
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │    Closed    │  Cleanup complete
    └──────────────┘
```

### Opening a Channel

The first packet on a new CID is the **open** packet:

```json
HEAD: {
  "c": 1,
  "type": "stream",     // Channel type identifier
  "seq": 0
}
BODY: [initial data, optional]
```

### Closing a Channel

Set `"end": true` on the final packet:

```json
HEAD: {
  "c": 1,
  "seq": 100,
  "ack": 99,
  "end": true
}
BODY: [final data, optional]
```

Both sides should send `end` for graceful close.

## Standard Channel Types

TNG defines several built-in channel types:

| Type | Reliability | Purpose |
|------|-------------|---------|
| `stream` | Reliable | Generic byte stream |
| `path` | Reliable | Path/connectivity negotiation |
| `peer` | Reliable | Routing requests |
| `connect` | Reliable | Connection assistance |
| `route_info` | Reliable | Gateway route propagation (Advanced) |
| `route_query` | Reliable | Gateway route solicitation (Advanced) |

### Routing Channels and Hop Limit

The `peer` and `connect` channels used for [routing](routing.md) include a `hops` field for loop prevention:

```json
HEAD: {
  "c": 1,
  "type": "peer",
  "seq": 0,
  "hops": 16      // Hop limit (required for routing channels)
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `hops` | u8 | Yes (routing) | Remaining hop count, decremented by each router |

**Hop Limit Rules:**

1. **Originator** sets `hops` to initial value (default: 16, max: 64)
2. **Router** decrements `hops` before forwarding to next hop
3. **Router** discards packet silently if `hops` reaches 0
4. **Receiver** processes packet regardless of `hops` value

```
Routing example with hop limit:

  A ──────────► R1 ──────────► R2 ──────────► B
    peer         peer           connect
    hops=16      hops=15        hops=14
```

This prevents packets from circulating indefinitely in case of routing loops. See [Loop Prevention](routing.md#loop-prevention) for details.

### Gateway Discovery Channels — Advanced

These channel types are used for exit gateway discovery and route propagation:

#### `route_info` Channel

Used to propagate gateway route information through the mesh:

```json
HEAD: {
  "c": 123,
  "type": "route_info",
  "seq": 0
}
BODY: {
  "gateway": "<gateway hashname>",
  "seqno": <32-bit sequence number>,
  "hops": <current hop count>,
  "via": "<peer hashname>",
  "capabilities": ["ipv4", "ipv6", "nat", "dns"],
  "prefixes": [
    {
      "prefix": "0.0.0.0/0",
      "type": "ipv4",
      "metric": {...}
    }
  ],
  "ttl": 300,
  "timestamp": <unix-seconds>
}
```

**Properties:**
- **Reliability**: Reliable (ordered, acknowledged)
- **Propagation**: Sent to directly linked peers, optionally re-propagated by routers
- **Loop prevention**: Uses sequence numbers and split horizon (see [Gateway Discovery](gateway.md#exit-gateway-discovery))

#### `route_query` Channel

Used to actively query peers for known gateway routes:

```json
HEAD: {
  "c": 124,
  "type": "route_query",
  "seq": 0
}
BODY: {
  "need": "ipv4",           // Optional: "ipv4" or "ipv6"
  "prefix": "0.0.0.0/0"     // Optional: specific prefix
}
```

**Response:**

```json
HEAD: {
  "c": 124,
  "type": "route_query",
  "seq": 0,
  "ack": 0
}
BODY: {
  "routes": [
    {
      "gateway": "<hashname>",
      "hops": <number>,
      "via": "<peer hashname>",
      "metric": {...},
      "prefixes": [...]
    }
  ]
}
```

**Properties:**
- **Reliability**: Reliable (request/response pattern)
- **Use case**: On-demand gateway discovery when route_info hasn't been received
- **Privacy**: Only queries directly linked peers (no mesh-wide broadcast)

See [Gateway - Exit Gateway Discovery](gateway.md#exit-gateway-discovery) for detailed protocol specification.

### Gateway Channel Types

The following channel types are used by the [Gateway](gateway.md) component for IP connectivity:

| Type | Reliability | Purpose |
|------|-------------|---------|
| `iptunnel` | Reliable | IP packet tunneling (Entry ↔ Exit Gateway) |
| `tcp_proxy` | Reliable | TCP connection proxy (TNG node → Exit Gateway) |
| `udp_proxy` | Unreliable | UDP datagram proxy (TNG node → Exit Gateway) |
| `dns_query` | Reliable | DNS resolution via Exit Gateway |

See [Gateway](gateway.md) for detailed specifications of these channel types.

### Stream Channel

Generic bidirectional byte stream:

```json
// Open
{"c": 1, "type": "stream", "seq": 0}

// Data
{"c": 1, "seq": 1, "ack": 0}
BODY: [stream data]

// Close
{"c": 1, "seq": 10, "ack": 8, "end": true}
```

### Path Channel

Exchange connectivity information:

```json
// Request paths
{"c": 2, "type": "path", "seq": 0}

// Response with paths
{"c": 2, "seq": 0, "ack": 0}
BODY: [{"type":"quic","ip":"1.2.3.4","port":42424}]
```

## Timeouts

| Timeout | Default | Purpose |
|---------|---------|---------|
| Open timeout | 10s | Time to receive response to open |
| Idle timeout | 60s | Close channel after inactivity |
| Retransmit timeout | 1s | Initial retransmission delay |
| Max retransmit timeout | 30s | Maximum backoff |

## Implementation Notes

### Buffering

```python
# Sender buffer
send_buffer: dict[int, tuple[bytes, float]]  # seq -> (data, sent_time)

# Receiver buffer (for reordering)
recv_buffer: dict[int, bytes]  # seq -> data
recv_next: int  # Next expected sequence
```

### Memory Limits

Implementations should limit:
- Maximum channels per link
- Maximum buffer size per channel
- Total memory across all channels

```python
MAX_CHANNELS = 256
MAX_BUFFER_PER_CHANNEL = 64 * 1024  # 64 KB
MAX_TOTAL_BUFFER = 1 * 1024 * 1024  # 1 MB
```

### Congestion Control

For reliable channels, consider:
- Slow start
- Congestion avoidance
- Fast retransmit/recovery
- Or delegate to transport (QUIC handles this)

## Error Handling

| Error | Action |
|-------|--------|
| Unknown CID | Ignore or send error response |
| Duplicate open | Ignore |
| Invalid sequence | Ignore packet |
| Max retries exceeded | Fail channel, notify application |

---

*Related: [E3X](e3x.md) | [Links](links.md) | [LOB Packets](lob.md) | [Gateway](gateway.md) | [Routing](routing.md)*
