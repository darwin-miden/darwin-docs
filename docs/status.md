# Darwin status — 2026-05-27

Live verification snapshot against Miden testnet + Sepolia + the
monorepo. This page complements [architecture.md](./architecture.md)
(what the system is) with **what is actually shipped and running
right now**.

## Headline

| Bloc | Done | Partial | External-blocked |
|---|---|---|---|
| **M1** | **6/6** | — | AggLayer L2→L1 (gateway-fm fix in flight) |
| **M2** | **4/4** | Real swap exec, NEAR canonical Miden listing | — |
| **M3** | **2/5** | — | 3 mainnet items, waitlist/launch (pre-launch) |

The 3 ❌ M3 items are 100% gated on Miden mainnet (target Q3 2026).

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

11 public repos under `github.com/darwin-miden/*`, all `v0.3.0-m3`
tags present on remotes. `next build` clean, `tsc --noEmit` clean,
Playwright 9/9 against `localhost:3010`.

## What is still missing (controllable)

1. `darwin.xyz` DNS + production hosting — pending operator DNS
2. ~~Live stress-scale run (100 concurrent deposits)~~ — ✅ run 2026-05-27, 99/100 in 155s, see above
3. ~~Bali mock bridge live restart~~ — ✅ stack live on `:8080` (`/v0/tokens` answering)
4. Real swap execution on Flow B — requires funded testnet positions

## What is missing (external blockers)

- AggLayer L2→L1 first successful claim — gateway-fm fix in flight
- Miden mainnet launch — target Q3 2026
- NEAR Intent canonical Miden listing — when NEAR team adds it
- 1Click prod hosted endpoint — when shipped
