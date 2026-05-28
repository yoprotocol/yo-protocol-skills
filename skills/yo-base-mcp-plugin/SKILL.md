---
name: yo-base-mcp-plugin
description: >-
  Base MCP plugin for YO Protocol — ERC-4626 yield vaults with ERC-7540 async
  redemption on Ethereum (1), Base (8453), and Arbitrum (42161). Both deposit
  and withdraw route through the YO Gateway. Use when the user wants to view
  YO vaults, check positions/balances/rewards, deposit, request a redeem
  (instant or async), or claim Merkl rewards via Base MCP's `send_calls`.
  Triggers on YO Protocol, yoUSD, yoETH, yoBTC, yoEUR, yoGOLD, yoUSDT,
  "deposit into yo", "withdraw from yo", "yo vault", or "claim yo rewards".
author: yoprotocol
homepage: https://github.com/yoprotocol/yo-protocol-skills
source: https://github.com/yoprotocol/yo-protocol-skills/tree/main/skills/yo-base-mcp-plugin
---

# YO Protocol Plugin

> [!IMPORTANT]
> Complete the short Base MCP onboarding flow defined in the base `SKILL.md` before calling any YO command or endpoint. Fetch the user's wallet address only when a flow actually needs it (positions, prepare/write flows, rewards).

YO Protocol is an ERC-4626 yield aggregator with **ERC-7540-style async redemption**. **Both deposit and withdraw go through the YO Gateway** at `0xF1EeE0957267b1A474323Ff9CfF7719E964969FA` (same address on every supported chain) — not direct calls to the vault. The Gateway handles slippage protection, partner IDs, and instant/async routing. A single redeem call either settles instantly (when the vault has enough idle assets) or queues; a YO solver auto-fulfills queued redeems on-chain within ~24 hours, so the user never sends a second transaction.

This plugin has two supported execution paths:

1. **CLI-capable harnesses:** use `@yo-protocol/cli` for vault state, positions, balances, deposits, and withdraws.
1. **Chat-only or no-shell harnesses:** use the YO HTTP API (`https://api.yo.xyz/api/v1`) for reads + `/transactions/zapIn` for deposits, and Base MCP's `read_contract` / contract encoding for withdraws and Merkl claims.

There is no hosted YO MCP server. The CLI does **not** produce calldata for claiming Merkl rewards — claims always use direct ABI encoding on the Merkl Distributor. The CLI also does not bridge across chains — cross-chain deposits go through `/transactions/zapIn`.

Prefer the CLI whenever the harness has shell access.

______________________________________________________________________

## Environment detection

Routing order:

1. **Shell/terminal available** (Claude Code, Codex, Cursor terminal, etc.): use [YO CLI path](#yo-cli-path).
1. **No shell**: use [YO API + on-chain path](#yo-api--on-chain-path).

All supported flows execute on chains `1`, `8453`, `42161` (the chains where the YO Gateway and vaults are deployed). Cross-chain deposits from other source chains via `/transactions/zapIn` are best-effort — see [Chains](#chains).

______________________________________________________________________

## Chains

YO vaults, the Gateway, and all officially-supported flows live on **three chains**:

| Chain    | ID    | `send_calls` name | YO API slug | CLI `--chain` |
| -------- | ----- | ----------------- | ----------- | ------------- |
| Base     | 8453  | `base`            | `base`      | `8453`        |
| Ethereum | 1     | `ethereum`        | `ethereum`  | `1`           |
| Arbitrum | 42161 | `arbitrum`        | `arbitrum`  | `42161`       |

`@yo-protocol/core` pins `SupportedChainId = 1 | 8453 | 42161` in `constants/chains.ts`. The CLI inherits this.

**Cross-chain deposits**: `/transactions/zapIn` may accept a `fromChainId` that is not a vault chain (the response includes a `CROSS_CHAIN_ROUTE` via Stargate that bridges to a vault chain before depositing). Support is best-effort and chain-by-chain — verify with the YO team before relying on a non-vault `fromChainId` in production. The response's `tx.chainId` is the **source** chain (where the user signs); the actual YO deposit happens later on the destination vault chain.

Map any numeric `chainId` to the matching `send_calls` name before calling Base MCP.

## Vault registry

| Vault        | Address                                      | Decimals | Underlying   |
| ------------ | -------------------------------------------- | -------- | ------------ |
| `yoETH`      | `0x3A43AEC53490CB9Fa922847385D82fe25d0E9De7` | 18       | WETH         |
| `yoBTC`      | `0xbCbc8cb4D1e8ED048a6276a5E94A3e952660BcbC` | 8        | WBTC / cbBTC |
| `yoUSD`      | `0x0000000f2eB9f69274678c76222B35eEc7588a65` | 6        | USDC         |
| `yoUSD Edge` | `0x5DD8BFa6C5C68D05d25EF6143E05C11E26c4cDB7` | 6        | USDC         |
| `yoEUR`      | `0x50c749aE210D3977ADC824AE11F3c7fd10c871e9` | 6        | EURC         |
| `yoGOLD`     | `0x586675A3a46B008d8408933cf42d8ff6c9CC61a1` | 6        | PAXG / XAUT  |
| `yoUSDT`     | `0xb9a7da9e90D3B428083BAe04b860faA6325b721e` | 6        | USDT         |

Gateway contract (CLI deposit `to` field, same address on every supported chain): `0xF1EeE0957267b1A474323Ff9CfF7719E964969FA`. The user must approve the deposit token to this Gateway before calling `deposit`.

______________________________________________________________________

## YO CLI path

The CLI outputs JSON to stdout, never signs, never broadcasts.

```bash
npx @yo-protocol/cli@latest [global flags] <command> [options]
```

Global flags:

| Flag              | Value                              | Default    |
| ----------------- | ---------------------------------- | ---------- |
| `--chain <id>`    | `1`, `8453`, or `42161` (numeric)  | `1`        |
| `--rpc-url <url>` | RPC endpoint (env: `YO_RPC_URL`)   | public RPC |
| `--json`          | Force JSON output (agent compat)   | off (TUI)  |
| `--raw`           | Skip decimal formatting on bigints | off        |

**Use `--json` on every command** — without it, output is interactive/text and not parseable. Set `YO_RPC_URL` to a paid RPC (e.g. Alchemy) for any command that touches chain state; public Base RPC rate-limits aggressively.

### Read commands

```bash
# Vault list / details
yo --chain 8453 --json vaults
yo --chain 8453 --json vault yoUSD
yo --chain 8453 --json yield yoUSD       # historical APY
yo --chain 8453 --json tvl yoUSD          # historical TVL
yo --chain 8453 --json share-price yoUSD  # historical share price
yo --chain 8453 --json perf yoUSD         # vs DeFiLlama benchmark
yo --chain 8453 --json prices             # asset prices

# User position(s)
yo --chain 8453 --json position yoUSD --user 0x...
yo --chain 8453 --json portfolio --user 0x...      # all vaults across chains
yo --chain 8453 --json user-perf yoUSD --user 0x...  # P&L (realized + unrealized)
yo --chain 8453 --json pending-redeems yoUSD --user 0x...
yo --chain 8453 --json history yoUSD --user 0x... --limit 20

# Rewards (read-only)
yo --chain 8453 --json rewards --user 0x...       # claimable Merkl ($YO and others)
yo --chain 8453 --json yo-rewards --user 0x...    # $YO native rewards / season campaigns
yo --chain 8453 --json leaderboard --token 0x...

# Self-describe
yo schema
```

`yo rewards --user` calls the Merkl API for proofs + amounts and the Merkl Distributor on-chain for `claimed(user, token)`. The response shape matches Merkl's `/v4/users/{user}/rewards` (see [Claim rewards](#claim-rewards)).

### Prepare commands

```bash
yo --chain 8453 --json prepare approve --token <tokenAddr> --amount <decimal> --spender 0xF1EeE0957267b1A474323Ff9CfF7719E964969FA
yo --chain 8453 --json prepare deposit --vault yoUSD --amount 1 --recipient 0x... [--slippage-bps 50]
yo --chain 8453 --json prepare redeem  --vault yoUSD --shares 1 --recipient 0x... [--slippage-bps 50]

# Atomic approve + deposit (preferred for deposits — produces an ordered batch)
yo --chain 8453 --json prepare deposit-with-approval \
   --vault yoUSD --token <underlying> --owner 0x... --recipient 0x... --amount 1 [--slippage-bps 50]
```

`--slippage-bps` is plain basis points (`50` = 0.5%, default). Amount/shares are human-readable decimals unless `--raw` is set.

### CLI output shapes

Single-call prepare (`approve`, `deposit`, `redeem`):

```json
{ "ok": true, "result": { "to": "0x...", "data": "0x...", "value": "0" } }
```

Ordered-batch prepare (`deposit-with-approval`):

```json
{
  "ok": true,
  "result": [
    { "to": "<underlying token>", "data": "0x095ea7b3…", "value": "0" },
    { "to": "0xF1EeE0…964969FA",  "data": "0x82b78ba7…", "value": "0" }
  ]
}
```

Read commands — examples:

```json
// yo position
{ "vault": "yoUSD", "shares": "2978919", "assets": "3128133",
  "shares_formatted": "2.978919", "assets_formatted": "3.128133" }

// yo pending-redeems
{ "assets": { "raw": 0, "formatted": "0" }, "shares": { "raw": 0, "formatted": "0" } }

// yo user-perf
{ "realized":   { "raw": -20491,  "formatted": "-0.020491" },
  "unrealized": { "raw": -113273, "formatted": "-0.113273" } }
```

Read commands are not consistently wrapped in `{ ok, result }` — some are bare. Errors always look like:

```json
{ "ok": false, "error": { "code": "RPC_ERROR", "message": "..." } }
```

If `ok === false`, stop and report the error. Do not invent replacement parameters.

### CLI orchestration

```
get_wallets -> user address
yo vaults | yo vault <id>  -> pick vault
yo position <vault> --user <addr>  -> verify state
yo prepare deposit-with-approval ...  ->  [approve, deposit] array
send_calls(chain="base", calls = result[])
user approves -> get_request_status(requestId) -> confirmed
```

### `send_calls` mapping (CLI → Base MCP)

```json
{
  "chain": "base",
  "calls": [
    { "to": "<result[0].to>", "value": "<result[0].value or 0x0>", "data": "<result[0].data>" },
    { "to": "<result[1].to>", "value": "<result[1].value or 0x0>", "data": "<result[1].data>" }
  ]
}
```

Convert the CLI's `--chain` numeric ID to a Base MCP name (`8453 → base`, `1 → ethereum`, `42161 → arbitrum`). `value` is always a decimal string in CLI output — convert to `0x` hex if Base MCP requires hex (`"0" → "0x0"`).

______________________________________________________________________

## YO API + on-chain path

For chat-only surfaces, chains the CLI doesn't cover (Optimism/Avalanche/etc.), and Merkl claims.

API host: `https://api.yo.xyz/api/v1`. **Every response is wrapped in `{ data, message, statusCode }`** — agent must drill into `.data` for the payload.

If `web_request` rejects `api.yo.xyz` (custom host, may not be allowlisted), ask the user to paste the JSON response into the chat and continue.

### On-chain reads (preferred, matches the dapp)

The dapp reads user state directly from the vault contracts via `balanceOf` + `convertToAssets` and `pendingRedeemRequest` — same pattern used by `hooks/usePositionValues.ts`. **Prefer this path for positions, pending redeems, share price, and idle-balance preview** — values are block-fresh and don't depend on the YO API being reachable.

Use Base MCP's `read_contract` on the vault contract:

| What                          | Function                                                                     | Notes                                                                                                                          |
| ----------------------------- | ---------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| User shares                   | `balanceOf(user) returns uint256`                                            | ERC-20 standard                                                                                                                |
| User assets (gross, no fee)   | `convertToAssets(shares) returns uint256`                                    | **Use this, not `previewRedeem`** — `previewRedeem` deducts the withdrawal fee and understates the position. Matches the dapp. |
| Pending async redeem          | `pendingRedeemRequest(user) returns (uint256 assets, uint256 pendingShares)` | Returns `(0, 0)` when no redeem is queued                                                                                      |
| Underlying token              | `asset() returns address`                                                    |                                                                                                                                |
| Vault TVL                     | `totalAssets() returns uint256`                                              |                                                                                                                                |
| Total shares                  | `totalSupply() returns uint256`                                              | Share price = `totalAssets / totalSupply`                                                                                      |
| Pending liabilities           | `totalPendingAssets() returns uint256`                                       | Used to compute idle balance                                                                                                   |
| Redeem preview (fee-adjusted) | `previewRedeem(shares) returns uint256`                                      | Use this to predict instant vs async                                                                                           |

**Instant-vs-async detection** — read on-chain rather than guessing:

```
idleBalance ≈ ERC20.balanceOf(<underlying>, <vault>) − totalPendingAssets()
instant ↔ previewRedeem(shares) ≤ idleBalance
```

If the inequality holds, `Gateway.redeem(...)` will settle in the same tx. Otherwise it queues async; a YO solver fulfills it on-chain within ~24h (no second user tx).

Reference: this is what `dapp/src/hooks/usePositionValues.ts` does (`useChainSharesBalances` + `useChainConvertToAssetsBatch`).

### API reads (alternative / aggregations)

The YO HTTP API is best for **multi-vault aggregations** and **fields not exposed on-chain** — APY, yoRanking, P&L, historical series, Merkl yield, etc.

API host: `https://api.yo.xyz/api/v1`. **Every response is wrapped in `{ data, message, statusCode }`** — drill into `.data` for the payload.

If `web_request` rejects `api.yo.xyz` (custom host, may not be allowlisted), ask the user to paste the JSON response into the chat and continue.

```
GET /vault                                                       # list all vaults
GET /vault/stats                                                 # list + APY/TVL/sharePrice/merklRewardYield
GET /vault/stats?secondary=true                                  # include secondary vaults
GET /vault/{network}/{vaultAddress}                              # single vault detail
GET /user/positions/{userAddress}                                # ?vaultAddress=&chainId= optional filters
GET /user/balance/{walletAddress}                                # wallet token balances across chains
GET /performance/user/{network}/{vault}/{user}                   # realized + unrealized P&L
GET /vault/pending-redeems/{userAddress}                         # batch pending across all vaults
GET /vault/pending-redeems/{network}/{vault}/{user}              # single vault pending
GET /merkl/campaigns/live                                        # active YO Merkl campaigns
GET /rewards/campaign/active                                     # active YO native campaign (RLP/airdrop)
GET /rewards/{campaignName}/{userAddress}                        # user allocation for a campaign
GET /rewards/user/{userAddress}/{assetAddress}                   # YO-aggregated rewards summary
```

**Vault response shape (verified against live API).** Each `/vault/stats` entry has top-level fields (no nested `stats` object):

```json
{
  "id": "yoUSD",
  "name": "yoUSD",
  "chain": { "id": 8453, "name": "base", ... },
  "asset":     { "address": "0x833589…", "symbol": "USDC", "decimals": 6 },
  "shareAsset":{ "address": "0x0000000f…", "symbol": "yoUSD", "decimals": 6 },
  "contracts": { "vaultAddress": "0x0000000f…", "authorityAddress": "0x..." },
  "tvl":         { "raw": 18601488071914, "formatted": "18601488.071914" },
  "sharePrice":  { "raw": 1050101,         "formatted": "1.050101" },
  "yield":       { "1d": "4.25", "7d": "4.15", "30d": "4.04" },
  "merklRewardYield": "7.8",
  "secondaryVaults": []
}
```

`yield.*` and `merklRewardYield` are **percentage strings** (e.g. `"4.15"` = 4.15% APY). `tvl.formatted` is in display units. `idleBalance` and `withdrawalFee` are **not** in the response — get those on-chain.

Other shapes:

```json
// /user/positions  →  data: [{ vaultId, vaultAddress, chainId, chainName, asset: {...}, position: { shares: {raw,formatted}, assets: {raw,formatted} } }, ...]
// /user/balance    →  data: { totalBalanceUsd, assets: [{ chainId, contractAddress, symbol, decimals, type, balance, balanceRaw, balanceUsd, price }] }
// /vault/pending-redeems/{user}  →  data: [{ vaultAddress, vaultName, network, chainId, status, assets: {...}, shares: {...} }, ...]
// /performance/user/...           →  data: { realized: {raw,formatted}, unrealized: {raw,formatted} }
```

The API's user position (`assets`) and on-chain `convertToAssets(shares)` agree to within one block of share-price drift (≈30 wei on a yoUSD position). Pick whichever is convenient — on-chain wins on freshness, API wins on aggregating across vaults/chains in a single round-trip.

### Deposit — `/transactions/zapIn`

The only public GET that returns unsigned deposit calldata. Handles same-chain and cross-chain deposits.

```
GET https://api.yo.xyz/api/v1/transactions/zapIn
    ?userAddress=<address>
    &fromChainId=<chainId of srcAsset>
    &toChainId=<chainId of vault>
    &srcAsset=<token address, or 0x0000000000000000000000000000000000000000 for native ETH>
    &dstAsset=<vault address — NOT the underlying>
    &amountIn=<raw integer string in srcAsset's smallest unit; no scientific notation>
    &slippage=<basis points, integer — 25 means 0.25%>
```

Response (verified):

```json
{
  "data": {
    "tx": { "to": "0xF1EeE0…964969FA", "from": "<user>", "data": "0x82b78ba7…", "value": "0", "chainId": 8453 },
    "gasLimit": "200000",
    "amountOut": "952289",
    "assetOut": "0x0000000f…",
    "steps": [ { "protocol": "yo", "type": "DEPOSIT", "chainId": 8453, "assetIn": "...", "assetOut": "..." } ],
    "numberOfSteps": 1,
    "type": "NATIVE"
  },
  "message": "SUCCESS",
  "statusCode": 200
}
```

`tx.to` is the YO Gateway. `tx.value` is a decimal string (convert to `0x0` hex if Base MCP requires hex; `value` defaults to `0x0` when omitted from the `send_calls` call).

### `send_calls` mapping (API deposit)

For an ERC-20 deposit, prepend an `approve(<tx.to>, amountIn)` call on `srcAsset`. For native ETH (`srcAsset == 0x0000…`), omit the approve and forward `tx.value`.

```json
{
  "chain": "base",
  "calls": [
    { "to": "<srcAsset>", "value": "0x0", "data": "<encoded approve(tx.to, amountIn)>" },
    { "to": "<tx.to>",    "value": "<tx.value or 0x0>", "data": "<tx.data>" }
  ]
}
```

### Withdraw — Gateway `redeem`

Use the Gateway, **not** direct `vault.requestRedeem`. The Gateway handles slippage, partner attribution, and instant/async routing internally.

Gateway function:

```
function redeem(
  address vault,
  uint256 shares,
  uint256 minAssetsOut,
  address recipient,
  uint256 partnerId
) returns (uint256)
```

- `minAssetsOut` = `applySlippage(quotePreviewRedeem(vault, shares), slippageBps)`. Get `quotePreviewRedeem` from the Gateway (not the vault — Gateway's preview is fee-adjusted with the right basis).
- `partnerId` defaults to `9999` per the SDK; pass `0` if you don't have an assigned partner ID.

**The Gateway must be approved to spend the user's vault shares.** The SDK's `prepareRedeemWithApproval` checks `vault.allowance(user, Gateway)`; if less than `shares`, it prepends an `approve(Gateway, maxUint256)` on the vault (one-time per user, Uniswap-pattern). The CLI's `yo prepare redeem` does **not** include this approval — issue a separate `yo prepare approve --token <vault> --spender <gateway> --amount <max>` on first use, or use the SDK directly.

`send_calls` mapping (with first-time approve):

```json
{
  "chain": "base",
  "calls": [
    { "to": "<vault>", "value": "0x0",
      "data": "<encoded approve(Gateway, maxUint256)>" },
    { "to": "0xF1EeE0957267b1A474323Ff9CfF7719E964969FA", "value": "0x0",
      "data": "<encoded redeem(vault, shares, minAssetsOut, user, 9999)>" }
  ]
}
```

After submission, monitor with `pendingRedeemRequest(user)` on the vault, or `GET /vault/pending-redeems/.../{user}`. When `pendingShares == 0` and the user's underlying balance has grown, the redeem has settled. **No user-side claim transaction is required for async redeems** — a YO solver calls `fulfillRedeem` on-chain.

### Claim rewards

**YO Merkl rewards always claim on Base**, regardless of the chain that earned them.

```
GET https://api.merkl.xyz/v4/users/{userAddress}/rewards?chainId=8453
```

Returns an **array** of per-chain wrappers (verified):

```json
[
  {
    "chain": { "id": 8453, "name": "Base", ... },
    "rewards": [
      {
        "root": "0x28f441bf…",
        "distributionChainId": 8453,
        "recipient": "0x77B4922F…9a149",
        "amount":  "14172098606012883675",   // cumulative lifetime
        "claimed": "10951906851596507470",   // already claimed on-chain
        "pending":     "4273406001446877",   // in dispute period — NOT yet claimable
        "token":   { "address": "0x1925450f…1A72", "symbol": "YO", "decimals": 18, ... },
        "proofs":  [ "0x88549…", "0x26387…", … ],
        "breakdowns": [ ... ]
      }
    ]
  }
]
```

Claimable now = `amount − claimed`. `pending` is rewards still in the dispute period and is **not** included in the claim — pass the full `amount` to the distributor; the contract handles subtracting `claimed`.

Distributor on Base: `0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae`.

```
function claim(
  address[] users,
  address[] tokens,
  uint256[] amounts,
  bytes32[][] proofs
)
```

For each reward where `amount > claimed`, push one element into each parallel array. All elements of `users` are typically the same wallet address.

```json
{
  "chain": "base",
  "calls": [
    { "to": "0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae", "value": "0x0",
      "data": "<encoded claim([user, …], [token, …], [amount, …], [proof, …])>" }
  ]
}
```

For YO-native campaign rewards (RLP, airdrops), use `GET /rewards/{campaignName}/{userAddress}` to get `{ tokenAddress, tokenAmount, proof[] }` and the per-campaign `UniversalRewardsDistributor` address, then call `claim(account, reward, claimable, proof)` on that distributor.

______________________________________________________________________

## Example prompts

### "Show YO vaults by APY"

CLI: `yo --chain 8453 --json vaults` → sort by `yield['7d']` (parse as float).
API: `GET /vault/stats` → drill into `data[]`, sort by `yield['7d']`.

### "What's my position in yoUSD?"

On-chain (preferred, block-fresh):

```
read_contract balanceOf(<user>)              on yoUSD vault  → shares
read_contract convertToAssets(shares)        on yoUSD vault  → assets (gross)
read_contract pendingRedeemRequest(<user>)   on yoUSD vault  → queued (assets, pendingShares)
# Optional, for P&L: GET /performance/user/base/<yoUSD>/<user>
```

CLI (same on-chain reads under the hood):

```
yo --chain 8453 --json position yoUSD --user <addr>
yo --chain 8453 --json pending-redeems yoUSD --user <addr>
yo --chain 8453 --json user-perf yoUSD --user <addr>   # for P&L
```

API (when aggregating across multiple vaults/chains in one request):

```
GET /user/positions/<addr>                                     # all vaults across all chains
GET /vault/pending-redeems/<addr>                              # batch pending
GET /performance/user/base/<yoUSD>/<addr>                      # realized + unrealized
```

### "Deposit 1 USDC into yoUSD on Base"

CLI:

```
yo --chain 8453 --json prepare deposit-with-approval \
   --vault yoUSD --token 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
   --owner <user> --recipient <user> --amount 1 --slippage-bps 50
send_calls(chain="base", calls = result[])
```

API:

```
GET /user/balance/<user>   # confirm USDC balance ≥ 1
GET /transactions/zapIn?userAddress=<user>&fromChainId=8453&toChainId=8453
    &srcAsset=0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913
    &dstAsset=0x0000000f2eB9f69274678c76222B35eEc7588a65
    &amountIn=1000000&slippage=25
send_calls(chain="base", calls = [
  { to: srcAsset, value: "0x0", data: encoded approve(tx.to, 1000000) },
  { to: data.tx.to, value: "0x0", data: data.tx.data }
])
```

### "Withdraw all my yoUSD"

On-chain (preferred — verify state before encoding):

```
read_contract balanceOf(<user>)               on yoUSD vault   → shares
read_contract previewRedeem(shares)           on yoUSD vault   → expected assets (fee-adjusted)
read_contract totalPendingAssets()            on yoUSD vault
read_contract balanceOf(<vault>)              on underlying    → vault's underlying balance
# idleBalance ≈ vault.underlying balanceOf − totalPendingAssets
# instant if previewRedeem(shares) ≤ idleBalance, otherwise ~24h async
read_contract allowance(<user>, Gateway)      on yoUSD vault   → if < shares, prepend approve(Gateway, maxUint256)

# Encode and submit
send_calls(chain="base", calls = [
  (optional) { to: <vault>, value: "0x0", data: encoded approve(Gateway, maxUint256) },
  {            to: 0xF1EeE0…964969FA, value: "0x0",
                data: encoded Gateway.redeem(<vault>, shares, minAssetsOut, <user>, 9999) }
])
```

CLI (same Gateway path under the hood):

```
yo --chain 8453 --json position yoUSD --user <addr>      # shares
# First-time withdraw only: approve vault shares to the Gateway
yo --chain 8453 --json prepare approve --token <yoUSD vault> --spender 0xF1EeE0…964969FA --amount <max> --decimals 6
yo --chain 8453 --json prepare redeem  --vault yoUSD --shares <shares> --recipient <user> --slippage-bps 50
send_calls(chain="base", calls = [approve?, redeem])
```

### "Claim my YO Merkl rewards"

Both paths:

```
GET https://api.merkl.xyz/v4/users/<user>/rewards?chainId=8453
# Filter rewards where amount > claimed
send_calls(chain="base", calls = [{
  to: 0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae,
  value: "0x0",
  data: encoded claim([user,...], [token,...], [amount,...], [proofs,...])
}])
```

______________________________________________________________________

## Quirks and gotchas

- **All YO API responses wrap the payload in `{ data, message, statusCode }`** — drill into `.data` first.
- **Vault fields are top-level** (`yield`, `tvl`, `sharePrice`, `merklRewardYield`, `contracts`, `asset`). There is no nested `stats` object.
- **`idleBalance` and `withdrawalFee` are not in the API** — read them on-chain when needed.
- **API `slippage` is plain basis points** (`25` = 0.25%, dapp default). **CLI `--slippage-bps` is also plain basis points** but defaults to `50` (0.5%).
- **`dstAsset` in `/transactions/zapIn` is always the vault address**, never the underlying token.
- **`amountIn` must be a raw integer string** in the source token's smallest unit. Never use JS scientific notation; the backend rejects it.
- **For position display, use `convertToAssets(shares)`**, not `previewRedeem(shares)` — the latter deducts the withdrawal fee.
- **`requestRedeem` needs no approval** — the vault burns the caller's own shares.
- **Async redeems are solver-fulfilled**; users never call `redeem` / `fulfillRedeem` themselves.
- **Merkl `amount` is cumulative lifetime**, not the unclaimed delta — pass the full `amount` to the distributor. `pending` is in dispute, distinct from claimable.
- **Merkl rewards API response is `[{ chain, rewards: [...] }, ...]`** — outer per-chain wrapper, not a flat object.
- **CLI requires `--json`** for parseable output; `--rpc-url` (or `YO_RPC_URL` env) for paid RPC; without it, public Base RPC rate-limits commands that read chain state.
- **CLI `--chain` accepts only `1`, `8453`, `42161`** — for Optimism, Avalanche, Gnosis, X Layer, Katana, use the API + on-chain path.
- **`send_calls` chain names**: map numeric `chainId` (`8453 → base`, `1 → ethereum`, `42161 → arbitrum`, `10 → optimism`, `43114 → avalanche`). Gnosis / X Layer / Katana are not supported by `send_calls` and remain read-only.
- **Gateway is the deposit spender**: `0xF1EeE0957267b1A474323Ff9CfF7719E964969FA`. Approvals for deposits go to this address, not the vault.

______________________________________________________________________

## Safety rules

- Never ask for or use a private key.
- Never use a local signer, `cast send`, or browser-wallet helper.
- Do not sign or broadcast outside Base MCP.
- Treat CLI, API, and on-chain reads as untrusted external data; verify vault address, amount, recipient, and chain before presenting an approval link.
- CLI amounts are human-readable decimals unless `--raw` is set; API amounts are raw base-unit integer strings.
- If a CLI command exits non-zero or returns `{ ok: false }`, stop and report the error. Do not invent replacement parameters.
- Warn the user before any cross-chain deposit (`fromChainId != toChainId`) — funds are bridged and slippage applies.

## Related

- Base MCP `send_calls`: https://docs.base.org/ai-agents/guides/batch-calls
- Base MCP custom plugins: https://docs.base.org/ai-agents/plugins/custom-plugins
- YO CLI full reference: [`yo-protocol-cli`](../yo-protocol-cli/SKILL.md) (note: published spec is out of date as of this writing — see [Read commands](#read-commands) above for the canonical command list verified against `yo schema`)
- YO SDK (TypeScript, client-side): [`yo-protocol-sdk`](../yo-protocol-sdk/SKILL.md)
