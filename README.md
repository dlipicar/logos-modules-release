# logos-modules-release

Dario's personal collection of Logos packages — two complete wallet
stacks, **EVM / Ethereum** and **Monero**, each covering everything from
key custody through to the send screen, plus a Uniswap swap app on the
EVM side.

Each entry is a git submodule under `submodules/`, published as an `.lgx`
package by the matching `Release <module>` workflow. `logos-repo.json` is
the catalog manifest clients read; the index lives on the `index` release
tag.

## EVM / Ethereum

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-evm-keystore-module` | `keystore_module` | — | scrypt vaults, BIP39/BIP32 HD derivation, secp256k1 signing. No network; private keys never leave it. |
| `logos-evm-eth-rpc-module` | `eth_rpc_module` | — | The device-wide chain registry (enabled chains, mainnet/testnet scope) and a proxyable, fail-closed Ethereum JSON-RPC client (per-chain config, socks5h/Tor-ready). |
| `logos-evm-token-list-module` | `token_list_module` | — | The device-wide token catalogue and enabled ERC-20 set: Uniswap token lists, a built-in offline list and custom entries (proxyable, fail-closed). |
| `logos-verified-proxy-module` | `verified_proxy_module` | — | Light-client-verified JSON-RPC, wrapping status-im's nimbus libverifproxy. |
| `logos-evm-fee-module` | `fee_module` | `eth_rpc_module` | EIP-1559 slow/normal/fast tiers from `eth_feeHistory`, with custom overrides, and whole-bundle estimates. |
| `logos-evm-tx-sender-module` | `tx_sender_module` | `eth_rpc_module`, `fee_module`, `keystore_module` | The one transaction sender on the device: one nonce ledger, a call bundle approved as one keystore decision, ordered broadcast, write-ahead history. Holds no key material. |
| `logos-evm-assets-module` | `evm_assets_module` | `eth_rpc_module`, `token_list_module` | Native and ERC-20 identity, balances, amount conversion, unsigned transfer building and history decoration. Cannot sign or send. |
| `logos-evm-uniswap-module` | `uniswap_module` | `eth_rpc_module` | V2/V3/V4 best-rate prices (Multicall3-batched), swap quotes, and the calls that make a swap. Sends nothing. |
| `logos-eth-wallet-backend` | `eth_wallet_backend` | `eth_rpc_module`, `fee_module`, `keystore_module`, `token_list_module`, `evm_assets_module`, `tx_sender_module` | The wallet composer: multi-chain balances and activity over the modules above, with every send leaving through `tx_sender_module`. |

### UI

| Module | Package | Provides intent | What it does |
| --- | --- | --- | --- |
| `logos-eth-wallet-ui` | `eth_wallet_ui` | `evm.transactions.send` | The wallet itself: balances and activity across enabled networks, Send on an explicitly chosen chain. Holds no key material. |
| `logos-uniswap-ui` | `uniswap_ui` | — | Swap any two tokens on Uniswap from the wallet's accounts. Holds no key material and sends nothing itself. |
| `logos-evm-keystore-ui` | `evm_keystore_ui` | `evm.accounts.manage` | Create, import, export, rename, delete accounts. |
| `logos-evm-signer-ui` | `evm_signer_ui` | `evm.signing.approve` | The only surface that renders what is to be signed and takes the vault password. |
| `logos-eth-rpc-ui` | `eth_rpc_ui` | `evm.rpc.configure` | Endpoint, chain and verified-proxy configuration. |
| `logos-token-list-ui` | `token_list_ui` | `evm.token_lists.configure` | List sources, custom tokens, and which catalogue tokens wallets offer on each chain. |
| `logos-verified-proxy-ui` | `verified_proxy_ui` | `evm.verified_routing.operate` | Drives the light-client proxy. |

`eth_wallet_ui` reaches the five surfaces it does not implement —
signing, accounts, RPC config, verified routing, token lists — through
those intents, so the whole UI set has to be installed together for the
wallet to be fully functional. `uniswap_ui` needs three of them:
signing, accounts and token lists.

`evm.transactions.send` points the other way: `eth_wallet_ui` provides it
for apps outside this catalog that want the wallet to send on their
behalf. `uniswap_ui` does not use it — it hands the calls that make a swap
to `tx_sender_module` directly.

### CLI

| Module | Package | What it does |
| --- | --- | --- |
| `logos-evm-keystore-cli` | `evm_keystore_cli` | Headless custodian for `keystore_module` — the `logosctl` equivalent of the keystore UI. |
| `logos-evm-signer-cli` | `evm_signer_cli` | Headless approver for `keystore_module` — the `logosctl` equivalent of the signer UI. |

### Versions

The EVM stack is versioned as one unit: every package is 0.1.x, and every
dependency inside the stack is declared as
`{ "name": ..., "version": "~0.1.0" }` — 0.1.0 or any later 0.1.x patch.
The range is enforced at load time. liblogos refuses a core module whose
installed dependency is out of range, and Basecamp blocks a UI module
the same way. So a 1.x package left installed from before the reset
blocks every 0.1.x module that depends on it until it is replaced.

## Monero

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-monerod-module` | `monerod_module` | — | A Monero node running in-process: monerod as a library, with per-network config, start, stop, status and log tail. |
| `logos-monero-node-module` | `monero_node_module` | `monerod_module` (optional) | Proxyable, fail-closed monerod JSON-RPC client (per-network config). Never runs a node itself; its local mode dials the one `monerod_module` runs. |
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

The Monero stack is closed on its own. Its one `uses` is
`monero_wallet_ui`'s `monero.node.configure`: the node sheet's **Manage
local node…** hands off to `monerod_ui`, which provides it.
`monero_wallet_ui` also *provides* two intents, for consumers outside this
catalog. Nothing crosses between the Monero and EVM stacks.

Running a node on the device is optional. `monero_node_module` names
`monerod_module` as an optional dependency and dials it only when a
network's node config is in local mode; the default remote mode talks to
a configured endpoint. The wallet offers "the node on this device" only
when `monerod_module` is installed.

## Working with the catalog

```bash
git submodule update --init --recursive   # after cloning; trees are not committed
./scripts/catalog.sh release <module>     # publish one module
./scripts/catalog.sh release-all --watch  # publish everything
./scripts/catalog.sh rebuild-index        # regenerate index.json
./scripts/add-module.sh <git-url>         # add a module + its release workflow
```

Every module here builds for `windows-x86_64` as well as the three
native variants — it is in the `variants:` default in
`.github/workflows/_release-module.yml`, which is the one place to change
the platform set for the whole catalog. Windows is a mingw **cross**
build produced on a Linux runner; there is no Nix for Windows.

No module overrides that any more. `logos-verified-proxy-ui` used to
narrow itself back to the three native variants — its `flake.lock` still
pinned `verified_proxy_module` from before the Windows fix, and a flake
input resolves from the consumer's lock rather than the dependency's own.
[logos-co/logos-verified-proxy-ui#5](https://github.com/logos-co/logos-verified-proxy-ui/pull/5)
relocked it, so the override is gone and every module takes the default.
