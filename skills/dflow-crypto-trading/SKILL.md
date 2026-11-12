---
name: dflow-crypto-trading
description: Use when building an app, script, or CLI that swaps one SPL token for another on Solana through the DFlow Trading API, including price quotes, /order requests, signing, and sending transactions. For tokenized equities, use dflow-equities-trading.
---

# DFlow Crypto Trading

Swap one SPL token for another with `GET /order`. The API returns a quote and a base64 transaction. The user signs it, and the app sends it through its own RPC.

## Ask First

Ask only what the request leaves open.

1. **Target:** web UI, backend service, or CLI.
2. **Endpoints:** developer or production. Both run on mainnet with the same features. Developer endpoints need no API key and have lower rate limits. Production sends the key as an `x-api-key` header and has higher limits. See [Endpoints](https://pond.dflow.net/get-started/endpoints).
3. **Platform fee:** DFlow takes no protocol fee. Will the integration monetize swaps with its own platform fee? If so, get the rate in bps, `platformFeeMode` (`inputMint` or `outputMint`), and a `feeAccount`. See [Platform Fees](https://pond.dflow.net/spot/trading/platform-fees).
4. **Gas sponsorship:** should swaps be gasless for users, with a sponsor wallet paying transaction fees and token account creation costs? If so, use `dflow-sponsored-swaps`.

## Journey

1. **Set up.** `npm install @solana/kit`. Use `@solana/kit` 8.0.0 or later. Read `DFLOW_TRADE_API_URL`, `DFLOW_API_KEY`, `SOLANA_RPC_URL`, and `SOLANA_PRIVATE_KEY` from the environment.
2. **Resolve inputs.** `inputMint` and `outputMint` are base58 mints. `amount` is the input amount in atomic units: SOL has 9 decimals, so 1 SOL is `1000000000`. USDC has 6, so 1 USDC is `1000000`. Read other mints' decimals with `getTokenSupply`. An `amount` that is not in atomic units returns `route_not_found`. [Error Codes](https://pond.dflow.net/resources/error-codes) lists every `/order` error and its fix.
3. **Request the order.** One `/order` call with `userPublicKey` and `transactionVersion: "v1"` returns both the quote and the transaction. Read the price from that response, and sign only after the user confirms. [Trading Data Model](https://pond.dflow.net/spot/trading/trade-api-data-model) lists every request parameter and response field.
   - **Slippage:** omit `slippageBps` for auto. To set a custom `slippageBps` or a `priceImpactTolerancePct` limit, read [Slippage and Price Impact](https://pond.dflow.net/spot/trading/slippage-and-price-impact).
   - **Priority fee:** to set `prioritizationFeeLamports`, read [Priority Fees](https://pond.dflow.net/spot/trading/priority-fees).
   - **Wallet without v1 support:** read the [v1 Migration Guide](https://pond.dflow.net/spot/trading/v1-migration-guide).
4. **Sign.** Decode the transaction and sign it with the user's key.
5. **Send.** Submit through your own RPC.
6. **Confirm.** Poll `getSignatureStatuses` until `confirmed` or `finalized`, or until the block height passes `lastValidBlockHeight`. Once a signature has been submitted, request a new order for the same trade only after that signature is confirmed, failed, or past `lastValidBlockHeight`, because a late landing and a second order would both execute.

## Example

The same flow with more detail is in [Send Crypto Orders](https://pond.dflow.net/spot/recipes/quickstart).

```typescript
import {
  createKeyPairSignerFromBytes, createSolanaRpc, getBase58Encoder, getBase64Encoder,
  getBase64EncodedWireTransaction, getTransactionDecoder, signTransaction,
} from "@solana/kit";

// Defaults to the developer endpoint: no API key, lower rate limits.
// Production apps set DFLOW_TRADE_API_URL=https://quote-api.dflow.net and DFLOW_API_KEY for higher rate limits.
const API = process.env.DFLOW_TRADE_API_URL ?? "https://dev-quote-api.dflow.net";
const headers: HeadersInit = {};
if (process.env.DFLOW_API_KEY) headers["x-api-key"] = process.env.DFLOW_API_KEY;

// The public RPC is rate-limited. Production apps set SOLANA_RPC_URL to a dedicated RPC.
const rpc = createSolanaRpc(process.env.SOLANA_RPC_URL ?? "https://api.mainnet-beta.solana.com");

// Keep the private key in the environment, never inline.
const signer = await createKeyPairSignerFromBytes(
  getBase58Encoder().encode(process.env.SOLANA_PRIVATE_KEY ?? "")
);

const params = new URLSearchParams({
  inputMint: "So11111111111111111111111111111111111111112",   // SOL
  outputMint: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v", // USDC
  amount: "1000000", // 0.001 SOL, atomic units
  userPublicKey: signer.address,
  transactionVersion: "v1",
});
const order = await fetch(`${API}/order?${params}`, { headers }).then((r) => r.json());

const tx = getTransactionDecoder().decode(getBase64Encoder().encode(order.transaction));
const signed = await signTransaction([signer.keyPair], tx);
const signature = await rpc
  .sendTransaction(getBase64EncodedWireTransaction(signed), { encoding: "base64" })
  .send();

let status = null;
while (status?.confirmationStatus !== "confirmed" && status?.confirmationStatus !== "finalized") {
  if ((await rpc.getBlockHeight().send()) > BigInt(order.lastValidBlockHeight)) {
    throw new Error("Transaction expired before it landed");
  }
  await new Promise((resolve) => setTimeout(resolve, 1000));
  [status] = (await rpc.getSignatureStatuses([signature]).send()).value;
}
```

## Rules

- Call `/order` without `userPublicKey` only when the wallet is not known yet, such as a UI that shows a price before the user connects. The response has the price fields and no transaction. Once the wallet is known, make the single call above.
- The Trading API accepts base58 mints only. Resolve any symbol to a mint in the app.
- The Trading API sets no CORS headers. Call `/order` from a server so the API key stays there. A browser app calls its own backend route.
- Set a nonzero `platformFeeBps` only when the swap collects the fee. `feeAccount` is required and must be an existing token account (ATA) for the fee mint, which is the input or output mint per `platformFeeMode`. Collecting in more than one token takes one fee account per token. The [Platform Fees](https://pond.dflow.net/spot/trading/platform-fees) page has the parameters and a runnable example.

## Reference

Query the DFlow docs MCP (`https://pond.dflow.net/mcp`) for anything not covered here. Without it, start from [llms.txt](https://pond.dflow.net/llms.txt).
