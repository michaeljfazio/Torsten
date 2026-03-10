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

## Preview Testnet Baselines (2026-03-10, latest commits: ae0cc46/55b0a0c/b72a44a)
- DB at slot ~106.4M / block ~4.09M / epoch 1232
- Peers: 13.58.19.0, 3.70.89.92, 99.80.240.19, 52.211.202.88, 3.74.40.92 (8 known, 5 hot)
- N2N handshake: ~585-745ms, version 14
- Ledger replay speed (full 4M block replay): ~16,303 blocks/sec
- Replay duration: 251 seconds for 4,093,131 blocks
- Build time: instant (binary current), ~28 seconds for full rebuild
- Final UTxO count after full replay: 2,935,708 at snapshot, 2,936,107 at tip
- Final snapshot size: 1,165 MB

## Genesis UTxO Seeding — FIXED (commit 01a4739, 2026-03-10)
- Byron genesis file: `config/preview-byron-genesis.json` (also preprod/mainnet)
- Config key: `ByronGenesisFile` in preview-config.json
- Seeding log line: `Ledger: genesis UTxOs seeded seeded=1 total_lovelace=30000000000000000 utxo_count=1`
- Preview has 1 nonAvvmBalance genesis UTxO of 30,000,000 ADA
- CRITICAL: seeding only happens when NO snapshot exists; must delete snapshot to trigger
- UTxO count climbs from 1 → 2.9M over full replay (correct behavior)
- Only credits stake when UTxO application succeeds (debit tracking now correct)

## VRF/Consensus at Tip (After Genesis UTxO Fix — commit 01a4739, 2026-03-10)
- All live blocks accepted: 8 new blocks accepted after catching up, zero VRF rejections
- No "Not a slot leader" errors observed (previous issue was fixed by correct stake distribution)
- Epoch nonce: 68727533dd7ba820be27e194df11bc20395b9f0d41d5f3c57c0e439749476a3d
- Node is FULLY FUNCTIONAL at chain tip

## Stake Distribution (After Genesis UTxO Fix — commit 01a4739, 2026-03-10)
- Total staked fraction sum: 0.95 (previously near-zero or over-accumulating)
- pool1xgmqwh23yc2jp52k7jn249x56v6nyhl9nhxaeg6hq8tmc5t78rq: Torsten 1.04%, Koios 4.51%
  (ratio: 0.23 of actual — still underreporting ~4x but functional for VRF)
- Stake distribution still shows discrepancy vs Koios, but no longer causes VRF rejection
- Investigation note: delegation_count=11,515 (unchanged) but utxo set now proper
- Likely residual: rewards/treasury not yet fully integrated into stake calculations

## Protocol Version Transitions — ALL CONFIRMED CORRECT (2026-03-10, commits ae0cc46/55b0a0c/b72a44a)
All four protocol version transitions verified in full 4M block replay:
- Epoch 3: 6.0 → 7.0 — CONFIRMED at 05:24:58 (3s into replay), 7 proposers
  Log: "Protocol version change via pre-Conway update epoch=3 from_major=6 from_minor=0 to_major=Some(7) to_minor=Some(0)"
- Epoch 22: 7.0 → 8.0 — CONFIRMED at 05:25:00 (2s later), 7 proposers
  Log: "Protocol version change via pre-Conway update epoch=22 from_major=7 from_minor=0 to_major=Some(8) to_minor=Some(0)"
- Epoch 646: 8.0 → 9.0 — CONFIRMED at 05:27:14 (proposals targeting epoch 645 applied at 646), 7 proposers
  Log: "Protocol version change via pre-Conway update epoch=646 from_major=8 from_minor=0 to_major=Some(9) to_minor=Some(0)"
- Epoch 743: 9.0 → 10.0 — CONFIRMED at 05:27:31 via Conway HardForkInitiation governance action
  Log: "Governance proposal ratified: GovActionId { transaction_id: Hash(049ae5d612b2fa825655809133b023d60c7f8cac683c278cf95de1622e4592f3), action_index: 0 }"
  Log: "Hard fork initiated: protocol version 10.0"
  3 governance proposals ratified and enacted at epoch 743 boundary
- CLI `query protocol-parameters` returns protocolVersion: {major:10, minor:0} — CORRECT

## Protocol Parameters Discrepancy (2026-03-10, after ALL version transitions fixed)
- maxBlockBodySize: CORRECT (90112)
- protocolVersion: CORRECT — {major:10, minor:0}
- committeeMinSize: 0 (Torsten) vs 3 (actual on-chain) — still discrepant, governance enactment bug
- maxTxExecutionUnits.memory: 14,000,000 (Torsten) vs 16,500,000 (actual) — governance PP update bug
- maxBlockExecutionUnits: {62M mem, 20B steps} (Torsten) vs {72M mem, 20B steps} (actual) — governance PP update bug
- Root cause of remaining discrepancies: Conway governance PP updates (not HardForkInitiation) not all being applied

## Known Issues (Current — after PPUP epoch-boundary fix, 2026-03-10)
1. **Stake distribution underreports ~4x** — VRF still works (threshold met), but values wrong
   - Possible cause: rewards not included in distribution
2. **Conway governance PP parameter updates only partially applied** — protocol version enactment works,
   but non-HardFork PP changes (executionUnits, committeeMinSize) not being applied correctly
3. **`query tip` returns zeros immediately after startup** — race condition (wait for replay to complete)
4. **`query stake-pools` garbled data** — CLI decoder mismatch
   - File: `crates/torsten-cli/src/commands/query.rs` lines 821-910

## Working Features Confirmed (2026-03-10, commits ae0cc46/55b0a0c/b72a44a, full 4M replay)
- Build: WORKS — clean, zero warnings
- Genesis UTxO seeding: WORKS — 1 UTxO seeded, grows to 2.9M during replay
- Peer connections: 5 peers established post-replay (13.58.19.0, 3.70.89.92, 99.80.240.19, 52.211.202.88, 3.74.40.92)
- Ledger replay: WORKS — no panics, 4.09M blocks in 251 seconds at 16,303 b/s
- Epoch transitions: WORKS (1,232 transitions logged)
- Pre-Conway PPUP: WORKS — v6→7 at epoch 3, v7→8 at epoch 22, v8→9 at epoch 646
- Conway governance HardFork: WORKS — v9→10 at epoch 743 via HardForkInitiation action
- Pool/delegation accumulation: WORKS (653 pools, 11,515 delegations)
- Governance ratification: WORKS — many proposals ratified and enacted throughout Conway era
- N2C query tip: WORKS — slot 106401541, block 4093131, epoch 1231
- N2C query protocol-parameters: WORKS — correct v10.0 returned
- Prometheus metrics: WORKS — sync_progress_percent: 10000 (100%)
- Rollback handling: WORKS (1 rollback observed, recovered cleanly)
- Snapshot saved after replay: WORKS (1,165MB)
- Live block stream: WORKS — steady block reception at chain tip (1927 blocks applied, 0 rejected)

## Prometheus Metrics (Preview after full replay, 2026-03-10, after protocol version fix)
- blocks_received_total: 1927, blocks_applied_total: 1927 (100% application rate)
- transactions_received_total: 1882, validated: 1882, rejected: 0
- peers_connected: 5, peers_cold: 3, peers_warm: 0, peers_hot: 5
- sync_progress_percent: 10000 (100.00%)
- slot_number: 106,464,614 (live, advancing)
- block_number: 4,095,058 (live, advancing)
- epoch_number: 1,232
- utxo_count: 2,936,107
- delegation_count: 11,515
- pool_count: 653
- treasury_lovelace: 42,293,664,776,351,527
- drep_count: 4, proposal_count: 2
- rollback_count_total: 1

## Operational Notes
- Always `--testnet-magic 2` with CLI query tip for correct syncProgress
- N2N port 3001 / Metrics port 12798 — conflict if old node running
- No `torsten-config.json` — use `config/preview-config.json` directly
- Delete ledger-snapshot.bin to trigger fresh replay with genesis UTxO seeding
- After genesis seeding fix: node IS functional at tip (blocks accepted)
- `TORSTEN_REPLAY_LIMIT=0` skips replay (old behavior, node works at tip but wrong ledger state)
- Snapshot grows to ~1.1GB with full UTxO state (was 83MB without it)
