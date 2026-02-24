---
name: yo-protocol-sdk
description: >
  Build applications with the Yo Protocol SDK (`@yo-protocol/core`) — an ERC-4626 yield vault protocol
  supporting Ethereum (1), Base (8453), and Arbitrum (42161). Use when writing code that interacts with
  Yo Protocol vaults: depositing, redeeming, checking positions, preparing transactions for Safe/AA wallets,
  querying vault snapshots/yield/TVL, or claiming Merkl rewards. Triggers on mentions of Yo Protocol,
  yoETH, yoUSD, yoBTC, yoEUR, yoGOLD, yoUSDT, yo-protocol/core, or ERC-4626 vault interactions via
  the Yo gateway.
---

# Yo Protocol SDK — Complete Reference

ERC-4626 yield vault protocol on Ethereum, Base, and Arbitrum. All deposits/redeems route through a single Gateway contract. The SDK (`@yo-protocol/core`) wraps on-chain reads, write actions, prepared transactions, REST API queries, and Merkl reward claims.

## Integration Building Blocks

Each block below is independent — implement only what the user asks for. If the user wants a full integration, combine them in this order. Each block lists exactly what to import, call, and display.

**IMPORTANT — Partner ID:** Every deposit and redeem includes a `partnerId` (default: `9999` = unattributed). Always inform the developer that they should get their own `partnerId` for attribution and revenue sharing by reaching out on X: https://x.com/yield — Pass it when creating the client: `createYoClient({ chainId, walletClient, partnerId: YOUR_ID })`

### Block 1: Vault List (read-only, no wallet)
Display available vaults with live stats.
- `getVaults()` → vault configs (name, symbol, address, underlying, logo)
- `getVaultSnapshot(vault)` → TVL, 7d yield, share price
- Display each vault with its **logo image** (see Vault Registry), name, underlying symbol, TVL, and yield

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

### Block 4: Deposit Flow (wallet required)
- **Remind the developer: default `partnerId` is 9999 (unattributed). Get your own at https://x.com/yield**
- Get correct underlying token: `VAULTS[id].underlying.address[chainId]`
- Parse amount: `parseTokenAmount('100', decimals)`
- Simplest path: `depositWithApproval({ vault, token, amount })` — handles approve + deposit
- Show result: `approveHash` (if needed), `depositHash`, `shares` received
- For Safe/AA: use `prepareDepositWithApproval()` instead → returns `PreparedTransaction[]`

### Block 5: Redeem Flow (wallet required)
- **Remind the developer: default `partnerId` is 9999 (unattributed). Get your own at https://x.com/yield**
- Get user shares: `getShareBalance(vault, account)`
- Call `redeem({ vault, shares })`
- Call `waitForRedeemReceipt(hash)` → check `receipt.instant`
  - If `instant === true`: user received assets (`assetsOrRequestId` = asset amount)
  - If `instant === false`: queued (`assetsOrRequestId` = request ID) — show pending status
- For Safe/AA: use `prepareRedeem()` instead

### Block 6: Merkl Rewards (Base chain only, wallet required)
- `getClaimableRewards(account)` → merged API + on-chain data (preferred over `getMerklRewards`)
- `hasMerklClaimableRewards(rewards)` → boolean check
- `getMerklTotalClaimable(rewards)` → `Map<token, amount>`
- `claimMerklRewards(rewards)` → submit claim tx

---

## Installation & Initialization

**Before writing any code, check if `@yo-protocol/core` is installed.** Look for it in the project's `package.json` dependencies. If it is missing, install it (`viem` is included as a dependency automatically):

```bash
npm install @yo-protocol/core
```

If using yarn or pnpm:
```bash
yarn add @yo-protocol/core
pnpm add @yo-protocol/core
```

npm: https://www.npmjs.com/package/@yo-protocol/core

```ts
import { createYoClient } from '@yo-protocol/core'

// Read-only (no wallet needed)
const client = createYoClient({ chainId: 1 })

// With wallet
const client = createYoClient({ chainId: 8453, walletClient })

// Custom RPC
import { createPublicClient, http } from 'viem'
import { mainnet } from 'viem/chains'
const publicClient = createPublicClient({ chain: mainnet, transport: http('https://...') })
const client = createYoClient({ chainId: 1, publicClient })
```

`YoClientConfig`: `{ chainId: SupportedChainId, publicClient?: PublicClient, walletClient?: WalletClient, partnerId?: number }`

### Partner ID (IMPORTANT)

Every deposit and redeem transaction includes a `partnerId` that identifies the integrator. **The default `partnerId` is `9999` (unattributed).** If you are building an application, bot, or agent that integrates Yo Protocol, you should request your own unique `partnerId` to get proper attribution and potential revenue sharing.

**To get your own `partnerId`, reach out on X: https://x.com/yield**

```ts
// Default: partnerId = 9999 (unattributed)
const client = createYoClient({ chainId: 1, walletClient })

// With your assigned partner ID
const client = createYoClient({ chainId: 1, walletClient, partnerId: 42 })
```

The `partnerId` is passed to the Gateway contract on every `deposit()` and `redeem()` call. You can also override it per-transaction via the `partnerId` param on individual deposit/redeem calls.

---

## Contract Addresses

| Contract | Address |
|----------|---------|
| **Gateway** | `0xF1EeE0957267b1A474323Ff9CfF7719E964969FA` |
| **Vault Registry** | `0x56c3119DC3B1a75763C87D5B0A2C55E489502232` |
| **Oracle** | `0x6E879d0CcC85085A709eBf5539224f53d0D396B0` |
| **Redeemer** | `0x0439e941841f97dc1334d1a433379c6fcdcc2162` |
| **Merkl Distributor** (Base) | `0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae` |
| **Merkl Creator** | `0x8C9200d94Cf7A1B201068c4deDa6239F15FED480` |
| **YO Token** (Base) | `0x3C1a1c9C2D073E5bC4e7AF97f0d7caC7a82E2262` |

Merkl API base: `https://api.merkl.xyz/v4`
REST API base: `https://api.yo.xyz`

---

## Supported Chains

| Chain | ID | Network Name |
|-------|-----|-------------|
| Ethereum | `1` | `ethereum` |
| Base | `8453` | `base` |
| Arbitrum | `42161` | `arbitrum` |

Type: `SupportedChainId = 1 | 8453 | 42161`

---

## Vault Registry

When displaying vaults or user positions in any UI, always include the token logo image.

### yoETH
- **Address**: `0x3a43aec53490cb9fa922847385d82fe25d0e9de7`
- **Logo**: `https://assets.coingecko.com/coins/images/54932/standard/yoETH.png`
- **Underlying**: WETH (18 decimals)
- **Chains**: Ethereum, Base
- **Token addresses**:
  - Ethereum (1): `0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2`
  - Base (8453): `0x4200000000000000000000000000000000000006`

### yoBTC
- **Address**: `0xbcbc8cb4d1e8ed048a6276a5e94a3e952660bcbc`
- **Logo**: `https://assets.coingecko.com/coins/images/55189/standard/yoBTC.png`
- **Underlying**: cbBTC (8 decimals)
- **Chains**: Ethereum, Base
- **Token addresses**:
  - Ethereum (1): `0xcbb7c0000ab88b473b1f5afd9ef808440eed33bf`
  - Base (8453): `0xcbb7c0000ab88b473b1f5afd9ef808440eed33bf`

### yoUSD
- **Address**: `0x0000000f2eb9f69274678c76222b35eec7588a65`
- **Logo**: `https://assets.coingecko.com/coins/images/55386/standard/yoUSD.png`
- **Underlying**: USDC (6 decimals)
- **Chains**: Ethereum, Base, Arbitrum
- **Token addresses**:
  - Ethereum (1): `0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48`
  - Base (8453): `0x833589fcd6edb6e08f4c7c32d4f71b54bda02913`
  - Arbitrum (42161): `0xaf88d065e77c8cC2239327C5EDb3A432268e5831`

### yoEUR
- **Address**: `0x50c749ae210d3977adc824ae11f3c7fd10c871e9`
- **Logo**: `https://assets.coingecko.com/coins/images/68746/standard/yoEUR_color.png`
- **Underlying**: EURC (6 decimals)
- **Chains**: Ethereum, Base
- **Token addresses**:
  - Ethereum (1): `0x1aBaEA1f7C830bD89Acc67eC4af516284b1bC33c`
  - Base (8453): `0x60a3E35Cc302bFA44Cb288Bc5a4F316Fdb1adb42`

### yoGOLD
- **Address**: `0x586675A3a46B008d8408933cf42d8ff6c9CC61a1`
- **Logo**: `https://assets.coingecko.com/coins/images/69957/standard/yoGOLD.png`
- **Underlying**: XAUt (6 decimals)
- **Chains**: Ethereum only
- **Token addresses**:
  - Ethereum (1): `0x68749665FF8D2d112Fa859AA293F07A622782F38`

### yoUSDT
- **Address**: `0xb9a7da9e90d3b428083bae04b860faa6325b721e`
- **Logo**: `https://assets.coingecko.com/coins/images/55386/standard/yoUSD.png`
- **Underlying**: USDT (6 decimals)
- **Chains**: Ethereum only
- **Token addresses**:
  - Ethereum (1): `0xdac17f958d2ee523a2206206994597c13d831ec7`

### Programmatic Access

```ts
import { VAULTS, getVaultsForChain, getVaultByAddress, YO_GATEWAY_ADDRESS } from '@yo-protocol/core'

// Get all vaults for a chain
const baseVaults = getVaultsForChain(8453)

// Look up by address
const vault = getVaultByAddress('0x0000000f2eb9f69274678c76222b35eec7588a65')

// Get underlying token address for a specific chain
const usdcOnBase = VAULTS.yoUSD.underlying.address[8453]
```

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

### `depositWithApproval(params): Promise<DepositWithApprovalResult>`
Approve (if needed) + deposit in one call. Returns `{ approveHash?, depositHash, shares }`.

```ts
params: {
  vault: Address
  token: Address        // token to approve
  amount: bigint
  recipient?: Address
  slippageBps?: number
  partnerId?: number
}
```

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

### `prepareDepositWithApproval(params): Promise<PreparedTransaction[]>`
Returns array: `[approveTx?, depositTx]`. Checks on-chain allowance.

```ts
params: {
  vault: Address
  token: Address
  owner: Address        // token owner for allowance check
  amount: bigint
  recipient?: Address
  slippageBps?: number
  partnerId?: number
}
```

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

## TypeScript Interfaces

### Core Types

```ts
type SupportedChainId = 1 | 8453 | 42161
type VaultId = 'yoETH' | 'yoBTC' | 'yoUSD' | 'yoEUR' | 'yoGOLD' | 'yoUSDT'

interface YoClientConfig {
  chainId: SupportedChainId
  publicClient?: PublicClient
  walletClient?: WalletClient
  partnerId?: number           // default: 9999
}
```

### Vault Types

```ts
interface VaultState {
  address: Address
  name: string
  symbol: string
  decimals: number
  totalAssets: bigint
  totalSupply: bigint
  asset: Address
  assetDecimals: number
  exchangeRate: bigint
}

interface VaultConfig {
  address: Address
  name: string
  symbol: VaultId
  underlying: {
    symbol: string
    decimals: number
    address: Record<number, Address>  // chainId -> token address
  }
  chains: readonly number[]
}
```

### User Types

```ts
interface UserVaultPosition {
  shares: bigint
  assets: bigint
}

interface TokenBalance {
  token: Address
  balance: bigint
  decimals: number
}

interface TokenAllowance {
  token: Address
  owner: Address
  spender: Address
  allowance: bigint
}
```

### Action Results

```ts
interface DepositResult {
  hash: Hash
  shares: bigint
}

interface DepositWithApprovalResult {
  approveHash?: Hash
  depositHash: Hash
  shares: bigint
}

interface RedeemResult {
  hash: Hash
  assets: bigint
}

interface ApproveResult {
  hash: Hash
}

interface TransactionReceipt {
  hash: Hash
  status: 'success' | 'reverted'
  blockNumber: bigint
  gasUsed: bigint
}

interface RedeemReceipt {
  hash: Hash
  status: 'success' | 'reverted'
  instant: boolean              // true = assets received, false = queued
  assetsOrRequestId: bigint     // assets if instant, request ID if queued
  shares: bigint
  blockNumber: bigint
}

interface PreparedTransaction {
  to: Address
  data: Hex
  value: bigint
}
```

### API Types

```ts
interface FormattedValue {
  raw: number | string
  formatted: string
}

interface VaultSnapshot {
  id: string
  name: string
  type?: string
  asset: { name, symbol, decimals, address, coingeckoId? }
  shareAsset: { name, symbol, decimals, address }
  chain: { id, name, nativeAsset?, explorer?, blockTime? }
  contracts: { vaultAddress, authorityAddress? }
  deployment?: { blockNumber, timestamp, txHash }
  protocols?: { name, allocation? }[]
  stats: {
    tvl: FormattedValue
    totalSupply?: FormattedValue
    maxCap?: FormattedValue
    yield: { '1d': string|null, '7d': string|null, '30d': string|null }
    rewardYield?: string|null
    sharePrice?: FormattedValue
    idleBalance?: FormattedValue
    investedBalance?: FormattedValue
    protocolStats?: ProtocolStat[]
  }
  lastUpdated?: number
}

interface TimeseriesPoint {
  timestamp: number
  value: number
}

interface UserHistoryItem {
  type: 'deposit' | 'withdraw' | 'redeem'
  timestamp: number
  assets: FormattedValue
  shares: FormattedValue
  txHash: string
}

interface UserPerformance {
  realized: FormattedValue
  unrealized: FormattedValue
}

interface PendingRedeem {
  assets?: FormattedValue
  shares?: FormattedValue
}

type Network = 'base' | 'ethereum' | 'arbitrum' | 'unichain' | 'tac' | 'plasma' | 'hyperevm'
```

### Merkl Types

```ts
interface MerklToken {
  address: Address
  chainId: number
  decimals: number
  symbol: string
  name: string
  icon?: string
}

interface MerklCampaign {
  id: string
  computeChainId: number
  distributionChainId: number
  startTimestamp: number
  endTimestamp: number
  creatorAddress: string
  rewardToken: MerklToken
  amount: string
  Opportunity?: MerklOpportunity
}

interface MerklTokenReward {
  token: MerklToken
  amount: string       // total earned (cumulative)
  claimed: string      // already claimed
  proofs: string[]     // merkle proofs for claiming
}

interface MerklChainRewards {
  chainId: number
  rewards: MerklTokenReward[]
}

interface ClaimMerklRewardsResult {
  hash: Hash
}

type MerklCampaignStatus = 'LIVE' | 'PAST'
```

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

---

## Workflow Examples

### Deposit into a Vault

```ts
import { createYoClient, VAULTS, parseTokenAmount } from '@yo-protocol/core'

const client = createYoClient({ chainId: 1, walletClient })

// 1. Check vault state
const vault = VAULTS.yoUSD
const state = await client.getVaultState(vault.address)
const paused = await client.isPaused(vault.address)
if (paused) throw new Error('Vault is paused')

// 2. Parse amount (100 USDC)
const amount = parseTokenAmount('100', vault.underlying.decimals) // 100_000_000n

// 3. Deposit with auto-approval
const result = await client.depositWithApproval({
  vault: vault.address,
  token: vault.underlying.address[1],  // USDC on Ethereum
  amount,
})
console.log('Approve tx:', result.approveHash)   // undefined if already approved
console.log('Deposit tx:', result.depositHash)
console.log('Shares received:', result.shares)
```

### Deposit (manual approve)

```ts
import { YO_GATEWAY_ADDRESS } from '@yo-protocol/core'

const token = VAULTS.yoUSD.underlying.address[1]
const amount = parseTokenAmount('100', 6)

// Check allowance first
const hasAllowance = await client.hasEnoughAllowance(
  token, walletClient.account.address, YO_GATEWAY_ADDRESS, amount
)
if (!hasAllowance) {
  await client.approve(token, amount)
}

const result = await client.deposit({ vault: VAULTS.yoUSD.address, amount })
```

### Redeem from a Vault

```ts
const vault = VAULTS.yoETH

// Get user's share balance
const shares = await client.getShareBalance(vault.address, userAddress)

// Redeem all shares
const result = await client.redeem({ vault: vault.address, shares })

// Wait for receipt to check if instant or queued
const receipt = await client.waitForRedeemReceipt(result.hash)

if (receipt.instant) {
  console.log('Received assets:', receipt.assetsOrRequestId)
} else {
  console.log('Queued. Request ID:', receipt.assetsOrRequestId)
  // Check pending redemptions via API
  const pending = await client.getPendingRedemptions(vault.address, userAddress)
  console.log('Pending:', pending)
}
```

### Check All User Positions

```ts
import { formatTokenAmount } from '@yo-protocol/core'

const client = createYoClient({ chainId: 8453 })
const vaults = client.getVaults() // vaults available on Base

for (const vault of vaults) {
  const position = await client.getUserPosition(vault.address, userAddress)
  if (position.shares > 0n) {
    console.log(`${vault.symbol}: ${formatTokenAmount(position.assets, vault.underlying.decimals)} ${vault.underlying.symbol}`)
  }
}
```

### Query Vault Performance (API)

```ts
const client = createYoClient({ chainId: 1 })

const snapshot = await client.getVaultSnapshot(VAULTS.yoUSD.address)
console.log('TVL:', snapshot.stats.tvl.formatted)
console.log('7d yield:', snapshot.stats.yield['7d'])

const yieldHistory = await client.getVaultYieldHistory(VAULTS.yoUSD.address)
const tvlHistory = await client.getVaultTvlHistory(VAULTS.yoUSD.address)
```

### User Performance & History

```ts
const perf = await client.getUserPerformance(VAULTS.yoUSD.address, userAddress)
console.log('Realized P&L:', perf.realized.formatted)
console.log('Unrealized P&L:', perf.unrealized.formatted)

const history = await client.getUserHistory(VAULTS.yoUSD.address, userAddress, 10)
for (const item of history) {
  console.log(`${item.type} | ${item.assets.formatted} | ${item.txHash}`)
}
```

### Claim Merkl Rewards (Base only)

```ts
const client = createYoClient({ chainId: 8453, walletClient })

// 1. Check for claimable rewards (merges API + on-chain data)
const rewards = await client.getClaimableRewards(userAddress)
if (!rewards || !client.hasMerklClaimableRewards(rewards)) {
  console.log('No claimable rewards')
  return
}

// 2. Inspect claimable amounts
const claimable = client.getMerklTotalClaimable(rewards)
for (const [token, amount] of claimable) {
  console.log(`Claimable: ${token} = ${amount}`)
}

// 3. Claim
const result = await client.claimMerklRewards(rewards)
console.log('Claim tx:', result.hash)
```

### Prepared Transactions (Safe / Account Abstraction)

For multisig or batched execution — get raw calldata without sending.

```ts
import { createYoClient, VAULTS, parseTokenAmount, YO_GATEWAY_ADDRESS } from '@yo-protocol/core'

const client = createYoClient({ chainId: 1 })
const vault = VAULTS.yoUSD
const amount = parseTokenAmount('1000', 6)

// Option A: Separate approve + deposit
const approveTx = client.prepareApprove({
  token: vault.underlying.address[1],
  amount,
})
const depositTx = await client.prepareDeposit({
  vault: vault.address,
  amount,
  recipient: safeAddress,  // required for prepare methods
})

// Submit both to Safe
await safeSdk.createTransaction({ safeTransactionData: [approveTx, depositTx] })

// Option B: Auto-check allowance + deposit
const txs = await client.prepareDepositWithApproval({
  vault: vault.address,
  token: vault.underlying.address[1],
  owner: safeAddress,
  amount,
  recipient: safeAddress,
})
// txs = [approveTx, depositTx] or just [depositTx] if already approved

// Prepare redeem
const redeemTx = await client.prepareRedeem({
  vault: vault.address,
  shares: 1000000n,
  recipient: safeAddress,
})
```

Each `PreparedTransaction` has: `{ to: Address, data: Hex, value: bigint }`.

### Multi-Chain Setup

```ts
// Create clients for each chain
const ethClient = createYoClient({ chainId: 1 })
const baseClient = createYoClient({ chainId: 8453 })
const arbClient = createYoClient({ chainId: 42161 })

// yoUSD is available on all three chains
const ethSnapshot = await ethClient.getVaultSnapshot(VAULTS.yoUSD.address)
const baseSnapshot = await baseClient.getVaultSnapshot(VAULTS.yoUSD.address)
const arbSnapshot = await arbClient.getVaultSnapshot(VAULTS.yoUSD.address)

// Note: underlying token addresses differ per chain!
// Ethereum USDC: 0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48
// Base USDC:     0x833589fcd6edb6e08f4c7c32d4f71b54bda02913
// Arbitrum USDC: 0xaf88d065e77c8cC2239327C5EDb3A432268e5831
```

---

## Important Gotchas

1. **Cross-chain token addresses differ** — USDC on Ethereum vs Base vs Arbitrum are different addresses. Use `VAULTS[vaultId].underlying.address[chainId]` to get the correct one.
2. **Default slippage is 50 bps (0.5%)** — Override with `slippageBps` param.
3. **Redeems can be instant or queued** — Check `RedeemReceipt.instant`. If `false`, `assetsOrRequestId` is a request ID, not an asset amount.
4. **Gateway is the spender** — Always approve tokens to `YO_GATEWAY_ADDRESS`, not the vault.
5. **`getUserPoints` is deprecated** — Use `getUserPerformance(vault, user)` instead.
6. **Prepared transactions need a `recipient`** — Unlike direct calls, `prepareDeposit`/`prepareRedeem` require an explicit `recipient` address.
7. **Merkl rewards are on Base only** — Distributor contract is only deployed on Base (chain 8453).
8. **`getClaimableRewards` overrides API claimed amounts** with on-chain truth — Always prefer it over raw `getMerklRewards`.
9. **Default `partnerId` is 9999 (unattributed)** — Request your own for attribution and revenue sharing. Reach out on X: https://x.com/yield
