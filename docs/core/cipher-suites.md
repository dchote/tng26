# Cipher Suites

A **Cipher Suite (CS)** is a collection of cryptographic algorithms used by E3X for key exchange, digital signatures, and authenticated encryption. TNG supports multiple cipher suites to accommodate different security requirements and platform capabilities.

## Overview

Each cipher suite provides:

| Function | Purpose |
|----------|---------|
| **Key Agreement** | Establish shared secret (ECDH or PQ-KEM) |
| **Digital Signature** | Authenticate endpoints |
| **AEAD Cipher** | Encrypt and authenticate data |

## Cipher Suite ID (CSID)

Each cipher suite is identified by a **CSID**, a single byte represented as two lowercase hex characters:

| Format | Description |
|--------|-------------|
| `4a`, `4b` | TNG modern suites |
| `3a` | Legacy compatibility |
| `0x01-0x0f` | Experimental/development |
| `0x10-0x17`, etc. | Application-specific custom |

CSIDs are sorted lexicographically for hashname generation: `1a < 2a < 3a < 4a < 4b`

## Cipher Suite Key (CSK)

The **CSK** is the public key bytes for a cipher suite. Its format and size depend on the cipher suite:

| CSID | CSK Contents | Size |
|------|--------------|------|
| CS4a | X25519 public ‖ Ed25519 public | 64 bytes |
| CS4b | ML-KEM-768 public ‖ X25519 public ‖ Ed25519 public | 1248 bytes |
| CS3a | Curve25519 public | 32 bytes |

## Recommended Cipher Suites

### CS4a (Recommended Default)

**Status**: Active, Recommended

| Component | Algorithm | Size |
|-----------|-----------|------|
| Key Agreement | X25519 | 32 bytes |
| Signature | Ed25519 | 64 bytes |
| AEAD | ChaCha20-Poly1305 | 32-byte key |

**Properties**:
- Modern, well-analyzed algorithms
- Efficient on all platforms
- Constant-time implementations available
- 128-bit security level

**CSK Format** (64 bytes):
```
┌────────────────────────────────────────────────────────────────┐
│  X25519 Public Key (32 bytes)  │  Ed25519 Public Key (32 bytes) │
└────────────────────────────────────────────────────────────────┘
```

**Key Generation**:
```python
def generate_cs4a():
    x25519_private = random_bytes(32)
    x25519_public = x25519_scalarmult_base(x25519_private)
    
    ed25519_seed = random_bytes(32)
    ed25519_private, ed25519_public = ed25519_keypair(ed25519_seed)
    
    csk = x25519_public + ed25519_public
    secrets = x25519_private + ed25519_private
    
    return csk, secrets
```

---

### CS4b (Post-Quantum Hybrid)

**Status**: Active, Recommended for high-security

| Component | Algorithm | Size |
|-----------|-----------|------|
| Key Encapsulation | ML-KEM-768 (FIPS 203) | 1184 bytes public |
| Key Agreement | X25519 | 32 bytes |
| Signature | Ed25519 | 64 bytes |
| AEAD | ChaCha20-Poly1305 | 32-byte key |

**Properties**:
- Quantum-resistant via ML-KEM
- Hybrid mode maintains classical security
- Larger keys and ciphertext
- NIST standardized (2024)

**CSK Format** (1248 bytes):
```
┌────────────────────────────────────────────────────────────────────────┐
│  ML-KEM-768 Public Key (1184 bytes)                                    │
├────────────────────────────────────────────────────────────────────────┤
│  X25519 Public Key (32 bytes)  │  Ed25519 Public Key (32 bytes)        │
└────────────────────────────────────────────────────────────────────────┘
```

**Hybrid Key Exchange**:
```python
def cs4b_key_exchange(local_secret, remote_csk):
    # ML-KEM key encapsulation
    mlkem_shared, mlkem_ciphertext = ml_kem_encaps(remote_csk[:1184])
    
    # X25519 key agreement
    x25519_shared = x25519(local_secret, remote_csk[1184:1216])
    
    # Combine secrets
    shared_secret = HKDF(mlkem_shared || x25519_shared)
    
    return shared_secret, mlkem_ciphertext
```

**When to Use CS4b**:
- Long-term secrets that must remain secure for decades
- Government or regulated environments requiring PQ readiness
- When bandwidth permits larger key sizes
- Future-proofing against quantum attacks

---

### CS3a (Legacy)

**Status**: Legacy, for backward compatibility only

| Component | Algorithm | Size |
|-----------|-----------|------|
| Key Agreement | Curve25519 | 32 bytes |
| AEAD | XSalsa20-Poly1305 | 32-byte key |

**Properties**:
- Compatible with original telehash v3
- No separate signature algorithm (key agreement provides authentication)
- Simpler but less flexible

**CSK Format** (32 bytes):
```
┌────────────────────────────────────────┐
│  Curve25519 Public Key (32 bytes)      │
└────────────────────────────────────────┘
```

**Limitations**:
- No post-quantum protection
- Limited authentication options
- Should not be used for new deployments

---

## CSID Negotiation

When two endpoints connect, they must agree on a cipher suite:

1. Both endpoints advertise supported CSIDs
2. Highest common CSID is selected
3. Handshake proceeds with selected cipher suite

### Negotiation Rules

```python
def select_csid(local_csids: set, remote_csids: set) -> str:
    common = local_csids & remote_csids
    if not common:
        raise NoCipherSuiteMatch()
    
    # Sort and select highest
    # Priority: 4b > 4a > 3a (lexicographic on hex)
    return max(common)
```

### Example

| Local | Remote | Selected |
|-------|--------|----------|
| {4a, 4b, 3a} | {4a, 3a} | 4a |
| {4b, 4a} | {4b, 4a} | 4b |
| {4a} | {3a} | Error: no match |

## Algorithm Details

### X25519

- Elliptic curve Diffie-Hellman on Curve25519
- 32-byte private key, 32-byte public key
- Provides ~128-bit security
- Constant-time implementations resist timing attacks

### Ed25519

- Edwards-curve digital signature
- 32-byte private key, 64-byte signature
- Deterministic signatures (no random nonce needed)
- Fast verification

### ChaCha20-Poly1305

- Stream cipher with authentication
- 32-byte key, 12 or 24-byte nonce
- 16-byte authentication tag
- Efficient in software (no AES hardware needed)

### ML-KEM-768 (FIPS 203)

- Lattice-based key encapsulation
- 1184-byte public key, 2400-byte private key
- 1088-byte ciphertext
- ~192-bit post-quantum security
- NIST standardized August 2024

## Implementation Requirements

### Randomness

All cipher suites require cryptographically secure random number generation:

```python
# Platform-specific secure random
import secrets
random_bytes = secrets.token_bytes
```

### Constant-Time Operations

Implementations MUST use constant-time:
- Key comparison
- MAC verification
- Scalar multiplication

### Key Storage

Private keys should be:
- Stored encrypted at rest
- Protected by OS keychain/keystore when available
- Zeroized after use

## Custom Cipher Suites

Applications may define custom CSIDs in the range `0x10-0x17`, `0x20-0x27`, etc. (mask `0xF8`):

```python
CUSTOM_CSID = "15"  # Custom cipher suite

# Must provide:
# - Key generation
# - Key agreement
# - AEAD encryption/decryption
# - CSK serialization
```

Custom cipher suites should:
- Document all algorithms used
- Specify CSK format
- Follow security best practices
- Not conflict with reserved CSIDs

## Security Levels

| CSID | Classical Security | Quantum Security |
|------|-------------------|------------------|
| CS4a | 128-bit | None |
| CS4b | 128-bit | 192-bit |
| CS3a | 128-bit | None |

## Migration Path

For applications transitioning to post-quantum:

1. **Now**: Deploy CS4a as default, CS4b where bandwidth allows
2. **Transition**: Increase CS4b deployment as PQ threats materialize
3. **Future**: CS4a becomes legacy, CS4b (or successor) becomes default

---

*Related: [Hashname](hashname.md) | [E3X](e3x.md) | [Technical Overview](../technical-overview.md)*
