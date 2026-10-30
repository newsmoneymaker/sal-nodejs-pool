# Installing a Salvium pool

Paths below are examples: the pool in `/opt/sal-nodejs-pool`, the node and wallet in `/opt/salvium`, all run by the user `salpool`
(the systemd templates in `deployment/systemd/` use these paths).

## 1. Salvium node and wallet

The Salvium source is published on GitHub (<https://github.com/salvium/salvium>). Build from source:

```
git clone --recursive https://github.com/salvium/salvium.git && cd salvium
# Debian/Ubuntu: build-essential cmake pkg-config git python3 libboost-all-dev libssl-dev libzmq3-dev libpgm-dev libunbound-dev libsodium-dev libhidapi-dev
#   libusb-1.0-0-dev libunwind-dev liblzma-dev libreadline-dev libexpat1-dev libprotobuf-dev protobuf-compiler libudev-dev libncurses5-dev libncursesw5-dev liblmdb-dev libminiupnpc-dev libuv1-dev
mkdir -p build/release && cd build/release && cmake -DCMAKE_BUILD_TYPE=Release ../.. && make -j4 daemon simplewallet wallet_rpc_server
# binaries in build/release/bin: salviumd, salvium-wallet-cli, salvium-wallet-rpc
```

Run the node (`deployment/systemd/sal-node.service`): `salviumd --data-dir /opt/salvium/data --non-interactive --rpc-bind-ip 127.0.0.1 --rpc-bind-port 19995 --p2p-bind-port 19996 --no-zmq`.
It finds its peers by itself (built-in seed nodes). While it synchronises, `get_last_block_header` answers with zeros: watch `get_info`
(`curl -s 127.0.0.1:19995/json_rpc -d '{"jsonrpc":"2.0","id":"0","method":"get_info"}'`, `height` against `target_height`).
Open the P2P port 19996 in your firewall if you want incoming peers.

Create the pool wallet (the 25 words are printed once: **write them down and keep them offline**, do not let the output land in a log). The pool's
own block-reward address **must be a Carrot address**: the node refuses `generateblocks`/mining to a legacy CryptoNote address with
"Mining to CryptoNote wallet address, but Carrot wallet address is required". `salvium-wallet-cli` creates a Carrot wallet by default on a current build:

```
salvium-wallet-cli --generate-new-wallet /opt/salvium/wallet/pool --password-file /opt/salvium/wallet/password --daemon-address 127.0.0.1:19995
```

Then run the wallet RPC (`deployment/systemd/sal-wallet-rpc.service`): `salvium-wallet-rpc --wallet-file /opt/salvium/wallet/pool --password-file /opt/salvium/wallet/password
--rpc-bind-ip 127.0.0.1 --rpc-bind-port 19998 --disable-rpc-login --daemon-address 127.0.0.1:19995 --trusted-daemon`. It listens on the local machine only and pays
without a password: keep only the coins that are about to be paid out in this wallet. The wallet's main (Carrot) address is the pool address
(`poolServer.poolAddress`): every block template pays its coinbase there.

## 2. RandomX helper

The helper must use the same RandomX as the node: use the RandomX library the node build made (`librandomx.a` in its `build/release/external/` tree,
the header `randomx.h` in the matching directory of the node's `external/` tree).

```
mkdir -p /opt/randomx/include /opt/randomx/lib
cp /path/to/randomx.h /opt/randomx/include && cp /path/to/librandomx.a /opt/randomx/lib
cd /opt/sal-nodejs-pool/hasher && make RANDOMX=/opt/randomx
```

The helper needs about 2.3 GB of RAM (RandomX dataset, fast mode; `"hasher": {"light": true}` uses 256 MB and is much slower, as this pool's own
`config.json` runs it by default).

## 3. Redis

Use a dedicated instance with a password and AOF (`deployment/redis-pool.conf.example`, unit `sal-pool-redis`, port 6394).

## 4. The pool

```
cd /opt/sal-nodejs-pool && npm install --production
cp config_examples/sal.json config.json      # then edit it
```

Edit `config.json`: `poolHost`, `poolServer.poolAddress` (the pool's Carrot address), the ports and the certificate for TLS (`poolServer.sslCert/sslKey`),
`redis`, `api.password`, `blockUnlocker.poolFee` and `donations`, `payments`. **Keep `payments.dryRun: true` until the rehearsal below.**

```
cp deployment/systemd/*.service /etc/systemd/system/ && systemctl daemon-reload
systemctl enable --now sal-pool-redis sal-node sal-wallet-rpc
systemctl enable --now sal-pool sal-pool-api sal-pool-unlocker sal-pool-payments sal-pool-charts
```

The pool runs as separate modules (`init.js -module=pool|api|unlocker|payments|chartsDataCollector`), each in its own unit. The pool module keeps one hasher process
(and its RandomX dataset), so run it as one process. Until payouts are proven, restrict the stratum ports with `poolServer.allowIPs`.

## 5. Website

Copy `website_example/` to the web root, set `poolHost`, the contact and links in `config.js`, and proxy `/api` to the pool API on 127.0.0.1:8136
(`deployment/apache-vhost.conf.example` exposes only the read-only methods; adjust the port there to match `api.port` in `config.json`).

## 6. Rehearse the payments

1. `payments.dryRun: true`: the log of `sal-pool-payments` shows what would be paid.
2. Fund the pool wallet with a few coins (or wait for the first block), credit a small balance in Redis to your own test addresses
   (`Salvium:workers:<address>`, field `balance`), set `payments.onlyAccounts` to them, `dryRun: false`, and watch the payout confirm.
3. Remove the test accounts from Redis and set `onlyAccounts` to `[]`.

Good to know: `deployment/pause-payments.sh` stops new payouts at once (resume with `deployment/resume-payments.sh`); a wallet cannot abandon a
transaction, so a payout that stays unconfirmed for `pendingTimeoutHours` is only reported in the log and must be checked by hand. This pool's
`blockUnlocker.js` still contains the reward-vesting code it inherited from the C64 Chain template it was adapted from (`blockUnlocker.vesting`):
Salvium's own coinbase has no vesting, so leave it unset/disabled in `config.json`.
