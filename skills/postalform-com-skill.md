---
title: PostalForm agent skill
description: Read this file before using PostalForm from an AI assistant, MCP client, coding agent, or autonomous runtime.
seotitle: PostalForm MCP agent skill
seo-description: Agent-ready setup instructions for using PostalForm MCP in ChatGPT, Gemini, Claude, Codex, and other MCP-capable clients to send real postal mail.
group: resources
indexable: true
nav: false
schema: webpage
eyebrow: Agent skill
published: 2026-05-30
updated: 2026-08-27
path: /skill
---
# PostalForm agent skill

Read this file before using PostalForm from an AI assistant, MCP client, coding agent, or autonomous runtime.

## Quick setup
- Remote MCP endpoint: `https://postalform.com/mcp`
- Transport: streamable HTTP
- Authentication: none required today for the hosted MCP endpoint
- A2A Agent Card: `https://postalform.com/.well-known/agent-card.json`
- MCP manifest: `https://postalform.com/.well-known/mcp.json`
- MCP Server Card: `https://postalform.com/.well-known/mcp/server-card.json`
- MCP Registry server manifest: `https://postalform.com/.well-known/mcp/server.json`
- OpenAPI: `https://postalform.com/openapi.json`
- Full developer docs: `https://postalform.com/developers`
- Autonomous payment guide: `https://postalform.com/agents`
- MPP-only endpoint guide: `https://postalform.com/mpp.md`
- MPP mail endpoint guide: `https://postalform.com/mpp-mail.md`
- MPP flower endpoint guide: `https://postalform.com/mpp-flowers.md`
- MPP shipping-label endpoint guide: `https://postalform.com/mpp-shipping-labels.md`
- Open Knowledge Format index: `https://postalform.com/okf/index.md`

If your client accepts a remote MCP URL, add `https://postalform.com/mcp` as a streamable HTTP MCP server named `postalform`.

## Works with
PostalForm is designed for MCP-capable clients and runtimes including ChatGPT custom connectors or OpenAI Responses API MCP tools, Gemini SDKs, Gemini CLI, Claude, Claude Code, Codex, Cursor, Windsurf, Cline, Replit, OpenClaw, Hermes, n8n, LangChain, and other clients that can connect to a remote streamable HTTP MCP endpoint.

For ChatGPT or OpenAI API flows, configure a remote MCP tool with `server_url` set to `https://postalform.com/mcp`. Require approval for tool calls that send document contents, addresses, or payment-related data.

For Gemini CLI, add PostalForm as an HTTP MCP server:

```bash
gemini mcp add --transport http postalform https://postalform.com/mcp
```

For Gemini SDK integrations, connect to the same MCP endpoint with an MCP client session and pass the session tools to the Gemini SDK's MCP/tool-calling integration.

## Tool choice
Use these tools to prepare an unpaid draft for buyer review:

- `postalform.create_letter_order_draft` for a plain letter that PostalForm renders into a printable PDF.
- `postalform.create_order_draft` for an uploaded PDF or a PDF file URL.
- `postalform.list_forms`, `postalform.get_form_schema`, and `postalform.create_form_order_draft` for supported form workflows.
- `postalform.search_addresses` when the user wants autocomplete-backed address validation.
- `postalform.get_order_status` after checkout starts or payment completes.

Use `postalform.create_machine_order` only when the owner explicitly authorized autonomous payment and the runtime has a spend limit. The first call returns an MPP or x402 payment challenge. Retry the same request with the payment credential after the payment client answers the challenge.

## Payment guidance
- After buyer approval, a client that obtains a compatible Stripe shared payment token can call `complete_checkout` with `checkout_session.id`, buyer details, and `payment_data` containing `provider: "stripe"` and the token. Otherwise present the returned `checkout_url`.
- Retry an interrupted checkout with the same session ID and token. An unverified or processing payment must not prompt another order or replacement payment. Check status and follow the returned messages.
- Clients that can answer MPP or x402 challenges can validate or create a machine order, show the preview and price, obtain approval, then answer the challenge. Verify the client's actual attachment and payment handoffs before relying on either payment path.
- Never send raw card details to PostalForm. Machine payment paths use MPP, x402, hosted checkout, or Stripe Shared Payment Tokens where applicable.

## Mailing rules
- PostalForm supports specific destination countries. Validate or quote the order before promising delivery.
- Certified/proof mail covers USPS Certified Mail for qualifying U.S. First Class mail, plus PinGen registered mail for supported European destinations (e.g. Switzerland, France, Belgium).
- Express, Fast, postcard, international, and provider-specific options vary by destination, size, address, and provider eligibility.
- PostalForm does not package or tender parcels. Its MPP-only shipping-label product returns a carrier PDF for a user-supplied parcel; use `https://postalform.com/mpp-shipping-labels.md` for that distinct flow.
- Do not use PostalForm to mail physical originals, checks, keys, ID cards, or packages, or for notarization, process serving, legal advice, or guaranteed delivery by a specific date.

## Best prompt for a user
Ask the user for the document or letter text, sender address, recipient address, and preferred mailing option. Create a draft, show its preview and total for approval, then use compatible Stripe token checkout or present the checkout URL.

## Minimal letter example
Call `postalform.create_letter_order_draft` with:

```json
{
  "letter": {
    "title": "Payment demand",
    "body": "Hello,\n\nThis is the letter body.\n\nSincerely,\nSender Example",
    "signature": "Sender Example"
  },
  "sender_name": "Sender Example",
  "sender_address_type": "Manual",
  "sender_address_manual": {
    "line1": "123 Sender St",
    "city": "Springfield",
    "state": "IL",
    "zip": "62701"
  },
  "recipient_name": "Recipient Example",
  "recipient_address_type": "Manual",
  "recipient_address_manual": {
    "line1": "456 Recipient Ave",
    "city": "Springfield",
    "state": "IL",
    "zip": "62701"
  }
}
```

Present the prepared draft and total for buyer approval. Then use `complete_checkout` if the client supplies a compatible Stripe token, or return `checkout_url` for hosted payment.

## Machine-readable references
- `https://postalform.com/llms.txt`
- `https://postalform.com/llms-full.txt`
- `https://postalform.com/ai.md`
- `https://postalform.com/developers.md`
- `https://postalform.com/agents.md`
- `https://postalform.com/mpp.md`
- `https://postalform.com/mpp-mail.md`
- `https://postalform.com/mpp-flowers.md`
- `https://postalform.com/mpp-shipping-labels.md`
- `https://postalform.com/pricing-calculator.md`
- `https://postalform.com/okf/index.md`
