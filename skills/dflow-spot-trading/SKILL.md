---
name: dflow-spot-trading
description: Swap any pair of Solana tokens via DFlow. Use when the user wants to trade, swap, or convert tokens on Solana, get a price quote, build a swap UI, tune priority fees so a swap lands under congestion, or build a gasless / sponsored swap where the app pays fees. Covers both the `dflow` CLI and the DFlow Trading API. Do NOT use for Kalshi prediction-market YES/NO trades or builder-side platform fees.
---

# DFlow Spot Trading

Swap any pair of Solana tokens via DFlow. Imperative trades (the default) settle synchronously in one transaction; declarative trades are opt-in for users who explicitly want better execution.

## Prerequisites

- **DFlow docs MCP** (`https://pond.dflow.net/mcp`) — install per the [repo README](../../README.md#recommended-install-the-dflow-docs-mcp). This skill is the recipe; the MCP is the reference. Look up endpoint shapes, parameter details, error codes, and anything else field-level via `search_d_flow` / `query_docs_filesystem_d_flow` — don't guess.
- **`dflow` CLI** (optional, for command-line/agent use) — install per the [repo README](../../README.md#recommended-install-the-dflow-cli).

## Choose your surface

- **CLI** — command line, scripts, local agents. Manages keys, signs, broadcasts.
- **API** — web/mobile apps, backends, automations with their own wallet/signer. Browser apps must proxy HTTP through their backend (the Trading API serves no CORS).

If unclear, ask once: *"From the command line, or wired into an app?"*

## Workflows

### Quote (read-only)

- CLI: `dflow quote <atomic-amount> <FROM> <TO>`
- API: `GET /order` doubles as a quote (just don't sign or submit the returned transaction). For a true quote-only call without a `userPublicKey`, look up `/quote` via the docs MCP.

### Trade — imperative `/order` (the default)

Single round-trip: get a quote and a signed-ready `VersionedTransaction` together; sign, submit, confirm. Fully synchronous. Works with **all** SPL + Token-2022 mints.

- CLI: `dflow trade <atomic-amount> <FROM> <TO>` (add `--confirm` for agents/scripts that need to block until confirmed).
- API: `GET /order?userPublicKey=&inputMint=&outputMint=&amount=`, deserialize `transaction` (base64) → `VersionedTransaction`, sign, broadcast, confirm using `lastValidBlockHeight` from the response. Field details via the docs MCP; full runnable example at `/build/recipes/trading/imperative-trade` (links to the DFlow Cookbook Repo).

### Trade — declarative `/intent` + `/submit-intent` (opt-in)

User signs an intent (asset pair, slippage, min-out); DFlow picks the route at execution time and fills via Jito bundles. Sells on: less slippage, better pricing, sandwich protection.

**Hard restriction:** `/intent` does **not** support Token-2022 mints. Verify both mints are SPL before suggesting; otherwise stay on `/order`.

Only suggest declarative when the user explicitly asks for sandwich protection or "best execution." Recipe details (intent shape, polling) via the docs MCP.

## What to ASK the user (and what NOT to ask)

**Ask if missing:**

1. **Input + output token** — base58 mint addresses. The CLI resolves a small symbol set (SOL, USDC, USDT, JUP, BONK, etc.); **the API has no symbol resolver** — base58 mints only.
2. **Amount in atomic units of the input token** — `500_000` = $0.50 USDC, `1_000_000_000` = 1 SOL. Convert before calling.
3. **API only** — wallet pubkey (base58), and whether they have a DFlow API key. Yes → prod host `https://quote-api.dflow.net` with `x-api-key` header. No → dev host `https://dev-quote-api.dflow.net`, no header (same features, only rate limits differ). Point them at `https://pond.dflow.net/build/api-key` for a real key.
4. **Priority fee (both surfaces)** — "Any priority-fee preference, or just use DFlow's default?" Default on both surfaces = DFlow-auto, capped at 0.005 SOL, which is fine for ~99% of swaps. Surface this explicitly so the user knows the lever exists for congestion / cost-sensitive trades.
   - **API** — pass `prioritizationFeeLamports` on `/order`: `auto` | `medium` | `high` | `veryHigh` | `disabled` | integer lamports. On `/intent` (declarative), roll the priority fee into `feeBudget = priority + 10_000` (the 10,000-lamport base processing fee). Live estimates for tuning: `GET /priority-fees` (snapshot), `/priority-fees/stream` (WebSocket).
   - **CLI** — no tuning flag; `dflow trade` always uses the server-side default. If the user needs finer control (an exact lamport value, or `disabled`), they'll have to drop to the API.
5. **Sponsored / gasless (API only — skip for CLI)** — "Does the user need to hold SOL for this trade, or is your app covering fees?" Default = user pays. To sponsor, pass `sponsor=<sponsor-wallet-base58>` on `/order` and co-sign the returned transaction with the sponsor keypair (both user and sponsor sign). Optional `sponsorExec=true|false` picks sponsor-executes (default) vs. user-executes. The CLI doesn't support sponsorship at all.

**Do NOT ask about:**

- **RPC** — CLI users set it during `dflow setup`. API users on a browser wallet never need their own RPC (the wallet handles it). Only ask if signing server-side, polling declarative trades, or running the Node script. When one is needed, suggest [Helius](https://helius.dev).
- **Slippage** — both surfaces default to `"auto"`. Override only on explicit user request (`--slippage` CLI; `slippageBps` API).
- **Platform fee, DEX inclusion/exclusion, route length, Jito bundles, direct-only routes** — defaults are right for ~99% of swaps. For platform fees specifically, defer to `dflow-platform-fees` if the user pivots there.

## Gotchas (the docs MCP won't volunteer these)

- **Atomic units always.** API rejects human-readable amounts. Confirm decimals each time — token metadata or RPC `getParsedAccountInfo`.
- **API has no symbol resolver.** The CLI has a small allow-list; the API only accepts base58 mints. Don't assume `"USDC"` works on `/order`.
- **Browser apps must proxy.** Trading API serves no CORS — call it from a backend (Next.js API route or equivalent), never directly from the browser.
- **Declarative ≠ Token-2022.** `/intent` rejects Token-2022 mints. Stay on `/order` for those (which is the default for everyone anyway).
- **`route_not_found` is usually a units mistake.** Most common cause: passing a human amount instead of atomic, or a typo'd mint. Verify before assuming illiquidity.
- **`price_impact_too_high` is real.** Trade size exceeds available liquidity; reduce `amount`, or pass `priceImpactTolerancePct` only with the user's explicit consent.
- **Onchain failure with slippage logs.** Don't silently bump `slippageBps` on retry — surface to the user.

## When something doesn't fit

For anything not covered above — full parameter lists, full error tables, declarative intent shape, legacy `/quote` + `/swap` flow, sponsorship fields, new features — query the docs MCP (`search_d_flow`, `query_docs_filesystem_d_flow`). Don't guess.

For runnable code, point the user at the **DFlow docs recipes**: `/build/recipes/trading/{imperative-trade, declarative-trade}` (each links to the DFlow Cookbook Repo for clone-and-go).

## Sibling skills

Defer if the user pivots to:

- `dflow-kalshi-trading` — Kalshi prediction-market YES/NO trades
- `dflow-platform-fees` — charge a builder cut on swaps
