# AI Customer Support Triage
An n8n workflow that receives customer support requests, triages them with an LLM, applies **deterministic business rules on top of the AI decision**, replies to the customer (WhatsApp or email), escalates to humans through WhatsApp, and watches SLA deadlines in the background.

It is built around one idea: **the LLM suggests, the workflow decides.** The model classifies and drafts; hard rules, fail-safes and SLA tracking keep the system predictable when the model is wrong, slow or unavailable.



---

## Table of contents

- [Features](#features)
- [Architecture](#architecture)
- [Decision rules](#decision-rules)
- [API](#api)
- [Data model](#data-model)
- [Setup](#setup)
- [Configuration](#configuration)
- [Testing](#testing)
- [Security and privacy](#security-and-privacy)
- [Known limitations](#known-limitations)

---

## Features

| Area | What it does |
|---|---|
| **Intake** | Authenticated webhook, input validation, `400` with details on bad requests |
| **Idempotency** | Identical message from the same customer within 10 minutes is detected and not processed twice |
| **Context** | Looks up the customer's recent tickets and passes counts (not personal data) to the AI |
| **AI triage** | Category, priority, sentiment, intent, summary, language, draft reply — enforced by a JSON schema via *Structured Output Parser* |
| **Business rules** | Keyword escalation, repeat-contact escalation, angry-customer escalation, reply sanitization |
| **Fail-safe** | If the AI fails or returns invalid output, the ticket goes to a human — never to an automatic reply |
| **Customer reply** | Routine questions get the AI reply; escalated tickets get a fixed, safe acknowledgement (en / es / ru). Delivered via WhatsApp when the customer wrote from WhatsApp, otherwise by email |
| **Staff alerts** | WhatsApp message to the support queue (or on-call for `urgent`) plus an email copy with the full message |
| **SLA tracking** | Every ticket gets an `sla_due_at` based on priority. A scheduled watchdog sends **one digest** of breached tickets and marks them as alerted |
| **Ticket lifecycle** | Separate `resolve` endpoint closes tickets so the watchdog stops tracking them |
| **Observability** | Global Error Trigger sends a WhatsApp alert with the failing node and execution link; retries on external calls |

---

### Design notes

- **Storage nodes replace `$json`.** After MongoDB and the Data Table node, `$json` contains the service response, not the ticket. The `Ticket Context` node restores the full ticket so every downstream node can use plain `$json`.
- **`alwaysOutputData` on `find` nodes.** A MongoDB `find` with no matches returns zero items and silently stops the workflow. Both lookups return one empty item instead and filter real tickets in the next node.
- **Single source of configuration.** Numbers, recipients and keywords live in the `Config` node, not inside code or expressions.
- **Partial failure tolerance.** Storage, email and alert nodes use `continueRegularOutput`, so one failing integration does not block the webhook response or the other branches.

---

## Decision rules

The LLM output is validated and then adjusted by rules in the `Parse & Apply Rules` node.

| Trigger | Effect |
|---|---|
| AI sets `requires_human = true` | Human handling |
| Keyword found in message (multilingual, whole-word match) | Human handling, priority at least `high` |
| Customer has ≥ N tickets in the last D days (default 3 in 7) | Human handling, priority raised by one level |
| Sentiment is `angry` | Human handling, priority at least `high` |
| AI failed, timed out, or returned invalid JSON / enum values | **Fail-safe:** priority `high`, human handling, generic summary |
| AI said no human needed, but the reply is empty | Human handling |

Every applied rule is recorded in `escalation_reasons` (for example `keywords:chargeback`, `repeat_contact:3`), so a human can see **why** a ticket was escalated.

**SLA targets** (defaults, configurable):

| Priority | Target |
|---|---|
| `urgent` | 60 min |
| `high` | 4 h |
| `medium` | 24 h |
| `low` | 72 h |

**Ticket statuses:** `auto_resolved` → answered automatically · `needs_human` → waiting for staff · `resolved` → closed through the resolve endpoint.

---

## API

Both endpoints require the configured **Header Auth** credential. In examples below `<HEADER_NAME>: <HEADER_VALUE>` is whatever you defined in that credential. While testing in the editor use `/webhook-test/...`; for the activated workflow use `/webhook/...`.

### `POST /webhook/customer-support`

Request:

```json
{
  "customer": {
    "id": "c-1042",
    "name": "Anna",
    "email": "anna@example.com",
    "phone": "34600111222"
  },
  "message": "Hi, my headphones arrived with a broken left earpiece. Can I get a replacement?",
  "order_id": "ORD-7781",
  "channel": "whatsapp"
}
```

| Field | Required | Notes |
|---|---|---|
| `customer.email` | yes | Must be a valid email |
| `message` (or `text`) | yes | Trimmed to 4000 characters |
| `customer.phone` | for `channel: "whatsapp"` | Digits are extracted; country code required, no `+` needed |
| `customer.id`, `customer.name` | no | Default to email / `Customer` |
| `order_id` | no | |
| `channel` | no | `webhook` (default), `email`, `whatsapp`, `web`, `chat` |

Responses:

| Code | Meaning | Body |
|---|---|---|
| `201` | Ticket created | `request_id`, `status`, `category`, `priority`, `requires_human`, `summary`, `sla_due_at` |
| `200` | Duplicate of a recent identical request | `request_id` (of the original), `status: "duplicate"` |
| `400` | Validation failed | `error: "validation_failed"`, `details: [...]` |

```json
{
  "request_id": "SUP-1760000000000-ab12cd",
  "status": "needs_human",
  "category": "warranty",
  "priority": "high",
  "requires_human": true,
  "summary": "Customer reports a broken earpiece on a recent order and asks for a replacement.",
  "sla_due_at": "2026-10-07T14:30:00.000Z"
}
```

### `POST /webhook/customer-support/resolve`

```json
{ "request_id": "SUP-1760000000000-ab12cd", "agent": "maria" }
```

Returns `{ "request_id": "...", "status": "resolved" }`. A malformed `request_id` makes the execution fail and triggers the error alert.

---

## Data model

### MongoDB — collection `support_tickets`

`request_id`, `received_at`, `customer_id`, `customer_name`, `customer_email`, `customer_phone`, `order_id`, `message`, `message_fingerprint`, `channel`, `category`, `priority`, `sentiment`, `intent`, `summary`, `requires_human`, `customer_reply`, `internal_action`, `language`, `escalation_reasons[]`, `previous_tickets`, `sla_due_at`, `sla_alerted`, `processed_at`, `status`

Added later by other flows: `sla_alerted_at` (watchdog), `resolved_at`, `resolved_by` (resolve endpoint).

Suggested indexes:

```js
db.support_tickets.createIndex({ request_id: 1 }, { unique: true });
db.support_tickets.createIndex({ customer_email: 1, received_at: -1 });
db.support_tickets.createIndex({ status: 1, sla_alerted: 1, sla_due_at: 1 });
```

### n8n Data Table — `support_tickets` (lightweight audit view)

| Column | Type |
|---|---|
| `request_id`, `received_at`, `customer_email`, `order_id`, `category`, `priority`, `sentiment`, `summary`, `status`, `sla_due_at` | string |
| `requires_human` | boolean |

---

## Setup

### Requirements

- An n8n instance that includes **Data Tables** and the **AI Agent** node (v3) with the Structured Output Parser
- MongoDB
- OpenAI API key
- Gmail account (OAuth2)
- WhatsApp Business Cloud API access (Meta app, phone number ID, access token)

### Steps

1. **Import** `customer-support-workflow.json` in n8n (*Workflows → Import from file*).
2. **Create credentials** and attach them to the nodes: Header Auth (both webhooks), OpenAI, MongoDB, Gmail OAuth2, WhatsApp.
3. **Create the Data Table** `support_tickets` with the columns above.
4. **Fill in configuration** in the `Config` and `Watchdog Config` nodes, and the recipient / phone number ID in the `Error Alert` node.
5. **Set the error workflow:** *Workflow settings → Error Workflow →* select this workflow. This makes the `Error Trigger` fire for its own failures.
6. **Create the MongoDB indexes** (recommended).
7. **Activate** the workflow.

> WhatsApp numbers must contain digits only, with country code and without `+`.

---

## Configuration

All values are in the **`Config`** node unless noted.

| Key | Default | Purpose |
|---|---|---|
| `support_whatsapp` | placeholder | Support queue number for escalations |
| `oncall_whatsapp` | placeholder | Number for `urgent` tickets |
| `support_email` | `support@electronics.com` | Escalation email copy |
| `whatsapp_phone_number_id` | placeholder | WhatsApp Business phone number ID |
| `repeat_window_days` | `7` | History window for repeat-contact detection |
| `repeat_contact_threshold` | `3` | Tickets in the window that trigger escalation |
| `duplicate_window_min` | `10` | Window for identical-message detection |
| `sla_urgent_min` / `sla_high_min` / `sla_medium_min` / `sla_low_min` | `60` / `240` / `1440` / `4320` | SLA targets in minutes |
| `escalation_keywords` | multilingual list | Comma-separated, matched as whole words, case-insensitive |

**`Watchdog Config`:** `sla_whatsapp` (recipient of the digest), `whatsapp_phone_number_id`.

---

## Testing

```bash
# 1. Routine question → expect 201, auto_resolved
curl -X POST "https://<n8n-host>/webhook-test/customer-support" \
  -H "Content-Type: application/json" -H "<HEADER_NAME>: <HEADER_VALUE>" \
  -d '{"customer":{"name":"Tom","email":"tom@example.com"},"message":"What are your support opening hours?"}'

# 2. Legal threat → expect needs_human, priority high or urgent
curl -X POST "https://<n8n-host>/webhook-test/customer-support" \
  -H "Content-Type: application/json" -H "<HEADER_NAME>: <HEADER_VALUE>" \
  -d '{"customer":{"name":"Tom","email":"tom@example.com"},"message":"I was charged twice. I will contact my lawyer and start a chargeback."}'

# 3. Invalid input → expect 400
curl -X POST "https://<n8n-host>/webhook-test/customer-support" \
  -H "Content-Type: application/json" -H "<HEADER_NAME>: <HEADER_VALUE>" \
  -d '{"customer":{"email":"not-an-email"},"message":""}'
```

| Scenario | How to trigger | Expected result |
|---|---|---|
| Duplicate | Send request #1 twice within 10 minutes | Second call returns `200` / `duplicate` |
| Repeat contact | Send 4 different messages from one email within 7 days | 4th ticket is escalated with `repeat_contact:3` |
| AI failure | Temporarily break the OpenAI credential | Ticket is created as `needs_human`, reason `ai_triage_failed` |
| SLA breach | Set `sla_urgent_min` to `1`, create an urgent ticket, wait for the next run | One WhatsApp digest, then no repeats |
| Resolve | Call the resolve endpoint for that ticket | Ticket no longer appears in the digest |
| Error handler | Send a malformed `request_id` to the resolve endpoint | WhatsApp error alert |

---

## Security and privacy

- **Authentication** on both webhooks via Header Auth.
- **Prompt-injection hardening.** The system prompt marks all customer fields as untrusted data. Beyond the prompt, the workflow never lets the model decide alone: keyword, repeat-contact and sentiment rules can only **raise** priority and force human handling.
- **No unsafe AI text to customers.** Links are stripped from AI replies, reply length is capped, and escalated tickets receive a fixed acknowledgement instead of generated text. The prompt forbids promising refunds, replacements or account changes.
- **Data minimization for the LLM.** The model receives name, message, order ID, channel and history *counts*. Email address and phone number are not sent.
- **Input limits.** Message, name and ID lengths are capped before processing.

---

## Known limitations

- **WhatsApp 24-hour window.** Free-form messages are allowed only within 24 hours of the recipient's last message to your number. Customer replies for WhatsApp-channel tickets are fine (the customer just wrote), but **staff alerts, the SLA digest and the error alert can fail outside that window**. For production, switch those nodes to an approved *template* message.
- **No WhatsApp outreach for web/email tickets.** Customers who did not come through WhatsApp are answered by email. Proactive WhatsApp messages need opt-in and templates.
- **Duplicate detection is exact-match** on a normalized message text for the same email. Paraphrased resubmissions are treated as new tickets (repeat-contact logic still applies).
- **Persistence is best-effort.** Storage nodes continue on error so the customer still gets a response. If MongoDB is down the ticket will not be stored; consider adding a retry queue or a stricter failure policy.
- **No rate limiting.** Put the webhook behind a reverse proxy or API gateway if it is publicly exposed.
- **Retention.** Customer messages are stored in plain text. Add a retention policy and access controls to meet your privacy requirements.
- **Resolve endpoint does not check that the ticket exists.** Updating an unknown `request_id` is a no-op.
- **Keyword and SLA values are examples**, not legal or operational advice. Tune them for your business.
