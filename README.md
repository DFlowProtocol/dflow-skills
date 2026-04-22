# DFlow Skills

A collection of [Claude Code Skills](https://docs.claude.com/en/docs/claude-code/skills) for working with [DFlow](https://dflow.net) — Solana spot trading, Kalshi prediction markets, Proof KYC, and adjacent monetization / fee / sponsorship surfaces.

Each skill is a focused recipe — a single `SKILL.md` that captures the workflow, decisions, and gotchas an agent needs to use DFlow well. The skills are deliberately light: for endpoint shapes, parameter details, and error codes they defer to the **DFlow docs MCP**, and for runnable code examples they point at the **DFlow docs recipes** at `https://pond.dflow.net/build/recipes` (each recipe page links to the DFlow Cookbook Repo for clone-and-go usage).

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

## Installing these skills

### Recommended: `npx skills`

The [`skills` CLI](https://github.com/vercel-labs/skills) (from Vercel Labs) auto-detects your coding agent — Claude Code, Cursor, Codex, OpenCode, and [40+ more](https://skills.sh) — installs every `SKILL.md` into the right place for each, and defaults to symlinks so `git pull` / `npx skills update` picks up upstream changes.

```bash
# Install all DFlow skills into every agent the CLI detects on your machine
npx skills add DFlowProtocol/dflow-skills

# Install globally (to ~/<agent>/skills/) instead of the current project
npx skills add DFlowProtocol/dflow-skills -g

# Install only one skill
npx skills add DFlowProtocol/dflow-skills -s dflow-kalshi-trading

# Install to a specific agent only
npx skills add DFlowProtocol/dflow-skills -a claude-code
npx skills add DFlowProtocol/dflow-skills -a cursor

# List available skills without installing
npx skills add DFlowProtocol/dflow-skills --list

# Update later
npx skills update
```

### Manual install (alternative)

If you'd rather not run `npx skills`:

**Claude Code / Cursor / Codex / any agent that reads a `skills/` directory:**

```bash
git clone https://github.com/DFlowProtocol/dflow-skills.git
mkdir -p ~/.claude/skills                          # or ~/.cursor/skills, ~/.codex/skills, etc.
ln -s "$(pwd)"/dflow-skills/skills/* ~/.claude/skills/
```

For project-scoped installs, target `.claude/skills/` (or the equivalent under your project root — see the [vercel-labs/skills agent table](https://github.com/vercel-labs/skills#supported-agents) for per-agent paths) instead of the `~/.<agent>/skills/` path.

Swap a single skill by pointing `ln -s` at that folder: `ln -s "$(pwd)/dflow-skills/skills/dflow-kalshi-trading" ~/.claude/skills/`. Prefer `cp -r`? Substitute it for `ln -s` — you'll just need to re-run after `git pull` to pick up updates.

**Claude.ai (web):**

1. Clone or download this repo.
2. Zip the `skills/<skill-name>/` folder you want (one zip per skill).
3. Claude.ai → **Settings → Capabilities → Skills → Upload skill**.

## Repo layout

```
skills/
  dflow-spot-trading/SKILL.md
  dflow-kalshi-trading/SKILL.md
  ... (one folder per skill, each just a SKILL.md)
```

