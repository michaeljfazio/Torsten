---
name: blueprint-overview
description: Cardano Blueprint project structure, scope, organization, and key metadata
type: reference
---

# Cardano Blueprint — Project Overview

## Repository & Publication

- **GitHub**: https://github.com/cardano-scaling/cardano-blueprint
- **Published docs**: https://cardano-scaling.github.io/cardano-blueprint
- **Branch**: `main`
- **Last updated**: 2026-03-07 (as of research date 2026-03-13)
- **License**: Apache 2.0
- **Build system**: mdBook with plugins: `mdbook-katex`, `mdbook-mermaid`, `mdbook-alerts`, `mdbook-toc`
- **Lead**: Sebastian Nagel (@ch1bo, Input Output Global)

## Project Mission

Implementation-independent documentation, specifications, diagrams, CDDL schemas, and test data enabling a wide developer audience to understand and build Cardano components. Aims to support node diversity and security through shared knowledge.

## Repository Structure

```
cardano-blueprint/
├── src/
│   ├── SUMMARY.md               # mdBook table of contents
│   ├── introduction/README.md   # Project intro and goals
│   ├── principles/README.md     # Design principles (e.g., worst-case optimization)
│   ├── network/                 # Network layer docs + CDDL
│   │   ├── README.md            # Network overview, N2N protocol version v14
│   │   ├── multiplexing.md      # Mux packet format
│   │   ├── mini-protocols.md    # State machines overview
│   │   └── node-to-node/
│   │       ├── handshake/       # README.md + messages.cddl + test-data/
│   │       ├── chainsync/       # README.md + messages.cddl + header.cddl
│   │       ├── blockfetch/      # README.md + messages.cddl + block.cddl
│   │       ├── txsubmission2/   # README.md + messages.cddl + tx.cddl + txId.cddl
│   │       └── keep-alive/      # README.md + messages.cddl
│   ├── consensus/               # Consensus layer docs
│   │   ├── README.md            # Protocol overview, era-to-protocol table
│   │   ├── chainvalid.md        # Chain validity rules
│   │   ├── chainsel.md          # Chain selection rules, k parameter, Genesis
│   │   ├── forging.md           # Block forging, leadership check
│   │   └── multiera.md          # Multi-era handling strategies
│   ├── storage/                 # Storage layer docs
│   │   ├── README.md            # Storage requirements, chain diffusion
│   │   └── cardano-node-chaindb/ # immutable/volatile/ledger directory format
│   ├── mempool/README.md        # Mempool requirements and behavior
│   ├── ledger/                  # Ledger rules docs
│   │   ├── README.md            # Pointers to formal specs, CDDL, conformance tests
│   │   ├── plan.md              # Future documentation plan (roadmap)
│   │   ├── block-validation.md  # Conway block validation (BBODY/LEDGERS/CERTS/GOV/UTXOW mermaid diagrams)
│   │   ├── transaction-fee.md   # Fee calculation algorithm with worked example
│   │   ├── state-transition.md  # Env/State/Signal model
│   │   ├── concepts/
│   │   │   ├── blocks.md        # Block structure, header/body split
│   │   │   ├── transactions.md  # STUB
│   │   │   └── determinism.md   # Transaction and script determinism
│   │   ├── state-transition/
│   │   │   ├── validity.md      # Multi-phase validity, static vs dynamic checks
│   │   │   └── reading-specs.md # STUB
│   │   ├── conformance-test-vectors/
│   │   │   ├── README.md        # Test vector description
│   │   │   └── vectors.tar.gz   # Conway era ledger test vectors (binary)
│   │   └── constraints/
│   │       └── header-body-split.md  # Header/body split implications, 3k/f forecast window
│   ├── plutus/                  # Plutus/UPLC documentation
│   │   ├── README.md            # Overview, resources
│   │   ├── syntax.md            # UPLC concrete syntax, version numbers, de Bruijn
│   │   ├── builtin.md           # Built-in types and functions
│   │   ├── cek.md               # CEK machine operational semantics
│   │   └── serialization.md     # UPLC serialization
│   ├── client/                  # Client interface docs
│   │   ├── README.md            # NTC and UTxO-RPC overview, implementation table
│   │   ├── node-to-client/
│   │   │   ├── README.md        # NTC mini-protocols overview
│   │   │   └── state-query/
│   │   │       ├── README.md    # LocalStateQuery protocol, getSystemStart query
│   │   │       ├── messages.cddl
│   │   │       ├── getSystemStart.cddl
│   │   │       └── examples/getSystemStart/  # query.cbor + result.cbor
│   │   └── utxo-rpc/README.md   # Pointer to utxorpc.org
│   ├── codecs/
│   │   ├── README.md            # CBOR/CDDL explanation, cddlc tool, import patterns
│   │   └── base.cddl            # Base type definitions
│   ├── styleguide.md
│   ├── logbook.md               # Project diary/decisions log
│   ├── CONTRIBUTING.md
│   └── logo/
```

## Key Values / Principles

- Accessible, Open, Minimal, Lightweight, Evidence-based
- Markdown + mdBook rendered site with Mermaid diagrams
- CDDL for wire format schemas, test data for validation
- Community-owned complement to CIP process

## Current State (as of 2026-03-13)

- **Well-documented**: Network protocols (mini-protocols, CDDL, state machines), consensus chain selection/validity, Conway block validation (detailed mermaid diagrams), transaction fees, Plutus/CEK, ledger validity
- **In progress**: Consensus forging (leadership schedule), storage ChainDB format, mempool
- **Stubs/TODO**: Peer Sharing mini-protocol, NTC Handshake/TxSubmission/TxMonitor/LocalChainSync, ledger transactions concept, reading specs, KeepAlive body

## Known Implementations Referenced

| Implementation | Language | NTC | UTxO-RPC |
|---|---|---|---|
| cardano-node | Haskell | Yes | No |
| dingo | Go | Yes | Yes |
| amaru | Rust | Unclear | Unclear |

Client implementations: cardano-cli, pallas, gouroboros

## External Resources Referenced

- Formal ledger spec: https://intersectmbo.github.io/formal-ledger-specifications/
- Ouroboros consensus report: https://ouroboros-consensus.cardano.intersectmbo.org/pdfs/report.pdf
- Network design report: https://ouroboros-network.cardano.intersectmbo.org/pdfs/network-design/network-design.pdf
- Network spec: https://ouroboros-network.cardano.intersectmbo.org/pdfs/network-spec/network-spec.pdf
- Era CDDL files: https://github.com/IntersectMBO/cardano-ledger (search .cddl files)
- CIP-0059 feature table: https://github.com/cardano-foundation/CIPs/blob/master/CIP-0059/feature-table.md
