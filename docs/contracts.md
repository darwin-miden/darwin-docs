# Live contracts & accounts

## Miden testnet

| Account | ID (hex) | Role |
|---|---|---|
| User / relay wallet | `0xed3cd5befa3207805f8529207cfc0d` | Falcon-512 custodial, drives Flow A/B/C + 1Click + Bali |
| v6 controller | `0x2a3ea0a268d97b80497d6a966e3141` | Current — slot 11 fee_recipient + receive_and_credit |
| v5 controller | `0x9419f2044acb77800a4c91a0cb50e5` | Per-user storage maps (slots 2/3/4/10) |
| v4 controller | `0x1975a9aa8572f8804fb38bee09fbdf` | Rebalance-aware (`execute_rebalance_step`) |
| v2 controller | `0xa25aa0b00007688024b74b05a52aab` | Real-bodies (preserves `receive_asset` MAST) |
| Basket faucet DCC | `0x2066f2da1f91ba202af5251d39101c` | Core Crypto basket token |
| Basket faucet DAG | `0xfb6811fd6399df206d44f62800620d` | Aggressive basket token |
| Basket faucet DCO | `0xbe4efc6729eb3220423b7d6d6a0942` | Conservative basket token |
| dETH faucet | `0xa095d9b3831e96206ff70c2218a6a9` | Miden-native ETH proxy |
| Pragma oracle | `0xd0e1384e21a6350029d80128eb5c44` | Live `get_median` feeds |
| Bali bridge | `0xc98bb07c188cd2500e13f68a069cdc` (`mcst1arychvrurzxdy5qwz0mg5p5umsvsepyx`) | Canonical L1↔L2 bridge account |
| Bali ETH faucet | `0xe63ba7bc2c19ff603c52c67fa4426d` (`mcst1arnrhfau9svl7cpu2tr8lfzzd5j87wwe`) | Bridged ETH (8 decimals) |

## Sepolia (ETH side)

| Contract | Address | Role |
|---|---|---|
| DarwinStrategy | `0x635E19c61CD09d145D57A88cE8185Ddf27fA356F` | M2 basket config registry (token list, weights, fees) |
| MockUSDC | `0x6dAb940a4E1d434965E22e9F6d624fF68F6922a0` | Permissionless faucet for testnet |
| DCC ERC20 | `0x1EB7Bd808402824232853e66DF6843D68462B7A4` | Wrapped DCC (M2 relay path) |
| DAG ERC20 | `0x73F18087dd45d180e75cADcD383479624326E336` | Wrapped DAG |
| DCO ERC20 | `0x6344469eB35Ff00d5892fD368727ad3C9E45677c` | Wrapped DCO |
| Bali bridge | `0x1348947e282138d8f377b467F7D9c2EB0F335d1f` | `bridgeAsset(76, dest, …)` canonical |
| 1Click mock | `localhost:8080` (dev) | Brian's mock for Sepolia↔Miden 1Click |

## v6 MAST roots

```
receive_and_credit  0xeae9e249a88021a2fb6bcae39148f528ee98d5fc884290a42f961b9a536c763e
receive_asset       0x75f638c65584d058542bcf4674b066ae394183021bc9b44dc2fdd97d52f9bcfb (same as v2)
set_user_position   0xa017ac3e12d53bad11bfb8b4289a3bd2c4deef4c67a5209c53703dacbbe2d335
set_target_weights  0x57a8ef319a2fe090f649760c4db4fdfc698496778daaea8f496cc46070e4057c
set_fees            0xf2624ee2a579f81446f60cba7fdb06058c36fa2a06fc1b67accaafdd0d86e3f8
set_fee_recipient   0x6721d6156a7a78b8eea224963e4375ee7423ac2d2f79d58a1c5af542f370d9a4
```

## Source verifications

The Sepolia stack is verified on Sourcify + Routescan + Blockscout for
`DarwinStrategy` and the three basket ERC20s. `DarwinRelayDeposit` is
the deprecated v1 escrow path — verify failed on bytecode mismatch
(deployed compile settings drifted from current source), kept as
historical reference only.
