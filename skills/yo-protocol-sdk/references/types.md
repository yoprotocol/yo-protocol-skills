# TypeScript Interfaces

## Core Types

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

## Vault Types

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

## User Types

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

## Action Results

```ts
interface DepositResult {
  hash: Hash
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

## API Types

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

## Merkl Types

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
