# **Grant Proposal**

*Confidential Basket Protocol on Miden*

*Tom David* \-  Founder, Darwin Protocol  
tg: @dyonisos10  
March 2026  
Confidential \- For Miden team review only

## **Darwin on Miden \- Architecture Overview**

**Miden-First Architecture**

Miden is the core protocol layer: all basket operations (deposit, mint, redeem) run natively on Miden with client-side STARK proofs. Pragma Oracle provides token prices for private NAV computation. AggLayer (BridgeAsset only, 10–20 min) exposes basket tokens to public chains. ETH users without a Miden account interact via Near Intent and a relay wallet, with automatic withdrawal to their EVM wallet.

**Flow A \- Deposit & Mint Basket Tokens**  
Two deposit paths converge into a single Miden-native flow. Miden users deposit directly via a client-side STARK-proven DepositNote. ETH users route through Near Intent and a relay wallet, bridging assets to Miden via AggLayer (10–20 min). In both cases, the Transaction Kernel validates the proof, Pragma Oracle provides token prices (\~200ms), and basket tokens are minted natively on Miden as a private STARK-proven note.

**Flow B \- Automated Rebalancing via Near Intent**  
Rebalancing runs entirely on Miden via Near Intent — no ETH-side execution required. Darwin's private account detects portfolio drift, computes the buy/sell delta per token, and routes the swap through Near Intent. Execution happens on the in-protocol Miden DEX (available before mainnet) or via the cross-chain trading SDK as fallback. All positions remain private by default. An ETH-side liquidity pool is available for rebalancing, but stays private via Near Intent \+ AggLayer.

## **Flow C \- Redeem Basket Tokens → Underlying Assets**  User initiate redemption from their private Miden account. A RedeemNote is built and proved client-side (STARK). The Transaction Kernel validates the proof, the Private Account calculates the pro-rata share using Pragma Oracle prices, and the basket token is burned on Miden. Miden users receive assets directly on Miden (private). ETH users receive assets via AggLayer BridgeAsset (10–20 min), automatically withdrawn to their EVM wallet by the relay wallet. 

## 

## 

## 

## 

## **Grant Proposal:** 

## **Milestone 1 \- Miden Core Layer: Private Execution, Basket Token Logic & Oracle Integration \- $10,000**

### **Overview**

## This milestone establishes Darwin's core protocol layer on Miden testnet. Following the updated Miden-first architecture, all basket operations run natively on Miden. The focus is on implementing the Private Execution Account, basket token mint/burn logic on Miden, Pragma Oracle integration for private NAV computation, and AggLayer BridgeAsset connectivity to expose basket tokens on public chains. This delivers Flow A (Miden-native deposit path) end-to-end on testnet.

## *Note: Pragma Oracle's availability and capability on Miden testnet is a key dependency for this milestone. We will implement a fallback mechanism and factor oracle readiness into the timeline accordingly (per Miden team guidance).*

## Darwin will also define and deploy 3–5 curated baskets (Core Crypto, DeFi Blue Chips, Privacy Coins…) and validate the deposit/mint flow on testnet.

## **1 \- Private Execution Account**

* ## Implement `RegularAccountImmutableCode` on Miden: StorageMap for user positions, balances, NAV, pending operations

* ## Design and implement DepositNote and RedeemNote scripts using `NoteScript.fromPackage(.masp)`

* ## Implement Transaction Kernel validation logic: STARK proof verification, note consumption

* ## Client-side proof generation pipeline using miden-client v0.13

## **2 – Basket Token Logic (Miden-Native)**

* ## Implement mint/burn logic natively on Miden, basket tokens created and destroyed directly on Miden, no ETH dependency for core logic

* ## Implement NAV computation on Miden using Pragma Oracle prices

* ## Define token weights and basket composition rules stored in Private Execution Account StorageMap

* ## Deploy 3 curated baskets on Miden testnet: Core Crypto, DeFi Blue Chips, Privacy Pack

**3 – Pragma Oracle Integration**

* Integrate Pragma Oracle on Miden testnet: connect whitelisted publisher feeds (WETH/USD, WBTC/USD, USDC/USD)  
* Expose `get_price(pair)` via WIT bindings to Private Execution Account  
* Implement private NAV calculation circuit using oracle prices (\~200ms latency)  
* Implement fallback mechanism in case of oracle unavailability (contingency per Miden team feedback)  
* Validate oracle price feed reliability and latency under testnet conditions

**4 – AggLayer BridgeAsset Integration**

* Implement AggLayer BridgeAsset to expose Darwin basket tokens on Ethereum and public chains  
* Implement ETH → Miden and Miden → ETH flows via AggLayer BridgeAsset  
* Handle 10–20 min bridge latency in the deposit UX

## **Deliverables**

* ## Private Execution Account deployed on Miden testnet

* ## Basket tokens mintable and burnable natively on Miden (3 curated baskets)

* ## Pragma Oracle live on testnet with 3 token pairs (with fallback mechanism)

* ## AggLayer BridgeAsset functional: basket tokens accessible on Ethereum

* ## Flow A end-to-end on testnet: Miden-native deposit → basket token minted (private, STARK proven)

* ## Architecture specification document \+ test report

## **Timeframe:** 1,5 month

## **Milestone 2: Near Intent Integration, Relay Wallet & Rebalancing \- $8,000**

### **Overview**

## This milestone delivers ETH user access via Near Intent and a relay wallet (based on Miden Guardian), automated rebalancing via Near Intent and the in-protocol Miden DEX, and completes all three flows (A, B, C) end-to-end on testnet. It closes with a third-party security audit covering Miden account logic, STARK circuits, Near Intent interactions, and AggLayer bridge security.

### **1 – Near Intent & Relay Wallet (Miden Guardian)**

* ## Implement Darwin protocol contracts on ETH: basket token mint/burn logic, strategy storage (token list, target weights, rebalancing rules, fee structure), NAV reference

* ## Implement fee collection logic (management fee \+ mint/redeem fee)

* ## Deploy 3 curated baskets on ETH testnet with configurable weights

* ## Build client-side transaction logic: basket token operations, NAV calculation on-chain

### **2 – Rebalancing Engine (Near Intent \+ In-Protocol DEX)**

* ## Implement drift detection: compare current basket weights vs. target weights, trigger rebalancing when deviation exceeds threshold

* ## Integrate on-chain price oracle on ETH (Chainlink / Uniswap TWAP) for rebalancing computation

* ## Compute delta (buy/sell amounts per token), execute swaps via Uniswap / 1inch

* ## Slippage protection \+ time-weighted execution for low liquidity scenarios

* ## Stress test: high volatility (frequent rebalancing), low volatility (minimal drift), large deposit/withdrawal scenarios

**3 \- Full Flow Completion (A, B, C)**

* Implement Flow C: RedeemNote on Miden → STARK proof → Transaction Kernel → burn on Miden → AggLayer BridgeAsset back to ETH → Darwin contract distributes underlying assets  
* Full end-to-end tests: Flow A \+ Flow B \+ Flow C on testnet with proof verification logs

## **Deliverables**

* ## Near Intent \+ relay wallet (Miden Guardian) functional on testnet

* ## Rebalancing live on Miden: drift detection → Near Intent → DEX swap → portfolio updated

* ## Flow A (both paths), Flow B, Flow C fully functional on testnet

* ## Full end-to-end test report (Flows A \+ B \+ C)

## **Timeframe:** 1,5 month

## **Milestone 3: Frontend, Mainnet Deployment & Public Launch \- $7,000**

### **Overview**

## This milestone covers Darwin's mainnet deployment (targeting Q3 2026), frontend integration with Miden SDK for client-side proof generation, and the public launch with full Miden ecosystem visibility. The deliverable is a production-ready confidential basket protocol on Miden mainnet, accessible from day one to both Miden-native users and ETH users via the relay wallet.

### **1 \- Frontend & Miden SDK Integration**

* ## Build Darwin frontend at darwin.xyz: basket browser (strategy, weights, risk level, historical performance), deposit/redeem interface for both Miden-native and ETH users (relay wallet UX), portfolio dashboard, private NAV tracking

* ## Integrate Miden SDK: users generate STARK proofs locally for all transactions. No server trust.

* ## Implement secure note handling: DepositNote creation, RedeemNote consumption, fee note routing

* ## Support dual user experience: Miden wallet connection (native) \+ EVM wallet connection (via relay wallet / Near Intent)

* ## UX targets: time-to-deposit \< 30s from wallet connect, proof generation \< 10s on standard laptop

### **2 \- Mainnet Deployment**

* ## Deploy Darwin Private Execution Account on Miden mainnet (day one or within first week of mainnet launch)

* ## Activate Pragma Oracle mainnet feeds (WETH/USD, WBTC/USD, USDC/USD), contingent on oracle mainnet availability

* ## Deploy AggLayer BridgeAsset on mainnet (basket tokens live on ETH \+ public chains)

* ## Activate Near Intent on mainnet for ETH user path and rebalancing

* ## Migrate 3 curated baskets from testnet to mainnet

**3 \- Public Launch & Ecosystem Visibility**

* ## Convert testnet waitlist to mainnet depositors

* ## Launch announcement via Miden official channels \+ Darwin socials (X)

* ## Publish open-source codebase on GitHub

* ## Begin institutional outreach: crypto treasuries, fund managers, privacy-focused projects

### **Deliverables**

* ## Darwin V1 live on Miden mainnet with 3 curated baskets

* ## Frontend with Miden SDK (client-side proving, dual wallet — Miden native \+ EVM relay)

* ## Near Intent \+ relay wallet live on mainnet (ETH user access)

* ## Open-source codebase on GitHub

* ## Launch announcement \+ ecosystem visibility (Miden launch partner)

## **Timeframe:** 1 months

## 

**Darwin x Miden \- TVL Performance Program**

This program defines a set of milestone-based rewards tied to the TVL that Darwin brings to the Miden network. Rather than a flat royalty model, each threshold unlocks a one-time payment, clean, verifiable, and directly aligned with Darwin's real business impact on the ecosystem. Miden pays only upon confirmed, sustained TVL delivery.

| Milestone | TVL | One Time Rewards | Cumulative Total |
| :---- | :---- | :---- | :---- |
| **M1** | **$100,000** | **$1,000** | **$1,000** |
| **M2** | **$500,000** | **$2,000** | **$3,000** |
| **M3** | **$1M** | **$3,500** | **$6,500** |
| **M4** | **$2M** | **$5,000** | **$11,500** |
| **M5** | **$5M** | **$8,000** | **$19,500** |
| **M6** | **$10M** | **$12,000** | **$31,500** |
| **M7** | **$20M** | **$18,000** | **$49,500** |
| **M8** | **$50M** | **$30,000** | **$79,500** |

