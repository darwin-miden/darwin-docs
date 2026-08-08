# Darwin — Architecture Specification

Darwin is a confidential basket protocol native to Miden. A user holds a
STARK-proven basket position as a **private note** in a self-custodial Miden
wallet: the token balance lives in the user's private vault, with no per-user
ledger on-chain. Value enters and leaves through the Epoch bridge
(Sepolia ⇄ Miden); deposit and redeem notes are executed by the Miden network
itself, not a custodian.

This document specifies the live rail. It supersedes the earlier design that
used an AggLayer/Bali bridge and a custodial relay wallet — those are retired
(§10). Section numbers below are stable anchors referenced from the note
scripts and libraries.

## §1 Overview

Two entry paths converge on one confidential position: an ETH user bridges
Sepolia USDC via Epoch and lets the network execute the deposit, or a native
Miden-wallet user signs a deposit note directly. In both cases basket tokens are
minted, bound 1:1 to the real collateral, into a private note only the depositor
can open.

## §2 Domains

| Domain | Role |
|---|---|
| **Sepolia (EVM)** | Users hold USDC and sign the bridge intent. Test USDC `0x2BB4FfD7…` (18-dec); Epoch's Compact/Allocator contracts settle the bridge. |
| **Operator** | Builds the confidential notes with the native `miden-client` (the browser Web SDK cannot yet build a note targeting a network account) and serves the frontend. Never custodies funds or keys. |
| **Miden (testnet)** | Where positions live: the user's private wallet, the bridged dUSDC faucet, the basket faucets (network accounts), and the network transaction (NTX) builder that executes deposit/redeem notes. |

## §3 Confidentiality model

- **Private:** the minted payback note contents; the basket balance (a private
  vault, no per-user on-chain ledger); the backup plaintext (AES key derived
  in-browser, never sent to the server).
- **Public by construction:** the collateral-carrying deposit/redeem notes (a
  network account only consumes public notes), the basket faucet vault balances
  (aggregate AUM), and the Epoch bridge legs.

The design keeps the resulting position confidential while the entry/exit legs
are transparent — a deliberate trade of on-chain executability (network
accounts) against full transaction privacy. Hardening the entry/exit legs is
tracked separately.

## §4 Self-custody & keys

The Miden wallet is derived deterministically from one EVM signature (EIP-712,
network-independent domain, low-s canonicalized, EOA-guarded). The Falcon key is
held in the browser and never reaches the operator. An encrypted, browser-side
backup (AES-GCM, key derived from the same signature) lets a user restore the
wallet on a new device by re-signing; only ciphertext is stored.

## §5 Accounts & components

- **User wallet** — a private Miden account; positions are the private token
  balances in its vault.
- **Basket faucets** — one **network-account** fungible faucet per basket (DCC,
  DAG, DCO). See §6.6.
- **Bridged dUSDC faucet** — the Miden-side representation of Epoch-bridged USDC
  (6-dec).
- **NTX builder** — the Miden network executor; consumes network-tagged public
  notes (deposit/redeem) and mints/burns inside the network transaction.
- **Backup controller** — a public NoAuth controller whose slot-10 StorageMap
  stores the AES-encrypted wallet backup (ciphertext only).

## §6 Protocol logic

### §6.2 NAV precision & math

Fixed-point weighted-sum and per-share routines (`nav.masm`, `math.masm`) compute
a basket NAV from constituent prices and target weights. Integer division uses
`math::felt_div` (u64-safe) for single-operand quotients, and `math::mul_div` for
the NAV mint/redeem ratios — `shares = net·S/V` and `payout = s·V/S` — whose
intermediate product exceeds u64 for any real basket. A plain field multiply would
reduce that product mod the Goldilocks prime `p` and silently over-mint; `mul_div`
instead forms the **exact 128-bit product** (`u64::widening_mul`) and performs a
restoring 128÷64 long division whose running remainder and quotient stay in u64, so
no field wrap can corrupt the share count. Out-of-range inputs **revert** rather
than wrapping (divisor 0 or ≥ 2⁶³, quotient ≥ 2⁶⁴, result ≥ `p`). Unit-tested on
the Miden VM against a `u128` reference, including a sweep across the u64 boundary
and products that exceed it.

### §6.3 Mint

The mint is bound to the **real drained collateral**, not to any emitter-supplied
value (`mint.masm`, and the confidential deposit note). The note reads the single
asset actually carried, asserts its faucet is the dUSDC faucet, and mints that
exact amount **1:1**. Fee/NAV storage felts are ignored by the mint — an emitter
cannot inflate the ratio. Validated on-chain (honest deposit/redeem conserve
exactly; non-dUSDC collateral is rejected).

### §6.4 Fees

Fee routines (`fees.masm`: management fee prorated over time, mint/redeem fee)
exist as library code and are exercised on the fee-routing controller. The **live
confidential mint does not apply a fee** — it mints 1:1 with collateral (§6.3).
Fees are therefore a separate, off-mint capability, not part of the confidential
deposit path.

### §6.5 Redeem

The inverse of the mint: the payout equals the **real burned amount**, read
before the burn, and the released asset's faucet is asserted to be dUSDC
(`redeem.masm`, the confidential redeem note). Conserves value exactly.

### §6.6 Basket faucet

Each basket is a **network-account** fungible faucet. On consuming a deposit
note, the faucet drains the dUSDC collateral into its vault and mints basket
tokens into a private payback note (§7.1). The network account's allowlist pins
the exact note-script roots it will execute, so a tampered note yields a root the
network refuses.

## §7 Note scripts

### §7.1 Deposit note

`confidential_deposit_note` carries the user's dUSDC collateral and targets a
basket faucet as a `NetworkAccountTarget` (execution hint `Always`). On
consumption it drains the collateral into the faucet vault and mints basket
tokens into a **private** payback note addressed to the depositor. The
collateral note is **public** (a network account only consumes public notes);
the payback is **private**.

### §7.2 Redeem note

`confidential_redeem_note` is symmetric: it carries basket tokens back to the
faucet, which burns them and releases the underlying dUSDC into a **private**
payback note; the released dUSDC is bridged back to Sepolia via Epoch (§10).

## §8 Oracle & NAV

The oracle adapter (`adapter.masm`, `darwin-oracle-adapter`) reads a Pragma
testnet median through a foreign-account call (`get_median`), returning the
median on the VM stack; feeds cover the basket constituents (BTC, ETH, WBTC,
USDT, DAI). The NAV math (§6.2) consumes those prices with the target weights. A
fallback degrades gracefully when a feed is unavailable. NAV is used for display
and valuation; the confidential mint itself is 1:1 (§6.3), independent of NAV.

**On-chain live pricing (FPI).** A basket faucet can price its vault entirely
on-chain, with no trusted price keeper: `pragma_fpi::load_prices` reads the live
Pragma median for each constituent through a foreign-procedure call
(`execute_foreign_procedure`), and `price_oracle::compute_v_fpi` values the vault
holdings at those live prices. Because the prices are read on-chain, no off-chain
actor can push a value to mint or redeem at a manipulated NAV, and a stale or
untracked feed makes the note **revert** rather than pricing against a stale value.
This replaces the keeper-written price slot; the remaining storage input is the
converted-cash watermark (vault bookkeeping, not a price).

## §9 Flows

- **Flow A — deposit** (`flow.masm`): bridge/faucet dUSDC → confidential deposit
  note → network drains collateral + mints basket tokens 1:1 into a private note
  → the browser imports and consumes it. STARK-proven, executed by the network.
- **Flow C — redeem**: burn basket tokens → network releases dUSDC into a private
  note → bridged back to Sepolia via Epoch.

## §10 Bridge & on-ramp

Value moves between Sepolia and Miden through **Epoch** (a hosted
allocator/solver), chosen on Miden-team guidance and validated end to end on
testnet. It replaces the earlier AggLayer/Bali path and the custodial relay
wallet, both retired.

- **In (deposit):** the user signs a Compact deposit on Sepolia; Epoch's solver
  bridges and delivers dUSDC to the user's Miden wallet, which funds the deposit
  note.
- **Out (redeem):** the released dUSDC is bridged back to the user's Sepolia
  address via an Epoch intent.

Basket positions are confidential and native to Miden, so basket tokens are
**not** exposed as ERC-20s on Ethereum; only the underlying collateral crosses
the bridge.

## §13 Rebalancing

Drift primitives (`drift.masm`: constituent weight, absolute drift, threshold
test) are implemented and unit-tested, and a rebalance-trigger note exists. A
full rebalancing **engine** — reading live weights, computing per-constituent
deltas, and routing swaps — is not part of the live confidential rail; the
on-chain `execute_rebalance_step` is a placeholder. Rebalancing is a roadmap
item, not a shipped flow.
