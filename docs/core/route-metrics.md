# Route Metrics

Route metrics define how TNG selects optimal paths through the mesh. This specification defines two tiers:

- **Core Metrics**: Required for all implementations
- **Advanced Metrics**: Optional, for gateway-capable nodes and optimized routing

## Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    ADVANCED METRICS (Optional)                   │
│  Path cost, bandwidth, latency, reliability, transport flags    │
│  → For gateway nodes and optimized routing                      │
├─────────────────────────────────────────────────────────────────┤
│                    CORE METRICS (Required)                       │
│  Hop count, sequence number                                      │
│  → Must be implemented by all TNG nodes                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## Core Metrics — Required

All TNG implementations that perform routing MUST support these metrics.

### Metric Fields

| Field | Type | Description |
|-------|------|-------------|
| `hops` | u8 | Number of hops to destination (0-64) |
| `seqno` | u32 | Route sequence number for freshness |

### Core Metric Structure

```python
@dataclass
class CoreRouteMetric:
    hops: int       # Number of hops (0 = directly connected, max 64)
    seqno: int      # 32-bit sequence number
```

### Hop Count

The primary routing metric. Lower hop count = preferred route.

| Value | Meaning |
|-------|---------|
| 0 | Directly connected (local link) |
| 1-16 | Typical mesh paths |
| 17-64 | Extended paths (specialized deployments) |
| >64 | Invalid (exceeds maximum) |

**Default maximum**: 16 hops  
**Absolute maximum**: 64 hops

```python
MAX_HOPS = 16  # Default maximum hop count for route selection
MAX_HOPS_ABSOLUTE = 64  # Absolute maximum (specialized deployments)
```

### Sequence Number

Identifies route freshness and prevents stale routes from being reactivated.

**Rules:**
1. Route source increments `seqno` when routes change
2. Receivers reject routes with `seqno < last_seen_seqno`
3. Same `seqno` with better metric is accepted
4. Wrap-around handled via unsigned comparison

See [Routing - Loop Prevention](routing.md#loop-prevention) for implementation details.

### Core Route Selection

With only core metrics, route selection uses:

```python
def select_route(candidates: list[RouteEntry]) -> RouteEntry:
    """Select best route using core metrics only"""
    # Filter valid routes
    MAX_HOPS = 16  # Default maximum hop count
    valid = [r for r in candidates if r.hops <= MAX_HOPS]
    
    if not valid:
        return None
    
    # Sort by: hop count (asc), then seqno (desc), then stability
    valid.sort(key=lambda r: (r.hops, -r.seqno, -r.updated_at))
    
    return valid[0]
```

---

## Advanced Metrics — Optional

Advanced metrics enable optimized routing for gateway-capable nodes. These are **not required** for basic TNG operation.

### Extended Metric Fields

| Field | Type | Description |
|-------|------|-------------|
| `hops` | u8 | Number of hops (from core) |
| `seqno` | u32 | Sequence number (from core) |
| `path_cost` | u32 | Cumulative weighted path cost |
| `min_bandwidth_kbps` | u32 | Bottleneck bandwidth on path |
| `max_latency_ms` | u16 | Worst-case one-way latency |
| `reliability_pct` | u8 | Path reliability (0-100%) |
| `transport_flags` | u16 | Capability intersection of path |

### Advanced Metric Structure

```python
@dataclass
class AdvancedRouteMetric(CoreRouteMetric):
    path_cost: int          # Cumulative cost (lower = better)
    min_bandwidth_kbps: int # Bottleneck bandwidth
    max_latency_ms: int     # Worst-case latency
    reliability_pct: int    # 0-100 reliability
    transport_flags: int    # Capability bitmask
```

### Path Cost Calculation

Path cost aggregates per-hop costs across the route:

```
Per-hop cost:
  hop_cost = base_cost
           + (1000 / bandwidth_kbps)     // Bandwidth penalty
           + (latency_ms / 10)           // Latency penalty
           + ((100 - reliability) * 5)   // Reliability penalty

Total path cost:
  path_cost = sum(hop_costs) + (hops * 100)
```

### Transport Capability Flags

Bitmask indicating what traffic types the path can carry:

| Bit | Flag | Description |
|-----|------|-------------|
| 0 | `MESH_RELAY` | Can relay E3X mesh traffic |
| 1 | `GATEWAY_ROUTE` | Valid path to exit gateway |
| 2 | `IP_TUNNEL` | Can carry IP tunnel traffic |
| 3 | `TCP_PROXY` | Can carry TCP proxy channels |
| 4 | `UDP_PROXY` | Can carry UDP proxy channels |
| 5 | `CONSTRAINED` | Low bandwidth/high latency path |
| 6-15 | Reserved | For future use |

```python
class TransportFlags:
    MESH_RELAY    = 0x0001
    GATEWAY_ROUTE = 0x0002
    IP_TUNNEL     = 0x0004
    TCP_PROXY     = 0x0008
    UDP_PROXY     = 0x0010
    CONSTRAINED   = 0x0020
```

Path flags are the **intersection** of all hop flags:
```python
path_flags = hop1_flags & hop2_flags & hop3_flags
```

---

## Transport Profiles — Advanced

Pre-defined metric profiles for each transport type. Used when calculating path costs.

### Logical Transports

| Transport | Base Cost | Bandwidth | Latency | Reliability | Flags |
|-----------|-----------|-----------|---------|-------------|-------|
| **QUIC** | 10 | 100 Mbps | 20ms | 99% | All |
| **WebTransport** | 15 | 50 Mbps | 50ms | 98% | All |
| **Bluetooth LE** | 50 | 100 KB/s | 30ms | 95% | TCP_PROXY, MESH_RELAY, GATEWAY_ROUTE, CONSTRAINED |

### Physical Transports

| Transport | Base Cost | Bandwidth | Latency | Reliability | Flags |
|-----------|-----------|-----------|---------|-------------|-------|
| **802.11 WiFi** | 20 | 50 Mbps | 5ms | 97% | All |
| **802.15.4** | 100 | 250 kbps | 15ms | 90% | MESH_RELAY, GATEWAY_ROUTE, CONSTRAINED |
| **LoRa** | 500 | 5 kbps | 500ms | 85% | MESH_RELAY, CONSTRAINED |
| **433/915MHz** | 400 | 10 kbps | 100ms | 80% | MESH_RELAY, CONSTRAINED |

### Transport Profile Definition

```python
TRANSPORT_PROFILES = {
    "quic": TransportProfile(
        base_cost=10,
        bandwidth_kbps=100_000,
        latency_ms=20,
        reliability_pct=99,
        flags=TransportFlags.MESH_RELAY | TransportFlags.GATEWAY_ROUTE |
              TransportFlags.IP_TUNNEL | TransportFlags.TCP_PROXY |
              TransportFlags.UDP_PROXY
    ),
    "lora": TransportProfile(
        base_cost=500,
        bandwidth_kbps=5,
        latency_ms=500,
        reliability_pct=85,
        duty_cycle_pct=1,  # EU868 regulatory limit
        flags=TransportFlags.MESH_RELAY | TransportFlags.CONSTRAINED
    ),
    # ... other transports
}
```

---

## IP Routing Suitability — Advanced

Not all transports can carry IP/gateway traffic. This table indicates suitability:

| Transport | IP Tunnel | TCP Proxy | UDP Proxy | Gateway Route | Notes |
|-----------|-----------|-----------|-----------|---------------|-------|
| **QUIC** | ✓ | ✓ | ✓ | ✓ | Primary IP transport |
| **WebTransport** | ✓ | ✓ | ✓ | ✓ | Browser environments |
| **Bluetooth LE** | ✗ | ✓ | ✗ | ✓ | Marginal, needs fragmentation |
| **802.11 WiFi** | ✓ | ✓ | ✓ | ✓ | Usually use QUIC over WiFi |
| **802.15.4** | ✗ | ✗ | ✗ | ✓ | Limited, 6LoWPAN possible |
| **LoRa** | ✗ | ✗ | ✗ | ✗ | **Not suitable for IP** |
| **433/915MHz** | ✗ | ✗ | ✗ | ✗ | **Not suitable for IP** |

### Why LoRa Cannot Route IP Traffic

LoRa is explicitly **excluded from IP routing** due to:

| Constraint | Value | Impact |
|------------|-------|--------|
| **Duty cycle** | 1% (EU) | Only 36 seconds transmit per hour |
| **Latency** | 56-1400ms | Per-packet, depending on SF |
| **Bandwidth** | 0.3-50 kbps | Cannot sustain TCP connections |
| **MTU** | 255 bytes | Extensive fragmentation for IP |

LoRa devices can still reach the internet by routing through a gateway bridge:

```
LoRa sensor → (LoRa) → Gateway node → (QUIC) → Exit Gateway → Internet
                           │
                           └── LoRa segment is NOT part of IP path
```

---

## Gateway Route Filtering — Advanced

Routes to exit gateways MUST only traverse transports with `GATEWAY_ROUTE` capability:

```python
def is_valid_gateway_path(route: RouteEntry) -> bool:
    """Check if route can reach exit gateway"""
    flags = route.transport_flags
    
    # Must have gateway route capability
    if not (flags & TransportFlags.GATEWAY_ROUTE):
        return False
    
    # Reject paths through LoRa/UART for gateway traffic
    if is_constrained_only_path(route):
        return False
    
    return True

def is_constrained_only_path(route: RouteEntry) -> bool:
    """Check if path only has constrained transports (no IP-capable hops)"""
    flags = route.transport_flags
    ip_capable = TransportFlags.IP_TUNNEL | TransportFlags.TCP_PROXY
    return (flags & ip_capable) == 0 and (flags & TransportFlags.CONSTRAINED)
```

---

## Advanced Route Selection

With advanced metrics, route selection considers multiple factors:

```python
def select_route_advanced(
    candidates: list[RouteEntry],
    need_gateway: bool = False,
    min_bandwidth: int = 0
) -> RouteEntry:
    """Select best route using advanced metrics"""
    
    # Filter by requirements
    valid = candidates
    
    if need_gateway:
        valid = [r for r in valid if is_valid_gateway_path(r)]
    
    if min_bandwidth > 0:
        valid = [r for r in valid if r.min_bandwidth_kbps >= min_bandwidth]
    
    if not valid:
        return None
    
    # Sort by composite cost
    valid.sort(key=lambda r: (
        r.path_cost,           # Primary: total path cost
        r.hops,                # Secondary: hop count
        -r.min_bandwidth_kbps, # Tertiary: prefer higher bandwidth
        r.max_latency_ms       # Quaternary: prefer lower latency
    ))
    
    return valid[0]
```

---

## Metric Propagation

When propagating routes, metrics are updated at each hop:

```python
def propagate_metric(incoming: AdvancedRouteMetric, link: Link) -> AdvancedRouteMetric:
    """Update metrics when propagating route through a link"""
    transport = link.active_transport
    profile = TRANSPORT_PROFILES[transport.type]
    
    return AdvancedRouteMetric(
        hops=incoming.hops + 1,
        seqno=incoming.seqno,  # Preserved from source
        path_cost=incoming.path_cost + calculate_hop_cost(profile),
        min_bandwidth_kbps=min(incoming.min_bandwidth_kbps, profile.bandwidth_kbps),
        max_latency_ms=max(incoming.max_latency_ms, profile.latency_ms),
        reliability_pct=int(incoming.reliability_pct * profile.reliability_pct / 100),
        transport_flags=incoming.transport_flags & profile.flags
    )
```

---

## Implementation Requirements

### Core (All Implementations)

- Track `hops` and `seqno` for all routes
- Implement hop limit enforcement (see [Routing](routing.md))
- Select routes by lowest hop count

### Advanced (Gateway Nodes)

- Calculate and propagate composite metrics
- Track transport capability flags
- Filter gateway routes by transport suitability
- Support bandwidth/latency-aware route selection

### Constrained Devices

Minimal implementations (LoRa sensors, etc.) need only:
- ~50 bytes for route table
- Hop count and seqno tracking
- No advanced metrics required

---

*Related: [Routing](routing.md) | [Mesh](mesh.md) | [Gateway](gateway.md) | [Transports](../transports/README.md)*
