---
name: zero-hash-onboard-participant
description: Onboard an individual or entity participant onto zerohash, check jurisdiction eligibility first, submit KYC documents, and poll until the participant is approved and has an account.
api: zerohash API
base_url: https://api.zerohash.com
spec: openapi/zero-hash-api-openapi.yml
operations:
  - GET /jurisdiction/countries
  - GET /jurisdiction/subdivisions
  - POST /jurisdiction/evaluate-onboarding
  - POST /participants/customers/new
  - POST /participants/entity/new
  - POST /participants/documents
  - POST /participants/entity/documents
  - GET /participant/{participant_code}/kyc_status
  - GET /participant/{participant_code}/basic_info
  - GET /participant/{participant_code}/full_info
  - GET /participant/{participant_code}/sanction_screening_info
  - POST /accounts
  - GET /accounts
generated: '2026-08-05'
method: generated
source: openapi/zero-hash-api-openapi.yml, conventions/zero-hash-conventions.yml
---

# Onboard a zerohash participant

A participant is the KYC/AML subject on the zerohash platform. Nothing else — no trade, deposit,
withdrawal or payout — can happen until a participant exists and is approved. Its identifier,
`participant_code`, is always **6 uppercase alphanumeric characters**.

## Before any call

Every request to `https://api.zerohash.com` (or `https://api.cert.zerohash.com` in the certification
environment) carries four headers:

- `X-SCX-API-KEY` — public key
- `X-SCX-PASSPHRASE` — passphrase set when the key was created
- `X-SCX-TIMESTAMP` — Unix seconds, and it must be within **60 seconds** of server time
- `X-SCX-SIGNED` — base64 HMAC-SHA256 over `timestamp + method + route + body`

Also send `X-Request-Id` — the server echoes it back, and support asks for it first. The source IP
must already be on the environment's allowlist; each environment has its own list.

## Steps

1. **Check the jurisdiction before collecting anything.** Call `POST /jurisdiction/evaluate-onboarding`
   with the prospective participant's country and subdivision. `GET /jurisdiction/countries` and
   `GET /jurisdiction/subdivisions` give the accepted values. Failing here saves the customer from
   handing over identity documents for a jurisdiction that will be rejected.

2. **Create the participant.**
   - Individual: `POST /participants/customers/new`
   - Entity: `POST /participants/entity/new`
   - Beneficiary-only: `POST /participants/beneficiaries/new`

   The response carries the `participant_code`. Persist it — it is the key for every later call.

3. **Attach documents when required.** `POST /participants/documents` for individuals,
   `POST /participants/entity/documents` for entities. Read back with
   `GET /participant/{participant_code}/full_info/document_metadata` and
   `GET /participant/{participant_code}/full_info/documents/{document_id}/download`.

4. **Poll KYC.** `GET /participant/{participant_code}/kyc_status` is the authoritative check.
   `GET /participant/{participant_code}/sanction_screening_info` reports the screening outcome and
   `GET /participant/{participant_code}/status_reason` explains a non-approved state. Prefer the
   `participant_status_changed` and `participant_updated` webhooks over polling — see
   `asyncapi/zero-hash-webhooks.yml`. Sequence webhook deliveries by the payload `timestamp`, never
   by arrival order, and de-duplicate on `x-zh-hook-notification-id`.

5. **Create the account.** `POST /accounts` creates the balance-bearing account; the response
   carries the ZRN (zerohash Resource Name) used by `GET /accounts/{zrn}/details`,
   `PATCH /accounts/{zrn}`, and the lock/unlock operations. Confirm with `GET /accounts`.

6. **Read limits.** `GET /participant/{participant_code}/limits` returns the transaction limits that
   will govern everything the participant does next.

## Error handling

Responses are `application/json` with a human-readable `error` string — not RFC 9457 problem+json.
Do not retry a `400`, `403`, `404`, `409` or `422`; they are deterministic. Retry `500` and `503`
with exponential backoff starting at 100–500ms. On `502`/`504` the state is indeterminate: read the
participant back with `GET /participants` or `GET /participants/{email}` before resubmitting, since
creation is not idempotent on these endpoints. See `errors/zero-hash-problem-types.yml`.

## Notes

- The published OpenAPI declares **no operationIds**, so operations are addressed by method + path.
- In the certification environment you can drive specific outcomes with the `X-Simulator-Scenario`
  header; it has no effect in production. See `sandbox/zero-hash-sandbox.yml`.
