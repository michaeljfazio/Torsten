---
name: blueprint-serialization
description: Cardano Blueprint wire format and CDDL documentation — CBOR encoding patterns, era dispatch, tag 24 (CBOR-in-CBOR), base types, cddlc tool
type: reference
---

# Cardano Blueprint — Serialization & Wire Formats

## Source Files

- `src/codecs/README.md` — CBOR/CDDL overview, cddlc tool, import patterns
- `src/codecs/base.cddl` — Base type definitions
- `src/network/node-to-node/*/messages.cddl` — Per-protocol message schemas
- `src/network/node-to-node/chainsync/header.cddl` — Block header era dispatch
- `src/network/node-to-node/blockfetch/block.cddl` — Block era dispatch
- `src/network/node-to-node/txsubmission2/tx.cddl` — Transaction era dispatch
- `src/network/node-to-node/txsubmission2/txId.cddl` — TxId era dispatch
- `src/client/node-to-client/state-query/messages.cddl` — LocalStateQuery
- `src/client/node-to-client/state-query/getSystemStart.cddl` — GetSystemStart

## Encoding Technology

- **CBOR** — Concise Binary Object Representation; compact binary encoding of JSON-like data
- **CDDL** — Concise Data Definition Language (RFC 8610); used to express schemas for CBOR
- Tool for combining modular CDDL: `cddlc` per RFC draft: https://datatracker.ietf.org/doc/draft-ietf-cbor-cddl-modules/

## CDDL Import Directives

Blueprint uses custom directives in CDDL files:
```cddl
;# import base as base        ; import qualified, access via base.X
;# include byron as byron      ; inline era CDDL qualified as byron.X
```

Ledger CDDLs are **self-contained per era** with repeated basic types.

## Base Type Definitions (base.cddl)

```cddl
blockNo      = word64
epochNo      = word64
slotNo       = word64
coin         = word64
rational     = [int, int]
keyhash      = bstr .size 28    ; 28-byte key hash
hash         = bstr .size 32    ; 32-byte hash
relativeTime = int

word8  = uint .size 1    ; 1 byte unsigned
word16 = uint .size 2    ; 2 bytes unsigned
word32 = uint .size 4    ; 4 bytes unsigned
word64 = uint .size 8    ; 8 bytes unsigned
```

## Era Dispatch Patterns

### ns7 — Namespace with era tag prefix

Used for: block bodies (BlockFetch), transactions (TxSubmission2), TxIds, block headers (ChainSync)

```cddl
ns7<byron, shelley, allegra, mary, alonzo, babbage, conway>
  = [6, conway]    ; Conway (current)
  / [5, babbage]
  / [4, alonzo]
  / [3, mary]
  / [2, allegra]
  / [1, shelley]
  / [0, byron]
```

Byron is tag `0` for most things, but note Byron has special cases:
- Block: tag `0` = regular block, tag `1` = EBB (Epoch Boundary Block)
- TxId: `[0, txid]`, `[1, certificateid]`, `[2, updid]`, `[3, voteid]`

### telescope7 — Historical encoding

Used for era transitions where past eras are nested.

```cddl
telescope7<byron, shelley, allegra, mary, alonzo, babbage, conway>
  = [pastEra, pastEra, pastEra, pastEra, pastEra, pastEra, currentEra<conway>]
  / [pastEra, pastEra, pastEra, pastEra, pastEra, currentEra<babbage>]
  / [pastEra, pastEra, pastEra, pastEra, currentEra<alonzo>]
  / [pastEra, pastEra, pastEra, currentEra<mary>]
  / [pastEra, pastEra, currentEra<allegra>]
  / [pastEra, currentEra<shelley>]
  / [currentEra<byron>]
```

## CBOR Tag 24 — CBOR-in-CBOR

Shelley+ headers and blocks are wrapped in CBOR tag 24 (embedded CBOR):
```cddl
serialisedShelleyHeader<era> = #6.24(bytes .cbor era)
serialisedShelleyTx<era>     = #6.24(bytes .cbor era)
serialisedCardanoBlock        = #6.24(bytes .cbor cardanoBlock)
```

This means: take era-specific CBOR bytes, wrap in a `bstr`, then tag with 24.

## Block Wire Format

Full block as sent over BlockFetch:
```cddl
serialisedCardanoBlock = #6.24(bytes .cbor cardanoBlock)

cardanoBlock = byron.block           ; no wrapper array for Byron
             / [2, shelley.block]
             / [3, allegra.block]
             / [4, mary.block]
             / [5, alonzo.block]
             / [6, babbage.block]
             / [7, conway.block]
```

## Header Wire Format

Headers sent over ChainSync:
```cddl
header = base.ns7<byronHeader,
                  serialisedShelleyHeader<shelley.header>,
                  serialisedShelleyHeader<allegra.header>,
                  serialisedShelleyHeader<mary.header>,
                  serialisedShelleyHeader<alonzo.header>,
                  serialisedShelleyHeader<babbage.header>,
                  serialisedShelleyHeader<conway.header>>

byronHeader = [byronRegularIdx, #6.24(bytes .cbor byron.blockhead)]
            / [byronBoundaryIdx, #6.24(bytes .cbor byron.ebbhead)]

byronBoundaryIdx = [0, base.word32]  ; EBB, word32 = epoch number
byronRegularIdx  = [1, base.word32]  ; Regular block
```

## LocalStateQuery Wire Format

```cddl
localStateQueryMessage = msgAcquire / msgAcquired / msgFailure / msgQuery /
                         msgResult / msgRelease / msgReAcquire / lsqMsgDone

msgAcquire   = [0, point]    ; acquire specific point
             / [8]            ; acquire volatile tip (no point)
             / [10]           ; volatile tip (newer variant)
msgAcquired  = [1]
msgFailure   = [2, failure]
msgQuery     = [3, query]    ; query is opaque 'any' at this level
msgResult    = [4, result]   ; result is opaque 'any' at this level
msgRelease   = [5]
msgReAcquire = [6, point]
             / [9]
             / [11]
lsqMsgDone   = [7]

failure = 0    ; acquireFailurePointTooOld
        / 1    ; acquireFailurePointNotOnChain
```

## UTCTime Encoding (SystemStart)

NOT standard CBOR/POSIX timestamp. Uses Haskell `ToCBOR UTCTime` encoding:
```cddl
time = [year, dayOfYear, timeOfDayPico]
year         = bigint
dayOfYear    = int
timeOfDayPico = bigint
```

Blueprint marks this as incorrect from CDDL spec perspective but matches actual wire format.

## LocalStateQuery — GetSystemStart

```cddl
; query.cddl
query = 1    ; tag for GetSystemStart

; result.cddl
result = [year, dayOfYear, timeOfDayPico]
```

## Ledger CDDL Files

Not hosted directly in Blueprint (noted as TODO), but referenced at:
- https://github.com/IntersectMBO/cardano-ledger (search path:.cddl)
- Conway era: https://github.com/IntersectMBO/cardano-ledger/blob/master/eras/conway/impl/cddl-files/conway.cddl

## Known Encoding Rules (from other sources, relevant to Blueprint context)

- **Tag 258** for sets: CBOR canonical encoding of unordered sets (pool IDs, owners) — elements MUST be sorted
- **Tag 24** for embedded CBOR: All Shelley+ headers and blocks use this when transmitted over network
- **CBOR sets vs arrays**: Many Shelley+ fields use tag 258 encoded sets rather than plain arrays
- **Protocol parameter encoding**: N2C PParams use integer keys 0-33 (not JSON strings)
- **Indefinite-length lists**: TxSubmission2 `txIdList` and `txList` use indefinite encoding; `txIdsAndSizes` uses definite

## CDDL Tool Usage

Blueprint uses `cddlc` for composing modular CDDL. The `{{#include file.cddl}}` syntax in markdown is mdBook's include directive — actual CDDL content is in the `.cddl` files.

When reading Blueprint CDDL files:
- `;# import base as base` — use `base.fieldname` to reference
- `;# include byron as byron` — full inline of era CDDL as `byron.typename`
- `#6.N(type)` — CBOR tag N wrapping type
- `bstr .size N` — byte string of exactly N bytes

## Known Gaps in Blueprint

- Tag 258 (CBOR sets) not documented
- Canonical encoding rules not documented
- Era-specific CDDL not hosted (only referenced externally)
- Protocol parameter CBOR encoding not documented
- Ledger state snapshot format not documented (CBOR format for state/UTxO)
- No documentation on BigNums, rational encoding beyond [int, int]
