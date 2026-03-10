# Epoch Nonce Calculation (Praos) - Definitive Reference

## Source Files
- Praos protocol: `ouroboros-consensus-protocol/src/.../Protocol/Praos.hs` (tickChainDepState, reupdateChainDepState)
- VRF nonce: `ouroboros-consensus-protocol/src/.../Protocol/Praos/VRF.hs` (vrfNonceValue, hashVRF)
- Nonce type: `cardano-ledger/libs/cardano-ledger-core/src/.../BaseTypes.hs` (Nonce, ⭒ operator)
- prevHashToNonce: `cardano-ledger/libs/cardano-protocol-tpraos/src/.../BHeader.hs`
- Stability window: `cardano-ledger/eras/shelley/impl/src/.../StabilityWindow.hs`
- TPraos TICKN (legacy): `cardano-ledger/libs/cardano-protocol-tpraos/src/.../Rules/Tickn.hs`
- TPraos UPDN (legacy): `cardano-ledger/libs/cardano-protocol-tpraos/src/.../Rules/Updn.hs`

## PraosState Fields
- `praosStateEvolvingNonce` (eta_v): updated EVERY block, NEVER reset
- `praosStateCandidateNonce` (eta_c): tracks eta_v until freeze point, then frozen
- `praosStateLabNonce`: prevHash of current block (= parent block hash)
- `praosStateLastEpochBlockNonce`: snapshot of labNonce at epoch boundary
- `praosStateEpochNonce`: computed at epoch boundary
- `praosStatePreviousEpochNonce`: prior epoch's nonce (for Peras)

## Per-Block Update (reupdateChainDepState)
```
eta = vrfNonceValue(vrf_certified_output)
    = blake2b_256(blake2b_256("N" || raw_vrf_output_bytes))   -- double hash with domain sep
newEvolvingNonce = blake2b_256(evolvingNonce || eta)           -- via ⭒ operator
candidateNonce = if slot + 4k/f < firstSlotNextEpoch
                 then newEvolvingNonce
                 else candidateNonce (frozen)
labNonce = prevHashToNonce(block.prevHash)                     -- just type cast, no rehash
```

## Epoch Boundary (tickChainDepState, fires BEFORE first block of new epoch)
```
epochNonce = blake2b_256(candidateNonce || lastEpochBlockNonce)  -- via ⭒
previousEpochNonce = old epochNonce
lastEpochBlockNonce = labNonce  -- snapshot current labNonce for next transition
-- evolvingNonce and candidateNonce are NOT reset, carry forward
```

## Stability Windows
- `computeStabilityWindow = ceiling(3k/f)` -- used for chain selection
- `computeRandomnessStabilisationWindow = ceiling(4k/f)` -- used for candidate nonce freeze
- Mainnet: k=2160, f=0.05 → stability=129600, randomness=172800

## Key Facts
- Evolving nonce is NEVER reset (carries across epochs)
- Candidate nonce freeze: last 4k/f slots of epoch (NOT first 4k/f or first 3k/f)
- No extra entropy in Praos (was only in TPraos/Shelley era)
- labNonce = prevHash of block = hash of PARENT block (just type cast, no additional hashing)
- At epoch boundary, lastEpochBlockNonce gets labNonce from last block processed before tick

## Torsten Bugs Found (2026-03-10)
1. Missing candidate_nonce (only has rolling_nonce)
2. Nonce window inverted (checks from start, should check from end; uses 3k/f not 4k/f)
3. Rolling nonce reset to genesis_hash at epoch boundary (should never reset)
4. Uses first_block_hash_prev_epoch instead of lab_nonce (last block's parent hash)
