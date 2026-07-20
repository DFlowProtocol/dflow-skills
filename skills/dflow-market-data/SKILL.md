---
name: dflow-market-data
description: Stream real-time DFlow spot market data over WebSocket — live top-of-book quotes, ten-level order-book depth, and priority-fee estimates for Solana token pairs. Use when the user wants a live price ticker, a streaming order book / depth chart / heatmap, real-time bid/ask for a pair, or to watch priority fees without polling. Read-only market data. Do NOT use to place a swap (that's `dflow-spot-trading`) or for builder platform fees (`dflow-platform-fees`).
---

# DFlow Market Data (Streaming)

Real-time market data for Solana spot pairs over WebSocket. **Read-only** — this streams prices, depth, and fee estimates for display/analysis; to actually execute a swap, use `dflow-spot-trading`.

## The three streams

All are paths on the DFlow Trade API **WebSocket** host — prod `wss://quote-api.dflow.net`, dev `wss://dev-quote-api.dflow.net`. One connection; subscribe per pair.

| Stream | Path | Gives you |
|---|---|---|
| Quotes | `/quote-stream` | live top-of-book bid/ask for a pair |
| Order book | `/book-stream` | ten levels of depth per side |
| Priority fees | `/priority-fees/stream` | live priority-fee estimates (no polling) |

> **Access to the quote and book streams is GATED at the API-key level.** A key that works fine for `/order` and the rest of the Trading API is **not** automatically allowed on the streams — the DFlow team has to enable stream access on that specific key. Until they do, the WebSocket upgrade is rejected no matter how correct the code is. Flag this up front: the most confusing failure here is a **valid, working key that the stream still refuses** — that's a permission on the key, not a bug in the integration. Point the user at the team to request stream access.

## Prerequisites

- **DFlow docs MCP** (`https://pond.dflow.net/mcp`) — this skill is the recipe; the MCP is the reference. **Before wiring a specific stream, load its message-schema page now** — `search_d_flow` / `query_docs_filesystem_d_flow` for `/resources/trading-api/websockets/{overview,quote-stream,book-stream,priority-fees-stream}`. Don't guess field names.
- A **DFlow API key** — see the api-key note under "What to ASK".

## THE gotcha: browsers can't set WebSocket headers → you MUST proxy the stream

DFlow authenticates the stream with an **`x-api-key` header on the WebSocket upgrade**. The browser `WebSocket` API **cannot set request headers** — its constructor is `new WebSocket(url, protocols)`, with no headers option (verified in a real browser engine: passing `{ headers }` sends nothing and can break the handshake). So **a browser cannot connect to the stream directly.**

Run a small **backend relay**: open the upstream WS with the header, pipe frames down to the browser over a header-less local WS. This is the WebSocket twin of the spot skill's "browser must proxy the Trading API (no CORS)" rule.

**Node / server-side has no such limit** — the `ws` library accepts `{ headers: { "x-api-key": KEY } }`. So: **Node/CLI → connect directly with the header; browser → proxy through your backend.**

```js
// Backend relay (Node, `ws`). Browser connects to THIS; it injects the key upstream.
import { WebSocketServer, WebSocket } from "ws";
const wss = new WebSocketServer({ server });          // same origin as your app
wss.on("connection", (client) => {
  const up = new WebSocket(`${process.env.DFLOW_TRADE_API_WS_URL}/book-stream`,
    { headers: { "x-api-key": process.env.DFLOW_API_KEY } });   // header only works server-side
  const q = [];
  up.on("open", () => { q.forEach((m) => up.send(m)); q.length = 0; });
  up.on("message", (d) => client.readyState === 1 && client.send(d.toString()));
  client.on("message", (m) => up.readyState === 1 ? up.send(m.toString()) : q.push(m.toString()));
  client.on("close", () => up.close()); up.on("close", () => client.close());
});
```

## Subscribe + handle frames (quote & book)

- **Subscribe:** `{ "op": "subscribe", "base_mint": "<mint>", "quote_mint": "<mint>" }` — **base58 mints, not symbols**. `unsubscribe` mirrors it. One connection multiplexes many pairs.
- **Frames batch per slot:** `{ u: <slot>, ts, updates: [ { sb, sq, ... } ] }`. Each entry in `updates[]` is keyed by its subject mints (`sb`/`sq`); a per-pair error arrives inline as `{ e: <code>, sb, sq }` — handle it **without tearing down the whole feed**. Load the stream's doc page for the exact per-level fields (`b`/`a`, `mid`, `tick`, `cumulative_size_human`, …).
- **Reconnect + re-subscribe.** WebSockets drop. On reopen, resend every subscription; back off (e.g. 500ms → 8s). Track the slot `u` (and `skipped`, default 0) to detect gaps.

## Caveats — set expectations

Book/quote levels are **approximations**: the book is **direct-routes-only** (10 levels); quotes are **direct + one-hop**, computed from a ~$10 USDC round-trip. They can differ from the real `/order` quote at trade time. Don't present them as an exchange-grade CLOB or as the executable price — for the price a user will actually get, quote `/order` (see `dflow-spot-trading`).

## What to ASK

- **API key.** Ask neutrally: *"Do you have a DFlow API key?"* — don't presuppose env vars. It's **one key for everything DFlow** (same `x-api-key`, Trade + Metadata, REST + WebSocket). Yes → prod `wss://quote-api.dflow.net` + `x-api-key`. No → dev `wss://dev-quote-api.dflow.net` (rate-limited). Prod key: `https://pond.dflow.net/get-started/api-key`.
- **Surface** — **browser** (needs the proxy above) or **Node/server** (connect directly with the header)?

## When something doesn't fit

Per-stream message schema, ping/keepalive, priority-fee fields → docs MCP (`/resources/trading-api/websockets/*`). Runnable reference: the **DFlow order-book-visualization** demo (a complete `/book-stream` + proxy build), plus docs recipes [`/spot/recipes/stream-order-book`](https://pond.dflow.net/spot/recipes/stream-order-book) and [`/spot/recipes/stream-quotes`](https://pond.dflow.net/spot/recipes/stream-quotes).

## Sibling skills

- `dflow-spot-trading` — to actually execute a swap (and to take a builder platform fee). **A "show live prices / order book / depth" task belongs here; a "buy / swap / convert" task belongs in `dflow-spot-trading`.**
