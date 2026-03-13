---
name: blueprint-network
description: Cardano Blueprint network protocol documentation — multiplexer, all mini-protocols (Handshake, ChainSync, BlockFetch, TxSubmission2, KeepAlive), CDDL, state machines
type: reference
---

# Cardano Blueprint — Network Layer

## Source Files

- `src/network/README.md` — Network overview, N2N version list
- `src/network/multiplexing.md` — Mux packet format
- `src/network/mini-protocols.md` — State machine conventions
- `src/network/node-to-node/handshake/` — README + messages.cddl + test-data/
- `src/network/node-to-node/chainsync/` — README + messages.cddl + header.cddl
- `src/network/node-to-node/blockfetch/` — README + messages.cddl + block.cddl
- `src/network/node-to-node/txsubmission2/` — README + messages.cddl + tx.cddl + txId.cddl
- `src/network/node-to-node/keep-alive/` — README + messages.cddl

## Current Protocol Versions

- **N2N version**: v14 (Blueprint documents v13/v14)
- **N2C version**: V19 (as documented in LocalStateQuery)

## Multiplexer

### Packet Format

```
┌─────────────────────────────────────────────────────────────┐
│ Transmission time (32-bit, µsec, lowest 32 bits, big-endian)│
├─┬──────────────────────────────┬───────────────────────────┤
│M│  Mini-protocol ID (15 bits)  │  Payload length N (16 bit)│
├─┴──────────────────────────────┴───────────────────────────┤
│                 Payload (variable, N bytes)                  │
└─────────────────────────────────────────────────────────────┘
```

| Field | Size | Meaning |
|---|---|---|
| Transmission time | 32 bits | Monotonic timestamp (µsec, lowest 32 bits) |
| M | 1 bit | Mode: 0 = from initiator, 1 = from responder |
| Mini-protocol ID | 15 bits | Protocol identifier |
| Payload length | 16 bits | Segment payload length N in bytes |
| Payload | N bytes | Raw payload data |

All fields: **big-endian** byte order.

Note: Multi-segment messages are not explicitly delimited — Blueprint notes this as unclear/TODO.
Maximum segment size: 65,535 bytes.

### Message Size Limit for Handshake

Handshake messages must not be split into segments (operates before mux fully set up): **5,760 bytes** maximum.

## Mini-Protocol State Machines

Convention:
- **Initiator** (client who opened connection) has agency first
- State names prefixed with `St`
- Message names prefixed with `Msg`
- Initiator agency: green; Responder agency: blue/underlined

## Handshake Protocol

**Mini-protocol number: 0**

Used to establish connection and negotiate protocol versions/parameters. Two variants: NTN and NTC (differ only in protocol parameters).

### State Machine

```
StPropose (initiator) → MsgProposeVersions → StConfirm
StConfirm (responder) → MsgAcceptVersion → End
StConfirm (responder) → MsgReplyVersion → End
StConfirm (responder) → MsgRefuse → End
```

### TCP Simultaneous Open

If both sides connect simultaneously, both send `MsgProposeVersions`. The received one in `StConfirm` is treated as `MsgReplyVersion` (same CBOR encoding).

### Timeouts

- Max wait in StPropose (for responder): **10 seconds**
- Max wait in StConfirm (for initiator): **10 seconds**

### NTN Handshake CDDL (v13/v14)

```cddl
handshakeMessage = msgProposeVersions / msgAcceptVersion / msgRefuse / msgQueryReply

msgProposeVersions = [0, versionTable]
msgAcceptVersion   = [1, versionNumber, nodeToNodeVersionData]
msgRefuse          = [2, refuseReason]
msgQueryReply      = [3, versionTable]

versionTable = { * versionNumber => nodeToNodeVersionData }
versionNumber = 13 / 14

nodeToNodeVersionData = [networkMagic, initiatorOnlyDiffusionMode, peerSharing, query]
networkMagic = 0..4294967295
initiatorOnlyDiffusionMode = bool
peerSharing = 0..1   ; 0 or 1
query = bool

refuseReason = refuseReasonVersionMismatch
             / refuseReasonHandshakeDecodeError
             / refuseReasonRefused

refuseReasonVersionMismatch      = [0, [*versionNumber]]
refuseReasonHandshakeDecodeError = [1, versionNumber, tstr]
refuseReasonRefused              = [2, versionNumber, tstr]
```

### Handshake Test Data

5 test cases in `src/network/node-to-node/handshake/test-data/test-0` through `test-4`. Each is a binary CBOR file (base64 encoded in repo).

## ChainSync Protocol

**Mini-protocol number: 2**

Pull-based protocol for transmitting chains of block headers.

### State Machine

```
StIdle (initiator) → MsgRequestNext → StCanAwait
StIdle (initiator) → MsgFindIntersect([point]) → StIntersect
StIdle (initiator) → MsgDone → End
StCanAwait (responder) → MsgAwaitReply → StMustReply
StCanAwait (responder) → MsgRollForward(header, tip) → StIdle
StCanAwait (responder) → MsgRollBackward(point_old, tip) → StIdle
StMustReply (responder) → MsgRollForward(header, tip) → StIdle
StMustReply (responder) → MsgRollBackward(point_old, tip) → StIdle
StIntersect (responder) → MsgIntersectFound(point_intersect, tip) → StIdle
StIntersect (responder) → MsgIntersectNotFound(tip) → StIdle
```

### ChainSync Pipelining (Tentative Diffusion)

Server can transmit one **tentative** header on top of selected chain before full validation. Allows parallel header diffusion and block downloading. Header invalidity does NOT terminate connection.

Constraints:
- Only **one** pipelined header at a time (must be on top of current selection)
- If server finds pipelined block invalid, should promptly announce to clients

### Access Patterns

- Immutable part: simple sequential iterator
- Volatile part: iterator must follow rollbacks
- Blocks that become immutable may move from volatile to immutable storage — implementation must handle

### CDDL

```cddl
chainSyncMessage = msgRequestNext / msgAwaitReply / msgRollForward /
                   msgRollBackward / msgFindIntersect / msgIntersectFound /
                   msgIntersectNotFound / chainSyncMsgDone

msgRequestNext       = [0]
msgAwaitReply        = [1]
msgRollForward       = [2, header.header, tip]
msgRollBackward      = [3, point, tip]
msgFindIntersect     = [4, [* point]]
msgIntersectFound    = [5, point, tip]
msgIntersectNotFound = [6, tip]
chainSyncMsgDone     = [7]

tip = [point, base.blockNo]

point = []                          ; genesis point
      / [base.slotNo, base.hash]
```

### Header CDDL

Header is a tagged/era-dispatched value:

```cddl
header = base.ns7<byronHeader,
                  serialisedShelleyHeader<shelley.header>,
                  ...same for allegra/mary/alonzo/babbage/conway>

byronHeader = [byronRegularIdx, #6.24(bytes .cbor byron.blockhead)]
            / [byronBoundaryIdx, #6.24(bytes .cbor byron.ebbhead)]

byronBoundaryIdx = [0, base.word32]
byronRegularIdx  = [1, base.word32]

serialisedShelleyHeader<era> = #6.24(bytes .cbor era)
```

Shelley+ headers are CBOR-in-CBOR (tag 24).

## BlockFetch Protocol

**Mini-protocol number: 3**

Pull-based protocol for fetching block bodies. Central BlockFetch decision component minimizes bandwidth by fetching each block from only one peer.

### Misbehavior (causes connection termination)

- State machine violation
- Server sends blocks not requested
- Block doesn't match announced header
- Valid header but invalid body

### State Machine

```
StIdle (initiator) → MsgClientDone → End
StIdle (initiator) → MsgRequestRange(point, point) → StBusy
StBusy (responder) → MsgNoBlocks → StIdle
StBusy (responder) → MsgStartBatch → StStreaming
StStreaming (responder) → MsgBlock(body) → StStreaming
StStreaming (responder) → MsgBatchDone → StIdle
```

### CDDL

```cddl
blockFetchMessage = msgRequestRange / msgClientDone / msgStartBatch /
                    msgNoBlocks / msgBlock / msgBatchDone

msgRequestRange = [0, point, point]
msgClientDone   = [1]
msgStartBatch   = [2]
msgNoBlocks     = [3]
msgBlock        = [4, block.block]
msgBatchDone    = [5]
```

### Block CDDL (era dispatch)

```cddl
serialisedCardanoBlock = #6.24(bytes .cbor cardanoBlock)

cardanoBlock = byron.block           ; no era tag for Byron
             / [2, shelley.block]
             / [3, allegra.block]
             / [4, mary.block]
             / [5, alonzo.block]
             / [6, babbage.block]
             / [7, conway.block]
```

Note: Byron takes tags 0 (regular) and 1 (EBB). Shelley starts at 2.

## TxSubmission2 Protocol

**Mini-protocol number: 4**

Pull-based protocol for diffusing pending transactions. IMPORTANT: transactions flow in the OPPOSITE direction from blocks. The "initiator" (client) GIVES transactions to the "responder" (server).

### State Machine

```
StInit (initiator) → MsgInit → StIdle
StIdle (responder) → MsgRequestTxIdsNonBlocking(ack, req) → StTxIdsNonBlocking
StIdle (responder) → MsgRequestTxIdsBlocking(ack, req) → StTxIdsBlocking
StTxIdsNonBlocking (initiator) → MsgReplyTxIds([(id,size)]) → StIdle
StTxIdsBlocking (initiator) → MsgReplyTxIds([(id,size)]) → StIdle
StTxIdsBlocking (initiator) → MsgDone → End
StIdle (responder) → MsgRequestTxs([id]) → StTxs
StTxs (initiator) → MsgReplyTxs([tx]) → StIdle
```

### Misbehavior

- State machine violation
- Too many/few transactions sent or acknowledged
- Requesting zero transactions
- Requesting transaction not announced

### CDDL

```cddl
txSubmission2Message = msgInit / msgRequestTxIds / msgReplyTxIds /
                       msgRequestTxs / msgReplyTxs / tsMsgDone

msgInit         = [6]
msgRequestTxIds = [0, tsBlocking, txCount, txCount]
msgReplyTxIds   = [1, txIdsAndSizes]    ; definite-length list
msgRequestTxs   = [2, txIdList]          ; indefinite-length list
msgReplyTxs     = [3, txList]            ; indefinite-length list
tsMsgDone       = [4]

tsBlocking      = false / true
txCount         = base.word16
txIdAndSize     = [base.txId, txSizeInBytes]
txIdsAndSizes   = [*txIdAndSize]
txIdList        = [*txId.txId]
txList          = [*tx.tx]
txSizeInBytes   = base.word32
```

Important: `txIdList` and `txList` use **indefinite-length** lists; `txIdsAndSizes` uses **definite-length**.

### TxId CDDL (era dispatch)

```cddl
txId = base.ns7<byronTxId, shelley.transaction_id, allegra.transaction_id,
                mary.transaction_id, alonzo.transaction_id,
                conway.transaction_id, babbage.transaction_id>

byronTxId = [0, byron.txid]       ; regular tx
          / [1, byron.certificateid]
          / [2, byron.updid]
          / [3, byron.voteid]
```

### Tx CDDL (era dispatch)

```cddl
tx = base.ns7<byron.tx,
              serialisedShelleyTx<shelley.transaction>,
              serialisedShelleyTx<allegra.transaction>,
              serialisedShelleyTx<mary.transaction>,
              serialisedShelleyTx<alonzo.transaction>,
              serialisedShelleyTx<babbage.transaction>,
              serialisedShelleyTx<conway.transaction>>

serialisedShelleyTx<era> = #6.24(bytes .cbor era)   ; CBOR-in-CBOR tag 24
```

## KeepAlive Protocol

**Mini-protocol number: TBD** — Body is TODO in Blueprint.

### State Machine

```
StClient (initiator) → MsgKeepAlive(word16) → StServer
StServer (responder) → MsgKeepAliveResponse(word16) → StClient
StClient (initiator) → MsgDone → StDone
```

### CDDL

```cddl
keepAliveMessage = msgKeepAlive / msgKeepAliveResponse / msgDone

msgKeepAlive         = [0, base.word16]
msgKeepAliveResponse = [1, base.word16]
msgDone              = [2]
```

## PeerSharing Protocol

**Documented as placeholder** (`<>` link in SUMMARY.md) — not yet written in Blueprint.

## Base CDDL Definitions

From `src/codecs/base.cddl`:

```cddl
blockNo = word64
epochNo = word64
slotNo  = word64
coin    = word64
rational = [int, int]
keyhash = bstr .size 28
hash    = bstr .size 32
relativeTime = int

word8  = uint .size 1
word16 = uint .size 2
word32 = uint .size 4
word64 = uint .size 8
```

### Era Dispatch Patterns

**ns7** (namespace tag, era as first element):
```cddl
ns7<byron, shelley, allegra, mary, alonzo, babbage, conway>
  = [6, conway] / [5, babbage] / [4, alonzo] / [3, mary]
  / [2, allegra] / [1, shelley] / [0, byron]
```

**telescope7** (pastEra/currentEra pattern for historical encoding):
```cddl
telescope7<...>
  = [pastEra, pastEra, pastEra, pastEra, pastEra, pastEra, currentEra<conway>] /
    [pastEra, pastEra, pastEra, pastEra, pastEra, currentEra<babbage>] /
    ... etc.
```

## Known Gaps in Blueprint

- PeerSharing mini-protocol: no documentation
- NTC Handshake: no documentation
- NTC LocalTxSubmission: no documentation in Blueprint. Authoritative sources:
  - CDDL: `cardano-diffusion/protocols/cddl/specs/local-tx-submission.cddl` (ouroboros-network repo)
  - Codec (CBOR encoding): `ouroboros-network/protocols/lib/.../LocalTxSubmission/Codec.hs`
  - Type.hs: `ouroboros-network/protocols/lib/.../LocalTxSubmission/Type.hs`
  - Conway rejection type: `eras/conway/impl/src/Cardano/Ledger/Conway.hs` — `newtype ApplyTxError ConwayEra = ConwayApplyTxError (NonEmpty (ConwayLedgerPredFailure ConwayEra))`
  - ConwayLedgerPredFailure CBOR: `eras/conway/impl/src/.../Rules/Ledger.hs` — tags 1-9
  - ConwayUtxowPredFailure CBOR: `eras/conway/impl/src/.../Rules/Utxow.hs` — tags 0-18
  - Sum encoding rule: `encodeListLen (fieldCount + 1) <> encodeWord tag` (from cardano-ledger-binary Coders.hs)
  - MsgRejectTx wire: array(2) [uint(2), rejectReason] where rejectReason = CBOR array of 1+ ConwayLedgerPredFailure
  - No HFC wrapper on reject payload; tx in MsgSubmitTx uses ns7 era-dispatch with tag 24
- NTC TxMonitor: no documentation
- NTC LocalChainSync: no documentation
- KeepAlive body is TODO
- Multiplexer multi-segment message delimitation: marked as unclear
- Handshake: `MsgReplyVersion` removed from CDDL without explanation
- Handshake size limit (5760 bytes) rationale unclear
