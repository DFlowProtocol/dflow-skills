---
name: dflow-alchemy
description: Query and integrate Alchemy blockchain APIs across EVM (Ethereum, Base, Arbitrum, BNB, …) and Solana. Use when the user wants token balances, NFT ownership/metadata, transfer history, token prices (spot or historical), multi-chain portfolio views, transaction simulation, webhooks, or raw JSON-RPC / Solana RPC — including to enrich a DFlow trading flow (check balances, price a token, read transfers). Covers both the API-key path (`$ALCHEMY_API_KEY`) and the keyless wallet-paid gateway (x402 / MPP). Do NOT use for executing DFlow swaps or Kalshi trades — that's `dflow-spot-trading` / `dflow-kalshi-trading`.
---

# Alchemy

Read blockchain data and wire Alchemy's APIs into code, across EVM chains and Solana. This is the **data / RPC / infra** companion to the DFlow trading skills: use it to look up balances, prices, NFTs, transfers, and portfolio state, or to run raw RPC — DFlow still owns trade execution.

## Prerequisites

- **An auth path** — see [Choose your auth path](#choose-your-auth-path) below. The default is a standard API key; the keyless gateway is the fallback when no key is available.
- **Alchemy docs** (`https://www.alchemy.com/docs`) — this skill is the *recipe* (which endpoint, which host, the gotchas); the docs are the *reference* (every parameter, every supported network, every error). Look field-level details up there — don't guess.

## Choose your auth path

Run the preflight gate before any network call:

1. **Is `$ALCHEMY_API_KEY` set?** Check with `echo $ALCHEMY_API_KEY`.
2. **If set → API-key path.** Put the key in the URL (or header for gRPC/Notify). This is the normal path.
3. **If unset → ask once:** *"Do you have an Alchemy API key (free at https://dashboard.alchemy.com/), or should I use the keyless wallet-paid gateway?"*
   - **Gets a key** → API-key path. Set `ALCHEMY_API_KEY` in the project `.env` (and ensure `.env` is git-ignored). Never echo the key value back.
   - **Wallet-paid** → gateway path (x402 or MPP). Wallet-based auth (SIWE for EVM, SIWS for Solana), pay-per-request in USDC. See [Keyless gateway](#keyless-gateway-x402--mpp).

**Never fall back to a public RPC (`publicnode`, `cloudflare-eth`, `llamarpc`, …) or a demo key (`/v2/demo`).** If neither auth path is available, stop and tell the user — don't silently degrade to an unauthenticated endpoint.

## Base URLs + auth (API-key path)

`$ALCHEMY_API_KEY` goes in the URL for HTTP/RPC; `<network>` is a **lowercase** slug like `eth-mainnet`, `base-mainnet`, `arb-mainnet`, `solana-mainnet`.

| Product | Base URL | Auth |
| --- | --- | --- |
| EVM JSON-RPC | `https://<network>.g.alchemy.com/v2/$ALCHEMY_API_KEY` | key in URL (HTTPS + WSS) |
| Solana RPC | `https://solana-mainnet.g.alchemy.com/v2/$ALCHEMY_API_KEY` | key in URL |
| NFT API | `https://<network>.g.alchemy.com/nft/v3/$ALCHEMY_API_KEY` | key in URL |
| Prices API | `https://api.g.alchemy.com/prices/v1/$ALCHEMY_API_KEY` | key in URL |
| Portfolio API | `https://api.g.alchemy.com/data/v1/$ALCHEMY_API_KEY` | key in URL |
| Notify (webhooks) | `https://dashboard.alchemy.com/api` | `X-Alchemy-Token: <notify-token>` |

## Endpoint selector (top tasks)

| You need | Method / route |
| --- | --- |
| EVM read/write | JSON-RPC `eth_*` |
| Realtime events | `eth_subscribe` (WSS) |
| Token balances | `alchemy_getTokenBalances` |
| Token metadata | `alchemy_getTokenMetadata` |
| Transfer history | `alchemy_getAssetTransfers` |
| NFT ownership | `GET /getNFTsForOwner` |
| NFT metadata | `GET /getNFTMetadata` |
| Token price (spot) | `GET /tokens/by-symbol` |
| Token price (historical) | `POST /tokens/historical` |
| Portfolio (multi-chain) | `POST /assets/*/by-address` |
| Simulate a tx | `alchemy_simulateAssetChanges` |
| Create a webhook | `POST /create-webhook` (Notify) |
| Solana NFT/asset data | `getAssetsByOwner` (DAS) |

## Quickstart (API-key path)

```bash
# EVM JSON-RPC read
curl -s https://eth-mainnet.g.alchemy.com/v2/$ALCHEMY_API_KEY \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}'

# ERC-20 token balances for an address
curl -s https://eth-mainnet.g.alchemy.com/v2/$ALCHEMY_API_KEY \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"alchemy_getTokenBalances","params":["0x..."]}'

# Spot prices by symbol
curl -s "https://api.g.alchemy.com/prices/v1/$ALCHEMY_API_KEY/tokens/by-symbol?symbols=ETH&symbols=USDC"

# NFT ownership
curl -s "https://eth-mainnet.g.alchemy.com/nft/v3/$ALCHEMY_API_KEY/getNFTsForOwner?owner=0x..."
```

## Keyless gateway (x402 / MPP)

For app code paying per-request with a wallet instead of a key. Two protocols — **ask the user which** before proceeding; don't pick for them:

- **x402** — USDC per call. Header `Payment-Signature: <base64>`. Supports EVM (SIWE) **and** Solana (SIWS). Libraries: `@alchemy/x402`, `@x402/fetch`. Gateway `https://x402.alchemy.com`, SIWE/SIWS domain `x402.alchemy.com`.
- **MPP** — Merchant Payment Protocol; pay with on-chain USDC (Tempo) or credit card (Stripe). EVM (SIWE) only. Library: `mppx`. Gateway `https://mpp.alchemy.com`.

Same API surface as the key path, just a different host and auth header. Replace `https://<network>.g.alchemy.com/v2/$KEY` with `https://x402.alchemy.com/<network>/v2` (or the MPP host) and attach the wallet auth + payment headers per protocol. On a `402`, extract the challenge header (`PAYMENT-REQUIRED` for x402, `WWW-Authenticate` for MPP), create the payment, and retry. Wallet type (EVM/Solana) is independent of the chain you're querying. The `mppx-account` and `mppx-sign` skills cover the MPP wallet + signing steps.

For full setup (wallet bootstrap, SIWE/SIWS signing, payment flow), see the gateway docs at `https://www.alchemy.com/docs`. **Never read/write wallet key files (`wallet.json`, `wallet-key.txt`, `.env`) with file tools.**

## What to ASK the user (and what NOT to ask)

**Ask, never infer:**

1. **Auth path** — only when `$ALCHEMY_API_KEY` is unset (see the gate). Neutral phrasing: *"Do you have an Alchemy API key, or use the keyless gateway?"*
2. **Gateway protocol** — x402 vs MPP, only on the keyless path. Don't choose for them.
3. **Which chain / network** — if ambiguous. EVM vs Solana, mainnet vs testnet.

**Infer if unambiguous:** the endpoint (from the task — use the [selector](#endpoint-selector-top-tasks)), the address/symbol from what the user gave you.

**Do NOT ask about:** RPC provider (it's Alchemy), or whether to use a public fallback (never).

## Gotchas (the docs won't always volunteer these)

- **Network slug casing differs by product.** Data APIs and JSON-RPC use **lowercase** (`eth-mainnet`, `base-mainnet`); the Notify/webhooks API uses **UPPERCASE** (`ETH_MAINNET`). Mixing them up is the most common 400.
- **JSON-RPC errors come back with HTTP 200.** The failure is in the `error` field of the body, not the status code. Always check `response.error`, not just the HTTP status.
- **`429` = rate limited.** Back off exponentially with jitter; check the compute-unit budget in the dashboard. Don't hammer-retry.
- **Paginate with `pageKey`.** `alchemy_getTokenBalances`, `alchemy_getAssetTransfers`, and the Portfolio endpoints page via `pageKey` — resume from it after a partial/failed fetch rather than refetching from the top.
- **Test on a testnet first** before pointing code at mainnet.
- **Never surface the API key** in conversation output, logs, or committed files. Treat it like a password; keep it in git-ignored `.env`.

## When something doesn't fit

For anything not covered above — full parameter lists, every supported network, error tables, Solana DAS / Yellowstone gRPC, Sui gRPC, Account Kit / smart wallets, webhook payloads and signature verification — see the Alchemy docs (`https://www.alchemy.com/docs`). For **live, in-session** querying or admin (not shipping code), the Alchemy CLI (`npm i -g @alchemy/cli`, then `alchemy <command>`) and the hosted Alchemy MCP (`https://mcp.alchemy.com/mcp`) are the better surfaces.

## Sibling skills

Defer if the user pivots to:

- `dflow-spot-trading` — execute a Solana token swap (Alchemy reads data; DFlow does the trade)
- `dflow-kalshi-trading` — Kalshi prediction-market YES/NO trades
- `mppx-account` / `mppx-sign` — manage and sign with the MPP wallet used by the keyless gateway
