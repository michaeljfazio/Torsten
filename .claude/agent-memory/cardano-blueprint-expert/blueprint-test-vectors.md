---
name: blueprint-test-vectors
description: Cardano Blueprint available test data and vectors — ledger conformance tests, handshake test data, LocalStateQuery examples
type: reference
---

# Cardano Blueprint — Test Vectors & Test Data

## Ledger Conformance Test Vectors

### Location

`src/ledger/conformance-test-vectors/`
- `README.md` — Description and generation instructions
- `vectors.tar.gz` — The actual test data (binary, download from GitHub)

### What They Cover

- **Conway era** ledger state transitions only
- Each vector = one transaction + "before" ledger state + "after" ledger state
- Vectors grouped into directories by unit test that generated them
- Numbered sequentially within each group
- Removed: Ledger V9 tests, `BodyRefScriptsSizeTooBig`, `TxRefScriptsSizeTooBig` (too large)

### Protocol Parameters Optimization

To reduce size, protocol parameter records are stored **by hash**:
- Ledger states reference protocol params by `Hash` instead of inline
- All unique parameter records stored in `pparams-by-hash/` directory

### Generation Source

Generated from SundaeSwap fork of cardano-ledger:
```
git clone git@github.com:SundaeSwap-finance/cardano-ledger-conformance-tests.git
cd cardano-ledger-conformance-tests
git checkout 34365e427e6507442fd8079ddece9f4a565bf1b9
cabal test cardano-ledger-conway
tar czf vectors.tar.gz eras/conway/impl/dump/*
```

Original discussion: https://github.com/IntersectMBO/cardano-ledger/issues/4892#issuecomment-2880444621

### Test Vector Format

Each test vector consists of:
1. Starting ledger state (CBOR encoded)
2. One or more transactions (CBOR encoded)
3. Expected resulting ledger state OR expected validation error

### Logbook Notes on Test Vectors (2025-03-25)

From project logbook — intent and planned evolution:
- Hand-written test scenarios provide "better signal to noise ratio" than generated conformance tests
- Plan: piggy-back on cardano-ledger test scenarios and dump vectors when running them
- Want to test at BBODY or LEDGERS rule level (block/transaction level "signal")
- Want to share ledger state across multiple test cases (reduce data size)
- Exact error codes out of scope, but common "slug/code" across implementations is desirable
- Best source: hand-crafted scenarios in cardano-ledger Imp tests (e.g., alonzo/impl/testlib/Test/Cardano/Ledger/Alonzo/Imp/UtxowSpec/Valid.hs)

## Handshake Test Data

### Location

`src/network/node-to-node/handshake/test-data/`

5 test cases: `test-0`, `test-1`, `test-2`, `test-3`, `test-4`

Each is a binary CBOR file stored as base64 in the GitHub API.

These are raw CBOR-encoded handshake messages for testing implementation compliance.

## LocalStateQuery Examples

### Location

`src/client/node-to-client/state-query/examples/getSystemStart/`

Files:
- `query.cbor` — Example `MsgQuery` for GetSystemStart
- `result.cbor` — Example `MsgResult` for GetSystemStart

These are binary CBOR files that can be used to test LSQ message parsing.

## Transaction Fee Worked Example

In `src/ledger/transaction-fee.md`:

Full mainnet transaction with known fee:
- **TxID**: `f06e17af7b0085b44bcc13f76008202c69865795841c692875810bc92948d609`
- **Raw hex**: 1358-byte transaction (full hex provided in doc)
- **Reference scripts**: 2 scripts (2469 + 15728 bytes)
- **Redeemers**: 3 redeemers with specific execution units
- **Computed min fee**: 578,786 lovelace
- **On-chain declared fee**: 601,677 lovelace

This can be used to test fee calculation implementation.

## Related External Test Resources

From Blueprint references and links:
- **Plutus conformance tests**: https://github.com/IntersectMBO/plutus/tree/master/plutus-conformance
- **Aiken acceptance tests**: https://github.com/aiken-lang/aiken/tree/main/examples/acceptance_tests/script_context/v3
- **Cardano-ledger Imp tests**: https://github.com/IntersectMBO/cardano-ledger/blob/master/eras/conway/impl/testlib/Test/Cardano/Ledger/Conway/Imp.hs
- **Cardano-ledger conformance test suite**: https://github.com/IntersectMBO/cardano-ledger/tree/master/libs/cardano-ledger-conformance
- **Ethereum tests (inspiration model)**: https://github.com/ethereum/tests

## What's NOT Available as Test Data in Blueprint

- No consensus test vectors (VRF, KES, opcerts)
- No network mini-protocol test vectors (except handshake)
- No Byron era test vectors
- No pre-Conway ledger test vectors
- No chain selection test vectors
- No mempool test vectors
- No Plutus execution test vectors (those are in plutus repo)
