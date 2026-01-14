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
  "seq": 0
}
BODY: {
  "hashname": "target-hashname",
  "keys": { "4a": "base32-csk" },
  "paths": [...]
}
```

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
  "seq": 0
}
BODY: {
  "hashname": "requesting-hashname",
  "keys": { "4a": "base32-csk" },
  "paths": [...]
}
```

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

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `unknown` | Router doesn't know target | Try another router |
| `rejected` | Target rejected connection | Report to application |
| `timeout` | No response from target | Retry or try another router |
| `rate_limited` | Router limiting traffic | Back off, try later |

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

*Related: [Links](links.md) | [Mesh](mesh.md) | [Channels](channels.md)*
