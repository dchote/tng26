# 433/915 MHz UART Radio

Generic sub-GHz radio modules with UART interfaces provide a simple, accessible entry point for TNG on embedded systems. These modules are widely available, inexpensive, and easy to integrate.

## Overview

| Property | Value |
|----------|-------|
| **Frequency** | 433 MHz (worldwide), 915 MHz (Americas) |
| **Range** | 100m - 1km (typical) |
| **Data Rate** | 1-100 kbps |
| **MTU** | 64 bytes (typical) |
| **Interface** | UART (serial) |

## Common Modules

| Module | Frequency | Range | Data Rate | MTU |
|--------|-----------|-------|-----------|-----|
| **HC-12** | 433.4-473 MHz | 1000m | 5-15 kbps | 60 bytes |
| **E32 (SX1278)** | 433/868/915 MHz | 3000m | 0.3-19.2 kbps | 58 bytes |
| **CC1101** | 315/433/868/915 MHz | 500m | 1.2-500 kbps | 61 bytes |
| **nRF905** | 433/868/915 MHz | 300m | 50 kbps | 32 bytes |
| **SYN115/SYN480R** | 315/433 MHz | 200m | 2.4 kbps | 64 bytes |

## Path Format

```json
{
  "type": "uart-radio",
  "address": 1234,
  "channel": 1,
  "baud_rate": 9600
}
```

## Frame Format

### Wire Format

```
┌──────────────────────────────────────────────────────────────────┐
│ SYNC │ SYNC │ LENGTH │ DEST │ SRC │ PAYLOAD │ CRC16              │
│ 0x7E │ 0x7E │  (1B)  │ (2B) │(2B) │(0-58B)  │  (2B)              │
└──────────────────────────────────────────────────────────────────┘
```

| Field | Size | Description |
|-------|------|-------------|
| SYNC | 2 bytes | Frame delimiter (0x7E 0x7E) |
| LENGTH | 1 byte | Payload + addresses length |
| DEST | 2 bytes | Destination address |
| SRC | 2 bytes | Source address |
| PAYLOAD | 0-58 bytes | TNG chunk data |
| CRC16 | 2 bytes | CRC-16-CCITT |

### Byte Stuffing

Escape 0x7E and 0x7D in payload:

| Byte | Escaped As |
|------|------------|
| 0x7E | 0x7D 0x5E |
| 0x7D | 0x7D 0x5D |

```python
def escape_frame(data: bytes) -> bytes:
    """Escape special bytes in frame"""
    result = bytearray()
    for byte in data:
        if byte == 0x7E:
            result.extend([0x7D, 0x5E])
        elif byte == 0x7D:
            result.extend([0x7D, 0x5D])
        else:
            result.append(byte)
    return bytes(result)

def unescape_frame(data: bytes) -> bytes:
    """Unescape special bytes in frame"""
    result = bytearray()
    i = 0
    while i < len(data):
        if data[i] == 0x7D and i + 1 < len(data):
            result.append(data[i + 1] ^ 0x20)
            i += 2
        else:
            result.append(data[i])
            i += 1
    return bytes(result)
```

## Chunking

Due to small MTU (typically 64 bytes):

### Chunk Header

```
Byte 0: [FIRST:1][LAST:1][SEQ:6]
Byte 1: [LENGTH_HIGH] (first chunk only)
Byte 2: [LENGTH_LOW]  (first chunk only)
```

### Parameters

| Module | MTU | Header | Max Payload |
|--------|-----|--------|-------------|
| HC-12 | 60B | 3B | 57B |
| E32 | 58B | 3B | 55B |
| Generic | 64B | 3B | 61B |

## Implementation

### Basic Serial Communication

```python
import serial
import struct

class UARTRadio:
    SYNC = b'\x7E\x7E'
    
    def __init__(self, port: str, baud_rate: int = 9600):
        self.serial = serial.Serial(
            port=port,
            baudrate=baud_rate,
            timeout=0.1
        )
        self.address = 0
    
    def send(self, dest: int, data: bytes):
        """Send frame to destination"""
        # Build frame
        length = len(data) + 4  # data + addresses
        frame = struct.pack('>BHH', length, dest, self.address) + data
        
        # Add CRC
        crc = crc16_ccitt(frame)
        frame += struct.pack('>H', crc)
        
        # Escape and add sync
        escaped = escape_frame(frame)
        packet = self.SYNC + escaped
        
        self.serial.write(packet)
    
    def receive(self) -> tuple[int, bytes]:
        """Receive frame, return (src_addr, data)"""
        # Wait for sync
        while True:
            byte = self.serial.read(1)
            if byte == b'\x7E':
                next_byte = self.serial.read(1)
                if next_byte == b'\x7E':
                    break
        
        # Read length
        length_byte = self._read_unescaped(1)
        length = length_byte[0]
        
        # Read frame
        frame = self._read_unescaped(length + 2)  # +2 for CRC
        
        # Verify CRC
        crc = struct.unpack('>H', frame[-2:])[0]
        if crc16_ccitt(frame[:-2]) != crc:
            raise CRCError()
        
        # Parse
        dest, src = struct.unpack('>HH', frame[:4])
        data = frame[4:-2]
        
        if dest != self.address and dest != 0xFFFF:
            return None  # Not for us
        
        return src, data
    
    def _read_unescaped(self, n: int) -> bytes:
        """Read n bytes, handling escape sequences"""
        result = bytearray()
        while len(result) < n:
            byte = self.serial.read(1)
            if byte == b'\x7D':
                next_byte = self.serial.read(1)
                result.append(next_byte[0] ^ 0x20)
            else:
                result.append(byte[0])
        return bytes(result)
```

### HC-12 Configuration

```python
class HC12(UARTRadio):
    def configure(self, channel: int = 1, power: int = 8, baud: int = 9600):
        """Configure HC-12 module via AT commands"""
        # Enter AT mode
        self.set_pin.low()  # SET pin low
        time.sleep(0.1)
        
        # Set channel (001-127)
        self._at_command(f"AT+C{channel:03d}")
        
        # Set power (1-8, 8=20dBm)
        self._at_command(f"AT+P{power}")
        
        # Set baud rate
        baud_codes = {1200: 1, 2400: 2, 4800: 3, 9600: 4, 19200: 5}
        self._at_command(f"AT+B{baud_codes[baud]}")
        
        # Exit AT mode
        self.set_pin.high()
        time.sleep(0.1)
    
    def _at_command(self, cmd: str) -> str:
        self.serial.write(f"{cmd}\r\n".encode())
        return self.serial.readline().decode().strip()
```

### E32 (SX1278) Configuration

```python
class E32(UARTRadio):
    def configure(
        self,
        address: int = 0,
        channel: int = 0,
        air_rate: int = 2,  # 0=0.3k, 1=1.2k, 2=2.4k, 3=4.8k, 4=9.6k, 5=19.2k
        power: int = 3,     # 0=20dBm, 1=17dBm, 2=14dBm, 3=10dBm
    ):
        """Configure E32 module"""
        # Enter config mode
        self.m0_pin.high()
        self.m1_pin.high()
        time.sleep(0.1)
        
        # Build config bytes
        config = bytes([
            0xC0,  # Save parameters
            (address >> 8) & 0xFF,
            address & 0xFF,
            (air_rate << 3) | 0x04,  # Air rate + UART 9600
            channel,
            0xC4 | power,  # Fixed mode + power
        ])
        
        self.serial.write(config)
        time.sleep(0.1)
        
        # Exit config mode
        self.m0_pin.low()
        self.m1_pin.low()
        time.sleep(0.1)
```

## Addressing

### Address Space

| Address | Purpose |
|---------|---------|
| 0x0000 | Reserved |
| 0x0001-0xFFFE | Device addresses |
| 0xFFFF | Broadcast |

### Address Assignment

```python
def hashname_to_uart_addr(hashname: str) -> int:
    """Generate UART address from hashname"""
    h = hashlib.sha256(hashname.encode()).digest()
    addr = struct.unpack('>H', h[:2])[0]
    
    # Avoid reserved addresses
    if addr == 0x0000:
        addr = 0x0001
    elif addr == 0xFFFF:
        addr = 0xFFFE
    
    return addr
```

## Pipe Implementation

```python
class UARTRadioPipe:
    def __init__(self, radio: UARTRadio, remote_addr: int):
        self.radio = radio
        self.remote_addr = remote_addr
        self._mtu = 61  # Conservative default
        self.recv_queue = asyncio.Queue()
    
    @property
    def mtu(self) -> int:
        return self._mtu
    
    async def send(self, packet: bytes):
        """Send packet, chunking if necessary"""
        chunks = chunk_packet(packet, self._mtu)
        
        for chunk in chunks:
            self.radio.send(self.remote_addr, chunk)
            await asyncio.sleep(0.05)  # Inter-packet delay
    
    async def receive(self) -> bytes:
        return await self.recv_queue.get()
    
    def on_data(self, src: int, data: bytes):
        """Called when data received from this peer"""
        if src == self.remote_addr:
            self.recv_queue.put_nowait(data)
```

## Collision Avoidance

### Simple CSMA

```python
class CSMARadio(UARTRadio):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.backoff_min = 0.01  # 10ms
        self.backoff_max = 0.1   # 100ms
    
    async def send_with_csma(self, dest: int, data: bytes):
        """Send with carrier sense"""
        attempts = 0
        max_attempts = 5
        
        while attempts < max_attempts:
            # Check if channel is clear (some modules support RSSI)
            if self.channel_clear():
                try:
                    self.send(dest, data)
                    return
                except Exception:
                    pass
            
            # Random backoff
            backoff = random.uniform(
                self.backoff_min * (2 ** attempts),
                self.backoff_max * (2 ** attempts)
            )
            await asyncio.sleep(backoff)
            attempts += 1
        
        raise ChannelBusy()
```

### Time-Division

For deterministic access:

```python
class TDMARadio(UARTRadio):
    def __init__(self, *args, slot: int, total_slots: int, slot_duration: float):
        super().__init__(*args)
        self.slot = slot
        self.total_slots = total_slots
        self.slot_duration = slot_duration
    
    async def wait_for_slot(self):
        """Wait for our time slot"""
        now = time.time()
        cycle_duration = self.total_slots * self.slot_duration
        
        # Calculate next slot start
        cycle_start = (now // cycle_duration) * cycle_duration
        slot_start = cycle_start + (self.slot * self.slot_duration)
        
        if slot_start < now:
            slot_start += cycle_duration
        
        await asyncio.sleep(slot_start - now)
```

## Discovery

### Beacon Broadcast

```python
async def broadcast_beacon(radio: UARTRadio, mesh):
    """Periodically broadcast presence"""
    beacon = {
        "type": "discover",
        "hashname": mesh.hashname[:8],  # Truncated
        "addr": radio.address,
    }
    
    while True:
        data = json.dumps(beacon).encode()
        radio.send(0xFFFF, data)  # Broadcast
        await asyncio.sleep(30)  # Every 30 seconds
```

### Listen for Beacons

```python
async def listen_for_beacons(radio: UARTRadio):
    """Listen for discovery beacons"""
    discovered = {}
    
    while True:
        try:
            src, data = radio.receive()
            if src is None:
                continue
            
            beacon = json.loads(data.decode())
            if beacon.get("type") == "discover":
                discovered[src] = beacon
                
        except (json.JSONDecodeError, UnicodeDecodeError):
            pass  # Not a beacon
        
        await asyncio.sleep(0.01)
```

## Power Considerations

### Sleep Mode

```python
class SleepableRadio(UARTRadio):
    def sleep(self):
        """Put radio in low-power mode"""
        # Module-specific sleep command
        self._at_command("AT+SLEEP")
    
    def wake(self):
        """Wake radio from sleep"""
        # Send any byte to wake
        self.serial.write(b'\xFF')
        time.sleep(0.1)
```

### Scheduled Wake

```python
class ScheduledRadio(SleepableRadio):
    def __init__(self, *args, wake_interval=60, wake_duration=5):
        super().__init__(*args)
        self.wake_interval = wake_interval
        self.wake_duration = wake_duration
    
    async def run_duty_cycle(self):
        while True:
            # Wake period
            self.wake()
            await asyncio.sleep(self.wake_duration)
            
            # Sleep period
            self.sleep()
            await asyncio.sleep(self.wake_interval - self.wake_duration)
```

## Error Handling

| Error | Cause | Action |
|-------|-------|--------|
| CRC error | Corruption | Discard, request retry |
| Timeout | Lost packet | Retry |
| Busy channel | Collision | Backoff and retry |
| No response | Out of range | Try different channel |

---

*Related: [Transport Overview](README.md) | [LoRa](lora.md) | [802.15.4](802.15.4.md)*
