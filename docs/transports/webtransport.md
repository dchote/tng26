# WebTransport

**WebTransport** enables TNG in web browsers, providing bidirectional communication over HTTP/3. It offers similar capabilities to QUIC while working within browser security constraints.

## Overview

| Property | Value |
|----------|-------|
| **Transport** | WebTransport (W3C) |
| **Protocol** | HTTP/3 |
| **MTU** | ~1200 bytes |
| **Reliability** | Configurable (streams or datagrams) |
| **Browser Support** | Chrome 97+, Edge 97+, Firefox 114+ |

## Why WebTransport?

| Feature | Benefit |
|---------|---------|
| **Browser-native** | No plugins or extensions required |
| **Low latency** | Unreliable datagrams available |
| **Multiplexed** | Multiple streams, no head-of-line blocking |
| **Secure** | TLS 1.3 required |

## Path Format

```json
{
  "type": "webtransport",
  "url": "https://example.com:443/tng"
}
```

### URL Requirements

- Must use `https://` scheme
- Valid TLS certificate required (not self-signed in browsers)
- Path can be customized per deployment

## Server Setup

### Endpoint URL

The server exposes a WebTransport endpoint:

```
https://example.com/tng
```

### HTTP/3 Server

```python
from aioquic.asyncio import serve
from aioquic.h3.connection import H3Connection

async def start_webtransport_server(mesh, host="0.0.0.0", port=443):
    config = QuicConfiguration(
        is_client=False,
        alpn_protocols=["h3"],
    )
    config.load_cert_chain("fullchain.pem", "privkey.pem")
    
    async def handle_webtransport(session):
        pipe = WebTransportPipe(session)
        mesh.on_pipe(pipe)
    
    server = await serve(
        host, port,
        configuration=config,
        create_protocol=WebTransportProtocol,
    )
    
    return server
```

### Request Handling

```python
class WebTransportProtocol:
    def handle_request(self, request):
        if request.path == "/tng":
            # Accept WebTransport session
            return WebTransportSession(self)
        else:
            return Http404()
```

## Client Connection

### Browser JavaScript

```javascript
class TNGWebTransport {
  constructor(url) {
    this.url = url;
    this.transport = null;
  }

  async connect() {
    this.transport = new WebTransport(this.url);
    await this.transport.ready;
    
    // Use datagrams for TNG packets
    this.writer = this.transport.datagrams.writable.getWriter();
    this.reader = this.transport.datagrams.readable.getReader();
    
    return this;
  }

  async send(packet) {
    await this.writer.write(packet);
  }

  async receive() {
    const { value, done } = await this.reader.read();
    if (done) throw new Error("Connection closed");
    return value;
  }

  close() {
    this.transport.close();
  }
}

// Usage
const pipe = await new TNGWebTransport("https://example.com/tng").connect();
await pipe.send(handshakePacket);
const response = await pipe.receive();
```

### Native Client

For non-browser clients:

```python
async def connect_webtransport(path: dict) -> Pipe:
    url = path["url"]
    
    async with WebTransportClient(url) as session:
        return WebTransportPipe(session)
```

## Packet Mapping

### Datagrams (Preferred)

Use WebTransport datagrams for TNG packets:

```javascript
// Send
const packet = new Uint8Array([...]); // TNG packet
await writer.write(packet);

// Receive
const { value } = await reader.read();
const packet = value; // TNG packet
```

### Streams (Fallback)

If datagrams are unavailable or unreliable:

```javascript
// Send via unidirectional stream
const stream = await transport.createUnidirectionalStream();
const writer = stream.getWriter();

// Length-prefix the packet
const length = new ArrayBuffer(2);
new DataView(length).setUint16(0, packet.length, false);
await writer.write(new Uint8Array(length));
await writer.write(packet);
await writer.close();
```

## Certificate Requirements

### Production

Web browsers require valid certificates:
- Issued by trusted CA (Let's Encrypt, etc.)
- Matching domain name
- Not expired

### Development

For local development:

```javascript
// Chrome flag for testing
// --ignore-certificate-errors-spki-list=<hash>

// Or use localhost with WebTransport dev server
const transport = new WebTransport("https://localhost:4433/tng");
```

## Connection Lifecycle

```javascript
class TNGWebTransportClient {
  async connect(url) {
    this.transport = new WebTransport(url);
    
    // Wait for connection
    await this.transport.ready;
    console.log("WebTransport connected");
    
    // Handle closure
    this.transport.closed.then(() => {
      console.log("WebTransport closed gracefully");
    }).catch((error) => {
      console.error("WebTransport error:", error);
    });
    
    return this;
  }
  
  async reconnect() {
    // WebTransport doesn't support reconnection
    // Create new connection
    await this.connect(this.url);
  }
}
```

## Pipe Implementation

```javascript
class WebTransportPipe {
  constructor(transport) {
    this.transport = transport;
    this.datagramWriter = transport.datagrams.writable.getWriter();
    this.datagramReader = transport.datagrams.readable.getReader();
    this._mtu = 1200;
  }

  get mtu() {
    return this._mtu;
  }

  async send(packet) {
    if (packet.length > this._mtu) {
      throw new Error("Packet too large");
    }
    await this.datagramWriter.write(packet);
  }

  async receive() {
    const { value, done } = await this.datagramReader.read();
    if (done) {
      throw new Error("Connection closed");
    }
    return value;
  }

  close() {
    this.transport.close();
  }
}
```

## Server Configuration

### Nginx (Reverse Proxy)

```nginx
# Note: Nginx doesn't natively support HTTP/3 WebTransport
# Use a dedicated HTTP/3 server or Cloudflare
```

### Caddy

```caddyfile
example.com {
    # Enable HTTP/3
    servers {
        protocol {
            experimental_http3
        }
    }
    
    reverse_proxy /tng localhost:8443 {
        transport http {
            versions h3
        }
    }
}
```

### Direct Deployment

For full control, run HTTP/3 server directly:

```python
# aioquic-based server
config = QuicConfiguration(
    alpn_protocols=["h3"],
    max_datagram_frame_size=1400,
)
```

## Browser Considerations

### CORS

WebTransport doesn't use CORS (it's not HTTP-based after initial handshake).

### Same-Origin Policy

WebTransport can connect to any origin with valid certificate.

### Service Workers

WebTransport can be used within service workers for background sync.

```javascript
// In service worker
self.addEventListener('sync', async (event) => {
  const transport = new WebTransport("https://example.com/tng");
  await transport.ready;
  // ... sync data
});
```

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `WebTransportError` | Connection failed | Check URL and certificate |
| `NetworkError` | Server unreachable | Retry with backoff |
| `InvalidStateError` | Already closed | Reconnect |

```javascript
try {
  const transport = new WebTransport(url);
  await transport.ready;
} catch (error) {
  if (error instanceof WebTransportError) {
    console.error("WebTransport failed:", error.message);
    // Fall back to alternative transport
  }
}
```

## Limitations

| Limitation | Description |
|------------|-------------|
| **HTTPS only** | No HTTP or localhost (without flags) |
| **Certificate** | Must be valid, CA-signed |
| **Browser only** | Native support varies |
| **No P2P** | Requires server intermediary |

## Fallback Strategy

When WebTransport fails, consider:

```javascript
async function connectWithFallback(endpoint) {
  // Try WebTransport first
  if (window.WebTransport) {
    try {
      return await connectWebTransport(endpoint);
    } catch (e) {
      console.warn("WebTransport failed, trying fallback");
    }
  }
  
  // Fall back to WebSocket (not ideal, but works)
  return await connectWebSocket(endpoint);
}
```

---

*Related: [Transport Overview](README.md) | [QUIC](quic.md) | [Bluetooth](bluetooth.md)*
