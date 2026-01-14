# Matter Protocol Patterns Analysis for TNG

## Executive Summary

This document analyzes the Matter/CHIP SDK (from [project-chip/connectedhomeip](https://github.com/project-chip/connectedhomeip)) to identify patterns for Thread-to-IP translation, IPv6 gateway functionality, and network bridging that TNG can adopt for providing IP tunneling and connectivity to embedded devices.

## Key Architectural Patterns

### 1. Thread Border Router (TBR) Architecture

Matter uses OpenThread Border Router (OTBR) to bridge between Thread (802.15.4) networks and IP networks. The core functions are:

| Function | Description |
|----------|-------------|
| **Bidirectional IP Connectivity** | Routes IPv6 packets between Thread mesh and WiFi/Ethernet |
| **Service Discovery Bridge** | mDNS (WiFi/Ethernet) ↔ SRP (Thread) |
| **Thread-over-Infrastructure** | Merges Thread partitions over IP links |
| **External Commissioning** | Allows IP-based devices to commission Thread devices |

**TNG Opportunity**: Implement a similar "TNG Border Gateway" that bridges TNG physical transports (LoRa, 802.15.4, 433/915MHz) to IP networks, enabling:
- Remote access to constrained devices
- Service discovery bridging
- Multi-transport mesh unification

### 2. Transport Abstraction Pattern

Matter uses a templated `TransportMgr` that can combine multiple transports:

```cpp
// Matter's approach - multiple transports combined
template <typename... TransportTypes>
class TransportMgr : public TransportMgrBase {
    Transport::Tuple<TransportTypes...> mTransport;
};

// Usage: TransportMgr<UDP, TCP, BLE>
```

**PeerAddress Abstraction**:
```cpp
enum class Type : uint8_t {
    kUndefined,
    kUdp,
    kBle,
    kTcp,
    kWiFiPAF,
    kNfc
};

class PeerAddress {
    Inet::IPAddress mIPAddress;
    Type mTransportType;
    uint16_t mPort;
    Inet::InterfaceId mInterface;
};
```

**TNG Adoption**: Our existing `Pipe` and `Transport` abstractions align well. We should add:
- A unified `PeerAddress` type that encapsulates path + transport
- Gateway transport type for bridged connections
- Interface binding for multi-homed gateways

### 3. IPv6 Address Architecture

Matter is IPv6-native with these address types:

| Address Type | Prefix | Usage |
|--------------|--------|-------|
| Link-Local | `fe80::/10` | Same-link communication |
| Unique Local Address (ULA) | `fd00::/8` | Fabric-scoped networks |
| Global Unicast | `2000::/3` | Internet-routable |
| Multicast | `ff00::/8` | Group communication |

**Key Implementation Details**:

1. **IPv4-Mapped IPv6**: Internal storage uses `::ffff:x.x.x.x` format
2. **Interface Scopes**: Link-local addresses include interface ID (`fe80::1%wlan0`)
3. **ULA Generation**: Fabric ID generates 40-bit global ID + 16-bit subnet
4. **Multicast Addresses**: Generated from fabric ID + group ID

```cpp
// Matter's multicast address generation
static PeerAddress Multicast(FabricId fabric, GroupId group) {
    const uint64_t prefix = 0xfd00000000000000 | ((fabric >> 8) & 0x00ffffffffffffff);
    uint32_t groupId = static_cast<uint32_t>((fabric << 24) & 0xff000000) | group;
    return UDP(IPAddress::MakeIPv6PrefixMulticast(scope, prefixLength, prefix, groupId));
}
```

**TNG Adoption**:
- Generate TNG mesh-local ULA prefixes from hashname fragments
- Support IPv6 multicast for group channels
- Implement address scope handling for multi-interface gateways

### 4. Route Advertisement and Discovery

Matter/OpenThread uses Network Data for route information:

```cpp
bool _HaveRouteToAddress(const IPAddress & destAddr) {
    otNetworkDataIterator routeIter = OT_NETWORK_DATA_ITERATOR_INIT;
    otExternalRouteConfig routeConfig;
    
    while (otNetDataGetNextRoute(instance, &routeIter, &routeConfig) == OT_ERROR_NONE) {
        if (ToIPPrefix(routeConfig.mPrefix).MatchAddress(destAddr)) {
            return true;
        }
    }
    return false;
}
```

**TNG Adoption**: Implement route advertisement in mesh protocol:
- Gateway nodes advertise reachable prefixes
- Mesh routing considers external reachability
- Prefix delegation for subnet management

### 5. Service Discovery Bridging

Matter bridges two service discovery protocols:

| Network | Protocol | Implementation |
|---------|----------|----------------|
| IP (WiFi/Ethernet) | mDNS/DNS-SD | Standard multicast DNS |
| Thread | SRP | Service Registration Protocol to Border Router |

The Border Router runs an SRP server that Thread devices register with, then advertises those services via mDNS on the IP network.

**TNG Adoption**:
- Gateway acts as service registry for mesh devices
- Proxies hashname lookups to/from DNS
- Advertises TNG endpoints via mDNS `_tng._udp`

### 6. Platform Endpoint Abstraction

Matter has platform-specific endpoint implementations:

```
UDPEndPoint (abstract)
├── UDPEndPointImplSockets     (POSIX/BSD sockets)
├── UDPEndPointImplLwIP        (LwIP for RTOS)
├── UDPEndPointImplOpenThread  (Direct Thread stack)
└── UDPEndPointImplNetworkFramework (Apple platforms)
```

Each implementation handles:
- Binding to addresses/ports
- Multicast group management
- Interface-specific operations
- Platform threading/locking

**TNG Adoption**: Similar structure for our transports:
```
TNG Transport (abstract)
├── QUIC Transport (desktop/server)
├── WebTransport (browser)
├── Bluetooth Transport (mobile)
├── Physical Transports
│   ├── 802.15.4 (Zigbee/Thread compatible)
│   ├── LoRa
│   └── UART Radio
└── Gateway Transport (bridge to other TNG segments)
```

### 7. Dataset/Configuration Management

Thread uses Operational Datasets for network configuration:

```cpp
struct OperationalDataset {
    // Network identification
    char networkName[16];
    uint16_t panId;
    uint64_t extendedPanId;
    
    // Security
    uint8_t networkKey[16];
    uint8_t pskc[16];
    
    // Timing
    uint64_t activeTimestamp;
    uint32_t channelMask;
    uint8_t channel;
};
```

**Failsafe Pattern**: Atomic configuration changes with rollback:
1. Arm failsafe timer
2. Apply pending configuration
3. If successful: commit and disarm
4. If failed/timeout: revert to previous

**TNG Adoption**: Network segment configuration:
```python
class MeshSegmentConfig:
    segment_id: bytes        # Unique segment identifier
    mesh_key: bytes          # Segment encryption key
    gateway_hashnames: list  # Authorized gateways
    allowed_transports: set  # Physical transports
    prefix: str              # IPv6 prefix for segment
```

## Proposed TNG Gateway Architecture

### Gateway Roles

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TNG Gateway Node                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │   IP-Side    │    │   Gateway    │    │ Mesh-Side    │          │
│  │  Transport   │◄──►│    Core      │◄──►│  Transport   │          │
│  │  (QUIC/WT)   │    │              │    │ (LoRa/15.4)  │          │
│  └──────────────┘    └──────────────┘    └──────────────┘          │
│         │                   │                   │                   │
│         │            ┌──────┴──────┐            │                   │
│         │            │             │            │                   │
│         ▼            ▼             ▼            ▼                   │
│  ┌────────────┐ ┌─────────┐ ┌──────────┐ ┌────────────┐           │
│  │   mDNS     │ │ Route   │ │ Address  │ │  Hashname  │           │
│  │  Proxy     │ │  Table  │ │   Pool   │ │  Registry  │           │
│  └────────────┘ └─────────┘ └──────────┘ └────────────┘           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### IPv6 Tunneling Model

Inspired by Thread Border Router's IPv6-native approach:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     IPv6 Address Assignment                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Gateway obtains:                                                    │
│  ├── Global prefix from ISP/router (2001:db8::/64)                  │
│  └── Generates mesh ULA prefix (fdXX:XXXX:XXXX::/48)                │
│      (XX derived from gateway's hashname)                           │
│                                                                      │
│  Mesh devices receive:                                               │
│  ├── Mesh-local ULA: fdXX:XXXX:XXXX:SSSS:IID                       │
│  │   (SSSS = segment, IID = derived from device hashname)           │
│  └── Global (if gateway delegates): 2001:db8::IID                   │
│                                                                      │
│  Routing:                                                            │
│  ├── Gateway advertises mesh prefix to IP network                   │
│  ├── Gateway routes packets between prefixes                        │
│  └── NAT66 optional for non-delegated networks                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Encapsulation Format

For tunneling TNG traffic over IP:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Gateway Tunnel Packet                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Outer Layer (QUIC/UDP):                                            │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ IPv6 Header │ UDP/QUIC │ TNG Gateway Header │ Inner Packet  │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  TNG Gateway Header (16 bytes):                                      │
│  ┌────────────┬────────────┬────────────┬────────────────────┐      │
│  │ Version(1) │ Type(1)    │ Flags(2)   │ Segment ID(4)      │      │
│  ├────────────┴────────────┴────────────┴────────────────────┤      │
│  │ Destination Hashname Fragment (8 bytes)                    │      │
│  └───────────────────────────────────────────────────────────┘      │
│                                                                      │
│  Types:                                                              │
│  - 0x01: Encapsulated E3X packet                                    │
│  - 0x02: Route advertisement                                        │
│  - 0x03: Hashname resolution request                                │
│  - 0x04: Hashname resolution response                               │
│  - 0x05: Keepalive                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## Implementation Recommendations

### Phase 1: Gateway Transport

1. **Define Gateway Path Format**:
```json
{
  "type": "gateway",
  "gateway_hashname": "kw3a...kkq",
  "gateway_ip": "2001:db8::1",
  "gateway_port": 42424,
  "segment_id": "0x12345678",
  "target_hashname": "mzxw...abc"
}
```

2. **Gateway Transport Interface**:
```python
class GatewayTransport(Transport):
    async def connect_via_gateway(
        self,
        gateway_path: dict,
        target_hashname: str
    ) -> Pipe:
        """Connect to a mesh device through a gateway"""
        
    async def register_mesh_device(
        self,
        device_hashname: str,
        physical_path: dict
    ) -> None:
        """Register a local mesh device for gateway access"""
```

### Phase 2: IPv6 Address Integration

1. **Hashname to IPv6 IID Mapping**:
```python
def hashname_to_iid(hashname: str) -> bytes:
    """Generate IPv6 Interface Identifier from hashname"""
    # Use first 8 bytes of hashname (already cryptographic)
    hn_bytes = base32_decode(hashname)
    return hn_bytes[:8]

def generate_mesh_ula(gateway_hashname: str, segment: int) -> str:
    """Generate ULA prefix for mesh segment"""
    global_id = hashname_to_iid(gateway_hashname)[:5]  # 40 bits
    return f"fd{global_id.hex()}:{segment:04x}::/64"
```

2. **Optional Direct IPv6 Socket Mode**:
```python
class IPv6MeshEndpoint:
    """Allow direct IPv6 socket access to mesh devices"""
    
    def bind(self, address: str, port: int):
        """Bind to mesh-local IPv6 address"""
        
    async def send_to(self, dest_addr: str, data: bytes):
        """Send to mesh device by IPv6 address"""
```

### Phase 3: Service Discovery Bridge

1. **mDNS Advertisement**:
```python
class TngMdnsProxy:
    async def advertise_mesh_device(
        self,
        hashname: str,
        service_type: str = "_tng._udp",
        txt_records: dict = None
    ):
        """Advertise mesh device via mDNS"""
        
    async def resolve_tng_service(
        self,
        name: str
    ) -> Optional[str]:
        """Resolve mDNS name to hashname"""
```

2. **Service Record Format**:
```
_tng._udp.local.
  SRV 0 0 42424 gateway.local.
  TXT "hn=kw3a...kkq" "cs4a=..." "seg=12345678"
```

## Security Considerations

### Gateway Authentication

- Gateways must be authorized by mesh segment configuration
- E3X encryption maintained end-to-end (gateway cannot decrypt)
- Gateway only sees hashnames and encrypted packets
- Optional: Gateway signs route advertisements

### Address Privacy

- IPv6 IID derived from hashname provides correlation risk
- Consider: Rotating addresses with ephemeral keys
- Consider: Privacy addresses with gateway-mediated mapping

### Trust Model

```
Device A ←──E3X──► Gateway ←──E3X──► Device B
              │
              └── Gateway sees: hashnames, packet sizes, timing
                  Gateway cannot: decrypt content, forge packets
```

## Next Steps

1. **Specification**: Draft TNG Gateway Protocol specification
2. **Prototype**: Implement basic gateway transport in reference implementation
3. **Testing**: Set up LoRa + QUIC gateway testbed
4. **Documentation**: Add gateway setup guide to docs

## References

- [Matter SDK Documentation](https://project-chip.github.io/connectedhomeip-doc/index.html)
- [OpenThread Border Router](https://openthread.io/guides/border-router)
- [RFC 4193 - Unique Local IPv6 Addresses](https://tools.ietf.org/html/rfc4193)
- [RFC 6775 - 6LoWPAN Neighbor Discovery](https://tools.ietf.org/html/rfc6775)
- [Thread 1.3 Specification](https://www.threadgroup.org/support#specifications)
