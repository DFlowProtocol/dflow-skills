---
name: dflow-proof-kyc
description: Integrate DFlow Proof — a Solana wallet identity-verification primitive (Stripe Identity under the hood) — to gate your own app's features behind KYC. Use when the user asks "how do I KYC a wallet?", "check if a wallet is verified", "add KYC to my DeFi app", "redirect to dflow.net/proof", or "gate a feature by jurisdiction or identity". Do NOT use for geoblocking (separate concern), for age gating (Proof doesn't currently verify age), or for spot swaps (no KYC required).
---

# DFlow Proof

Proof is DFlow's identity-verification primitive for Solana wallets. Stripe Identity verifies the person once; Proof links that verified identity to one or more wallet addresses. Builders query a public endpoint to check status.

Use Proof to self-gate your own app — any product that needs identity-based KYC (jurisdiction-gated DeFi, regulated token sales, identity-attested access, etc.) can use Proof as a ready-made primitive.

## Prerequisites

- **DFlow docs MCP** (`https://pond.dflow.net/mcp`) — install per the [repo README](../../README.md#recommended-install-the-dflow-docs-mcp). This skill is the recipe; the MCP is the reference. Look up the deep-link signing code, the full parameter list, and user-journey diagrams via `search_d_flow` / `query_docs_filesystem_d_flow` — don't guess.

## Self-gating your own app

Use this when you need KYC for *your own* app.

### Check verification status

`GET https://proof.dflow.net/verify/{address}` → `{ "verified": boolean }`. **Public, no auth.** Use it to gate features, to decide whether to show a "Verify me" CTA, or to short-circuit a restricted action.

### Redirect the user to verify (deep link)

If the wallet isn't verified, redirect the user to Proof's hosted flow. The deep link carries a signed ownership proof so Proof can link the wallet to the verified identity automatically:

- URL: `https://dflow.net/proof?wallet=<addr>&signature=<sig>&timestamp=<ms>&redirect_uri=<url>`
- Optional: `email`, `projectId`.
- Signature: user signs the exact message `Proof KYC verification: {timestamp}` (Unix ms, 13 digits) with their wallet; base58-encode the bytes.
- Full signing snippet and parameter table → docs MCP, or read directly: [`/proof/partner-integration`](https://pond.dflow.net/proof/partner-integration).

### Handle the return

User lands back on your `redirect_uri`. Re-query `/verify/{address}` to confirm status. If `verified: true`, proceed; otherwise, surface an appropriate "verification pending / failed" message.

### Embedded wallets (Privy, Turnkey, …)

Proof leans on exactly one wallet capability: signing the ownership message. Any wallet that exposes

```ts
signMessage: (message: Uint8Array) => Promise<Uint8Array>
```

drives the deep-link flow unchanged — embedded-wallet SDKs (Privy, Turnkey, etc.) expose this the same as Phantom/Solflare, so there's **no separate Proof path for embedded wallets**.

- **Status checks need no wallet.** `GET /verify/{address}` is public and takes only the address, so the gate/CTA logic is identical no matter how the wallet is custodied.
- **Generating the ownership signature** is the only place the wallet is involved. Build `Proof KYC verification: {timestamp}` (Unix ms, 13 digits), UTF-8-encode, sign the **raw bytes** with the provider's signer, then **base58**-encode the result and pass it as `signature` on the deep link:

```ts
import bs58 from "bs58";
const timestamp = Date.now();                                   // Unix ms, 13 digits
const messageBytes = new TextEncoder().encode(`Proof KYC verification: ${timestamp}`);
const signatureBytes = await wallet.signMessage(messageBytes);  // Privy / Turnkey embedded signer
const signature = bs58.encode(signatureBytes);                  // base58 — NOT hex, NOT JSON.stringify
// → build https://dflow.net/proof?wallet=<pubkey>&signature=<signature>&timestamp=<timestamp>&redirect_uri=<url>
```

- **The one prerequisite to check:** the provider must expose **raw message signing** over bytes. If the SDK only signs transactions (no `signMessage`), it can't produce the ownership proof — confirm that before wiring Proof to an embedded-wallet provider.

## What to ASK the user (and what NOT to ask)

**Ask if missing:**

1. **Wallet pubkey** — the address to verify / check.
2. **App's callback URL** (`redirect_uri`) — where Proof sends the user after verification.
3. **Web or native mobile** — changes the redirect_uri guidance (universal / app links for mobile; see Gotchas).

**Do NOT ask about:**

- **API key** — `/verify/{address}` is public, no auth. Proof itself has no API key concept.

## Gotchas (the docs MCP won't volunteer these)

- **Proof is not required for DFlow spot swaps.** Don't state "all DFlow trades need KYC" — they don't.
- **Proof doesn't verify age.** Stripe Identity captures name, address, email, and government-issued ID, but Proof does **not** currently check or expose date-of-birth. Don't use Proof for age gating — you won't get what you need.
- **Enforced on both dev and prod.** Many agents assume dev is unprotected; it isn't.
- **Redirect URI scheme restrictions.** Proof only redirects to `https:`, `chrome-extension:`, and `moz-extension:` URLs. Custom schemes (`myapp://callback`) **fail silently** — no redirect, no error. Native mobile → universal links (iOS) / app links (Android), which are `https:` URLs that deep-link into the app.
- **The public endpoint is booleanized.** `/verify/{address}` returns `{ verified: true | false }`. There's no `pending` / `failed` / `unverified` distinction — everything non-verified collapses to `false`. If you need those states for UX, infer them from your own session state (did the user come back from Proof?), not from the public check.
- **Cache `true`, not `false`.** Once verified, a wallet stays verified; caching avoids repeated checks. But unverified is volatile — never cache it, because it flips the moment the user completes the flow.
- **For self-gating your own app, verify server-side.** If your backend is the thing enforcing a KYC-gated feature, don't trust a client's cached status. Re-query `/verify/{address}` from your backend before unlocking the gated action.
- **Embedded wallets work.** Privy, Turnkey, etc. — the only requirement is raw `signMessage` over bytes. See *Embedded wallets* above for the signing snippet; there's no separate Proof path for them.
- **One verified identity → unlimited wallets.** No cap. A user who verified on wallet A can link wallet B, C, D, and onward without re-doing ID + liveness — just a fresh ownership signature from each new wallet.
- **Free + Stripe Identity under the hood.** No fee to builders or users. Users complete Stripe's document + liveness flow.
- **Proof is not geoblocking.** KYC ≠ jurisdictional restriction; they are separate concerns.

## When something doesn't fit

Defer to the docs MCP for full reference — specifically:

- [`/proof/introduction`](https://pond.dflow.net/proof/introduction) — top-level Proof overview: when to use it, the identity-graph model, and what gets verified.
- [`/proof/partner-integration`](https://pond.dflow.net/proof/partner-integration) — deep-link code (signature generation, URL building), caching sample, handling edge cases (signature expiration, user cancellation, network errors), security guidance, and redirect-scheme debugging (`#deep-link-parameters`).
- [`/proof/user-journeys`](https://pond.dflow.net/proof/user-journeys) — diagrams for new-direct, new-from-partner, and returning-user flows.
- [`/resources/proof-api/verify-address`](https://pond.dflow.net/resources/proof-api/verify-address) — the single public endpoint's reference.

## Sibling skills

- `dflow-spot-trading` — Solana token swaps; no Proof required, ever.
- `dflow-market-data` — live spot market-data streams; read-only, no Proof required.
