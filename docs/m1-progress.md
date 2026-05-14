---
title: Milestone 1 Implementation Progress
version: 0.1.0
status: living document — updated as M1 work lands
last_updated: 2026-05-14
---

# Darwin Protocol — Milestone 1 Implementation Progress

A short, week-by-week log of what has actually landed against the [M1 Architecture Specification](m1-architecture-spec.md). Mirrors the in-flight state of the [`darwin-miden`](https://github.com/darwin-miden) repos so the Miden team and any external reviewer can audit progress at a glance without trawling commit history.

## Week 0 — 2026-05-14

**Scaffolding shipped.** Nine repositories under [darwin-miden](https://github.com/darwin-miden) created with MIT licence, CI workflows, and READMEs (`darwin-protocol`, `darwin-sdk`, `darwin-baskets`, `darwin-oracle-adapter`, `darwin-bridge-adapter`, `darwin-infra`, `darwin-docs`, `darwin-frontend`, `.github`).

**Specs delivered.** Full M1 architecture spec, test report skeleton, and getting-started guide live on this repo.

**Toolchain bootstrapped.** `midenup` + Miden v0.14 stable toolchain installed and verified end-to-end on a developer machine: created a private testnet wallet, requested 100 tokens from the public faucet, consumed the resulting note via a local STARK proof, balance reconciled on the testnet (txs `0xaafc6cb0…1891458a` and `0xda8eeee3…fbfda949`).

**MASM libraries shipped.** Six library modules under `darwin-protocol/asm/lib/` totalling ~250 lines of MASM v0.23:

| Library | Purpose | Tests |
|---|---|---|
| `darwin::math` | u64-safe `felt_div` wrapping `miden::core::math::u64::div` | 3 |
| `darwin::nav` | weighted-sum primitives + `nav_per_share` | 6 |
| `darwin::mint` | par-value and standard mint formulas | 9 |
| `darwin::fees` | streamed management-fee accrual + bps-fee deduction | 9 |
| `darwin::redeem` | redeem-value + per-constituent release | 9 |
| `darwin::flow` | higher-level Flow A / Flow C wrappers composing the above | 7 |

**Build pipeline.** `build.rs` parses every MASM module under `asm/lib/` and emits two artefacts (`darwin-primitives.masl`, `darwin-flow.masl`) to `$OUT_DIR`, which the crate `include_bytes!`s. Public helpers `primitives_library()` and `flow_library()` return runtime `Library` handles that downstream tests and the deployment binary can attach to any `Assembler`.

**Test coverage.** 60+ tests across the workspace, all green: ~46 are real MASM executed through `miden-vm` 0.23 with the `miden-core-lib` u64-division event handler registered on the host. `cargo clippy -D warnings` clean across every crate.

**Math libraries' `darwin-baskets` integration.** All three M1 baskets (DCC, DAG, DCO) drive their procedure calls from the manifests in `darwin-baskets/manifests/*.toml`, validated by the basket loader.

**Round-trip invariants.** A new `tests/round_trip_masm.rs` integration suite asserts the deposit-then-redeem identity at par-state (zero fees) and the expected ~30 bps loss when both legs charge the standard fee. Validates that the mint and redeem libraries compose correctly across their MASM boundaries.

**SDK request validation.** `darwin-sdk`'s `DepositRequest` now carries the user's recipient wallet, distinguishes resolved-vs-unresolved basket handles (cargo `with_protocol_account` / `with_basket_token_faucet` setters), and refuses placeholder or duplicate asset entries before constructing a note. Mirrored by `darwin-notes` which exposes the on-chain `DepositNoteInputs` / `RedeemNoteInputs` shapes.

**CI everywhere.** Every Rust repo (`darwin-baskets`, `darwin-oracle-adapter`, `darwin-bridge-adapter`, `darwin-protocol`, `darwin-sdk`) has GitHub Actions running fmt + clippy + test on every push and PR. `darwin-infra` lints shell scripts and validates the docker-compose. `darwin-frontend` type-checks and lints. Dependabot is enabled across all repos.

**Helpers spread across repos.** Basket lookup by symbol and faucet-alias deduplication (`darwin-baskets`); fresh-quote validation (`darwin-oracle-adapter`); hex round-trip for `EthAddress` (`darwin-bridge-adapter`); a `check-toolchain.sh` sanity script (`darwin-infra`); a static basket catalogue (`darwin-frontend`).

**First on-chain artefact: `dETH` faucet deployed on public Miden testnet.** The first Darwin component to live on a public chain: a `FungibleFaucet` for `DETH` at account id `0xa095d9b3831e96206ff70c2218a6a9`, deployed via the `miden client new-account --account-type fungible-faucet` flow against `~/.miden/packages/basic-fungible-faucet.masp`. The deploying transaction (`0xd2645c81…c3909e7`) is client-side STARK-proven, submitted to the testnet node, and 1_000_000 base units were minted to the Darwin team test wallet (`0x5230eb6eb7ba5c8…`). Tracked authoritatively in `darwin-baskets/state/testnet.toml`. This is the proof-of-pipeline-end-to-end for M1 — every subsequent on-chain artefact (DAG / DCO faucets, protocol accounts, oracle adapter, bridge components) follows the same deployment path.

## What is *not* shipped yet

In rough order of expected delivery:

- **`AccountComponent` integration.** Blocked on the ecosystem-wide version skew between `miden-objects` 0.12 (pinned to `miden-assembly` 0.19) and the Darwin libraries which are assembled with `miden-assembly` 0.23 to access `miden-core-lib`'s u64 division. `darwin-protocol/src/component.rs` documents this in detail. Unblocks when `miden-protocol` ships a release that bundles `miden-assembly` 0.23.
- **Storage reads in MASM.** The `compute_nav` procedure needs to read pool positions from the protocol account's slot 2 StorageMap. Once `AccountComponent` is wireable, the `miden-tx` test harness gives us an account-execution context where storage reads can be exercised end-to-end.
- **Cross-component calls to the oracle adapter.** Same dependency on the `miden-tx` harness.
- **`DepositNote` and `RedeemNote` script bundles.** The MASM bodies are sketched in `darwin-protocol/crates/darwin-notes/asm/`; consumable `NoteScript.fromPackage(.masp)` requires the same account-context wiring as the storage reads.
- **End-to-end Flow A on public testnet.** Requires the four items above plus a small amount of operator-side setup (deploy the four custom asset faucets, register them, mint test balances).
- **AggLayer bridge integration on public testnet.** Requires the bridge-admin coordination flagged in the spec's §10 decision matrix.

## How to verify

```bash
gh repo clone darwin-miden/darwin-protocol
cd darwin-protocol
cargo test                # 57 tests, all green
cargo clippy --all-targets --all-features -- -D warnings
```

The first build is slow (the Miden toolchain is large); subsequent builds use `Swatinem/rust-cache` in CI.
