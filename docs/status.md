# Darwin status — 2026-05-27

Live verification snapshot against Miden testnet + Sepolia + the
monorepo. This page complements [architecture.md](./architecture.md)
(what the system is) with **what is actually shipped and running
right now**.

## Headline

| Bloc | Done | Partial | External-blocked |
|---|---|---|---|
| **M1** | **6/6** ✅ | — | — (L2→L1 was *not* blocked — it just needs the explicit `claimAsset` trigger, now scripted + proven) |
| **M2** | **4/4** ✅ | Brian's 1Click outbound poller, NEAR canonical Miden listing | — |
| **M3** | **2/5** | — | 3 mainnet items (Miden mainnet Q3 2026), DNS darwin.xyz (operator), waitlist/launch (pre-launch) |

**No bloqueur remaining inside our scope.** The L2→L1 last-mile we believed was stuck on gateway-fm was a missing `claimAsset` call on our side; closed today with `darwin-infra/scripts/bali-l1-claim.sh`. Sepolia claim txs [`0xc5e6bc11…`](https://sepolia.etherscan.io/tx/0xc5e6bc113ea639a56897da8be3e7dc58b8013c458ee684aa77756d6e3fb0e3df) + [`0x826e1e16…`](https://sepolia.etherscan.io/tx/0x826e1e16349bf51f2ced344de02def3c26ca723468d2ccc32f66f24d94642ce1) verify end-to-end. Flow B real swap exec verified [`0x6ab60429…`](https://sepolia.etherscan.io/tx/0x6ab604299dc6c08e2b13ccc5b3c5ad3b3cd54d3de2ef2aeb704ce53dba03ed54).

### Full prod-ready demo path (added 2026-05-27 evening)

The redemption pipeline is now wired end-to-end through canonical
Bali instead of Brian's 1Click mock. Three pieces shipped this
afternoon:

- **`RedeemPanel`** on `/portfolio` — user clicks Redeem, the
  relay v2 worker creates a pending redemption row.
- **`darwin_relay_v2_worker` B2AGG outbound** — the worker burns
  basket tokens on the v6 controller, then emits a canonical
  `B2AggNote` from the relay wallet to the user's Sepolia EOA via
  the Bali bridge. No 1Click in the loop, no Brian-mock dependency.
  Selectable via `DARWIN_RELAY_V2_OUTBOUND_MODE` (default `b2agg`;
  set `bridge_out_v1` to keep the legacy path).
- **`BaliClaimPanel`** on `/portfolio` — user clicks "Claim on
  Sepolia" for any `ready_for_claim=true` deposit, the panel
  fetches the merkle proof, builds the calldata, and calls
  `claimAsset` via wagmi. The worker's `process_outbound_status_b2agg`
  poller then sees the new `claim_tx_hash` and updates
  `redemptions.sepolia_release_tx` so the lifecycle table
  reflects the L1 release.

The whole loop is therefore: **UI click → relay worker → on-chain
B2AGG → user UI claim → L1 release → UI lifecycle update**, all in
≤90 min wall time, all trustless past the user signing two
transactions.

### Live B2AGG round-trip — verified end-to-end (2026-05-27 evening)

The B2AGG-mode worker switch was put through a full real run this
evening. Single redemption (`31eedae6-…`, DAG basket, amount=100
in 8-decimal base units), all four expected tx hashes captured:

| Step | Wall clock | Tx |
|---|---|---|
| 1. Redemption submitted via `POST /v0/redeem` | t=0 | `redemption_id=31eedae6-1eb7-48f7-aa6e-37ea3f50b3ad` |
| 2. Worker burns basket on Miden (`atomic_redeem_note`) | t+~30s | `0xb11c0c44d15a55eaa2b8d8c15ed0aa9bb5148322d5a49bf90ecd295242b0bf2d` |
| 3. Worker emits canonical B2AGG note | t+~35s | `0x2d4343d6e0fc403d0641321856a6c338644800bec27ea9b122a644cb09fbfc5b` |
| 4. AggLayer cert settles (`ready_for_claim=true`) | t+~20 min | bridge service `deposit_cnt=9` |
| 5. User calls `claimAsset` on Sepolia | t+~22 min | [`0xbb4e2ff2b28ee2000757ff0a5048c5c443f631e1e9763500049e7beb39aafba4`](https://sepolia.etherscan.io/tx/0xbb4e2ff2b28ee2000757ff0a5048c5c443f631e1e9763500049e7beb39aafba4) |
| 6. Worker status poller writes `sepolia_release_tx` | t+~22.5 min | sqlite ✓ within next 30s tick |

Total wall time burn-to-credit: **~25 min** (the agglayer hourly
cadence is the dominant component; we caught a mid-epoch cert
which made this run faster than the typical 30-90 min window).

Two correctness bugs were caught + fixed during this verification
and are pinned in `v0.4.1-m3` on darwin-relay:

- decimal scaling Miden 8-dec → Sepolia 18-dec wei
- status poller false-positive cross-match with legacy 1Click
  redemptions of the same basket_amount

## Live numbers (verified 2026-05-27)

### Read-only target NAV (proposal "<200ms NAV view" target)

Pragma testnet → cached server-side (15 s warmer) → off-chain
`Σ weight × price` math → frontend badge.

```
n=30 calls   p50=13.4ms  p90=20.8ms  p99=24.0ms  min=7.7ms  max=24.0ms
30/30 calls under 200ms.
```

Pragma stays the source of truth. When a Pragma publisher is
clearly broken (USDT/USD on testnet currently posts at 1e6 scale),
the route flags the source as `pragma-miden+fallback` and
substitutes CoinGecko **for that pair only**. Healthy pairs keep
their Pragma value. No silent rescaling, no publisher relay, no
on-chain caching, no trust added.

### On-chain `compute_nav` (different path)

When the controller needs NAV *inside* a transaction (e.g. an
`apply_redeem` invocation), it executes the `compute_nav` MASM
proc, which reads vault holdings + Pragma prices via foreign
account + does felt division on-chain. End-to-end this is ~10 s,
dominated by the Miden block time + foreign-account state proof.
This is structurally bounded by the underlying chain, not by
Darwin. See [architecture.md §three-flows](./architecture.md) for
where this matters.

### Bali agglayer round-trip — fresh runs (2026-05-27)

Fresh L1↔L2 round-trip submitted to capture new evidence:

**L1 → L2 (Sepolia → Miden, 0.0005 ETH) — ✅ CLAIMED:**

- Sepolia tx [`0x814a3175adfe3fe396eb2641747d345bdfcf1ecfde9a2070bb748203a8e0782c`](https://sepolia.etherscan.io/tx/0x814a3175adfe3fe396eb2641747d345bdfcf1ecfde9a2070bb748203a8e0782c) block `10931760`
- Miden claim tx `0xf299cfbe5f839b05923e97c6adaf0c408ab823b056349c397b2327b7c50f5a67`
- Round-trip ~60 min (Sepolia submit → Bali aggsender GER push → solver mint)
- `deposit_cnt=1132975`, `global_index=18446744073710684591`
- Destination: relay wallet `0xed3cd5befa3207805f8529207cfc0d`
  (ETH-padded form `0x00000000ed3cd5befa3207805f8529207cfc0d00`)

**L2 → L1 (Miden → Sepolia) — ✅ END-TO-END PROVEN:**

The L2→L1 path requires three steps that we now have all
instrumented:

  1. Submit the B2AGG note on Miden via `gateway-fm/miden-agglayer`'s
     `bridge-out-tool` (we already had this).
  2. Wait for the agglayer certificate to settle (~30–90 min, hourly
     cadence). The bridge service's `/api/bridges/<dest>` flips
     `ready_for_claim` to `true` when ready.
  3. **Call `claimAsset` on the Sepolia bridge** with the merkle
     proof from `/api/merkle-proof?deposit_cnt=N&net_id=76`. The
     Bali stack does *not* auto-claim — the recipient (or any payer)
     must submit this final tx. This was the missing step in our
     prior pipeline: certificates were `ready_for_claim=true` for
     hours, the persistent watch correctly reported `claims=0`, and
     we mistakenly believed gateway-fm was holding the line.

Helper: [`darwin-infra/scripts/bali-l1-claim.sh`](https://github.com/darwin-miden/darwin-infra/blob/main/scripts/bali-l1-claim.sh)
takes `DEPOSIT_CNT` + `DEST_ADDR`, fetches the proof, builds the
SMT arrays, submits `claimAsset(bytes32[32], bytes32[32], uint256,
bytes32, bytes32, uint32, address, uint32, address, uint256, bytes)`.

Live evidence 2026-05-27:

| Step | Miden tx (B2AGG note) | Sepolia claim tx | Amount |
|---|---|---|---|
| deposit_cnt=7 (prior outbound) | `0x222e2015…` | [`0x826e1e16…`](https://sepolia.etherscan.io/tx/0x826e1e16349bf51f2ced344de02def3c26ca723468d2ccc32f66f24d94642ce1) | 0.0005 ETH |
| deposit_cnt=8 (fresh outbound) | `0x0b8c59ea…` | [`0xc5e6bc11…`](https://sepolia.etherscan.io/tx/0xc5e6bc113ea639a56897da8be3e7dc58b8013c458ee684aa77756d6e3fb0e3df) | 0.0005 ETH |

Both `status=1`, ClaimEvents emitted at the L1 bridge address, dev
key balance credited (minus gas) for each. Persistent watch exited
with `✓ claim landed` at iter 1202.

### Flow B swap leg — real exec on Sepolia (proof-of-execution)

An earlier audit flagged Flow B's swap exec as
"read-only quotes only". `darwin-sdk/rust/scripts/rebalance_exec_sepolia.sh`
now signs + submits the three-step leg (wrap → approve →
exactInputSingle) against Sepolia's canonical Uniswap V3
SwapRouter02. Verified live 2026-05-27:

```
wrap     0xeb679515cfaecd3623abe0f4d7b2cb69be9aa543ee2ef3b4c19b3fff251ee4d6
approve  0xbe0ca19f73fad0835eed765da490f1a91353b4890df6f8c83baea635668a9024
swap     0x6ab604299dc6c08e2b13ccc5b3c5ad3b3cd54d3de2ef2aeb704ce53dba03ed54
```

Input: 0.001 ETH → 0.004992 USDC (Sepolia pool is thinly liquid +
we use `amountOutMinimum=0`; economics are not real, this is
proof-of-execution only). The calldata shape is byte-for-byte what
the rebalance bot would emit in production; the only difference is
the trigger (manual run vs `DriftDetected` event) and the
`amountOutMinimum` guard, which would be set from a fresh quote.

Verify on
[Sepolia Etherscan](https://sepolia.etherscan.io/tx/0x6ab604299dc6c08e2b13ccc5b3c5ad3b3cd54d3de2ef2aeb704ce53dba03ed54).

### Stress throughput on the EVM-side relay (Sepolia)

100 deposits fired from a single EOA against `DarwinRelayDeposit` on
Sepolia via the canonical Sepolia public RPC, with nonces pre-pinned
so the 10 in-flight submitters actually parallelise instead of
colliding on nonce.

```
N=100  P=10  ok=99/100  wall=155s
submit_seconds  p50=12  p75=22  p90=24  p95=25  p99=25  min=9  max=25
```

Throughput: ~0.65 tx/s sustained from one EOA. The single failure
(idx=67) was a transient RPC hiccup; 99% success rate end-to-end.
Run captured 2026-05-27 in
[`darwin-relay/results/stress-scale-1779835339.tsv`](https://github.com/darwin-miden/darwin-relay/blob/main/results/stress-scale-1779835339.tsv).

### Relay observability

Relay v2 exposes `/metrics` in Prometheus text format on the same
axum listener as the REST API. Series:

- `darwin_relay_up` — 1 while alive
- `darwin_relay_intents_total{stage="…"}` — gauge per stage
- `darwin_relay_redemptions_total{stage="…"}` — gauge per stage
- `darwin_relay_positions` — distinct (user, basket) tracked

Counts are sqlite-derived at scrape time, so a restart never zeroes
the dashboard. A 6-panel Grafana JSON ships in
`darwin-relay/observability/`.

## On-chain anchors (verified live)

### Miden testnet — controllers

| Account | ID | Role |
|---|---|---|
| v6 controller | `0x2a3ea0a268d97b80497d6a966e3141` | Current default in worker + frontend. Strict superset of v5, adds slot 11 fee_recipient + `receive_and_credit` |
| v5 controller | `0x9419f2044acb77800a4c91a0cb50e5` | Per-user storage maps (slot 10) |
| v4 controller | `0x1975a9aa8572f8804fb38bee09fbdf` | `execute_rebalance_step` (Flow B) |
| v2 controller | `0xa25aa0b00007688024b74b05a52aab` | Real-bodies (preserves `receive_asset` MAST) |
| Relay/user wallet | `0xed3cd5befa3207805f8529207cfc0d` | Falcon-512 custodial |

### Miden testnet — faucets + oracle

| Account | ID | Role |
|---|---|---|
| DCC faucet | `0x2066f2da1f91ba202af5251d39101c` | Core Crypto basket |
| DAG faucet | `0xfb6811fd6399df206d44f62800620d` | Aggressive basket |
| DCO faucet | `0xbe4efc6729eb3220423b7d6d6a0942` | Conservative basket |
| dETH faucet | `0xa095d9b3831e96206ff70c2218a6a9` | Miden-native ETH proxy |
| Pragma oracle | `0xd0e1384e21a6350029d80128eb5c44` | `get_median` live |
| Bali bridge | `0xc98bb07c188cd2500e13f68a069cdc` | Canonical L1↔L2 (network 76) |

### Sepolia (ETH side)

| Contract | Address | Codesize |
|---|---|---|
| DarwinStrategy | `0x635E19c61CD09d145D57A88cE8185Ddf27fA356F` | 6383 |
| MockUSDC | `0x6dAb940a4E1d434965E22e9F6d624fF68F6922a0` | 1862 |
| DCC ERC20 | `0x1EB7Bd808402824232853e66DF6843D68462B7A4` | 5148 |
| DAG ERC20 | `0x73F18087dd45d180e75cADcD383479624326E336` | 5148 |
| DCO ERC20 | `0x6344469eB35Ff00d5892fD368727ad3C9E45677c` | 5148 |
| Bali bridge L1 | `0x1348947e282138d8f377b467F7D9c2EB0F335d1f` | 2583 |

## Repos + tags

11 public repos under `github.com/darwin-miden/*`, all `v0.4.0-m3` or later (darwin-infra latest: `v0.4.4-m3`)
tags present on remotes. `next build` clean, `tsc --noEmit` clean,
Playwright 9/9 against `localhost:3010`.

## What is still missing (controllable)

1. `darwin.xyz` DNS + production hosting — pending operator DNS
2. ~~Live stress-scale run (100 concurrent deposits)~~ — ✅ run 2026-05-27, 99/100 in 155s, see above
3. ~~Bali mock bridge live restart~~ — ✅ stack live on `:8080` (`/v0/tokens` answering)
4. ~~Real swap execution on Flow B~~ — ✅ executed 2026-05-27 (tx `0x6ab60429…` Sepolia), see above

## What is missing (external blockers)

- AggLayer L2→L1 first successful claim — gateway-fm fix in flight
- Miden mainnet launch — target Q3 2026
- NEAR Intent canonical Miden listing — when NEAR team adds it
- 1Click prod hosted endpoint — when shipped
