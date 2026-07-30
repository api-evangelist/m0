---
name: Quote and settle a cross-chain stablecoin swap on M0
description: >-
  Use the M0 Orchestration (liquidity) REST API to discover supported assets,
  get a quote for moving/converting between M0 extensions and other stablecoins,
  build an EIP-2612 permit for gasless intent-based settlement, and track the
  resulting cross-chain order.
api: openapi/m0-orchestration-openapi-original.json
base_url: https://gateway.m0.xyz/v1/orchestration
auth: api-key header `x-api-key`
operations:
- quote_getSupportedAssets
- quote_quote
- permit_permitQuote
- permit_permitBuild
- order_getOrder
- order_getOrders
- order_cancelOrder
generated: '2026-07-20'
method: generated
source: openapi/m0-orchestration-openapi-original.json
---

# Quote and settle a cross-chain stablecoin swap on M0

All requests go to `https://gateway.m0.xyz/v1/orchestration` (use the
`gateway.stage.m0.xyz` host to work against stage). Send your key in the
`x-api-key` header on every call. Credentials are issued by the M0 team.

## Steps

1. **List supported assets** — call `quote_getSupportedAssets`
   (`GET /supported-assets`) to get the tokens you can use as source or
   destination, with their `chain`, `runtime`, `address`, `decimals`, and
   `symbol`.

2. **Get a quote** — call `quote_quote` (`POST /quote`) with the asset route and
   amount. The response returns one or more quotes including signable
   `payloads` and `estFillTime`.

3. **Quote the permit** — call `permit_permitQuote` (`POST /permit/quote`) to
   price the intent-based permit path for the chosen route.

4. **Build the permit** — call `permit_permitBuild` (`POST /permit/build`). The
   response contains EIP-2612 typed data (`Eip2612TypedData` / `Eip712Domain`)
   to sign client-side; this is the gasless, replay-protected settlement path
   (per-order `nonce`), which is why the API has no HTTP idempotency key.

5. **Track the order** — call `order_getOrder`
   (`GET /orders/{originChain}/{orderId}`) for full on-chain state plus indexer
   enrichment, or `order_getOrders` (`GET /orders`) to list with `sender`,
   `status`, chain-id filters and `limit`/`offset` pagination
   (`total`/`limit`/`offset` in the response).

6. **Cancel if needed** — call `order_cancelOrder`
   (`POST /orders/{originChain}/{orderId}/cancel`) to build a cancellation
   transaction. This returns `409 OrderNotCancellable` if the order is already
   `COMPLETED` or `CANCELLED`.

## Error handling

Every error returns the custom `ErrorBody` envelope
`{ code, message, requestId }`. **Branch on `code`** (the stable `ErrorCode`
enum — e.g. `NoQuotesAvailable`, `OrderNotFound`, `OrderNotCancellable`,
`PermitExpired`, `InvalidPermitSignature`), not on the HTTP status, since
multiple errors can share a status. Log `requestId` and include it in support
tickets. See `errors/m0-error-codes.yml` and `conventions/m0-conventions.yml`.
