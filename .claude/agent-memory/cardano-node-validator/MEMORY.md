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

## Preview Testnet Baselines (2026-03-13, v0.1.0-alpha.10 — commit 05e3f38)
- DB at slot ~106.691M / block ~4.1020M / epoch 1234
- Peer connected: 3.70.89.92:3001 rtt_ms=749 (5 hot peers total)
- Snapshot load time: ~8 seconds (no replay, instant)
- Snapshot size: 1111.4 MB, 2,937,499 UTxOs at start, 2,937,531 at shutdown
- Catch-up from snapshot to tip: 88 blocks in ~8s (was 28 blocks in ~18s)
- At-tip block frequency: new blocks every ~20-60s, some with txs (txs=1)
- Flush on SIGTERM: 93 volatile blocks flushed → 5.3s snapshot save → clean shutdown
- No warnings, no errors, no panics in full run

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

## Known Issues (Current — 2026-03-13, v0.1.0-alpha.8)
1. **N2C large response reassembly** — `recv_segment` only handles one 65535-byte segment
   - Affects: `query drep-state`, `query pool-params`
   - File: `crates/torsten-network/src/n2c_client.rs:687`
2. **N2C Broken Pipe warnings** — observed after failed queries (broken pipe on client disconnect)
   - `WARN torsten_network::n2c: N2C connection error: IO error: Broken pipe (os error 32)`
   - Not critical — client disconnects after a failed query; server handles it gracefully
3. **Conway governance PP parameter updates only partially applied** (ongoing)

## Working Features Confirmed (2026-03-13, v0.1.0-alpha.10 validation, commit 05e3f38)
- Build: WORKS — clean, zero warnings, compiles in 33s (incremental)
- Snapshot load (restart): WORKS — 8s load time, no replay
- Peer connections: WORKS — 3.134.226.73:3001 rtt_ms=590
- Catch-up to tip: WORKS — 28 blocks applied in ~18s
- At-tip block reception: WORKS — Conway blocks every ~20-60s
- All protocol version upgrades: WORKS (Byron→Conway historical replay confirmed)
- N2C query tip: WORKS — syncProgress=100.00%, era=Conway
- N2C protocol-parameters: WORKS — full Conway PParams with Plutus V1/V3 cost models
- N2C stake-distribution: WORKS — 755 pools listed with fractions
- N2C gov-state: WORKS — committee_members=1, active_proposals=2
- N2C treasury: WORKS — Treasury=42.3B ADA, Reserves=2.6B ADA
- N2C committee-state: WORKS — 1 active member
- N2C constitution: WORKS (empty/null on preview)
- N2C stake-snapshot: WORKS — mark/set/go stakes for all 755 pools
- Prometheus metrics: WORKS — http://localhost:12798/metrics (Haskell-compatible format)
- SIGTERM shutdown: WORKS — flushes 37 volatile blocks, saves 1111.4 MB snapshot

## Operational Notes
- Always `--testnet-magic 2` with CLI query tip for correct syncProgress
- N2N port 3001 / Metrics port 12798 — conflict if old node running
- No `torsten-config.json` — use `config/preview-config.json` directly
- On restart after clean SIGTERM: snapshot loads in ~7-9 seconds (NO replay) — FIXED
- On restart after crash/SIGKILL: still does full replay (~110s) — volatile data lost without flush
- `TORSTEN_REPLAY_LIMIT=0` skips replay (old behavior, wrong ledger state)
- Snapshot grows to ~1.1GB with full UTxO state
- CLI subcommand is `query treasury` (not `query account-state`)
