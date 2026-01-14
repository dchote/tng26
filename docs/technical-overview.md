# Technical Overview

This document provides an architectural overview of the Telehash Next Generation (TNG) protocol and serves as a hub linking to detailed specifications for each component.

## Protocol Stack

TNG is organized into distinct layers, each with specific responsibilities:

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                          │
│                   (Your App / Service)                          │
├─────────────────────────────────────────────────────────────────┤
│                     Gateway Layer (Optional)                    │
│        Entry Gateway │ IP Router │ Exit Gateway │ NAT           │
├─────────────────────────────────────────────────────────────────┤
│                        Mesh Layer                               │
│              Mesh Manager │ Discovery │ Routing                 │
├─────────────────────────────────────────────────────────────────┤
│                        Link Layer                               │
│                Links │ Channels │ Multi-Path                    │
├─────────────────────────────────────────────────────────────────┤
│                    E3X (Encrypted Exchange)                     │
│            Handshakes │ Sessions │ Messages                     │
├─────────────────────────────────────────────────────────────────┤
│                    Cryptographic Layer                          │
│          Hashnames │ Cipher Suites │ Post-Quantum               │
├─────────────────────────────────────────────────────────────────┤
│                      Encoding Layer                             │
│                   LOB Packets │ CBOR                            │
├─────────────────────────────────────────────────────────────────┤
│                   Logical Transports                            │
│            QUIC │ WebTransport │ Bluetooth LE                   │
├─────────────────────────────────────────────────────────────────┤
│                  Physical Transports                            │
│        802.11 │ 802.15.4 │ LoRa │ 433/915MHz Radio              │
└─────────────────────────────────────────────────────────────────┘
```

## Core Building Blocks

| Component | Description | Specification |
|-----------|-------------|---------------|
| **Hashname** | 52-character base32 cryptographic endpoint identity | [core/hashname.md](core/hashname.md) |
| **LOB Packets** | Length-Object-Binary packet encoding format | [core/lob.md](core/lob.md) |
| **E3X** | End-to-end encrypted exchange protocol | [core/e3x.md](core/e3x.md) |
| **Cipher Suites** | Pluggable cryptographic algorithm groups | [core/cipher-suites.md](core/cipher-suites.md) |
| **Channels** | Multiplexed reliable/unreliable data streams | [core/channels.md](core/channels.md) |
| **Links** | Encrypted connections between endpoints | [core/links.md](core/links.md) |
| **Mesh** | Collection of links with discovery | [core/mesh.md](core/mesh.md) |
| **Routing** | Trusted packet forwarding via routers | [core/routing.md](core/routing.md) |
| **Route Metrics** | Route cost calculation and transport capabilities | [core/route-metrics.md](core/route-metrics.md) |
| **Gateway** | IP-over-mesh tunneling with entry/exit nodes | [core/gateway.md](core/gateway.md) |

## Transport Layer

### Logical Transports

These operate over IP networks and provide higher-level abstractions:

| Transport | Use Case | MTU | Specification |
|-----------|----------|-----|---------------|
| **QUIC** | Primary internet transport | ~1200B | [transports/quic.md](transports/quic.md) |
| **WebTransport** | Browser/web applications | ~1200B | [transports/webtransport.md](transports/webtransport.md) |
| **Bluetooth LE** | Mobile, short-range IoT | 247B | [transports/bluetooth.md](transports/bluetooth.md) |

See [transports/README.md](transports/README.md) for the transport layer architecture.

### Physical Layer Transports

These operate directly on radio hardware for constrained environments:

| Physical Layer | Use Case | MTU | Range | Specification |
|----------------|----------|-----|-------|---------------|
| **802.11 (WiFi)** | Local network | 2304B | ~100m | [transports/802.11.md](transports/802.11.md) |
| **802.15.4** | Zigbee/Thread mesh | 127B | 10-100m | [transports/802.15.4.md](transports/802.15.4.md) |
| **LoRa** | Long-range IoT | 255B | 2-15km | [transports/lora.md](transports/lora.md) |
| **433/915MHz** | Simple embedded | 64B | 100m-1km | [transports/uart-radio.md](transports/uart-radio.md) |

See [transports/README.md](transports/README.md) for the transport layer architecture including physical transports.

## Architecture Diagram

```
                                    ┌──────────────┐
                                    │  Application │
                                    └──────┬───────┘
                                           │
                    ┌──────────────────────┼──────────────────────┐
                    │                      │                      │
                    ▼                      ▼                      ▼
             ┌────────────┐         ┌────────────┐         ┌────────────┐
             │    Mesh    │◄───────►│  Discovery │◄───────►│  Routing   │
             └─────┬──────┘         └────────────┘         └────────────┘
                   │
         ┌─────────┼─────────┐
         │         │         │
         ▼         ▼         ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │ Link 1 │ │ Link 2 │ │ Link N │
    └───┬────┘ └───┬────┘ └───┬────┘
        │          │          │
        └──────────┼──────────┘
                   │
         ┌─────────┼─────────┐
         │         │         │
         ▼         ▼         ▼
    ┌────────┐ ┌────────┐ ┌────────┐
    │Reliable│ │Unreliab│ │  ...   │  ← Channels
    │Channel │ │Channel │ │        │
    └───┬────┘ └───┬────┘ └───┬────┘
        │          │          │
        └──────────┼──────────┘
                   │
                   ▼
          ┌───────────────┐
          │      E3X      │
          │  ┌─────────┐  │
          │  │Handshake│  │
          │  └────┬────┘  │
          │       ▼       │
          │  ┌─────────┐  │
          │  │ Session │  │
          │  └────┬────┘  │
          │       ▼       │
          │  ┌─────────┐  │
          │  │Messages │  │
          │  └─────────┘  │
          └───────┬───────┘
                  │
        ┌─────────┼─────────┐
        │         │         │
        ▼         ▼         ▼
   ┌────────┐ ┌────────┐ ┌────────┐
   │ CS4a   │ │ CS4b   │ │ CS3a   │  ← Cipher Suites
   │(Modern)│ │ (PQ)   │ │(Legacy)│
   └───┬────┘ └───┬────┘ └───┬────┘
       │          │          │
       └──────────┼──────────┘
                  │
                  ▼
          ┌───────────────┐
          │  LOB Packets  │
          │ (or CBOR)     │
          └───────┬───────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
     ▼            ▼            ▼
┌─────────┐ ┌──────────┐ ┌──────────┐
│  QUIC   │ │WebTransp.│ │Bluetooth │  ← Logical Transports
└────┬────┘ └────┬─────┘ └────┬─────┘
     │           │            │
     │      ┌────┴────────────┴────┐
     │      │    Chunking Layer    │
     │      └──────────┬───────────┘
     │                 │
     │    ┌────────────┼────────────┐
     │    │            │            │
     ▼    ▼            ▼            ▼
┌──────┐┌──────┐ ┌──────────┐ ┌──────────┐
│802.11││802.15│ │   LoRa   │ │433/915MHz│  ← Physical Transports
└──────┘└──────┘ └──────────┘ └──────────┘
```

## Quick Reference

### Hashname Format

A hashname is a 52-character base32 string representing a SHA-256 hash of the endpoint's public keys:

```
kw3akwcypoedvfdquuppofpujbu7rplhj3vjvmvbkvf7z3do7kkq
```

### Cipher Suite Summary

| CSID | Algorithms | Status | Use Case |
|------|------------|--------|----------|
| **CS4a** | X25519 + Ed25519 + ChaCha20-Poly1305 | Recommended | General use |
| **CS4b** | ML-KEM-768 + X25519 + Ed25519 + ChaCha20-Poly1305 | Recommended | Post-quantum |
| **CS3a** | Curve25519 + XSalsa20 + Poly1305 | Legacy | Backward compatibility |

### LOB Packet Format

```
┌─────────────────────────────────────────────────────┐
│ LENGTH (2 bytes, big-endian)                        │
├─────────────────────────────────────────────────────┤
│ HEAD (0 to LENGTH bytes, JSON if ≥7 bytes)          │
├─────────────────────────────────────────────────────┤
│ BODY (remaining bytes, binary)                      │
└─────────────────────────────────────────────────────┘
```

### Link States

| State | Description |
|-------|-------------|
| **Unresolved** | Hashname known, but keys or paths incomplete |
| **Down** | Keys validated, path available, not connected |
| **Up** | Active connection with completed handshake |

### Channel Types

| Type | Reliability | Use Case |
|------|-------------|----------|
| **Reliable** | Ordered, acknowledged | Data transfer, streams |
| **Unreliable** | Best-effort, unordered | Real-time, gaming |

## Security Model

### Trust Assumptions

1. **Endpoint Identity**: Hashnames are self-certified via public key cryptography
2. **Link Establishment**: Requires mutual authentication via handshake
3. **Router Trust**: Routers are explicitly trusted by the application; they see connection metadata but not content
4. **No Global State**: Each endpoint's view of the mesh is private

### Cryptographic Guarantees

- **Confidentiality**: All channel data is encrypted
- **Integrity**: All packets are authenticated (AEAD)
- **Forward Secrecy**: Ephemeral keys ensure past sessions remain secure
- **Mutual Authentication**: Both endpoints prove identity during handshake

### Metadata Protection

- Traffic analysis resistance via optional cloaking
- No central directory of endpoints
- Private mesh topology (not shared with peers)
- Minimal information revealed to routers

## Implementation Guidance

### Minimum Implementation

A minimal TNG implementation must support:

1. **Hashname generation** from at least one cipher suite
2. **LOB packet encoding/decoding**
3. **E3X handshake and channel encryption** for at least CS4a
4. **At least one transport** (QUIC recommended for internet)

### Recommended Implementation

A full implementation should include:

1. Multiple cipher suites (CS4a + CS4b for PQ, optionally CS3a for legacy)
2. Multiple transports based on target platform
3. Mesh management with discovery
4. Routing support for NAT traversal

### Platform Considerations

| Platform | Recommended Transports | Cipher Suites | Notes |
|----------|----------------------|---------------|-------|
| Server | QUIC | CS4a, CS4b | High performance |
| Browser | WebTransport | CS4a | JavaScript crypto |
| Mobile | QUIC, Bluetooth | CS4a, CS4b | Battery aware |
| Embedded | 802.15.4, LoRa, UART | CS4a | Memory constrained |

## Document Index

### Core Protocol

- [Hashname](core/hashname.md) - Endpoint identity
- [LOB Packets](core/lob.md) - Packet encoding
- [E3X](core/e3x.md) - Encrypted exchange
- [Cipher Suites](core/cipher-suites.md) - Cryptographic algorithms
- [Channels](core/channels.md) - Data streams
- [Links](core/links.md) - Connections
- [Mesh](core/mesh.md) - Network topology
- [Routing](core/routing.md) - Packet forwarding
- [Route Metrics](core/route-metrics.md) - Route cost calculation and transport capabilities
- [Gateway](core/gateway.md) - IP-over-mesh tunneling

### Transports

- [Transport Overview](transports/README.md) - Architecture
- [QUIC](transports/quic.md) - Primary IP transport
- [WebTransport](transports/webtransport.md) - Browser transport
- [Bluetooth LE](transports/bluetooth.md) - Short-range wireless

### Physical Transports

- [802.11 WiFi](transports/802.11.md) - WiFi direct/mesh
- [802.15.4](transports/802.15.4.md) - Zigbee/Thread
- [LoRa](transports/lora.md) - Long-range radio
- [433/915MHz UART](transports/uart-radio.md) - Simple radios

---

*For a high-level introduction, see the [Product Overview](product-overview.md).*
