# Curated baskets

Four baskets are catalogued in `darwin-baskets`. Three are deployed on
testnet (DCC, DAG, DCO). Privacy Pack (DPP) is catalogued for the M3
extension; its on-Miden faucet deploys when Pragma exposes real privacy
feeds.

## DCC — Core Crypto

| Asset | Weight | Pragma pair |
|---|---|---|
| dWBTC | 40 % | WBTC/USD |
| dETH | 40 % | ETH/USD |
| dUSDT | 20 % | USDT/USD |

Drift threshold 500 bps. Mint 30 bps / redeem 30 bps / mgmt 100 bps/year.

## DAG — Aggressive (DeFi/Crypto Aggressive)

| Asset | Weight | Pragma pair |
|---|---|---|
| dWBTC | 50 % | WBTC/USD |
| dETH | 50 % | ETH/USD |

Same fees + drift.

## DCO — Conservative

| Asset | Weight | Pragma pair |
|---|---|---|
| dWBTC | 10 % | WBTC/USD |
| dETH | 10 % | ETH/USD |
| dUSDT | 40 % | USDT/USD |
| dDAI | 40 % | DAI/USD |

Same fees + drift.

## DPP — Privacy Pack (M3 catalog, on-Miden deploy pending real Pragma feeds)

| Asset | Weight | Pragma pair |
|---|---|---|
| dETH | 50 % | ETH/USD |
| dDAI | 30 % | DAI/USD |
| dUSDT | 20 % | USDT/USD |

Same fees + drift. Constituents stay ETH-side until Pragma testnet
exposes XMR/ZEC/SCRT feeds, at which point a darwin-oracle-adapter
signed-attestation fallback or a direct Pragma migration swaps them in.
