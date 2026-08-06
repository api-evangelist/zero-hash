---
name: zero-hash-crypto-withdrawal
description: Withdraw crypto from zerohash safely — validate the destination address, price the network fee, create the withdrawal request, execute it, and reconcile on-chain status.
api: zerohash API
base_url: https://api.zerohash.com
spec: openapi/zero-hash-api-openapi.yml
operations:
  - GET /withdrawals/validate_address
  - GET /withdrawals/estimate_network_fee
  - GET /withdrawals/locked_network_fee
  - POST /withdrawals/requests
  - GET /withdrawals/requests
  - GET /withdrawals/requests/{id}
  - DELETE /withdrawals/requests/{id}
  - POST /withdrawals/execute
  - GET /withdrawals/digital_asset_addresses
  - GET /withdrawals/fiat_accounts
  - GET /accounts
generated: '2026-08-05'
method: generated
source: openapi/zero-hash-api-openapi.yml, https://docs.zerohash.com/docs/withdrawals-releasing-funds
---

# Withdraw crypto from zerohash

Withdrawals are request-then-execute, deliberately. The split exists so an approval step can sit
between them — a request can be cancelled, an executed withdrawal cannot be recalled.

## Steps

1. **Validate the destination.** `GET /withdrawals/validate_address` before anything else. An
   address that is well-formed for the wrong network is the most expensive mistake available here.
   Check `GET /withdrawals/digital_asset_addresses` for addresses already registered, and
   `GET /withdrawals/fiat_accounts` for the fiat equivalent. Memo/destination-tag chains need the
   memo carried too.

2. **Price the network fee.** `GET /withdrawals/estimate_network_fee` gives an estimate;
   `GET /withdrawals/locked_network_fee` returns a locked fee you can quote to the customer without
   drift.

3. **Confirm the balance.** `GET /accounts` — check that the account is neither locked
   (`POST /accounts/{zrn}/lock`) nor withdraw-locked (`POST /accounts/{zrn}/withdraw_lock`), since
   both block the release.

4. **Create the request.** `POST /withdrawals/requests`. Read back with
   `GET /withdrawals/requests/{id}`; `DELETE /withdrawals/requests/{id}` cancels one that has not
   been executed.

5. **Execute.** `POST /withdrawals/execute` releases the funds.

6. **Reconcile.** Poll `GET /withdrawals/requests/{id}`, or consume the `deposit.status_changed` and
   `account_balance.changed` webhooks. Blockchain confirmation is asynchronous — the API returning
   success means the transaction was broadcast, not that it is final.

## Retry discipline

`POST /withdrawals/execute` takes no `Idempotency-Key`. On a `502` or `504` the state is
indeterminate: **do not blind-retry.** Read the request back with `GET /withdrawals/requests/{id}`
and only re-execute if it is still un-executed. A `503` is safe to retry. `4xx` responses are
deterministic — fix the input.

## Alternate rails

- **Lightning:** `POST /deposits/bolt11` and `POST /deposits/bolt11/test_payment` on the deposit
  side; see the BOLT11 and LNURL guides.
- **UMA:** `GET /withdrawals/umalookup/{recipient}` then `POST /withdrawals/uma`.
- **Convert and withdraw:** `POST /convert_withdraw/rfq` then `POST /convert_withdraw/execute` when
  the withdrawal needs a conversion first. That execute call *is* documented as safe to retry on
  `502`/`503`/`504`, with a `404` meaning "look the quote up rather than resubmit".

## Notes

- The published OpenAPI declares **no operationIds**; operations are addressed by method + path.
- Test in CERT against public testnet faucets — zerohash does not supply test tokens. See
  `sandbox/zero-hash-sandbox.yml`.
