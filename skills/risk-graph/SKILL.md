---
name: risk-graph
description: >-
  Use Risk Graph's paid agent API (`/api/v1/agent/*`) — a DeFi risk-intelligence
  graph covering pools, assets, protocols, and chains, monetised via the
  [x402](https://x402.org) micropayment standard over USDC on Base mainnet
  (chain id `8453`). Every endpoint costs between $0.001 and $0.75 per call;
  the payer pays no gas — settlement is EIP-3009 `transferWithAuthorization`
  via the Coinbase CDP facilitator. Use whenever an agent needs structured,
  programmatic DeFi-risk data and is willing to pay per call: ranking pools by
  risk, comparing protocols, fetching chain-level risk signals, mapping
  dependency chains between pools/assets, extracting subgraphs around a node,
  explaining why a node is risky, or building an autonomous agent that
  consumes on-chain risk intelligence. Make sure to use this skill whenever
  the user mentions Risk Graph, risk-graph, DeFi risk score, pool risk,
  protocol risk, chain risk, asset risk, node risk explain, subgraph
  extraction, dependency chain, `/api/v1/agent/`, x402, paid agent API, USDC
  paywall, EIP-3009 paywall, Coinbase CDP facilitator, or building AI agents
  that consume on-chain risk intelligence — even if they don't explicitly
  name "Risk Graph". Triggers on mentions of Risk Graph, risk-graph API,
  DeFi risk scoring, x402 paywall, USDC micropayments on Base, agent-facing
  DeFi data, pool/protocol/asset/chain risk lookup, dependency-graph traversal,
  subgraph extraction, or paid `/api/v1/agent/*` endpoints.
author: yoprotocol
homepage: https://github.com/yoprotocol/yo-protocol-skills
source: https://github.com/yoprotocol/yo-protocol-skills/tree/main/skills/risk-graph
---

Official Yo Protocol skill.
Canonical repository: https://github.com/yoprotocol/yo-protocol-skills

# Risk Graph Agent API — Complete Reference

Risk Graph quantifies DeFi risk as a graph: every Pool, Asset, Protocol, and
Chain is a node, every dependency between them (collateral, allocations,
issuance, hosting, etc.) is a typed edge, and every node carries a composite
risk score that propagates through its neighborhood.

The `/api/v1/agent/*` namespace is the **public, paid, machine-readable**
surface designed for autonomous consumption. It is monetised via the
[x402 payment standard](https://x402.org) v2 — agents pay tiny amounts of
USDC per call, no API keys, no signup.

## Base URL

```
https://risk-graph-gx7v4.ondigitalocean.app
```

Every endpoint below resolves to `https://risk-graph-gx7v4.ondigitalocean.app/api/v1/agent/...`. The 402 challenge's `resource.url` field is the canonical source — agents reading the URL from the invoice keep working if the host ever moves.

## Installation

```bash
npm install @x402/fetch @x402/core @x402/evm viem
# or
pnpm add @x402/fetch @x402/core @x402/evm viem
```

`@x402/fetch` wraps the standard `fetch` API: when a request returns `402 Payment Required`, the wrapper signs an EIP-3009 USDC authorization and retries
automatically.

## Wallet and funding

To consume the API the agent needs:

1. A Base-mainnet EVM private key (e.g. via `viem`'s `privateKeyToAccount`).
1. **USDC on Base mainnet** at `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
   (6 decimals). Approximately $1 covers ~1000 cheap calls.
1. **No ETH required.** Settlement uses EIP-3009 `transferWithAuthorization`,
   so the CDP facilitator submits the on-chain transaction and pays the gas.
   The agent only signs.

## The x402 payment handshake

Every paid endpoint follows the same three-step pattern. Use `@x402/fetch` and
all of this happens automatically; the description below is for understanding
and debugging.

**Step 1 — unpaid request:**

```http
GET /api/v1/agent/pools HTTP/1.1
Accept: application/json
```

**Step 2 — `402 Payment Required`.** The body is empty. The invoice lives in
the `PAYMENT-REQUIRED` response header as base64-encoded JSON:

```json
{
  "x402Version": 2,
  "error": "Payment required",
  "resource": {
    "url": "https://risk-graph-gx7v4.ondigitalocean.app/api/v1/agent/pools",
    "description": "All risk-scored pools, sortable + filterable — primary agent payload",
    "mimeType": ""
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "10000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x<recipient>",
      "maxTimeoutSeconds": 300,
      "extra": { "name": "USD Coin", "version": "2" }
    }
  ]
}
```

`resource.url` is the canonical, self-describing URL — treat it as the source
of truth for the production base URL instead of hard-coding any host. Amounts
are USDC atomic units (six decimals): `"1000"` = $0.001, `"10000"` = $0.01,
`"100000"` = $0.10, `"750000"` = $0.75.

**Step 3 — paid retry.** Sign the EIP-3009 authorization, base64-encode it,
send it in an `X-PAYMENT` request header. The server verifies, executes the
handler, settles on-chain after the response is buffered, and returns:

```http
HTTP/1.1 200 OK
PAYMENT-RESPONSE: <base64(json)>
Content-Type: application/json

{ "data": { … }, "message": "OK", "statusCode": 200 }
```

The decoded `PAYMENT-RESPONSE` carries the on-chain confirmation:

```json
{
  "success": true,
  "payer": "0x<agent-wallet>",
  "transaction": "0x<base-mainnet-tx-hash>",
  "network": "eip155:8453"
}
```

The `transaction` field is a verifiable Base mainnet transaction hash (look
it up on [basescan.org](https://basescan.org)).

### No-charge guarantee on failure

Settlement runs **after** the handler responds, and **only on 2xx**. If the
handler returns `400` (validation), `404` (node not found), or `5xx` (server
error), there is no `PAYMENT-RESPONSE` header and no on-chain transfer — the
wallet is debited only when data is actually delivered. This makes it safe
to probe with imperfect inputs.

______________________________________________________________________

## Endpoint catalogue

All endpoints are `GET`. Prices are in USDC. Paths are versioned at `/v1`;
breaking changes ship under `/v2`.

### Discovery — $0.001 each

| Path                            | Returns                                                    | When to call                            |
| ------------------------------- | ---------------------------------------------------------- | --------------------------------------- |
| `/api/v1/agent/schema`          | `{ riskTiers, nodeTypes, edgeTypes, edgeConstraints }`     | **First call** — defines the data model |
| `/api/v1/agent/stats`           | `{ nodes, edges, byNetwork, byProtocol, byTier, avgRisk }` | Query-scope sizing                      |
| `/api/v1/agent/chains`          | `{ items, total, page, hasMore }` of `Chain` nodes         | Catalogue by chain                      |
| `/api/v1/agent/protocols`       | `{ items, total, page, hasMore }` of `Protocol` nodes      | Catalogue by protocol                   |
| `/api/v1/agent/search?q=<text>` | `{ nodes }`                                                | Full-text node search by name/symbol    |

### Core lists & lookups — $0.01 each

| Path                                    | Returns                                                  | When to call               |
| --------------------------------------- | -------------------------------------------------------- | -------------------------- |
| `/api/v1/agent/assets`                  | `{ items, total, page, hasMore }` of `Asset` nodes       | Asset catalogue + risk     |
| `/api/v1/agent/pools`                   | `{ items, total, page, hasMore }` of `Pool` nodes        | **Primary agent payload**  |
| `/api/v1/agent/nodes?label=<NodeLabel>` | `{ nodes, total, page }`                                 | Generic paginated listing  |
| `/api/v1/agent/scores/:nodeId`          | `{ nodeId, labels, score, penaltyBps, tier, breakdown }` | Raw score for one node     |
| `/api/v1/agent/scorecards`              | Bare array of scoring rubrics / question sets            | How scores are constructed |

### Computed — $0.05 each

| Path                                            | Returns                         | When to call                                                           |
| ----------------------------------------------- | ------------------------------- | ---------------------------------------------------------------------- |
| `/api/v1/agent/node/:nodeId`                    | `{ node, edges, neighbors }`    | One node + full 1-hop neighborhood                                     |
| `/api/v1/agent/compare?nodeId=<id>&days=<1-90>` | Time-series snapshot comparison | Risk drift over a window (`404` if snapshots insufficient — no charge) |

### Research-grade — $0.10 each

| Path                                   | Returns                                                                                            | When to call                |
| -------------------------------------- | -------------------------------------------------------------------------------------------------- | --------------------------- |
| `/api/v1/agent/scores/:nodeId/explain` | Full recursive score breakdown — component sub-scores, composite traversal, contributing neighbors | "Why is this node risky?"   |
| `/api/v1/agent/status`                 | `{ summary, byType, incomplete, unscored }`                                                        | Scoring completion overview |

### Heavy traversal — $0.75 each

| Path                                                    | Returns                                                                | When to call                      |
| ------------------------------------------------------- | ---------------------------------------------------------------------- | --------------------------------- |
| `/api/v1/agent/dependencies?nodeId=<id>&maxHops=<1-10>` | `{ paths }` — dependency chain from a node, optionally toward an asset | Reachability / dependency mapping |
| `/api/v1/agent/subgraph?nodeId=<id>&maxHops=<1-5>`      | `{ nodes, edges }`                                                     | Deep neighborhood research        |

Optional `targetAsset=<nodeId>` on `/dependencies` constrains paths that reach
that specific asset.

______________________________________________________________________

## Response envelope and shape exceptions

Every response is wrapped:

```json
{ "data": <payload>, "message": "OK", "statusCode": 200 }
```

Always read your payload from `data`. The shape of `data` varies by endpoint:

| Endpoint family                                            | `data` shape                                            |
| ---------------------------------------------------------- | ------------------------------------------------------- |
| Sorted listings (`chains`, `protocols`, `assets`, `pools`) | `{ items, total, page, hasMore }`                       |
| `/agent/nodes`                                             | `{ nodes, total, page }` *(note: `nodes`, not `items`)* |
| `/agent/search`                                            | `{ nodes }`                                             |
| `/agent/scorecards`                                        | Bare array — not wrapped                                |
| Single-node endpoints                                      | A single object                                         |

These quirks are stable but easy to miss — when the LLM is unsure, prefer
defensive access: `data?.items ?? data?.nodes ?? data ?? []`.

`Cache-Control` is `no-cache, no-store, must-revalidate` on every paid
response. Do not rely on shared HTTP caches; cache locally if needed.

______________________________________________________________________

## Common types

```ts
type NodeId = string;                              // e.g. "pool:base:0x8eff…"
type NodeLabel = "Pool" | "Asset" | "Protocol" | "Chain";

type EdgeType =
  | "TOKENIZES"
  | "ACCEPTS_DEPOSITS_IN"
  | "IS_COLLATERALIZED_BY"
  | "BORROWS"
  | "ALLOCATES"
  | "BACKED_BY"
  | "ISSUED_BY"
  | "OPERATED_BY"
  | "USES_LENDING_VENUE"
  | "USES_TRADING_VENUE"
  | "LIVES_ON"
  | "BRIDGED_VIA"
  | "DEPENDS_ON_PROTOCOL";

type PropertyValue = string | number | boolean | null;

interface GraphNode {
  id: NodeId;
  labels: NodeLabel[];
  properties: Record<string, PropertyValue>;
}

interface GraphEdge {
  type: EdgeType;
  sourceId: NodeId;
  targetId: NodeId;
  properties?: Record<string, PropertyValue>;
}
```

### Risk-related properties on nodes

| Property           | Meaning                                                           |
| ------------------ | ----------------------------------------------------------------- |
| `_riskScore`       | Composite risk score, `0..1` (lower is riskier)                   |
| `_riskTier`        | Coarse tier letter (`A` best → `F` worst); exact set in `/schema` |
| `_riskPenaltyBps`  | Cumulative penalty in basis points                                |
| `_riskCompositeOf` | JSON string of the component nodes that contributed               |

### `NodeId` convention

Deterministic, content-addressable: `<label>:<chain-or-namespace>:<address-or-slug>`.
Examples:

- `pool:base:0x8effa741061aaa2d8a5012a9b09a2d31d8b628d7`
- `asset:ethereum:0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` (USDC)
- `protocol:morpho`
- `chain:base`

______________________________________________________________________

## Listing query params

The sorted listing endpoints (`/chains`, `/protocols`, `/assets`, `/pools`)
share this parameter set:

| Param         | Type            | Default      | Notes                                |
| ------------- | --------------- | ------------ | ------------------------------------ |
| `sortBy`      | string          | `_riskScore` | any property on the node             |
| `sortDir`     | `asc` \| `desc` | `desc`       |                                      |
| `tiers`       | csv string      | —            | e.g. `A,B,C` — filter by `_riskTier` |
| `chains`      | csv string      | —            | filter by chain slug                 |
| `protocols`   | csv string      | —            | filter by protocol slug              |
| `q`           | string          | —            | substring search on name             |
| `filterKey`   | string          | —            | property key to filter on            |
| `filterValue` | string          | —            | exact-match value for `filterKey`    |
| `page`        | int ≥ 1         | —            | omit to return all rows              |
| `limit`       | int 1–200       | —            | omit to return all rows              |

`/agent/nodes` accepts a different (smaller) set: `label`, `protocol`,
`network`, `page` (default `1`), `limit` (default `20`, max `100`).

______________________________________________________________________

## Minimal client (~10 lines, Node)

```ts
import { x402Client }            from '@x402/core/client';
import { ExactEvmScheme }        from '@x402/evm/exact/client';
import { wrapFetchWithPayment }  from '@x402/fetch';
import { privateKeyToAccount }   from 'viem/accounts';

const signer = privateKeyToAccount(process.env.EVM_PRIVATE_KEY as `0x${string}`);
const client = new x402Client();
client.register('eip155:*', new ExactEvmScheme(signer));  // wildcard covers any eip155 chain
const fetchPaid = wrapFetchWithPayment(fetch, client);

const res    = await fetchPaid('https://risk-graph-gx7v4.ondigitalocean.app/api/v1/agent/pools?limit=10');
const json   = await res.json();                           // { data: { items, … }, … }
const items  = json?.data?.items ?? [];
const txHash = res.headers.get('payment-response');        // base64 settlement receipt
```

Register `eip155:*` rather than a specific chain — the wildcard handles any
EVM network the invoice declares, so the same client works against mainnet,
testnet, or a future chain without code changes.

______________________________________________________________________

## Workflow examples

### Discover, then deep-dive on a pool

```ts
// 1. Learn the data model (one-time, $0.001)
const schema = (await (await fetchPaid(`${BASE}/api/v1/agent/schema`)).json()).data;

// 2. Find candidate pools — top 50 by TVL ($0.01)
const pools = (await (await fetchPaid(
  `${BASE}/api/v1/agent/pools?sortBy=totalAssetsUsd&sortDir=desc&limit=50`
)).json()).data.items;

// 3. Drill into one (full breakdown, $0.10)
const top = pools[0];
const explain = (await (await fetchPaid(
  `${BASE}/api/v1/agent/scores/${encodeURIComponent(top.id)}/explain`
)).json()).data;

// 4. Map the surrounding risk graph ($0.75)
const subgraph = (await (await fetchPaid(
  `${BASE}/api/v1/agent/subgraph?nodeId=${encodeURIComponent(top.id)}&maxHops=2`
)).json()).data;
```

Total spend: $0.861.

### Rank protocols by risk and compare

```ts
const protocols = (await (await fetchPaid(
  `${BASE}/api/v1/agent/protocols?sortBy=_riskScore&sortDir=asc&limit=20`
)).json()).data.items;

for (const p of protocols.slice(0, 3)) {
  const r = (await (await fetchPaid(
    `${BASE}/api/v1/agent/scores/${encodeURIComponent(p.id)}`
  )).json()).data;
  console.log(p.properties.name, r.score.toFixed(3), r.tier);
}
```

### Find what an asset depends on

```ts
const paths = (await (await fetchPaid(
  `${BASE}/api/v1/agent/dependencies?nodeId=${encodeURIComponent('pool:base:0x8eff…')}&maxHops=5&targetAsset=${encodeURIComponent('asset:base:0x8335…2913')}`
)).json()).data.paths;
```

______________________________________________________________________

## Important notes

1. **`resource.url` from the invoice is canonical.** It carries the production
   host. Don't hard-code a base URL — read it from the first 402 response and
   cache it for the session.

1. **Wildcard signer registration.** Register `eip155:*` so the client works
   whatever network the invoice declares. The wildcard scheme covers both
   Base mainnet today and any future EVM network.

1. **Fund with USDC, not ETH.** EIP-3009 settlements are gasless from the
   payer's perspective — only USDC is required in the wallet. The facilitator
   pays gas.

1. **No-charge on failure means you can probe.** Trying an unknown `nodeId`
   returns `404` with no settlement, so it's safe to call `/scores/:id` or
   `/node/:id` with uncertain input. Validation errors (`400`) and server
   errors (`5xx`) likewise skip settlement.

1. **`/compare` is the one endpoint that often `404`s legitimately.** It
   requires at least two snapshots in the requested window — for a freshly
   indexed node there may not be enough history yet. Don't treat the 404 as
   an error; treat it as "no data available for this window".

1. **Cache the schema response.** `/agent/schema` rarely changes; caching it
   for the agent session saves $0.001 on every subsequent call and avoids
   re-fetching the static data-model declaration.

1. **List responses can be large.** `/agent/pools` without a `limit` returns
   the full catalogue (sometimes >1 MB). Pass `limit` and `page` to bound
   payload size; iterate via `hasMore`.

1. **Sibling discovery surfaces.** The same content is also exposed at
   `https://risk-graph-gx7v4.ondigitalocean.app/llms.txt` (overview + catalogue) and `/llms-full.txt`
   (full reference). Both are crawlable by AI assistants and listed in the
   site's sitemap — this skill is the in-IDE equivalent for Claude Code.
