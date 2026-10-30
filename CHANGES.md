# Changes

## 1.0.0

* First release: Salvium (Monero based) pool derived from arqma-nodejs-pool, qwc-nodejs-pool, veil-nodejs-pool and epic-nodejs-pool. Stratum for `rx/0`,
  share checking with the RandomX helper `hasher/rxhash` built with the node's own RandomX library, `get_block_template` / `submit_block`, Monero-style
  CryptoNote addresses (`SaLv...`) for login, plus Salvium's own "Carrot" address family (`SC1...`) required for the pool's own block-reward address.
* **Wallet RPC asset filtering:** Salvium's wallet RPC is multi-asset ("Carrot"); the payment processor's input selection sets `asset_type` on every
  transfer destination so a payout is never built against the wrong asset.
* The variable difficulty of a worker is remembered across reconnects (`poolServer.diffMemoryMinutes`, default 15).
* Real miner shares accepted end to end against the live node and pool; a real block was found and paid out correctly (fixing, along the way, a
  RandomX-variant bug where a borrowed `rxhash` library from another coin silently rejected every real share as "Bad hash" -- see the project's own
  `critical-wrong-randomx-variant-bug` notes).
* No reward vesting: unlike the C64 Chain template this pool was adapted from, Salvium's coinbase pays the pool wallet directly with no locked
  portion, so `blockUnlocker.vesting` stays disabled.
