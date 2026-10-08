# Lead Intake & Qualification

<img width="1503" height="542" alt="Lead Intake   Qualification" src="https://github.com/user-attachments/assets/1996edf7-d2d9-4d9b-ad70-5342ca771c6b" />

An n8n workflow that receives inbound leads via webhook, scores them with AI plus deterministic rules, stores the result, and routes each lead as HOT, WARM, or COLD.

## How it works

1. **Intake and validation.** `POST /lead-intake`. Fields are stripped of control characters and truncated. `name`, `email`, and `message` are required; otherwise the workflow returns `400`.
2. **Duplicate and rate-limit guard.** Previous submissions from the same email within `duplicate_window_min` are looked up in MongoDB. An identical message returns `200 duplicate`; exceeding `max_submissions_per_window` returns `429 rate_limited`.
3. **Async response.** A new lead immediately gets `202 Accepted` with a `lead_id`, and processing continues in the background.
4. **AI scoring.** An AI Agent (OpenAI `gpt-5-mini`, structured output) evaluates only the message text and returns 0-65 points, plus intent, fit, a short summary, and a reply draft.
5. **Business rules.** A Code node adds deterministic bonuses and assigns the temperature.
6. **Storage.** The lead is saved to MongoDB (`leads` collection) and to the n8n Data Table `qualified_leads`.
7. **Routing.**

| Category | Condition | Actions |
|----------|-----------|---------|
| HOT | score >= 80 | HubSpot contact, Slack alert to `#sales`, personalized email, Google Sheets row |
| WARM | 50 <= score < 80 | Personalized follow-up email, Google Sheets row |
| COLD | score < 50 | Static acknowledgement email, Google Sheets row |

## Scoring

Total score is up to 65 points from the AI plus up to 35 points from bonuses, capped at 100.

| Rule | Points |
|------|--------|
| AI score based on message content | 0-65 |
| Business email (not a free email domain) | +5 |
| Demo requested (`demo_requested`) | +10 |
| 100+ employees | +10 |
| 500+ employees (stacks with the previous rule) | +10 |

The breakdown is stored in `score_breakdown`. If the AI output cannot be parsed, the lead is floored at `warm_threshold` and flagged `ai_failed`, so a real lead never silently falls into COLD.

All thresholds, bonuses, limits, and the free email domain list live in the **Config** node.

## Request format

```bash
curl -X POST https://<your-n8n-host>/webhook/lead-intake \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Anna Smith",
    "email": "anna@example.com",
    "message": "We need to automate invoice processing, about 2000 per month.",
    "company": "Example Inc",
    "phone": "+1 555 0100",
    "employees": 250,
    "industry": "logistics",
    "source": "website",
    "website": "https://example.com",
    "demo_requested": true
  }'
```

Required: `name`, `email`, `message`. All other fields are optional.

### Responses

| Code | Body | Meaning |
|------|------|---------|
| 202 | `{ "lead_id": "...", "status": "received" }` | Lead accepted for processing |
| 200 | `{ "lead_id": "...", "status": "duplicate" }` | Same message already received within the window |
| 429 | `{ "status": "rate_limited" }` | Too many submissions from this email |
| 400 | `{ "error": "validation_failed", "details": [...] }` | Validation error |

## Setup

1. Import `lead_qualification_crm.json` into n8n (Workflows -> Import from File).
2. Configure credentials: MongoDB, OpenAI, HubSpot (App Token), Slack, Gmail (OAuth2), Google Sheets (OAuth2).
3. Create a Data Table named `qualified_leads` with columns: `lead_id`, `received_at`, `name`, `email`, `company`, `score`, `temperature`, `intent`, `fit`, `next_action`, `qualified_at`.
4. In the **Qualified Leads** node, replace `REPLACE_WITH_GOOGLE_SHEET_ID` with your spreadsheet ID. The sheet must be named `Qualified Leads` and have the same columns as the Data Table.
5. Check the Slack channels (`#sales`, `#automation-alerts`) and the "Northstar Automation" sign-off in the email nodes.
6. Set this workflow as its own Error Workflow in its settings so the Error Trigger can fire.
7. Activate the workflow.


## Security

- All lead fields are treated as untrusted input. The AI system prompt instructs the model to ignore any instructions embedded in lead data (prompt injection).
- The AI score is capped at 0-65, so manipulating the message text cannot push a lead into HOT without the deterministic bonuses.
- Links are stripped from the generated email text, and values in the Slack message are escaped to prevent `<!channel>` and arbitrary link injection.
- The webhook has no authentication by default. For production, enable Header Auth or restrict access at the proxy level.

## Limitations

- Deduplication compares the first 300 characters of the normalized message and checks at most 20 recent submissions per email.
- Integration nodes use `onError: continueRegularOutput`, so a failure in one integration (for example Slack) does not stop the rest of the chain.
