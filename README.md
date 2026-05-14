# Darwin Protocol — Documentation

Darwin is a **confidential basket protocol on Miden**. Users deposit underlying crypto assets and receive a single basket token whose Net Asset Value (NAV) tracks a weighted index. All basket operations run natively on Miden with client-side STARK proofs, so individual positions, balances, and NAV remain private by default.

## Contents

- [Getting Started — Darwin on Miden Testnet](docs/getting-started.md) — install the toolchain, create a Miden wallet, request testnet tokens from the faucet, bring up the local AggLayer stack, build and test every Darwin crate.
- [Milestone 1 Architecture Specification](docs/m1-architecture-spec.md) — the technical contract for the Miden core layer delivered under M1 of the Darwin x Miden grant: Private Execution Account model, three curated baskets (Core Crypto, Aggressive, Conservative), Pragma Oracle integration, AggLayer bridge integration via `miden-agglayer` v0.14-alpha, Flow A end-to-end on testnet.
- [Milestone 1 Test Report](docs/m1-test-report.md) — companion to the spec; records the test scenarios, on-chain commitments, proof timings, and outstanding items as M1 implementation progresses.

## Project layout

Darwin is organised as a multi-repo project under the [`darwin-miden`](https://github.com/darwin-miden) GitHub organisation:

| Repo | Purpose |
|---|---|
| [`darwin-protocol`](https://github.com/darwin-miden/darwin-protocol) | Core MASM: Darwin Protocol Account, basket faucets, asset faucets, note scripts |
| [`darwin-sdk`](https://github.com/darwin-miden/darwin-sdk) | Client SDK (Rust + TypeScript) wrapping `miden-client` and `miden-agglayer` |
| [`darwin-baskets`](https://github.com/darwin-miden/darwin-baskets) | Versioned basket manifests and Rust loader |
| [`darwin-oracle-adapter`](https://github.com/darwin-miden/darwin-oracle-adapter) | Pragma Oracle adapter and signed-attestation fallback |
| [`darwin-bridge-adapter`](https://github.com/darwin-miden/darwin-bridge-adapter) | AggLayer bridge integration: B2AGG / CLAIM note infrastructure, L1 wrapper ERC20 stubs |
| [`darwin-infra`](https://github.com/darwin-miden/darwin-infra) | Local dev stack, CI workflows, deployment scripts |
| [`darwin-docs`](https://github.com/darwin-miden/darwin-docs) | This documentation |
| [`darwin-frontend`](https://github.com/darwin-miden/darwin-frontend) | Frontend at darwin.xyz (M3 deliverable) |

## License

MIT.
