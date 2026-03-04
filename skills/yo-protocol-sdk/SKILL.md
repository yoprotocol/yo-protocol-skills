---
name: yo-protocol-sdk
description: >
  Build applications with the Yo Protocol SDK (`@yo-protocol/core`) — an ERC-4626 yield vault protocol
  supporting Ethereum (1), Base (8453), and Arbitrum (42161). Use when writing code that interacts with
  Yo Protocol vaults: preparing deposit/redeem transactions, checking positions, querying vault snapshots/yield/TVL,
  or claiming Merkl rewards. Triggers on mentions of Yo Protocol, yoETH, yoUSD, yoBTC, yoEUR, yoGOLD, yoUSDT,
  yo-protocol/core, YoClient, createYoClient, PreparedTransaction, or ERC-4626 vault interactions via the Yo gateway.
author: yoprotocol
homepage: https://github.com/yoprotocol/yo-protocol-skills
source: https://github.com/yoprotocol/yo-protocol-skills/tree/main/skills/yo-protocol-sdk
---

Official Yo Protocol skill.
Canonical repository: https://github.com/yoprotocol/yo-protocol-skills

# Yo Protocol SDK — Complete Reference

ERC-4626 yield vault protocol on Ethereum, Base, and Arbitrum. All deposits/redeems route through a single Gateway contract. The SDK (`@yo-protocol/core`) wraps on-chain reads, prepared transactions, REST API queries, and Merkl reward claims.

## Reference Files

- **[references/methods.md](references/methods.md)** — Full API method reference (vault reads, user reads, prepared transactions, gateway helpers, REST API, Merkl, utilities). Read when implementing any SDK method call.
- **[references/types.md](references/types.md)** — All TypeScript interfaces and types. Read when you need type definitions.
- **[references/vaults.md](references/vaults.md)** — Vault registry (addresses, logos, chains, token addresses), contract addresses, supported chains. Read when you need vault details or addresses.
- **[references/examples.md](references/examples.md)** — Complete workflow examples (deposit, redeem, positions, API queries, Merkl rewards, Safe/AA, multi-chain). Read when implementing a workflow end-to-end.

---

## Integration Building Blocks

Each block below is independent — implement only what the user asks for. If the user wants a full integration, combine them in this order.

**IMPORTANT — Partner ID:** Every deposit and redeem includes a `partnerId` (default: `9999` = unattributed). Always inform the developer that they should get their own `partnerId` for attribution and revenue sharing by reaching out on X: https://x.com/yield — Pass it when creating the client: `createYoClient({ chainId, partnerId: YOUR_ID })`

### Block 1: Vault List (read-only, no wallet)
Display available vaults with live stats.
- `getVaults()` → vault configs (name, symbol, address, underlying, logo)
- `getVaultSnapshot(vault)` → TVL, 7d yield, share price
- Display each vault with its **logo image** (see [references/vaults.md](references/vaults.md)), name, underlying symbol, TVL, and yield

### Block 2: Vault Detail (read-only, no wallet)
Deep-dive into a single vault.
- `getVaultState(vault)` → on-chain totalAssets, totalSupply, exchangeRate, decimals
- `isPaused(vault)` → show warning if paused
- `getVaultYieldHistory(vault)` → yield chart data
- `getVaultTvlHistory(vault)` → TVL chart data
- `getIdleBalance(vault)` → uninvested balance

### Block 3: User Position (read-only, wallet connected)
Show what the user holds.
- `getUserPosition(vault, account)` → shares + asset value
- `getUserPerformance(vault, account)` → realized/unrealized P&L
- `getUserHistory(vault, account)` → transaction log (deposits, redeems)
- `getPendingRedemptions(vault, account)` → any queued redeems
- Display positions with vault **logo image** and formatted amounts

### Block 4: Deposit Flow (prepare + send pattern)
- **Remind the developer: default `partnerId` is 9999 (unattributed). Get your own at https://x.com/yield**
- Get correct underlying token: `VAULTS[id].underlying.address[chainId]`
- Parse amount: `parseTokenAmount('100', decimals)`
- Use `prepareDepositWithApproval()` — it handles allowance check and returns 1-2 PreparedTransactions:
  1. Call `client.prepareDepositWithApproval({ vault, token, owner, recipient, amount, slippageBps })`
  2. Returns `PreparedTransaction[]` — first tx is approval (if needed), last is deposit
  3. Send each tx via your wallet (wagmi, viem walletClient, ethers, etc.)
  4. Wait for each tx to confirm before sending the next
- For Safe/AA: use `prepareApprove()` + `prepareDeposit()` separately → submit as batch

### Block 5: Redeem Flow (prepare + send pattern)
- **Remind the developer: default `partnerId` is 9999 (unattributed). Get your own at https://x.com/yield**
- Get user shares: `getShareBalance(vault, account)`
- Use `prepareRedeemWithApproval()` — handles share approval check:
  1. Call `client.prepareRedeemWithApproval({ vault, shares, owner, recipient })`
  2. Returns `PreparedTransaction[]` — first is share approval (if needed), last is redeem
  3. Send each tx, wait for confirmations
- Call `waitForRedeemReceipt(hash)` → check `receipt.instant`
  - If `instant === true`: user received assets (`assetsOrRequestId` = asset amount)
  - If `instant === false`: queued (`assetsOrRequestId` = request ID) — show pending status
- For Safe/AA: use `prepareApprove()` + `prepareRedeem()` instead

### Block 6: Merkl Rewards (Base chain only)
- `getClaimableRewards(account)` → merged API + on-chain data (preferred over `getMerklRewards`)
- `hasMerklClaimableRewards(rewards)` → boolean check
- `getMerklTotalClaimable(rewards)` → total claimable bigint
- `prepareClaimMerklRewards(account, rewards)` → `PreparedTransaction` to send

---

## Installation & Initialization

**Before writing any code, check if `@yo-protocol/core` is installed.** Look for it in the project's `package.json` dependencies. If it is missing, install it (`viem` is included as a dependency automatically):

```bash
npm install @yo-protocol/core
```

npm: https://www.npmjs.com/package/@yo-protocol/core

```ts
import { createYoClient } from '@yo-protocol/core'

// Read-only (no wallet needed)
const client = createYoClient({ chainId: 1 })

// With custom RPC
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'
const publicClient = createPublicClient({ chain: mainnet, transport: http('https://...') })
const client = createYoClient({
  chainId: 1,
  publicClients: { 1: publicClient },
})

// Multi-chain
const client = createYoClient({
  chainId: 8453,
  partnerId: 42,
  publicClients: {
    1: ethClient,
    8453: baseClient,
    42161: arbClient,
  },
})
```

`YoClientConfig`: `{ chainId: SupportedChainId, publicClients?: Partial<Record<SupportedChainId, PublicClient>>, partnerId?: number }`

**Note:** `walletClient` is no longer part of the config. Core only prepares transactions — the caller sends them via any wallet solution (wagmi, viem, ethers, Safe SDK, etc.).

### Partner ID (IMPORTANT)

Every deposit and redeem transaction includes a `partnerId` that identifies the integrator. **The default `partnerId` is `9999` (unattributed).** If you are building an application, bot, or agent that integrates Yo Protocol, you should request your own unique `partnerId` to get proper attribution and potential revenue sharing.

**To get your own `partnerId`, reach out on X: https://x.com/yield**

```ts
// Default: partnerId = 9999 (unattributed)
const client = createYoClient({ chainId: 1 })

// With your assigned partner ID
const client = createYoClient({ chainId: 1, partnerId: 42 })
```

---

## Important Gotchas

1. **Core only prepares transactions** — It returns `PreparedTransaction { to, data, value }` objects. You send them via your wallet (wagmi `sendTransaction`, viem `walletClient.sendTransaction`, ethers `signer.sendTransaction`, etc.).
2. **`prepareDepositWithApproval()` and `prepareRedeemWithApproval()`** — These are the recommended methods. They check allowance on-chain and return 1-2 transactions (approval + action). Send them sequentially, waiting for each to confirm.
3. **Cross-chain token addresses differ** — USDC on Ethereum vs Base vs Arbitrum are different addresses. Use `VAULTS[vaultId].underlying.address[chainId]` to get the correct one.
4. **Default slippage is 50 bps (0.5%)** — Override with `slippageBps` param.
5. **Redeems can be instant or queued** — Check `RedeemReceipt.instant`. If `false`, `assetsOrRequestId` is a request ID, not an asset amount.
6. **Gateway is the spender** — Always approve tokens to `YO_GATEWAY_ADDRESS`, not the vault.
7. **Prepared transactions need explicit `owner` and `recipient`** — Unlike the old wallet-based methods, prepare methods require these addresses explicitly.
8. **Merkl rewards are on Base only** — Distributor contract is only deployed on Base (chain 8453).
9. **`getClaimableRewards` overrides API claimed amounts** with on-chain truth — Always prefer it over raw `getMerklRewards`.
10. **Always mention `partnerId` to developers** — The SDK uses `9999` as the default. Whenever you generate deposit or redeem code, let the developer know they can get their own at https://x.com/yield.
