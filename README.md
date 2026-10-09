# n8n Business Automation

## Workflows at a glance

| # | Workflow | Trigger | Uses AI | Main integrations |
|---|---|---|---|---|
| 1 | [Customer Support Automation](#1-customer-support-automation) | Webhook, schedule (SLA watchdog) | Yes | MongoDB, n8n Data Tables, WhatsApp, Gmail |
| 2 | [Lead Qualification Automation](#2-lead-qualification-automation) | Webhook | Yes | MongoDB, n8n Data Tables, HubSpot, Slack, Gmail, Google Sheets |
| 3 | [E-commerce Fraud Detection](#3-e-commerce-fraud-detection) | Webhook | Yes, only when needed | MongoDB, Slack, Gmail |
| 4 | [Crypto Rates Sync with Anomaly Guard](#4-crypto-rates-sync-with-anomaly-guard) | Schedule | No | MongoDB, CoinGecko, Binance, WhatsApp |
| 5 | [Crypto Exchange FAQ Assistant](#5-crypto-exchange-faq-assistant) | Webhooks (ingest and ask) | Yes | OpenAI, Qdrant, MongoDB, n8n Data Tables, Slack |
| 6 | [Data Tables Retention Cleanup](#6-data-tables-retention-cleanup) | Schedule, manual | No | n8n Data Tables, Slack |

---

## Workflows

### 1. Customer Support Automation

AI-powered customer support workflow that:

* validates and processes incoming requests;
* analyzes customer context and conversation history;
* classifies and prioritizes tickets with AI;
* applies deterministic escalation rules;
* stores data in MongoDB and n8n Data Tables;
* automatically replies to customers;
* escalates complex cases to support staff;
* monitors SLA deadlines.

**Stack:** n8n · OpenAI · MongoDB · n8n Data Tables · WhatsApp Business Cloud API · Gmail

**Docs:** [`customer_support_triage/README.md`](customer_support_triage/README.md)

### 2. Lead Qualification Automation

Automated B2B lead processing workflow that:

* receives and validates incoming leads;
* prevents duplicates and excessive submissions;
* evaluates leads with AI;
* applies deterministic scoring rules;
* classifies leads as HOT, WARM, or COLD;
* stores qualification results;
* routes leads to CRM and communication channels;
* generates personalized follow-ups.

**Stack:** n8n · OpenAI · MongoDB · n8n Data Tables · HubSpot · Slack · Gmail · Google Sheets

**Docs:** [`lead_qualification_crm/README.md`](lead_qualification_crm/README.md)

### 3. E-commerce Fraud Detection

Automated e-commerce transaction security and risk analysis workflow that:

* receives and validates incoming transaction webhooks;
* normalizes order parameters and evaluates risk signals;
* calculates deterministic risk scores (LOW, MEDIUM, HIGH);
* bypasses AI for LOW risk orders to optimize latency and LLM costs;
* analyzes complex MEDIUM and HIGH risk patterns using AI with built-in output guardrails;
* stores transactional and audit logs in MongoDB;
* triggers real-time alerts via Slack and email notifications via Gmail for manual reviews;
* sends customer-safe payment review updates without exposing internal security metrics.

**Stack:** n8n · LLM · MongoDB · Slack · Gmail

**Docs:** [`ecommerce_order_fraud_detection/README.md`](ecommerce_order_fraud_detection/README.md)

### 4. Crypto Rates Sync with Anomaly Guard

Scheduled, AI-free data synchronization workflow that keeps cryptocurrency prices in MongoDB trustworthy:

* fetches prices on a schedule from CoinGecko with an automatic Binance fallback (on request errors **and** on incomplete or invalid data);
* compares every new price with the last accepted one;
* holds suspicious jumps and accepts them only after N consecutive, mutually consistent readings, so a real market move is never blocked forever;
* stores accepted prices and an append-only price history;
* tracks the health of the data sources in MongoDB (consecutive failures, alert cooldown);
* sends WhatsApp alerts: one digest per run for price anomalies, outage alerts, and a recovery notice;
* retries an undelivered outage alert on the next run instead of losing it;
* reports unexpected errors through a global error handler.

**Stack:** n8n · MongoDB · CoinGecko API · Binance public API · WhatsApp Business Cloud API

**Docs:** [`rates_sync_with_anomaly_guard/README.md`](rates_sync_with_anomaly_guard/README.md)

### 5. Crypto Exchange FAQ Assistant

Two workflows provide a retrieval-grounded FAQ service for a cryptocurrency exchange:

* the protected ingest endpoint validates FAQ documents, chunks and embeds them, and indexes them in Qdrant;
* the ask endpoint retrieves relevant passages and uses Router, Answer and Critic agents to classify questions and validate cited answers;
* account-specific, suspected-fraud and complaint questions are escalated instead of answered from public FAQ material;
* unanswered questions and QA results are tracked in n8n Data Tables, with alerts and escalations sent to Slack.

**Stack:** n8n · OpenAI · Qdrant · MongoDB Chat Memory · n8n Data Tables · Slack

**Docs:** [`crypto_faq_rag/README.md`](crypto_faq_rag/README.md)

### 6. Data Tables Retention Cleanup

Scheduled, AI-free maintenance workflow that removes old rows from the configured FAQ-assistant Data Tables:

* plans cleanup by numeric epoch-millisecond timestamp columns;
* defaults to dry-run, with a per-table deletion cap and a best-effort run lock;
* records per-table results in `maintenance_log` and sends Slack summaries for problems (or every run when enabled);
* does not clean MongoDB collections or Data Tables that are not listed in its `Config`.

**Stack:** n8n · n8n Data Tables · Slack

**Docs:** [`tables_cleanup/README.md`](tables_cleanup/README.md)

---

## Design principles

The same engineering ideas appear across the workflows.

- **AI suggests, rules decide.** Where an LLM is used, its output is validated against a schema, clamped, and then adjusted by deterministic rules (escalation keywords, repeat-contact detection, scoring bonuses, risk thresholds). The model cannot talk the workflow into a better outcome.
- **Fail-safe defaults.** When the AI fails, work is routed to a human or to a safe middle ground instead of being auto-answered or silently dropped.
- **Untrusted input.** Requests are validated, normalized and length-capped; AI prompts treat user text as data; outbound text is sanitized (links removed, Slack text escaped, fixed templates for risky cases); only the data the model needs is sent to it.
- **Idempotency and abuse protection.** Duplicate submissions are detected, repeated submissions are rate-limited, and public endpoints do not leak internal scores.
- **State in the database, not in the workflow.** Streak counters, failure counters, SLA flags and rate-limit checks are stored in MongoDB, so behaviour survives restarts and spans scheduled runs.
- **Resilient integrations.** External calls use retries and error outputs; one failing integration does not stop the other branches; alerts are deduplicated so people are not spammed.
- **Observable.** A global Error Trigger reports the failing node, the error and an execution link; routing decisions are stored with their reasons (for example `escalation_reasons`, `score_breakdown`).
- **Configurable.** Thresholds, limits and recipients live in a `Config` node instead of being hard-coded in expressions.

---

## Getting started

### Requirements

- An n8n instance with the required nodes: **Data Tables** (workflows 1, 2, 5 and 6) and **AI Agent / LangChain** nodes (workflows 1–3 and 5; see each README for details)
- MongoDB
- OpenAI API key (workflows 1–3 and 5)
- Qdrant (workflow 5)
- Accounts for the integrations you plan to use: WhatsApp Business Cloud API, Gmail, Slack, HubSpot, Google Sheets

### Import a workflow

1. Open the workflow folder and read its `README.md`.
2. In n8n choose *Workflows → Import from file* and select the workflow's `.json` file from its folder (the exact filename is listed in its README).
3. Create the credentials the workflow needs and attach them to the nodes.
4. Create the required MongoDB indexes and any n8n Data Tables listed in the workflow README.
5. Fill in the `Config` node (recipients, thresholds, limits).
6. In *Workflow settings → Error Workflow* select the workflow itself, so its `Error Trigger` fires for its own failures.
7. Test with the `curl` examples in the workflow README, then activate it.

---

## Before you publish or deploy

- **Remove credential references.** Exported workflows contain credential IDs and an instance ID. Delete the `credentials` blocks and `meta.instanceId` before publishing the JSON, and never commit real phone numbers, e-mail addresses or tokens.
- **WhatsApp 24-hour window.** Free-form WhatsApp messages can only be sent within 24 hours of the recipient's last message to your business number. Replies to customers who just wrote are fine; staff alerts and digests should use approved **template** messages in production.
- **Public webhooks.** Add a CAPTCHA or honeypot on the frontend, restrict CORS to your domain and rate-limit at the reverse proxy.
- **Personal data.** Tickets and leads contain customer data. Define a retention policy and restrict database access.
- **Check the update nodes.** In MongoDB *update* nodes, make sure **Upsert** is enabled where the workflow relies on creating documents (for example, the rates workflow).
- **Tune the numbers.** Thresholds, SLA targets and scoring bonuses are examples, not recommendations.
