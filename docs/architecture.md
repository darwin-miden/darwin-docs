# Architecture

Darwin is Miden-first. Basket positions are **confidential** — minted into
private notes in a self-custodial Miden wallet, with no per-user ledger
on-chain. The EVM side is a thin on/off-ramp via the Epoch bridge.

See [`architecture-spec.md`](./architecture-spec.md) for the full specification.

## Components

```
  ETH user (Sepolia)
        │  Epoch bridge (Sepolia USDC → dUSDC on Miden)
        ▼
  user's private Miden wallet  (Falcon-512, key in-browser)
        │  confidential_deposit_note  (carries dUSDC collateral, PUBLIC)
        ▼
  basket faucet  (network account)
        │  network drains collateral, mints basket tokens 1:1
        ▼
  private payback note  →  the user's confidential position
```

## Account types on Miden

| Account | Type | Role |
|---|---|---|
| User wallet | private, Falcon-512 | Holds the confidential basket position (a private token balance) |
| Basket faucet | network-account `FungibleFaucet` | Consumes the deposit note, drains the dUSDC collateral, mints basket tokens (DCC / DAG / DCO) into a private note |
| dUSDC faucet | `FungibleFaucet` | Miden-side representation of Epoch-bridged USDC (6-dec) |
| Backup controller | public NoAuth | Stores the AES-encrypted wallet backup (ciphertext only) |
| Pragma oracle | foreign account | Live median price reads for NAV |

## Positions

A position is the user's private token balance. There is **no per-user ledger
slot** on-chain — a confidentiality improvement over a storage-map ledger. The
balance is read client-side from the private vault.

## Flows

- **Deposit** — the user's dUSDC funds a `confidential_deposit_note` the network
  consumes; it drains the collateral into the basket faucet and mints basket
  tokens **1:1** into a private note only the depositor can open.
- **Redeem** — a `confidential_redeem_note` burns basket tokens; the network
  releases the underlying dUSDC into a private note, bridged back to Sepolia via
  Epoch.

Rebalancing is a roadmap item, not a live flow: drift math is implemented and
unit-tested, but the on-chain rebalance step is a placeholder.

## Bridge

Value moves between Sepolia and Miden via **Epoch** (a hosted allocator/solver,
chosen on Miden-team guidance and validated end to end on testnet). It replaces
the earlier AggLayer/Bali path and the custodial relay wallet, both retired.
