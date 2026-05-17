---
title: Darwin — confidential baskets, native to Miden (M2 launch draft)
status: draft for review before tweet thread + blog post
last_updated: 2026-05-17
---

# Launch announcement — draft for review

Three versions below: (1) the **tweet thread** for X, (2) the **short blog post** for `darwin.xyz/blog`, and (3) the **Miden ecosystem update** copy we can hand to the Miden Labs team if they want to mention Darwin in their channels.

Pick whichever subset reads best for the moment. They all share the same five claims:

- Live on Miden testnet for both deposit and redeem.
- ETH-side stack live on Sepolia, anyone can try.
- Relay walks deposits autonomously in ~65 s.
- Open source from day one (`github.com/darwin-miden`).
- M2 of the Miden grant complete, M1 + M2 status visible on `darwin.xyz/status`.

Hard rules (per the user's editorial style):

- No em-dashes (use commas or short sentences).
- No author names, no Claude / AI mentions.
- Plain English, run-on sentences are fine if they flow.
- Numbers where claims are made.

---

## (1) Tweet thread

> Darwin Protocol is live on Miden testnet.
>
> Confidential basket protocol. Client-side STARK proven. You hold the position, only you can see it.
>
> M1 and M2 of the Miden grant both shipped. Below is what works today and what you can poke at.

> 1/ The flow.
>
> An ETH user deposits USDC on Sepolia, gets a Darwin basket ERC20 (wDCC / wDAG / wDCO) back in 65 seconds. Behind the scenes a private STARK-proven position lives on Miden in a controller the relay drives.
>
> No Miden account needed, no Falcon-512 key, no proof generation in the browser.

> 2/ Three atomic flows on Miden testnet, all on-chain:
>
> — Flow A (deposit). Atomic note carries asset plus math plus drain loop. Consumed by the controller's receive_asset.
> — Flow B (rebalance). Zero-asset trigger note calls execute_rebalance_step on the v4 controller.
> — Flow C (redeem). Symmetric of A, the redeem note burns user shares into the controller.
>
> Tx hashes on darwin.xyz/flows.

> 3/ Live Pragma oracle read end to end.
>
> Darwin transactions read real prices from Pragma's deployed oracle on Miden testnet. ETH/USD around 2194 USD when last sampled, BTC/USD around 78k.
>
> See `cargo run -p darwin-protocol-account --features pragma-live --bin oracle_query_real`.

> 4/ The relay.
>
> ETH users without Miden keys onboard through darwin-relay, an escrow contract on Sepolia plus a Rust service. Five autonomous runs recorded, the last one settled in 65 seconds with no retries.
>
> 0x7e5279AD…7c93 on sepolia.etherscan.io.

> 5/ Open source.
>
> Every contract, every controller, every Rust crate. 290 green tests across the workspace.
>
> github.com/darwin-miden
> Live status: darwin.xyz/status
> Try a deposit: darwin.xyz/baskets/dcc
>
> Tagged v0.2.0-m2 across all repos.

---

## (2) Short blog post (darwin.xyz/blog/m2-shipped)

**Darwin M2 is live. Here's what shipped.**

Today we crossed the M2 milestone of the Darwin x Miden grant. Both M1 (Miden core, basket logic, oracle) and M2 (relay, rebalancing, full flows) are now visible on testnet, end to end.

The short version: an Ethereum-native user can deposit USDC on Sepolia and hold a Darwin basket position on Ethereum (wDCC, wDAG, wDCO) in about 65 seconds, without ever touching Miden directly. Behind that simple UX, the position lives privately on Miden in a controller the relay drives, and Pragma's on-chain oracle is what the protocol reads when it needs prices.

**What's live today**

The Miden side runs on testnet:

- Six controllers deployed, each `RegularAccountImmutableCode`, private storage, Falcon-512 auth.
- Three basket-token faucets (DCC, DAG, DCO) plus four asset faucets (dETH, dWBTC, dUSDT, dDAI).
- Live Pragma oracle integration. Darwin's `oracle_query_real` reads ETH/USD and BTC/USD end-to-end inside a Miden transaction.
- Three atomic flows on chain: deposit (Flow A), rebalance trigger (Flow B), redeem (Flow C). Tx hashes on `darwin.xyz/flows`.

The Ethereum side runs on Sepolia:

- DarwinStrategy registry plus three basket tokens (DCC, DAG, DCO).
- DarwinRelayDeposit escrow that the off-chain relay watches.
- MockUSDC as the deposit currency since Sepolia has no canonical USDC.

The relay autonomously caught five live deposit events on Sepolia. The most recent one walked the full state machine in 65 seconds with zero retries.

**What's substituted from the original grant**

The grant assumed Near Intents would expose Miden as a destination chain by now and that Miden Guardian would act as a relay primitive. Neither is the case today: Near Intents has no Miden listing, and Guardian is documented as a non-custodial state coordinator, never a fund mover.

Darwin built its own minimal relay to fill the gap. The day Near Intents adds Miden, we swap the relay for the canonical path with no on-chain change. The day the AggLayer public bridge ships on testnet, the same swap closes the ETH-roundtrip leg.

These are external dependencies, not blockers we can resolve. We've been transparent about them with the Miden team and documented them on `darwin.xyz/status`.

**What's next**

M3 is mainnet, frontend with Miden's web SDK for client-side proving, and public launch. Mainnet of every layer (Miden, Pragma, AggLayer) is still pending on those teams. Until then we keep polishing the relay, write the swap-execution leg for rebalancing, and let users try the full Sepolia flow on `darwin.xyz/baskets`.

**Where to look**

- Site: `darwin.xyz`
- Try a deposit: `darwin.xyz/baskets/dcc`
- Live status M1 + M2: `darwin.xyz/status`
- All Miden testnet accounts: `darwin.xyz/accounts`
- Flow A / B / C on-chain runs: `darwin.xyz/flows`
- Source: `github.com/darwin-miden`
- M2 audit doc: `github.com/darwin-miden/darwin-docs/blob/main/docs/m2-status.md`

Open source from day one. Tag `v0.2.0-m2` across every repo pins the shipped state.

---

## (3) Miden ecosystem update (for the Miden Labs team)

Hey, quick update for the ecosystem section if you want a Darwin entry.

Darwin Protocol just crossed M2 of the grant. ETH-side stack live on Sepolia, Miden-side controllers and Flow A / B / C all live on testnet, end to end. 290 green tests across the workspace, every contract and crate in `github.com/darwin-miden`, tag `v0.2.0-m2` everywhere.

Highlights for the ecosystem post:

- 19 Darwin-owned accounts running on Miden testnet, including the live Pragma read path (Darwin's `oracle_query_real` returns real ETH/USD and BTC/USD on testnet).
- An ETH-native user can deposit USDC on Sepolia and receive a Darwin basket ERC20 on Ethereum in 65 seconds. The autonomous relay walks the deposit through claim, bridge stub, Miden mint, ERC20 mint, confirm.
- All three atomic flows (Flow A deposit, Flow B rebalance trigger, Flow C redeem) on-chain on testnet, with tx hashes browsable on `darwin.xyz/flows`.

We're open about what's gated on external pieces (canonical AggLayer bridge, Near Intents Miden destination) and what we substituted in the meantime (Darwin-operated relay). Full M2 audit at `github.com/darwin-miden/darwin-docs/blob/main/docs/m2-status.md`.

Anything else you'd like us to include or trim, happy to adjust.

---

## Channels checklist (when ready to fire)

- [ ] Tweet thread on @darwin_protocol (5 tweets above, scheduled 9am CET / 3am PT to catch both timezones)
- [ ] Blog post live at `darwin.xyz/blog/m2-shipped`
- [ ] Forward Miden ecosystem update to the Miden Labs BD email
- [ ] Tag everyone in the Miden ecosystem who would care (Pragma, OpenZeppelin Guardian, gateway-fm)
- [ ] Update darwin.xyz/status pin if needed
- [ ] Pin the tweet thread for a week
- [ ] Save the call-to-action link `darwin.xyz/baskets/dcc` somewhere obvious in the Miden Discord
