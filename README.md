# sal-nodejs-pool

Mining pool software for **Salvium (SAL)** written in Node.js: a stratum server for Salvium's RandomX (`rx/0`) with share checking, block accounting
through the pool wallet, and batch payouts. It is a fork of [cryptonote-nodejs-pool](https://github.com/dvandal/cryptonote-nodejs-pool) by Dvandal
(GNU GPL v2), via its [arqma-nodejs-pool](https://github.com/newsmoneymaker/arqma-nodejs-pool) adaptation, for the Monero-style daemon and wallet RPC
of Salvium (a Monero fork with its own "Carrot" address scheme). Live example: <https://sal.pool-pay.com>.

## What it does

* **Stratum server** (plain TCP and TLS ports, XMRig protocol with the `algo` extension): the pool asks the node for a block template
  (`get_block_template`), gives every miner the hashing blob and a share target that follows the miner's hashrate (vardiff, remembered across reconnects),
  and checks every share itself with a small C++ helper (`hasher/rxhash`): RandomX of the hashing blob with the seed hash as key, using Salvium's own
  RandomX variant. A share that meets the network difficulty is submitted to the node with `submit_block`.
  The pool announces the algorithm `rx/0` in every job: the miner has to know it ([poolpayminer](https://github.com/newsmoneymaker/poolpayminer) does).
* **Accounts** are Salvium addresses: the legacy CryptoNote family (`SaLv...`, standard or subaddress; integrated addresses are not accepted) or the
  newer "Carrot" family (`SC1...`), optionally `+worker`, with a reward mode prefix `prop:` (shared, default) or `solo:`. Mining itself REQUIRES the
  pool's own block-reward address to be a Carrot address (Salvium's node refuses a legacy address for `generateblocks`); a miner's own payout address
  may be either kind.
* **Rewards:** PROP with time weighting (slush) or SOLO.
* **Block unlocker:** a block is settled after `depth` blocks. The reward is what the pool wallet received in the block's coinbase transaction (wallet
  `get_transfer_by_txid` on the block's `miner_tx_hash`); a block that is no longer on the chain is marked orphaned and nothing is credited.
* **Payment processor:** everyone who is due is paid in **one wallet `transfer` per round** (the network fee is shared by the paid miners or paid by
  the pool). Salvium's wallet RPC is multi-asset ("Carrot"): the input-selection code filters the wallet's outputs by matching `asset_type` so a
  payout is never built from the wrong asset. The balance is debited before sending; a batch whose outcome is unknown (crash, timeout) is found again
  among the wallet's transfers by its destinations and never sent twice; refused or failed batches go back to the balances. Dry-run mode, a whitelist
  for rehearsals and an emergency brake (`deployment/pause-payments.sh`).
* **Website and API:** a ready website (`website_example/`) with the dashboard and its graphs, blocks, payments, top miners, worker statistics, a
  "Getting started" page with a config generator, and the public read-only JSON API.
* **Protection against connection floods:** limits per IP, a login deadline, an optional IP allow list, banning of miners with many invalid shares.

## Developer donation (please read)

The pool takes a **developer donation** from the reward of every block it finds, before the miners' shares are computed. It is configured in
`config.json` (see `config_examples/sal.json`):

```json
"blockUnlocker": {
  "poolFee": 1,
  "donations": { "SC1YourPoolAddress...": 0.5 }
}
```

* `poolFee` is the fee of the pool operator (percent). `donations` is a table `Salvium address -> percent` (up to 10% per entry) for the developers of
  this software; **the default is 0.5% to the address of the project's own pool**. The donation is paid like any other balance.
* It is included in the "Pool Fee" figure that the pool's website and API show to the miners (as in the original software).
* You are free to change the percentage, the address or to empty the table (`"donations": {}`): it is your pool and the license is the GPL.
  Please tell your miners the truth about the fees of your pool.
* This has nothing to do with the miner poolpayminer (a separate project with its own fee).

## Installation

See [docs/INSTALL.md](docs/INSTALL.md): the Salvium node and wallet (build from source), the RandomX helper, Redis, the pool services (systemd
templates in `deployment/`), the website and the first payout rehearsal.

Requirements: Linux, Node.js 18 or newer, Redis, a Salvium node (`salviumd`) synchronised with the network and `salvium-wallet-rpc` with the pool
wallet (holding a Carrot address), a C++ compiler and the node's RandomX library for the helper, a web server for the website and a TLS certificate
for the TLS stratum ports.

## Miners

[poolpayminer](https://github.com/newsmoneymaker/poolpayminer) (Windows and Linux, free, has its own fee) mines Salvium with the standard `rx/0`
algorithm. Example: `poolpayminer -a rx/0 --tls -o sal.pool-pay.com:4601 -u SaLvYourAddress+rigname -p x -k`.

## Money warning

The payment processor moves real coins. Rehearse first: `"dryRun": true`, then a whitelist (`onlyAccounts`) with a few small payouts of your own,
then enable it. A transaction that has been sent to the network can not be cancelled. Keep the wallet backup (the 25 word seed) and the wallet password private.

## License and credits

GNU GPL v2 (see [LICENSE](LICENSE)). Based on cryptonote-nodejs-pool, Copyright (c) Dvandal and contributors. Salvium and RandomX are separate programs
and are not included.
