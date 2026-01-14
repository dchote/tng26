# Bluetooth LE Transport

**Bluetooth Low Energy (BLE)** enables TNG for short-range, low-power communication. It's ideal for mobile devices, wearables, and IoT devices that need local mesh connectivity.

## Overview

| Property | Value |
|----------|-------|
| **Transport** | Bluetooth LE 5.x |
| **Range** | 10-100 meters |
| **MTU** | 247 bytes (negotiable) |
| **Power** | Very low |
| **Discovery** | BLE advertising |

## Why Bluetooth LE?

| Feature | Benefit |
|---------|---------|
| **Low power** | Battery-friendly for mobile/IoT |
| **Ubiquitous** | Available on most modern devices |
| **Short range** | Natural security boundary |
| **No infrastructure** | Works without WiFi/cellular |

## Path Format

```json
{
  "type": "bluetooth",
  "address": "AA:BB:CC:DD:EE:FF"
}
```

### Address Types

| Type | Format | Example |
|------|--------|---------|
| Public | 48-bit IEEE | `AA:BB:CC:DD:EE:FF` |
| Random Static | 48-bit random | `FA:BB:CC:DD:EE:FF` |

## GATT Service

TNG defines a custom GATT service for communication:

### Service UUID

```
TNG Service: 0000FE50-0000-1000-8000-00805F9B34FB
```

### Characteristics

| Characteristic | UUID | Properties | Description |
|----------------|------|------------|-------------|
| **TX** | `0000FE51-...` | Write, Write Without Response | Send packets |
| **RX** | `0000FE52-...` | Notify, Indicate | Receive packets |
| **MTU** | `0000FE53-...` | Read | Negotiated MTU |

### Characteristic UUIDs

```
TX:  0000FE51-0000-1000-8000-00805F9B34FB
RX:  0000FE52-0000-1000-8000-00805F9B34FB
MTU: 0000FE53-0000-1000-8000-00805F9B34FB
```

## L2CAP Connection-Oriented Channels

For higher throughput, use L2CAP CoC (preferred over GATT):

### PSM (Protocol/Service Multiplexer)

```
TNG L2CAP PSM: 0x0050 (dynamic range)
```

### L2CAP CoC Benefits

| Feature | GATT | L2CAP CoC |
|---------|------|-----------|
| MTU | 247 bytes | 65535 bytes |
| Throughput | ~10 KB/s | ~100 KB/s |
| Latency | Higher | Lower |
| Complexity | Lower | Higher |

## Discovery

### Advertising

TNG endpoints advertise their presence:

```python
# Advertising data
adv_data = {
    "flags": [LE_GENERAL_DISCOVERABLE, BR_EDR_NOT_SUPPORTED],
    "complete_local_name": "TNG-Device",
    "service_uuids_16": [0xFE50],  # TNG service
}

# Scan response (optional)
scan_response = {
    "manufacturer_data": {
        0xFFFF: hashname[:8]  # First 8 bytes of hashname
    }
}
```

### Scanning

```python
async def scan_for_tng_devices(timeout=10.0):
    """Scan for nearby TNG devices"""
    devices = []
    
    async with BleakScanner() as scanner:
        await asyncio.sleep(timeout)
        
        for device in scanner.discovered_devices:
            if 0xFE50 in device.metadata.get("uuids", []):
                devices.append({
                    "address": device.address,
                    "name": device.name,
                    "rssi": device.rssi,
                })
    
    return devices
```

## Connection Flow

### As Central (Client)

```python
from bleak import BleakClient

async def connect_ble(path: dict) -> Pipe:
    address = path["address"]
    
    client = BleakClient(address)
    await client.connect()
    
    # Negotiate MTU
    mtu = await client.get_mtu()
    
    # Subscribe to RX notifications
    await client.start_notify(RX_CHAR_UUID, on_packet_received)
    
    return BlePipe(client, mtu)
```

### As Peripheral (Server)

```python
from bleak import BleakServer

async def start_ble_server(mesh):
    server = BleakServer()
    
    # Add TNG service
    service = server.add_service(TNG_SERVICE_UUID)
    
    tx_char = service.add_characteristic(
        TX_CHAR_UUID,
        ["write", "write-without-response"],
        on_write=lambda data: mesh.receive_packet(data)
    )
    
    rx_char = service.add_characteristic(
        RX_CHAR_UUID,
        ["notify"],
    )
    
    await server.start()
    return server
```

## MTU Negotiation

### Request Larger MTU

```python
async def negotiate_mtu(client, desired_mtu=512):
    """Request larger MTU for better throughput"""
    try:
        actual_mtu = await client.mtu_exchange(desired_mtu)
        return actual_mtu - 3  # ATT header overhead
    except Exception:
        return 20  # Minimum MTU
```

### MTU Considerations

| Platform | Max MTU | Notes |
|----------|---------|-------|
| iOS | 512 | Automatic negotiation |
| Android | 517 | Request required |
| Linux | 512 | Depends on adapter |
| Windows | 512 | Depends on driver |

## Packet Fragmentation

For packets larger than MTU:

```python
def fragment_packet(packet: bytes, mtu: int) -> list[bytes]:
    """Fragment packet for BLE transmission"""
    chunks = []
    
    for i in range(0, len(packet), mtu - 1):
        chunk = packet[i:i + mtu - 1]
        
        # Add header byte
        header = 0x00
        if i == 0:
            header |= 0x80  # First fragment
        if i + mtu - 1 >= len(packet):
            header |= 0x40  # Last fragment
        
        chunks.append(bytes([header]) + chunk)
    
    return chunks

def reassemble_packet(fragments: list[bytes]) -> bytes:
    """Reassemble fragmented BLE packet"""
    packet = b""
    
    for fragment in fragments:
        header = fragment[0]
        data = fragment[1:]
        packet += data
        
        if header & 0x40:  # Last fragment
            break
    
    return packet
```

## Pipe Implementation

```python
class BlePipe(Pipe):
    def __init__(self, client, mtu):
        self.client = client
        self._mtu = mtu
        self.recv_queue = asyncio.Queue()
        self.fragment_buffer = []
    
    @property
    def mtu(self) -> int:
        return self._mtu
    
    async def send(self, packet: bytes):
        if len(packet) <= self._mtu:
            await self.client.write_gatt_char(TX_CHAR_UUID, packet)
        else:
            # Fragment and send
            for fragment in fragment_packet(packet, self._mtu):
                await self.client.write_gatt_char(TX_CHAR_UUID, fragment)
    
    async def receive(self) -> bytes:
        return await self.recv_queue.get()
    
    def on_notification(self, data: bytes):
        # Handle fragmentation
        header = data[0]
        if header & 0x80:  # First fragment
            self.fragment_buffer = [data]
        else:
            self.fragment_buffer.append(data)
        
        if header & 0x40:  # Last fragment
            packet = reassemble_packet(self.fragment_buffer)
            self.recv_queue.put_nowait(packet)
            self.fragment_buffer = []
    
    def close(self):
        asyncio.create_task(self.client.disconnect())
```

## Power Management

### Connection Intervals

| Mode | Interval | Power | Latency |
|------|----------|-------|---------|
| High performance | 7.5-15ms | High | Low |
| Balanced | 30-50ms | Medium | Medium |
| Low power | 100-500ms | Low | High |

### Recommendations

```python
# For real-time communication
conn_params = {
    "min_interval": 7.5,   # ms
    "max_interval": 15,    # ms
    "latency": 0,
    "timeout": 2000,       # ms
}

# For background sync
conn_params = {
    "min_interval": 100,
    "max_interval": 200,
    "latency": 4,          # Skip up to 4 intervals
    "timeout": 6000,
}
```

## Platform-Specific Notes

### iOS

```swift
// CoreBluetooth usage
let peripheral = CBPeripheral(...)
peripheral.writeValue(data, for: characteristic, type: .withoutResponse)
```

### Android

```kotlin
// Android BLE
val gatt = device.connectGatt(context, false, callback)
gatt.requestMtu(512)
```

### Web Bluetooth

```javascript
// Limited browser support
const device = await navigator.bluetooth.requestDevice({
  filters: [{ services: ['0000fe50-0000-1000-8000-00805f9b34fb'] }]
});
```

## Security Considerations

### Pairing

TNG doesn't require BLE pairing (E3X provides security), but pairing can add:
- Privacy (random addresses)
- Connection encryption (defense in depth)

### Just Works Pairing

```python
# If pairing required
io_capability = NO_INPUT_NO_OUTPUT  # Just Works
```

### Privacy

Use random resolvable addresses to prevent tracking:

```python
# Enable address rotation
advertiser.set_privacy_mode(RESOLVABLE_PRIVATE_ADDRESS)
```

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| `BleakError` | Connection failed | Retry with backoff |
| `ATT_ERROR` | Write rejected | Check MTU, retry |
| `DISCONNECTED` | Link lost | Reconnect |

---

*Related: [Transport Overview](README.md) | [QUIC](quic.md) | [WebTransport](webtransport.md)*
