# yo-protocol-skills

Claude skills for [Yo Protocol](https://yo.xyz) — ERC-4626 yield vaults on Ethereum, Base, and Arbitrum.

## Skills

### [`yo-protocol-sdk`](./skills/yo-protocol-sdk/SKILL.md)

Build applications with `@yo-protocol/core`. Covers depositing, redeeming, checking positions, prepared transactions for Safe/AA wallets, querying vault snapshots/yield/TVL, and claiming Merkl rewards.

### [`yo-protocol-cli`](./skills/yo-protocol-cli/SKILL.md)

Use the `@yo-protocol/cli` agent-first transaction builder. Covers `yo info`, `yo read`, `yo prepare`, `yo api`, and `yo schema` — outputs structured JSON, never requires private keys.

## Usage

These skills are auto-indexed by [SkillsMP](https://skillsmp.com). To use them locally, copy the relevant `SKILL.md` into your `.claude/skills/` directory.

## Vaults

| Vault  | Underlying | Chains                   |
| ------ | ---------- | ------------------------ |
| yoETH  | WETH       | Ethereum, Base           |
| yoBTC  | cbBTC      | Ethereum, Base           |
| yoUSD  | USDC       | Ethereum, Base, Arbitrum |
| yoEUR  | EURC       | Ethereum, Base           |
| yoGOLD | XAUt       | Ethereum                 |
| yoUSDT | USDT       | Ethereum                 |

## Links

- [Yo Protocol](https://yo.xyz)
- [npm: @yo-protocol/core](https://www.npmjs.com/package/@yo-protocol/core)
- [X / Twitter](https://x.com/yield)
