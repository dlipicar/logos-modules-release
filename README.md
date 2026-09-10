# logos-modules-release

Dario's personal collection of Logos packages — two complete wallet
stacks, **EVM / Ethereum** and **Monero**, each covering everything from
key custody through to the send screen.

Each entry is a git submodule under `submodules/`, published as an `.lgx`
package by the matching `Release <module>` workflow. `logos-repo.json` is
the catalog manifest clients read; the index lives on the `index` release
tag.

## EVM / Ethereum

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-evm-keystore-module` | `keystore_module` | — | scrypt vaults, BIP39/BIP32 HD derivation, secp256k1 signing. No network; private keys never leave it. |
| `logos-evm-eth-rpc-module` | `eth_rpc_module` | — | Proxyable, fail-closed Ethereum JSON-RPC client (per-chain config, socks5h/Tor-ready). |
| `logos-evm-token-list-module` | `token_list_module` | — | Uniswap token lists + custom entries, downloaded/parsed/merged (proxyable, fail-closed). |
| `logos-verified-proxy-module` | `verified_proxy_module` | — | Light-client-verified JSON-RPC, wrapping status-im's nimbus libverifproxy. |
| `logos-evm-fee-module` | `fee_module` | `eth_rpc_module` | EIP-1559 slow/normal/fast tiers from `eth_feeHistory`, with custom overrides. |
| `logos-eth-wallet-backend` | `eth_wallet_backend` | `eth_rpc_module`, `fee_module`, `keystore_module`, `token_list_module` | The coordinator: balances, Send orchestration, local transaction history. One active network at a time. |

### UI

| Module | Package | Provides intent | What it does |
| --- | --- | --- | --- |
| `logos-eth-wallet-ui` | `eth_wallet_ui` | — | The wallet itself: Send, balances, activity. Holds no key material. |
| `logos-evm-keystore-ui` | `evm_keystore_ui` | `evm.accounts.manage` | Create, import, export, rename, delete accounts. |
| `logos-evm-signer-ui` | `evm_signer_ui` | `evm.signing.approve` | The only surface that renders what is to be signed and takes the vault password. |
| `logos-eth-rpc-ui` | `eth_rpc_ui` | `evm.rpc.configure` | Endpoint, chain and verified-proxy configuration. |
| `logos-token-list-ui` | `token_list_ui` | `evm.token_lists.configure` | List sources, custom tokens, per-chain catalogues. |
| `logos-verified-proxy-ui` | `verified_proxy_ui` | `evm.verified_routing.operate` | Drives the light-client proxy. |

`eth_wallet_ui` reaches the four surfaces it does not implement —
signing, accounts, RPC config, verified routing — through those intents,
so the whole UI set has to be installed together for the wallet to be
fully functional.

### CLI

| Module | Package | What it does |
| --- | --- | --- |
| `logos-evm-keystore-cli` | `evm_keystore_cli` | Headless custodian for `keystore_module` — the `logosctl` equivalent of the keystore UI. |
| `logos-evm-signer-cli` | `evm_signer_cli` | Headless approver for `keystore_module` — the `logosctl` equivalent of the signer UI. |

## Monero

### Core

| Module | Package | Depends on | What it does |
| --- | --- | --- | --- |
| `logos-monero-node-module` | `monero_node_module` | — | Proxyable, fail-closed monerod JSON-RPC client (per-network config). |
| `logos-monero-wallet-core-module` | `monero_wallet_core_module` | `monero_node_module` | The wallet engine: wraps monero_c (wallet2) over its C ABI. The only module that holds a key or a password. |
| `logos-monero-wallet-backend` | `monero_wallet_backend` | `monero_wallet_core_module`, `monero_node_module` | Coordinator: registry, sync, balances, history and the build-review-broadcast send flow. Holds no key material. |

### UI

| Module | Package | Provides intents | What it does |
| --- | --- | --- | --- |
| `logos-monero-wallet-ui` | `monero_wallet_ui` | `monero.wallet.unlock`, `monero.accounts.manage` | The wallet app: balances, a reviewed send, receive with QR, activity, wallet management. |

### CLI

| Module | Package | What it does |
| --- | --- | --- |
| `logos-monero-wallet-cli` | `monero_wallet_cli` | Headless Monero wallet for `logosctl`: open, read, transfer, review, broadcast. Holds the custodian and approver roles. |

Unlike the EVM set, the Monero stack is closed on its own: no module
declares `uses`, so nothing here depends on an intent another module has
to provide. `monero_wallet_ui` *provides* two, for consumers outside this
catalog.

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

One exception: `logos-verified-proxy-ui` narrows itself back to the three
native variants in its own `release-<module>.yml`. Not because Windows is
broken — `verified_proxy_module` cross-builds as of `8576a58` — but
because this UI's `flake.lock` still pins the module before that fix, and
a flake input resolves from the consumer's lock rather than the
dependency's own. Once
[logos-co/logos-verified-proxy-ui#5](https://github.com/logos-co/logos-verified-proxy-ui/pull/5)
merges and a Windows build is confirmed green, drop the override.
