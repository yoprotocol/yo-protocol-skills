# Agent Skills

Claude Code skills for [YO Protocol](https://yo.xyz).

## Skills

| Skill                                                      | Description                                                                                                                       |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [`yo-protocol-sdk`](skills/yo-protocol-sdk/SKILL.md)       | Build applications with `@yo-protocol/core` — deposits, redeems, positions, prepared transactions, vault snapshots, Merkl rewards |
| [`yo-protocol-react`](skills/yo-protocol-react/SKILL.md)   | React hooks and components for `@yo-protocol/react` — YieldProvider, query/action hooks, migration from yo-kit                    |
| [`yo-protocol-cli`](skills/yo-protocol-cli/SKILL.md)       | Agent-first CLI (`@yo-protocol/cli`) — `yo prepare`, `yo position`, `yo rewards`, `yo schema`, JSON output, no private keys       |
| [`yo-base-mcp-plugin`](skills/yo-base-mcp-plugin/SKILL.md) | Base MCP plugin for YO Protocol — drives the YO CLI / HTTP API and routes unsigned Gateway calldata through `send_calls`          |
| [`yo-design`](skills/yo-design/SKILL.md)                   | Production-grade React + Tailwind v4 interfaces in YO's dark-theme aesthetic                                                      |
| [`risk-graph`](skills/risk-graph/SKILL.md)                 | Consume Risk Graph's paid `/api/v1/agent/*` API — DeFi risk-intelligence as a graph, monetised via x402 over USDC on Base         |

## Installation

Install individual skills via [skills.sh](https://skills.sh):

```bash
npx skills add yoprotocol/yo-protocol-skills --skill yo-protocol-sdk
npx skills add yoprotocol/yo-protocol-skills --skill yo-protocol-react
npx skills add yoprotocol/yo-protocol-skills --skill yo-protocol-cli
npx skills add yoprotocol/yo-protocol-skills --skill yo-base-mcp-plugin
npx skills add yoprotocol/yo-protocol-skills --skill yo-design
npx skills add yoprotocol/yo-protocol-skills --skill risk-graph
```

Or install all skills at once:

```bash
npx skills add yoprotocol/yo-protocol-skills --all
```

## Development

| Command     | Description                                         |
| ----------- | --------------------------------------------------- |
| `just sync` | Commit, install skills to `~/.agents`, commit there |
| `just mw`   | Format markdown files                               |
| `just mc`   | Check markdown formatting                           |

## License

[MIT](LICENSE)
