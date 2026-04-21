---
name: dflow-kalshi-market-scanner
description: Find Kalshi prediction markets on DFlow that match a criterion — arbitrage (YES+NO<$1), cheap long-shots, near-certain short-dated plays, biggest movers, widest spreads, highest volume, closing soonest, and series/event-level scans. Use when the user asks "where's the free money?", "any mispriced markets?", "cheap YES with volume", "what moved today?", "markets closing soon", "cheapest YES in this event", "top markets by volume", or "alert me when X happens" (streaming). Do NOT use to place orders (use `dflow-kalshi-trading`), to view a user's own positions (use `dflow-kalshi-portfolio`), or for general live-data plumbing unrelated to a scan (use `dflow-kalshi-market-data`).
---

# DFlow Kalshi Market Scanner

Find Kalshi markets that match a **criterion**. This skill is a set of named **scans** (filter-and-rank recipes) over the DFlow Metadata API.

## Prerequisites

- **DFlow docs MCP** (`https://pond.dflow.net/mcp`) — install per the [repo README](../../README.md#recommended-install-the-dflow-docs-mcp). This skill is the recipe; the MCP is the reference. Look up exact query params, pagination, response shapes, and anything else field-level via `search_d_flow` / `query_docs_filesystem_d_flow` — don't guess.

## Surface

All scans here run against the **Metadata API** (`https://pond.dflow.net/build/metadata-api`) — REST for point-in-time queries, WebSockets for continuous streams. You can call both from anywhere: a quick `curl` from the command line, a Node/Python script, a cron job, a backend service, or a Next.js route proxying a browser UI.

If the user says "run this from my terminal", **don't reach for the `dflow` CLI** — it has no discovery subcommands. Write a short HTTP/WS script that hits the Metadata API instead.

## The scanner skeleton

Every scan is the same four steps. Build around this pattern — don't reinvent it per scan:

1. **Enumerate the universe** — `GET /api/v1/markets` (flat) or `GET /api/v1/events?withNestedMarkets=true` (grouped). Filter to `status=active`. Page through until done. Pass `isInitialized=true` only if the user wants markets tradable on DFlow *right now* (see Gotchas).
2. **Grab the per-market signal.** For top-of-book scans the signal is already on the market object (`yesBid`, `yesAsk`, `noBid`, `noAsk`, `volume24h`, `closeTime`) — **no orderbook call needed**. For momentum use candlesticks or the `prices` / `trades` WebSocket channels. For ladder depth use `/api/v1/orderbook/by-mint/{mint}`. For recent prints use `/api/v1/trades` or the `trades` channel.
3. **Compute the metric.**
4. **Filter and rank.** Return the top-N (default 10, ask if the user wants more).

### Polling vs streaming

Pick the mode that fits the intent:

- **Polling (REST)** — right when the user wants a snapshot ("show me the top 10 right now", "list all markets with X"). Re-run on a cadence if they want it fresh.
- **Streaming (WebSocket)** — right when the user wants to *act on an event* as it happens ("alert me when YES+NO drops below $1", "flag any market that moves > 5% in a minute", "trade when X trades"). Subscribe to the relevant channel (`prices`, `trades`, `orderbook`) at `wss://<host>/api/v1/ws` and compute the metric on each update. For the full streaming plumbing (reconnection, backoff, subscription lifecycle), hand off to `dflow-kalshi-market-data`.

Exact endpoint params, channel payloads, and pagination → docs MCP.

## Scans to offer

Each scan = a user question + a metric. Plug the metric into the skeleton. Default thresholds below are **starting points** — confirm or adjust with the user.

### 1. Arbitrage — `YES + NO < $1`

*"Find markets where I can buy both sides for under a dollar."*
- Metric: `parseFloat(yesAsk) + parseFloat(noAsk) < 1.00`.
- Rank: largest gap (`1 - sum`) descending.
- Skip rows where either `yesAsk` or `noAsk` is null (no resting ask on that side — not a real arb).

### 2. Long-shot YES

*"Cheap YES that's actually trading."*
- Default metric (confirm with user): `yesAsk < 0.03` **and** `volume24h > 100` (dollar-equivalent).
- Rank: `volume24h` descending.
- A 1¢ market with zero volume is noise, not a long-shot — the volume filter is what separates them.

### 3. Near-certain short-dated YES

*"YES above 97¢ closing soon — grind the theta."*
- Default metric (confirm): `yesAsk > 0.97` **and** `closeTime - now < 48h`.
- Rank: `closeTime` ascending.

### 4. Momentum

*"What moved in the last hour?"* or *"Alert me when something moves"*
- **Polling**: `/api/v1/market-candlesticks` per market at the smallest interval, compare latest close vs the close N minutes ago. Per-market and expensive — pre-filter the universe to top-of-volume first.
- **Streaming**: subscribe to the `prices` channel (`all: true` or a ticker list) and compute rolling pct change in memory. Much cheaper for the "alert when X happens" variant, and this is how you'd wire "trade when a market moves > N%" (hand the matching market to `dflow-kalshi-trading`).
- Default threshold (confirm): `abs(pctChange) > 5%` over the last 60 minutes.
- Rank: pct change descending (directional) or absolute (two-sided).

### 5. Widest bid-ask spreads

*"Inefficient markets — market-make or avoid."*
- Metric: `parseFloat(yesAsk) - parseFloat(yesBid)` (NO side is symmetric).
- Rank: spread descending.
- Source: market object; no extra calls.

### 6. Highest volume

*"Where's the action?"*
- Metric: `volume24h` (24h window) or sum over `/api/v1/trades` since a cutoff (intraday). For a live feed, subscribe to the `trades` channel and aggregate in a rolling window.
- Rank: volume descending.

### 7. Closing soonest

*"Theta clock."*
- Metric: `closeTime - now`.
- Rank: ascending.
- Most useful stacked with scan 3 ("near-certain AND closing soon") or scan 6 ("busy AND closing soon").

### 8. Event- and series-level scans

*"Cheapest YES across all outcomes in this event", "do mutually-exclusive buckets sum > 1?"*
- **Within one event** (e.g. "Fed raises rates by X bps" with a bucket per outcome): pull `GET /api/v1/events/{ticker}?withNestedMarkets=true`, then reduce across the nested markets (`min(yesAsk)`, `Σ yesAsk`, etc.). Events are the natural scope for single-winner scans.
- **Across a series**: pull `GET /api/v1/series/{ticker}` plus its events, then roll up.
- **There is no `mutuallyExclusive` flag on series or events.** Summing YES across outcomes only makes sense when the outcomes are a partition of one future (one must happen, exactly one can happen). That's a judgment from the event/series title and contract terms — not a field lookup. When in doubt, surface the numbers and flag the assumption to the user.

## Point lookups (N=1)

When the user already has one market in mind, skip the skeleton:
- By ticker: `GET /api/v1/markets/{ticker}`.
- By outcome mint: `GET /api/v1/market-by-mint/{mint}`.
- By event: `GET /api/v1/events/{eventTicker}?withNestedMarkets=true`.
- Free-text: `GET /api/v1/search` (natural-language to events/markets).

## What to ASK the user (and what NOT to ask)

**Ask if missing:**

1. **Which scan** (or a plain-English intent you can map to one).
2. **Thresholds** the scan needs. Suggest the defaults above (e.g. "under 3¢", "above 97¢", "48h window", "5% move") and let the user override.
3. **Polling vs streaming** — if the intent sounds like "show me now" go REST; if it sounds like "alert me / react when" go WebSocket.
4. **Top-N** (default 10).
5. **API key** — many of these scans touch lots of markets (full-universe enumeration, per-market candlesticks, continuous WS subscriptions) and will hit dev rate limits fast. Ask whether they have a DFlow API key. If yes → prod host (`https://prediction-markets-api.dflow.net` REST, `wss://prediction-markets-api.dflow.net/api/v1/ws` WS) with `x-api-key` on the header (REST **and** the WS upgrade). If no → dev host (`https://dev-prediction-markets-api.dflow.net`, `wss://dev-prediction-markets-api.dflow.net/api/v1/ws`), and point them at `https://pond.dflow.net/build/api-key` for the real thing.

**Do NOT ask about:**
- **RPC, wallet, signing** — this skill is read-only public metadata. No transactions.
- **Settlement mint / slippage / fees** — those are trade-side concerns. If the user pivots to placing an order on a market you surfaced, hand off to `dflow-kalshi-trading`.

## Gotchas (the docs MCP won't volunteer these)

- **Top-of-book lives on the market object.** `yesBid` / `yesAsk` / `noBid` / `noAsk` are already there. Don't loop the orderbook endpoint just to get best prices.
- **The orderbook returns only bid ladders** (`yes_bids`, `no_bids`). Best YES *ask* is derived: `1 - max(no_bids keys)` (a NO bid at `p` is a YES offer at `1-p`). Only matters if the user wants ladder depth.
- **Two price scales.** Market/orderbook prices are 4-decimal probability strings (`"0.4200"`). Trade prices (REST and `trades` channel) are integer 0–10000, with `yes_price_dollars` / `no_price_dollars` string fields alongside. Normalize before you compute.
- **`isInitialized` filter.** Short-duration markets (15-min crypto, etc.) are often active on Kalshi but not yet tokenized on DFlow. Without the filter, scans include them; with `isInitialized=true`, only markets tradable on DFlow right now. Usually you want `true`.
- **Null bids/asks.** Illiquid markets have null top-of-book fields. Every scan that reads them must skip nulls, not treat them as zero.
- **Maintenance window** — Kalshi is offline **Thursdays 3:00–5:00 AM ET**. Top-of-book and volume fields can go stale or missing during the window; WS updates can go quiet. If scans look empty or weirdly wrong during the window, that's why.
- **Pagination.** `/markets` and `/events` paginate. Large scans need the loop — partial results silently drop matches.
- **WebSocket `all: true` is firehose-y.** Subscribing `all: true` on busy channels (esp. `prices`) streams every update across every market. Prefer ticker lists when the scan only cares about a known set; use `all: true` only when the scan really is universe-wide.

## When something doesn't fit

For anything not covered above — full parameter lists, pagination tokens, response schemas, WS reconnection semantics, rare filters (sports, tags, categories, series search), candlestick intervals — query the docs MCP (`search_d_flow`, `query_docs_filesystem_d_flow`). Don't guess.

## Sibling skills

When the user pivots from discovery to action, hand off:
- `dflow-kalshi-trading` — actually buy/sell/redeem a market you found here.
- `dflow-kalshi-portfolio` — view *their* positions and P&L.
- `dflow-kalshi-market-data` — general live orderbook / trade / price streaming outside the "named scan" shape (reconnection patterns, full payload schemas, in-game live data).
- `dflow-proof-kyc` — verify a wallet so it can actually buy what you surfaced.
