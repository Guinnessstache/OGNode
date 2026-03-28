# OGNode

**OriginTrail v1 TRAC Recovery Tool**

A browser-based tool for checking and withdrawing staked TRAC tokens from the original OriginTrail v1 node network (ODN). If you ran a data holding node before OriginTrail moved to DKG v6/v8, your TRAC is likely still sitting in the old `ProfileStorage` contract. This tool replicates the full withdrawal flow that was available on the now-defunct `node-profile.origintrail.io` site.

**Live site:** https://Guinnessstache.github.io/ognode

---

## What it does

- **Balance check** — reads your staked TRAC, reserved amount, and reputation score directly from the `ProfileStorage` contract on Ethereum mainnet. Read-only, no wallet needed for this step.
- **Gas estimation** — shows the live ETH cost for both withdrawal transactions and whether it's worth it vs. the TRAC value at current prices.
- **Auto-detect pending withdrawals** — if `startTokenWithdrawal` was already called on your identity in the past and never finished, the tool detects it and skips straight to the one-click collect step.
- **Full two-step withdrawal** — if starting fresh: signs `startTokenWithdrawal`, waits the 5-minute delay (with countdown), then signs `withdrawTokens`.
- **MetaMask and Ledger support** — MetaMask via browser injection; Ledger via WalletConnect QR code.

---

## How to use

1. Open the [live site](https://Guinnessstache.github.io/ognode)
2. Connect your **management wallet** — the wallet you used to manage your node on node-profile.origintrail.io
3. Enter your **ERC725 identity address** — this is your node's on-chain identity contract, NOT your wallet address. It's the address you pasted into node-profile.origintrail.io
4. Click **Check balance**
5. If a pending withdrawal is detected and ready → click **Sign & collect TRAC now** (one transaction)
6. If starting fresh → set the amount, sign Step 1, wait 5 minutes, sign Step 2

---

## Technical details

### Contracts (Ethereum mainnet)

| Contract | Address |
|---|---|
| Hub | `0x89777F4D16F0a263F47EaD07cbCAb9497861aa79` |
| Profile | `0x7875357a329de15c525f4aaa0d9dd849fcfc2729` |
| ProfileStorage (token vault) | `0x306D5E8AF6AeB73359dcC5E22C894e2588F76FFB` |
| TRAC token | `0xaa7a9ca87d3694b5755f213b5d04094b8d0f0a6f` |

### Function selectors (reverse-engineered from bytecode)

The ProfileStorage contract source was never verified on Etherscan. These selectors were decoded directly from the contract bytecode dispatch table:

| Selector | Function |
|---|---|
| `0x7a766460` | `getStake(address)` |
| `0x5ffe0282` | `getStakeReserved(address)` |
| `0x56582bf9` | `getWithdrawalAmount(address)` |
| `0x9344ea6e` | `getWithdrawalTimestamp(address)` |
| `0x2ba97595` | `getWithdrawalPending(address)` |
| `0x9c89a0e2` | `getReputation(address)` |
| `0x389eb9f9` | `withdrawalTime()` (on Profile) |
| `0x07ab5e84` | `startTokenWithdrawal(address,uint256)` (on Profile) |
| `0x49df728c` | `withdrawTokens(address)` (on Profile) |
| `0xb61d27f6` | `execute(uint256,address,uint256,bytes)` (ERC725 identity) |

### Transaction flow

Withdrawals are routed through your ERC725 identity contract, exactly as the original site did:

```
management_wallet → identity.execute(0, Profile, 0, startTokenWithdrawal(identity, amount))
                 → identity.execute(0, Profile, 0, withdrawTokens(identity))
```

The Profile contract address was located by reading the Hub's storage mapping using `web3_sha3` via RPC to compute the storage slot for the key `"Profile"`.

### RPC fallback chain

Queries use four public Ethereum RPC endpoints with automatic fallback:
1. `https://eth-mainnet.public.blastapi.io`
2. `https://ethereum.publicnode.com`
3. `https://eth.llamarpc.com`
4. `https://1rpc.io/eth`

---

## Known limitations

### Reserved stake is likely unrecoverable

The most important limitation: the `stakeReserved` field in ProfileStorage tracks how much of your TRAC is locked in active data holding jobs. For most abandoned v1 nodes, this will be close to 100% of the total stake.

In the v1 ODN system, reserved stake is only released when:
- A holding job is formally finalized on-chain (the data holder proves they stored the data for the full duration), OR
- A litigation/replacement procedure runs its course

Since the v1 network is no longer active and the infrastructure to finalize jobs has been decomissioned, the reserved portion for most nodes is effectively **permanently stuck**. The contract holds the accounting but there is no longer a mechanism to trigger the finalization that would release it.

Only the **unreserved** portion (total stake minus reserved) can be withdrawn. For many nodes this will be a small amount or zero.

### Other limitations

- **Ethereum mainnet only** — this tool does not support xDAI, NeuroWeb, Base, or any other chain. Those use different contracts entirely.
- **Unverified contract source** — the ProfileStorage contract source code was never submitted to Etherscan for verification. All function selectors were reverse-engineered from the compiled bytecode. They have been tested and confirmed working but this is not a formally audited interface.
- **WalletConnect v1** — the WalletConnect integration uses v1 of the protocol. Newer versions of Ledger Live may require WalletConnect v2. If the QR code flow doesn't work, try connecting Ledger via MetaMask instead (MetaMask supports Ledger natively via USB).
- **No event indexing** — the Holding and HoldingStorage contracts emit no on-chain events, making it impossible to enumerate which offer IDs a node was holding without an external indexer. OTHub (the community indexer) appears to be offline. This means there is no way to trigger job finalization without knowing the offer IDs.
- **Client-side only** — this is a static HTML page with no backend. All computation happens in your browser. No data is sent anywhere except Ethereum RPC calls and the CoinGecko price API.
- **Use at your own risk** — this tool has not been formally audited. Always read the transaction details in your wallet before signing. Never sign a transaction you don't understand.

---

## Running locally

No build step required — it's a single HTML file.

```bash
git clone https://github.com/Guinnessstache/ognode
cd ognode
python3 -m http.server 8080
# Open http://localhost:8080 in Chrome
```

> **Note:** MetaMask does not inject into `file://` pages. Always serve via a local HTTP server or use the live GitHub Pages URL.

---

## Background

OriginTrail launched its v1 Decentralized Network (ODN) in 2019. Node operators staked TRAC tokens as collateral to participate in data holding jobs — the tokens were locked in the `ProfileStorage` contract for the duration of each job. The network migrated to DKG v6 and later v8, which use entirely different contracts. The v1 contracts were never formally closed out, leaving TRAC locked for operators who didn't withdraw before the migration.

The original withdrawal interface at `node-profile.origintrail.io` has since been decommissioned. OGNode reconstructs that same withdrawal flow by reading the original contract addresses from the hub's on-chain registry and reverse-engineering the function signatures from bytecode.

---

## Disclaimer

OGNode is an independent community tool. It is not affiliated with, endorsed by, or supported by OriginTrail or Trace Labs. TRAC is a utility token — this tool does not provide financial advice. Use at your own risk.
