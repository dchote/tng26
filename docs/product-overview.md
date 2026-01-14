# Product Overview

## Vision

**Privacy-first mesh networking for the post-quantum era.**

Telehash Next Generation (TNG) is a complete re-imagination of the original telehash protocol, designed to provide secure, private, decentralized communication between any devices—from cloud servers to constrained embedded sensors. In an era of increasing surveillance and the looming threat of quantum computing, TNG ensures that your communications remain private today and secure tomorrow.

## Design Principles

### 1. End-to-End Encryption by Default

Every connection in TNG is encrypted. There is no "plaintext mode." All data exchanged between endpoints is protected using modern cryptographic primitives, ensuring that only the intended recipients can read the content.

### 2. Transport Agnostic

TNG operates over any network layer. Whether you're communicating over:
- High-speed internet via QUIC
- Browser applications via WebTransport
- Short-range IoT via Bluetooth LE
- Long-range sensors via LoRa
- Simple embedded devices via 433/915MHz radio

The protocol adapts to your transport while maintaining the same security guarantees.

### 3. Self-Sovereign Identity

Every endpoint generates its own cryptographic identity (hashname) without relying on any central authority. There are no certificate authorities, no identity providers, no single points of failure. You own your identity.

Optional integration with W3C Decentralized Identifiers (DIDs) allows bridging to broader identity ecosystems while maintaining self-sovereignty.

### 4. Quantum-Resistant Ready

TNG includes cipher suites with post-quantum cryptographic algorithms (ML-KEM) in hybrid mode. This provides protection against future quantum computers while maintaining compatibility with classical cryptography.

### 5. Minimal Metadata Leakage

The protocol is designed to minimize what third parties can learn about your communications:
- No central servers that can be subpoenaed
- Optional traffic cloaking to prevent fingerprinting
- Private mesh topology (your view of the network is yours alone)
- Trusted routing with minimal exposure

## Key Capabilities

### Secure Peer-to-Peer Connections

Establish encrypted links directly between any two endpoints. Each link provides:
- Mutual authentication via cryptographic handshakes
- Perfect forward secrecy via ephemeral keys
- Multiple simultaneous channels (reliable and unreliable)
- Automatic reconnection and path failover

### NAT Traversal and Relay Routing

Real-world networks are messy. TNG handles:
- NAT traversal via ICE-style connectivity checks
- Trusted routers for relaying when direct connection fails
- Multi-path connections for redundancy
- Automatic path optimization

### Offline-First with Async Messaging

Not all communication is real-time. TNG supports:
- Asynchronous encrypted messages that can be delivered later
- Store-and-forward patterns for intermittent connectivity
- Delay-tolerant networking for constrained environments

### Cross-Platform Support

TNG is designed to run everywhere:
- **Servers**: High-performance implementations in Rust, Go, or C
- **Mobile**: Native libraries for iOS and Android
- **Browsers**: WebTransport-based JavaScript implementation
- **Embedded**: Minimal footprint for microcontrollers (ARM Cortex-M, ESP32, etc.)
- **Constrained IoT**: Support for sub-GHz radios with 64-byte MTUs

## Use Cases

### IoT Device Mesh Networking

Connect sensors, actuators, and gateways in a secure mesh:
- Agricultural sensors over LoRa communicating with farm gateways
- Smart home devices forming local Zigbee/Thread meshes
- Industrial IoT with mixed WiFi and 802.15.4 networks

### Private Messaging Applications

Build messaging apps with true privacy:
- No central server that stores your messages
- End-to-end encryption with forward secrecy
- Metadata protection (who talks to whom is private)
- Works across mobile, desktop, and web

### Decentralized Application Infrastructure

Power the next generation of decentralized apps:
- Peer-to-peer data synchronization
- Distributed storage networks
- Blockchain and Web3 communication layers
- Federated social networks

### Secure Remote Access

Replace traditional VPNs with something better:
- Direct encrypted tunnels to your devices
- No central VPN server to trust or attack
- Works across network boundaries and NATs
- Lower latency than hub-and-spoke VPN architectures

### Emergency and Disaster Communication

When infrastructure fails:
- Mesh networking over available radios
- No dependency on cellular or internet infrastructure
- Long-range communication via LoRa
- Interoperability across different physical layers

## Comparison with Alternatives

| Feature | TNG | libp2p | Tor | WireGuard |
|---------|-----|--------|-----|-----------|
| End-to-end encryption | ✓ | ✓ | ✓ | ✓ |
| Post-quantum ready | ✓ | Partial | ✗ | ✗ |
| Transport agnostic | ✓ | ✓ | ✗ | ✗ |
| Embedded support | ✓ | Limited | ✗ | Limited |
| Metadata privacy | ✓ | Limited | ✓ | ✗ |
| No central infrastructure | ✓ | ✓ | ✗ (relays) | ✗ (server) |
| Physical layer support | ✓ | ✗ | ✗ | ✗ |
| Self-sovereign identity | ✓ | ✓ | ✓ | ✗ |

### vs. libp2p

libp2p is an excellent modular networking stack used by IPFS and other projects. TNG differs in:
- **Privacy focus**: TNG minimizes metadata exposure by default
- **Simpler API**: Fewer concepts to learn, easier integration
- **Physical layer support**: Native support for constrained radios (LoRa, 802.15.4, sub-GHz)
- **Post-quantum cryptography**: Built-in hybrid PQ cipher suites

### vs. Tor

Tor provides anonymity through onion routing. TNG differs in:
- **Lower latency**: Direct connections when possible
- **Not anonymity-focused**: Privacy without the Tor network overhead
- **Embedded support**: Runs on microcontrollers
- **Transport flexibility**: Works over any network layer

### vs. WireGuard

WireGuard is an excellent modern VPN. TNG differs in:
- **Application-layer protocol**: No kernel module required
- **Multi-transport**: Works over QUIC, WebTransport, Bluetooth, LoRa, etc.
- **Mesh networking**: Peer-to-peer, not client-server
- **Self-sovereign identity**: No need to exchange public keys out-of-band

## Project Roadmap

### Phase 1: Core Protocol Specification ✅ (Complete)
- ✅ Specification of core building blocks (Hashname, LOB, E3X, Cipher Suites, Channels, Links, Mesh, Routing)
- ✅ Logical transport specifications (QUIC, WebTransport, Bluetooth LE)
- ✅ Physical transport specifications (802.11, 802.15.4, LoRa, 433/915MHz UART)
- ⏳ Reference implementation in Rust (Planned)

### Phase 2: Ecosystem (Planned)
- Additional language bindings (JavaScript, Python, Go)
- Mobile SDKs (iOS, Android)
- Embedded libraries (C, no-std Rust)

### Phase 3: Advanced Features (Planned)
- Group channels with MLS-style encryption
- Decentralized discovery mechanisms
- Enhanced routing algorithms

---

*For technical details, see the [Technical Overview](technical-overview.md).*
