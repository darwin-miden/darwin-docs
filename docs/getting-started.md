---
title: Getting Started — Darwin on Miden Testnet
version: 0.1.0
status: draft
last_updated: 2026-05-14
---

# Getting Started — Darwin on Miden Testnet

This guide walks through everything you need to install, set up a wallet, fund it from the public Miden testnet faucet, and exercise Darwin Protocol locally. It is the companion to the [M1 Architecture Specification](m1-architecture-spec.md).

Time budget: ~30 minutes on a clean machine.

---

## 1. Prerequisites

Install these first if you do not have them:

| Tool | Purpose | Install |
|---|---|---|
| Rust 1.90+ | Build every Darwin Rust crate, plus Miden | `curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs \| sh` |
| Foundry (`forge`, `cast`, `anvil`) | Local L1 chain + Solidity tooling | `curl -L https://foundry.paradigm.xyz \| bash && foundryup` |
| Docker + Docker Compose | Local Darwin / AggLayer stack | [docs.docker.com/get-docker](https://docs.docker.com/get-docker/) |
| Node 20+ | TypeScript SDK and frontend | [nodejs.org](https://nodejs.org/) or `nvm install 20` |
| `gh` CLI (optional) | Cloning private Darwin repos | `brew install gh` |

Then clone the org and use the shared installer script:

```bash
gh repo clone darwin-miden/darwin-infra
./darwin-infra/scripts/install-toolchain.sh
```

The installer pulls Foundry (if missing) and installs `midenup`, which in turn fetches the Miden Rust compiler and the `miden-client` CLI.

Verify:

```bash
midenup --version
miden --version
```

---

## 2. Create a Miden testnet wallet

The Miden client stores a local SQLite database plus a keystore. Pick a directory you control:

```bash
export MIDEN_STORE_DIR="$HOME/.miden"
mkdir -p "$MIDEN_STORE_DIR"
```

Create a new private account (this is just a regular Miden wallet — Darwin does *not* require users to deploy a Darwin account):

```bash
miden new-account \
    --store-dir "$MIDEN_STORE_DIR" \
    --storage-mode private \
    --auth single-sig
```

The CLI prints the new `AccountId`. Save it; you will use it as the recipient address for Darwin deposits and as the sender for redeems.

To list your wallets later:

```bash
miden list-accounts --store-dir "$MIDEN_STORE_DIR"
```

---

## 3. Fund the wallet from the testnet faucet

The public Miden testnet faucet at [faucet.testnet.miden.io](https://faucet.testnet.miden.io/) dispenses native testnet tokens. To request from the CLI:

```bash
miden faucet-request \
    --account <your-account-id> \
    --network testnet
```

Or visit `https://faucet.testnet.miden.io/` in a browser, paste your `AccountId`, and click *Request*.

Confirm your wallet received the tokens:

```bash
miden balance --account <your-account-id> --network testnet
```

You should see the public faucet's asset in the vault within a few blocks.

---

## 4. Bring up the local Darwin / AggLayer stack (optional)

To exercise the full bridge round-trip end-to-end, run the `gateway-fm/miden-agglayer` proxy locally alongside an Anvil L1 and a Miden devnet node:

```bash
cd darwin-infra
./scripts/up.sh
```

This first clones [`gateway-fm/miden-agglayer`](https://github.com/gateway-fm/miden-agglayer) into `external/miden-agglayer` and then runs `docker compose up --build`. The first build takes 5–10 min; subsequent runs are fast.

Once up:

- Bridge JSON-RPC at `http://localhost:8546` (looks like an EVM node, talks to Miden underneath)
- Health endpoint at `http://localhost:8546/health`
- Prometheus metrics at `http://localhost:8546/metrics`
- Miden node gRPC at `localhost:57291`
- Anvil L1 at `http://localhost:8545`

Tear down:

```bash
./scripts/down.sh
```

---

## 5. Build and test the Darwin Rust crates

Each Darwin crate is independently buildable:

```bash
# Basket manifests + loader
gh repo clone darwin-miden/darwin-baskets
cd darwin-baskets && cargo test && cd ..

# Pragma oracle adapter (Rust types + WIT + MASM scaffold)
gh repo clone darwin-miden/darwin-oracle-adapter
cd darwin-oracle-adapter && cargo test && cd ..

# AggLayer bridge adapter (B2AGG builder + claim recognition + Solidity stub)
gh repo clone darwin-miden/darwin-bridge-adapter
cd darwin-bridge-adapter && cargo test && cd ..

# Core protocol workspace (4 crates)
gh repo clone darwin-miden/darwin-protocol
cd darwin-protocol && cargo test && cd ..

# Client SDK
gh repo clone darwin-miden/darwin-sdk
cd darwin-sdk/rust && cargo test && cd ../..
```

All of these should be green today. They contain scaffolded Rust APIs and MASM stubs; the actual basket controller MASM bodies fill in during the M1 implementation phase.

---

## 6. Deploy Darwin to your testnet wallet

A reproducible deployment recipe lives at [`darwin-infra/scripts/deploy-testnet.sh`](https://github.com/darwin-miden/darwin-infra/blob/main/scripts/deploy-testnet.sh). It prints — and with `--execute` re-runs — every `miden` CLI call that materialised the 10+ Darwin accounts currently live on testnet:

```bash
git clone git@github.com:darwin-miden/darwin-infra.git
cd darwin-infra
./scripts/deploy-testnet.sh                # dry-run: print the recipe
./scripts/deploy-testnet.sh --execute      # actually deploy
```

To inspect the current public state without redeploying, the bundled testnet inventory parses `darwin-baskets/state/testnet.toml` and prints a single-screen summary:

```bash
git clone git@github.com:darwin-miden/darwin-baskets.git
cd darwin-baskets
cargo run --bin testnet_inventory
```

To see the off-chain rebalance planner output (M2 prep) for any of the three baskets:

```bash
git clone git@github.com:darwin-miden/darwin-sdk.git
cd darwin-sdk/rust
cargo run --bin rebalance_demo                       # all three at par
cargo run --bin rebalance_demo -- DCC --skew 2.0     # one basket, perturbed
```

---

## 7. Troubleshooting

### "transaction conflicts with current mempool state"

The local `miden-client` SQLite has drifted from the node. Either:

- Unlock all accounts: `miden unlock-accounts --store-dir "$MIDEN_STORE_DIR"`
- Reset the local store (keeps keys): `rm "$MIDEN_STORE_DIR"/store.sqlite3*` and let the next command re-sync.

### "failed to authenticate when downloading repository" (cargo)

Cargo cannot pull a private repo over HTTPS with `gh`'s OAuth token. Two workarounds:

- The Darwin crates use SSH git URLs by default. Make sure your SSH key is added to GitHub (`ssh -T git@github.com` should succeed).
- Enable `git-fetch-with-cli`: each Darwin crate ships a `.cargo/config.toml` with `[net] git-fetch-with-cli = true`, which delegates fetching to your local `git` binary.

### Pragma address rotated

`darwin-oracle-adapter` resolves the Pragma oracle address from its own storage at runtime (§8.2 of the spec). When Pragma rotates on testnet, the Darwin team submits an `update_pragma_address` admin transaction; user accounts are unaffected. If you're running locally and Pragma is unreachable, the adapter switches to the signed-attestation fallback.

---

## 8. What to read next

- [Milestone 1 Architecture Specification](m1-architecture-spec.md) — the technical contract for Darwin's Miden layer.
- [`darwin-baskets/README.md`](https://github.com/darwin-miden/darwin-baskets) — manifest schema, validation rules, and the M1 baskets.
- [`darwin-protocol/README.md`](https://github.com/darwin-miden/darwin-protocol) — workspace layout and the four core crates.
- The `miden-agglayer` v0.14-alpha [`SPEC.md`](https://github.com/0xMiden/protocol/tree/next/crates/miden-agglayer) — Miden's own AggLayer bridge specification.
