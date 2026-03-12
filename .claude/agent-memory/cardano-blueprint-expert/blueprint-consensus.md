---
name: blueprint-consensus
description: Cardano Blueprint consensus layer documentation — chain validity, chain selection, forging, multi-era handling, Praos/TPraos/BFT/Genesis protocols
type: reference
---

# Cardano Blueprint — Consensus Layer

## Source Files

- `src/consensus/README.md` — Overview
- `src/consensus/chainvalid.md` — Chain validity rules
- `src/consensus/chainsel.md` — Chain selection, k parameter, Ouroboros Genesis
- `src/consensus/forging.md` — Block forging, leadership check
- `src/consensus/multiera.md` — Multi-era handling

## Protocol by Era

| Era | Protocol | Notes |
|-----|----------|-------|
| Byron | Ouroboros Classic | Retired |
| Byron (reimplementation, forging) | Ouroboros BFT | |
| Byron (reimplementation, processing) | Ouroboros Permissive BFT | |
| Shelley | Ouroboros Transitional Praos (TPraos) | |
| Allegra | Ouroboros TPraos | |
| Mary | Ouroboros TPraos | |
| Alonzo | Ouroboros TPraos | |
| Babbage | Ouroboros Praos | |
| Conway | Ouroboros Praos | Current |

Era transitions are enacted on-chain through specialized transactions.

## Architecture

The Consensus layer:
- Invokes the Ledger layer for block body validation
- Persists chains in the Storage layer
- Chains are diffused via the Network layer
- Block bodies come from the Mempool

Responsibilities:
1. **Chain validity check** — cryptographic verification, hash checking, header consistency
2. **Chain selection** — choosing best chain among competing candidates
3. **Leadership check and block forging** — determining who can forge per slot

## Chain Validity

### Envelope Validity (Protocol-independent)

Applies as a first sanity check before protocol-specific checks:
- Block number >= previous (or equal if previous was EBB)
- Slot number >= previous (or equal if EBB)
- Hash of previous block matches
- If block is a known checkpoint, must match checkpoint data
- Era of header matches era of body
- Byron: not EBB when none was expected
- Shelley+:
  - Protocol version <= max understood by this node
  - Header size <= max header size (from protocol params)
  - Body size <= max body size (from protocol params)

### Ouroboros BFT Validity

Block `B_i = (h, d, sl, σ_sl, σ_block)`:
- Signatures correct (slot signature + whole-block signature)
- Issuer delegated in Genesis block
- Issuer has not signed more than allowed blocks recently
- Slot > last signed slot
- `h` is hash of previous block
- Body `d` is valid sequence of transactions

### Ouroboros PBFT Validity

- EBB: valid if header is valid
- Regular block: valid as per Ouroboros BFT

### Ouroboros Praos/TPraos Validity

**TODO in Blueprint** — marked as `TODO` in chainvalid.md. See the Shelley/Praos papers and formal specs.

### Skipping Validation on Trusted Data

Blocks replayed from local trusted storage can skip validation checks (already validated). Used when restarting node and replaying chain from disk.

## Chain Selection

### Security Parameter k

- Mainnet: k = 2160 blocks
- Maximum rollback: chains forking more than k blocks ago are never considered
- Subdivides chain into **Immutable** (>= k+1 blocks from tip) and **Volatile** (k blocks from tip)
- Immutable part can be safely persisted to disk

### Forecast Range

- How far ahead header validity can be checked using fixed ledger state
- In Praos: `3k/f = 129,600` slots (~36 hours on mainnet)
- Headers can be validated using chain/ledger state at intersection if distance <= forecast range

### Chain Selection Rules by Protocol

**Classic / BFT / PBFT**: Longest chain wins. Tie → keep current.

**TPraos / Praos**: Same `maxvalid` function as Classic:
- Longer candidates preferred
- Ties broken in favor of already-selected chain

**Tie-breakers in Praos (Conway refinement)**:
1. Compare by length (longer wins)
2. If tips are from the same pool: higher opcert counter wins (can increase by at most 1)
3. If tips are from different pools: lower VRF value wins
   - Up through Babbage: unconditional VRF comparison
   - Conway+: VRF comparison only if blocks differ by <= n slots (prevents late blocks winning)

### Ouroboros Genesis (Sync Mode)

Used only during initial synchronization with the network:
- Problem: Praos length-based selection vulnerable to adversarial long chains during sync
- Genesis refinement: choose based on **block density** within a genesis window from intersection point
- Genesis window: `3k/f` slots
- Honest chain will be denser than adversarial chain within this window
- Gracefully converges to Praos length-based comparison once synced

Practical refinements:
- **Limit on Eagerness**: (TODO in Blueprint)
- **Limit on Patience**: (TODO in Blueprint)

## Block Forging

### Leadership Check in Ouroboros Praos

Probability of pool `U_i` with relative stake `α_i` being elected:
```
p_i = φ_f(α_i) = 1 - (1 - f)^α_i
```

Where `f` = active slot coefficient (mainnet: `0.05`, i.e., 1 block per 20 slots on average).

Important properties:
- Multiple slot leaders possible (events are independent)
- Slots with no leader possible
- Only the slot leader knows it is a leader
- Probability independent of whether stake is split among virtual parties

### Forging Sequence

1. Determine if stake pool is entitled to produce a block this slot
2. Acquire transactions from Mempool
3. Pack data into block body, produce header, emit signature

Note: Blueprint marks leadership schedule details (stability periods, epoch snaphots, which distribution is used) as TODO.

### Multi-leader / Multi-height Battles

- **Slot battle**: Multiple pools elected same slot → momentary fork → only one survives
- **Height battle**: Adjacent slots, block 1 doesn't reach node 2 before node 2 forges → short-lived fork
- Fast forging + fast diffusion is critical to minimize height battles

## Multi-Era Handling Strategies

Blueprint describes three approaches (DnD alignment metaphor):

| Approach | Description | Pros | Cons |
|---|---|---|---|
| Chaotic Evil | Only one era supported; update = hard upgrade | Simplest | No on-chain governance; history needs special code |
| Chaotic Good | Current + next era supported | On-chain governance possible | Two eras to maintain; history still special |
| True Neutral | All eras supported | Full code reuse, on-chain governance in types | Complex abstraction; steep onboarding |

Cardano is True Neutral — supports all eras simultaneously.

Era boundary subtleties: time handling, forecast range — content is **TODO** in Blueprint.

## Header/Body Split Rationale

- Headers contain consensus evidence (cryptographic proofs); bodies contain ledger data
- Constant-time validation of candidate chains using just headers → eliminates DoS attacks
- Enables separate ChainSync (headers) and BlockFetch (bodies) protocols
- Chain selection operates on headers only

## Resilience Principle

> The cost of the worst case should be no greater than the cost of the best case.

Do NOT optimize for best case — exposes node to DoS if adversary can trigger worst case.

## Ledger Interface Requirements

Consensus requires the Ledger to provide:
1. **Apply blocks** — validate and update ledger state
2. **Apply transactions** — single-transaction validation for mempool
3. **Tick time** — epoch boundary transitions, time-based state changes
4. **Forecasting** — predict ledger view (e.g., stake distribution) for future slots within forecast range

## Key Constants

- Mainnet `k` = 2160 blocks
- Mainnet `f` = 0.05 (active slot coefficient)
- Mainnet slot duration = 1 second (Praos)
- Genesis/forecast window = `3k/f` = 129,600 slots (~36 hours)
