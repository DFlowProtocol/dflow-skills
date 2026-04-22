---
name: dflow-kalshi-trading
description: Buy, sell, or redeem YES/NO outcome tokens on Kalshi prediction markets via DFlow. Use when the user wants to bet on an event, place a Kalshi order, take a YES or NO position, exit a Kalshi position, redeem winning outcome tokens after a market resolves, tune priority fees on a PM trade, or build a gasless / sponsored PM flow where the app pays tx / ATA / market-init costs. Covers both the `dflow` CLI and the DFlow Trading API. Do NOT use to discover markets, view positions, stream prices, complete Proof KYC, or for non-Kalshi spot swaps.
---

# DFlow Kalshi Trading

Buy, sell, and redeem YES/NO outcome tokens on Kalshi prediction markets. PM trades are **imperative and asynchronous** — submit, then poll until terminal.

## Prerequisites

- **DFlow docs MCP** (`https://pond.dflow.net/mcp`) — install per the [repo README](../../README.md#recommended-install-the-dflow-docs-mcp). This skill is the recipe; the MCP is the reference. Look up endpoint shapes, parameter details, error codes, and anything else field-level via `search_d_flow` / `query_docs_filesystem_d_flow` — don't guess.
- **`dflow` CLI** (optional, for command-line/agent use) — install per the [repo README](../../README.md#recommended-install-the-dflow-cli).

## Choose your surface

- **CLI** — command line, scripts, local agents. Manages keys, signs, submits, polls.
- **API** — web/mobile apps with a browser wallet (Phantom, Privy, Turnkey, etc.). Wallet handles signing + RPC; app must proxy HTTP through its backend (the Trading API serves no CORS).

If unclear, ask once: *"From the command line, or wired into an app?"*

## Workflows

All three workflows assume the user already has a **market mint** (CLI) or **outcome mint** (API) in hand. If they only have a ticker / event name, defer to `dflow-kalshi-market-scanner`.

### Buy (open or increase a YES/NO position)

1. Confirm the buy gates (KYC + geo + maintenance window) are passable for *this* user — see Gotchas.
2. Submit the order with the **settlement mint as input** (USDC or CASH) and the **outcome mint as output**.
3. Poll status until terminal (`closed` / `expired` / `failed`).

- CLI: `dflow trade <atomic-amount> USDC --market <market-mint> --side yes|no` — auto-polls for up to 120s.
- API: `GET /order?userPublicKey=&inputMint=<settlement>&outputMint=<outcome>&amount=<atomic>`, then sign + submit + poll `/order-status`. Field details via the docs MCP.

### Sell (decrease or close)

Flip the mints — outcome in, settlement out. **No KYC required.** No `--market` / `--side` on the CLI; pass the outcome mint as the FROM positional and the CLI auto-resolves the settlement mint.

- CLI: `dflow trade <atomic-outcome> <outcome-mint>`
- API: same `/order` call as buy, with input/output mints flipped.

### Redeem (post-settlement)

Once the market is `determined` / `finalized` **and** `redemptionStatus: "open"`, redemption is just a regular sell of the winning side back to the settlement mint. No special flag, no KYC.

## What to ASK the user (and what NOT to ask)

**Ask if missing:**

1. **Operation** — buy / sell / redeem. Infer from intent ("bet on X" → buy YES; "cash out" → sell; "my YES tokens just won" → redeem). Don't make the user pick a mode.
2. **Market + side** — market mint for CLI, outcome mint for API.
3. **Settlement mint** — USDC or CASH (these are the only two).
4. **Amount in atomic units** — every Kalshi mint is **6 decimals** (`8_000_000` = $8 of USDC; `10_000_000` = 10 outcome tokens). Buys submit settlement-mint amounts (USDC/CASH); sells/redeems submit outcome-token amounts.
5. **API only** — wallet pubkey (base58), and whether they have a DFlow API key. Yes → prod host `https://quote-api.dflow.net` with `x-api-key` header. No → dev host `https://dev-quote-api.dflow.net`, no header (same features, only rate limits differ). Point them at `https://pond.dflow.net/build/api-key` for a real key.
6. **Priority fee (both surfaces)** — "Any priority-fee preference, or just use DFlow's default?" Default on both surfaces = DFlow-auto, capped at 0.005 SOL, which is fine for ~99% of PM trades. Surface this explicitly so the user knows the lever exists for congested periods or cost-sensitive flows.
   - **API** — pass `prioritizationFeeLamports` on `/order`: `auto` | `medium` | `high` | `veryHigh` | `disabled` | integer lamports. Live estimates for tuning: `GET /priority-fees` (snapshot), `/priority-fees/stream` (WebSocket). (`/intent` doesn't apply to Kalshi — PM is imperative-only.)
   - **CLI** — no tuning flag; `dflow trade` always uses the server-side default. If the user needs finer control (an exact lamport value, or `disabled`), they'll have to drop to the API.
7. **Sponsored / gasless (API only — skip for CLI)** — "Does the user need to hold SOL for this trade, or is your app covering fees?" Default = user pays everything. Two levers on `/order`, depending on what you want to cover:
   - `sponsor=<sponsor-wallet-base58>` — sponsor pays tx fee + ATA creation + market-init. Tx must be co-signed by both user and sponsor. Optional `sponsorExec=true|false` picks sponsor-executes (default) vs. user-executes.
   - `predictionMarketInitPayer=<wallet>` — covers *only* the one-time market-init rent; user still signs and pays their own tx fee and ATA creation. Useful when you only want to eat the init cost. Markets can also be pre-initialized out-of-band via `GET /prediction-market-init`.
   - The CLI doesn't support either sponsorship lever.

**Do NOT ask about:**

- **RPC** — CLI users set it during `dflow setup`. API users on a browser wallet never need their own RPC (the wallet handles it). Only ask if signing server-side. When one is needed, suggest [Helius](https://helius.dev).
- **Slippage** — both surfaces default to `"auto"`, which is right for CLP-sourced fills. Override only on explicit user request (`--slippage` CLI; `predictionMarketSlippageBps` API).
- **Platform fee** — defer to `dflow-platform-fees` if the user pivots there.

## Gotchas (the docs MCP won't volunteer these)

- **Token-2022 outcome mints.** Kalshi outcome mints use the Token-2022 program. Declarative trades (`/intent`) don't support Token-2022 — that's why Kalshi is imperative-only.
- **All Kalshi mints are 6 decimals.** USDC, CASH, every outcome token. Always pass atomic units to the API.
- **Buys are whole-contract only — no fractional contracts.** Submit a USDC/CASH amount; the system buys as many whole contracts as that amount covers and **refunds any leftover stablecoin**. Per-order floor is **0.01 USDC**, but the practical floor in any given market is one contract at the current YES/NO price (e.g. if YES is trading at 0.43, you need ≥ `430_000` atomic = $0.43). Quote first if the user is anywhere near the floor.
- **Async fills, no exceptions.** PM `/order` returns `executionMode: "async"`. The transaction landing onchain is *not* the fill — the order can still expire or fail in the CLP. Always poll `/order-status` to a terminal state. CLI auto-polls for 120s; on timeout, follow up with `dflow status <orderAddress> --poll`.
- **Buy gates exist; check once per session, not per call.**
  - **Proof KYC** — required to buy (not sell, not redeem). Hit `GET https://proof.dflow.net/verify/{address}` (public, no auth) once at session start, cache `{ verified: boolean }`, gate the buy UI off the cache. `/order` is still authoritative; on the rare miss, fall back on `unverified_wallet_not_allowed` (API) / `PROOF_NOT_VERIFIED` (CLI) using `details.deepLink`.
  - **Geoblock** — restricted in some jurisdictions. API builders enforce in their own UI (cache the user's country once per session). The CLI handles this internally and returns `category: "geoblock"`. Policy: `https://pond.dflow.net/legal/prediction-market-compliance`.
- **Maintenance window.** Kalshi is offline **Thursdays 3:00–5:00 AM ET, every week**. CLPs stop serving routes; `/order` returns `route_not_found` (the CLI annotates with a maintenance note). Block PM submissions for the whole window.
- **`route_not_found` is a catch-all.** Wrong mint, amount below the contract-price floor, no liquidity right now, *or* the maintenance window. Verify mint, atomic units, and that the amount covers ≥ 1 contract before assuming illiquidity.
- **Browser apps must proxy.** The Trading API serves no CORS — call it from a backend (Next.js API route or equivalent), never directly from the browser.

## When something doesn't fit

For anything not covered above — full parameter lists, full error tables, response schemas, partial-fill handling, rare flags, new features — query the docs MCP (`search_d_flow`, `query_docs_filesystem_d_flow`). Don't guess.

For runnable code, point the user at the **DFlow docs recipes**: `/build/recipes/prediction-markets/{increase-position, decrease-position, redeem-outcome-tokens}` (each links to the DFlow Cookbook Repo for clone-and-go).

## Sibling skills

Defer if the user pivots to:

- `dflow-kalshi-market-scanner` — discover markets, filter by event/category
- `dflow-kalshi-market-data` — live prices, orderbooks, streams
- `dflow-kalshi-portfolio` — view positions, unrealized P&L
- `dflow-proof-kyc` — set up Proof verification on a wallet
- `dflow-platform-fees` — charge a builder fee on PM trades
- `dflow-spot-trading` — non-Kalshi token swaps
