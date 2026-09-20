---
name: postalform-com-machine-order-x402-mpp
description: Validate, preview, get owner approval, then pay for and track a PostalForm print-and-mail order through the x402 or MPP HTTP 402 flow, with the request_id idempotency rule the contract enforces.
api: openapi/postalform-com-machine-payments-openapi.json
operations:
  - validateMachineOrder
  - createMachineOrder
  - getMachineOrder
  - validateMppMachineOrder
  - createMppMachineOrder
  - getMppMachineOrder
method: generated
generated: '2026-09-19'
grounding: >-
  All six operationIds exist verbatim in openapi/postalform-com-machine-payments-openapi.json (info.title
  "PostalForm Machine Payments API", servers[0] https://postalform.com). Headers, status codes, error codes and
  the idempotency rule are quoted from that contract's info.description / x-agent-guidance and from
  https://postalform.com/agents and https://postalform.com/mpp.md. Nothing here was invented.
---

# Pay for real mail as an agent (x402 or MPP)

Base URL `https://postalform.com`. Everything is `application/json`. No API key exists for this surface — an
HTTP 402 payment challenge is the gate. Only use this path when the owner has explicitly authorized autonomous
spend and the runtime enforces a spend cap outside the prompt (the provider's own guidance).

## 0. Decide the family once

- **x402** (USDC on Base, `eip155:8453`, facilitator `api.cdp.coinbase.com`): `validateMachineOrder` → `createMachineOrder` → `getMachineOrder`. Challenge in `PAYMENT-REQUIRED`, retry with `PAYMENT-SIGNATURE`, success carries `PAYMENT-RESPONSE`.
- **MPP** (Stripe Machine Payments Protocol — Tempo, Stripe shared payment token `spt_...`, or card-MPP): `validateMppMachineOrder` → `createMppMachineOrder` → `getMppMachineOrder`. Challenge in one or more `WWW-Authenticate: Payment ...` headers, retry with `Authorization: Payment ...`, success carries `Payment-Receipt`.

Price range on both create operations is stated in `x-payment-info`: `minPrice 3.40`, `maxPrice 200.00` (USD).

## 1. Build the body (`MachineOrderRequest`)

Required: `request_id` (UUID — **the idempotency key**), `buyer_name`, `buyer_email` (set as the Stripe receipt email), `sender_name`, and a recipient (or a `bulk` object). Exactly ONE document source: `pdf` (prefer `{ "upload_token": "..." }`; also `{ download_url, file_id }`, a `data:application/pdf;base64` URL, or an allowlisted HTTPS URL), `letter` (string or `{ title, body, signature, format: text|html|markdown|rtf }`), or `form` (from `listMachineForms` → `getMachineFormSchema`).

For each address party choose exactly one strategy: `*_address_type: "Address"` with `*_address_id` + `*_address_text` (Loqate), or `*_address_type: "Manual"` with `*_address_manual` `{ line1, line2?, city, state?, zip, countryCode? }`. Countries: `US, CA, AT, BE, CH, DE, ES, FR, GB, IN, LU, NL` (default US).

Options: `double_sided` (default true), `color` (default false), `mail_class` `standard|priority|express`, `certified`, `certified_return_receipt`, `return_receipt_format`, `restricted_delivery`, `signature_required`, `mailpiece_type` `letter|postcard` (+ `postcard_size` `4x6|6x9|11x6`).

## 2. Validate first — always (no payment side effects)

`POST /api/machine/orders/validate` (`validateMachineOrder`) or `POST /api/machine/mpp/orders/validate` (`validateMppMachineOrder`) with the full body.

- **200** `MachineOrderValidationResponse`: `request_id`, `request_hash`, `order_id`, `status`, `quote` (and `bulk.recipient_count` for campaigns). Show the quote to the owner.
- **422** `invalid_request`: fix every entry in `errors[]` (`path`, `code`, `message`, optional `hint`, `fix_examples`). Nothing was created.
- **409** `request_id_mismatch`: this `request_id` was already used with a different body. Use the original body or a fresh UUID.

## 3. Create — expect 402

`POST /api/machine/orders` (`createMachineOrder`) or `POST /api/machine/mpp/orders` (`createMppMachineOrder`) with the **identical** body and the **same** `request_id`.

- **402** — the normal first answer. x402: body `X402PaymentRequiredResponse` (`status: payment_required`, `order_id`, `request_id`, optional `preview_url`) + `PAYMENT-REQUIRED` header. MPP: `application/problem+json` body `MppPaymentRequiredResponse` (`code: payment_required`, `supported_methods` ⊆ `tempo|stripe_spt|card`, `next_step`, optional `preview_url`, `price_breakdown`) + `WWW-Authenticate: Payment` header(s).
- If `preview_url` is present, show the owner the signed short-lived PDF preview **before** paying.
- Keep the returned `order_id` — it may differ from your `request_id` when PostalForm reuses a recent unpaid draft.

## 4. Get owner approval, then pay and retry the SAME request

Answer exactly one challenge with the wallet or MPP client, then repeat the create call with the byte-identical body and the same `request_id`, adding `PAYMENT-SIGNATURE` (x402) or `Authorization: Payment ...` (MPP). Do **not** create a second order after the 402. Do not send raw card numbers.

- **202** `MachineOrder` with `status: settled_pending_webhook` — payment accepted, Stripe still finalizing. Poll; **do not pay again**.
- **200** `MachineOrder` with `payment_status: paid`.
- **402 again** — you changed the body or `request_id`, chose the wrong challenge, or sent a bare `spt_...` instead of a serialized credential.
- **500** `payment_error` — verification failed; check status before retrying.

## 5. Track

`GET /api/machine/orders/{id}` (`getMachineOrder`) or `GET /api/machine/mpp/orders/{id}` (`getMppMachineOrder`). `{id}` accepts the canonical `order_id` or any aliased `request_id`. Read `payment_status`, `status`, `is_paid`, `tracking_number` / `tracking_url` (Certified Mail only), and the `err_*` electronic-return-receipt fields. `404` `Order not found.` for unknown ids. A tracking number alone does not prove mailing or delivery.

## 6. Second mailing?

Generate a **fresh** `request_id`. PostalForm only collapses recent *unpaid* duplicate drafts; a paid order with the same body under the same `request_id` is a mismatch, not a resend.

## Reversal

No cancel or refund operation exists on this contract. Before printing or carrier handoff, email support@postalform.com — cancellation is at PostalForm's discretion; Express orders enter production immediately after payment and cannot be canceled. For a machine payment the provider states the Stripe PaymentIntent can be refunded and USDC returned to the payer wallet. After carrier acceptance nothing can be recalled.
