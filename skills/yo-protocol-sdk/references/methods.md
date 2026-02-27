# SDK Method Reference

## Table of Contents
- [Vault Reads (on-chain)](#vault-reads-on-chain)
- [User Reads (on-chain)](#user-reads-on-chain)
- [Write Actions (require wallet)](#write-actions-require-wallet)
- [Prepared Transactions (calldata mode)](#prepared-transactions-calldata-mode)
- [Gateway Quote Helpers](#gateway-quote-helpers)
- [Gateway Allowance Helpers](#gateway-allowance-helpers)
- [REST API Methods](#rest-api-methods)
- [Merkl Reward Methods](#merkl-reward-methods)
- [Transaction Helpers](#transaction-helpers)
- [Utilities](#utilities)

---

## Vault Reads (on-chain)

### `getVaults(): VaultConfig[]`
Return known vault configs for the client's chain. No RPC call.

### `getVaultState(vault: Address): Promise<VaultState>`
Full on-chain vault state: name, symbol, decimals, totalAssets, totalSupply, asset address, exchangeRate.

### `previewDeposit(vault: Address, assets: bigint): Promise<bigint>`
ERC-4626 `previewDeposit` — how many shares for a given asset amount.

### `previewRedeem(vault: Address, shares: bigint): Promise<bigint>`
ERC-4626 `previewRedeem` — how many assets for a given share amount.

### `convertToAssets(vault: Address, shares: bigint): Promise<bigint>`
ERC-4626 `convertToAssets` — share-to-asset conversion without fees.

### `convertToShares(vault: Address, assets: bigint): Promise<bigint>`
ERC-4626 `convertToShares` — asset-to-share conversion without fees.

### `isPaused(vault: Address): Promise<boolean>`
Check if vault deposits are paused.

### `getIdleBalance(vault: Address): Promise<bigint>`
Get vault's idle (uninvested) balance.

---

## User Reads (on-chain)

### `getTokenBalance(token: Address, account: Address): Promise<TokenBalance>`
ERC-20 balance. Returns `{ token, balance, decimals }`.

### `getShareBalance(vault: Address, account: Address): Promise<bigint>`
User's vault share balance (raw bigint).

### `getUserPosition(vault: Address, account: Address): Promise<UserVaultPosition>`
User's position as `{ shares, assets }` — shares held and their asset value.

### `getAllowance(token: Address, owner: Address, spender: Address): Promise<TokenAllowance>`
ERC-20 allowance. Returns `{ token, owner, spender, allowance }`.

### `hasEnoughAllowance(token: Address, owner: Address, spender: Address, amount: bigint): Promise<boolean>`
Check if allowance >= amount.

---

## Write Actions (require wallet)

All write methods require `walletClient` set via constructor or `setWalletClient()`.

### `approve(token: Address, amount: bigint, spender?: Address): Promise<ApproveResult>`
ERC-20 approve. Spender defaults to Gateway. Returns `{ hash }`.

### `approveMax(token: Address, spender?: Address): Promise<ApproveResult>`
Approve max uint256. Spender defaults to Gateway. Returns `{ hash }`.

### `deposit(params): Promise<DepositResult>`
Deposit assets via Gateway. Returns `{ hash, shares }`.

```ts
params: {
  vault: Address
  amount: bigint        // asset amount to deposit
  recipient?: Address   // defaults to wallet account
  slippageBps?: number  // default 50 (0.5%)
  partnerId?: number    // default from client config
}
```

### ~~`depositWithApproval()`~~ — DEPRECATED, DO NOT USE
This method is unreliable because it bundles approve + deposit without properly waiting for tx confirmations. **Always use separate `approve()` → `waitForTransaction()` → `deposit()` instead.** See the Deposit workflow example in [examples.md](examples.md).

### `redeem(params): Promise<RedeemResult>`
Redeem shares via Gateway. Returns `{ hash, assets }`.

```ts
params: {
  vault: Address
  shares: bigint        // shares to redeem
  recipient?: Address   // defaults to wallet account
  slippageBps?: number  // default 50
  minAssetsOut?: bigint // override slippage calc
  partnerId?: number
}
```

---

## Prepared Transactions (calldata mode)

For Safe multisig, Account Abstraction, or batched transactions. Returns `PreparedTransaction { to, data, value }`.

### `prepareApprove(params: { token, amount, spender? }): PreparedTransaction`
Synchronous. No RPC call.

### `prepareDeposit(params): Promise<PreparedTransaction>`
**Requires `recipient`** (unlike `deposit()`). Queries gateway for minShares.

### `prepareRedeem(params): Promise<PreparedTransaction>`
**Requires `recipient`**. Queries gateway for minAssetsOut.

### ~~`prepareDepositWithApproval()`~~ — DEPRECATED, DO NOT USE
Use `prepareApprove()` + `prepareDeposit()` separately instead. For Safe/AA, submit both as a batch transaction.

---

## Gateway Quote Helpers

On-chain reads against the Gateway contract (may differ from direct vault reads due to fees).

### `quotePreviewDeposit(vault: Address, assets: bigint): Promise<bigint>`
### `quotePreviewRedeem(vault: Address, shares: bigint): Promise<bigint>`
### `quotePreviewWithdraw(vault: Address, assets: bigint): Promise<bigint>`
### `quoteConvertToAssets(vault: Address, shares: bigint): Promise<bigint>`
### `quoteConvertToShares(vault: Address, assets: bigint): Promise<bigint>`

---

## Gateway Allowance Helpers

### `getShareAllowance(vault: Address, owner: Address): Promise<bigint>`
Share allowance granted to the Gateway.

### `getAssetAllowance(vault: Address, owner: Address): Promise<bigint>`
Asset allowance granted to the Gateway.

---

## REST API Methods

Off-chain queries to `https://api.yo.xyz`. No RPC needed.

### `getVaultSnapshot(vault: Address): Promise<VaultSnapshot>`
Comprehensive vault data: TVL, yield (1d/7d/30d), protocols, share price, idle balances.

### `getVaultYieldHistory(vault: Address): Promise<TimeseriesPoint[]>`
Historical yield data. Returns `{ timestamp, value }[]`.

### `getVaultTvlHistory(vault: Address): Promise<TimeseriesPoint[]>`
Historical TVL data. Returns `{ timestamp, value }[]`.

### `getUserHistory(vault: Address, user: Address, limit?: number): Promise<UserHistoryItem[]>`
User transaction history (deposits, withdraws, redeems).

### `getUserPerformance(vault: Address, user: Address): Promise<UserPerformance>`
User realized/unrealized P&L. Returns `{ realized: FormattedValue, unrealized: FormattedValue }`.

### `getUserPoints(user: Address): Promise<UserPoints | null>` *(deprecated)*
Use `getUserPerformance` instead.

### `getPendingRedemptions(vault: Address, user: Address): Promise<PendingRedeem>`
User's pending redeem requests.

### `getVaultPendingRedeems(vault: Address): Promise<PendingRedeem>`
Total pending redeems for a vault.

---

## Merkl Reward Methods

### `getMerklCampaigns(options?: { status?: 'LIVE' | 'PAST' }): Promise<MerklCampaign[]>`
List Yo-related Merkl campaigns.

### `getMerklRewards(user: Address): Promise<MerklChainRewards | null>`
Raw API rewards for user on client's chain.

### `getMerklClaimedAmount(user: Address, token: Address): Promise<bigint>`
On-chain claimed amount for a specific reward token.

### `getClaimableRewards(user: Address): Promise<MerklChainRewards | null>`
**Preferred method.** Merges API rewards with on-chain claimed amounts for accurate claimable data.

### `claimMerklRewards(chainRewards: MerklChainRewards): Promise<ClaimMerklRewardsResult>`
Claim rewards on-chain. Pass the result of `getClaimableRewards()`. Returns `{ hash }`.

### Utility methods (static-like)
- `getMerklClaimableAmount(reward: MerklTokenReward): bigint` — claimable for single reward
- `hasMerklClaimableRewards(chainRewards: MerklChainRewards): boolean` — any claimable?
- `getMerklTotalClaimable(chainRewards: MerklChainRewards): Map<string, bigint>` — all claimable by token

---

## Transaction Helpers

### `waitForTransaction(hash): Promise<TransactionReceipt>`
Wait for any tx confirmation. Returns `{ hash, status, blockNumber, gasUsed }`.

### `waitForRedeemReceipt(hash): Promise<RedeemReceipt>`
Wait for redeem tx and decode the `YoGatewayRedeem` event. Returns `{ hash, status, instant, assetsOrRequestId, shares, blockNumber }`.

### `setWalletClient(walletClient: WalletClient): void`
Set or change the wallet client after construction.

---

## Utilities

### Formatting

```ts
import { formatTokenAmount, parseTokenAmount, formatUsd, formatPercent, formatCompactNumber } from '@yo-protocol/core'

formatTokenAmount(1000000n, 6)                          // "1"
formatTokenAmount(1000000n, 6, { maxDecimals: 2 })      // "1"
parseTokenAmount("1.5", 18)                             // 1500000000000000000n
formatUsd(1234.5)                                       // "$1,234.50"
formatPercent(0.0534)                                   // "5.34%"
formatCompactNumber(1234567)                            // "1.23M"
```

### Math Helpers (big.js-based)

All accept `string | number | bigint | Big`:

`mul`, `div`, `add`, `sub`, `mulDiv`, `gt`, `gte`, `lt`, `lte`, `eq`, `toFixed`, `toNumber`, `abs`, `pow`, `sqrt`, `toBigInt`, `toStr`

`applySlippage(value: bigint, slippageBps: number): bigint` — `value - (value * bps) / 10000`

### Validation

```ts
import { validateAddress, validateAmount } from '@yo-protocol/core'

validateAddress('0x...', 'vault')   // throws if invalid
validateAmount(100n, 'amount')      // throws if <= 0
```
