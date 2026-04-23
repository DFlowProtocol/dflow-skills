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
| [`dflow-spot-trading`](skills/dflow-spot-trading/SKILL.md)                   | Swap any pair of Solana tokens via DFlow CLI or Trading API.                           |
| [`dflow-kalshi-trading`](skills/dflow-kalshi-trading/SKILL.md)               | Buy, sell, and redeem YES/NO outcome tokens on Kalshi prediction markets.              |
| [`dflow-kalshi-market-scanner`](skills/dflow-kalshi-market-scanner/SKILL.md) | Discover and filter Kalshi events, markets, series, tags, and historical candlesticks. |
| [`dflow-kalshi-market-data`](skills/dflow-kalshi-market-data/SKILL.md)       | Real-time orderbook, trade, and live-data streams for Kalshi markets.                  |
| [`dflow-kalshi-portfolio`](skills/dflow-kalshi-portfolio/SKILL.md)           | View open positions, unrealized P&L, and reclaim rent from empty outcome accounts.     |
| [`dflow-proof-kyc`](skills/dflow-proof-kyc/SKILL.md)                         | Integrate Proof identity verification so wallets can buy on Kalshi.                    |
| [`dflow-platform-fees`](skills/dflow-platform-fees/SKILL.md)                 | Take a builder cut on swaps and PM trades (`platformFeeBps`, `platformFeeScale`).      |


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

Prefer to inspect before executing? Split the install into two steps:

```bash
curl -fsSL https://cli.dflow.net -o dflow-install.sh
less dflow-install.sh     # audit, then:
sh dflow-install.sh
```

`dflow setup` is interactive — it asks for a wallet, passphrase, and Solana RPC URL. After that, every skill that touches the CLI just works.

## Security

Honest disclosure, since the `npx skills` install flow flags the trading skills as "HIGH-RISK" (Socket correctly notes they're "purpose-aligned" and "not overtly malicious," but describes the capability surface accurately):

- **What these skills are.** Plain markdown with YAML frontmatter. They don't execute at install time, at skill-load time, or at all — they're instructions that load into an agent's context window when the agent decides they're relevant.
- **What they enable.** An AI agent reading these skills can execute real-money Solana trades on behalf of the wallet you've configured (CLI: your passphrase-protected vault; API: the wallet adapter your app wires up). Treat that capability like an API-key scope — don't grant access to an agent you wouldn't trust with a terminal session.
- **Authorization stays at the wallet, not the agent.** CLI trades sign through [Open Wallet Standard](https://openwallet.foundation/) — keys live encrypted in a local vault and never enter the agent's context. API trades sign through whichever wallet adapter the app uses (Phantom, Privy, Turnkey, etc.). The agent constructs the request; the wallet signs it.
- **The docs MCP is read-only.** `https://pond.dflow.net/mcp` serves DFlow documentation. No trade execution, no key handling. It's also optional — skills work without it (agents just guess more on field-level questions).

## Repo layout

```
skills/
  dflow-spot-trading/SKILL.md
  dflow-kalshi-trading/SKILL.md
  ... (one folder per skill, each just a SKILL.md)
```

