---
name: dflow-spot-trading
description: Swap any pair of Solana tokens via DFlow. Use when the user wants to trade, swap, or convert tokens on Solana, get a price quote, build a swap UI, tune priority fees so a swap lands under congestion, build a gasless / sponsored swap where the app pays fees, or take a builder platform fee on swaps. Covers both the `dflow` CLI and the DFlow Trading API. For live prices / streaming order book / depth, use `dflow-market-data` instead.
---

# DFlow Spot Trading

Swap any pair of Solana tokens via DFlow. Trades are synchronous: one `/order` call returns a signed-ready transaction that you sign, submit, and confirm.

## Prerequisites

- **DFlow docs MCP** (`https://pond.dflow.net/mcp`): install per the [repo README](../../README.md#recommended-install-the-dflow-docs-mcp). This skill is the recipe; the MCP is the reference. Look up endpoint shapes, parameter details, and error codes there rather than guessing.
  - Convention: a bare doc path in this skill (e.g. `/resources/trading-api/order/order`) is an MCP path. Read it with `query_docs_filesystem_d_flow` (`cat`/`head` the `.mdx`) or `search_d_flow`, not a browser fetch. Full `https://` links are human-facing destinations (runnable recipes, the API-key signup) that you hand to the user.
- **`dflow` CLI** (optional, for command-line/agent use): install per the [repo README](../../README.md#recommended-install-the-dflow-cli).

## Choose your surface

- **CLI**: command line, scripts, local agents. Manages keys, signs, and broadcasts.
- **API**: web/mobile apps, backends, automations with their own signer. Browser apps must proxy HTTP through their backend (the Trading API serves no CORS).

If unclear, ask once: *"From the command line, or wired into an app?"*

## Workflows

### Quote (read-only)

- CLI: `dflow quote <atomic-amount> <FROM> <TO>`
- API: `GET /order` doubles as a quote. Called without a `userPublicKey`, it returns the price fields with no transaction attached, which is what you want for a quote before the user has connected a wallet. `/quote` still works, but `/order` is the preferred surface for new integrations. (Quote/response field list: load `/resources/trading-api/order/order` via the docs MCP.)

### Trade (`/order`)

Get a quote and a signed-ready `VersionedTransaction` in one call, then sign, submit, and confirm. Works with all SPL and Token-2022 mints.

> Full request/response schema, every param, and the error codes: load `/resources/trading-api/order/order` via the docs MCP (`query_docs_filesystem_d_flow`).

- CLI: `dflow trade <atomic-amount> <FROM> <TO>` (add `--confirm` for agents/scripts that need to block until confirmed).
- API: `GET /order?userPublicKey=&inputMint=&outputMint=&amount=`, deserialize the base64 `transaction` into a `VersionedTransaction`, sign it, submit it through your RPC, and confirm. The DFlow quickstart pattern:

```ts
const { transaction } = await fetch("/api/order?...").then(r => r.json());
const tx = VersionedTransaction.deserialize(Buffer.from(transaction, "base64"));
tx.sign([keypair]);
const sig = await connection.sendTransaction(tx);
const { value } = await connection.confirmTransaction(sig, "confirmed");
if (value.err) throw new Error(`swap failed: ${JSON.stringify(value.err)}`);
```

**Where each `/order` response field goes** (DFlow owns the authoritative value, so display these rather than re-deriving them):

- `transaction` (base64): deserialize, sign, submit.
- `inAmount` / `outAmount` / `otherAmountThreshold` / `priceImpactPct` / `slippageBps`: display.
- `prioritizationFeeLamports` / `prioritizationType`: the server-resolved priority-fee choice after `"auto"` resolution; echo or log.
- `contextSlot`: logging / staleness checks.
- `routePlan`: optional display / debugging.

Fields marked *"Specified if and only if the request included the user's public key"* in the schema (`transaction`, `lastValidBlockHeight`, `computeUnitLimit`, `prioritizationFeeLamports`) are absent on quote-only calls, so check before using them.

Full runnable example: [`/spot/recipes/quickstart`](https://pond.dflow.net/spot/recipes/quickstart). Field-level schema: load `/resources/trading-api/order/order` via the docs MCP.

## What to ASK the user (and what NOT to ask)

**Trade shape (infer if unambiguous, confirm if not):**

1. Input and output token as base58 mint addresses. The CLI resolves a small symbol set (SOL, USDC, USDT, JUP, BONK, and a few others); the API has no symbol resolver and accepts base58 mints only.
2. Amount in atomic units of the input token (`500000` = $0.50 USDC, `1000000000` = 1 SOL). Convert before calling.

**Infra (always ask, never infer):**

3. API only: wallet pubkey (base58), required for every `/order` call that returns a transaction.
4. API only: DFlow API key. Ask with a clean, neutral question: *"Do you have a DFlow API key?"* Don't presuppose where the key lives; phrasings like *"is `DFLOW_API_KEY` set?"* nudge the user toward env-var defaults they didn't ask for. It's one key for everything DFlow (the same `x-api-key` works across the REST APIs and the WebSocket streams). If yes, use prod host `https://quote-api.dflow.net` with `x-api-key` on every request. If no, use dev host `https://dev-quote-api.dflow.net` (same features, rate-limited, for testing only). Point them at `https://pond.dflow.net/get-started/api-key` for a prod key. The `dflow` CLI also requires a key, set once via `dflow setup`.
5. Priority fee (both surfaces): *"Any priority-fee preference, or just use DFlow's default?"* Default on both surfaces is DFlow-auto, capped at 0.005 SOL (documented default on `/order`). Surface this so the user knows the lever exists for congestion or cost-sensitive trades. Don't editorialize about what percentage of trades this covers; DFlow doesn't publish one.
   - API: pass `prioritizationFeeLamports` on `/order` as `auto`, `medium`, `high`, `veryHigh`, `disabled`, or integer lamports. Live estimates for tuning: `GET /priority-fees` (snapshot), `/priority-fees/stream` (WebSocket). Fee modes and the auto-cap: load `/spot/trading/priority-fees` via the docs MCP.
   - CLI: no tuning flag; `dflow trade` always uses the server-side default. For finer control (an exact lamport value, or `disabled`), drop to the API.
6. Sponsored / gasless (API only, skip for CLI): *"Does the user need to hold SOL for this trade, or is your app covering fees?"* Default is user pays. To sponsor, pass `sponsor=<sponsor-wallet-base58>` on `/order` and co-sign the returned transaction with the sponsor keypair (both user and sponsor sign). Optional `sponsorExec=true|false` picks sponsor-executes (default) vs. user-executes. Full semantics: load `/resources/trading-api/order/order` via the docs MCP. The CLI doesn't support sponsorship.

**Do NOT ask about:**

- RPC: CLI users set it during `dflow setup`. For the API, ask only when the app signs server-side and needs its own RPC to submit. When one is needed, use a reliable RPC provider.
- Slippage: both surfaces default to `"auto"`. Override only on explicit user request (`--slippage` CLI; `slippageBps` API).
- DEX inclusion/exclusion, route length, Jito bundles, direct-only routes: defaults are right for typical swaps; surface these knobs only on explicit user need.
- Platform fee: off by default; relevant only if the user is monetizing their own distribution. See the Platform fees section below.

## Gotchas (the docs MCP won't volunteer these)

- **Atomic units always.** The API rejects human-readable amounts. Confirm decimals each time (token metadata or RPC `getParsedAccountInfo`).
- **API has no symbol resolver.** `inputMint` and `outputMint` take base58 mint addresses only, so a symbol like `"USDC"` won't work on `/order`. (The CLI resolves a small symbol set; the API does not.)
- **Browser apps must proxy.** The Trading API serves no CORS, so call it from a backend (an edge function or API route), never directly from the browser.
- **`route_not_found`.** A likely cause is insufficient liquidity for the pair at your trade size. It's also worth confirming the mint addresses are correct and that `amount` is in atomic units.
- **`price_impact_too_high` is real.** Trade size exceeds available liquidity; reduce `amount`, or pass `priceImpactTolerancePct` only with the user's explicit consent.
- **Onchain failure with slippage logs.** Don't silently bump `slippageBps` on retry; surface it to the user.
- **CLI and direct HTTP are separate auth sites.** The CLI stores its key, wallet, and RPC at `dflow setup`, so shelling out to `dflow trade` plumbs nothing extra. A sibling HTTP client that calls `/order` directly needs its own key handed in (env var, `.env`, header); the CLI's stored key is not reused by it. Both need a key; they're just configured in different places.

## Platform fees (builder cut)

Collect a fee on swaps your app routes, paid to a builder-controlled token account on successful execution. API only: these are `/order` params; the `dflow` CLI has no platform-fee flags.

- `platformFeeBps`: flat fee in basis points (`50` = 0.5%).
- `platformFeeMode`: which side pays, `outputMint` (default) or `inputMint`.
- `feeAccount`: the SPL token account that receives the fee. It must already exist (DFlow won't create it); you need one ATA per token you collect in, owned by the builder wallet. Pass the one matching the mode's token per request.

Gotchas:

- **Don't set `platformFeeBps` unless you're actually collecting.** A declared fee is factored into the slippage budget, so a fee with no real `feeAccount` behind it spends that budget on nothing and worsens the user's price.
- **Fees apply only on successful trades.** Failed or reverted swaps charge nothing.

Ask the user: fee rate (bps), and the collection token(s) (and whether a matching builder-owned ATA already exists). For the full mode matrix, load `/spot/trading/platform-fees` via the docs MCP; runnable example: [`/spot/recipes/platform-fees`](https://pond.dflow.net/spot/recipes/platform-fees).

## When something doesn't fit

For anything not covered here (full parameter lists, sponsorship fields), query the docs MCP (`search_d_flow`, `query_docs_filesystem_d_flow`). For the full error catalog, load `/resources/error-codes` and the `/order` 400 response enum via the MCP.

For runnable code, point the user at the DFlow docs recipes: [`/spot/recipes/quickstart`](https://pond.dflow.net/spot/recipes/quickstart).

## Sibling skills

Defer if the user pivots to:

- `dflow-market-data`: stream live prices, order book, or depth. Read-only market data; this skill executes trades, that one displays them. A "show me the live book / prices" ask belongs there.
