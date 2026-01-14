# TNG26 - Telehash Next Generation

**Privacy-first mesh networking for the post-quantum era.**

TNG26 is a modern reimagining of the Telehash protocol. The protocol itself is called **TNG (Telehash Next Generation)**, and this repository (TNG26) contains its complete specification and documentation.

## Project Goals

### 1. **Privacy-First Communication**

Create a networking protocol where privacy is not an afterthought but a fundamental design principle:

- **End-to-end encryption by default** - Every connection is encrypted; there is no "plaintext mode"
- **Minimal metadata leakage** - Third parties learn as little as possible about your communications
- **No central infrastructure** - No servers that can be subpoenaed or compromised
- **Self-sovereign identity** - Generate your own cryptographic identity without relying on certificate authorities

### 2. **Quantum-Resistant Security**

Prepare for the post-quantum computing era:

- **Hybrid post-quantum cryptography** - ML-KEM (FIPS 203) combined with classical algorithms
- **Future-proof cipher suites** - Protection against both current and future threats
- **Graceful migration path** - Support for both classical and post-quantum algorithms

### 3. **Universal Transport Support**

Enable secure communication over any network layer:

- **Logical transports**: QUIC (internet), WebTransport (browsers), Bluetooth LE (mobile/IoT)
- **Physical transports**: 802.11 (WiFi), 802.15.4 (Zigbee/Thread), LoRa (long-range), 433/915MHz (simple radios)
- **Transport-agnostic design** - Same security guarantees regardless of underlying network

### 4. **Cross-Platform Compatibility**

Run TNG everywhere:

- **Servers**: High-performance implementations (Rust, Go, C)
- **Mobile**: Native libraries for iOS and Android
- **Browsers**: WebTransport-based JavaScript implementation
- **Embedded**: Minimal footprint for microcontrollers (ARM Cortex-M, ESP32, etc.)
- **Constrained IoT**: Support for devices with 64-byte MTUs

### 5. **Decentralized Mesh Networking**

Build a truly decentralized network:

- **Peer-to-peer connections** - Direct encrypted links between endpoints
- **Private mesh topology** - Your view of the network is yours alone
- **NAT traversal** - Works across network boundaries without central servers
- **Multi-path redundancy** - Automatic failover and load balancing

### 6. **Developer-Friendly Design**

Make secure networking accessible:

- **Simple API** - Fewer concepts to learn than alternatives
- **Comprehensive documentation** - Clear specifications and guides
- **Reference implementations** - Working code to learn from and build upon
- **Modern standards** - Aligned with 2026 best practices (Noise Protocol, DIDs, CBOR)

## Key Features

- 🔒 **End-to-end encryption** with perfect forward secrecy
- 🌐 **Transport agnostic** - Works over IP, Bluetooth, LoRa, and more
- 🔐 **Post-quantum ready** - Hybrid ML-KEM cipher suites
- 🆔 **Self-sovereign identity** - No certificate authorities required
- 📱 **Cross-platform** - Servers, mobile, browsers, embedded devices
- 🔗 **Mesh networking** - Peer-to-peer with NAT traversal
- 📡 **Physical layer support** - Direct radio communication for IoT
- 🚫 **No central infrastructure** - Truly decentralized

## Use Cases

- **IoT Device Mesh Networking** - Secure sensor networks over LoRa, Zigbee, WiFi
- **Private Messaging** - End-to-end encrypted messaging without central servers
- **Decentralized Applications** - Infrastructure for peer-to-peer apps
- **Secure Remote Access** - Better than traditional VPNs
- **Emergency Communication** - Mesh networking when infrastructure fails

## Documentation

- **[Product Overview](docs/product-overview.md)** - High-level introduction and use cases
- **[Technical Overview](docs/technical-overview.md)** - Architecture and specifications

### Core Protocol
- **[Hashname](docs/core/hashname.md)** - Cryptographic endpoint identity
- **[LOB Packets](docs/core/lob.md)** - Packet encoding format
- **[E3X](docs/core/e3x.md)** - End-to-end encrypted exchange
- **[Cipher Suites](docs/core/cipher-suites.md)** - Cryptographic algorithms
- **[Channels](docs/core/channels.md)** - Multiplexed data streams
- **[Links](docs/core/links.md)** - Encrypted connections
- **[Mesh](docs/core/mesh.md)** - Network topology management
- **[Routing](docs/core/routing.md)** - Packet forwarding and NAT traversal

### Transports
- **[Transport Overview](docs/transports/README.md)** - Architecture and common interfaces
- **Logical**: [QUIC](docs/transports/quic.md) | [WebTransport](docs/transports/webtransport.md) | [Bluetooth LE](docs/transports/bluetooth.md)
- **Physical**: [802.11 WiFi](docs/transports/802.11.md) | [802.15.4](docs/transports/802.15.4.md) | [LoRa](docs/transports/lora.md) | [433/915MHz UART](docs/transports/uart-radio.md)

## Project Status

**Current Phase**: Specification Complete ✅ | Implementation Planned ⏳

This repository contains the **complete specification** for TNG (Telehash Next Generation), including:
- ✅ Core protocol building blocks (all 8 components specified)
- ✅ Logical transport specifications (QUIC, WebTransport, Bluetooth LE)
- ✅ Physical transport specifications (802.11, 802.15.4, LoRa, 433/915MHz UART)
- ✅ Security and cryptography requirements

**Next Steps**: Reference implementations and language bindings are planned for Phase 2.

## Comparison

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

## Contributing

This project is in the documentation and specification phase. Contributions are welcome for:
- Specification improvements and clarifications
- Documentation enhancements
- Reference implementation planning
- Use case documentation

## License

See [LICENSE](LICENSE) file for details.

---

**Telehash Next Generation** - Secure, private, decentralized networking for everyone.
