---
name: zero-hash-payout
description: Send a zerohash payout with correct idempotency — link an external account, validate in dry-run mode, then submit the payout under a stable Idempotency-Key and reconcile the settlement.
api: zerohash API
base_url: https://api.zerohash.com
spec: openapi/zero-hash-api-openapi.yml
operations:
  - POST /payments/external_accounts
  - GET /payments/external_accounts
  - POST /payments/external_accounts/{external_account_id}/close
  - POST /payouts
  - GET /payouts
  - GET /payments/status
  - GET /payments/settlements
  - GET /payments/history
  - POST /payments/{transaction_id}/retry
generated: '2026-08-05'
method: generated
source: openapi/zero-hash-api-openapi.yml, https://docs.zerohash.com/docs/new-payouts-api-integration-guide
---

# Send a zerohash payout

`POST /payouts` is the single-call payout added in the 2026-06-24 release. It is the one operation
in the zerohash API with a first-class `Idempotency-Key` header, and using it correctly is the
difference between a retried request and a double payment.

## Steps

1. **Link the destination.** `POST /payments/external_accounts` registers the bank or wallet account.
   `GET /payments/external_accounts` lists what is already linked;
   `POST /payments/external_accounts/{external_account_id}/close` retires one.

2. **Dry-run first.** `POST /payouts` accepts a `validate` mode. In validate mode the
   `Idempotency-Key` header is ignored and nothing moves — use it to surface schema, limit and
   destination problems before committing.

3. **Submit for real.** Call `POST /payouts` with `validate` omitted or `false`. The
   `Idempotency-Key` header is **required** in this mode.

   Generate the key once per logical payout — derive it from your own payout record ID, never from
   a timestamp or a fresh UUID per attempt. zerohash stores a SHA-256 hash of the canonicalized JSON
   body alongside the key: replaying the **same key with a different body** returns HTTP `400`. That
   is a feature — it catches an amount or destination that changed between attempts.

4. **Reconcile.** `GET /payouts` accepts the `idempotency_key` as a lookup parameter, so after any
   ambiguous failure you can ask "did this payout land?" without guessing. `GET /payments/status`,
   `GET /payments/settlements` and `GET /payments/history` cover the settlement lifecycle, and
   `POST /payments/{transaction_id}/retry` re-drives a failed payment.

## Retry discipline

- `503` → safe to retry directly.
- `502` / `504` → the state is indeterminate. Retry **with the same `Idempotency-Key`**, or look the
  payout up by that key first.
- `400` after a retry with the same key means the body changed. Stop and investigate; do not force
  a new key to get past it.
- `4xx` other than the above are deterministic — do not retry.

Back off exponentially from 100–500ms and reuse `X-Request-Id` across retries of the same attempt.

## Related surfaces

- `payment_status_changed` and `external_account_status_changed` webhooks carry the state
  transitions — see `asyncapi/zero-hash-webhooks.yml`. De-duplicate on `x-zh-hook-notification-id`
  and order by the payload `timestamp`.
- Internal movement between zerohash accounts is `POST /transfers`, which uses a different mechanism:
  a `client_transfer_id` body field that must be unique platform-wide for a **rolling 72 hours**.
- `POST /payments/rfq` then `POST /payments/execute` is the quoted-payment path when FX or an asset
  conversion is involved.

## Notes

- The published OpenAPI declares **no operationIds**; operations are addressed by method + path.
- `POST /payouts` is the only operation in the spec that takes an `Idempotency-Key` header. Do not
  assume it works elsewhere.
