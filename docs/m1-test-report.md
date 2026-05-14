---
title: Milestone 1 Test Report
version: 0.1.0
status: draft (skeleton — sections populated during M1 implementation)
last_updated: 2026-05-14
---

# Darwin Protocol — Milestone 1 Test Report

This document is the M1 deliverable companion to the [Architecture Specification](m1-architecture-spec.md). It records the test scenarios executed during the M1 implementation window, with concrete evidence (commit hashes, on-chain commitments, proof logs, latency measurements). Each section is initially a skeleton; rows are populated as the corresponding code lands and is exercised.

The report covers exactly the scope of M1 §2.2 / §11. M2 work (rebalancing, Near Intent / Miden Guardian, full Flow C UX for ETH users) has its own report.

---

## 1. Test environment

| Item | Value |
|---|---|
| Miden testnet | (TBD: testnet id, commit, RPC endpoint at time of test) |
| `miden-client` version | (TBD: should be 0.14.x) |
| `miden-agglayer` commit (`next` branch) | (TBD: pin) |
| Pragma oracle account id on Miden testnet | (TBD: snapshot value, dynamic resolution active) |
| Darwin Protocol Account (Core Crypto / DCC) | (TBD) |
| Darwin Protocol Account (Aggressive / DAG) | (TBD) |
| Darwin Protocol Account (Conservative / DCO) | (TBD) |
| `dETH` / `dWBTC` / `dUSDT` / `dDAI` faucet IDs | (TBD) |
| DCC / DAG / DCO faucet IDs | (TBD) |
| Oracle adapter account id | (TBD) |
| AggLayer bridge live on public testnet during M1 window? | (TBD: yes/no — drives §10.5 decision matrix branch) |
| Test runner machine | (TBD: e.g. "Apple M2 MacBook Pro, 16 GB RAM, macOS 26.0") |

## 2. Unit tests (MASM + Rust)

### 2.1 `darwin-protocol` crates

| Procedure / module | Crate | Status | Coverage notes |
|---|---|---|---|
| `DarwinBasketController::compute_nav` | `darwin-protocol-account` | TODO | |
| `DarwinBasketController::apply_deposit` | `darwin-protocol-account` | TODO | |
| `DarwinBasketController::apply_redeem` | `darwin-protocol-account` | TODO | |
| `DarwinBasketController::compute_mint_amount` | `darwin-protocol-account` | TODO | incl. par-value branch |
| `DarwinBasketController::compute_redeem_amount` | `darwin-protocol-account` | TODO | |
| `DarwinBasketController::accrue_management_fee` | `darwin-protocol-account` | TODO | lazy accrual invariant |
| `agglayer_faucet::asset_to_origin_asset` (DCC, DAG, DCO) | `darwin-basket-faucet` | TODO | scale + origin addr correctness |
| `basket faucet::burn` (DCC, DAG, DCO) | `darwin-basket-faucet` | TODO | only-owner check |
| `DepositNote` script | `darwin-notes` | TODO | |
| `RedeemNote` script | `darwin-notes` | TODO | |

### 2.2 Oracle adapter

| Procedure | Status | Notes |
|---|---|---|
| `read_price` happy path | TODO | |
| `read_price` stale → fallback path | TODO | |
| `update_pragma_address` admin gate | TODO | |
| Signed-attestation verification | TODO | |

### 2.3 Bridge adapter (Rust)

Status: implemented and green on the `693a1ea` commit of `darwin-bridge-adapter`. 3 tests:

- `eth_address_round_trip` — `EthAddress::from_bytes` round-trips through `as_bytes`.
- `b2agg_builder_requires_asset_and_destination` — builder returns `MissingAsset` when called without inputs.
- `b2agg_builder_happy_path` — builder produces a populated `B2AggBuild` with the expected fields.

## 3. Integration tests (Rust against local Miden devnet)

| Scenario | Status | Notes |
|---|---|---|
| Deploy `dETH`/`dWBTC`/`dUSDT`/`dDAI` faucets on local devnet | TODO | |
| Deploy oracle adapter (pointing at mock Pragma) | TODO | |
| Deploy DCC / DAG / DCO basket faucets | TODO | |
| Deploy DCC / DAG / DCO Darwin Protocol Accounts | TODO | |
| Single-asset deposit into DCC | TODO | |
| Three-asset deposit into DCC | TODO | |
| Two-asset deposit into DAG | TODO | |
| Four-asset deposit into DCO | TODO | |
| First-ever deposit (par-value branch) per basket | TODO | |
| Deposit during simulated Pragma outage → fallback | TODO | |
| Two deposits same user same block — lazy mgmt fee | TODO | |
| Two deposits different users — wallet-level privacy | TODO | |
| Deposit + Redeem round-trip per basket | TODO | round-trip invariant (modulo fees) |
| Redeem with 4-constituent (DCO) — bps rounding | TODO | |
| Redeem with insufficient balance | TODO | |

## 4. End-to-end tests on Miden public testnet

| Scenario | Status | Tx hashes | Proof time |
|---|---|---|---|
| Deploy 3 Darwin Protocol Accounts + faucets on public testnet | TODO | | |
| Mint DCC via public testnet `DepositNote` consumption | TODO | | |
| Redeem DCC via public testnet `RedeemNote` consumption | TODO | | |
| Same on DAG | TODO | | |
| Same on DCO | TODO | | |
| Deposit just before / after expiry | TODO | | |

## 5. AggLayer bridge tests

Per §10.5 decision matrix, the bridge end-to-end is run against either the canonical Miden testnet bridge (if live during the M1 window) or the `gateway-fm/miden-agglayer` docker-compose stack. The branch taken is recorded here.

| Scenario | Branch (canonical / local stack) | Status |
|---|---|---|
| B2AGG out of DCC | TODO | |
| B2AGG out of DAG | TODO | |
| B2AGG out of DCO | TODO | |
| B2AGG before basket-token registration → expected panic | TODO | |
| CLAIM in for `agglayer_faucet_eth` → wallet receives wETH | TODO | |
| CLAIM in followed by DepositNote into DCC | TODO | |
| CLAIM with stale GER → rejection | TODO | |
| Full L1 → Miden → DCC mint → B2AGG → L1 round-trip | TODO | |

## 6. Performance metrics

Measured on the machine recorded in §1.

| Metric | Target (spec §11.4) | Measured |
|---|---|---|
| Proof generation, single-asset deposit | < 10 s | TODO |
| Proof generation, three-asset deposit | < 15 s | TODO |
| In-circuit NAV computation cycles | establish baseline | TODO |
| Oracle adapter Pragma read latency | < 500 ms | TODO |

## 7. Privacy validation

Sanity checks for the spec's §3.3 privacy claims. Each check looks at what an outside observer could learn from on-chain commitments + public events alone.

| Check | Status | Notes |
|---|---|---|
| Protocol account state opaque on-chain (only commitment visible) | TODO | confirm via miden-node RPC |
| Deposit amounts not leaked via DepositNote consumption | TODO | |
| Per-user DCC balance not directly observable for `Private`-mode wallets | TODO | |
| Pool position deltas across blocks reveal no per-user info beyond aggregate | TODO | |
| Linkability stress test: two users depositing within the same block | TODO | |

## 8. Outstanding items at report close

(Filled at the end of M1 with anything that did not get exercised — for example, "DCC registration in AggLayer bridge not obtained; round-trip exercised against local stack only".)

---

## Appendix — Commit pins

| Repo | Commit | When |
|---|---|---|
| `darwin-protocol` | TODO | |
| `darwin-baskets` | TODO | |
| `darwin-oracle-adapter` | TODO | |
| `darwin-bridge-adapter` | TODO | |
| `darwin-sdk` | TODO | |
| `darwin-infra` | TODO | |
| `0xMiden/protocol` (miden-base / miden-agglayer source pin) | TODO | branch `next` |
| `0xMiden/miden-client` | TODO | |
| `gateway-fm/miden-agglayer` | TODO | |
| `astraly-labs/pragma-miden` | TODO | |
