---
name: zero-hash-account-funding
description: Fund a customer account on zerohash with stablecoins or fiat — generate a deposit address, quote and execute the fund conversion, and reconcile the deposit through webhooks.
api: zerohash API
base_url: https://api.zerohash.com
spec: openapi/zero-hash-api-openapi.yml
operations:
  - POST /deposits/digital_asset_addresses
  - GET /deposits/digital_asset_addresses
  - PATCH /deposits/digital_asset_addresses
  - GET /deposits/fiat_accounts
  - GET /deposits
  - GET /deposits/crypto
  - GET /deposits/crypto/{deposit_id}
  - POST /deposits/fund
  - POST /fund/rfq
  - POST /fund/deposit
  - GET /fund/transactions
  - GET /accounts
  - GET /accounts/{account_id}/movements
generated: '2026-08-05'
method: generated
source: openapi/zero-hash-api-openapi.yml, https://docs.zerohash.com/docs/fund-integration-guide-api
---

# Fund a zerohash account

Account Funding lets a customer top up in USDC, USDT, PYUSD or RLUSD across 15+ chains and land in
the account currency your platform runs on. It is a deposit rail plus a conversion, not just a
transfer.

## Steps

1. **Generate the deposit address.** `POST /deposits/digital_asset_addresses` creates a
   participant-scoped address for the asset and network.
   `GET /deposits/digital_asset_addresses` lists existing ones and
   `PATCH /deposits/digital_asset_addresses` updates one. Show the customer the address **and** the
   network; a correct address on the wrong chain is an unrecoverable send.

2. **Watch for the inbound deposit.** `GET /deposits/crypto` and
   `GET /deposits/crypto/{deposit_id}` read status. Prefer the `deposit.status_changed` and
   `deposit_fund_complete` webhooks over polling — see `asyncapi/zero-hash-webhooks.yml`.

3. **Quote and execute the conversion.** `POST /fund/rfq` returns the quote for converting the
   deposited asset into the destination currency; `POST /fund/deposit` (or `POST /deposits/fund`)
   commits it. `GET /fund/transactions` is the funding-transaction ledger.

4. **Reconcile.** `GET /accounts` for balances, `GET /accounts/{account_id}/movements` for the
   ledger movements behind them, and `GET /deposits` for the full deposit history. The
   `account_balance.changed` webhook mirrors the balance transition.

## Webhook handling

Deliveries are retried up to 4 times at 250ms, then 3 more with exponential backoff, then dropped.
So the consumer, not zerohash, owns exactly-once:

- de-duplicate on `x-zh-hook-notification-id`
- branch on `x-zh-hook-payload-type`
- **order by the payload `timestamp`, never by arrival order**
- return quickly and process asynchronously, or you will burn the retry budget

Endpoints are not self-service: you supply production and certification URLs to zerohash and they
are configured within 2 business days.

## Retry discipline

`GET` is always safe. For the funding POSTs, a `503` is safe to retry; on `502`/`504` read the state
back through `GET /fund/transactions` or `GET /deposits/crypto/{deposit_id}` before resubmitting —
neither `POST /fund/rfq` nor `POST /fund/deposit` accepts an `Idempotency-Key`.

## Notes

- The published OpenAPI declares **no operationIds**; operations are addressed by method + path.
- The embedded route to the same flow is the Fund SDK module
  (`@zerohash-sdk/fund-js` / `@zerohash-sdk/fund-react`), authorized by
  `POST /client_auth_token` and revoked by `POST /revoke_auth_token`. See
  `components/zero-hash-components.yml`.
