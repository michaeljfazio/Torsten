# Node Validator Agent Memory

## Key Files
- Node binary: `./target/release/torsten-node`
- CLI binary: `./target/release/torsten-cli`
- Config dir: `./config/` (preview-config.json, preview-topology.json)
- Preview DB: `./db-preview/` — slot ~106.4M, block ~4.09M
- Ledger snapshot: `<db>/ledger-snapshot.bin`
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

## Preview Testnet Baselines (2026-03-10)
- DB at slot ~106.4M / block ~4.09M / epoch 1232
- Peers: 13.58.19.0, 3.70.89.92, 99.80.240.19, 52.211.202.88, 3.74.40.92 (8 known, 5 hot)
- N2N handshake: ~566-740ms, version 14
- Ledger replay speed (full 4M block replay): ~16,303 blocks/sec
- Final UTxO count after full replay: 2,935,708 at snapshot, 2,936,107 at tip
- Final snapshot size: ~1,165 MB

## VRF Stake Bug — CRITICAL UNRESOLVED (commit ca423d7, 2026-03-10)
- lncf (Euler CF) fix did NOT resolve VRF rejections — bug is in stake accounting, not math
- Pool pool1ynfnjspgckgxjf2zeye8s33jz3e3ndk9pcwp0qzaupzvvd8ukwt rejected at slot 106474747
- Torsten relative_stake = 0.000588 (0.059%) vs Koios actual = 0.0601 (6.0%)
- 102x undercount of per-pool stake fraction causes strict VRF rejection
- After first rejection, node stalls permanently ("does not connect to tip" cascade)
- Snapshot CLI shows 55M ADA set stake for this pool; the fraction implies only ~575k ADA is used
- Koios epoch 1232 total active_stake: 1,177,946,537 ADA; Torsten: 977,890,873 ADA (17% under)
- Root cause: stake_distribution.stake_map (UTxO tracking) does not fully accumulate stake
  for all delegated credentials → pool_stake in epoch snapshots is systematically undercounted
- See `vrf-rejection-analysis.md` for full analysis

## Known Issues (Current — 2026-03-10)
1. **CRITICAL: VRF rejections cause chain liveness failure**
   - set_snapshot.pool_stake is ~96-102x too low per pool relative to Koios
   - VRF threshold 96x too small → valid blocks rejected → node stalls permanently at tip
   - Fix needed: trace why stake_distribution.stake_map values are so much smaller than on-chain
   - The 17% total undercount (977B vs 1177B lovelace) suggests reward accounts not included
     OR stake_map entries missing for many delegators (not registered before delegation)
2. **Conway governance PP parameter updates only partially applied**
   - non-HardFork PP changes (executionUnits, committeeMinSize) not all applied
3. **`query tip` returns zeros immediately after startup** — race condition (wait for replay)
4. **`query stake-pools` garbled data** — CLI decoder mismatch

## Working Features Confirmed (2026-03-10, full 4M replay)
- Build: WORKS — clean, zero warnings
- Genesis UTxO seeding: WORKS — 1 UTxO seeded, grows to 2.9M during replay
- Peer connections: 5 peers established
- Ledger replay: WORKS — no panics, 4.09M blocks in 251 seconds at 16,303 b/s
- Epoch transitions: WORKS (1,232 transitions)
- Pre-Conway PPUP: v6→7 at epoch 3, v7→8 at epoch 22, v8→9 at epoch 646
- Conway governance HardFork: v9→10 at epoch 743
- Rollback handling: WORKS
- N2C query tip / protocol-parameters: WORKS
- Prometheus metrics: WORKS — sync_progress_percent: 10000 (100%)

## Operational Notes
- Always `--testnet-magic 2` with CLI query tip for correct syncProgress
- N2N port 3001 / Metrics port 12798 — conflict if old node running
- No `torsten-config.json` — use `config/preview-config.json` directly
- Delete ledger-snapshot.bin to trigger fresh replay with genesis UTxO seeding
- `TORSTEN_REPLAY_LIMIT=0` skips replay (old behavior, node works at tip but wrong ledger state)
- Snapshot grows to ~1.1GB with full UTxO state
