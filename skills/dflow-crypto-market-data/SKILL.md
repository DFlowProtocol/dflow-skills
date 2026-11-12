---
name: dflow-crypto-market-data
description: Use when building a live price display, order book or depth chart, or any read-only market data feed for Solana token pairs from DFlow WebSocket streams, without placing a trade. Crypto pairs only.
---

# DFlow Crypto Market Data

The Trading API streams live market data over WebSocket. Subscribe to a pair of mints and read an update each slot, with no polling.

| Stream | Path | What you get | Coverage |
| --- | --- | --- | --- |
| Quote stream | `/quote-stream` | Top-of-book bid and ask | Direct and one-hop routes |
| Book stream | `/book-stream` | Ten levels of depth per side | Direct routes only |

The streams cover crypto pairs. Use the quote stream for price displays, pre-trade pricing, and many pairs at once. Use the book stream for depth: order book visualizations, depth charts, and sizing larger trades against available liquidity.

## Ask First

Ask only what the request leaves open.

1. **Need:** top-of-book price or depth. This picks the stream.
2. **Target:** web UI, backend service, or CLI.
3. **Endpoints:** developer or production. Both run on mainnet with the same features. Developer endpoints need no API key and have lower rate limits. Production sends the key as an `x-api-key` header, and access to each stream is granted per API key. See [Endpoints](https://pond.dflow.net/get-started/endpoints).

## Journey

1. **Set up.** `npm install ws`. The WebSocket base URL is `wss://dev-quote-api.dflow.net` on the developer endpoints and `wss://quote-api.dflow.net` on production. Append the stream path.
2. **Connect.** Production sends `x-api-key` as a header on the upgrade request, never as a URL query parameter. A key that works for `/order` has no stream access until stream access is enabled for it. [Request an API key](https://pond.dflow.net/get-started/api-key) with stream access.
3. **Subscribe.** Once the socket is open, send `{ "op": "subscribe", "base_mint": "<mint>", "quote_mint": "<mint>" }`. Send `unsubscribe` with the same fields to stop a pair. Mints are base58 addresses.
4. **Read updates.** Each message is a batch for one slot: `u` is the slot, `ts` is unix seconds, and `updates` has one entry per pair. An entry with an `e` field carries an error code in place of prices, such as `no_route` on the quote stream and `no_market` on the book stream. A mint the stream does not track returns a separate frame, `{ "type": "bad_request", "message": "..." }`, in place of an update. Surface its `message` to the user.
5. **Render.** See the field lists below.

## Fields

Quote stream update (`updates[]`):
- `sb`, `sq`: base and quote mints.
- `b`, `a`: bid and ask price, quote per base. Either is `null` when that side of the pair has no route.
- `B`, `A`: bid and ask size of the $10 USDC equivalent trade the quote is priced on, `null` when its price is.

Book stream update (`updates[]`):
- `sb`, `sq`: base and quote mints.
- `mid`, `tick`: mid price and tick width, quote per base.
- `b`, `a`: ten levels per side, ordered from the spot outward. Each level is `{ price, cumulative_size_human }`, where the size is the cumulative base size from the spot out to that level.
- `crossed`: whether the aggregated book was crossed before it was uncrossed.

[Quote Stream](https://pond.dflow.net/resources/trading-api/websockets/quote-stream) and [Book Stream](https://pond.dflow.net/resources/trading-api/websockets/book-stream) have the complete message formats and TypeScript types.

## Example

```typescript
import WebSocket from "ws";

// Defaults to the developer endpoint: no API key, lower rate limits.
// Production apps set DFLOW_TRADE_API_WS_URL=wss://quote-api.dflow.net and DFLOW_API_KEY.
const WS_URL = process.env.DFLOW_TRADE_API_WS_URL ?? "wss://dev-quote-api.dflow.net";
const API_KEY = process.env.DFLOW_API_KEY;

const SOL = "So11111111111111111111111111111111111111112";
const USDC = "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v";

// "/quote-stream" for top of book, "/book-stream" for depth.
const ws = new WebSocket(`${WS_URL}/quote-stream`, API_KEY ? { headers: { "x-api-key": API_KEY } } : undefined);

ws.on("open", () => {
  ws.send(JSON.stringify({ op: "subscribe", base_mint: SOL, quote_mint: USDC }));
});

ws.on("message", (data) => {
  const frame = JSON.parse(data.toString());
  for (const update of frame.updates ?? []) {
    if (update.e) {
      console.log(`${update.sb} / ${update.sq}: ${update.e}`);
      continue;
    }
    console.log(`slot ${frame.u}  bid ${update.b}  ask ${update.a}`);
  }
});

ws.on("error", (err) => console.error("WebSocket error:", err));
ws.on("close", () => console.log("Connection closed"));
```

## Rules

- Each quote is priced from a $10 USDC equivalent round-trip trade through the best route. The bid is the sell leg and the ask is the buy leg. [Stream Top-of-Book Quotes](https://pond.dflow.net/spot/recipes/stream-quotes) explains how quotes are formed.
- Quotes and book levels are approximations computed at runtime from current routes. They can differ from the quote an `/order` request returns, and a bid can briefly exceed the ask between slots. Present them as a live read on the market, not an execution price.
- Keep the API key on a backend. A browser app connects to a backend proxy that holds the key, connects to DFlow, and relays messages.
- For a single price without a stream, call `/order` without `userPublicKey`. See `dflow-crypto-trading`.
- Priority fee estimates stream at `/priority-fees/stream` with no subscribe message. See [Priority Fees Stream](https://pond.dflow.net/resources/trading-api/websockets/priority-fees-stream).

## Reference

Query the DFlow docs MCP (`https://pond.dflow.net/mcp`) for anything not covered here. Without it, start from [llms.txt](https://pond.dflow.net/llms.txt). Runnable versions of the example: [Stream Top-of-Book Quotes](https://pond.dflow.net/spot/recipes/stream-quotes) and [Stream Order Book Depth](https://pond.dflow.net/spot/recipes/stream-order-book).
