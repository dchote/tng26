# E3X - End-to-End Encrypted Exchange

**E3X** is the cryptographic core of TNG, providing end-to-end encrypted communication between endpoints. It handles all security primitives: key exchange, session establishment, message encryption, and channel management.

## Overview

E3X provides two communication modes:

| Mode | Description | Use Case |
|------|-------------|----------|
| **Messages** | Asynchronous encrypted packets | One-off communication, handshakes |
| **Channels** | Synchronous encrypted streams | Ongoing data transfer |

Both modes ensure:
- **Confidentiality**: Only intended recipient can read
- **Integrity**: Tampering is detected
- **Authentication**: Sender identity is verified

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Application                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         Channels                            │
│              (reliable / unreliable streams)                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                         Exchange                            │
│              (session state between endpoints)              │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
       ┌───────────┐   ┌───────────┐   ┌───────────┐
       │ Handshake │   │  Session  │   │ Messages  │
       │  (sync)   │   │   Keys    │   │  (async)  │
       └───────────┘   └───────────┘   └───────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      Cipher Suite                           │
│              (CS4a, CS4b, CS3a, etc.)                       │
└─────────────────────────────────────────────────────────────┘
```

## Exchange

An **Exchange** represents the cryptographic session state between two endpoints. It is created by combining:

1. **Local keys**: Your endpoint's private and public keys
2. **Remote keys**: The other endpoint's public keys

### Exchange State

| Property | Description |
|----------|-------------|
| `local_csid` | Local cipher suite ID |
| `remote_csid` | Remote cipher suite ID |
| `session_id` | 16-byte unique identifier |
| `ephemeral_keys` | Per-session key pair |
| `shared_secret` | Derived key material |
| `at` | Handshake timestamp |
| `encrypt_key` | Outgoing encryption key |
| `decrypt_key` | Incoming decryption key |

### Token

Each exchange has a 16-byte **token** derived from the ephemeral keys. This token:
- Identifies the exchange without revealing identity
- Is included in channel packets for routing
- Changes when ephemeral keys rotate

## Messages (Asynchronous)

Messages are standalone encrypted packets that can be sent without an established session. They're used for:

- Initial handshakes
- Out-of-band communication
- Store-and-forward scenarios

### Message Encryption

```
┌─────────────────────────────────────────────────────────────┐
│  Ephemeral Public Key (32 bytes)                            │
├─────────────────────────────────────────────────────────────┤
│  Nonce (24 bytes)                                           │
├─────────────────────────────────────────────────────────────┤
│  Encrypted Inner Packet (variable)                          │
├─────────────────────────────────────────────────────────────┤
│  MAC (16 bytes)                                             │
└─────────────────────────────────────────────────────────────┘
```

### Message Flow

1. Sender generates ephemeral key pair
2. Sender computes shared secret with recipient's public key
3. Sender encrypts inner packet with AEAD
4. Recipient uses ephemeral public key + own private key to decrypt

## Handshakes

A **handshake** is a special message type used to establish an exchange and derive session keys. TNG handshakes follow patterns inspired by the [Noise Protocol Framework](https://noiseprotocol.org/).

### Handshake Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **XX** | Mutual authentication, both keys transmitted | General use |
| **IK** | Initiator known, responder transmits | Connecting to known peer |
| **NK** | Responder known, no initiator key | Anonymous initiation |

### XX Handshake (Default)

```
    Initiator                              Responder
        │                                      │
        │  ── Message 1: e, s ──────────────►  │
        │     (ephemeral + static key)         │
        │                                      │
        │  ◄── Message 2: e, ee, se, s, es ──  │
        │     (ephemeral + encrypted static)   │
        │                                      │
        │  ── Message 3: es ─────────────────► │
        │     (confirmation)                   │
        │                                      │
       ═══════════ Session Established ════════════
        │                                      │
        │  ◄─────── Channel Data ───────────►  │
```

### Handshake Packet Format

```json
HEAD: {
  "type": "link",
  "at": 1234567890,
  "csid": "4a"
}
BODY: [attached packet with keys and intermediates]
```

The attached packet contains:
- Public key for the handshake cipher suite
- Intermediate hashes for other cipher suites (hashname verification)

### `at` Timestamp

The `at` field is a Unix timestamp (seconds) that:
- Prevents replay attacks
- Establishes handshake ordering
- Should be within reasonable clock skew (±5 minutes)

## Session Keys

After handshake completion, the exchange derives symmetric keys:

```python
# Key derivation (conceptual)
shared_secret = ECDH(local_ephemeral, remote_ephemeral)

encrypt_key = HKDF(
    shared_secret,
    info=local_ephemeral_pub || remote_ephemeral_pub,
    salt=exchange_id
)

decrypt_key = HKDF(
    shared_secret,
    info=remote_ephemeral_pub || local_ephemeral_pub,
    salt=exchange_id
)
```

### Forward Secrecy

Ephemeral keys provide **perfect forward secrecy**:
- Compromise of long-term keys doesn't reveal past sessions
- Each session has unique keys
- Keys are deleted after session ends

## Channel Encryption

Once a session is established, channel packets are encrypted:

```
┌─────────────────────────────────────────────────────────────┐
│  Token (16 bytes) - identifies exchange                     │
├─────────────────────────────────────────────────────────────┤
│  Nonce (24 bytes)                                           │
├─────────────────────────────────────────────────────────────┤
│  Encrypted Channel Packet (variable)                        │
├─────────────────────────────────────────────────────────────┤
│  MAC (16 bytes)                                             │
└─────────────────────────────────────────────────────────────┘
```

### Encryption Process

```python
def encrypt_channel(exchange, inner_packet):
    nonce = generate_random_nonce(24)
    
    ciphertext, mac = AEAD_encrypt(
        key=exchange.encrypt_key,
        nonce=nonce,
        plaintext=inner_packet,
        associated_data=exchange.token
    )
    
    return exchange.token + nonce + ciphertext + mac
```

## Cloaking

**Cloaking** randomizes all bytes on the wire to prevent traffic fingerprinting:

1. A cloaking key is derived from the exchange
2. All packet bytes (including token) are XORed with a keystream
3. Receivers try decloaking with known exchange keys

### Cloaked Packet

```
┌─────────────────────────────────────────────────────────────┐
│  Random Length Padding (0-255 bytes)                        │
├─────────────────────────────────────────────────────────────┤
│  XOR-masked Token + Nonce + Ciphertext + MAC                │
└─────────────────────────────────────────────────────────────┘
```

Cloaking makes TNG traffic indistinguishable from random data.

## Exchange Lifecycle

```
    ┌──────────────┐
    │    Created   │  Exchange created with remote keys
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │   Handshake  │  Sending/receiving handshake messages
    │   Pending    │
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │    Synced    │  Session keys established
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │    Active    │  Channels can be opened
    └──────┬───────┘
           │
           ▼
    ┌──────────────┐
    │    Closed    │  Session ended, keys deleted
    └──────────────┘
```

## API Overview

### Self (Local Endpoint)

```python
class Self:
    def __init__(self, keypairs: dict[str, KeyPair]):
        """Create local endpoint with cipher suite keys"""
    
    def decrypt(self, message: bytes) -> tuple[Exchange, bytes]:
        """Decrypt an incoming message"""
```

### Exchange

```python
class Exchange:
    def __init__(self, local: Self, remote_keys: dict[str, bytes]):
        """Create exchange with remote endpoint's public keys"""
    
    @property
    def token(self) -> bytes:
        """16-byte exchange identifier"""
    
    def handshake(self, at: int) -> bytes:
        """Generate handshake message"""
    
    def verify(self, message: bytes) -> bool:
        """Verify message is from this exchange"""
    
    def sync(self, handshake: bytes) -> bool:
        """Process incoming handshake, establish session"""
    
    def encrypt(self, packet: bytes) -> bytes:
        """Encrypt packet for channel"""
    
    def decrypt(self, packet: bytes) -> bytes:
        """Decrypt incoming channel packet"""
```

## Security Considerations

### Replay Protection

- `at` timestamps prevent handshake replay
- Channel sequence numbers prevent data replay
- Nonces must never repeat with same key

### Identity Protection

- Static keys are encrypted in handshake (pattern dependent)
- Token doesn't reveal hashname
- Cloaking prevents fingerprinting

### Key Compromise

- Ephemeral keys limit damage from compromise
- Rotate long-term keys if compromise suspected
- Use post-quantum cipher suites (CS4b) for future-proofing

---

*Related: [Cipher Suites](cipher-suites.md) | [Channels](channels.md) | [Links](links.md)*
