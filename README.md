# DFlow Skills

A collection of [Claude Code Skills](https://docs.claude.com/en/docs/claude-code/skills) for working with [DFlow](https://dflow.net) — Solana spot trading, Kalshi prediction markets, Proof KYC, and adjacent monetization / fee / sponsorship surfaces.

Each skill is a focused recipe, a single `SKILL.md` that captures the workflow, decisions, and gotchas an agent needs to use DFlow well. The skills are deliberately light: for endpoint shapes, parameter details, and error codes they defer to the **DFlow docs MCP**, and for runnable code examples they point at the **DFlow docs recipes** at `https://pond.dflow.net/build/recipes` (each recipe page links to the DFlow Cookbook Repo for clone-and-go usage).

## Install

```bash
npx skills add DFlowProtocol/dflow-skills
```

The `skills` [CLI](https://github.com/vercel-labs/skills) is interactive, it detects your agents, lets you pick which skills and which agents to install to, and asks whether you want project or global scope.

## Skills


| Skill                                                               | What it does                                                                           |
| ------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| [`dflow-spot-trading`](skills/dflow-spot-trading)                   | Swap any pair of Solana tokens via DFlow CLI or Trading API.                           |
| [`dflow-kalshi-trading`](skills/dflow-kalshi-trading)               | Buy, sell, and redeem YES/NO outcome tokens on Kalshi prediction markets.              |
| [`dflow-kalshi-market-scanner`](skills/dflow-kalshi-market-scanner) | Discover and filter Kalshi events, markets, series, tags, and historical candlesticks. |
| [`dflow-kalshi-market-data`](skills/dflow-kalshi-market-data)       | Real-time orderbook, trade, and live-data streams for Kalshi markets.                  |
| [`dflow-kalshi-portfolio`](skills/dflow-kalshi-portfolio)           | View open positions, unrealized P&L, and reclaim rent from empty outcome accounts.     |
| [`dflow-proof-kyc`](skills/dflow-proof-kyc)                         | Integrate Proof identity verification so wallets can buy on Kalshi.                    |
| [`dflow-platform-fees`](skills/dflow-platform-fees)                 | Take a builder cut on swaps and PM trades (`platformFeeBps`, `platformFeeScale`).      |


## Recommended: install the DFlow docs MCP

Every skill tells the agent to query the DFlow docs MCP for anything reference-y. The skills are deliberately the *recipe* (workflow ordering, gates, defaults, gotchas); the MCP is the *reference* (every parameter, every endpoint, every error code). The skills work without it but the agent will guess on field-level questions; with it, the agent can look things up canonically.

Server URL: `https://pond.dflow.net/mcp`

### Cursor

Add to `.cursor/mcp.json` (workspace) or `~/.cursor/mcp.json` (global):

```json
{
  "mcpServers": {
    "DFlow": {
      "url": "https://pond.dflow.net/mcp"
    }
  }
}
```

### Claude Code CLI

```bash
claude mcp add --transport http DFlow https://pond.dflow.net/mcp
```

### Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "DFlow": {
      "url": "https://pond.dflow.net/mcp"
    }
  }
}
```

### Claude.ai (web)

Add it as a Connector under Settings → Connectors using the URL above.

## Recommended: install the `dflow` CLI

Skills that cover the CLI surface (spot trading, Kalshi trading, portfolio, etc.) assume `dflow` is available on `PATH`. If it isn't, the skill will tell the agent to install it:

```bash
curl -fsS https://cli.dflow.net | sh
dflow setup
```

`dflow setup` is interactive — it asks for a wallet, passphrase, and Solana RPC URL. After that, every skill that touches the CLI just works.

## Repo layout

```
skills/
  dflow-spot-trading/SKILL.md
  dflow-kalshi-trading/SKILL.md
  ... (one folder per skill, each just a SKILL.md)
```

