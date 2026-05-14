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
| Miden testnet | Block height ≥ 695354 at first deployment (2026-05-14) |
| Miden node | Public testnet, default endpoint |
| `miden-client` version | 0.14.8 |
| `miden-vm` / `miden-assembly` / `miden-core-lib` | 0.23 (used in the Darwin libraries) |
| `miden-objects` / `miden-tx` | 0.12 / 0.14 (used by the deployment binary; version skew to v0.23 line documented in `darwin-protocol/src/component.rs`) |
| `miden-agglayer` commit (`next` branch) | v0.14-alpha (not yet bundled at test time) |
| Pragma oracle account id on Miden testnet | `0xec7e450b91bf690015ad79573689f1` (snapshot — adapter resolves dynamically) |
| `dETH` faucet | `0xa095d9b3831e96206ff70c2218a6a9` |
| `dWBTC` faucet | `0x7a45cb24ada22120246bcf54196e12` |
| `dUSDT` faucet | `0xd3789f451ddd4720602ba9eb1a268d` |
| `dDAI` faucet | `0xb526deb0408a29207e4f27ed57bf1a` |
| `DCC` basket-token faucet | `0x2066f2da1f91ba202af5251d39101c` |
| `DAG` basket-token faucet | `0xfb6811fd6399df206d44f62800620d` |
| `DCO` basket-token faucet | `0xbe4efc6729eb3220423b7d6d6a0942` |
| Darwin Protocol Account (DCC / DAG / DCO) | Not yet deployed (blocked on version skew) |
| Oracle adapter account id | Not yet deployed |
| AggLayer bridge live on public testnet during M1 window? | Not confirmed at time of writing — Darwin team contact pending |
| Test runner machine | Apple-M2 MacBook, macOS 26.3, Rust 1.95.0 |

## 2. Unit tests (MASM + Rust)

### 2.1 `darwin-protocol` crates — MASM math libraries

All exercised via `miden-vm` 0.23 with the `miden-core-lib` u64-division event handler registered on the host. Run with `cargo test` in `darwin-protocol`.

| Library | Procedures | Tests | Status |
|---|---|---|---|
| `darwin::math` | `felt_div` (u64-safe via `miden::core::math::u64::div`) | 3 | ✅ green |
| `darwin::nav` | `weighted_sum_{2,3,4}`, `nav_per_share` | 6 | ✅ green |
| `darwin::mint` | `par_value`, `standard` | 9 | ✅ green |
| `darwin::fees` | `accrue_management`, `deduct_bps_fee` | 9 | ✅ green |
| `darwin::redeem` | `redeem_value_usd`, `release_amount` | 9 | ✅ green |
| `darwin::flow` | `mint_amount_for_{2,3,4}_asset_deposit`, `release_amount_for_constituent` | 7 | ✅ green |
| `round_trip_masm` | deposit→redeem invariants across the libraries | 3 | ✅ green |
| **MASM total** | | **46** | ✅ |
| `DarwinBasketController::compute_nav` | `darwin-protocol-account` | TODO | Pending account-context test harness |
| `DarwinBasketController::apply_deposit` | `darwin-protocol-account` | TODO | Pending account-context test harness |
| `DarwinBasketController::apply_redeem` | `darwin-protocol-account` | TODO | Pending account-context test harness |
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

### 2.3 Bridge adapter (Rust + Solidity)

Rust side: 5 tests, all green. `eth_address_round_trip`, `eth_address_hex_round_trip`, `eth_address_parse_rejects_malformed`, `b2agg_builder_requires_asset_and_destination`, `b2agg_builder_happy_path`.

Solidity side: 8 Foundry tests, all green. Run with `forge test`:

| Test | Asserts |
|---|---|
| `test_initial_supply_is_zero` | newly deployed wrapper has zero total supply |
| `test_owner_is_bridge` | constructor sets the AggLayer bridge as the Ownable owner |
| `test_metadata_records_miden_origin` | `midenOriginToken`, `midenNetworkId`, `name`, `symbol` round-trip |
| `test_only_bridge_can_mint` | non-owner `mint` reverts |
| `test_bridge_can_mint_to_user` | happy path mint credits balance + supply |
| `test_bridge_can_burn_from_user` | bridge can debit a balance and shrink supply |
| `test_user_cannot_burn_their_own` | non-owner `burnFrom` reverts |
| `test_mint_then_burn_round_trip_preserves_zero_supply` | net-zero invariant |

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

Done so far:

| Scenario | Status | Tx hash |
|---|---|---|
| Deploy `dETH` FungibleFaucet | ✅ | `0xd2645c81130aafea22c638ef35833b72c2960d8d05845b584ee9dc294c3909e7` |
| Deploy `dWBTC` FungibleFaucet | ✅ | `0x33c2c0248d28f9caee2bcbc474146472a886c082b77986e3873ffa5019d72d99` |
| Deploy `dUSDT` FungibleFaucet | ✅ | `0x32cd61c2500c257e60a8026541e65208024e6b4345af5f949a681954cae0f90a` |
| Deploy `dDAI` FungibleFaucet | ✅ | `0x2d534d2aecc7bded638610b4456780e8bd43c6954b086e7aa0ed4ef0f7c8dfd0` |
| Deploy `DCC` basket-token faucet | ✅ | `0x8da73c534cf5802b7a0b30815492d74daab4a14f1ec967b37911c7ed94843e15` |
| Deploy `DAG` basket-token faucet | ✅ | `0x420d8bda3a81ca39d767fe858e8bb662d7ef8852d11fd7c5f0934319367fbf5d` |
| Deploy `DCO` basket-token faucet | ✅ | `0x9f2cfef38b0a8a29732ce5caf190e578b707e294d611e6f3f8919f50d7747906` |
| Receive faucet tokens from `faucet.testnet.miden.io` to the test wallet | ✅ | `0xaafc6cb005744e27b990888aafbe0b87c864a995f3bb8d555d541c3b1891458a` |
| Consume the faucet note locally (STARK-prove client-side) | ✅ | `0xda8eeee31bd901d71b70dc33a7ddac09d92b308eb6d3ab5aa46de332fbfda949` |

Pending:

| Scenario | Status |
|---|---|
| Deploy 3 Darwin Protocol Accounts on public testnet | Pending — version skew with miden-objects 0.12 |
| Deploy Pragma oracle adapter on public testnet | Pending |
| Mint DCC via public testnet `DepositNote` consumption | Pending |
| Redeem DCC via public testnet `RedeemNote` consumption | Pending |
| Same on DAG | Pending |
| Same on DCO | Pending |
| Deposit just before / after expiry | Pending |

> **Note on the sync bug.** Between the mint transactions above and writing this section, the public Miden testnet node started returning a protobuf wire-type decode error on `sync_transactions` (`AccountId.id: NoteMetadataHeader.sender: ...: invalid wire type: SixtyFourBit (expected LengthDelimited)`). The deploying transactions are visible in the explorer; the local client is unable to consume the mint notes until the node ships a fix. The deployments themselves are unaffected.

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
