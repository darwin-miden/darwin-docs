# Architecture

Darwin is Miden-first. All basket operations run natively on Miden;
the EVM side is a thin onboarding/offboarding layer.

## Components

```
┌─────────────────┐           ┌─────────────────────────────┐
│  ETH user       │ 1Click /  │  Bali agglayer (canonical)  │
│  (Sepolia)      │   B2AGG   │  + 1Click mock (dev)        │
└────────┬────────┘  ─────►   └─────────────┬───────────────┘
         │                                  │
         │                                  ▼
         │                       ┌──────────────────────────┐
         │                       │  relay v2 (Miden side)   │
         │                       │  Falcon-512 custodial    │
         │                       │  axum REST + worker      │
         │                       └─────────────┬────────────┘
         │                                     │
         ▼                                     ▼
┌─────────────────┐                ┌────────────────────────┐
│  user wallet    │ atomic notes   │  v6 controller         │
│  (Miden-native) │ ─────────────► │  (basket logic +       │
│                 │                │  storage maps +        │
└─────────────────┘                │  fee recipient)        │
                                   └────────────┬───────────┘
                                                │
                                                ▼
                                   ┌────────────────────────┐
                                   │  basket-token faucets  │
                                   │  DCC / DAG / DCO / DPP │
                                   └────────────────────────┘
```

## Account types on Miden

| Account | Type | Role |
|---|---|---|
| User wallet | `RegularAccountUpdatableCode`, Falcon-512 | Holds basket-tokens (Miden-native users) |
| Relay wallet | `RegularAccountUpdatableCode`, Falcon-512 | Custodial holder for ETH-native users |
| v6 controller | `RegularAccountImmutableCode` | Basket math + per-user position ledger + fee recipient pointer |
| Basket-token faucet | `FungibleFaucet` | Mints/burns DCC, DAG, DCO, DPP |
| Pragma oracle | `RegularAccountImmutableCode` | Live price feeds (foreign-account reads) |

## v6 controller storage layout

```
slot  0  VERSION                  value
slot  1  BASKET_FAUCET_ID         value
slot  2  pool_positions           map[faucet_id → u64 amount]
slot  3  target_weights           map[basket_id → bps_word]
slot  4  fees                     map[basket_id → bps_word]
slot 10  user_positions           map[(user_id ‖ basket_id) → amt_word]
slot 11  fee_recipient            value(account_id_word)
```

## Three flows

- **Flow A — Deposit & Mint**: user wallet emits an atomic_deposit_note
  carrying the asset. v6 controller consumes it; receive_asset moves
  the asset into the controller vault and set_user_position writes the
  user's basket-token credit into slot 10. v3 of the note fuses both
  into a single `call.X` via `receive_and_credit`.

- **Flow B — Rebalance**: a trigger note is emitted when basket weights
  drift past threshold. The v4-or-newer controller consumes it and
  runs execute_rebalance_step on-chain. Swap leg currently uses
  Uniswap V3 / Paraswap as cross-chain SDK fallback; the in-protocol
  Miden DEX is the long-run target.

- **Flow C — Redeem**: atomic_redeem_note burns basket tokens at the
  controller. Pro-rata constituent release is computed via Pragma
  on-chain (or the CoinGecko fallback). For ETH-side users, the
  released underlyings travel back via Bali agglayer (B2AGG note →
  Sepolia release).

See `relay-v2-spec.md` for the relay's Miden-side custodial design
and `bali-integration.md` for the canonical L1↔L2 bridge.
