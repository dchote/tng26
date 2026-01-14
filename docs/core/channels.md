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

*Related: [E3X](e3x.md) | [Links](links.md) | [LOB Packets](lob.md)*
