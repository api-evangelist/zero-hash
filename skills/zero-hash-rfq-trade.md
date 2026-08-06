---
name: zero-hash-rfq-trade
description: Buy or sell crypto on zerohash through the RFQ liquidity flow — request a quote, execute it inside its validity window, and reconcile the resulting trade and balances.
api: zerohash API
base_url: https://api.zerohash.com
spec: openapi/zero-hash-api-openapi.yml
operations:
  - GET /assets
  - GET /index
  - GET /market_data/symbol_data
  - POST /liquidity/rfq
  - GET /liquidity/rfq
  - PATCH /liquidity/rfq
  - POST /liquidity/execute
  - GET /trades
  - GET /trades/{trade_id}
  - GET /accounts
  - GET /positions
generated: '2026-08-05'
method: generated
source: openapi/zero-hash-api-openapi.yml, https://docs.zerohash.com/docs/error-handling
---

# Trade on zerohash with RFQ liquidity

RFQ is a two-call flow: you ask for a price, then you take it. The quote has a short life, so the
two calls belong together in one code path — never behind a user confirmation screen that can sit
idle.

## Steps

1. **Confirm the instrument.** `GET /assets` lists supported assets; `GET /market_data/symbol_data`
   and `GET /index` give reference pricing. Do not hard-code a symbol list — assets are added and
   delisted through the release notes.

2. **Request the quote.** `POST /liquidity/rfq` with the participant, side, symbol and quantity or
   notional. The response carries a `quote_id` and its expiry. `GET /liquidity/rfq` reads a quote
   back; `PATCH /liquidity/rfq` updates one.

3. **Execute inside the window.** `POST /liquidity/execute` with the `quote_id`. This is the call
   that moves money.

4. **Reconcile.** `GET /trades/{trade_id}` and `GET /trades` confirm settlement; `GET /accounts`
   and `GET /positions` confirm the resulting balances. The `trade.status_changed` and
   `account_balance.changed` webhooks carry the same transitions without polling.

## Retry discipline — this is the part that matters

`POST /liquidity/execute` is explicitly documented as **safe to retry** on `502`, `503` and `504`.
On a `404`, do **not** re-execute: use the `quote_id` to check whether the trade already executed.
This is the provider's own guidance in the error-handling guide, and it is the only supported
protection here — the execute call takes no `Idempotency-Key` header.

`GET` calls are always safe to retry. Use exponential backoff starting at 100–500ms. Send an
`X-Request-Id` and reuse the same value across retries of the same logical operation.

## Alternatives

- **CLOB.** For order-book execution rather than RFQ, use `POST /orders/v1/insert_order`,
  `POST /orders/v1/cancel_order`, `POST /orders/v1/search_orders` and
  `POST /orders/v1/search_executions`, or connect to the FIX 5.0 gateway.
- **Convert and withdraw.** `POST /convert_withdraw/rfq` then `POST /convert_withdraw/execute`
  combines the trade and the payout; it carries the same retry semantics as `/liquidity/execute`.
- **Prices feed.** The private WebSocket `prices` topic at `wss://ws.zerohash.com` streams RFQ
  bid/ask levels. It covers RFQ products only, not CLOB markets.

## Notes

- The published OpenAPI declares **no operationIds**; operations are addressed by method + path.
- No rate-limit contract is published. `429` is described in the error-handling guide but is not in
  any operation's responses and there are no `RateLimit-*` headers, so build backoff defensively.
