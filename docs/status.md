# Darwin status — 2026-07-19

Live verification snapshot against Miden testnet + Sepolia. Complements
[architecture.md](./architecture.md) (what the system is) and
[architecture-spec.md](./architecture-spec.md) (the full spec) with **what is
actually shipped and running right now**.

## Live rail

The live product is the **confidential rail**: an ETH user bridges Sepolia USDC
to Miden via Epoch, funds a `confidential_deposit_note`, and the network drains
the collateral into a basket faucet and mints basket tokens **1:1** into a
private note only the depositor can open. Redeem is symmetric. Positions are
private token balances — there is no per-user ledger on-chain.

- **Frontend:** [darwin.market](https://darwin.market) — self-custody
  (browser-derived Miden wallet) and native-Miden-wallet deposit paths, both on
  the confidential rail.
- **Bridge:** Epoch (hosted allocator/solver), Sepolia ⇄ Miden. Replaced the
  earlier AggLayer/Bali path and the custodial relay wallet, both retired.
- **Mint integrity:** validated on-chain — honest deposit/redeem conserve
  exactly, non-dUSDC collateral is rejected, and the mint ratio is bound to the
  real drained collateral (no emitter-controlled fee/NAV lever).

## NAV & oracle

### Read-only NAV latency

Pragma testnet → cached server-side → off-chain `Σ weight × price` → frontend
badge.

```
n=30 calls   p50=13.4ms  p90=20.8ms  p99=24.0ms  min=7.7ms  max=24.0ms
30/30 under 200ms.
```

When a Pragma publisher is clearly broken (a testnet feed posting at the wrong
scale), the route flags the source and substitutes a public price feed **for
that pair only**; healthy pairs keep their Pragma value. No silent rescaling, no
on-chain caching.

### On-chain `compute_nav`

When NAV is needed *inside* a transaction, the controller executes the
`compute_nav` MASM proc, reading vault holdings + Pragma prices via foreign
account and doing felt division on-chain (~10 s, bounded by block time +
foreign-account state proof, not by Darwin). The live confidential deposit
itself mints 1:1 and does not depend on NAV.

## On-chain note execution (atomic-flow proofs)

The on-chain note-execution primitive — a note carrying an asset + math, consumed
and executed inside a controller's transaction context — is proven on testnet:

| Flow | Consuming tx | Block |
|---|---|---|
| Atomic deposit | [`0x2e211adf…`](https://testnet.midenscan.com/tx/0x2e211adf6f382749641b9e7324e89c85a0880238df29d154676377166ae856e2) | 703322 |
| Atomic redeem | [`0x005c4eec…`](https://testnet.midenscan.com/tx/0x005c4eec575800d251c12d84eeaa6cc1f2ffd98d090c291161f45e9e2e2a7800) | 777149 |

The note script runs `felt_div` on-chain, then drains the asset into the
controller vault. See [/flows](https://darwin.market/flows) for the full step
trace.

## On-chain anchors

### Miden testnet — confidential basket faucets (live rail)

| Faucet | ID |
|---|---|
| DCC | `0x9463767a994d6f9178fce256261430` |
| DAG | `0x2fe3469cccf61a710d321df38c4ca1` |
| DCO | `0xf1a4752b3689beb110eebec647df20` |
| dUSDC (Epoch-bridged) | `0xfc90f0f4da30e51168453b60eafed7` |
| Pragma oracle | `0xd0e1384e21a6350029d80128eb5c44` |

### Sepolia

| Asset | Address |
|---|---|
| USDC (bridged via Epoch) | `0x2BB4FfD7E2c6D432b697554Efd77fA13bdbefd69` |

## Repos

Public repos under `github.com/darwin-miden/*`. `next build` clean,
`tsc --noEmit` clean.

## Roadmap (not yet live)

- **Rebalancing** — drift math is implemented and unit-tested; the on-chain
  rebalance step is a placeholder, not a live flow.
- **Fresh confidential-rail tx capture** — the deposit/redeem proofs above are
  the atomic-note primitive; capturing the confidential-rail deposit/redeem tx
  hashes on the live faucets is pending.
- **Mainnet** — Miden mainnet deployment.
