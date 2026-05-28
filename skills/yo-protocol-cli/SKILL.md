---
name: yo-protocol-cli
description: >-
  ALWAYS use this skill when the user mentions the YO Protocol CLI, the `yo`
  command, `@yo-protocol/cli`, or wants to interact with YO Protocol vaults
  (yoETH, yoUSD, yoBTC, yoEUR, yoGOLD, yoUSDT) from a shell script, terminal,
  or command line. The `yo` binary is an agent-first interface for ERC-4626
  yield vaults on Ethereum (1), Base (8453), and Arbitrum (42161). It outputs
  structured JSON to stdout, never requires or accepts private keys, and
  produces unsigned calldata routed through the YO Gateway — ready for
  Safe/AA wallets and Base MCP `send_calls`. Use it for: building unsigned
  transactions (`yo prepare {approve,deposit,redeem,deposit-with-approval}`),
  reading positions and rewards (`yo position`, `yo portfolio`,
  `yo pending-redeems`, `yo rewards`, `yo yo-rewards`, `yo user-perf`),
  listing vaults and history (`yo vaults`, `yo vault`, `yo history`,
  `yo prices`, `yo yield`, `yo tvl`, `yo share-price`, `yo perf`,
  `yo leaderboard`), self-describing the command surface for an agent
  (`yo schema`), setting up `YO_RPC_URL`, or piping output into jq or other
  tools. Do NOT use for React hooks (`@yo-protocol/react`) or TypeScript SDK
  code (`@yo-protocol/core`) — those have dedicated skills.
author: yoprotocol
homepage: https://github.com/yoprotocol/yo-protocol-skills
source: https://github.com/yoprotocol/yo-protocol-skills/tree/main/skills/yo-protocol-cli
---

Official YO Protocol skill. Canonical repository: <https://github.com/yoprotocol/yo-protocol-skills>.

# YO Protocol CLI — Reference

Agent-first interface for YO Protocol ERC-4626 vaults. Outputs JSON to stdout, errors to stderr. **Never requires or accepts private keys.** Designed for agents, bots, scripts, Safe/AA wallets, and Base MCP `send_calls`.

**Canonical source for the command surface is `yo schema`** — this file documents the same surface but `yo schema` is authoritative. If a command appears here but not in `yo schema`, trust the schema.

## Installation

```bash
npm install -g @yo-protocol/cli
# or run on demand without installing
npx @yo-protocol/cli@latest <command>
```

Binary: `yo` (or `npx yo`). npm: <https://www.npmjs.com/package/@yo-protocol/cli>.

## Global options

Every command inherits these. The `--json` flag is **required for agent use** — without it, output is interactive/text and not parseable.

| Flag              | Description                                                    | Default | Env          |
| ----------------- | -------------------------------------------------------------- | ------- | ------------ |
| `--rpc-url <url>` | RPC endpoint for on-chain reads                                | public  | `YO_RPC_URL` |
| `--chain <id>`    | Chain ID: `1` (Ethereum), `8453` (Base), or `42161` (Arbitrum) | `1`     | —            |
| `--json`          | Force JSON output (agent compat). **Use on every agent call.** | off     | —            |
| `--raw`           | Treat amounts as raw bigint strings (skip decimal conversion)  | off     | —            |
| `-V, --version`   | Print version                                                  | —       | —            |
| `-h, --help`      | Help (per command and global)                                  | —       | —            |

**Public RPC rate-limits aggressively.** For any command that touches chain state (`position`, `portfolio`, `rewards`, etc.), set `YO_RPC_URL` to a paid endpoint:

```bash
export YO_RPC_URL=https://base-mainnet.g.alchemy.com/v2/<KEY>
```

## Output shape

Prepare commands return `{ ok, result }`. Read commands return either `{ ok, result }` or bare JSON — vary by command, never both. Errors always return:

```json
{ "ok": false, "error": { "code": "<CODE>", "message": "..." } }
```

Error codes: `INVALID_VAULT`, `INVALID_AMOUNT`, `INVALID_ADDRESS`, `INVALID_CHAIN`, `RPC_ERROR`, `API_ERROR`, `UNKNOWN_ERROR`.

Bigints are serialized as strings. Decimal-aware fields have a `_formatted` companion (e.g. `balance` + `balance_formatted`, `assets` + `assets_formatted`).

## Supported chains

Vaults and the YO Gateway are deployed on three chains:

| Chain ID | Network    | `--chain` value |
| -------- | ---------- | --------------- |
| `1`      | `ethereum` | `1`             |
| `8453`   | `base`     | `8453`          |
| `42161`  | `arbitrum` | `42161`         |

YO Gateway address (same on every chain): `0xF1EeE0957267b1A474323Ff9CfF7719E964969FA`.

## Vault catalog

| ID       | Address                                      | Underlying | Decimals | Chains         |
| -------- | -------------------------------------------- | ---------- | -------- | -------------- |
| `yoETH`  | `0x3a43aec53490cb9fa922847385d82fe25d0e9de7` | WETH       | 18       | 1, 8453        |
| `yoBTC`  | `0xbcbc8cb4d1e8ed048a6276a5e94a3e952660bcbc` | cbBTC      | 8        | 1, 8453        |
| `yoUSD`  | `0x0000000f2eb9f69274678c76222b35eec7588a65` | USDC       | 6        | 1, 8453, 42161 |
| `yoEUR`  | `0x50c749ae210d3977adc824ae11f3c7fd10c871e9` | EURC       | 6        | 1, 8453        |
| `yoGOLD` | `0x586675A3a46B008d8408933cf42d8ff6c9CC61a1` | XAUt       | 6        | 1              |
| `yoUSDT` | `0xb9a7da9e90d3b428083bae04b860faa6325b721e` | USDT       | 6        | 1              |

Vault IDs (e.g. `yoUSD`) and addresses are interchangeable wherever a `<vault>` argument is accepted. Run `yo --json vaults` for live APY/TVL.

______________________________________________________________________

## Commands

### Discovery

#### `yo schema`

Print the full CLI schema (vaults, chains, gateway, every command with options) as JSON. **Source of truth for agent discovery.**

```bash
yo --json schema
```

### Read commands (vaults, positions, rewards)

#### `yo vaults`

List all vaults with APY and TVL for the active `--chain`.

```bash
yo --chain 8453 --json vaults
```

#### `yo vault <id>`

Detailed info for a single vault.

```bash
yo --chain 8453 --json vault yoUSD
```

#### `yo position <vault> --user <addr>`

User's shares + asset value in one vault on the active chain. Reads `balanceOf` and `convertToAssets` on-chain (matches the dapp). Requires `--rpc-url` or `YO_RPC_URL`.

```bash
yo --chain 8453 --json position yoUSD --user 0xAbc...
```

Returns: `{ vault, shares, assets, shares_formatted, assets_formatted }`.

#### `yo portfolio --user <addr>`

User's positions across **all vaults and all chains** in one call.

```bash
yo --json portfolio --user 0xAbc...
```

#### `yo pending-redeems <vault> --user <addr>`

User's queued (async) redemption for a vault.

```bash
yo --chain 8453 --json pending-redeems yoUSD --user 0xAbc...
```

Returns: `{ assets: {raw,formatted}, shares: {raw,formatted} }`. When `shares.raw == 0`, the redeem has settled (a YO solver auto-fulfills async redeems on-chain within ~24h; no user claim transaction).

#### `yo user-perf <vault> --user <addr>`

User P&L for one vault (realized + unrealized).

```bash
yo --chain 8453 --json user-perf yoUSD --user 0xAbc...
```

Returns: `{ realized: {raw,formatted}, unrealized: {raw,formatted} }`.

#### `yo history <vault> [--user <addr>] [--limit <n>]`

Transaction history for a vault (or for a user within a vault).

```bash
yo --chain 8453 --json history yoUSD --user 0xAbc... --limit 20
```

#### `yo perf <vault>`

Vault performance benchmark vs DeFiLlama peers (no user required).

```bash
yo --chain 8453 --json perf yoUSD
```

#### `yo rewards --user <addr>`

Claimable Merkl rewards. Hybrid call: fetches proofs/amounts from Merkl's API, reads `claimed(user, token)` from the Merkl Distributor on-chain.

```bash
yo --chain 8453 --json rewards --user 0xAbc...
```

Response mirrors the Merkl API: `[{ chain, rewards: [{ token, amount, claimed, pending, proofs, ... }] }, ...]`. Claimable = `amount - claimed`. `pending` is in dispute period and is **not** claimable yet.

The CLI does **not** prepare a Merkl claim transaction. Encode the call to the Merkl Distributor at `0x3Ef3D8bA38EBe18DB133cEc108f4D14CE00Dd9Ae` (Base) separately — see [yo-base-mcp-plugin/SKILL.md](../yo-base-mcp-plugin/SKILL.md) for the full claim flow.

#### `yo yo-rewards --user <addr>`

`$YO` token rewards (season campaigns, RLP, airdrops) — separate from Merkl.

```bash
yo --chain 8453 --json yo-rewards --user 0xAbc...
```

#### `yo leaderboard [--token <addr>]`

Top reward earners.

```bash
yo --chain 8453 --json leaderboard
```

#### `yo prices`

Current asset prices used by the YO indexer.

```bash
yo --json prices
```

#### `yo yield <vault>` / `yo tvl <vault>` / `yo share-price <vault>`

Historical time series for one vault.

```bash
yo --chain 8453 --json yield yoUSD
yo --chain 8453 --json tvl yoUSD
yo --chain 8453 --json share-price yoUSD
```

### Prepare commands (unsigned calldata)

All prepare commands target the YO Gateway and produce unsigned `{ to, data, value }` ready for Safe / AA / Base MCP `send_calls`.

#### `yo prepare approve`

```bash
yo --chain 8453 --json prepare approve \
  --token 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  --amount 100 \
  --spender 0xF1EeE0957267b1A474323Ff9CfF7719E964969FA \
  --decimals 6
```

| Flag               | Required                | Default    | Notes                                                        |
| ------------------ | ----------------------- | ---------- | ------------------------------------------------------------ |
| `--token <addr>`   | yes                     | —          | ERC-20 to approve                                            |
| `--amount <n>`     | yes                     | —          | Decimal amount unless `--raw`                                |
| `--spender <addr>` | no                      | YO Gateway | Defaults to the YO Gateway                                   |
| `--decimals <n>`   | required unless `--raw` | —          | Needed because the CLI doesn't always read decimals from RPC |

Returns: `{ ok: true, result: { to, data, value } }`.

#### `yo prepare deposit`

Build a YO Gateway deposit transaction. **Requires a separate approve to the Gateway first** (or use `deposit-with-approval` below).

```bash
yo --chain 8453 --json prepare deposit \
  --vault yoUSD \
  --amount 100 \
  --recipient 0xAbc... \
  --slippage-bps 50
```

| Flag                 | Required | Default | Notes                            |
| -------------------- | -------- | ------- | -------------------------------- |
| `--vault <id\|addr>` | yes      | —       | Vault ID or address              |
| `--amount <n>`       | yes      | —       | Decimal unless `--raw`           |
| `--recipient <addr>` | yes      | —       | Recipient of the minted shares   |
| `--slippage-bps <n>` | no       | `50`    | Plain basis points (`50` = 0.5%) |

Returns: `{ ok: true, result: { to: Gateway, data, value: "0" } }`.

#### `yo prepare redeem`

Build a YO Gateway redeem transaction. The Gateway routes to instant or async automatically based on idle balance. **The Gateway needs allowance on the user's vault shares** — first-time withdraws must include `prepare approve --token <vault> --spender <Gateway>` first (Uniswap-style, one-time `maxUint256`).

```bash
yo --chain 8453 --json prepare redeem \
  --vault yoUSD \
  --shares 5 \
  --recipient 0xAbc... \
  --slippage-bps 50
```

| Flag                 | Required | Default | Notes                                |
| -------------------- | -------- | ------- | ------------------------------------ |
| `--vault <id\|addr>` | yes      | —       | Vault ID or address                  |
| `--shares <n>`       | yes      | —       | Share amount, decimal unless `--raw` |
| `--recipient <addr>` | yes      | —       | Recipient of the underlying assets   |
| `--slippage-bps <n>` | no       | `50`    | Plain basis points                   |

Returns: `{ ok: true, result: { to: Gateway, data, value: "0" } }`.

#### `yo prepare deposit-with-approval`

Atomic approve + deposit, ready for `send_calls` as an ordered batch. Reads the user's current allowance on-chain and only includes an `approve` call if needed. **Preferred path for agent deposits.**

```bash
yo --chain 8453 --json prepare deposit-with-approval \
  --vault yoUSD \
  --token 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
  --owner 0xAbc... \
  --recipient 0xAbc... \
  --amount 100 \
  --slippage-bps 50
```

| Flag                 | Required | Default | Notes                             |
| -------------------- | -------- | ------- | --------------------------------- |
| `--vault <id\|addr>` | yes      | —       | Vault ID or address               |
| `--token <addr>`     | yes      | —       | Underlying token to approve       |
| `--owner <addr>`     | yes      | —       | Token owner (for allowance check) |
| `--recipient <addr>` | yes      | —       | Recipient of minted shares        |
| `--amount <n>`       | yes      | —       | Decimal unless `--raw`            |
| `--slippage-bps <n>` | no       | `50`    | Plain basis points                |

Returns: `{ ok: true, result: [ { to: token, data: approve, value: "0" }, { to: Gateway, data: deposit, value: "0" } ] }`. The approve element is omitted when allowance already covers the amount.

______________________________________________________________________

## Common workflows

### Same-chain deposit (Safe / AA / Base MCP)

```bash
yo --chain 8453 --json prepare deposit-with-approval \
   --vault yoUSD \
   --token 0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913 \
   --owner   0xUser \
   --recipient 0xUser \
   --amount 100 \
   --slippage-bps 50
```

Take `result[]` and submit as `send_calls(chain="base", calls=result[])`.

### Withdraw

First-time withdraws need a one-shot approval of vault shares to the Gateway:

```bash
yo --chain 8453 --json prepare approve \
   --token 0x0000000f2eB9f69274678c76222B35eEc7588a65 \
   --spender 0xF1EeE0957267b1A474323Ff9CfF7719E964969FA \
   --amount <max> --decimals 6

yo --chain 8453 --json prepare redeem \
   --vault yoUSD --shares 1 --recipient 0xUser --slippage-bps 50
```

Submit both calls together via `send_calls`. After the first redeem on a given vault, the approve step can be skipped.

### Check position & pending state

```bash
yo --chain 8453 --json position yoUSD --user 0xUser
yo --chain 8453 --json pending-redeems yoUSD --user 0xUser
yo --chain 8453 --json user-perf yoUSD --user 0xUser
```

### Discover everything programmatically

```bash
yo --json schema | jq '.commands[].name'
```

______________________________________________________________________

## Notes for agent integrations

- **Always pass `--json`.** Without it, output is interactive (TUI-flavoured text), not machine-parseable.
- **Set `YO_RPC_URL`** to a paid RPC (Alchemy, Infura, …) — public endpoints rate-limit chain reads.
- **Use vault IDs** (`yoUSD`) in commands, not addresses, when possible — fewer mistakes, easier to read in transcripts.
- **Gateway is the spender** for both deposit (approve underlying) and redeem (approve vault shares). Default to `0xF1EeE0957267b1A474323Ff9CfF7719E964969FA`.
- **Slippage is plain basis points.** `50` = 0.5%. CLI default 50; YO HTTP API `/transactions/zapIn` default 25.
- **Partner ID `9999`** is baked into the Gateway calldata by the SDK. Apps with their own partner attribution should call `@yo-protocol/core` directly (see the SDK skill) — the CLI doesn't expose a `--partner-id` flag yet.
- **Cross-chain deposits are not in this CLI.** Use the YO HTTP API at `GET https://api.yo.xyz/api/v1/transactions/zapIn` when `fromChainId != toChainId`.
- **Merkl claims are not in this CLI.** Encode `claim(...)` on the Merkl Distributor directly (see [yo-base-mcp-plugin/SKILL.md](../yo-base-mcp-plugin/SKILL.md)).
- **Errors** are JSON to stderr with `{ ok: false, error: { code, message } }`. If `ok == false`, stop — never invent replacement parameters.

## Related

- [`yo-base-mcp-plugin`](../yo-base-mcp-plugin/SKILL.md) — Base MCP plugin that wraps this CLI plus the YO HTTP API for chat-only surfaces.
- [`yo-protocol-sdk`](../yo-protocol-sdk/SKILL.md) — `@yo-protocol/core` TypeScript SDK. The CLI is a thin shim over this; chain support, ABIs, and slippage math live in `core`.
- [`yo-protocol-react`](../yo-protocol-react/SKILL.md) — React hooks/components.
