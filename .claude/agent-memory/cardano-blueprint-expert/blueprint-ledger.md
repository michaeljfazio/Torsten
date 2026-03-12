---
name: blueprint-ledger
description: Cardano Blueprint ledger rules documentation — UTxO model, block/tx validation, fees, multi-phase validity, determinism, Conway block validation EraRules
type: reference
---

# Cardano Blueprint — Ledger Layer

## Source Files

- `src/ledger/README.md` — Overview, pointers to formal specs and CDDL
- `src/ledger/plan.md` — Documentation roadmap (what's planned)
- `src/ledger/block-validation.md` — Conway block validation with full mermaid flow diagrams
- `src/ledger/transaction-fee.md` — Fee calculation algorithm, worked example
- `src/ledger/state-transition.md` — Env/State/Signal model
- `src/ledger/concepts/blocks.md` — Block structure, header/body split
- `src/ledger/concepts/transactions.md` — STUB only
- `src/ledger/concepts/determinism.md` — Transaction and script determinism
- `src/ledger/state-transition/validity.md` — Multi-phase validity, static vs dynamic checks
- `src/ledger/state-transition/reading-specs.md` — STUB
- `src/ledger/constraints/header-body-split.md` — Header/body split implications, forecast window

## Block Structure

```
Block
├── Header (consensus relevance)
└── Body
    ├── Transaction Bodies  (compute resulting state)
    ├── Witnesses           (cryptographic validity proofs)
    ├── Auxiliary Data      (metadata, off-chain data, not processed by ledger state)
    └── Transaction Validity (list of phase-2 invalid transactions in block)
```

Key insight: For historical/trusted blocks, only Transaction Bodies need to be deserialized — witnesses and Aux Data don't affect state.

## State Transition Model

Three components:
- **Environment** (read-only): protocol parameters, current slot/block number
- **State**: current ledger state including UTxO set
- **Signal**: one of:
  - Block body (when new block added)
  - Transaction (mempool validation)
  - Tick (epoch boundary / time passage)

Ticks must always be valid. Block/tx validity is checked inductively from initial state.

## Determinism

### Transaction Determinism

Given valid transaction, outputs are fully determined by the transaction itself. Only need ledger state to check validity (e.g., inputs not spent) — not to compute outputs.

### Script Determinism

Given phase-1 valid transaction, script validity is determined only by transaction data and resolved transaction outputs. External ledger state cannot affect script execution.

### Implications for Node Developers

1. Historical block processing: only Transaction Bodies need deserializing (validity already established)
2. Expensive checks (crypto, script execution) only needed once — on mempool entry. Subsequent revalidation only needs dynamic checks.

## Multi-Phase Validity (Alonzo+)

### Phase 1

- Regular checks: size, fee, input validity, etc.
- Bounded work
- Failure: transaction does NOT go on chain, no fee paid
- Subject to dynamic re-checking when ledger state changes

### Phase 2

- Plutus script execution
- Only runs if Phase 1 passes
- Failure: transaction CAN go on chain (collateral consumed instead of regular inputs)
- Collateral must be locked by phase-1 verifiable input (VKey or native script)
- Static check: not re-evaluated on ledger state changes

### Babbage Collateral Return

Collateral in excess of minimum required is returned to a `collateral_return_address` when phase-2 fails. Added to address user concerns about losing more than necessary.

## Static vs Dynamic Checks

**Static checks** — evaluated using only transaction and its resolved inputs:
- Cryptographic signature checks
- Native (multisig/timelock) scripts
- Phase-2 checks (Plutus scripts)

**Dynamic checks** — require UTxO or other ledger state:
- Inputs exist in UTxO
- Transaction within validity window
- Transaction size vs protocol params

### Four Validation Scenarios

1. **Mempool entry**: All checks (static + dynamic)
2. **Mempool revalidation after new block**: Dynamic checks only
3. **Downloading new block from peer**: All checks
4. **Replaying trusted local block**: No checks — just apply transition

## Transaction Fee Calculation (Conway/Protocol v10)

### Formula

`total_fee = base_fee + ref_script_fee + execution_fee`

### Base Fee

```
base_fee = minFeeConstant + minFeeCoefficient * tx_byte_length
```
All integers — no fractional lovelace.

### Reference Script Fee

Algorithm (iterative over 25,600-byte increments):
1. Sum lengths of all `scriptRef` raw script bytes from inputs and reference inputs
2. Starting `baseFee = minFeeReferenceScripts.base` (= 15 lovelace on mainnet), `remaining = sum(scriptRefLengths)`
3. While remaining > 0:
   - Add `baseFee * min(remaining, range)` to fee
   - Scale `baseFee *= multiplier` (= 1.2, rational)
   - Decrease `remaining -= min(remaining, range)` (range = 25,600 bytes)
4. Take ceiling of result (because multiplier is rational)

### Execution Fee

```
execution_fee = ceiling(sum over redeemers of:
    redeemer.memory_units * prices.memory +
    redeemer.steps * prices.steps)
```
Take ceiling at the very end (prices are rational).

### Protocol Parameters Needed

- `minFeeConstant` (= 155381 lovelace on mainnet at time of example)
- `minFeeCoefficient` (= 44 lovelace/byte)
- `minFeeReferenceScripts`: {base, multiplier, range} (= {15, 1.2, 25600} on mainnet)
- `prices.memory` (= 0.0577 lovelace/memory unit)
- `prices.steps` (= 0.0000721 lovelace/step)

### Worked Example (mainnet tx f06e17af...)

- Tx: 1358 bytes
- 2 reference scripts: 2469 + 15728 = 18197 bytes total
- 3 redeemers: (1057954 mem, 335346191 steps), (28359, 8270119), (40799, 12323280)
- Base fee: 155381 + 44 * 1358 = 215,133 lovelace
- Ref script fee: 15 * 18197 = 272,955 lovelace (fits in first 25600-byte tier)
- Execution fee: ceil(90697.606839) = 90,698 lovelace
- **Total minimum: 578,786 lovelace** (on-chain declared 601,677 — slight padding common)

## Conway Block Validation — EraRules

### BBODY (entry point)

1. `conwayBbodyTransition`:
   - `totalScriptRefSize <= maxRefScriptSizePerBlock`
   - Updates state
2. `alonzoBbodyTransition`:
   - Calls EraRule LEDGERS
   - Checks `txTotalExUnits <= ppMaxExUnits`
   - Returns `BbodyState`

### LEDGERS

Calls Shelley `ledgersTransition` which repeats `ledgerTransition` per transaction:
- Optional: `EraRule Mempool` — `failOnNonEmpty unelectedCommitteeMembers`
- If `isValid = True`:
  - Check `submittedTreasuryValue == actualTreasuryValue`
  - Check `totalRefScriptSize <= maxRefScriptSizePerTx`
  - `failOnNonEmpty nonExistentDelegations`
  - Call EraRule CERTS, GOV, UTXOW
- If `isValid = False`:
  - Skip CERTS/GOV/UTXOW, use current (utxoState, certState)

### CERTS

Processes list of certificates:
- If empty: `validateZeroRewards`, update DRep expiry
- If non-empty: recurse then call EraRule CERT

CERT dispatches to:
- `ConwayTxCertDeleg` → EraRule DELEG
  - `ConwayRegCert`: checkDepositAgainstPParams, checkStakeKeyNotRegistered
  - `ConwayUnregCert`: checkInvalidRefund, isJust mUMElem, checkStakeKeyHasZeroRewardBalance
  - `ConwayDelegCert`: checkStakeKeyIsRegistered, checkStakeDelegateeRegistered
  - `ConwayRegDelegCert`: checkDepositAgainstPParams, checkStakeKeyNotRegistered, checkStakeKeyHasZeroRewardBalance
- Pool cert → EraRule POOL (Shelley):
  - `regPool`: verify netId, hash size, minPoolCost, pay deposit if new
  - `RetirePool`: verify pool exists, epoch within range
- Gov cert → EraRule GOVERT:
  - `ConwayRegDRep`: Map.notMember, deposit==ppDRepDeposit
  - `ConwayUnregDRep`: isJust mDRepState, failOnJust drepRefundMismatch
  - `ConwayUpdateDRep`: Map.member cred vsDReps
  - `ConwayResignCommitteeColdKey` / `ConwayAuthCommitteeHotKey`: checkAndOverwriteCommitteeMemberState

### GOV

- `failOnJust badHardFork`
- `actionWellFormed`
- `refundAddress`, `nonRegisteredAccounts`
- `pProcDeposit == expectedDeposit`
- `pProcReturnAddr == expectedNetworkId`
- Per action type:
  - `TreasuryWithdrawals`: mismatchedAccounts, checkPolicy
  - `UpdateCommittee`: Set.null conflicting, Map.null invalidMembers
  - `ParameterChange`: checkPolicy
- `ancestryCheck`
- `failOnNonEmpty unknownVoters`
- `failOnNonEmpty unknownGovActionIds`
- `checkBootstrapVotes`
- `checkVotesAreNotForExpiredActions`
- `checkVotersAreValid`
- Returns `updatedProposalStates`

### UTXOW

`babbageUtxowTransition`:
- `validateFailedBabbageScripts`
- `babbageMissingScripts`
- `missingRequiredDatums`
- `hasExactSetOfRedeemers`
- `Shelley.validateVerifiedWits` — signature verification
- `validateNeededWitnesses`
- `Shelley.validateMetadata`
- `validateScriptsWellFormed`
- `ppViewHashesMatch` — protocol param hash check
- Calls EraRule UTXO (Conway)

### UTXO (Conway)

- `disjointRefInputs`
- `Allegra.validateOutsideValidityIntervalUtxo`
- `Alonzo.validateOutsideForecast`
- `Shelley.validateInputSetEmptyUTxO`
- `feesOk`
- `Shelley.validateBadInputsUTxO`
- `Shelley.validateValueNotConservedUTxO`
- `validateOutputTooSmallUTxO`
- `Alonzo.validateOutputTooBigUTxO`
- `Shelley.validateOutputBootAddrAttrsTooBig`
- `Shelley.validateWrongNetwork`
- `Shelley.validateWrongNetworkWithdrawal`
- `Alonzo.validateWrongNetworkInTxBody`
- `Shelley.validateMaxTxSizeUTxO`
- `Alonzo.validateExUnitsTooBigUTxO`
- `Alonzo.validateTooManyCollateralInputs`
- Calls EraRule UTXOS (Conway)

### UTXOS (Conway)

- If `isValidTxL = True`: `expectScriptsToPass` → update utxos'
- If `isValidTxL = False`: `evalPlutusScripts FAIL` → collateral consumed, update utxos'

## Header/Body Split Implications

- `3k/f` slots = "forecast window" (also "stability window" in Shelley spec — same value, different concepts)
- Transactions cannot affect header validity within this window
- Achieved by using **past stake snapshot** (not current) for leader election
- Ledger must maintain relevant snapshots (mark/set/go model)

## Authoritative Sources

- **Formal ledger spec**: https://intersectmbo.github.io/formal-ledger-specifications/
- **Haskell specs**: https://github.com/IntersectMBO/cardano-ledger (README has all era PDFs)
- **CDDL files**: https://github.com/search?q=repo%3AIntersectMBO%2Fcardano-ledger+path%3A.cddl
- **Conway formal spec**: https://intersectmbo.github.io/formal-ledger-specifications/conway-ledger.pdf

## Documentation Gaps in Blueprint

- Transaction concept page is a stub
- Reading specs page is a stub
- No era-specific rules documented except Conway
- No reward calculation documentation
- No treasury/deposits documentation
- No governance ratification thresholds
- Non-integral math section planned but not written
- Snapshots section planned but not written
