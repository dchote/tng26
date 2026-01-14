# Mesh

A **Mesh** is an endpoint's collection of [links](links.md) and the management layer for network connectivity. Each endpoint maintains its own private view of the mesh—there is no global shared state.

## Overview

```
                         Your Endpoint
                        ┌─────────────┐
                        │    Mesh     │
                        └──────┬──────┘
           ┌───────────────────┼───────────────────┐
           │                   │                   │
     ┌─────▼─────┐       ┌─────▼─────┐       ┌─────▼─────┐
     │  Link A   │       │  Link B   │       │  Link C   │
     │  (peer)   │       │ (router)  │       │  (peer)   │
     └───────────┘       └───────────┘       └───────────┘
```

## Properties

| Property | Description |
|----------|-------------|
| **Private** | Your mesh topology is not shared with others |
| **Local** | Each endpoint has its own independent view |
| **Dynamic** | Links can be added, removed, or change state |
| **Flat** | No hierarchy; all links are peers (routers are just peers with a role) |

## Mesh Components

### Links Collection

The mesh manages all links:

```python
class Mesh:
    def __init__(self, local_keys: dict):
        self.hashname = generate_hashname(local_keys)
        self.links: dict[str, Link] = {}  # hashname -> Link
        self.routers: list[str] = []  # hashnames of routers
        self.routes: RouteTable = RouteTable()  # route table for destinations
        self.discovery_mode = False
        self.is_router = False  # Whether this endpoint acts as a router
    
    def add_link(self, link_json: dict) -> Link:
        """Add or update a link from JSON"""
        hashname = link_json.get("hashname")
        if not hashname:
            hashname = generate_hashname(link_json["keys"])
        
        if hashname in self.links:
            link = self.links[hashname]
            link.update(link_json)
        else:
            link = Link(hashname)
            link.update(link_json)
            self.links[hashname] = link
        
        return link
```

### Router Designation

Some links can be designated as **routers** (see [routing](routing.md)):

```python
def set_router(self, hashname: str, is_router: bool):
    """Mark a link as a trusted router"""
    if is_router:
        if hashname not in self.routers:
            self.routers.append(hashname)
    else:
        if hashname in self.routers:
            self.routers.remove(hashname)
```

## Route Table

The mesh maintains a route table for tracking paths to destinations beyond directly-linked peers. This is essential for [loop prevention](routing.md#loop-prevention) and efficient routing.

### Route Entry

Each route entry contains:

```python
@dataclass
class RouteEntry:
    destination: str      # Target hashname
    via: str              # Next-hop peer hashname
    seqno: int            # Route sequence number (for freshness)
    hops: int             # Hop count metric
    updated_at: float     # When route was last updated
    expires_at: float     # When route expires
```

### Route Table Implementation

```python
class RouteTable:
    DEFAULT_TTL = 300  # 5 minutes
    
    def __init__(self):
        self.routes: dict[str, RouteEntry] = {}
        self.feasible_distance: dict[str, tuple[int, int]] = {}  # dest -> (seqno, hops)
    
    def update(self, dest: str, via: str, seqno: int, hops: int) -> bool:
        """
        Update route if valid. Returns True if route was accepted.
        
        Implements:
        - Sequence number validation (reject stale routes)
        - Feasibility condition (loop prevention)
        - Split horizon (caller's responsibility)
        """
        # Check sequence number
        existing = self.routes.get(dest)
        if existing and seqno < existing.seqno:
            return False  # Reject older sequence
        
        # Check feasibility condition
        if not self._is_feasible(dest, seqno, hops):
            return False  # Would create loop
        
        # Accept route
        now = time.time()
        self.routes[dest] = RouteEntry(
            destination=dest,
            via=via,
            seqno=seqno,
            hops=hops,
            updated_at=now,
            expires_at=now + self.DEFAULT_TTL
        )
        return True
    
    def _is_feasible(self, dest: str, seqno: int, hops: int) -> bool:
        """Check feasibility condition (Babel-style loop prevention)"""
        if dest not in self.feasible_distance:
            self.feasible_distance[dest] = (seqno, hops)
            return True
        
        stored_seqno, stored_hops = self.feasible_distance[dest]
        
        if seqno > stored_seqno:
            # New epoch - reset feasible distance
            self.feasible_distance[dest] = (seqno, hops)
            return True
        
        if seqno == stored_seqno and hops <= stored_hops:
            # Same epoch, acceptable metric
            if hops < stored_hops:
                self.feasible_distance[dest] = (seqno, hops)
            return True
        
        return False  # Worse than feasible distance
    
    def get(self, dest: str) -> Optional[RouteEntry]:
        """Get valid route to destination"""
        entry = self.routes.get(dest)
        if entry and time.time() < entry.expires_at:
            return entry
        return None
    
    def remove(self, dest: str):
        """Remove route to destination"""
        self.routes.pop(dest, None)
    
    def expire(self):
        """Remove all expired routes"""
        now = time.time()
        expired = [d for d, r in self.routes.items() if now >= r.expires_at]
        for dest in expired:
            del self.routes[dest]
            # Clean up feasible_distance for expired routes
            self.feasible_distance.pop(dest, None)
```

### Split Horizon

When propagating routes, never advertise a route back to the peer from which it was learned:

```python
def propagate_routes(self, exclude_peer: str = None):
    """Propagate known routes to peers (respecting split horizon)"""
    for dest, route in self.routes.routes.items():
        for peer_hashname, link in self.links.items():
            # Split horizon: don't send back to source
            if peer_hashname == route.via:
                continue
            
            # Don't send to explicitly excluded peer
            if peer_hashname == exclude_peer:
                continue
            
            # Only propagate if acting as router
            if peer_hashname in self.routers or self.is_router:
                self._send_route_update(link, route)
```

### Route Events

The mesh emits route-related events:

| Event | Data | Description |
|-------|------|-------------|
| `route_added` | RouteEntry | New route learned |
| `route_updated` | RouteEntry | Existing route updated |
| `route_removed` | destination | Route expired or withdrawn |

### Receiving Route Updates

When a peer sends route information:

```python
def on_route_received(self, from_link: Link, route_info: dict):
    """Handle incoming route information from a peer"""
    # Gateway route_info uses "gateway" field, mesh routes use "destination"
    dest = route_info.get("gateway") or route_info.get("destination")
    seqno = route_info["seqno"]
    hops = route_info["hops"] + 1  # Increment hop count
    
    if self.routes.update(dest, from_link.hashname, seqno, hops):
        self._emit("route_updated", self.routes.get(dest))
        
        # Optionally re-propagate (if router)
        if self.is_router:
            self.propagate_routes(exclude_peer=from_link.hashname)
```

## Discovery

Discovery allows endpoints to find each other on local networks without prior knowledge.

### Discovery Mode

When enabled, the endpoint:
1. Broadcasts its presence on supported transports
2. Listens for announcements from other endpoints
3. Reports discoveries to the application

```python
def enable_discovery(self, timeout: float = 30.0):
    """Enable discovery mode temporarily"""
    self.discovery_mode = True
    self._broadcast_presence()
    
    # Auto-disable after timeout
    schedule(timeout, self.disable_discovery)

def disable_discovery(self):
    """Disable discovery mode"""
    self.discovery_mode = False
```

### Discovery Packet

Discovery announcements contain minimal information:

```json
{
  "type": "discover",
  "hashname": "kw3a...kkq",
  "keys": {
    "4a": "base32-csk"
  }
}
```

### Privacy Considerations

Discovery reveals your presence on the local network:

- **Enable sparingly**: Only when actively looking for peers
- **Short duration**: Auto-disable after timeout (e.g., 30 seconds)
- **User-initiated**: Triggered by explicit action ("pair device", "add friend")

### Passive Listening

Even with discovery disabled, endpoints **listen** for announcements:

```python
def on_discover_received(self, announcement: dict, path: Path):
    """Called when discovery announcement received"""
    # Always notify app, even if not announcing
    self._notify_app("discovered", {
        "hashname": announcement["hashname"],
        "keys": announcement["keys"],
        "path": path
    })
```

This allows receiving without revealing your own presence.

## Link Management

### Adding Links

Links can be added from various sources:

```python
# From JSON (stored or received)
link = mesh.add_link({
    "hashname": "kw3a...kkq",
    "keys": {"4a": "..."},
    "paths": [{"type": "quic", "ip": "1.2.3.4", "port": 42424}]
})

# From URI
link = mesh.link_from_uri("tng://kw3a...kkq?cs4a=...&ip=1.2.3.4&port=42424")

# From discovery
link = mesh.add_link(discovered_announcement)

# Just hashname (requires resolution)
link = mesh.add_link({"hashname": "kw3a...kkq"})
```

### Connecting Links

```python
# Connect a specific link
link = mesh.links["kw3a...kkq"]
await link.connect()

# Connect using routers if direct fails
async def connect_with_fallback(self, hashname: str):
    link = self.links[hashname]
    
    # Try direct paths
    for path in link.direct_paths:
        try:
            await link.connect_via(path)
            return
        except ConnectionFailed:
            continue
    
    # Fall back to routers
    for router_hn in self.routers:
        router = self.links[router_hn]
        if router.state == "up":
            await self._peer_request(router, hashname)
            return
    
    raise NoRoutesAvailable()
```

### Removing Links

```python
def remove_link(self, hashname: str):
    """Remove a link from the mesh"""
    if hashname in self.links:
        link = self.links[hashname]
        link.close()
        del self.links[hashname]
```

## Events

The mesh emits events for application handling:

| Event | Data | Description |
|-------|------|-------------|
| `link_created` | Link | New link added to mesh |
| `link_up` | Link | Link connected successfully |
| `link_down` | Link | Link disconnected |
| `link_removed` | hashname | Link removed from mesh |
| `discovered` | announcement | Discovery announcement received |
| `channel_request` | Link, channel | Incoming channel request |

```python
class Mesh:
    def on(self, event: str, handler: Callable):
        """Register event handler"""
        self.handlers[event].append(handler)
    
    def _emit(self, event: str, data):
        for handler in self.handlers.get(event, []):
            handler(data)
```

## Mesh API

### High-Level Interface

```python
# Create mesh with keypairs
mesh = Mesh.create(keypairs)  # or Mesh.generate() for new keys

# Get mesh's own hashname
print(mesh.hashname)

# Add and connect to a peer
link = mesh.link("tng://kw3a...kkq?cs4a=...&ip=1.2.3.4&port=42424")
await link.ready()

# Open a channel
channel = await link.open_channel("stream")
await channel.send(b"Hello!")
data = await channel.receive()

# Handle incoming channels
@mesh.on("channel_request")
def on_channel(link, channel):
    if channel.type == "stream":
        channel.accept()
        process_stream(channel)

# Enable discovery
mesh.discover(timeout=30)

@mesh.on("discovered")
def on_discovered(info):
    print(f"Found peer: {info['hashname']}")
```

### Link State Queries

```python
# Get all links
all_links = mesh.links.values()

# Filter by state
up_links = [l for l in mesh.links.values() if l.state == "up"]

# Get link by hashname
link = mesh.links.get("kw3a...kkq")

# Check if connected
is_connected = link and link.state == "up"
```

## Persistence

Mesh state can be persisted for reconnection:

```python
def save_mesh(self) -> dict:
    """Export mesh state for storage"""
    return {
        "hashname": self.hashname,
        "links": [
            {
                "hashname": link.hashname,
                "keys": link.keys,
                "paths": link.paths,
                "is_router": link.hashname in self.routers
            }
            for link in self.links.values()
        ]
    }

def load_mesh(self, state: dict):
    """Restore mesh state from storage"""
    for link_data in state["links"]:
        link = self.add_link(link_data)
        if link_data.get("is_router"):
            self.set_router(link.hashname, True)
```

## Security Model

### No Global State

Unlike some P2P systems, TNG meshes have no shared global view:
- Your links are your business
- No gossiping of peer information
- No DHT or distributed routing table

### Explicit Trust

Every link must be explicitly accepted:
- Application decides which hashnames to link with
- No auto-acceptance of incoming connections
- Router trust is explicit per-link

### Minimal Exposure

- Discovery is opt-in and time-limited
- Paths are shared only with intended recipients
- Routers see connection metadata but not content

---

*Related: [Links](links.md) | [Routing](routing.md) | [Route Metrics](route-metrics.md) | [Technical Overview](../technical-overview.md)*
