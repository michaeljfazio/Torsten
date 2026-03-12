---
name: blueprint-storage
description: Cardano Blueprint storage layer documentation — ChainDB structure, immutable/volatile/ledger directories, storage requirements
type: reference
---

# Cardano Blueprint — Storage Layer

## Source Files

- `src/storage/README.md` — Storage requirements, chain diffusion
- `src/storage/cardano-node-chaindb/README.md` — ChainDB format (immutable/volatile/ledger)

## Storage Requirements

Any storage system for Cardano must provide:
1. **Fast sequential access to immutable blocks** — syncing peers request historical blocks sequentially
2. **Fast sequential access to current selection blocks** — peers request during sync and when caught-up
3. **Fast rollback in volatile part** — switch to alternative chain quickly
4. **Fast identification of volatile chains** — even with out-of-order block arrival
5. **Fast restart** — avoid full chain replay, while supporting k-deep forks

NOTE: Storage does NOT need to provide Durability (ACID D) — upstream peers can replace lost blocks.

## Volatile vs Immutable Split

Based on the `k` security parameter (k = 2160 on mainnet):
- **Immutable**: blocks at depth > k from tip — guaranteed never to change
- **Volatile**: k blocks from tip — may be rolled back

```
Storage:
├── Immutable (cold, efficient sequential storage)
│   └── Sequential block chain
├── Volatile (hot, efficient rollback)
│   └── Current selection blocks + candidate blocks
└── Recent Ledger States (hot, efficient rollback)
    └── State snapshots for k volatile blocks
```

## cardano-node ChainDB Directory Structure

```
db/
├── immutable/
│   ├── 00000.chunk      ; blocks stored in 21600-slot chunks
│   ├── 00000.primary    ; primary index file
│   ├── 00000.secondary  ; secondary index (56 bytes/entry, no header, big-endian)
│   └── ...
├── ledger/
│   └── 164021355/       ; snapshot at slot 164021355
│       ├── state        ; CBOR-encoded LedgerState (except UTxO)
│       ├── tables/
│       │   ├── tvar     ; V2InMemory: CBOR-encoded UTxO set
│       │   └── data.mdb ; V1LMDB: LMDB database with UTxO set
│       └── meta         ; Backend identifier + checksum
└── volatile/
    ├── blocks-0.dat     ; Volatile blocks
    └── ...
```

## Immutable Database Details

Blocks are stored in numbered chunk files. Each chunk covers 21,600 slots (one epoch in Shelley+ era).

**Index files**:
- `primary`: Maps slot numbers to offsets within the chunk
- `secondary`: 56-byte entries, no header, big-endian encoding, includes CRC32 checksums for block integrity

**Chunk files**: Sequential binary blocks, read via memory-mapped I/O.

Blueprint says: TODO for explaining NestedCtxt, Header, Block, the Hard Fork Block format.

## Volatile Database Details

Contains:
1. All blocks in current selection (beyond immutable tip)
2. Other blocks (connected or disconnected) with slot > k-th block from tip

These blocks can form the current selection or a fork less than k blocks deep.

Blueprint says: TODO for volatile block storage format.

## Ledger State Snapshots

Stored at immutable block boundaries. Named after slot number.

**Components**:
- `state` file: CBOR-encoded LedgerState (all except UTxO)
- `tables/` directory: UTxO set (backend-dependent)
- `meta` file: backend identifier + checksum for consistency

**Backends**:
| Backend | tables/ content | Format |
|---|---|---|
| V2InMemory | `tvar` file | CBOR-encoded UTxO set |
| V1LMDB | `data.mdb` | LMDB database |

**Snapshot naming**: `<slotno>/` — may not be most recent immutable block (snapshots are periodic due to cost).

**Backend conversion**: Available via `snapshot-converter` tool in ouroboros-consensus.

## Chain Diffusion

Chain diffusion is a joint responsibility of Consensus, Network, and Storage layers:
- Storage provides data to serve to peers
- ChainSync (headers) and BlockFetch (bodies) serve data from storage
- Consensus decides how storage is mutated (via chain selection)

Mathematical model assumes instantaneous transmission — real-world uses block-by-block streaming with Ouroboros Genesis providing safety during initial sync.

## Mithril Integration

ChainDB format (immutable directory structure) is the de-facto standard for chain distribution:
- Mithril signs this directory structure
- Snapshot archives are distributed as `tar.zst` containing immutable/ directory contents
- Enables fast sync from trusted checkpoint

## Known Gaps in Blueprint

- Chunk file internal format not documented (NestedCtxt, Hard Fork Block wrapping)
- Volatile database file format not documented
- Ledger state CBOR schema not documented
- No documentation of CRC32 verification details in secondary index
- No documentation of snapshot consistency checking
