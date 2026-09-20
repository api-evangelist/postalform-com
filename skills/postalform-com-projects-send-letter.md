---
name: postalform-com-projects-send-letter
description: Send a letter from server-side code through PostalForm Projects — test key first, upload intent, quote, idempotent order, then poll or receive signed webhooks.
api: openapi/postalform-com-projects-openapi.json
operations:
  - createUploadIntent
  - completeDocumentUpload
  - createLetterQuote
  - createLetter
  - getLetter
  - createWebhookEndpoint
  - listWebhookEvents
method: generated
generated: '2026-09-19'
grounding: >-
  All seven operationIds exist verbatim in openapi/postalform-com-projects-openapi.json (info.title "PostalForm
  Projects Public API", info.version 2026-05-06, servers[0] https://projects.postalform.com, securitySchemes
  bearerAuth). Key prefixes, the test-mode guarantees and the webhook signature header are quoted from
  https://postalform.com/developer-mail-api.md and https://projects.postalform.com/llm-context.txt. Nothing
  here was invented.
---

# Send a letter from code with PostalForm Projects

Base URL `https://projects.postalform.com/api/v1`. Auth: `Authorization: Bearer <key>` on every call. Two key
modes per workspace: `pf_test_...` (free, simulated fulfillment, never sends mail) and `pf_live_...` (prepaid
credits, real mail). Build against the test key until the whole flow passes.

## 1. Upload the PDF (two steps)

`POST /documents/upload-intent` (`createUploadIntent`) with `{ "content_type": "application/pdf", "byte_size": <bytes ≤ 26214400>, "page_count"?: <n> }` → `UploadIntent` with upload instructions and a `document_id`. Upload the bytes as instructed, then `POST /documents/{document_id}/complete` (`completeDocumentUpload`). Projects limits uploads to 25 MiB.

## 2. Quote — the price is final before you commit

`POST /letters/quotes` (`createLetterQuote`) with `{ "document_id", "mail_class"?: "usps_first_class" | "usps_priority" | "usps_express", "color"?: false, "double_sided"?: true, "certified"?: false, "certified_return_receipt"?, "return_receipt_format"?: "electronic" | "physical", "restricted_delivery"?, "signature_required"?, "standalone_address_page"? }` → `Quote` with `quote_id`, `price_cents`, `currency: usd`, `pricing_version`, `expires_at`. Priority/Express cannot be combined with certified or registered proof mail (spec). Re-quote after `expires_at`.

## 3. Create the order — Idempotency-Key is REQUIRED

`POST /letters` (`createLetter`) with header `Idempotency-Key: <stable value for this one intended mailing>` and body `{ "quote_id", "recipient": MailingAddress, "sender"?: MailingAddress, "metadata"?: {...} }`.

- **201** `Letter` — created.
- **200** `Letter` — idempotent replay: the same key with the same quote returned the existing order instead of creating another.

Reuse the same key on network retries; never mint a new one for the same mailing. `sender` is "strongly recommended for all live mail and required for some destinations and mailpiece options".

## 4. Follow fulfillment

Poll `GET /letters/{order_id}` (`getLetter`) → `Letter` with `status` ∈ `queued, document_preparing, document_prepared, submitted, accepted, in_transit, delivered, returned, submission_pending, failed, canceled`, `mode`, `price_cents`, `tracking_number` / `tracking_url` / `tracking_status` when available, and the `err_*` return-receipt fields. `submission_pending` means the submit job entered a physical-mail safety window where PostalForm will not blindly retry (duplicate-mail risk) — reconcile before acting.

Or register a webhook: `POST /webhook-endpoints` (`createWebhookEndpoint`) with an `https://` URL → `WebhookEndpointSecretResponse` whose `signing_secret` is returned **once**. Verify the `PostalForm-Signature` header on every delivery. Events: `postalform.letter.accepted | in_transit | delivered | returned | failed | canceled` (and the `postalform.postcard.*` twins); payload `{ id, type, data.object: Letter, mailpiece: { status, tracking_number, tracking_status } }`. Failed deliveries are retried; inspect them with `GET /webhook-events` (`listWebhookEvents`) and replay with `POST /webhook-events/{event_id}/replay`.

## Going live

Rotate to the live key (`POST /api-keys/rotate` returns the secret once), fund prepaid credits (`GET /credits/balance`, `POST /credits/checkout-session`, optional auto-refill), and keep the test-mode integration as your regression harness. Live orders reserve and capture the quoted amount from the credit ledger.

## Reversal

The contract exposes no cancel operation; `canceled` is a status you can observe and a webhook event you can receive. Before printing or carrier handoff, email support@postalform.com; after carrier acceptance an order cannot be recalled. Errors caused by PostalForm's own printing/processing are remedied by reprint or refund.
