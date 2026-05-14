---
title: Milestone 1 Implementation Progress
version: 0.1.0
status: living document — updated as M1 work lands
last_updated: 2026-05-14
---

# Darwin Protocol — Milestone 1 Implementation Progress

A short, week-by-week log of what has actually landed against the [M1 Architecture Specification](m1-architecture-spec.md). Mirrors the in-flight state of the [`darwin-miden`](https://github.com/darwin-miden) repos so the Miden team and any external reviewer can audit progress at a glance without trawling commit history.

## Week 0 — 2026-05-14

**Scaffolding shipped.** Nine repositories under [darwin-miden](https://github.com/darwin-miden) created with MIT licence and READMEs (`darwin-protocol`, `darwin-sdk`, `darwin-baskets`, `darwin-oracle-adapter`, `darwin-bridge-adapter`, `darwin-infra`, `darwin-docs`, `darwin-frontend`, `.github`). No GitHub Actions / CI runs on the org by design — checks happen on the developer's machine before push.

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

**Local-only checks.** All `cargo fmt`, `cargo clippy -D warnings`, `cargo test`, and `forge test` are part of the developer loop and run cleanly on every commit. The repos do not run GitHub Actions — checks happen on the developer's machine before push.

**Helpers spread across repos.** Basket lookup by symbol and faucet-alias deduplication (`darwin-baskets`); fresh-quote validation (`darwin-oracle-adapter`); hex round-trip for `EthAddress` (`darwin-bridge-adapter`); a `check-toolchain.sh` sanity script (`darwin-infra`); a static basket catalogue (`darwin-frontend`).

**All four Darwin asset faucets deployed on public Miden testnet.** Every basket-constituent stand-in is now a real, public account on the Miden testnet:

| Asset | Account ID | Decimals | Deploying tx |
|---|---|---|---|
| dETH  | `0xa095d9b3831e96206ff70c2218a6a9` | 18 | `0xd2645c81…c3909e7` |
| dWBTC | `0x7a45cb24ada22120246bcf54196e12` | 8  | `0x33c2c024…19d72d99` |
| dUSDT | `0xd3789f451ddd4720602ba9eb1a268d` | 6  | `0x32cd61c2…cae0f90a` |
| dDAI  | `0xb526deb0408a29207e4f27ed57bf1a` | 18 | `0x2d534d2a…f7c8dfd0` |

Each was deployed via `miden client new-account --account-type fungible-faucet` against `~/.miden/packages/basic-fungible-faucet.masp` and activated on-chain by an initial mint to the Darwin team test wallet (`0x5230eb6eb7ba5c8…`), client-side STARK-proven. Tracked authoritatively in [`darwin-baskets/state/testnet.toml`](https://github.com/darwin-miden/darwin-baskets/blob/main/state/testnet.toml). The faucet account IDs are now the canonical `FaucetId`s the SDK uses to construct deposit notes.

This is the proof-of-pipeline-end-to-end for M1. Every subsequent on-chain artefact (the three protocol accounts, the basket-token faucets, the oracle adapter, the bridge wrappers) follows the same deployment path. The next gate is the M1 spec §5 protocol accounts, blocked on the `miden-objects` / `miden-assembly` version skew documented in `darwin-protocol/src/component.rs`.

**Three basket-token faucets deployed (DCC, DAG, DCO).** Every M1 basket token now has a public on-chain account:

| Basket | Faucet account ID | Decimals | Deploying tx |
|---|---|---|---|
| DCC | `0x2066f2da1f91ba202af5251d39101c` | 8 | `0x8da73c53…ed94843e15` |
| DAG | `0xfb6811fd6399df206d44f62800620d` | 8 | `0x420d8bda…319367fbf5d` |
| DCO | `0xbe4efc6729eb3220423b7d6d6a0942` | 8 | `0x9f2cfef3…50d7747906` |

Currently owned by the Darwin team's signing key; ownership transfers to the corresponding protocol account once the version-skew unblocks (`miden::standards::access::ownable_two_step`).

**Solidity wrapper (`WrappedBasketToken.sol`) compiles + tests under Foundry.** Eight passing tests (`forge test`) cover the bridge-only mint/burn surface, metadata fields, and the mint→burn round-trip. Ready to deploy on the local Anvil L1 from `darwin-infra` or on any AggLayer-connected testnet.

**Pragma adapter Rust expanded.** New `darwin_oracle_adapter::pragma` module ships the live testnet feed list (`SUPPORTED_PAIRS`), the alias→pair mapping (`pragma_pair_for_alias`), a deterministic felt-id encoder (`pair_id_felt`), and a snapshot of the current testnet Pragma oracle ID. Four new unit tests, eight total in the crate.

**`AccountComponent` compiles via the v0.19 path.** A self-contained `asm/controller_v0_19.masm` controller source compiles cleanly through `miden_objects::assembly::Assembler` (transitively `miden-assembly` 0.19) and yields an `AccountComponent` marked as `RegularAccountImmutableCode`-supporting. Procedure bodies are stubs (`push.0`) but the component is wire-compatible with `miden-client`'s `AccountBuilder` and demonstrates the deployment shape. The `deploy_m1` binary now exercises this for each of DCC / DAG / DCO and prints the green ticks.

**🎯 Three Darwin Protocol Accounts deployed on public Miden testnet.** A new Rust→Miden crate (`darwin-protocol/crates/darwin-controller-pkg`) ships the `DarwinBasketController` as an `#[component]`-annotated Rust struct. `cargo miden build` lowers it through the Miden Wasm→MASM pipeline into a `.masp` package, and `miden client new-account` deploys one account per basket:

| Basket | Protocol-account ID |
|---|---|
| DCC | `0xaa20da7d98c2e29022510aa786948f` |
| DAG | `0x53c54781b7b091905a948b5e3f92fe` |
| DCO | `0xa3a0e023381d709060a19527e73f95` |

`RegularAccountUpdatableCode`, private storage, Falcon-512 auth, controlled by the Darwin team's local keystore. The eight controller procedures (spec §5.3) are exported with placeholder bodies — they let the account compile, deploy, and accept transactions, while the real basket logic in the v0.23 `darwin::*` math libraries is plugged in once the ecosystem version-skew unblocks.

The Darwin Protocol Accounts join the seven faucets already on-chain. **10 Darwin accounts are now live on public Miden testnet** — four asset faucets, three basket-token faucets, three protocol controllers — recorded in [`darwin-baskets/state/testnet.toml`](https://github.com/darwin-miden/darwin-baskets/blob/main/state/testnet.toml).

**Flow A user side bootstrapped.** A second wallet — a public-storage `RegularAccount` at `0xed3cd5befa3207805f8529207cfc0d` — simulates an end-user inside Flow A. To put real basket-constituent assets into its vault, two on-chain steps ran:

1. The Darwin team's wallet P2ID-transferred `1_000_000` base units of the public Miden faucet token to the user (`tx 0xfe0a531e…116bea`).
2. The Darwin `dETH` faucet minted `5_000` base units directly into the user wallet (`tx 0xdf09fbe2…2fec095`, output note `0xa35ab164…14d9ff3`).

After that mint the user holds enough dETH to drive the wallet → controller deposit path once the protocol-account bodies wire in.

**Reproducible deploy recipe shipped.** [`darwin-infra/scripts/deploy-testnet.sh`](https://github.com/darwin-miden/darwin-infra/blob/main/scripts/deploy-testnet.sh) prints — or, with `--execute`, replays — the exact `miden new-wallet` / `new-faucet` / `mint` / `send` sequence that materialised every account in `darwin-baskets/state/testnet.toml`. A grant reviewer can re-run it against the same testnet RPC with their own signing key and get the same 10-account topology.

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

The first build is slow because the Miden toolchain is large; subsequent builds reuse the local `target/` cache.
