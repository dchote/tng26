# Transport Layer

The transport layer provides network connectivity for TNG, carrying encrypted [E3X](../core/e3x.md) packets between endpoints. TNG supports two categories of transports:

- **Logical Transports**: Operate over IP networks with higher-level abstractions (QUIC, WebTransport, Bluetooth)
- **Physical Transports**: Operate directly on radio hardware for constrained embedded devices (802.11, 802.15.4, LoRa, UART Radio)

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     E3X Encrypted Packets                       │
└───────────────────────────────┬─────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────────────┐
        │                       │                               │
        ▼                       ▼                               ▼
┌───────────────────────────────────────────┐           ┌───────────────┐
│           Logical Transports              │           │   Chunking    │
│  ┌──────┐  ┌────────────┐  ┌───────────┐  │           │    Layer      │
│  │ QUIC │  │WebTransport│  │Bluetooth  │  │           └───────┬───────┘
│  └──────┘  └────────────┘  └───────────┘  │                   │
└───────────────────────────────────────────┘                   ▼
                                                ┌───────────────────────────────┐
                                                │     Physical Transports       │
                                                │ ┌──────┐ ┌───────┐ ┌────────┐│
                                                │ │802.11│ │802.15.4│ │  LoRa  ││
                                                │ └──────┘ └───────┘ └────────┘│
                                                │         ┌──────────┐         │
                                                │         │433/915MHz│         │
                                                │         └──────────┘         │
                                                └───────────────────────────────┘
```

---

## Logical Transports

Logical transports operate over IP networks and provide higher-level abstractions.

| Transport | Specification | Use Case | MTU |
|-----------|---------------|----------|-----|
| **QUIC** | [quic.md](quic.md) | Primary internet transport | ~1200B |
| **WebTransport** | [webtransport.md](webtransport.md) | Browser applications | ~1200B |
| **Bluetooth LE** | [bluetooth.md](bluetooth.md) | Mobile, short-range IoT | 247B |

---

## Physical Transports

Physical transports enable TNG on constrained embedded devices that communicate directly over radio hardware without IP networking. These require chunking for small MTUs.

| Transport | Specification | MTU | Range | Data Rate |
|-----------|---------------|-----|-------|-----------|
| **802.11** | [802.11.md](802.11.md) | 2304B | ~100m | 1Mbps-1Gbps |
| **802.15.4** | [802.15.4.md](802.15.4.md) | 127B | 10-100m | 250kbps |
| **LoRa** | [lora.md](lora.md) | 255B | 2-15km | 0.3-50kbps |
| **433/915MHz** | [uart-radio.md](uart-radio.md) | 64B | 100m-1km | 1-100kbps |

---

## Transport Interface

All transports implement a common interface:

### Pipe Abstraction

A **Pipe** represents an active transport path to another endpoint:

```python
class Pipe:
    """Abstract transport connection"""
    
    @property
    def mtu(self) -> int:
        """Maximum transmission unit"""
        pass
    
    @property
    def latency(self) -> float:
        """Estimated round-trip time in seconds"""
        pass
    
    async def send(self, packet: bytes) -> None:
        """Send an encrypted packet"""
        pass
    
    async def receive(self) -> bytes:
        """Receive an encrypted packet"""
        pass
    
    def close(self) -> None:
        """Close the pipe"""
        pass
```

### Transport Manager

```python
class Transport:
    """Abstract transport implementation"""
    
    @property
    def type(self) -> str:
        """Transport type identifier (e.g., 'quic', 'lora')"""
        pass
    
    async def listen(self, address: str) -> None:
        """Start listening for incoming connections"""
        pass
    
    async def connect(self, path: dict) -> Pipe:
        """Connect to a path, return a Pipe"""
        pass
    
    def on_pipe(self, callback: Callable[[Pipe], None]) -> None:
        """Register callback for incoming pipes"""
        pass
```

---

## Path Formats

Each transport defines its path format:

### Logical Transport Paths

```json
// QUIC
{ "type": "quic", "ip": "192.168.1.100", "port": 42424 }

// WebTransport
{ "type": "webtransport", "url": "https://example.com:443/tng" }

// Bluetooth LE
{ "type": "bluetooth", "address": "AA:BB:CC:DD:EE:FF" }
```

### Physical Transport Paths

```json
// 802.11 WiFi
{ "type": "802.11", "bssid": "AA:BB:CC:DD:EE:FF", "channel": 6 }

// 802.15.4 (Zigbee/Thread)
{ "type": "802.15.4", "pan_id": "0x1234", "short_addr": "0x5678", "channel": 15 }

// LoRa
{ "type": "lora", "address": 12345678, "spreading_factor": 7, "frequency": 868100000 }

// 433/915MHz UART Radio
{ "type": "uart-radio", "address": 1234, "channel": 1 }
```

---

## URI Format

TNG endpoints can be represented as URIs for out-of-band sharing:

```
tng://<hashname>?<params>
```

### Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `cs4a` | CS4a public key (base32) | `cs4a=aiw4cw...` |
| `cs4b` | CS4b public key (base32) | `cs4b=mzxw6y...` |
| `ip` | IP address | `ip=192.168.1.100` |
| `port` | Port number | `port=42424` |
| `url` | WebTransport URL | `url=https://...` |
| `bt` | Bluetooth address | `bt=AA:BB:CC:DD:EE:FF` |

### Example URIs

```
# QUIC endpoint
tng://kw3a...kkq?cs4a=aiw4cw...&ip=192.168.1.100&port=42424

# WebTransport endpoint  
tng://kw3a...kkq?cs4a=aiw4cw...&url=https://example.com/tng

# Multiple paths
tng://kw3a...kkq?cs4a=...&ip=1.2.3.4&port=42424&bt=AA:BB:CC:DD:EE:FF
```

---

## Transport Selection

When multiple transports are available:

### Priority Order

1. **Direct QUIC**: Lowest latency for internet
2. **WebTransport**: Browser-compatible fallback
3. **Bluetooth LE**: Short-range backup
4. **Physical transports**: When IP unavailable

### Selection Algorithm

```python
def select_transport(paths: list[dict], capabilities: set[str]) -> dict:
    """Select best path from available options"""
    
    priority = ["quic", "webtransport", "bluetooth", "802.11", "lora", "802.15.4", "uart-radio"]
    
    for transport_type in priority:
        if transport_type not in capabilities:
            continue
        
        for path in paths:
            if path["type"] == transport_type:
                return path
    
    return None
```

---

## Chunking (Physical Transports)

Physical transports with small MTUs require packet chunking.

### Chunk Format

```
┌──────────────────────────────────────────────────────────────┐
│  Header (1-3 bytes)  │  Payload (MTU - header)               │
└──────────────────────────────────────────────────────────────┘

Byte 0: [FIRST:1][LAST:1][SEQ:6]
Byte 1: [LENGTH_HIGH] (if FIRST)
Byte 2: [LENGTH_LOW]  (if FIRST)
```

### Chunking Parameters

| Physical Layer | MTU | Header | Payload | Max Packet |
|----------------|-----|--------|---------|------------|
| 802.11 | 2304B | 0 | 2304B | No chunking needed |
| 802.15.4 | 127B | 3B | 124B | 7936B (64 chunks) |
| LoRa | 255B | 4B | 251B | 16064B (64 chunks) |
| 433/915MHz | 64B | 3B | 61B | 3904B (64 chunks) |

### Chunking Implementation

```python
def chunk_packet(packet: bytes, mtu: int, header_size: int) -> list[bytes]:
    """Split packet into chunks for low-MTU transports"""
    payload_size = mtu - header_size
    chunks = []
    
    total_length = len(packet)
    offset = 0
    seq = 0
    
    while offset < total_length:
        is_first = (offset == 0)
        chunk_data = packet[offset:offset + payload_size]
        is_last = (offset + payload_size >= total_length)
        
        header = (seq & 0x3F)
        if is_first:
            header |= 0x80
        if is_last:
            header |= 0x40
        
        if is_first and header_size >= 3:
            chunk = bytes([header, (total_length >> 8) & 0xFF, total_length & 0xFF])
        else:
            chunk = bytes([header])
        
        chunk += chunk_data
        chunks.append(chunk)
        
        offset += payload_size
        seq = (seq + 1) & 0x3F
    
    return chunks


def reassemble_chunks(chunks: list[bytes]) -> bytes:
    """Reassemble chunks into original packet"""
    packet = b""
    expected_length = None
    
    for chunk in chunks:
        header = chunk[0]
        is_first = bool(header & 0x80)
        is_last = bool(header & 0x40)
        
        if is_first:
            expected_length = (chunk[1] << 8) | chunk[2]
            data = chunk[3:]
        else:
            data = chunk[1:]
        
        packet += data
        
        if is_last:
            break
    
    return packet
```

---

## Framing (Physical Transports)

Physical transports need packet boundary detection:

### Frame Format

```
┌────────────────────────────────────────────────────────────────┐
│ SYNC (2B) │ LENGTH (2B) │ DATA (N bytes) │ CRC16 (2B)         │
└────────────────────────────────────────────────────────────────┘

SYNC: 0x7E 0x7E
```

### CRC-16

```python
def crc16_ccitt(data: bytes) -> int:
    crc = 0xFFFF
    for byte in data:
        crc ^= byte << 8
        for _ in range(8):
            if crc & 0x8000:
                crc = (crc << 1) ^ 0x1021
            else:
                crc <<= 1
            crc &= 0xFFFF
    return crc
```

---

## Addressing (Physical Transports)

| Transport | Address Format | Size |
|-----------|----------------|------|
| 802.11 | MAC address | 6 bytes |
| 802.15.4 | Short (2B) or Extended (8B) | 2-8 bytes |
| LoRa | Device address | 4 bytes |
| 433/915MHz | Custom (preamble) | 1-4 bytes |

### Address Mapping

```python
class AddressTable:
    def __init__(self):
        self.hashname_to_addr = {}
        self.addr_to_hashname = {}
    
    def register(self, hashname: str, address: bytes):
        self.hashname_to_addr[hashname] = address
        self.addr_to_hashname[address] = hashname
```

---

## Duty Cycle (Physical Transports)

Many physical layers have regulatory duty cycle restrictions:

| Region | Band | Max Duty Cycle |
|--------|------|----------------|
| EU | 868 MHz | 1% (36s/hour) |
| EU | 869.4-869.65 MHz | 10% |
| US | 915 MHz | No limit (FCC Part 15) |

---

## Multi-Path Operation

TNG supports simultaneous operation over multiple transports:

```
                    ┌─────────────┐
              ┌─────│    Link     │─────┐
              │     └─────────────┘     │
              │                         │
       ┌──────▼──────┐          ┌───────▼─────┐
       │ Pipe: QUIC  │          │ Pipe: LoRa  │
       │ Primary     │          │ Backup      │
       └─────────────┘          └─────────────┘
```

### Benefits

- **Redundancy**: Automatic failover
- **Load balancing**: Spread traffic across paths
- **Latency optimization**: Use fastest path per-packet

---

## Keepalive

| Transport | Keepalive Interval | Method |
|-----------|-------------------|--------|
| QUIC | 15 seconds | PING frames |
| WebTransport | 30 seconds | HTTP/3 PING |
| Bluetooth LE | 30 seconds | L2CAP ping |
| Physical | Application-defined | Periodic beacon |

---

## Discovery

| Transport | Discovery Method |
|-----------|------------------|
| QUIC | mDNS/DNS-SD |
| WebTransport | N/A (requires URL) |
| Bluetooth LE | BLE advertising |
| 802.11 | Beacon frames, probe requests |
| 802.15.4 | Beacon requests, active scan |
| LoRa | Periodic broadcasts |
| 433/915MHz | Listen-before-talk |

---

## Security Considerations

### Transport-Level Encryption

TNG packets are encrypted by E3X before transport. Additional encryption:

| Transport | Additional Encryption |
|-----------|----------------------|
| QUIC | TLS 1.3 (optional) |
| WebTransport | TLS 1.3 (required) |
| Bluetooth LE | None (E3X sufficient) |
| Physical | None (E3X sufficient) |

### Path Privacy

- Sharing paths reveals network addresses
- Consider privacy implications when distributing URIs
- Use routers to hide direct paths

---

## Error Handling

| Error | Action |
|-------|--------|
| Connection refused | Try next path |
| Timeout | Retry with backoff |
| MTU exceeded | Chunk (physical) or fail |
| CRC failure | Discard, request retry |
| Transport unavailable | Fall back to alternative |

---

## Implementation Notes

### Buffer Sizing

```python
RECV_BUFFER = 64 * 1024    # 64 KB receive buffer
SEND_BUFFER = 64 * 1024    # 64 KB send buffer
MAX_PACKET = 1400          # Maximum packet size (logical)
```

### Physical Transport Requirements

- Frame sync detection
- CRC verification
- Chunking/reassembly
- Address management
- Duty cycle management (where required)
- Power management (embedded)

---

*Logical: [QUIC](quic.md) | [WebTransport](webtransport.md) | [Bluetooth](bluetooth.md)*

*Physical: [802.11](802.11.md) | [802.15.4](802.15.4.md) | [LoRa](lora.md) | [433/915MHz](uart-radio.md)*
