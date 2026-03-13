---
name: MsgRejectTx Wire Format
description: Complete CBOR encoding chain for LocalTxSubmission MsgRejectTx, from mini-protocol envelope through HFC wrapping to all Conway predicate failure types with their tag numbers
type: reference
---

## Wire Format Summary

MsgRejectTx = `[2, <hfc_wrapped_error>]`

For Conway era with HFC enabled:
```
[2, [1, [6, <conway_error_list>]]]
     ^   ^   ^
     |   |   `-- NS wrapper: era index 6 = Conway
     |   `------ EitherMismatch success: array(1)
     `---------- msgRejectTx tag
```

The error payload is `NonEmpty ConwayLedgerPredFailure` encoded as a CBOR list.

## Key Files
- Mini-protocol codec: ouroboros-network/protocols/lib/.../LocalTxSubmission/Codec.hs
- HFC wrapping: ouroboros-consensus/.../HardFork/Combinator/Serialisation/SerialiseNodeToClient.hs
- HFC sum encoding: ouroboros-consensus/.../HardFork/Combinator/Serialisation/Common.hs (encodeNS, encodeEitherMismatch)
- Conway ApplyTxError: cardano-ledger/eras/conway/impl/src/.../Conway.hs (newtype over NonEmpty ConwayLedgerPredFailure)
- Shelley serialization instance: ouroboros-consensus-cardano/src/shelley/.../Shelley/Node/Serialisation.hs line 356-358 (uses toEraCBOR @era)
- ConwayLedgerPredFailure: .../Conway/Rules/Ledger.hs (tags 1-9)
- ConwayUtxowPredFailure: .../Conway/Rules/Utxow.hs (tags 0-18)
- ConwayUtxoPredFailure: .../Conway/Rules/Utxo.hs (tags 0-22)
- ConwayUtxosPredFailure: .../Conway/Rules/Utxos.hs (tags 0-1)
- ConwayCertsPredFailure: .../Conway/Rules/Certs.hs (tags 0-1)
- ConwayCertPredFailure: .../Conway/Rules/Cert.hs (tags 1-3)
- ConwayDelegPredFailure: .../Conway/Rules/Deleg.hs (tags 1-8)
- ConwayGovCertPredFailure: .../Conway/Rules/GovCert.hs (tags 0-5)
- ShelleyPoolPredFailure: .../Shelley/Rules/Pool.hs (tags 0-6, manual encCBOR, NOT Sum DSL)
- ConwayGovPredFailure: .../Conway/Rules/Gov.hs (tags 0-18)
- Mismatch type: .../BaseTypes.hs (EncCBORGroup = flat supplied,expected; EncCBOR = array(2)[supplied,expected])

## Sum DSL Wire Format
`Sum Constructor TAG !> To field1 !> To field2` encodes as:
`array(N+1) [TAG, field1_cbor, field2_cbor]`
where N = number of fields

`ToGroup` inlines the group encoding (Mismatch inlines 2 fields into the parent array).
`swapMismatch` swaps supplied/expected order in some failures (check each one).
