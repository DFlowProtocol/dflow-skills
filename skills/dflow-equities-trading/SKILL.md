---
name: dflow-equities-trading
description: Use when building an app, script, or CLI that buys or sells tokenized equities (stocks and ETFs such as AAPL) on Solana through the DFlow Trading API, including equity mint lookup, sync and async orders, the submit endpoint, and tracking async orders. For other SPL tokens, use dflow-crypto-trading.
---

# DFlow Tokenized Equities Trading

Buy or sell a tokenized equity with `GET /order`, the same endpoint as crypto. A market maker fills the order. The response says how it settles, and the app submits and tracks it accordingly.

## Ask First

Ask only what the request leaves open.

1. **Target:** web UI, backend service, or CLI.
2. **Endpoints:** developer or production. Both run on mainnet with the same features. Developer endpoints need no API key and have lower rate limits. Production sends the key as an `x-api-key` header and has higher limits. See [Endpoints](https://pond.dflow.net/get-started/endpoints).
3. **Mint selection:** `GET /equities` can list more than one mint for a ticker, one per provider. Liquidity and pricing differ between them, so how the app chooses is the developer's design decision. Raise it, and never default to the first entry.
4. **Platform fee:** DFlow takes no protocol fee. Will the integration monetize trades with its own platform fee? If so, get the rate in bps, `platformFeeMode`, and a `feeAccount`. See [Equity Fees](https://pond.dflow.net/equities/equity-fees).
5. **Gas sponsorship:** should trades be gasless for users, with a sponsor wallet paying transaction fees and token account creation costs? If so, use `dflow-sponsored-swaps`.

## Journey

1. **Set up.** `npm install @solana/kit`. Use `@solana/kit` 8.0.0 or later. Read `DFLOW_TRADE_API_URL`, `DFLOW_EQUITIES_SUBMIT_URL`, `DFLOW_API_KEY`, `SOLANA_RPC_URL`, and `SOLANA_PRIVATE_KEY` from the environment.
2. **Look up the mint.** `GET /equities` returns `{ ticker, mint, provider }` entries. A ticker can appear once per provider, each with its own mint. Resolve the ticker to one mint, and when a ticker lists several, follow the developer's selection design. See [Find Equities](https://pond.dflow.net/equities/recipes/find-equities).
3. **Request the order.** One `/order` call with `userPublicKey` and `transactionVersion: "v1"` returns the quote and the transaction. USDC and an equity mint are the pair: a buy has USDC as `inputMint`, and a sell has the equity as `inputMint`. Another token paired with an equity routes through USDC. An equity paired with another equity returns `unsupported_equity_pair`. A sell's `amount` is the raw token amount in the mint's decimals (read with `getTokenSupply`), before the scaled UI multiplier.
4. **Sign.** Use `partiallySignTransaction`. The market maker signs after the user, and `signTransaction` throws on a transaction that is not fully signed.
5. **Submit.** Branch on the response:
   - `submitEndpointRequired: true`: send the signed transaction to the submit endpoint.
   - Otherwise: send it through your own RPC.
6. **Confirm or track.** Branch on `executionMode`:
   - `sync`: the trade settles in one transaction. Poll `getSignatureStatuses` until `confirmed` or `finalized`, or until the block height passes `lastValidBlockHeight`.
   - `async`: the first transaction opens an escrow and the maker fills it later. The user cannot cancel it. Track it with `GET /order-status` until it is `expired`, `failed`, or `closed`. See [Track an Async Equity Order](https://pond.dflow.net/equities/recipes/track-order-status).
7. **Display amounts.** When the response has `outAmountScaledUi` or `inAmountScaledUi`, show those instead of the raw amounts. [Track Equity Balances](https://pond.dflow.net/equities/recipes/track-equity-balances) covers balances.

A buy is `sync` when the best-priced market maker has inventory and `async` otherwise. A sell is always `sync`. Handle both modes. See [Equity Order Execution](https://pond.dflow.net/equities/orders).

Once a signature has been submitted, request a new order for the same trade only after that signature is confirmed, failed, or past `lastValidBlockHeight`, because a late landing and a second order would both execute.

## Example

The same flow with more detail is in [Send Equity Orders](https://pond.dflow.net/equities/recipes/quickstart).

```typescript
import {
  createKeyPairSignerFromBytes, createSolanaRpc, getBase58Encoder, getBase64Encoder,
  getBase64EncodedWireTransaction, getTransactionDecoder, partiallySignTransaction,
} from "@solana/kit";

// Defaults to the developer Trading API endpoint: no API key, lower rate limits.
// Production apps set DFLOW_TRADE_API_URL=https://quote-api.dflow.net and DFLOW_API_KEY.
// Set DFLOW_EQUITIES_SUBMIT_URL to the submit endpoint that matches the Trading API host. The Endpoints page lists both.
const API = process.env.DFLOW_TRADE_API_URL ?? "https://dev-quote-api.dflow.net";
const SUBMIT_URL = process.env.DFLOW_EQUITIES_SUBMIT_URL ?? "https://submit.equities-api.dflow.net/submit-transaction";
const headers: HeadersInit = {};
if (process.env.DFLOW_API_KEY) headers["x-api-key"] = process.env.DFLOW_API_KEY;

const rpc = createSolanaRpc(process.env.SOLANA_RPC_URL ?? "https://api.mainnet-beta.solana.com");
const submitter = createSolanaRpc(SUBMIT_URL, { headers });

// Keep the private key in the environment, never inline.
const signer = await createKeyPairSignerFromBytes(
  getBase58Encoder().encode(process.env.SOLANA_PRIVATE_KEY ?? "")
);

const TICKER = "AAPL";
const equities: { ticker: string; mint: string; provider: string }[] =
  await fetch(`${API}/equities`, { headers }).then((r) => r.json());
const matches = equities.filter((e) => e.ticker === TICKER);
// A ticker can list one mint per provider. Liquidity and pricing differ between them,
// so choosing is an app decision. Do not default to the first entry.
if (matches.length !== 1) throw new Error(`${TICKER} lists ${matches.length} mints: choose one`);
const equity = matches[0];

const params = new URLSearchParams({
  inputMint: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", // USDC
  outputMint: equity.mint,
  amount: "1000000", // 1 USDC, atomic units
  userPublicKey: signer.address,
  transactionVersion: "v1",
});
const order = await fetch(`${API}/order?${params}`, { headers }).then((r) => r.json());

const tx = getTransactionDecoder().decode(getBase64Encoder().encode(order.transaction));
const signed = await partiallySignTransaction([signer.keyPair], tx);
const sender = order.submitEndpointRequired ? submitter : rpc;
const signature = await sender
  .sendTransaction(getBase64EncodedWireTransaction(signed), { encoding: "base64" })
  .send();

if (order.executionMode === "async") {
  // Track with GET /order-status (signature and lastValidBlockHeight) until a terminal status.
} else {
  // Confirm onchain through your own RPC, even when the submit endpoint sent it.
  let status = null;
  while (status?.confirmationStatus !== "confirmed" && status?.confirmationStatus !== "finalized") {
    if ((await rpc.getBlockHeight().send()) > BigInt(order.lastValidBlockHeight)) {
      throw new Error("Transaction expired before it landed");
    }
    await new Promise((resolve) => setTimeout(resolve, 1000));
    [status] = (await rpc.getSignatureStatuses([signature]).send()).value;
  }
}
```

## Rules

- Call `/order` without `userPublicKey` only when the wallet is not known yet. The response has the price fields and no transaction. Once the wallet is known, make the single call above.
- The Trading API accepts base58 mints only. Resolve any ticker to a mint with `GET /equities`.
- The Trading API sets no CORS headers. Call `/order` from a server so the API key stays there. A browser app calls its own backend route.
- Set a nonzero `platformFeeBps` only when the trade collects the fee. `feeAccount` is required and must be an existing token account for the fee mint. A `sync` order honors `platformFeeMode`. An `async` order always charges the fee in USDC, to the `feeAccount` you passed, on the filled amount. If the fee account is missing or frozen, the fill still executes and no fee is charged. See [Equity Fees](https://pond.dflow.net/equities/equity-fees).
- Equity request parameters such as `allowAsyncExec`, `allowSyncExec`, `destinationTokenAccount`, and `revertWallet` change how an order settles. Read [Equity Order Execution](https://pond.dflow.net/equities/orders) before setting any of them.
- Submit and order errors are listed in [Error Codes](https://pond.dflow.net/resources/error-codes#equities-api).

## Reference

Query the DFlow docs MCP (`https://pond.dflow.net/mcp`) for anything not covered here. Without it, start from [llms.txt](https://pond.dflow.net/llms.txt).
