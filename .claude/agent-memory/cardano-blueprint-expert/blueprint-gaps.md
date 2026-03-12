---
name: blueprint-gaps
description: Known gaps and incomplete areas in the Cardano Blueprint — what's missing, stubs, TODOs, and where to find authoritative sources instead
type: reference
---

# Cardano Blueprint — Known Gaps & Incomplete Areas

## Summary

Blueprint is explicitly described as a "work in progress." Major sections are documented with WARNING banners. This file catalogues what IS and IS NOT covered, so the agent can direct users to authoritative fallback sources.

## Consensus Layer Gaps

### Ouroboros Praos / TPraos Chain Validity

`chainvalid.md` has explicit `TODO` for Praos and TPraos validity rules. Only BFT/PBFT validity is documented.

**Fallback**: Praos paper, TPraos (Shelley spec Section 12), cardano-ledger/ouroboros-consensus Haskell source

### Leadership Schedule

`forging.md` is missing:
- How/when leadership schedule is computed
- Stability periods for schedule publication
- Which epoch's stake distribution is used (it's 2 epochs ago)
- Exact VRF leader check formula beyond `φ_f(α_i) = 1 - (1-f)^α_i`

**Fallback**: Shelley spec, Praos paper, ouroboros-consensus source

### Multi-Era Time Handling

`multiera.md` has `## Time` and `## Forecast range` sections that are empty.

**Fallback**: ouroboros-consensus documentation, EraHistory documentation

### Ouroboros Genesis Details

`chainsel.md` has TODO sections for:
- Limit on Eagerness
- Limit on Patience
- Genesis state machine (dynamo)

**Fallback**: Ouroboros Genesis paper, cardano-node implementation

### KES, VRF, Opcerts

Not documented in Blueprint at all.

**Fallback**: Shelley spec (KES, opcerts), VRF paper (ECVRF-ED25519-SHA512-Elligator2), ouroboros-consensus

## Ledger Layer Gaps

### Transactions Concept Page

`ledger/concepts/transactions.md` is a stub: "This page is currently a stub".

### How to Read Ledger Specs

`ledger/state-transition/reading-specs.md` is a stub: "TODO This section is a stub".

### Pre-Conway Era Validation Rules

Block validation is only documented for Conway. No documentation for:
- Byron transaction rules
- Shelley transaction rules
- Allegra/Mary/Alonzo/Babbage rules

**Fallback**: Era-specific PDFs from https://github.com/IntersectMBO/cardano-ledger#cardano-ledger

### Reward Calculation

Not documented. Critical for understanding staking incentives.

**Fallback**: Shelley spec appendix, cardano-ledger source

### Deposit Tracking

Not documented.

### Treasury Mechanics

Not documented (except as a governance action target).

### Era-Specific Block/Transaction Formats

Blueprint says: "make ledger CDDLs available through blueprint directly" — marked as TODO. Currently only references external cardano-ledger CDDL files.

**Fallback**: https://github.com/IntersectMBO/cardano-ledger (search .cddl files)

### Governance Ratification

No dedicated governance documentation. CIP-1694 ratification thresholds, DRep/SPO/CC voting rules, enactment — not documented.

**Fallback**: CIP-1694, Conway formal spec

### Non-Integral Math

Planned section in ledger but not written (noted in plan.md).

### Ledger State Serialization

Planned but not written:
- Transaction and block formats (defers to external CDDL)
- Ledger state CBOR format
- Non-canonical serialization rules

**Fallback**: ouroboros-consensus snapshot format documentation, cardano-ledger CBOR instances

### Epoch Structure

"The structure of an epoch" is listed in plan.md as needed but not written.

## Network Layer Gaps

### PeerSharing Mini-Protocol

`<>` placeholder in SUMMARY.md — completely undocumented.

**Fallback**: Ouroboros network spec PDF, cardano-node source

### NTC (Node-To-Client) Protocols

Only `LocalStateQuery` has any content (and it's incomplete):
- NTC Handshake: no documentation
- LocalTxSubmission: no documentation
- TxMonitor: no documentation
- LocalChainSync: no documentation

**Fallback**: Ouroboros network spec, pallas library, cardano-node source

### LocalStateQuery — getCurrentPParams

The `getCurrentPParams` query is noted as TODO in the LocalStateQuery page.

### KeepAlive Body

`keep-alive/README.md` says "TODO: fill this section".

### Multiplexer Multi-Segment Delimitation

Blueprint notes as unclear: "How are multi-segment messages delimited? - there is no 'start of message' flag or 'N of M' counter".

### Handshake MsgReplyVersion

`MsgReplyVersion` is no longer in the CDDL but documented in the state machine — Blueprint notes this as inconsistency.

## Storage Layer Gaps

### ChainDB Immutable Format

`cardano-node-chaindb/README.md` says "TODO explain NestedCtxt, Header, Block, the Hard Fork Block, chunks, primary and secondary indices."

### Volatile Database Format

"TODO describe the format in which the blocks are stored" for volatile DB.

### Ledger State Snapshot Backend

Documentation of V2InMemory (`tvar` file) and V1LMDB (`data.mdb`) backends is present but incomplete.

## Serialization / CDDL Gaps

### Tag 258 (CBOR Sets)

Not documented in Blueprint. Tag 258 is used extensively for sets in Cardano CBOR (pool IDs, witnesses, etc.). Elements must be sorted for canonical encoding.

### Canonical Encoding Rules

Not documented. Important for deterministic hash computation.

### Protocol Parameter Integer Key Encoding

N2C PParams use integer keys 0-33 (not strings). Not documented.

### BigNum / Rational Encoding

Only `rational = [int, int]` defined. No documentation of bignum formats used in various protocol fields.

## Plutus / Smart Contract Gaps

### Script Execution Cost Model

`plutus/cek.md` covers machine semantics but not the cost model (how CPU/memory steps are counted and bounded).

### Plutus Version Compatibility

No documentation of which Plutus language version maps to which ledger language version.

### Script Context Format

No documentation of the `ScriptContext` format passed to scripts (important for implementing Phase-2 validation).

## Mempool Gaps

### Fairness

`mempool/README.md` has "TODO: describe fairness and what mempools should do".

### Back-Pressure Details

Mentioned conceptually but not specified in detail.

## What to Use Instead

| Topic | Authoritative Source |
|---|---|
| Formal ledger rules | https://intersectmbo.github.io/formal-ledger-specifications/ |
| Era PDFs (Shelley, Alonzo, etc.) | https://github.com/IntersectMBO/cardano-ledger#cardano-ledger |
| Conway spec | https://intersectmbo.github.io/formal-ledger-specifications/conway-ledger.pdf |
| Era CDDL schemas | https://github.com/IntersectMBO/cardano-ledger (search .cddl) |
| Consensus implementation | https://ouroboros-consensus.cardano.intersectmbo.org |
| Network spec | https://ouroboros-network.cardano.intersectmbo.org/pdfs/network-spec/network-spec.pdf |
| CIP-1694 Governance | https://github.com/cardano-foundation/CIPs/blob/master/CIP-1694/README.md |
| Praos paper | https://iohk.io/en/research/library/papers/ouroboros-praos-an-adaptively-secure-semi-synchronous-proof-of-stake-protocol/ |
| VRF (ECVRF) | RFC 9381 / draft-irtf-cfrg-vrf |
| KES | Haskell implementation in pallas-crypto |
| Plutus CEK spec | https://plutus.cardano.intersectmbo.org/resources/plutus-core-spec.pdf |
| Plutus builtins | https://plutus.cardano.intersectmbo.org/haddock/latest/plutus-core/ |
| Reference Haskell implementation | https://github.com/IntersectMBO/cardano-node |
