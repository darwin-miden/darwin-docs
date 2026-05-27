# Darwin Protocol — Documentation

Darwin is a **confidential basket protocol on Miden**. Users deposit underlying crypto assets and receive a single basket token whose Net Asset Value tracks a weighted index. All basket operations run natively on Miden with client-side STARK proofs, so individual positions, balances, and NAV remain private by default.

The product surface lives at [darwin.xyz](https://darwin.xyz). This repo is the home of the longer-form internal notes that are too dense to belong on the site.

**Browse this documentation as a site:** [`darwin-miden.github.io/darwin-docs`](https://darwin-miden.github.io/darwin-docs/) — most reviewers should start with [demo.md](https://darwin-miden.github.io/darwin-docs/demo) (end-to-end UI walkthrough with live tx hashes) and [status.md](https://darwin-miden.github.io/darwin-docs/status) (verified state).

## Project layout

Darwin is organised as a multi-repo project under the [`darwin-miden`](https://github.com/darwin-miden) GitHub organisation:

| Repo | Purpose |
|---|---|
| [`darwin-protocol`](https://github.com/darwin-miden/darwin-protocol) | Core MASM: Darwin Protocol Account, basket faucets, asset faucets, note scripts |
| [`darwin-sdk`](https://github.com/darwin-miden/darwin-sdk) | Client SDK (Rust + TypeScript) wrapping `miden-client` and `miden-agglayer` |
| [`darwin-baskets`](https://github.com/darwin-miden/darwin-baskets) | Versioned basket manifests and Rust loader |
| [`darwin-oracle-adapter`](https://github.com/darwin-miden/darwin-oracle-adapter) | Pragma Oracle adapter and signed-attestation fallback |
| [`darwin-bridge-adapter`](https://github.com/darwin-miden/darwin-bridge-adapter) | AggLayer bridge integration: B2AGG / CLAIM note infrastructure, L1 wrapper ERC20s |
| [`darwin-relay`](https://github.com/darwin-miden/darwin-relay) | ETH-side escrow + Rust relay service that lets ETH users onboard without managing a Miden key |
| [`darwin-infra`](https://github.com/darwin-miden/darwin-infra) | Local dev stack, deployment scripts |
| [`darwin-docs`](https://github.com/darwin-miden/darwin-docs) | This documentation |
| [`darwin-frontend`](https://github.com/darwin-miden/darwin-frontend) | Frontend at darwin.xyz |

## License

MIT.
