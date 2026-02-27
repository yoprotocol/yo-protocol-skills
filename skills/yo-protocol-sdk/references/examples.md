# Workflow Examples

## Deposit into a Vault

**Always use separate `approve()` → wait for confirmation → `deposit()`.** Do NOT use `depositWithApproval()`.

```ts
import { createYoClient, VAULTS, parseTokenAmount, YO_GATEWAY_ADDRESS } from '@yo-protocol/core'

const client = createYoClient({ chainId: 1, walletClient })

// 1. Check vault state
const vault = VAULTS.yoUSD
const paused = await client.isPaused(vault.address)
if (paused) throw new Error('Vault is paused')

// 2. Parse amount (100 USDC)
const token = vault.underlying.address[1]  // USDC on Ethereum
const amount = parseTokenAmount('100', vault.underlying.decimals) // 100_000_000n

// 3. Check allowance and approve if needed
const hasAllowance = await client.hasEnoughAllowance(
  token, walletClient.account.address, YO_GATEWAY_ADDRESS, amount
)
if (!hasAllowance) {
  const approveResult = await client.approve(token, amount)
  // IMPORTANT: Wait for approve tx to confirm before depositing!
  await client.waitForTransaction(approveResult.hash)
}

// 4. Deposit (only after approval is confirmed on-chain)
const result = await client.deposit({ vault: vault.address, amount })
console.log('Deposit tx:', result.hash)
console.log('Shares received:', result.shares)
```

## Redeem from a Vault

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

## Check All User Positions

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

## Query Vault Performance (API)

```ts
const client = createYoClient({ chainId: 1 })

const snapshot = await client.getVaultSnapshot(VAULTS.yoUSD.address)
console.log('TVL:', snapshot.stats.tvl.formatted)
console.log('7d yield:', snapshot.stats.yield['7d'])

const yieldHistory = await client.getVaultYieldHistory(VAULTS.yoUSD.address)
const tvlHistory = await client.getVaultTvlHistory(VAULTS.yoUSD.address)
```

## User Performance & History

```ts
const perf = await client.getUserPerformance(VAULTS.yoUSD.address, userAddress)
console.log('Realized P&L:', perf.realized.formatted)
console.log('Unrealized P&L:', perf.unrealized.formatted)

const history = await client.getUserHistory(VAULTS.yoUSD.address, userAddress, 10)
for (const item of history) {
  console.log(`${item.type} | ${item.assets.formatted} | ${item.txHash}`)
}
```

## Claim Merkl Rewards (Base only)

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

## Prepared Transactions (Safe / Account Abstraction)

For multisig or batched execution — get raw calldata without sending. **Always use separate `prepareApprove()` + `prepareDeposit()`.**

```ts
import { createYoClient, VAULTS, parseTokenAmount, YO_GATEWAY_ADDRESS } from '@yo-protocol/core'

const client = createYoClient({ chainId: 1 })
const vault = VAULTS.yoUSD
const amount = parseTokenAmount('1000', 6)

// Build approve + deposit as separate prepared transactions
const approveTx = client.prepareApprove({
  token: vault.underlying.address[1],
  amount,
})
const depositTx = await client.prepareDeposit({
  vault: vault.address,
  amount,
  recipient: safeAddress,  // required for prepare methods
})

// Submit both to Safe as a batch
await safeSdk.createTransaction({ safeTransactionData: [approveTx, depositTx] })

// Prepare redeem
const redeemTx = await client.prepareRedeem({
  vault: vault.address,
  shares: 1000000n,
  recipient: safeAddress,
})
```

Each `PreparedTransaction` has: `{ to: Address, data: Hex, value: bigint }`.

## Multi-Chain Setup

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
