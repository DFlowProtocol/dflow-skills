---
name: dflow-sponsored-swaps
description: Use when building gasless swaps on Solana through the DFlow Trading API, where a sponsor wallet pays transaction fees and token account creation costs so users trade without holding SOL. Applies to crypto and tokenized equity orders.
---

# DFlow Sponsored Swaps

Pass `sponsor` on `GET /order` and the sponsor wallet pays the transaction fee and token account creation costs. The user and the sponsor both sign the returned transaction.

This skill covers what sponsorship changes. For the rest of the flow, use `dflow-crypto-trading` or `dflow-equities-trading`.

## Ask First

Ask only what the request leaves open.

1. **Execution mode:** `sponsorExec`.
   - `true` (default): the sponsor executes the swap. The user transfers input tokens to the sponsor, the sponsor swaps with its own token accounts, then transfers the output to the user.
   - `false`: the user executes the swap from their own token accounts. The sponsor still pays fees and account creation.
2. **Where each key lives:** the sponsor key belongs on a server. See Architecture.

## Architecture

The sponsor's private key stays on a server and never reaches a browser or user device.

1. **Server:** requests `/order` with `sponsor`, `sponsorExec`, `userPublicKey`, and `transactionVersion: "v1"`. It keeps the order it received and sends `order.transaction` to the user's app.
2. **User's app:** the user signs in their wallet without sending, then returns the signed transaction to the server.
3. **Server:** checks that the returned transaction has the same message as the one it issued, adds the sponsor signature, sends it, and confirms.

The server co-signs only a transaction it issued. A sponsor that signs whatever transaction a client sends pays fees for anything the client builds.

For the equities path, the sponsor's signature goes in before the transaction is sent to the submit endpoint. See [Equity Order Execution](https://pond.dflow.net/equities/orders).

## Endpoints

- [`GET /order`](https://pond.dflow.net/resources/trading-api/order/order): pass `sponsor` and `sponsorExec`.
- [`GET /quote`](https://pond.dflow.net/resources/trading-api/imperative/quote): pass `sponsoredSwap=true` and `sponsorExec`. `/order` and `/swap` infer sponsorship from `sponsor`.
- [`POST /swap`](https://pond.dflow.net/resources/trading-api/imperative/swap): pass `sponsor` and `sponsorExec` in the body.

[Sponsored Swaps](https://pond.dflow.net/spot/trading/sponsored-swaps) lists every parameter, including `sponsorIntermediateCloseAuthority`.

## Example

Server-side functions for steps 1 and 3, then the user's app for step 2.

```typescript
import {
  createKeyPairSignerFromBytes, createSolanaRpc, getBase58Encoder, getBase64Encoder,
  getBase64EncodedWireTransaction, getTransactionDecoder, signTransaction,
} from "@solana/kit";

// Defaults to the developer endpoint. Production apps set DFLOW_TRADE_API_URL=https://quote-api.dflow.net and DFLOW_API_KEY.
const API = process.env.DFLOW_TRADE_API_URL ?? "https://dev-quote-api.dflow.net";
const headers: HeadersInit = {};
if (process.env.DFLOW_API_KEY) headers["x-api-key"] = process.env.DFLOW_API_KEY;

const rpc = createSolanaRpc(process.env.SOLANA_RPC_URL ?? "https://api.mainnet-beta.solana.com");

// Server only. Keep the sponsor key in the environment, never inline.
const sponsor = await createKeyPairSignerFromBytes(
  getBase58Encoder().encode(process.env.SPONSOR_PRIVATE_KEY ?? "")
);

const issued = new Map<string, { message: Uint8Array; lastValidBlockHeight: number }>();

// Step 1: request the order and keep what was issued.
export async function createSponsoredOrder(userPublicKey: string, inputMint: string, outputMint: string, amount: string) {
  const params = new URLSearchParams({
    inputMint, outputMint, amount, userPublicKey,
    sponsor: sponsor.address,
    sponsorExec: "false", // the user executes the swap
    transactionVersion: "v1",
  });
  const order = await fetch(`${API}/order?${params}`, { headers }).then((r) => r.json());
  const tx = getTransactionDecoder().decode(getBase64Encoder().encode(order.transaction));
  issued.set(userPublicKey, { message: new Uint8Array(tx.messageBytes), lastValidBlockHeight: order.lastValidBlockHeight });
  return order; // send order.transaction to the user's app to sign
}

// Step 3: verify, co-sign, send, confirm.
export async function sponsorAndSend(userPublicKey: string, userSignedBase64: string) {
  const record = issued.get(userPublicKey);
  if (!record) throw new Error("No issued order for this user");
  const tx = getTransactionDecoder().decode(getBase64Encoder().encode(userSignedBase64));
  const same = tx.messageBytes.length === record.message.length && tx.messageBytes.every((b, i) => b === record.message[i]);
  if (!same) throw new Error("Transaction does not match the issued order");

  const signed = await signTransaction([sponsor.keyPair], tx);
  const signature = await rpc
    .sendTransaction(getBase64EncodedWireTransaction(signed), { encoding: "base64" })
    .send();

  let status = null;
  while (status?.confirmationStatus !== "confirmed" && status?.confirmationStatus !== "finalized") {
    if ((await rpc.getBlockHeight().send()) > BigInt(record.lastValidBlockHeight)) {
      throw new Error("Transaction expired before it landed");
    }
    await new Promise((resolve) => setTimeout(resolve, 1000));
    [status] = (await rpc.getSignatureStatuses([signature]).send()).value;
  }
  issued.delete(userPublicKey);
  return signature;
}
```

Step 2, in the user's app. The routes are your own backend. The user signs with `partiallySignTransaction`, because `signTransaction` throws on a transaction that is not fully signed. A browser wallet signs through its own API instead of a keypair.

```typescript
import {
  getBase64Encoder, getBase64EncodedWireTransaction, getTransactionDecoder, partiallySignTransaction,
} from "@solana/kit";

export async function swapSponsored(userSigner, inputMint: string, outputMint: string, amount: string) {
  const post = (path: string, body: object) =>
    fetch(path, { method: "POST", headers: { "content-type": "application/json" }, body: JSON.stringify(body) }).then((r) => r.json());

  const order = await post("/api/sponsored-order", { userPublicKey: userSigner.address, inputMint, outputMint, amount });
  const tx = getTransactionDecoder().decode(getBase64Encoder().encode(order.transaction));
  const signed = await partiallySignTransaction([userSigner.keyPair], tx);

  const { signature } = await post("/api/sponsored-send", {
    userPublicKey: userSigner.address,
    transaction: getBase64EncodedWireTransaction(signed),
  });
  return signature;
}
```

## Rules

- The sponsor key never leaves the server, and the server calls `/order` so the API key stays there too.
- Both the user and the sponsor must sign. `sponsorExec` changes who owns the swap's token accounts, not who signs.
- Fund the sponsor wallet with SOL for fees and token account creation. A sponsored swap draws on it for every user.
- Once a signature has been submitted, request a new order for the same trade only after that signature is confirmed, failed, or past `lastValidBlockHeight`, because a late landing and a second order would both execute.

## Reference

Query the DFlow docs MCP (`https://pond.dflow.net/mcp`) for anything not covered here. Without it, start from [llms.txt](https://pond.dflow.net/llms.txt).
