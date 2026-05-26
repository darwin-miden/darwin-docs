# Darwin docs index

- [architecture.md](./architecture.md) — components, accounts, flows
- [contracts.md](./contracts.md) — live testnet + Sepolia contracts with addresses
- [baskets.md](./baskets.md) — DCC, DAG, DCO, DPP composition
- [bali-integration.md](./bali-integration.md) — canonical Sepolia↔Miden bridge
- [relay-v2-spec.md](./relay-v2-spec.md) — Miden-side custodial relay design

Each repo also ships its own `README.md` with usage + run instructions:

- [darwin-protocol](https://github.com/darwin-miden/darwin-protocol) — MASM controllers, atomic notes, build/deploy binaries
- [darwin-relay](https://github.com/darwin-miden/darwin-relay) — axum REST + on-chain worker
- [darwin-frontend](https://github.com/darwin-miden/darwin-frontend) — Next.js + Miden Web SDK
- [darwin-sdk](https://github.com/darwin-miden/darwin-sdk) — Rust/TS client wrappers
- [darwin-baskets](https://github.com/darwin-miden/darwin-baskets) — TOML manifests + loader
- [darwin-bridge-adapter](https://github.com/darwin-miden/darwin-bridge-adapter) — AggLayer L1 contracts
- [darwin-oracle-adapter](https://github.com/darwin-miden/darwin-oracle-adapter) — Pragma adapter + fallback
- [darwin-infra](https://github.com/darwin-miden/darwin-infra) — local Docker stack + deploy scripts
