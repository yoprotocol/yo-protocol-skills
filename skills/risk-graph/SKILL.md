---
name: risk-graph
description: >-
  Use Risk Graph's paid agent API (`/api/v1/agent/*`) — a DeFi risk-intelligence
  graph covering pools, assets, protocols, and chains, monetised via the
  [x402](https://x402.org) micropayment standard over USDC on Base mainnet
  (chain id `8453`). Calls cost between $0.001 and $1.00 each; the payer pays no
  gas — settlement is EIP-3009 `transferWithAuthorization` via the Coinbase CDP
  facilitator. Risk Graph is an analytical explorer, not a bulk feed: the only
  risk signal exposed is an objective A–F letter grade (`_riskTier`) per entity,
  and there is no listing or full-dataset export. Use it whenever an agent needs
  to look up a specific pool/asset/protocol/chain (by address or name) and read
  its risk grade, or map a pool's direct (1-hop) on-chain dependencies, and is
  willing to pay per call. Make sure to use this skill whenever the user mentions
  Risk Graph, risk-graph, DeFi risk grade, pool risk, protocol risk, chain risk,
  asset risk, risk-tier lookup, pool dependency mapping, `/api/v1/agent/`, x402,
  paid agent API, USDC paywall, EIP-3009 paywall, Coinbase CDP facilitator, or
  building AI agents that consume on-chain risk intelligence — even if they don't
  explicitly name "Risk Graph". Triggers on mentions of Risk Graph, risk-graph
  API, DeFi risk grading, x402 paywall, USDC micropayments on Base, agent-facing
  DeFi data, pool/protocol/asset/chain risk lookup, 1-hop dependency mapping, or
  paid `/api/v1/agent/*` endpoints.
author: yoprotocol
homepage: https://github.com/yoprotocol/yo-protocol-skills
source: https://github.com/yoprotocol/yo-protocol-skills/tree/main/skills/risk-graph
---

Official Yo Protocol skill.
Canonical repository: https://github.com/yoprotocol/yo-protocol-skills

# Risk Graph Agent API — Complete Reference

Risk Graph is an **analytical explorer**: it scores DeFi entities (pools,
assets, protocols, chains) and exposes an objective **A–F letter grade** per
entity. It is deliberately **not** a bulk data feed — there is no listing,
enumeration, or full-dataset export, and the underlying scoring model is not
fetchable. You look up specific assets/protocols you already know and read their
grade.

The `/api/v1/agent/*` namespace is the **public, paid, machine-readable**
surface designed for autonomous consumption. It is monetised via the
[x402 payment standard](https://x402.org) v2 — agents pay tiny amounts of
USDC per call, no API keys, no signup.

> **Stability:** `/api/v1/agent/*` is versioned via the `/v1` segment. Breaking
> changes ship as `/v2`. Prices and response shapes are stable within a major
> version; price changes are announced before they take effect.

## Base URL

```
https://risk.yo.xyz
```

Every endpoint below resolves to `https://risk.yo.xyz/api/v1/agent/...`. The 402
challenge's `resource.url` field is the canonical source — agents reading the
URL from the invoice keep working if the host ever moves.

## Installation

```bash
npm install @x402/fetch @x402/core @x402/evm viem
# or
pnpm add @x402/fetch @x402/core @x402/evm viem
```

`@x402/fetch` wraps the standard `fetch` API: when a request returns
`402 Payment Required`, the wrapper signs an EIP-3009 USDC authorization and
retries automatically.

## Wallet and funding

To consume the API the agent needs:

1. A Base-mainnet EVM private key (e.g. via `viem`'s `privateKeyToAccount`).
1. **USDC on Base mainnet** at `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`
   (6 decimals). At the base price, ~$1 covers ~1000 cheap calls.
1. **No ETH required.** Settlement uses EIP-3009 `transferWithAuthorization`,
   so the CDP facilitator submits the on-chain transaction and pays the gas.
   The agent only signs.

## The x402 payment handshake — v2

Every paid endpoint follows the same pattern. Use `@x402/fetch` and all of this
happens automatically; the description below is for understanding and debugging.

### Network and asset

| Field | Value |
| --- | --- |
| Protocol version | `2` |
| Scheme | `exact` |
| Network | `eip155:8453` (Base mainnet) |
| Asset | USDC at `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` |
| Asset decimals | `6` |
| Settlement | EIP-3009 `transferWithAuthorization` via Coinbase CDP facilitator — the **payer pays no gas**, only USDC |

Amounts in the invoice are USDC atomic units (six decimals): `"1000"` = $0.001,
`"50000"` = $0.05, `"100000"` = $0.10, `"1000000"` = $1.00.

**Step 1 — unpaid request:**

```http
GET /api/v1/agent/node/pool:base:0x8eff…  HTTP/1.1
Host: risk.yo.xyz
Accept: application/json
```

**Step 2 — `402 Payment Required`.** The body is empty. The invoice lives in
the `PAYMENT-REQUIRED` response header as base64-encoded JSON:

```json
{
  "x402Version": 2,
  "error": "Payment required",
  "resource": {
    "url": "https://risk.yo.xyz/api/v1/agent/node/pool:base:0x8eff…",
    "description": "Single node — properties + risk grade only (no neighborhood)",
    "mimeType": ""
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:8453",
      "amount": "50000",
      "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "payTo": "0x<recipient>",
      "maxTimeoutSeconds": 300,
      "extra": { "name": "USD Coin", "version": "2" }
    }
  ]
}
```

`resource.url` is the canonical, self-describing URL — treat it as the source of
truth for the production base URL instead of hard-coding any host.

> **The invoice amount can vary by payer.** The price is resolved per request
> (see [Per-payer economics](#per-payer-economics)): a brand-new payer's first
> call carries a one-time onboarding fee, and heavy cumulative usage escalates
> the price. Always sign the amount in the invoice you receive; if the amount
> changed since a prior call, the server re-issues a `402` with the correct
> amount (one extra round-trip).

**Step 3 — paid retry.** Sign the EIP-3009 authorization, base64-encode it, and
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

The `transaction` field is a verifiable Base mainnet transaction hash (look it
up on [basescan.org](https://basescan.org)).

### No-charge guarantee on failure

The paywall settles **after** the handler responds, and **only on `2xx`**. Any
rejection — `400` (bad query), `403` (provenance), `404` (unknown id or removed
endpoint), `429` (rate limit), `5xx` — returns **no `PAYMENT-RESPONSE` header
and no on-chain transfer**. So the point lookups are safe to probe: a `/node` or
`/dependencies` miss returns `404` and is free.

`/search` is the one exception — it **always settles, including a zero-result
query** (`200` with an empty `nodes` array), because the targeted scan is the
billable work. Spend the $0.05+ point lookups only on ids you already hold.

______________________________________________________________________

## The only risk signal: letter grade

The single risk signal exposed on this surface is the letter grade
**`_riskTier`** (`A` safest → `F` riskiest), present on every node's
`properties`. Numeric scores, penalties, composites, the scoring rubric, and the
grade score-thresholds are **intentionally not provided** anywhere — in any
payload, error, or the schema. Do not present or infer a numeric risk score;
quote the grade.

```ts
type NodeId = string;                              // e.g. "pool:base:0x8eff…"
type NodeLabel = "Pool" | "Asset" | "Protocol" | "Chain";
type PropertyValue = string | number | boolean | null;

interface GraphNode {
  id: NodeId;
  labels: NodeLabel[];
  // allow-listed public fields per label + `_riskTier`; no numeric `_risk*`,
  // no rubric inputs (`derived*`, `_override_*`, peg/backing), no internal fields
  properties: Record<string, PropertyValue>;
}

interface GraphEdge {
  type: string;                                    // e.g. "ACCEPTS_DEPOSITS_IN", "BACKED_BY"
  sourceId: NodeId;
  targetId: NodeId;
  properties: Record<string, PropertyValue>;
}
```

`NodeId` follows a deterministic, content-addressable convention:
`<label>:<chain-or-namespace>:<address-or-slug>`. Examples:

- `pool:base:0x8effa741061aaa2d8a5012a9b09a2d31d8b628d7`
- `asset:ethereum:0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48` (USDC)
- `protocol:morpho`
- `chain:base`

______________________________________________________________________

## Response envelope

Every response is wrapped:

```json
{ "data": <payload>, "message": "OK", "statusCode": 200 }
```

Always read your payload from `data`. Paid responses are marked `Cache-Control:
private` so shared caches can't replay them — cache locally if you need to.

______________________________________________________________________

## Endpoint reference

All endpoints are `GET`. **Four paid endpoints; everything else is removed**
(removed paths answer `404` and, being unpriced, never `402` or charge).

### `GET /api/v1/agent/schema` — $0.001

The data-model declaration — node labels, their queryable properties, edge
types, and the grade letters. **Call this first.** Numeric score thresholds are
not included.

### `GET /api/v1/agent/search` — $0.001 — the sole discovery path

Targeted lookup. You must pass a concrete **asset address** or **protocol/asset
name**; empty and wildcard queries are rejected (`400`). Results are
**hard-capped at 10**. There is no way to list or page the full dataset.

| Param | Type | Required | Notes |
| --- | --- | --- | --- |
| `q` | string | yes | asset address or protocol/asset name; no `*`/`%` wildcards |
| `label` | `NodeLabel` | no | restrict to one label (`Pool`, `Asset`, `Protocol`, `Chain`) |
| `limit` | int 1–10 | no | default `10`, capped at `10` |
| `tvl` | string | no | comparator filter, e.g. `>1000000` or `<5e8` |
| `grade` | string | no | grade comparator over A–F, e.g. `A`, `<=B`, `>=C` |

```ts
data: { nodes: GraphNode[] }   // identity + _riskTier + display props; no edges, no scores
```

There is no sort-by-score or filter-by-score — the numeric model is never
queryable.

### `GET /api/v1/agent/node/:nodeId` — $0.05

A single node's properties + `_riskTier`. **No edges or neighbors** —
relationship data is available only via `/dependencies`. Returns `404` (no
charge) if the node isn't indexed.

```ts
data: GraphNode
```

### `GET /api/v1/agent/dependencies` — $1.00 — the sole topology path

The direct (**1-hop**) dependencies of a **Pool** — its immediate neighbors and
the edge type to each. Pools only: a non-pool target returns `400` (no charge).
There is no multi-hop traversal and no subgraph export.

| Param | Type | Required | Notes |
| --- | --- | --- | --- |
| `nodeId` | NodeId | yes | a Pool node id |

```ts
data: { edges: GraphEdge[]; neighbors: GraphNode[] }
```

> **Provenance:** `/dependencies` is intended for pools you surfaced via your own
> prior `/search`. See [Per-payer economics](#per-payer-economics).

______________________________________________________________________

## Per-payer economics

Your identity is the **payer EVM address** recovered from the signed payment.
Pricing and limits are keyed to it.

- **One-time onboarding fee.** A payer's first ever settled call carries a
  one-time, non-refundable onboarding fee (no held balance / no custody).
  Subsequent calls price normally.
- **Escalating price.** Sustained high volume from a single payer gets
  progressively more expensive — ×5 per order of magnitude of cumulative settled
  calls beyond the first ~100 (capped). Individual lookups stay cheap; bulk
  extraction does not.
- **How pricing surfaces.** Pricing is keyed to the payer address, which only
  appears once the payment is signed — so the *first* (header-less) `402` always
  quotes the base price. When your true price (onboarding fee and/or escalation)
  is higher, the server re-issues a `402` with the corrected amount before
  settling (one extra round-trip). A brand-new wallet's first `402` therefore
  shows the base price, **not** the onboarding fee; always sign the amount in the
  invoice you receive.
- **Rate limit.** ~10 requests/second per payer. Over the limit returns `429`
  with a `Retry-After` header (no charge).
- **Provenance.** `/dependencies` is intended for pools you surfaced through your
  own `/search`. Requesting topology for an id you never searched may be rejected
  with `403` (no charge).

______________________________________________________________________

## Errors

| Status | Meaning | Charged? |
| --- | --- | --- |
| `200` | success — `PAYMENT-RESPONSE` header present | yes |
| `400` | invalid query (empty/wildcard search, non-pool dependencies) | no |
| `402` | unpaid — `PAYMENT-REQUIRED` header present | no |
| `403` | provenance — dependencies on an id you didn't surface | no |
| `404` | point-lookup miss (`/node`, `/dependencies`) or removed endpoint | no |
| `429` | rate limit — `Retry-After` present | no |
| `5xx` | server error | no |

> A zero-result `/search` is **not** a `404` — it returns `200` with an empty
> `nodes` array and settles normally; the targeted scan itself is the billable
> work. Only point lookups (`/node`, `/dependencies`) `404` on a miss.

Errors come back wrapped in the standard envelope:

```json
{ "data": null, "message": "…", "statusCode": 404 }
```

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

const res     = await fetchPaid('https://risk.yo.xyz/api/v1/agent/search?q=morpho');
const json    = await res.json();                          // { data: { nodes }, … }
const nodes   = json?.data?.nodes ?? [];
const receipt = res.headers.get('payment-response');       // base64 settlement receipt
```

Register `eip155:*` rather than a specific chain — the wildcard handles any EVM
network the invoice declares, so the same client works against mainnet, testnet,
or a future chain without code changes. Fund the signer with a small balance of
USDC on Base mainnet (no ETH required — the facilitator pays gas).

______________________________________________________________________

## Workflow example — discover, then deep-dive on a pool

```ts
const BASE = 'https://risk.yo.xyz';

// 1. Learn the data model (one-time, $0.001)
const schema = (await (await fetchPaid(`${BASE}/api/v1/agent/schema`)).json()).data;

// 2. Find the entity you care about — targeted, capped at 10 ($0.001)
const found = (await (await fetchPaid(
  `${BASE}/api/v1/agent/search?q=morpho&label=Pool&grade=<=B`
)).json()).data.nodes;

// 3. Read one node's letter grade ($0.05) — properties + _riskTier only
const pool = found[0];
const node = (await (await fetchPaid(
  `${BASE}/api/v1/agent/node/${encodeURIComponent(pool.id)}`
)).json()).data;
console.log(node.properties.name, node.properties._riskTier);

// 4. Map that pool's direct (1-hop) dependencies ($1.00) — pools only
const deps = (await (await fetchPaid(
  `${BASE}/api/v1/agent/dependencies?nodeId=${encodeURIComponent(pool.id)}`
)).json()).data;   // { edges, neighbors }
```

______________________________________________________________________

## Update cadence and data freshness

Risk grades recompute continuously as the underlying on-chain data, oracle
prices, and protocol configurations change. When citing a grade in a downstream
artifact (report, tool call), quote both the grade (A–F) and the access date.

______________________________________________________________________

## Important notes

1. **`resource.url` from the invoice is canonical.** It carries the production
   host — don't hard-code a base URL; read it from the first 402 response and
   cache it for the session.

1. **Wildcard signer registration.** Register `eip155:*` so the client works
   whatever network the invoice declares — Base mainnet today, any future EVM
   network without code changes.

1. **Fund with USDC, not ETH.** EIP-3009 settlements are gasless from the
   payer's perspective; only USDC is required. The facilitator pays gas.

1. **No-charge on failure means you can probe — except `/search`.** An unknown
   `nodeId` on `/node` or `/dependencies` returns `404` with no settlement, so
   it's safe to call with uncertain input. Validation errors (`400`), provenance
   (`403`), rate limits (`429`), and server errors (`5xx`) likewise skip
   settlement. `/search` is the exception: it settles even on zero results.

1. **`/search` is the only discovery path.** There is no listing, enumeration,
   sort-by-score, or full-dataset export. Pass a concrete asset address or
   protocol/asset name; empty and wildcard queries are rejected, and results are
   capped at 10.

1. **`/dependencies` is 1-hop and pools-only.** It returns a pool's immediate
   neighbors and edge types — no multi-hop traversal, no subgraph. Use ids you
   surfaced via your own `/search` (provenance).

1. **Quote the letter grade, never a number.** `_riskTier` (A–F) is the only
   risk value exposed; numeric scores, penalties, and the rubric are never
   returned.

1. **Cache the schema response.** `/agent/schema` rarely changes; caching it for
   the agent session saves $0.001 on every subsequent call.

1. **Sibling discovery surfaces.** `https://risk.yo.xyz` serves only the paid
   `/api/v1/agent/*` API — it has no web pages. The same content in
   human/crawler form lives on the website at
   `https://yo.xyz/risk-graph/llms.txt` (overview + catalogue) and
   `https://yo.xyz/risk-graph/llms-full.txt` (full reference). Both are crawlable
   by AI assistants and listed in the site's sitemap — this skill is the in-IDE
   equivalent for Claude Code.

______________________________________________________________________

## Out of scope

Risk Graph is an analytical explorer. It does not provide financial advice,
custodial services, or trade execution. It exposes letter grades only, and
provides no bulk, listing, or full-dataset access.
