# DFlow Skills

A collection of [Claude Code Skills](https://docs.claude.com/en/docs/claude-code/skills) for working with [DFlow](https://dflow.net) — Solana spot trading, Kalshi prediction markets, Proof KYC, and adjacent monetization / fee / sponsorship surfaces.

Each skill is a focused recipe — a single `SKILL.md` that captures the workflow, decisions, and gotchas an agent needs to use DFlow well. The skills are deliberately light: for endpoint shapes, parameter details, and error codes they defer to the **DFlow docs MCP**, and for runnable code examples they point at the **DFlow docs recipes** at `https://pond.dflow.net/build/recipes` (each recipe page links to the DFlow Cookbook Repo for clone-and-go usage).

## Skills

| Skill | What it does |
| ----- | ------------ |
| [`dflow-spot-trading`](skills/dflow-spot-trading) | Swap any pair of Solana tokens via DFlow CLI or Trading API. |
| [`dflow-kalshi-trading`](skills/dflow-kalshi-trading) | Buy, sell, and redeem YES/NO outcome tokens on Kalshi prediction markets. |
| [`dflow-kalshi-market-scanner`](skills/dflow-kalshi-market-scanner) | Discover and filter Kalshi events, markets, series, tags, and historical candlesticks. |
| [`dflow-kalshi-market-data`](skills/dflow-kalshi-market-data) | Real-time orderbook, trade, and live-data streams for Kalshi markets. |
| [`dflow-kalshi-portfolio`](skills/dflow-kalshi-portfolio) | View open positions, unrealized P&L, and reclaim rent from empty outcome accounts. |
| [`dflow-proof-kyc`](skills/dflow-proof-kyc) | Integrate Proof identity verification so wallets can buy on Kalshi. |
| [`dflow-platform-fees`](skills/dflow-platform-fees) | Take a builder cut on swaps and PM trades (`platformFeeBps`, `platformFeeScale`). |

## Recommended: install the DFlow docs MCP

Every skill tells the agent to query the DFlow docs MCP (`search_d_flow`, `query_docs_filesystem_d_flow`) for anything reference-y. The skills are deliberately the *recipe* (workflow ordering, gates, defaults, gotchas); the MCP is the *reference* (every parameter, every endpoint, every error code). The skills work without it but the agent will guess on field-level questions; with it, the agent can look things up canonically.

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

`dflow setup` is interactive — it asks for a wallet, passphrase, and Solana RPC URL ([Helius](https://helius.dev) recommended for production). After that, every skill that touches the CLI just works.

## Installing these skills

Pick the pattern that matches your host. We recommend **symlinking** (not copying) — that way `git pull` on the repo updates every installed skill in place.

### Claude Code (personal) — all skills

```bash
git clone https://github.com/DFlowProtocol/dflow-skills.git
mkdir -p ~/.claude/skills
ln -s "$(pwd)"/dflow-skills/skills/* ~/.claude/skills/
```

### Claude Code (personal) — one skill

```bash
git clone https://github.com/DFlowProtocol/dflow-skills.git
mkdir -p ~/.claude/skills
ln -s "$(pwd)/dflow-skills/skills/dflow-kalshi-trading" ~/.claude/skills/
```

Swap `dflow-kalshi-trading` for any skill folder under `skills/`.

### Claude Code (project-scoped)

Same commands, but target `.claude/skills/` in your project root instead of `~/.claude/skills/`. Commits alongside your project; only that project sees the skills.

### Cursor

Same commands, target `~/.cursor/skills/` (personal) or `.cursor/skills/` (workspace-scoped).

### Claude.ai (web)

1. Clone or download this repo.
2. Zip the `skills/<skill-name>/` folder you want (one zip per skill).
3. Claude.ai → **Settings → Capabilities → Skills → Upload skill**.

### Prefer `cp -r` over `ln -s`?

Fine — substitute `cp -r` for `ln -s` in any of the commands above. You just won't auto-pick-up updates; you'll need to re-copy after `git pull`.

## Repo layout

```
skills/
  dflow-spot-trading/SKILL.md
  dflow-kalshi-trading/SKILL.md
  ... (one folder per skill, each just a SKILL.md)
```
