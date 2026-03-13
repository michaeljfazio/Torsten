# Node Validator Agent Memory

## Key Files
- Node binary: `./target/release/torsten-node`
- CLI binary: `./target/release/torsten-cli`
- Config dir: `./config/` (preview-config.json, preview-topology.json)
- Preview DB: `./db-preview/` — slot ~106.686M, block ~4.1018M, epoch 1234
- Ledger snapshot: `<db>/ledger-snapshot.bin` (~1,111 MB)
- Node logs: `/tmp/torsten-validation-run.log`

## Startup Command Pattern
```
TORSTEN_PIPELINE_DEPTH=150 ./target/release/torsten-node run \
  --config config/preview-config.json \
  --topology config/preview-topology.json \
  --database-path ./db-preview \
  --socket-path ./node.sock \
  --host-addr 0.0.0.0 --port 3001 \
  > /tmp/torsten-validation-run.log 2>&1 &
```
NOTE: Always `pkill -f torsten-node && rm -f ./node.sock` before restart.
NOTE: Delete `db-preview/ledger-snapshot.bin` to force fresh replay with genesis seeding.

## Preview Testnet Baselines (2026-03-13, run #4 — main v0.1.0-alpha.14)
- DB at slot ~106.699M / block ~4.1022M / epoch 1234 (after shutdown flush)
- Peer connected: 3.134.226.73:3001 rtt_ms=587 (5 hot peers total)
- Snapshot loaded with v1→v2 VERSION MISMATCH WARN but succeeded (new fields at end of struct)
- Catch-up to tip after snapshot load: 39 blocks in ~23s
- At-tip block reception: new blocks every ~20-60s, txs=1-2
- Flush on SIGTERM: 44 volatile blocks flushed → 4.5s snapshot save → clean shutdown
- 1 warning: "Snapshot version mismatch — snapshot may fail to load" (non-fatal this time)
- Build time: 32.61s (incremental, 7 crates recompiled)

## Preview Testnet Baselines (2026-03-13, run #3 — main v0.1.0-alpha.13)
- DB at slot ~106.698M / block ~4.1022M / epoch 1234 (after shutdown flush)
- Peer connected: 13.58.19.0:3001 rtt_ms=586 (5 hot peers total)
- Snapshot FAILED to load (bincode struct change — see Known Issues #1)
- Genesis replay: 4,102,144 blocks in 112s at 36,577 blk/s — fast!
- Catch-up to tip after replay: 53 blocks
- At-tip block reception: new blocks every ~20-60s, txs=0-6
- Flush on SIGTERM: 57 volatile blocks flushed → 4.5s snapshot save → clean shutdown
- 1 warning: snapshot deserialization failure (bincode break)
- Build time: 32.60s (incremental, 7 crates recompiled)

## Prometheus Metrics Port Conflict Note
- Haskell cardano-node also runs on `localhost:12798` (127.0.0.1 only)
- Torsten binds to `*:12798` (all interfaces)
- `curl http://localhost:12798/metrics` hits the HASKELL node (more specific binding wins)
- Use `curl http://192.168.1.111:12798/metrics` (LAN IP) to reach Torsten metrics
- Or: kill the Haskell node before running Torsten for exclusive port access

## N2C Client Bug: Large Response Reassembly (2026-03-13, OPEN)
- **BUG**: `recv_segment()` in `crates/torsten-network/src/n2c_client.rs:687` only handles ONE mux segment
- Fix needed: collect multiple segments with same protocol_id until CBOR is complete
- Queries affected: `query drep-state` and `query pool-params` (and any large response)
- Error message: `Error: Failed to query DRep state: Protocol error: response too large`
- Root cause: fixed 65536-byte buffer; responses >65535 bytes need multi-segment reassembly
- The send path already handles chunking (`encode()` splits payloads), but recv does not reassemble
- Buffer at line 688: `let mut buf = vec![0u8; 65536];`

## Storage Architecture (post-redesign, commit 83c4f11)
- ImmutableDB: append-only `.chunk` + `.secondary` files (db-preview/immutable/)
- tip.meta file tracks immutable tip (binary: slot u64 BE + hash32 + block_no u64 BE)
- VolatileDB: in-memory HashMap, last k=2160 blocks, LOST ON RESTART
- ChainDB: routes volatile→immutable for blocks deeper than k
- UtxoStore: cardano-lsm in `db-preview/snapshots/latest/` for UTxO-HD (on-disk UTxO set)
- Ledger snapshot: bincode-serialized LedgerState, epoch-numbered + latest symlink

## FIXED: Shutdown Flush (2026-03-13, validated multiple runs)
- On SIGTERM: volatile blocks flushed to ImmutableDB FIRST, then snapshot saved
- Log sequence: "Flushed volatile blocks to ImmutableDB blocks=N" → "Snapshot saved" → "Shutdown complete"
- On restart: snapshot loads in ~7-9 seconds (NO replay) — confirmed working
- File: `crates/torsten-storage/src/chain_db.rs` line 417 (flush_all_to_immutable)
- File: `crates/torsten-node/src/node.rs` line ~1547 (shutdown flow)

## Known Issues (Current — 2026-03-13, v0.1.0-alpha.14)
1. **WARNING: Bincode snapshot version mismatch on struct field addition** — adding new fields to
   bincode-serialized structs can break existing snapshots. Version bump + end-of-struct appending
   partially mitigates this. The v1→v2 migration succeeded this run (fields appended at end) but
   the warning fires. Future: implement proper migration chain or use serde_json/postcard.
   - Commit 9eb94b1 bumped SNAPSHOT_VERSION to 2; commit 8e00ce4 added fields to GovernanceState
   - Warning: `WARN torsten_ledger::state: Snapshot version mismatch — snapshot may fail to load. snapshot_version=1 current_version=2`
   - File: `crates/torsten-ledger/src/state/mod.rs` (SNAPSHOT_VERSION = 2, load_snapshot)
   - Impact: warning fires but snapshot loads successfully IF new fields were appended at end
2. **N2C large response reassembly** — `recv_segment` only handles one 65535-byte segment
   - Affects: `query drep-state`, `query pool-params`
   - File: `crates/torsten-network/src/n2c_client.rs:687`

## Working Features Confirmed (2026-03-13, run #4, main branch v0.1.0-alpha.14)
- Build: WORKS — clean, zero warnings, compiles in 32.61s (incremental)
- Snapshot load (v1→v2 mismatch): WORKS with warning — no replay needed
- Peer connections: WORKS — 3.134.226.73:3001 rtt_ms=587, 5 hot peers
- Catch-up to tip: WORKS — 39 blocks applied in ~23s after snapshot restore
- At-tip block reception: WORKS — Conway blocks every ~20-60s, txs=1-2
- N2C query tip: WORKS — syncProgress=100.00%, era=Conway, slot 106699813
- N2C ratify-state: WORKS — enacted=0, expired=0, delayed=false (new in v0.1.0-alpha.14)
- N2C treasury: WORKS — Treasury=42,301,639,077 ADA, Reserves=2,625,733,633 ADA
- N2C constitution: WORKS — empty URL + zero hash + no guardrail (expected on preview)
- N2C protocol-parameters: WORKS — full Conway PParams with Plutus V1/V3 cost models
- N2C stake-distribution: WORKS — 657 pools listed with fractions
- N2C gov-state: WORKS — committee_members=1, active_proposals=2 (TreasuryWithdrawals)
- Prometheus metrics: WORKS — http://192.168.1.111:12798/metrics (use LAN IP)
  - blocks_applied=44, sync_progress=10000 (100.00%), utxo_count=2,937,685
  - peers_connected=5, hot=5, epoch=1234, treasury=42.3T lovelace
  - drep_count=8791, proposal_count=2, pool_count=657, delegation_count=11522
- SIGTERM shutdown: WORKS — flushes 44 volatile blocks, saves 1112.8 MB snapshot in 4.5s

## Operational Notes
- Always `--testnet-magic 2` with CLI query tip for correct syncProgress
- N2N port 3001 / Metrics port 12798 — conflict if old node running
- No `torsten-config.json` — use `config/preview-config.json` directly
- On restart after clean SIGTERM: snapshot loads in ~7-9 seconds (NO replay) — FIXED
- On restart after crash/SIGKILL: still does full replay (~110s) — volatile data lost without flush
- `TORSTEN_REPLAY_LIMIT=0` skips replay (old behavior, wrong ledger state)
- Snapshot grows to ~1.1GB with full UTxO state
- CLI subcommand is `query treasury` (not `query account-state`)
