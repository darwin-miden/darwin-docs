# Bali agglayer integration

The canonical Sepolia ↔ Miden bridge for Darwin. Operated by
gateway-fm via the [miden-agglayer](https://github.com/gateway-fm/miden-agglayer)
stack. Separate from BrianSeong99's 1Click mock (which lives at
`http://localhost:8080` in dev).

## Network params (post-relaunch 2026-05-26)

| field | value |
|---|---|
| Network ID | `76` |
| L2 chain ID | `1022211914` |
| Bech32 HRP | `mcst1` |
| Bridge account (Miden) | `mcst1arychvrurzxdy5qwz0mg5p5umsvsepyx` / `0xc98bb07c188cd2500e13f68a069cdc` |
| ETH faucet (Miden) | `mcst1arnrhfau9svl7cpu2tr8lfzzd5j87wwe` / `0xe63ba7bc2c19ff603c52c67fa4426d` |
| Sepolia bridge contract | `0x1348947e282138d8f377b467F7D9c2EB0F335d1f` |
| Bridge service API | `https://miden-testnet-bridge.dev.eu-north-3.gateway.fm` |
| Pinned `miden-client` | `0.14.4` (Poseidon2) |

The pre-relaunch network 73 deposits are **permanently frozen**. Anything
that still references `mtst1` HRP, `0xa75ca0c…` bridge, or
`network=73` is from the old outpost and not recoverable.

## L1 → L2 path (Sepolia → Miden)

1. `bridgeAsset(76, dest, amount, 0x0, true, 0x)` on the Sepolia
   bridge contract. `dest` is the 20-byte ETH-padded form of the
   recipient's 15-byte Miden account ID — the frontend's `midenToEthDest()`
   helper handles the padding.
2. Bridge service indexes the deposit (`/api/bridges/:dest`) — visible
   immediately with `ready_for_claim=false`.
3. ~25-30 min later: aggsender pushes the GER, `ready_for_claim`
   flips to `true`, bridge solver mints a P2ID note on Miden carrying
   the ETH (from faucet `0xe63ba7bc…`) to the destination. The note's
   tx hash appears in the deposit row's `claim_tx_hash`.
4. The recipient's miden-client sync picks the note up. The relay v2
   worker auto-drains it into the relay vault on its next tick.

### Decimals

The Bali ETH faucet uses **8 decimals**, not 18. A 0.001 ETH
Sepolia deposit (`10^15` wei) lands as `100000` base units on
Miden (`0.001 * 10^8`).

### Frontend test panel

`darwin-frontend/src/components/BaliDepositPanel.tsx` drives the L1→L2
flow from the browser. Connect an ETH wallet on `/portfolio`, set the
amount and Miden destination, click the bridge button. The panel
polls `/api/bridges/:dest` every 30 s and shows the lifecycle:
`awaiting-wallet → tx-sent → indexing → ready-to-claim → claimed`.

## L2 → L1 path (Miden → Sepolia)

Three steps, the last of which is easy to forget:

1. **Submit B2AGG note on Miden** via gateway-fm's `bridge-out-tool`
   (canonical Bali outbound format). Brian's 1Click mock outbound
   uses a different `bridge_out_v1` MASM script and is not the same
   thing. `bridge-out-tool` lives in
   [`gateway-fm/miden-agglayer`](https://github.com/gateway-fm/miden-agglayer/tree/main/scripts);
   build with `cargo build --release --bin bridge-out-tool`.

2. **Wait for the agglayer certificate to settle.** AggLayer
   settles certificates on a once-per-hour cadence, aggsender
   builds at the 50%-of-epoch mark; wall time ~30–90 min. Poll
   `/api/bridges/<eth_dest>`; the deposit is ready when
   `ready_for_claim=true`.

3. **Call `claimAsset` on the Sepolia bridge.** This step is
   permissionless and *required* — the Bali stack does *not*
   auto-claim. Until someone submits it, the funds sit on the L1
   bridge contract indefinitely. The merkle proof comes from
   `/api/merkle-proof?deposit_cnt=<N>&net_id=76`. The 11-arg
   signature is
   `claimAsset(bytes32[32], bytes32[32], uint256, bytes32, bytes32,
   uint32, address, uint32, address, uint256, bytes)`.

   `darwin-infra/scripts/bali-l1-claim.sh` wraps all of step 3
   (fetch proof + pad SMT to 32 siblings + cast send):

   ```sh
   DEPOSIT_CNT=8 DEST_ADDR=0xf6d3C9Ed... ./scripts/bali-l1-claim.sh
   ```

### First successful L2→L1 round-trip on Bali net 76

Verified 2026-05-27:

| Step | Tx |
|---|---|
| L2 burn (B2AGG note) | Miden `0x0b8c59ea7318c7bd0d782b4a513ee53f65ef9dd42f725e96f59f1a9611f337b6` |
| L1 claim | Sepolia [`0xc5e6bc113ea639a56897da8be3e7dc58b8013c458ee684aa77756d6e3fb0e3df`](https://sepolia.etherscan.io/tx/0xc5e6bc113ea639a56897da8be3e7dc58b8013c458ee684aa77756d6e3fb0e3df) |

Amount: 50000 Miden ETH-faucet units → 0.0005 ETH on Sepolia.
Wall time burn-to-claim ~25 min on this run (would have been longer
if the agglayer cert hadn't been already mid-epoch at burn time).

## Verification artefacts

First full L1→L2 round-trip on the relaunched outpost (2026-05-26):

- Sepolia tx `0x0e246200f3b0fe345f34cf31d6ecb4154ff39d4a80427412596ec66448fddec3`
  (block 10925457)
- bridge service deposit_cnt `1132814`, dest_net `76`
- Claim tx on Miden `0x9697f4a109e8e80b0d271640931ecef53e2b5a4a13a0cf52878c8408b3348de7`
- ~30 min wall-clock from `cast send` to relay vault credited

## Gotchas

- Sepolia bridge contract is the same on both networks (73 + 76). Only
  the `destinationNetwork` arg distinguishes them. Submitting with
  `73` after the relaunch sends funds to a network where the L2 stack
  is gone — funds frozen, not recoverable.
- The bridge service indexes by **destination address** (the ETH-padded
  form), not by sender. Querying `/api/bridges/:sender_eoa` always
  returns empty for our deposits.
- The trailing zero byte on `midenToEthDest()` is significant — without
  it the bridge can decode the Miden ID but the alignment is wrong and
  the claim note never lands on the right account.
