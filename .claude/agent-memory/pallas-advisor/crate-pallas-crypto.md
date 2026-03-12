---
name: crate-pallas-crypto
description: Full API surface of pallas-crypto including Ed25519, KES Sum6Kes, Blake2b, VRF, memsec, and nonce modules
type: reference
---

# pallas-crypto (v1.0.0-alpha.5)

## Overview

Description: "Shared Cryptographic primitives". Apache-2.0. Provides all cryptographic primitives used by the Cardano protocol.

## Module Structure

```
pallas_crypto::
  hash::     // Blake2b hashing: Hash<N>, Hasher<N>
  kes::      // Key Evolving Signatures: Sum1Kes–Sum7Kes, Sum1CompactKes–Sum7CompactKes
  key::      // Ed25519 key management
  memsec::   // Memory security (secure zeroization)
  nonce::    // Nonce generation/management
```

## Features

- Default: no features (hash + key modules available)
- `kes`: enables the KES module (required for Sum6Kes)
- `relaxed`: enables relaxed mode (accessed via pallas-primitives)

## Hash Module (`pallas_crypto::hash`)

```rust
pub struct Hash<const DIGEST_SIZE: usize>([u8; DIGEST_SIZE]);
// Type aliases commonly used:
// Hash<28> = Blake2b-224 (policy IDs, pool IDs, address key hashes)
// Hash<32> = Blake2b-256 (block hashes, tx hashes)

pub struct Hasher<const DIGEST_SIZE: usize> { ... }
// Hasher::<224>::new() — 28-byte Blake2b
// Hasher::<256>::new() — 32-byte Blake2b
// Methods:
//   .input(&[u8])  — feed bytes
//   .finalize()    — get Hash<N>
```

Key design: handles CBOR encoding requirements for Cardano without extra allocations.

## KES Module (`pallas_crypto::kes`)

**Feature flag required**: `kes`

### Available KES implementations:

```rust
// Standard (stores intermediate verification keys in signature)
pub struct Sum1Kes  ... Sum7Kes
pub struct Sum1KesSig ... Sum7KesSig

// Compact (reconstructs intermediate VKs from signature data)
pub struct Sum1CompactKes ... Sum7CompactKes
```

Cardano mainnet uses **Sum6Kes** (depth 6, supports up to 2^6 - 1 = 63 key evolutions).

### Key Size

`Sum6Kes::SIZE + 4` bytes total buffer (SIZE bytes for key material + 4 bytes for period counter)

From memory notes: key buffer = 612 bytes total (608 + 4 byte period counter)
`INDIVIDUAL_SECRET_SIZE + 6×32 + 6×64` bytes = key hierarchy for depth 6

### Traits

```rust
pub trait KesSk<'a>: Sized {
    fn keygen(key_buffer: &'a mut [u8], seed: &[u8]) -> (Self, PublicKey);
    fn sign(&self, message: &[u8]) -> Self::Sig;
    fn update(&mut self) -> Result<(), Error>;
    fn get_period(&self) -> u32;
    fn from_bytes(bytes: &'a mut [u8]) -> Result<Self, Error>;
    fn as_bytes(&self) -> &[u8];
    type Sig: KesSig;
}

pub trait KesSig: Sized {
    fn verify(&self, period: u32, public_key: &PublicKey, message: &[u8]) -> Result<(), Error>;
}
```

### Torsten Usage (from torsten-crypto/src/kes.rs)

```rust
use pallas_crypto::kes::common::PublicKey as KesPublicKey;
use pallas_crypto::kes::errors::Error as PallasKesError;
use pallas_crypto::kes::summed_kes::{Sum6Kes, Sum6KesSig};
use pallas_crypto::kes::traits::{KesSig, KesSk};
```

**Critical note**: Sum6Kes::Drop zeroizes the key buffer. Torsten must copy bytes before any drop. This is a memory safety feature but requires careful lifecycle management.

## Key Module (`pallas_crypto::key`)

Provides Ed25519 key management. Specific API not deeply researched — likely:
- Ed25519 private key generation
- Public key derivation
- Signing
- Verification

## Memsec Module (`pallas_crypto::memsec`)

Memory security utilities for sensitive cryptographic material:
- Secure zeroization on drop
- Protection against compiler optimization eliding zeroing

## Nonce Module (`pallas_crypto::nonce`)

Nonce generation and management. Used in Cardano consensus for epoch nonce calculation.

## What Torsten Uses

torsten-crypto wraps pallas-crypto for:
1. **KES**: `kes_keygen`, `kes_sign_message`, `kes_verify`, `kes_update`, `kes_sk_to_pk` — all via Sum6Kes
2. **Hashing**: Via pallas-crypto Hash<N> types (Hash32, Hash28 in torsten-primitives map to these)

## What Torsten Does NOT Use from pallas-crypto

1. **VRF**: Torsten uses `vrf_dalek` crate directly for ECVRF-ED25519-SHA512-Elligator2
2. **CompactKes**: Torsten uses Sum6Kes (standard), not the compact variant
3. **Memsec directly**: May be used transitively through KES operations

## Known Issues / Caveats

1. **28-byte hash padding bug**: Pallas Hash<28> types (DRep keys, pool voter keys, required signers) cannot be directly converted to Hash<32> via `Hash<32>::from()`. Torsten must pad these manually. This is a common source of bugs when working with mixed hash sizes.
2. **KES buffer lifecycle**: Sum6Kes zeroizes on drop — torsten must copy bytes before drop when serializing/deserializing KES keys.
3. **`kes` feature flag**: Must explicitly enable in Cargo.toml: `pallas-crypto = { version = "...", features = ["kes"] }`

## Gaps in pallas-crypto

1. **No native VRF**: pallas-crypto does not include VRF (ECVRF-ED25519-SHA512-Elligator2). Torsten uses `vrf_dalek` for this.
2. **No BIP-32 HD derivation**: For wallet key derivation (but torsten-node doesn't need this)
3. **No bech32 key encoding**: Separate pallas-bech32 crate handles this
