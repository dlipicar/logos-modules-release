# logos-modules-release

Dario's personal collection of Logos packages — four wallet stacks,
**EVM / Ethereum**, **Bitcoin**, **Monero** and **Zcash**, each covering
everything from key custody (and, where it applies, a node of your own)
through to the send screen, plus a Uniswap swap app on the EVM side — and
what they share: the **signer manager** with its hardware signers
(Keycard, Keycard Shell, air-gapped QR wallets) and the **Tor** that
comes with Logos.

Each entry is a git submodule under `submodules/`, published as an `.lgx`
package by the matching `Release <module>` workflow. `logos-repo.json` is
the catalog manifest clients read; the index lives on the `index` release
tag.

## Versions

Every package in this catalog is 0.2.x, and every dependency between two
of its packages is declared as `{ "name": ..., "version": "~0.2.0" }` —
0.2.0 or any later 0.2.x patch. The range is enforced at load time.
liblogos refuses a core module whose installed dependency is out of
range, and Basecamp blocks a UI module the same way. So a 0.1.x package
left installed from before blocks every 0.2.x module that depends on it
until it is replaced. Edges to platform modules (`modules_state`,
`package_manager`) carry no range.

## Accounts and signing

Shared by the EVM and Bitcoin wallets: one keystore holds both chains'
accounts, and every signature goes through the signer manager.

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-evm-keystore-module` | `keystore_module` | — | scrypt vaults, BIP39/BIP32 HD derivation, secp256k1 signing. No network; private keys never leave it. |
| `logos-signer-manager-module` | `signer_manager_module` | `keystore_module`; `keycard_module`, `keycard_shell_module`, `qr_signer_module` (optional) | Takes every signature request, shows it through the signer app, and has the account's own signer sign it. Holds no keys. |
| `logos-keycard-module` | `keycard_module` | — | Keycard signer: accounts offered from a bare Keycard in a smart-card reader, signed once the card's PIN opens it. Holds no secret but a 3.x card's pairing. |
| `logos-keycard-shell-module` | `keycard_shell_module` | — | Keycard Shell signer over USB: each request is signed on the Shell's own screen. Holds no secret. |
| `logos-qr-scanner-module` | `qr_scanner_module` | — | Shows QR codes and reads them through the camera for other modules, via the Logos QR helper it bundles. Only decoded text comes back. |
| `logos-qr-signer-module` | `qr_signer_module` | `qr_scanner_module` | Air-gapped wallets that sign over animated QR codes (BC-UR, ERC-4527), such as the Keycard Shell, Keystone, Passport and Jade. Holds no secret. |

### UI

| Module | Package | Provides intents | What it does |
| --- | --- | --- | --- |
| `logos-evm-signer-ui` | `evm_signer_ui` | `evm.signing.approve`, `wallets.unlock` | The only surface that renders what is to be signed, in the signer manager's own words, and where each wallet is unlocked once per session for every app. |
| `logos-evm-keystore-ui` | `evm_keystore_ui` | `evm.accounts.manage`, `keystore.accounts.open` | Create, import, export, rename, delete accounts, and decide apps' requests to open them. |
| `logos-keycard-ui` | `keycard_ui` | `evm.accounts.manage` | Keycard wallets and accounts, through a smart-card reader or a Keycard Shell over USB. |
| `logos-qr-signer-ui` | `qr_signer_ui` | — | Adds an air-gapped wallet's accounts from its account code, and chooses which ones Logos apps may use. |

### CLI

| Module | Package | What it does |
| --- | --- | --- |
| `logos-evm-keystore-cli` | `evm_keystore_cli` | Headless custodian for `keystore_module` and `signer_manager_module` — the `logosctl` equivalent of the keystore UI. |
| `logos-evm-signer-cli` | `evm_signer_cli` | Headless approver for `signer_manager_module` — the `logosctl` equivalent of the signer UI. |

## Tor

| Module | Package | Depends on / provides | What it does |
| --- | --- | --- | --- |
| `logos-tor-module` | `tor_module` | — | Tor for Logos modules, so no one has to run it themselves: Arti in-process. Modules make HTTP requests through it over IPC, and circuits are never shared between modules; wallets whose connections are not HTTP use its SOCKS port on loopback. |
| `logos-tor-ui` | `tor_ui` | provides `tor.manage` | Whether Tor works and why not, which apps use it, what it carried, Reconnect, Forget Tor's data, and its log. |

The Bitcoin, Monero and Zcash wallets go through it by default (their node
modules depend on `tor_module`), and each wallet app hands off to
`tor_ui` through `tor.manage`.

## EVM / Ethereum

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-evm-eth-rpc-module` | `eth_rpc_module` | `verified_proxy_module` (optional) | The device-wide chain registry (enabled chains, mainnet/testnet scope) and a proxyable, fail-closed Ethereum JSON-RPC client (per-chain config, socks5h/Tor-ready). |
| `logos-evm-beacon-api-module` | `beacon_api_module` | — | Proxyable, fail-closed Ethereum Beacon API client: per-chain consensus-layer provider lists with failover, serving the light-client routes the verified proxy reads. socks5h/Tor-ready. |
| `logos-evm-token-list-module` | `token_list_module` | — | The device-wide token catalogue and enabled ERC-20 set: Uniswap token lists, a built-in offline list and custom entries (proxyable, fail-closed). |
| `logos-verified-proxy-module` | `verified_proxy_module` | `beacon_api_module` | Light-client-verified JSON-RPC, wrapping status-im's nimbus libverifproxy. |
| `logos-evm-fee-module` | `fee_module` | `eth_rpc_module` | EIP-1559 slow/normal/fast tiers from `eth_feeHistory`, with custom overrides, and whole-bundle estimates. |
| `logos-evm-tx-sender-module` | `tx_sender_module` | `eth_rpc_module`, `fee_module`, `signer_manager_module` | The one transaction sender on the device: one nonce ledger, a call bundle approved as one decision through the signer manager, ordered broadcast, write-ahead history. Holds no key material. |
| `logos-evm-assets-module` | `evm_assets_module` | `eth_rpc_module` | Native and ERC-20 identity, balances, amount conversion, unsigned transfer building and history decoration, over the token rows its caller passes in. Cannot sign or send. |
| `logos-evm-uniswap-module` | `uniswap_module` | `eth_rpc_module` | V2/V3/V4 best-rate prices (Multicall3-batched), swap quotes, and the calls that make a swap. Sends nothing. |
| `logos-eth-wallet-backend` | `eth_wallet_backend` | `eth_rpc_module`, `fee_module`, `keystore_module`, `token_list_module`, `evm_assets_module`, `tx_sender_module`, `signer_manager_module` | The wallet composer: multi-chain balances and activity over the modules above, with every send leaving through `tx_sender_module`. |
| `logos-uniswap-backend` | `uniswap_backend` | `eth_rpc_module`, `token_list_module`, `evm_assets_module`, `fee_module`, `keystore_module`, `signer_manager_module`, `tx_sender_module`, `uniswap_module` | The Uniswap app's backend: quotes, swaps and swap history, composed from the modules above. Holds no key material. |

### UI

| Module | Package | Provides intent | What it does |
| --- | --- | --- | --- |
| `logos-eth-wallet-ui` | `eth_wallet_ui` | `evm.transactions.send` | The wallet itself: balances and activity across enabled networks, Send on an explicitly chosen chain. Holds no key material. |
| `logos-uniswap-ui` | `uniswap_ui` | — | Swap any two tokens on Uniswap from the wallet's accounts: a view over `uniswap_backend`. Holds no key material and sends nothing itself. |
| `logos-eth-rpc-ui` | `eth_rpc_ui` | `evm.rpc.configure` | Endpoint, chain and verified-proxy configuration. |
| `logos-token-list-ui` | `token_list_ui` | `evm.token_lists.configure` | List sources, custom tokens, and which catalogue tokens wallets offer on each chain. |
| `logos-verified-proxy-ui` | `verified_proxy_ui` | `evm.verified_routing.operate` | Drives the light-client proxy. |

`eth_wallet_ui` reaches the surfaces it does not implement — signing and
unlocking, accounts, RPC config, verified routing, token lists — through
those intents, so it needs the UIs above and the signer and keystore UIs
installed with it to be fully functional. `uniswap_ui` needs four of
them: signing, unlocking, accounts and token lists.

`evm.transactions.send` points the other way: `eth_wallet_ui` provides it
for apps outside this catalog that want the wallet to send on their
behalf. `uniswap_ui` does not use it — `uniswap_backend` hands the calls
that make a swap to `tx_sender_module` directly.

## Bitcoin

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-bitcoind-module` | `bitcoind_module` | — | A Bitcoin node (Bitcoin Core) running in-process, serving the wallet Core's RPC over Logos IPC: per-network config, start, stop, flush, status and log tail. |
| `logos-bitcoin-node-module` | `bitcoin_node_module` | `tor_module`; `bitcoind_module` (optional) | The wallet's sources per network: Kyoto peers, Electrum servers and your own node (in Logos or over RPC), with Tor per source and address-free health probes. Fail-closed: a source routed through Tor never falls back to a direct connection. |
| `logos-bitcoin-wallet-core-module` | `bitcoin_wallet_core_module` | `signer_manager_module` | The wallet engine: BDK, watch-only, its database encrypted under the wallet's own key from the signer manager, signing through it, and a chain authority every server answer is checked against. |
| `logos-bitcoin-wallet-backend` | `bitcoin_wallet_backend` | `bitcoin_wallet_core_module`, `bitcoin_node_module` | Coordinator: custodian roles, the wallet registry, sends from request to settlement, and the node module's source settings. Holds no key material and takes no password. |

### UI

| Module | Package | Provides intent | What it does |
| --- | --- | --- | --- |
| `logos-bitcoin-wallet-ui` | `bitcoin_wallet_ui` | — | The wallet: the confirmed balance and what is still on its way, sends approved in the signer app, a fresh receive address with a BIP21 QR, activity, and the sources the wallet reads the chain from. Holds no key material. |
| `logos-bitcoind-ui` | `bitcoind_ui` | `bitcoin.node.configure` | Runs and manages the local node: sync progress, peers, per-network settings, the Tor proxy, the RPC port it can open, and the node log. |

### CLI

| Module | Package | What it does |
| --- | --- | --- |
| `logos-bitcoin-wallet-cli` | `bitcoin_wallet_cli` | Headless Bitcoin wallet sessions for `logosctl`: relays to `bitcoin_wallet_backend`, and prompts when a send awaits approval in `evm_signer_cli`. Never handles a password. |

Bitcoin accounts live in the same keystore as the EVM ones and are signed
through the signer manager, so `bitcoin_wallet_ui` uses `wallets.unlock`,
`evm.signing.approve` and `evm.accounts.manage` from the section above,
plus `tor.manage` and `bitcoin.node.configure`. Running a node on the
device is optional: `bitcoin_node_module` names `bitcoind_module` as an
optional dependency.

## Monero

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-monerod-module` | `monerod_module` | — | A Monero node running in-process: monerod as a library, with per-network config, start, stop, status and log tail. |
| `logos-monero-node-module` | `monero_node_module` | `tor_module`; `monerod_module` (optional) | Fail-closed monerod JSON-RPC client, the wallet's node per network: "Use Tor for" picks a direct node or one over Tor, `tor_module` by default. Never runs a node itself; its local mode dials the one `monerod_module` runs. |
| `logos-monero-wallet-core-module` | `monero_wallet_core_module` | `monero_node_module` | The wallet engine: wraps monero_c (wallet2, built from source) over its C ABI. The only module that holds a key or a password. |
| `logos-monero-wallet-backend` | `monero_wallet_backend` | `monero_wallet_core_module`, `monero_node_module` | Coordinator: registry, sync, balances, history and the build-review-broadcast send flow. Holds no key material. |

### UI

| Module | Package | Provides intents | What it does |
| --- | --- | --- | --- |
| `logos-monero-wallet-ui` | `monero_wallet_ui` | `monero.wallet.unlock`, `monero.accounts.manage` | The wallet app: balances, a reviewed send, receive with QR, activity, wallet management. |
| `logos-monerod-ui` | `monerod_ui` | `monero.node.configure` | Runs and manages the local node: sync progress, peers, settings and the node log. |

### CLI

| Module | Package | What it does |
| --- | --- | --- |
| `logos-monero-wallet-cli` | `monero_wallet_cli` | Headless Monero wallet for `logosctl`: open, read, transfer, review, broadcast. Holds the custodian and approver roles. |

The Monero stack keeps its own keys and passwords and shares only Tor with
the rest of the catalog. `monero_wallet_ui` uses `monero.node.configure` —
the node sheet's **Manage local node…** hands off to `monerod_ui` — and
`tor.manage`. It also *provides* two intents, for consumers outside this
catalog.

Running a node on the device is optional. `monero_node_module` names
`monerod_module` as an optional dependency and dials it only when a
network's node config is in local mode; the default remote mode talks to
a configured endpoint. The wallet offers "the node on this device" only
when `monerod_module` is installed.

## Zcash

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-zebrad-module` | `zebrad_module` | — | A Zcash node (Zebra) running in-process, serving the wallet its lightwalletd service over Logos IPC: per-network config, start, stop, status and log tail. |
| `logos-zcash-node-module` | `zcash_node_module` | `tor_module`; `zebrad_module` (optional) | lightwalletd-protocol servers per network: presets, the proxy (Tor by default), a route table per call class, and live health polled over Tor. Fail-closed: no proxy, no route. |
| `logos-zcash-wallet-core-module` | `zcash_wallet_core_module` | `tor_module`; `zebrad_module` (optional) | The wallet engine: a librustzcash light client with keys sealed at rest, a SQLCipher database, and sync over Tor. Answers only `zcash_wallet_backend`. |
| `logos-zcash-wallet-backend` | `zcash_wallet_backend` | `zcash_wallet_core_module`, `zcash_node_module` | Coordinator: roles, engine jobs, events and server routes. Holds no key material. |

### UI

| Module | Package | Provides intents | What it does |
| --- | --- | --- | --- |
| `logos-zcash-wallet-ui` | `zcash_wallet_ui` | `zcash.wallet.unlock`, `zcash.accounts.manage` | The wallet: shielded and transparent balances, password-approved sends, shielding, receive addresses with a QR, history, and wallet management. |
| `logos-zebrad-ui` | `zebrad_ui` | `zcash.node.configure` | Runs and manages the local node: sync progress, peers, settings, the ports it opens, and the node log. |

### CLI

| Module | Package | What it does |
| --- | --- | --- |
| `logos-zcash-wallet-cli` | `zcash_wallet_cli` | Headless Zcash wallet sessions for `logosctl`: relays to `zcash_wallet_backend` and takes secrets as `@file` arguments. |

Like Monero, the Zcash stack keeps its own keys and passwords and shares
only Tor. `zcash_wallet_ui` uses `zcash.node.configure`, which
`zebrad_ui` provides, and `tor.manage`.

## Working with the catalog

```bash
git submodule update --init --recursive   # after cloning; trees are not committed
./scripts/catalog.sh release <module>     # publish one module
./scripts/catalog.sh release-all --watch  # publish everything
./scripts/catalog.sh rebuild-index        # regenerate index.json
./scripts/add-module.sh <git-url>         # add a module + its release workflow
```

For now every module is released for `darwin-arm64` and `linux-amd64`
only. Every module also exposes `linux-arm64` and `windows-x86_64`; the
`variants:` line in `.github/workflows/_release-module.yml` is the one
place to change the platform set for the whole catalog, and a forced
release (`./scripts/catalog.sh release-all --force`) adds the others to
an already-published version. Windows is a mingw **cross**
build produced on a Linux runner; there is no Nix for Windows.

No module overrides that any more. `logos-verified-proxy-ui` used to
narrow itself back to the three native variants — its `flake.lock` still
pinned `verified_proxy_module` from before the Windows fix, and a flake
input resolves from the consumer's lock rather than the dependency's own.
[logos-co/logos-verified-proxy-ui#5](https://github.com/logos-co/logos-verified-proxy-ui/pull/5)
relocked it, so the override is gone and every module takes the default.
