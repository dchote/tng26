# QUIC Transport

**QUIC** is the primary transport for TNG over IP networks. It provides multiplexed, reliable, encrypted transport with built-in congestion control and NAT traversal capabilities.

## Overview

| Property | Value |
|----------|-------|
| **Transport** | QUIC (RFC 9000) |
| **Default Port** | 42424 |
| **MTU** | ~1200 bytes |
| **Reliability** | Built-in |
| **Encryption** | TLS 1.3 (in addition to E3X) |

## Why QUIC?

QUIC provides several advantages over raw UDP:

| Feature | Benefit |
|---------|---------|
| **Multiplexing** | Multiple streams without head-of-line blocking |
| **Congestion Control** | Fair bandwidth sharing |
| **Connection Migration** | Survives IP address changes |
| **0-RTT** | Fast reconnection |
| **NAT Keepalive** | Built-in PING frames |

## Path Format

```json
{
  "type": "quic",
  "ip": "192.168.1.100",
  "port": 42424
}
```

### IPv6 Support

```json
{
  "type": "quic",
  "ip": "2001:db8::1",
  "port": 42424
}
```

## Connection Establishment

### ALPN

TNG uses Application-Layer Protocol Negotiation with:

```
ALPN: "tng/1"
```

### TLS Configuration

Since E3X provides end-to-end encryption, TNG can use minimal TLS:

```python
# Server-side TLS config
tls_config = {
    "alpn_protocols": ["tng/1"],
    "verify_mode": "none",  # E3X handles authentication
    "min_version": "TLS 1.3",
}

# Self-signed certificate is acceptable
# (E3X provides the actual security)
```

### Connection Flow

```
    Client                                    Server
       │                                         │
       │ ── QUIC Initial (ClientHello) ────────►│
       │                                         │
       │ ◄── QUIC Initial (ServerHello) ────────│
       │                                         │
       │ ── QUIC Handshake ────────────────────►│
       │                                         │
       │ ◄── QUIC Handshake ────────────────────│
       │                                         │
      ════════════ QUIC Connected ══════════════
       │                                         │
       │ ── TNG Handshake (E3X) ───────────────►│
       │                                         │
       │ ◄── TNG Handshake (E3X) ───────────────│
       │                                         │
      ════════════ TNG Link Up ═════════════════
```

## Packet Mapping

TNG packets are sent as QUIC datagrams or streams:

### Datagrams (Preferred)

For latency-sensitive data, use QUIC DATAGRAM frames (RFC 9221):

```python
# Enable datagrams in QUIC config
quic_config.enable_datagrams = True
quic_config.max_datagram_size = 1200

# Send TNG packet as datagram
connection.send_datagram(tng_packet)
```

### Streams (Fallback)

If datagrams unavailable, use unidirectional streams:

```python
# Create stream for each packet
stream = connection.create_unidirectional_stream()
stream.write(len(tng_packet).to_bytes(2, 'big') + tng_packet)
stream.close()
```

## Port and Discovery

### Default Port

TNG listens on UDP port **42424** by default.

### mDNS Discovery

For local network discovery, advertise via mDNS:

```
Service: _tng._udp.local
TXT records:
  - hashname=kw3a...kkq
  - cs4a=base32key
```

### DNS-SD

```
_tng._udp.example.com. 86400 IN SRV 0 5 42424 tng.example.com.
```

## NAT Traversal

QUIC's NAT traversal capabilities:

### Connection ID

QUIC uses connection IDs rather than IP:port tuples, enabling:
- NAT rebinding tolerance
- Connection migration

### Keepalive

Send QUIC PING frames to maintain NAT mappings:

```python
KEEPALIVE_INTERVAL = 15  # seconds

async def keepalive_loop(connection):
    while connection.is_open():
        await asyncio.sleep(KEEPALIVE_INTERVAL)
        connection.send_ping()
```

### ICE-style Probing

For difficult NAT scenarios, implement connectivity checks:

```python
async def probe_paths(self, candidate_paths: list[dict]):
    """Probe multiple paths to find working one"""
    for path in candidate_paths:
        try:
            conn = await asyncio.wait_for(
                self.connect(path),
                timeout=5.0
            )
            return conn
        except (asyncio.TimeoutError, ConnectionError):
            continue
    
    raise NoPathAvailable()
```

## Implementation

### Server

```python
import asyncio
from aioquic.asyncio import serve
from aioquic.quic.configuration import QuicConfiguration

async def start_quic_server(mesh, port=42424):
    config = QuicConfiguration(
        is_client=False,
        alpn_protocols=["tng/1"],
        max_datagram_frame_size=1400,
    )
    config.load_cert_chain("cert.pem", "key.pem")
    
    async def handle_connection(reader, writer):
        pipe = QuicPipe(reader, writer)
        mesh.on_pipe(pipe)
    
    await serve(
        "0.0.0.0",
        port,
        configuration=config,
        stream_handler=handle_connection,
    )
```

### Client

```python
async def connect_quic(path: dict) -> Pipe:
    config = QuicConfiguration(
        is_client=True,
        alpn_protocols=["tng/1"],
        verify_mode=ssl.CERT_NONE,  # E3X handles auth
    )
    
    async with connect(
        path["ip"],
        path["port"],
        configuration=config,
    ) as connection:
        return QuicPipe(connection)
```

### Pipe Implementation

```python
class QuicPipe(Pipe):
    def __init__(self, connection):
        self.connection = connection
        self._mtu = 1200
    
    @property
    def mtu(self) -> int:
        return self._mtu
    
    async def send(self, packet: bytes):
        if len(packet) <= self._mtu:
            self.connection.send_datagram(packet)
        else:
            raise PacketTooLarge()
    
    async def receive(self) -> bytes:
        return await self.connection.receive_datagram()
    
    def close(self):
        self.connection.close()
```

## Configuration

### Recommended Settings

```python
QuicConfiguration(
    # Protocol
    alpn_protocols=["tng/1"],
    
    # Limits
    max_datagram_frame_size=1400,
    max_stream_data_bidi_local=1_000_000,
    max_stream_data_bidi_remote=1_000_000,
    
    # Timeouts
    idle_timeout=60.0,
    
    # Congestion
    congestion_control_algorithm="cubic",
)
```

### Constrained Environments

For embedded systems or slow networks:

```python
QuicConfiguration(
    max_datagram_frame_size=512,
    max_stream_data_bidi_local=64_000,
    idle_timeout=120.0,
)
```

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `CONNECTION_REFUSED` | No server at address | Try next path |
| `CONNECTION_TIMEOUT` | Network unreachable | Retry with backoff |
| `PROTOCOL_ERROR` | ALPN mismatch | Check configuration |
| `IDLE_TIMEOUT` | No activity | Reconnect |

## Security Considerations

### Certificate Validation

Since E3X provides authentication:
- Self-signed certificates are acceptable
- Certificate validation is optional
- Connection security comes from E3X handshake

### DDoS Protection

- Implement connection rate limiting
- Use QUIC retry tokens
- Consider source address validation

## Performance

### Typical Latency

| Scenario | RTT |
|----------|-----|
| Local network | <1ms |
| Same region | 10-50ms |
| Cross-continent | 100-200ms |

### Throughput

QUIC congestion control automatically adapts to available bandwidth.

---

*Related: [Transport Overview](README.md) | [WebTransport](webtransport.md) | [Bluetooth](bluetooth.md)*
