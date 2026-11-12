# DFlow Skills

Agent skills for building on [DFlow](https://dflow.net), the trading infrastructure for Solana. Each skill is a `SKILL.md` that gives a coding agent the journey, the decisions to raise with the developer, and a working example. Parameters, response fields, and error codes stay in the [DFlow docs](https://pond.dflow.net), which agents read through the docs MCP.

## Skills

| Skill | Use when |
| --- | --- |
| [`dflow-crypto-trading`](skills/dflow-crypto-trading/SKILL.md) | Swapping one SPL token for another through the Trading API: `/order` requests, signing, sending, and confirming. |
| [`dflow-equities-trading`](skills/dflow-equities-trading/SKILL.md) | Buying or selling tokenized equities: mint lookup, sync and async orders, the submit endpoint, and tracking async orders. |
| [`dflow-crypto-market-data`](skills/dflow-crypto-market-data/SKILL.md) | Building live price displays or order book depth from the quote and book WebSocket streams. |
| [`dflow-sponsored-swaps`](skills/dflow-sponsored-swaps/SKILL.md) | Building gasless swaps where a sponsor wallet pays transaction fees and token account creation costs. |

## Install

```bash
npx skills add DFlowProtocol/dflow-skills
```

The `skills` [CLI](https://github.com/vercel-labs/skills) detects your agents and lets you pick which skills and agents to install to. Skills install to the current project by default. Add `-g` to install them globally.

Update installed skills with `npx skills update`, and list them with `npx skills list`.

## Install the DFlow docs MCP

The skills point agents to the DFlow docs MCP for reference material. Connect it so an agent can look up exact parameters and error codes.

```
https://pond.dflow.net/mcp
```

For Claude Code:

```bash
claude mcp add --transport http DFlow https://pond.dflow.net/mcp
```

[MCP Server](https://pond.dflow.net/ai/mcp) has setup steps for Claude, Cursor, VS Code, and Windsurf. Without the MCP, agents start from [llms.txt](https://pond.dflow.net/llms.txt).

## Conventions

- Code examples use `@solana/kit` 8.0.0 or later and request v1 transactions.
- Examples default to the developer endpoints. Both developer and production endpoints run on mainnet with the same features. Developer endpoints need no API key and have lower rate limits.
- Each skill lists the questions an agent should raise before it writes code.
