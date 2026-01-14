# LoRa Physical Layer

**LoRa (Long Range)** is a spread-spectrum modulation technique enabling long-range, low-power communication. It's ideal for IoT applications requiring kilometer-range connectivity with minimal infrastructure.

## Overview

| Property | Value |
|----------|-------|
| **Modulation** | CSS (Chirp Spread Spectrum) |
| **Frequency** | 868 MHz (EU), 915 MHz (US), 433 MHz |
| **Range** | 2-15+ km (rural), 1-5 km (urban) |
| **Data Rate** | 0.3-50 kbps |
| **MTU** | 255 bytes |

## Why LoRa?

| Feature | Benefit |
|---------|---------|
| **Long range** | Kilometers without infrastructure |
| **Low power** | Years on battery |
| **Penetration** | Good building penetration |
| **License-free** | ISM bands |

## Path Format

```json
{
  "type": "lora",
  "address": 12345678,
  "spreading_factor": 7,
  "bandwidth": 125000,
  "frequency": 868100000
}
```

## LoRa vs LoRaWAN

| Aspect | LoRa (P2P) | LoRaWAN |
|--------|------------|---------|
| **Architecture** | Peer-to-peer | Star-of-stars |
| **Complexity** | Simple | Complex |
| **Infrastructure** | None | Gateways required |
| **TNG Use** | Primary | Via gateway |

TNG primarily uses LoRa in peer-to-peer mode.

## Radio Parameters

### Spreading Factor (SF)

| SF | Data Rate | Range | Airtime (50B) |
|----|-----------|-------|---------------|
| 7 | 5.47 kbps | Short | 56 ms |
| 8 | 3.13 kbps | Medium | 100 ms |
| 9 | 1.76 kbps | Medium | 180 ms |
| 10 | 0.98 kbps | Long | 370 ms |
| 11 | 0.54 kbps | Long | 740 ms |
| 12 | 0.29 kbps | Very Long | 1.4 s |

### Bandwidth

| Bandwidth | Use Case |
|-----------|----------|
| 125 kHz | Standard |
| 250 kHz | Higher data rate |
| 500 kHz | Fastest |

### Coding Rate

| CR | Overhead | Error Correction |
|----|----------|------------------|
| 4/5 | 25% | Low |
| 4/6 | 50% | Medium |
| 4/7 | 75% | High |
| 4/8 | 100% | Maximum |

## Regional Frequencies

### EU868

| Channel | Frequency | Duty Cycle |
|---------|-----------|------------|
| 0 | 868.1 MHz | 1% |
| 1 | 868.3 MHz | 1% |
| 2 | 868.5 MHz | 1% |
| 3-7 | 867.1-867.9 MHz | 1% |

### US915

| Sub-band | Channels | Frequency Range |
|----------|----------|-----------------|
| 1 | 0-7 | 902.3-903.7 MHz |
| 2 | 8-15 | 903.9-905.3 MHz |
| ... | ... | ... |

### AS923

| Channel | Frequency |
|---------|-----------|
| 0 | 923.2 MHz |
| 1 | 923.4 MHz |

## Duty Cycle Compliance

### EU Regulations

```python
class DutyCycleManager:
    # EU868 sub-bands
    DUTY_CYCLES = {
        (868.0, 868.6): 0.01,    # 1%
        (868.7, 869.2): 0.001,   # 0.1%
        (869.4, 869.65): 0.10,   # 10%
    }
    
    def __init__(self):
        self.usage = defaultdict(float)
        self.window = 3600  # 1 hour
    
    def get_duty_cycle(self, freq: float) -> float:
        for (low, high), duty in self.DUTY_CYCLES.items():
            if low <= freq / 1e6 <= high:
                return duty
        return 0.01  # Default 1%
    
    def can_transmit(self, freq: float, airtime: float) -> bool:
        duty = self.get_duty_cycle(freq)
        current_usage = self.usage[freq]
        return (current_usage + airtime) / self.window <= duty
    
    def record_transmission(self, freq: float, airtime: float):
        self.usage[freq] += airtime
```

### Time-on-Air Calculation

```python
def calculate_airtime(
    payload_size: int,
    sf: int = 7,
    bw: int = 125000,
    cr: int = 5,
    preamble: int = 8,
    header: bool = True,
    crc: bool = True
) -> float:
    """Calculate LoRa transmission time in seconds"""
    
    # Symbol duration
    t_sym = (2 ** sf) / bw
    
    # Preamble duration
    t_preamble = (preamble + 4.25) * t_sym
    
    # Payload symbols
    de = 1 if sf >= 11 and bw == 125000 else 0
    h = 0 if header else 1
    crc_bits = 16 if crc else 0
    
    payload_bits = 8 * payload_size - 4 * sf + 28 + crc_bits - 20 * h
    payload_symbols = 8 + max(
        math.ceil(payload_bits / (4 * (sf - 2 * de))) * cr,
        0
    )
    
    t_payload = payload_symbols * t_sym
    
    return t_preamble + t_payload
```

## Chunking

Due to 255-byte MTU:

### Chunk Header

```
Byte 0: [FIRST:1][LAST:1][SEQ:6]
Byte 1: [LENGTH_HIGH] (first chunk only)
Byte 2: [LENGTH_LOW]  (first chunk only)
Byte 3: [DEST_ADDR_HIGH] (first chunk only)
Byte 4: [DEST_ADDR_LOW]  (first chunk only)
```

### Parameters

| Parameter | Value |
|-----------|-------|
| MTU | 255 bytes |
| Header (first) | 5 bytes |
| Header (cont) | 1 byte |
| Payload (first) | 250 bytes |
| Payload (cont) | 254 bytes |

## Implementation

### RadioHead Library (Arduino)

```cpp
#include <RH_RF95.h>

RH_RF95 rf95(CS_PIN, INT_PIN);

void setup() {
    rf95.init();
    rf95.setFrequency(868.1);
    rf95.setSpreadingFactor(7);
    rf95.setSignalBandwidth(125000);
    rf95.setCodingRate4(5);
    rf95.setTxPower(14);
}

void sendPacket(uint8_t* data, uint8_t len) {
    rf95.send(data, len);
    rf95.waitPacketSent();
}
```

### SX127x Direct (Python)

```python
from sx127x import SX127x

class LoRaTransport:
    def __init__(self, spi_bus, cs_pin, reset_pin, dio0_pin):
        self.radio = SX127x(spi_bus, cs_pin, reset_pin, dio0_pin)
        
        # Configure for EU868
        self.radio.set_frequency(868.1e6)
        self.radio.set_spreading_factor(7)
        self.radio.set_bandwidth(125e3)
        self.radio.set_coding_rate(5)
        self.radio.set_tx_power(14)
    
    async def send(self, data: bytes):
        # Check duty cycle
        airtime = calculate_airtime(len(data))
        if not self.duty_cycle.can_transmit(self.frequency, airtime):
            raise DutyCycleExceeded()
        
        self.radio.send(data)
        await self.radio.wait_tx_done()
        
        self.duty_cycle.record_transmission(self.frequency, airtime)
    
    async def receive(self, timeout: float = None) -> bytes:
        self.radio.receive()
        
        if timeout:
            data = await asyncio.wait_for(
                self.radio.wait_rx_done(),
                timeout
            )
        else:
            data = await self.radio.wait_rx_done()
        
        return data
```

### LoRa-E5 Module (AT Commands)

```python
class LoRaE5:
    def __init__(self, serial_port: str):
        self.serial = serial.Serial(serial_port, 9600)
    
    def configure(self, freq: int, sf: int, bw: int):
        self.command(f"AT+MODE=TEST")
        self.command(f"AT+TEST=RFCFG,{freq},{sf},125,12,15,14,ON,OFF,OFF")
    
    def send(self, data: bytes):
        hex_data = data.hex()
        self.command(f"AT+TEST=TXLRPKT,\"{hex_data}\"")
    
    def receive(self) -> bytes:
        response = self.wait_for("+TEST: RX")
        hex_data = response.split('"')[1]
        return bytes.fromhex(hex_data)
    
    def command(self, cmd: str) -> str:
        self.serial.write(f"{cmd}\r\n".encode())
        return self.serial.readline().decode().strip()
```

## Addressing

### Device Address

4-byte address for P2P communication:

```python
# Generate from hashname
def hashname_to_lora_addr(hashname: str) -> int:
    h = hashlib.sha256(hashname.encode()).digest()
    return struct.unpack('>I', h[:4])[0]

# Broadcast address
BROADCAST_ADDR = 0xFFFFFFFF
```

### Packet Format

```
┌───────────────────────────────────────────────────────────────────┐
│ Dest Addr │ Src Addr │ Length │ Payload │ CRC16                   │
│   (4B)    │   (4B)   │  (1B)  │ (0-242) │  (2B)                   │
└───────────────────────────────────────────────────────────────────┘
```

## Channel Hopping

For longer transmissions or regulatory compliance:

```python
class ChannelHopper:
    def __init__(self, channels: list[float]):
        self.channels = channels
        self.index = 0
    
    def next_channel(self) -> float:
        channel = self.channels[self.index]
        self.index = (self.index + 1) % len(self.channels)
        return channel

# EU868 channels
hopper = ChannelHopper([868.1, 868.3, 868.5])
```

## Power Management

### Listen Before Talk (LBT)

```python
def can_transmit_lbt(radio, threshold: int = -80) -> bool:
    """Check channel before transmitting"""
    rssi = radio.get_rssi()
    return rssi < threshold
```

### CAD (Channel Activity Detection)

```python
async def wait_for_clear_channel(radio, max_wait: float = 5.0):
    """Wait for channel to be clear using CAD"""
    start = time.time()
    
    while time.time() - start < max_wait:
        if not radio.cad_detected():
            return True
        await asyncio.sleep(0.1)
    
    return False
```

## Gateway Mode

For connecting LoRa devices to IP networks:

```python
class LoRaGateway:
    def __init__(self, lora: LoRaTransport, mesh: Mesh):
        self.lora = lora
        self.mesh = mesh
    
    async def run(self):
        while True:
            # Receive from LoRa
            packet = await self.lora.receive()
            
            # Forward to TNG mesh
            src_addr, dest_addr, data = parse_lora_packet(packet)
            
            if dest_addr == self.lora_addr:
                # For local mesh
                self.mesh.receive_packet(data)
            else:
                # Route to destination
                await self.forward_to_mesh(dest_addr, data)
```

---

*Related: [Transport Overview](README.md) | [802.15.4](802.15.4.md) | [433/915MHz](uart-radio.md)*
