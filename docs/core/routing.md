# Routing

**Routing** in TNG enables endpoints to communicate even when they cannot establish a direct connection. A **router** is an endpoint that forwards packets between two other endpoints, enabling NAT traversal and connectivity across network boundaries.

## Overview

```
     Endpoint A                Router                 Endpoint B
    (behind NAT)             (public IP)            (behind NAT)
         │                        │                        │
         │◄──── Link (A↔R) ──────►│◄──── Link (R↔B) ──────►│
         │                        │                        │
         │ ═══════════════════════╪════════════════════════│
         │        Logical Link (A↔B) via Router           │
         │                        │                        │
```

## Router Role

A router is simply an endpoint that:
1. Has a [link](links.md) to at least one of the endpoints
2. Is willing to forward packets between endpoints
3. Is explicitly trusted by the requesting endpoint

### Router Requirements

| Requirement | Description |
|-------------|-------------|
| **Reachable** | Must be connectable by requesting endpoints |
| **Trusted** | Must be designated as router by the application |
| **Willing** | Must accept peer/connect channel requests |
| **Capable** | Sufficient bandwidth and resources |

## Routing Channels

Routing uses two special channel types:

### `peer` Channel

Sent **to** a router to request connection assistance:

```json
HEAD: {
  "c": 1,
  "type": "peer",
  "seq": 0,
  "hops": 16
}
BODY: {
  "hashname": "target-hashname",
  "keys": { "4a": "base32-csk" },
  "paths": [...]
}
```

The `hops` field is **required** for loop prevention (see [Loop Prevention](#loop-prevention)).

The router:
1. Looks up the target hashname
2. Opens a `connect` channel to the target
3. Forwards handshake packets between endpoints

### `connect` Channel

Sent **from** a router to notify of incoming connection:

```json
HEAD: {
  "c": 2,
  "type": "connect",
  "seq": 0,
  "hops": 15
}
BODY: {
  "hashname": "requesting-hashname",
  "keys": { "4a": "base32-csk" },
  "paths": [...]
}
```

The `hops` field is decremented by the router before forwarding (see [Loop Prevention](#loop-prevention)).

The target can:
- Accept by responding with handshake
- Reject by closing the channel
- Ignore (timeout)

## Routing Flow

### 1. Peer Request

```
    Endpoint A                   Router                   Endpoint B
         │                          │                          │
         │ ─── peer{B's hashname} ─►│                          │
         │                          │                          │
```

### 2. Connect Notification

```
    Endpoint A                   Router                   Endpoint B
         │                          │                          │
         │                          │ ─── connect{A's info} ──►│
         │                          │                          │
```

### 3. Handshake Relay

```
    Endpoint A                   Router                   Endpoint B
         │                          │                          │
         │ ─── handshake ──────────►│─── handshake ───────────►│
         │                          │                          │
         │ ◄─── handshake ──────────│◄─── handshake ───────────│
         │                          │                          │
```

### 4. Direct Connection (Optional)

After handshake, endpoints may establish direct connection:

```
    Endpoint A                   Router                   Endpoint B
         │                          │                          │
         │ ◄════════════════════════╪══════════════════════════│
         │      Direct Link (if possible)                      │
```

### 5. Continued Relay (If Direct Fails)

If direct connection fails, router continues relaying:

```
    Endpoint A                   Router                   Endpoint B
         │                          │                          │
         │ ─── channel data ───────►│─── channel data ────────►│
         │ ◄─── channel data ───────│◄─── channel data ────────│
         │                          │                          │
```

## Router Implementation

### Accepting Router Role

```python
class Router:
    def __init__(self, mesh: Mesh):
        self.mesh = mesh
        self.relays: dict[tuple[str, str], RelayState] = {}
    
    def on_peer_request(self, link: Link, channel: Channel, request: dict):
        """Handle incoming peer request"""
        target_hashname = request["hashname"]
        
        # Check if we know the target
        target_link = self.mesh.links.get(target_hashname)
        if not target_link or target_link.state != "up":
            channel.send({"error": "unknown"})
            return
        
        # Open connect channel to target
        connect_channel = target_link.open_channel("connect")
        connect_channel.send({
            "hashname": link.hashname,
            "keys": request.get("keys"),
            "paths": request.get("paths")
        })
        
        # Set up relay
        self._create_relay(link, channel, target_link, connect_channel)
```

### Relay State

```python
class RelayState:
    def __init__(self, peer_a: Link, peer_b: Link):
        self.peer_a = peer_a
        self.peer_b = peer_b
        self.bytes_relayed = 0
        self.created_at = time.time()
```

## Rate Limiting

Routers should implement rate limiting to prevent abuse:

```python
class RateLimiter:
    def __init__(self):
        self.limits = {
            "relay_bytes_per_second": 100_000,  # 100 KB/s per relay
            "max_relays_per_peer": 10,
            "relay_timeout": 300,  # 5 minutes idle timeout
        }
    
    def check_relay(self, relay: RelayState) -> bool:
        # Check bandwidth
        if relay.bytes_per_second > self.limits["relay_bytes_per_second"]:
            return False
        
        # Check timeout
        if relay.idle_time > self.limits["relay_timeout"]:
            return False
        
        return True
```

## Path Advertisement

Routers can advertise themselves in path exchanges:

```json
{
  "type": "peer",
  "router": "router-hashname"
}
```

When an endpoint receives this path, it knows the target is reachable via that router.

## Privacy Considerations

### What Routers See

| Information | Visible to Router |
|-------------|-------------------|
| Endpoint hashnames | Yes |
| Connection timing | Yes |
| Data volume | Yes |
| Channel content | **No** (encrypted) |
| Channel types | **No** (encrypted) |

### Minimizing Router Trust

- Use multiple routers for redundancy
- Rotate routers periodically
- Prefer direct connections when possible
- Use [cloaking](e3x.md#cloaking) to hide traffic patterns

## Multi-Router Scenarios

### Redundant Routing

Use multiple routers for reliability:

```
                   ┌─── Router 1 ───┐
                   │                │
    Endpoint A ────┤                ├──── Endpoint B
                   │                │
                   └─── Router 2 ───┘
```

### Chained Routing

Route through multiple hops (not recommended for latency):

```
    Endpoint A ──── Router 1 ──── Router 2 ──── Endpoint B
```

Each hop adds:
- Latency
- Another trusted party
- Potential failure point

## Router Selection

Applications should select routers based on:

| Criteria | Description |
|----------|-------------|
| **Availability** | Router must be online and reachable |
| **Latency** | Lower latency preferred |
| **Trust** | Must trust router with metadata |
| **Capacity** | Sufficient bandwidth for expected traffic |

```python
def select_router(self, target: str) -> Optional[Link]:
    """Select best router for connecting to target"""
    candidates = []
    
    for router_hn in self.mesh.routers:
        router = self.mesh.links.get(router_hn)
        if router and router.state == "up":
            candidates.append((router, router.latency))
    
    if not candidates:
        return None
    
    # Sort by latency
    candidates.sort(key=lambda x: x[1])
    return candidates[0][0]
```

## Loop Prevention

Loop prevention is **required** for all TNG implementations that perform routing or relay. These mechanisms prevent packets from circulating indefinitely in the mesh and ensure route convergence.

### Hop Limit (TTL) — Required

Every routed packet includes a `hops` field that limits propagation depth:

```
Relayed packet through router chain:

  A ──peer──► R1 ──peer──► R2 ──connect──► B
      hops=16     hops=15       hops=14
```

**Rules:**

1. **Originator** sets `hops` to initial value (default: 16)
2. **Router** decrements `hops` before forwarding
3. **Router** discards packet if `hops` reaches 0
4. **Maximum** configurable value is 64

```python
DEFAULT_HOPS = 16
MAX_HOPS = 64

def forward_packet(self, packet: dict, from_link: Link, to_link: Link):
    """Forward a routed packet, enforcing hop limit"""
    hops = packet.get("hops", DEFAULT_HOPS)
    
    if hops <= 0:
        # Drop packet - TTL exceeded
        self._log_ttl_exceeded(packet, from_link)
        return
    
    # Decrement and forward
    packet["hops"] = hops - 1
    to_link.send(packet)
```

**All routers MUST implement hop limit enforcement.**

### Route Sequence Numbers — Required

Each route source maintains a 32-bit sequence number that identifies route freshness:

| Field | Type | Description |
|-------|------|-------------|
| `seqno` | u32 | Incremented when routes change |

**Rules:**

1. **Source** (router or gateway) increments `seqno` when its routes change
2. **Receivers** only accept routes with `seqno >= last_seen_seqno`
3. **Duplicate** `seqno` from same source is ignored (already processed)
4. **Wrap-around** handled via unsigned comparison with window

```python
class RouteTable:
    def __init__(self):
        self.routes: dict[str, RouteEntry] = {}  # destination -> entry
    
    def update_route(self, dest: str, via: str, seqno: int, hops: int) -> bool:
        """Update route if sequence number is newer or equal with better metric"""
        existing = self.routes.get(dest)
        
        if existing:
            # Reject older sequence numbers
            if seqno < existing.seqno:
                return False
            
            # Same seqno: only accept if better metric
            if seqno == existing.seqno and hops >= existing.hops:
                return False
        
        self.routes[dest] = RouteEntry(
            destination=dest,
            via=via,
            seqno=seqno,
            hops=hops,
            updated_at=time.time()
        )
        return True
```

**All nodes maintaining route tables MUST implement sequence number tracking.**

### Feasibility Conditions — Recommended

Feasibility conditions (inspired by Babel RFC 8966) provide additional loop-freedom guarantees during route convergence:

| Concept | Description |
|---------|-------------|
| **Feasible Distance (FD)** | Best metric ever seen for a route to destination |
| **Reported Distance (RD)** | Metric advertised by neighbor |
| **Feasibility Condition** | Accept route only if RD < FD |

**Rules:**

1. **Track FD** for each destination (best hop count ever seen)
2. **Accept** new route only if its metric is ≤ FD
3. **Reset FD** when sequence number increases (new route epoch)
4. **Starvation recovery**: Request new seqno from source if no feasible route exists

```python
class FeasibilityTracker:
    def __init__(self):
        self.feasible_distance: dict[str, tuple[int, int]] = {}  # dest -> (seqno, hops)
    
    def is_feasible(self, dest: str, seqno: int, hops: int) -> bool:
        """Check if route meets feasibility condition"""
        if dest not in self.feasible_distance:
            # First route to this destination - always feasible
            self.feasible_distance[dest] = (seqno, hops)
            return True
        
        stored_seqno, stored_hops = self.feasible_distance[dest]
        
        if seqno > stored_seqno:
            # New epoch - reset feasible distance
            self.feasible_distance[dest] = (seqno, hops)
            return True
        
        if seqno == stored_seqno and hops <= stored_hops:
            # Same epoch, better or equal metric - feasible
            if hops < stored_hops:
                self.feasible_distance[dest] = (seqno, hops)
            return True
        
        # Worse metric than feasible distance - reject
        return False
```

**Recommended for routers; optional for leaf nodes.**

### Split Horizon — Required

Never advertise a route back to the peer from which it was learned:

```
    Gateway G                    Router R                    Node N
        │                            │                          │
        │ ─── route_info{G} ───────► │                          │
        │                            │ ─── route_info{G} ──────►│
        │                            │                          │
        │ ◄─ NO route_info{G} back ─ │  (split horizon)         │
```

```python
def propagate_route(self, route: RouteEntry, exclude_peer: str = None):
    """Propagate route to peers, respecting split horizon"""
    for peer_hashname, link in self.mesh.links.items():
        # Split horizon: don't send back to source
        if peer_hashname == route.via:
            continue
        
        # Don't send to explicitly excluded peer
        if peer_hashname == exclude_peer:
            continue
        
        # Only propagate if we're configured as a router
        if self.is_router:
            self._send_route_info(link, route)
```

### Route Table Structure

Nodes maintaining routes use this structure:

```python
@dataclass
class RouteEntry:
    destination: str      # Target hashname (or gateway hashname for exit routes)
    via: str              # Next-hop peer hashname
    seqno: int            # Route sequence number
    hops: int             # Hop count to destination
    updated_at: float     # Timestamp of last update
    expires_at: float     # When route expires (updated_at + TTL)

class RouteTable:
    routes: dict[str, RouteEntry]           # destination -> best route
    feasible_distance: dict[str, tuple]     # destination -> (seqno, best_hops)
    
    def get_route(self, destination: str) -> Optional[RouteEntry]:
        """Get best route to destination, if valid"""
        entry = self.routes.get(destination)
        if entry and time.time() < entry.expires_at:
            return entry
        return None
    
    def expire_routes(self):
        """Remove expired routes"""
        now = time.time()
        expired = [d for d, r in self.routes.items() if now >= r.expires_at]
        for dest in expired:
            del self.routes[dest]
```

### Route Metrics — Core

For basic route selection, TNG uses hop count as the primary metric:

| Metric | Type | Description |
|--------|------|-------------|
| `hops` | u8 | Number of hops to destination (lower = better) |

Route selection prefers:
1. Lower hop count
2. More recent sequence number (if hop count equal)
3. Existing route (stability, if metrics equal)

For advanced metrics including bandwidth, latency, and transport capabilities, see [Route Metrics](route-metrics.md).

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `unknown` | Router doesn't know target | Try another router |
| `rejected` | Target rejected connection | Report to application |
| `timeout` | No response from target | Retry or try another router |
| `rate_limited` | Router limiting traffic | Back off, try later |
| `ttl_exceeded` | Hop limit reached zero | Packet dropped silently (logged) |
| `stale_route` | Sequence number too old | Request fresh route |

## Becoming a Router

To offer routing services:

```python
# Enable routing for specific links
mesh.accept_routing_for(["friend-hashname-1", "friend-hashname-2"])

# Or enable for all linked peers
mesh.accept_routing_for_all()

# Handle incoming peer requests
@mesh.on("peer_request")
def on_peer_request(requesting_link, target_hashname):
    if is_allowed(requesting_link, target_hashname):
        return True  # Accept
    return False  # Reject
```

## Security Considerations

### Router Trust

- Router sees connection metadata
- Compromised router can deny service
- Compromised router cannot read encrypted content
- Consider router reputation/selection carefully

### Denial of Service

- Routers are potential DDoS targets
- Implement rate limiting
- Consider proof-of-work for relay requests
- Monitor for abuse patterns

### Anonymity

TNG routing does **not** provide anonymity:
- Router knows both endpoints
- Use Tor or similar if anonymity required

---

*Related: [Links](links.md) | [Mesh](mesh.md) | [Channels](channels.md) | [Route Metrics](route-metrics.md)*
