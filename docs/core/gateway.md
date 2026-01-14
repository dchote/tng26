# Gateway

The Gateway enables IP connectivity through a TNG mesh, providing three primary capabilities:

1. **Mesh Internet Access**: Non-TNG clients connect to WiFi served by an Entry Gateway and access the internet via an Exit Gateway elsewhere in the mesh
2. **Mesh VPN**: Encrypted site-to-site IP tunneling over a TNG mesh
3. **Direct Exit Access**: TNG-native applications open TCP/UDP connections directly through Exit Gateways without needing an IP stack

This is essentially a **decentralized mesh VPN** - providing IP connectivity through a privacy-preserving TNG mesh.

---

## Overview

```
┌──────────┐      WiFi        ┌─────────────┐                    ┌─────────────┐      Internet
│  Client  │◄────────────────►│ Entry Node  │                    │  Exit Node  │◄────────────►
│ (Laptop) │  192.168.x.x     │  (WiFi AP)  │                    │  (NAT/GW)   │   Public IP
└──────────┘                  └──────┬──────┘                    └──────┬──────┘
                                     │                                  │
                                     │         TNG Mesh                 │
                              ┌──────┴──────────────────────────────────┴──────┐
                              │  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
                              │  │ Transit │──│ Transit │──│ Transit │        │
                              │  │  Node   │  │  Node   │  │  Node   │        │
                              │  └─────────┘  └─────────┘  └─────────┘        │
                              │      (LoRa, 802.15.4, QUIC, etc.)             │
                              └───────────────────────────────────────────────┘
```

A client device (laptop, phone) connects to WiFi provided by a TNG **Entry Gateway**. The client receives an IP address and can browse the internet normally. Under the hood:

1. The Entry Gateway captures the client's IP traffic
2. Traffic is encapsulated and sent through the TNG mesh (encrypted via E3X)
3. An **Exit Gateway** with internet connectivity decapsulates and forwards to the internet
4. Responses flow back through the same path

Transit nodes in the mesh only see encrypted TNG packets - they have no awareness of the IP traffic inside.

---

## Use Cases

### Mesh Internet Access

Share internet connectivity across a TNG mesh. An Exit Gateway with internet access serves clients connected to Entry Gateways anywhere in the mesh.

**Example**: A rural community mesh where one node has satellite internet. Other nodes throughout the village provide WiFi access points, with traffic routing through the mesh to the satellite uplink.

### Mesh VPN

Encrypted site-to-site connectivity over untrusted networks. Two or more sites run TNG gateways, with traffic between them tunneled through the mesh.

**Example**: Connect office networks across the internet using TNG mesh links, with traffic encrypted end-to-end between entry and exit points.

### Embedded Device IP Connectivity

Provide IP addresses to constrained mesh devices (LoRa sensors, 802.15.4 nodes) that cannot run a full IP stack. The gateway proxies IP connectivity on their behalf.

**Example**: A LoRa temperature sensor network where sensors communicate via TNG mesh, but data is accessible via standard IP from a gateway.

### Censorship Circumvention

Route traffic through exit nodes in different jurisdictions to bypass network restrictions.

**Example**: A mesh spanning multiple countries where users can select which exit node to use based on content accessibility or privacy requirements.

### TNG-Native Internet Access

TNG applications can directly use exit gateways for TCP/UDP connectivity without needing an IP stack or entry gateway. The TNG library provides socket-like APIs that tunnel through the mesh.

**Example**: An embedded sensor running the TNG library that needs to POST data to a cloud API. It opens a TCP connection directly through an exit gateway without running DHCP, WiFi, or IP routing.

---

## Direct Exit Gateway Access

TNG-native applications can connect directly to exit gateways for TCP and UDP connectivity, bypassing the need for an entry gateway or local IP stack entirely.

```
TUNNELED ACCESS (Entry Gateway required):
┌─────────┐     WiFi      ┌─────────┐     TNG Mesh     ┌─────────┐     Internet
│ Client  │──IP packets──►│  Entry  │──IP-in-tunnel──►│  Exit   │──IP packets──►
│ (any)   │               │ Gateway │                  │ Gateway │
└─────────┘               └─────────┘                  └─────────┘

DIRECT ACCESS (TNG library only):
┌─────────┐                                            ┌─────────┐     Internet
│TNG Node │──────────────TNG Mesh─────────────────────►│  Exit   │──TCP/UDP────►
│(app+lib)│       (tcp_proxy / udp_proxy channels)     │ Gateway │
└─────────┘                                            └─────────┘
```

### When to Use Direct Access

Direct access is ideal when:

- The device runs the TNG library natively (embedded systems, IoT)
- No local IP stack is available or desired
- Lower overhead than full IP tunneling is needed
- The application knows exactly what connections it needs

Direct access is **not** suitable when:

- Arbitrary applications need internet (use Entry Gateway instead)
- The device doesn't run TNG natively
- Full IP compatibility is required

### TCP Proxy Channel

To open a TCP connection through an exit gateway, create a `tcp_proxy` channel:

```json
{
  "type": "tcp_proxy",
  "c": 1234,
  "host": "api.example.com",
  "port": 443
}
```

| Field | Description |
|-------|-------------|
| `type` | Must be `tcp_proxy` |
| `c` | Channel ID (assigned by opener) |
| `host` | Target hostname or IP address |
| `port` | Target TCP port |

**Channel behavior:**

1. TNG node opens `tcp_proxy` channel to exit gateway
2. Exit gateway establishes TCP connection to target host:port
3. If connection succeeds, exit sends channel acknowledgment
4. If connection fails, exit sends error and closes channel
5. Data flows bidirectionally: channel bytes ↔ TCP stream
6. Either end can close; channel close = TCP FIN

**Connection lifecycle:**

```
TNG Node                     Exit Gateway                   Target
    │                             │                            │
    │──tcp_proxy channel open────►│                            │
    │   {host, port}              │                            │
    │                             │──────TCP SYN──────────────►│
    │                             │◄─────TCP SYN+ACK───────────│
    │                             │──────TCP ACK──────────────►│
    │◄──channel ack───────────────│                            │
    │                             │                            │
    │══════════data═══════════════│═══════════data════════════►│
    │◄═════════data═══════════════│◄══════════data═════════════│
    │                             │                            │
    │──channel close─────────────►│                            │
    │                             │──────TCP FIN──────────────►│
    │                             │◄─────TCP FIN+ACK───────────│
```

**Error responses:**

| Error | Meaning |
|-------|---------|
| `connection_refused` | Target refused connection |
| `host_unreachable` | Cannot reach target host |
| `dns_failed` | Hostname resolution failed |
| `policy_denied` | Exit policy blocks this destination |
| `timeout` | Connection timed out |

### UDP Proxy Channel

For UDP communication, create a `udp_proxy` channel:

```json
{
  "type": "udp_proxy",
  "c": 1234,
  "bind_port": 0
}
```

| Field | Description |
|-------|-------------|
| `type` | Must be `udp_proxy` |
| `c` | Channel ID |
| `bind_port` | Local port to bind (0 = ephemeral) |

**Datagram format:**

Each message on the channel carries one UDP datagram with a header:

```
┌─────────────────────────────────────────────────────────────────┐
│                    UDP Proxy Message                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┬──────────────┬─────────────────────────────┐  │
│  │  Addr Length │   Address    │        UDP Payload          │  │
│  │   (1 byte)   │  (variable)  │        (variable)           │  │
│  └──────────────┴──────────────┴─────────────────────────────┘  │
│                                                                  │
│  Address format: [IP bytes (4 or 16)][Port (2 bytes big-endian)] │
│  Addr Length: 6 for IPv4, 18 for IPv6                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Sending a datagram:**

```
TNG Node → Exit: [addr_len=6][93.184.216.34:53][DNS query bytes]
Exit forwards UDP packet to 93.184.216.34:53
```

**Receiving a datagram:**

```
Exit receives UDP response from 93.184.216.34:53
Exit → TNG Node: [addr_len=6][93.184.216.34:53][DNS response bytes]
```

**Channel behavior:**

- Channel is unreliable (best-effort, no ordering guarantees)
- Exit gateway binds a UDP socket on behalf of the TNG node
- Datagrams can be sent to any destination (like unconnected UDP socket)
- Responses from any source are forwarded back
- Channel timeout closes after inactivity (default: 60 seconds)

### DNS Resolution

TNG nodes needing hostname resolution can use the `dns_query` channel type:

```json
{
  "type": "dns_query",
  "c": 1234,
  "name": "api.example.com",
  "record": "A"
}
```

| Field | Description |
|-------|-------------|
| `name` | Hostname to resolve |
| `record` | Record type: `A`, `AAAA`, `CNAME`, `MX`, `TXT`, etc. |

**Response:**

```json
{
  "type": "dns_response",
  "c": 1234,
  "answers": [
    {"type": "A", "value": "93.184.216.34", "ttl": 300}
  ]
}
```

Alternatively, `tcp_proxy` and `udp_proxy` accept hostnames directly - the exit gateway resolves them, preventing DNS leakage from the TNG node.

### TNG Library API

Implementations should provide high-level APIs for direct exit access:

```python
# TCP connection through exit gateway
async def example_tcp():
    # Select an exit gateway (from route advertisements or config)
    exit_gw = mesh.get_exit_gateway()
    
    # Open TCP connection through exit
    conn = await mesh.tcp_connect(
        exit=exit_gw,
        host="api.example.com",
        port=443
    )
    
    # Use like a normal socket
    await conn.write(b"GET / HTTP/1.1\r\nHost: api.example.com\r\n\r\n")
    response = await conn.read(4096)
    
    # Close connection
    await conn.close()


# UDP through exit gateway  
async def example_udp():
    exit_gw = mesh.get_exit_gateway()
    
    # Create UDP socket through exit
    sock = await mesh.udp_socket(exit=exit_gw)
    
    # Send datagram
    dns_query = build_dns_query("example.com", "A")
    await sock.sendto(dns_query, ("8.8.8.8", 53))
    
    # Receive response
    response, addr = await sock.recvfrom(512)
    
    await sock.close()


# DNS resolution through exit
async def example_dns():
    exit_gw = mesh.get_exit_gateway()
    
    # Resolve hostname
    addresses = await mesh.dns_resolve(
        exit=exit_gw,
        name="api.example.com",
        record="A"
    )
    # Returns: ["93.184.216.34"]
```

### REST Client (Optional)

For common API interactions, TNG implementations may provide an optional REST client built on top of the `tcp_proxy` channel. This offers a high-level HTTP/HTTPS interface designed for resource-constrained devices that need to call REST APIs.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         TNG Device Application                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  response = await rest.post("https://api.io/data", json={"temp": 23})   │
│                                                                          │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      TNG REST Client (optional)                          │
│  - HTTP/1.1 request formatting    - JSON encoding/decoding              │
│  - TLS handshake (for HTTPS)      - Response parsing                    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         tcp_proxy channel                                │
│                    (to exit gateway via TNG mesh)                        │
└─────────────────────────────────────────────────────────────────────────┘
```

#### REST API Examples

```python
# Initialize REST client with exit gateway
rest = mesh.rest_client(exit=mesh.get_exit_gateway())

# Simple GET request
response = await rest.get("https://api.example.com/status")
print(response.status)      # 200
print(response.json())      # {"status": "ok", "version": "1.2.3"}

# POST with JSON body
response = await rest.post(
    "https://api.example.com/sensors/data",
    json={
        "device_id": "sensor-01",
        "temperature": 23.5,
        "humidity": 65.2
    },
    headers={"Authorization": "Bearer token123"}
)

if response.status == 201:
    print("Data uploaded successfully")

# PUT to update a resource
response = await rest.put(
    "https://api.example.com/devices/sensor-01/config",
    json={"interval": 60, "threshold": 30.0}
)

# DELETE a resource
response = await rest.delete("https://api.example.com/alerts/old")

# Streaming response for large payloads (e.g., firmware download)
async for chunk in rest.get_stream("https://api.example.com/firmware/v2.bin"):
    await flash.write(chunk)
```

#### Request and Response Objects

```python
class RestResponse:
    status: int           # HTTP status code (200, 404, 500, etc.)
    headers: dict         # Response headers (lowercase keys)
    body: bytes           # Raw response body
    
    def json(self) -> dict:
        """Parse body as JSON."""
        pass
    
    def text(self) -> str:
        """Decode body as UTF-8 string."""
        pass
    
    @property
    def ok(self) -> bool:
        """True if status is 2xx."""
        return 200 <= self.status < 300
```

#### Supported Methods

| Method | Function | Description |
|--------|----------|-------------|
| GET | `rest.get(url, headers=None)` | Retrieve a resource |
| POST | `rest.post(url, json=None, body=None, headers=None)` | Create a resource |
| PUT | `rest.put(url, json=None, body=None, headers=None)` | Replace a resource |
| PATCH | `rest.patch(url, json=None, body=None, headers=None)` | Partial update |
| DELETE | `rest.delete(url, headers=None)` | Remove a resource |

#### TLS/HTTPS Modes

The REST client supports two TLS modes:

**Exit-Terminated TLS (Default)**:
- TNG device sends `https://` URL to exit gateway
- Exit gateway handles TLS handshake with destination
- Simpler for constrained devices (no TLS stack needed)
- Traffic between device and exit is E3X encrypted

```python
# Exit-terminated TLS (default)
rest = mesh.rest_client(exit=exit_gw, tls_mode="exit")
response = await rest.get("https://api.example.com/data")
```

**End-to-End TLS**:
- TNG device performs TLS handshake through tcp_proxy tunnel
- True end-to-end encryption to destination
- Requires TLS implementation on device
- Exit gateway cannot inspect traffic content

```python
# End-to-end TLS (device handles TLS)
rest = mesh.rest_client(exit=exit_gw, tls_mode="e2e")
response = await rest.get("https://api.example.com/data")
```

#### Configuration Options

```python
rest = mesh.rest_client(
    exit=exit_gw,
    
    # TLS settings
    tls_mode="exit",              # "exit" or "e2e"
    verify_certs=True,            # Validate server certificates
    
    # Timeouts
    connect_timeout=10.0,         # Connection timeout (seconds)
    read_timeout=30.0,            # Read timeout (seconds)
    
    # Connection management
    keepalive=True,               # Reuse connections to same host
    max_connections=4,            # Max concurrent connections
    
    # Retry behavior
    retry_count=0,                # Number of retries on failure
    retry_backoff=1.0,            # Backoff multiplier between retries
)
```

#### Lightweight Design

The REST client is intentionally minimal for embedded use:

| Feature | Included | Notes |
|---------|----------|-------|
| HTTP/1.1 | Yes | Minimal implementation |
| JSON encoding/decoding | Yes | Built-in |
| Chunked transfer encoding | Yes | For streaming |
| Connection reuse | Yes | Keep-alive support |
| Cookies | No | Stateless by design |
| Redirects | No | Explicit handling required |
| Multipart forms | No | Use raw body instead |
| HTTP/2 | No | Too complex for constrained devices |

For redirects, check the `Location` header manually:

```python
response = await rest.get("https://example.com/old-path")
if response.status in (301, 302, 307, 308):
    new_url = response.headers.get("location")
    response = await rest.get(new_url)
```

#### Error Handling

```python
# Note: Import paths shown are pseudocode examples.
# Actual implementation structure may vary by language/platform.
from tng.rest import RestError, ConnectionError, TimeoutError

try:
    response = await rest.get("https://api.example.com/data")
    if not response.ok:
        print(f"HTTP error: {response.status}")
except TimeoutError:
    print("Request timed out")
except ConnectionError as e:
    print(f"Connection failed: {e}")
except RestError as e:
    print(f"REST client error: {e}")
```

#### Typical Use Case: IoT Sensor

```python
async def report_sensor_data():
    """Periodic sensor data upload to cloud API."""
    exit_gw = mesh.get_exit_gateway()
    rest = mesh.rest_client(exit=exit_gw)
    
    while True:
        # Read sensor
        temp = await sensor.read_temperature()
        humidity = await sensor.read_humidity()
        
        # Upload to API
        try:
            response = await rest.post(
                "https://api.iot-cloud.com/v1/telemetry",
                json={
                    "device": mesh.local_hashname()[:16],
                    "readings": {
                        "temperature": temp,
                        "humidity": humidity
                    },
                    "timestamp": time.time()
                },
                headers={"X-API-Key": API_KEY}
            )
            
            if response.ok:
                print(f"Uploaded: {response.json()}")
            else:
                print(f"Upload failed: {response.status}")
                
        except Exception as e:
            print(f"Error: {e}")
        
        await asyncio.sleep(60)  # Report every minute
```

### Exit Gateway Proxy Implementation

Exit gateways supporting direct access must handle the proxy channel types:

```python
class ExitProxyHandler:
    async def handle_tcp_proxy(self, channel, host: str, port: int):
        """Handle tcp_proxy channel request."""
        # Check policy
        if not self.policy.allows_destination(host, port):
            await channel.send_error("policy_denied")
            return
        
        # Resolve hostname if needed
        try:
            addr = await self.resolve(host)
        except DNSError:
            await channel.send_error("dns_failed")
            return
        
        # Establish TCP connection
        try:
            sock = await asyncio.open_connection(addr, port)
        except ConnectionRefusedError:
            await channel.send_error("connection_refused")
            return
        except OSError:
            await channel.send_error("host_unreachable")
            return
        
        # Acknowledge and relay
        await channel.send_ack()
        await self.relay_bidirectional(channel, sock)
    
    async def handle_udp_proxy(self, channel, bind_port: int):
        """Handle udp_proxy channel request."""
        # Bind UDP socket
        sock = socket.socket(AF_INET, SOCK_DGRAM)
        sock.bind(("0.0.0.0", bind_port))
        
        # Relay datagrams
        await channel.send_ack({"bound_port": sock.getsockname()[1]})
        await self.relay_udp(channel, sock)
```

### Security Considerations for Direct Access

**Trust model:**

```
TNG Node ◄──E3X encrypted──► Exit Gateway ◄──cleartext──► Destination
                                   │
                                   └── Exit sees all proxied traffic
                                       (like any SOCKS/HTTP proxy)
```

**Security properties:**

- Transit nodes cannot inspect proxy traffic (E3X encrypted)
- Exit gateway sees destination hosts, ports, and all traffic content
- TLS/HTTPS recommended for sensitive data (encrypted end-to-end through exit)
- Exit gateway is trusted to correctly proxy (no content modification)

**Policy enforcement:**

Exit gateways can restrict direct access:

```python
class DirectAccessPolicy:
    # Allowed destination patterns
    allowed_hosts: list[str]       # ["*.example.com", "api.service.io"]
    allowed_ports: list[int]       # [80, 443, 853]
    
    # Blocked destinations
    blocked_hosts: list[str]       # ["*.internal.corp"]
    blocked_cidrs: list[str]       # ["10.0.0.0/8", "192.168.0.0/16"]
    
    # Rate limiting
    max_connections_per_node: int  # 100
    max_bandwidth_per_node: int    # 10 Mbps
    
    # Protocol restrictions
    allow_tcp: bool                # True
    allow_udp: bool                # True
    allow_dns: bool                # True
```

**Preventing abuse:**

- Exit gateways should implement rate limiting
- Connection limits prevent resource exhaustion
- Bandwidth caps prevent single node from monopolizing exit
- Logging optional but may be required for abuse prevention

---

## Gateway Roles

A TNG node can perform one or more of these gateway roles:

| Role | Function | Required Components |
|------|----------|---------------------|
| **Entry Gateway** | Provides network access to non-TNG clients | WiFi AP, DHCP/DHCPv6, Router Advertisement, Tunnel Ingress |
| **Exit Gateway** | Provides internet/IP connectivity to mesh | NAT/NAT66, Tunnel Egress, IP Routing |
| **Transit Node** | Forwards encapsulated traffic through mesh | Standard TNG routing only |

### Entry Gateway

The Entry Gateway is the client-facing component:

- Operates a WiFi access point (or wired Ethernet segment)
- Runs DHCP server for IPv4 address assignment
- Sends Router Advertisements for IPv6 SLAAC
- Captures client IP traffic and encapsulates it for mesh transport
- Maintains client session state and address mappings

### Exit Gateway

The Exit Gateway provides connectivity to external networks:

- Has direct IP connectivity (internet, corporate network, etc.)
- Performs NAT for outbound IPv4 traffic
- Performs NAT66 or direct routing for IPv6
- Decapsulates tunnel traffic and forwards to destination
- Maintains NAT state for return traffic
- Handles `tcp_proxy` and `udp_proxy` channels for direct access (see [Direct Exit Gateway Access](#direct-exit-gateway-access))

### Transit Node

Transit nodes require no special configuration for gateway traffic:

- Forward TNG packets using standard mesh routing
- Cannot inspect tunnel contents (E3X encrypted)
- May implement QoS policies for tunnel traffic channels

### Combined Entry/Exit

A single node can act as both Entry and Exit Gateway simultaneously. This is useful for:

- Simple deployments with one gateway
- Local network access (Entry) plus internet access (Exit)
- Mesh nodes that serve local clients and also provide exit capacity

---

## IP Router Component

The IP Router runs on Entry and Exit Gateways, handling the translation between IP traffic and TNG mesh transport.

```
┌─────────────────────────────────────────────────────────────────┐
│                      IP Router Component                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Routing    │  │   Address    │  │    Tunnel    │           │
│  │    Table     │  │    Pool      │  │   Manager    │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                 │                 │                    │
│         └─────────────────┼─────────────────┘                    │
│                           │                                      │
│                    ┌──────▼───────┐                             │
│                    │   Packet     │                             │
│                    │  Processor   │                             │
│                    └──────────────┘                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Routing Table

Maps destination IP prefixes to exit gateway hashnames:

```python
class RouteEntry:
    prefix: str           # e.g., "0.0.0.0/0" or "10.0.0.0/8"
    exit_hashname: str    # Hashname of exit gateway
    metric: int           # Route preference (lower = better)
    expires: float        # Timestamp when route expires
```

The routing table is populated by:

1. **Static configuration**: Manually configured routes
2. **Route advertisements**: Received from exit gateways via mesh
3. **Policy rules**: Override routes based on client or destination

### Address Pool

**Entry Gateway** - manages client address assignment:

```python
class ClientSession:
    client_mac: bytes      # Client's MAC address
    ipv4_addr: str         # Assigned IPv4 (e.g., "10.42.0.5")
    ipv6_addr: str         # Assigned IPv6 ULA
    session_id: int        # 24-bit tunnel session identifier
    created: float         # Session creation timestamp
    last_active: float     # Last packet timestamp
```

**Exit Gateway** - manages NAT state:

```python
class NATMapping:
    session_id: int        # From tunnel header
    internal_addr: str     # Client's address (from entry)
    external_port: int     # Allocated external port
    protocol: int          # TCP/UDP/ICMP
    created: float
    expires: float
```

### Tunnel Manager

Handles encapsulation and decapsulation of IP packets:

- Creates TNG channels to exit gateways
- Encapsulates IP packets with tunnel headers
- Decapsulates received tunnel packets
- Manages channel lifecycle and reconnection

### Packet Processor

The central forwarding engine:

**Entry Gateway flow**:
1. Receive IP packet from local interface (WiFi/Ethernet)
2. Lookup or create client session
3. Lookup route for destination
4. Encapsulate packet with tunnel header
5. Send via TNG channel to exit gateway

**Exit Gateway flow**:
1. Receive tunnel packet from TNG channel
2. Decapsulate and extract IP packet
3. Apply NAT (update source address/port)
4. Forward to destination via local IP stack
5. For responses: reverse NAT and tunnel back

---

## Tunnel Protocol

IP packets are encapsulated in TNG channels for mesh transport.

### Channel Type

Gateway tunnels use a dedicated channel type:

```json
{
  "type": "iptunnel",
  "c": 1234
}
```

The `iptunnel` channel type indicates IP tunnel traffic. Implementations may apply specific policies (QoS, rate limiting) to tunnel channels.

**Transport Requirements**: `iptunnel` channels MUST only traverse transports with `IP_TUNNEL` capability. Paths through LoRa or 433/915MHz UART are invalid for IP tunneling. See [Route Metrics - Gateway Route Filtering](route-metrics.md#gateway-route-filtering) for details.

### Tunnel Packet Format

```
┌─────────────────────────────────────────────────────────────────┐
│                    Tunnel Packet Format                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────┬────────────────┬─────────────────────────┐  │
│  │  Tunnel Header │     Flags      │       IP Packet         │  │
│  │   (4 bytes)    │   (2 bytes)    │      (variable)         │  │
│  └────────────────┴────────────────┴─────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Tunnel Header (4 bytes)

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  Type |                 Session ID                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Field | Bits | Description |
|-------|------|-------------|
| Version | 4 | Protocol version (currently 1) |
| Type | 4 | Packet type (see below) |
| Session ID | 24 | Identifies client session |

**Packet Types**:

| Type | Value | Description |
|------|-------|-------------|
| IPv4 | 0x4 | Payload is IPv4 packet |
| IPv6 | 0x6 | Payload is IPv6 packet |
| Control | 0x0 | Tunnel control message |

### Flags (2 bytes)

```
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|F|M|C|       Reserved          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

| Flag | Bit | Description |
|------|-----|-------------|
| F (Fragment) | 0 | Packet is a fragment |
| M (More) | 1 | More fragments follow |
| C (Compressed) | 2 | IP header compression applied |
| Reserved | 3-15 | Must be zero |

### Fragmentation

For low-MTU physical transports, IP packets may need fragmentation at the tunnel layer:

```
Original IP packet (1500 bytes) over LoRa (255 byte MTU):

Fragment 1: [Tunnel Hdr F=1 M=1][Frag Hdr][IP bytes 0-200]
Fragment 2: [Tunnel Hdr F=1 M=1][Frag Hdr][IP bytes 201-400]
...
Fragment N: [Tunnel Hdr F=1 M=0][Frag Hdr][IP bytes final]
```

Fragment header (when F=1):

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Fragment ID           |        Fragment Offset        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Control Messages

Type 0x0 packets carry tunnel control messages in the payload:

| Control Type | Description |
|--------------|-------------|
| `session_start` | Entry requests session setup |
| `session_ack` | Exit acknowledges session |
| `session_end` | Either end terminates session |
| `keepalive` | Maintain session state |

Control message format (JSON in payload):

```json
{
  "ctrl": "session_start",
  "session_id": 12345,
  "client_ipv4": "10.42.0.5",
  "client_ipv6": "fd12:3456:789a::5"
}
```

---

## Address Assignment

### Entry Gateway: Client Addressing

The Entry Gateway assigns IP addresses to connecting clients.

#### IPv4 (DHCP)

Each Entry Gateway operates a DHCP server with a unique subnet:

```
Gateway subnet: 10.<gateway_id>.0.0/24
  where gateway_id = first byte of hashname (0-255)

Example for hashname starting with "kw3a...":
  k = 10 in base32 → gateway_id = 10
  Subnet: 10.10.0.0/24
  Gateway IP: 10.10.0.1
  Client range: 10.10.0.2 - 10.10.0.254
```

DHCP options provided:

| Option | Value |
|--------|-------|
| Router | Gateway IP (e.g., 10.10.0.1) |
| DNS | Gateway IP (proxy) or configured DNS |
| Lease time | 3600 seconds (configurable) |

#### IPv6 (SLAAC)

Entry Gateways advertise an IPv6 ULA prefix derived from the gateway's hashname:

```
ULA prefix format:
fd + [5 bytes from hashname] + [segment] :: /64

Example for hashname "kw3akwcypo...":
  Base32 decode first 8 chars → bytes
  Take first 5 bytes: 0x56, 0x76, 0x16, 0xb0, 0x9c
  Segment: 0x0001

  Prefix: fd56:7616:b09c:0001::/64
```

Router Advertisement contents:

| Field | Value |
|-------|-------|
| Prefix | ULA /64 from hashname |
| Prefix flags | On-link, Autonomous |
| Router lifetime | 1800 seconds |
| MTU | Effective tunnel MTU |

Clients generate addresses via SLAAC using their interface identifier.

### Exit Gateway: NAT

The Exit Gateway performs Network Address Translation for outbound traffic.

#### IPv4 NAT (NAPT)

Standard port-based NAT:

```
Outbound:
  Client: 10.10.0.5:54321 → Destination: 93.184.216.34:443
  After NAT: <Exit Public IP>:30000 → Destination: 93.184.216.34:443

Inbound (response):
  Source: 93.184.216.34:443 → <Exit Public IP>:30000
  After NAT: Source: 93.184.216.34:443 → Client: 10.10.0.5:54321
```

NAT table entry:

```python
{
  "session_id": 12345,
  "protocol": "tcp",
  "internal": "10.10.0.5:54321",
  "external_port": 30000,
  "destination": "93.184.216.34:443",
  "expires": 1704067200
}
```

#### IPv6 NAT66

For IPv6, the Exit Gateway can either:

1. **NAT66**: Translate ULA source to exit's global IPv6
2. **Direct routing**: If the exit has delegated prefix, route directly

NAT66 is simpler but loses end-to-end addressing. Direct routing requires prefix delegation from the exit's upstream.

---

## Exit Gateway Discovery — Advanced

Exit gateways announce their availability through the mesh using a link-based propagation protocol. This is an **advanced feature** - basic TNG operation does not require gateway discovery.

### Link-Based Propagation Model

**Critical**: TNG has no broadcast capability. Gateway announcements propagate only through existing E3X links:

```
Exit Gateway (G) announces to directly linked peers:

    G ──link──► Peer A ──link──► Peer B ──link──► Peer C
    │                    │                    │
    │  route_info{G}     │  route_info{G}     │  route_info{G}
    │  hops=0            │  hops=1            │  hops=2
    │                    │  via=A             │  via=B
```

- Gateways send `route_info` to their **directly linked peers** only
- Peers (routers) optionally re-announce to their own peers
- Each hop increments the hop count and tracks the via-peer
- Nodes learn gateway routes through their existing link topology

### Route Information Protocol

The `route_info` channel type carries gateway route information:

```json
HEAD: {
  "c": <channel_id>,
  "type": "route_info",
  "seq": 0
}
BODY: {
  "gateway": "<gateway hashname>",
  "seqno": <32-bit sequence number>,
  "hops": <current hop count>,
  "via": "<peer hashname>",  // Next-hop peer (for split horizon)
  "capabilities": ["ipv4", "ipv6", "nat", "dns"],
  "prefixes": [
    {
      "prefix": "0.0.0.0/0",
      "type": "ipv4",
      "metric": {
        "hops": 0,
        "path_cost": 10,
        "min_bandwidth_kbps": 100000,
        "max_latency_ms": 20,
        "transport_flags": 0x003F  // All capabilities
      }
    }
  ],
  "ttl": 300,
  "timestamp": <unix-seconds>
}
```

| Field | Type | Description |
|-------|------|-------------|
| `gateway` | string | Exit gateway's hashname |
| `seqno` | u32 | Route sequence number (incremented on changes) |
| `hops` | u8 | Current hop count from gateway |
| `via` | string | Next-hop peer hashname (for split horizon) |
| `capabilities` | array | Supported features (ipv4, ipv6, nat, dns) |
| `prefixes` | array | Reachable IP prefixes with metrics |
| `ttl` | u16 | Seconds until route expires |
| `timestamp` | u64 | Unix timestamp of announcement |

### Propagation Rules

1. **Gateway originates** `route_info` with `hops=0` to all directly linked peers
2. **Receiving peer** may re-propagate to its own peers (if configured as router)
3. **Re-propagation** increments `hops` and updates `via` path
4. **Split horizon**: Never send route back to the peer it came from (check `via` field)
5. **TTL limits** propagation depth (default max hops: 16)
6. **Sequence numbers** prevent stale routes (see [Routing - Loop Prevention](routing.md#loop-prevention))

### Gateway Solicitation (route_query)

Nodes can actively query for gateway routes:

```json
HEAD: {
  "c": <channel_id>,
  "type": "route_query",
  "seq": 0
}
BODY: {
  "need": "ipv4",           // Optional: "ipv4" or "ipv6"
  "prefix": "0.0.0.0/0"     // Optional: specific prefix
}
```

**Response:**

```json
HEAD: {
  "c": <channel_id>,
  "type": "route_query",
  "seq": 0,
  "ack": 0
}
BODY: {
  "routes": [
    {
      "gateway": "G1",
      "hops": 2,
      "via": "R",
      "metric": {...},
      "prefixes": [...]
    },
    {
      "gateway": "G2",
      "hops": 3,
      "via": "R",
      "metric": {...},
      "prefixes": [...]
    }
  ]
}
```

**Discovery Flow:**

```
Node N needs internet access, queries its linked router R:

    N ──────────────────────────────────────────► R (router)
       route_query: {"need": "ipv4", "prefix": "0.0.0.0/0"}
    
    N ◄────────────────────────────────────────── R
       route_response: {
         "routes": [
           {"gateway": "G1", "hops": 2, "via": "R", "metric": {...}},
           {"gateway": "G2", "hops": 3, "via": "R", "metric": {...}}
         ]
       }
```

### Transport Capability Filtering

Routes to exit gateways MUST only traverse transports with `GATEWAY_ROUTE` capability. See [Route Metrics](route-metrics.md#gateway-route-filtering) for details.

**Invalid paths** (rejected):
- LoRa-only paths
- 433/915MHz-only paths
- Paths without `GATEWAY_ROUTE` capability

**Valid paths** (accepted):
- QUIC, WebTransport, 802.11 (excellent)
- Bluetooth LE (marginal, needs fragmentation)
- 802.15.4 (limited, but acceptable)

---

## Route Advertisement (Legacy)

**Note**: The following `route_advert` format is the legacy approach. New implementations should use the `route_info` protocol described above for link-based propagation.

Exit Gateways advertise their availability and capabilities to the mesh.

### Advertisement Message

Exit gateways periodically send route advertisements via mesh broadcast or to known entry gateways:

```json
{
  "type": "route_advert",
  "gateway": "kw3akwcypoedvfdquuppofpujbu7rplhj3vjvmvbkvf7z3do7kkq",
  "prefixes": [
    {"prefix": "0.0.0.0/0", "type": "ipv4"},
    {"prefix": "::/0", "type": "ipv6"}
  ],
  "metrics": {
    "bandwidth_mbps": 100,
    "latency_ms": 50,
    "cost": 1
  },
  "capabilities": ["ipv4", "ipv6", "nat"],
  "ttl": 300,
  "timestamp": 1704067200
}
```

| Field | Description |
|-------|-------------|
| `gateway` | Exit gateway's hashname |
| `prefixes` | Reachable IP prefixes (default route = internet) |
| `metrics.bandwidth_mbps` | Approximate available bandwidth |
| `metrics.latency_ms` | Approximate latency to internet |
| `metrics.cost` | Administrative preference (lower = preferred) |
| `capabilities` | Supported features |
| `ttl` | Seconds until advertisement expires |

### Route Selection

Entry gateways select exits based on:

1. **Reachability**: Exit must be reachable through mesh
2. **Transport capability**: Path must have `GATEWAY_ROUTE` capability (see [Route Metrics - Transport Capability Flags](route-metrics.md#transport-capability-flags))
3. **Prefix match**: Exit must advertise route for destination
4. **Metrics**: Prefer lower path cost, then higher bandwidth, then lower latency
5. **Policy**: Local policy may override (e.g., prefer specific exits)

```python
# Import TransportFlags from route-metrics (see [Route Metrics](route-metrics.md#transport-capability-flags))
from route_metrics import TransportFlags

def select_exit(destination_ip: str, available_exits: list) -> str:
    """Select best exit gateway for destination."""
    candidates = []
    
    for exit in available_exits:
        # Check transport capability (use consistent metric access)
        metric = exit.metric if hasattr(exit, 'metric') else None
        if not metric or not (metric.transport_flags & TransportFlags.GATEWAY_ROUTE):
            continue  # Skip exits without gateway route capability
        
        # Check if exit advertises a matching route
        for route in exit.prefixes:
            if ip_matches_prefix(destination_ip, route.prefix):
                candidates.append((exit, route))
                break
    
    if not candidates:
        return None
    
    # Sort by: path_cost (asc), min_bandwidth (desc), max_latency (asc), hops (asc)
    # Use consistent metric access pattern
    candidates.sort(key=lambda x: (
        x[0].metric.path_cost if hasattr(x[0].metric, 'path_cost') else 0,
        -x[0].metric.min_bandwidth_kbps if hasattr(x[0].metric, 'min_bandwidth_kbps') else 0,
        x[0].metric.max_latency_ms if hasattr(x[0].metric, 'max_latency_ms') else 0,
        x[0].metric.hops if hasattr(x[0].metric, 'hops') else 0
    ))
    
    return candidates[0][0].gateway
```

For advanced route selection with transport-aware filtering, see [Route Metrics - Advanced Route Selection](route-metrics.md#advanced-route-selection).

### Multi-Exit Load Balancing

With multiple exit gateways, entry gateways may distribute traffic:

- **Per-session**: All packets for a session use same exit (preserves NAT state)
- **Per-destination**: Different destinations may use different exits
- **Failover**: Switch exits if primary becomes unreachable

---

## Security and Privacy Model

### Trust Boundaries

```
┌─────────┐        ┌─────────┐                    ┌─────────┐        ┌─────────┐
│ Client  │◄─WiFi─►│  Entry  │◄───TNG/E3X────────►│  Exit   │◄─Net──►│  Dest   │
└─────────┘        └─────────┘                    └─────────┘        └─────────┘
     │                  │                              │                  │
     │   Client trusts  │    E3X encrypted tunnel     │   Exit sees      │
     │   Entry with     │    Transit nodes see        │   decrypted      │
     │   cleartext      │    only encrypted blobs     │   traffic        │
     │                  │                              │                  │
```

### What Each Component Sees

**Tunneled Access (via Entry Gateway):**

| Component | Can See | Cannot See |
|-----------|---------|------------|
| **Client** | Own traffic (cleartext) | Other clients' traffic |
| **Entry Gateway** | All client traffic (cleartext), client addresses | Traffic content after TLS |
| **Transit Node** | Hashnames, packet sizes, timing | IP addresses, traffic content |
| **Exit Gateway** | Decrypted tunnel traffic, destinations | Client's real identity (sees session ID) |
| **Destination** | Exit's IP address as source | Client's real IP or location |

**Direct Access (TNG-native via proxy channels):**

| Component | Can See | Cannot See |
|-----------|---------|------------|
| **TNG Node** | Own traffic, exit gateway hashname | Other nodes' traffic |
| **Transit Node** | Hashnames, packet sizes, timing | Proxy traffic content, destinations |
| **Exit Gateway** | All proxied traffic, target hosts/ports | TNG node's physical location |
| **Destination** | Exit's IP address as source | TNG node's identity or hashname |

### Security Properties

- **Transit encryption**: E3X encrypts tunnel between Entry and Exit; transit nodes cannot inspect
- **Client isolation**: Each client has unique session ID; clients cannot see each other's traffic
- **NAT anonymity**: Destination sees Exit's IP, not client's (shared among all exit users)

### Privacy Properties

- **No broadcast discovery**: Clients must know Entry Gateway out-of-band (no mDNS/beacon)
- **Minimal metadata at transit**: Transit nodes see only hashnames and encrypted packets
- **Multiple exits**: Traffic can be distributed across exits to prevent single point of observation
- **No logging requirement**: Gateways can operate without persistent traffic logs

### Gateway Authorization

Gateways can implement access control:

**Entry Gateway**:
```python
class EntryPolicy:
    allowed_exits: list[str]      # Hashnames of permitted exit gateways
    allowed_clients: list[str]    # MAC addresses or "any"
    bandwidth_limit: int          # Per-client bandwidth cap
```

**Exit Gateway**:
```python
class ExitPolicy:
    allowed_entries: list[str]    # Hashnames of permitted entry gateways
    allowed_destinations: list    # IP prefixes clients may access
    blocked_destinations: list    # IP prefixes to block
    
    # Direct access (proxy) settings
    allow_direct_access: bool     # Whether to accept tcp_proxy/udp_proxy channels
    direct_allowed_nodes: list    # Hashnames permitted for direct access ("*" = any)
    direct_allowed_ports: list    # Ports allowed for direct proxy (e.g., [80, 443])
```

**Transit Node**:
```python
class TransitPolicy:
    allow_tunnel_channels: bool   # Whether to route iptunel channels
    tunnel_priority: int          # QoS priority for tunnel traffic
```

### Threat Mitigations

| Threat | Mitigation |
|--------|------------|
| Eavesdropping at transit | E3X encryption; transit sees only ciphertext |
| Rogue entry gateway | Client must trust entry (like any WiFi AP) |
| Rogue exit gateway | Entry selects exits; exit authorization policy |
| Traffic correlation | Multiple exits; timing obfuscation (optional) |
| Session hijacking | Session IDs bound to E3X session keys |

---

## Implementation Notes

### MTU Considerations

Tunnel overhead reduces effective MTU:

```
Underlying transport MTU (e.g., QUIC): 1200 bytes
- TNG packet overhead:                   ~50 bytes
- Tunnel header:                           6 bytes
- Available for IP packet:             ~1144 bytes
```

Entry gateways should:
1. Advertise reduced MTU in DHCP/RA
2. Implement tunnel-layer fragmentation for oversized packets
3. Support Path MTU Discovery responses

### Channel Management

Tunnel channels between Entry and Exit gateways:

```python
class TunnelChannel:
    exit_hashname: str
    channel_id: int
    reliable: bool        # True for TCP-like traffic, False for UDP
    created: float
    last_packet: float
    bytes_sent: int
    bytes_received: int
```

- Use **reliable channels** for TCP traffic (preserves ordering)
- Use **unreliable channels** for UDP traffic (lower latency)
- Multiplex sessions over shared channels to exit gateway

### Session Lifecycle

```
1. Client connects to WiFi
2. Entry assigns IP via DHCP/SLAAC
3. Client sends first packet
4. Entry creates session, selects exit
5. Entry opens tunnel channel to exit (if not exists)
6. Entry sends session_start control message
7. Exit acknowledges, allocates NAT state
8. Traffic flows through tunnel
9. Keepalives maintain session (every 30s)
10. Session expires after inactivity (300s default)
11. Entry sends session_end, exit cleans NAT state
```

### Example Configuration

**Entry Gateway**:
```json
{
  "role": "entry",
  "wifi": {
    "ssid": "TNG-Mesh",
    "interface": "wlan0"
  },
  "dhcp": {
    "enabled": true,
    "range_start": "10.42.0.10",
    "range_end": "10.42.0.250",
    "lease_time": 3600
  },
  "ipv6": {
    "enabled": true,
    "prefix_from_hashname": true
  },
  "policy": {
    "allowed_exits": ["*"],
    "prefer_exits": ["kw3a...abc", "mzxw...xyz"]
  }
}
```

**Exit Gateway**:
```json
{
  "role": "exit",
  "upstream": {
    "interface": "eth0",
    "ipv4": true,
    "ipv6": true
  },
  "nat": {
    "ipv4_enabled": true,
    "ipv6_mode": "nat66"
  },
  "advertise": {
    "prefixes": ["0.0.0.0/0", "::/0"],
    "bandwidth_mbps": 100,
    "cost": 1
  },
  "policy": {
    "allowed_entries": ["*"],
    "blocked_destinations": ["10.0.0.0/8", "192.168.0.0/16"]
  }
}
```

---

## Packet Flow Example

**Client requests https://example.com:**

```
1. Client (10.42.0.5) sends TCP SYN to 93.184.216.34:443
   
2. Entry Gateway receives packet on WiFi interface
   - Lookup/create session for client MAC → session_id=12345
   - Lookup route for 93.184.216.34 → Exit "kw3a..."
   - Encapsulate: [Tunnel Hdr: v=1, type=4, sid=12345][IP packet]
   - Send via TNG channel to Exit

3. Transit nodes forward encrypted TNG packet
   - See: Entry hashname → Exit hashname, encrypted blob
   - Don't see: IP addresses, ports, or content

4. Exit Gateway receives tunnel packet
   - Decapsulate → extract IP packet
   - Apply SNAT: 10.42.0.5:54321 → <Exit IP>:30000
   - Forward to 93.184.216.34:443

5. example.com responds to Exit IP

6. Exit Gateway receives response
   - Lookup NAT table: port 30000 → session 12345
   - Reverse NAT: dst → 10.42.0.5:54321
   - Encapsulate and send back through tunnel

7. Entry Gateway decapsulates, delivers to client
```

---

*See also: [Routing](routing.md) | [Channels](channels.md) | [E3X](e3x.md) | [Mesh](mesh.md)*
