---
name: dflow-platform-fees
description: Monetize a DFlow integration by collecting a builder-defined fee on spot trades your app routes through the Trade API — a fixed percentage via `platformFeeBps`. Use when the user asks "how do I take a cut of trades?", "add a builder fee", "monetize my swap UI", "charge a platform fee", "how does platformFeeBps work?", or "where do my fees get paid?". Do NOT use to run a trade itself (use `dflow-spot-trading` — it also covers priority fees and sponsored / gasless flows).
---

# DFlow Platform Fees

Collect a fee on trades your app routes through the DFlow Trade API, paid to a **builder-controlled token account** on successful execution. This is the builder→user fee — builders monetizing their distribution.

## Prerequisites

- **DFlow docs MCP** (`https://pond.dflow.net/mcp`) — install per the [repo README](../../README.md#recommended-install-the-dflow-docs-mcp). This skill is the recipe; the MCP is the reference. Exact parameter encoding, response shapes, and the full mode matrix live there — don't guess.

## Surface

**API only.** Platform fees are `/order` parameters on the Trade API. The `dflow` CLI is for direct self-trading, not for monetizing a distribution product, so it doesn't expose platform-fee flags — don't hunt for a `--platform-fee-bps` option.

Which host depends on API-key status, same as the other trading skills: prod `https://quote-api.dflow.net` with `x-api-key`, dev `https://dev-quote-api.dflow.net`.

## Fee model — `platformFeeBps`

Flat percentage of the trade in basis points (1 bps = 0.01%). `platformFeeBps: 50` → 0.5% fee.

## Core `/order` parameters

Full param details and encoding → docs MCP, or read the pages directly: [`/spot/trading/platform-fees`](https://pond.dflow.net/spot/trading/platform-fees), [`/spot/recipes/platform-fees`](https://pond.dflow.net/spot/recipes/platform-fees).

- `platformFeeBps` — fixed fee in bps.
- `platformFeeMode` — which side pays the fee: `outputMint` (default) or `inputMint`.
- `feeAccount` — the SPL token account that receives the fee. **Must already exist** before the trade; DFlow won't create it.

## Fee accounts (ATAs)

You need **one ATA per token you collect in**. A builder collecting in USDC and SOL needs a USDC ATA and a SOL ATA, both already created, both controlled by the builder's wallet. Pass the relevant one as `feeAccount` per request — DFlow reads it, validates it matches the mode's token, and transfers on success.

## What to ASK the user (and what NOT to ask)

**Ask if missing:**

1. **Rate** — bps value for the fee.
2. **Collection token(s)** — which token(s) do you want the fee paid in, and do you already have a matching ATA owned by the builder wallet?
3. **DFlow API key.** Platform fees are an HTTP-only feature (params on the user's own `/order` call) — there's no CLI flag for them, so you're always plumbing the key into the script's HTTP client. **Ask with a clean, neutral question: *"Do you have a DFlow API key?"*** Don't presuppose where the key lives — phrasings like *"do you have it in env?"* or *"is `DFLOW_API_KEY` set?"* nudge the user toward env-var defaults they didn't ask for. Surface the choice; don't silently fall back to env or to dev. It's **one DFlow key everywhere** — same `x-api-key` unlocks Trade API + Metadata API, REST + WebSocket. Yes → prod `https://quote-api.dflow.net` + `x-api-key`. No → dev `https://dev-quote-api.dflow.net`, rate-limited. Pointer: `https://pond.dflow.net/get-started/api-key`.

**Do NOT ask about:**
- RPC, signing, slippage — orthogonal to fees; the base trading skill handles them.
- Anything about who the *user* is — platform fees are a per-request parameter, not a wallet-level setting.

## Gotchas (the docs MCP won't volunteer these)

- **Don't set `platformFeeBps` if you're not collecting.** The API factors a declared fee into slippage tolerance; if the fee isn't actually taken onchain, the slippage budget gets "spent" on nothing and user pricing worsens. Only pass a nonzero value when there's a real `feeAccount` at the other end.
- **`feeAccount` must exist before the trade.** DFlow doesn't create it for you. If it's missing, the trade fails.
- **One ATA per collected token.** USDC fee account ≠ SOL fee account. Create what you need upfront.
- **Fees only apply on successful trades.** Failed / cancelled / reverted trades → no fee charged, no transfer. Don't count failures as fee-bearing volume.

## When something doesn't fit

Defer to the docs MCP for exact parameter encoding, the code recipe at [`/spot/recipes/platform-fees`](https://pond.dflow.net/spot/recipes/platform-fees) (runnable), and the FAQ entries on slippage interaction.

## Sibling skills

- `dflow-spot-trading` — build the base spot `/order` call; layer these params on top. Also covers priority fees and sponsored / gasless flows.
