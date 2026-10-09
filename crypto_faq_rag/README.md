# Crypto Exchange FAQ Assistant

Two n8n workflows for a grounded cryptocurrency-exchange FAQ assistant built with OpenAI, Qdrant, MongoDB Chat Memory, n8n Data Tables, and Slack.

- `crypto_faq_rag_ingest.json` — protected `POST /kb/ingest` endpoint for indexing FAQ documents
<img width="1486" height="486" alt="Crypto FAQ RAG (Ingest)" src="https://github.com/user-attachments/assets/dd7ae759-ab1d-4d1e-a6fa-6db732324508" />
  
- `crypto_faq_rag_ask.json` — public `POST /kb/ask` endpoint for answering questions using retrieval and three AI agents (Router, Answer, Critic)
<img width="1454" height="567" alt="Crypto FAQ RAG (Ask)" src="https://github.com/user-attachments/assets/f908df48-c11f-48ff-9abe-1481a227df6b" />

## Requirements

- n8n with Code, Webhook, HTTP Request, Data Tables, AI Agent, OpenAI, MongoDB Chat Memory, and Slack nodes available.
- Qdrant reachable from n8n.
- OpenAI API credentials for chat models and embeddings.
- MongoDB for agent memory.
- Slack credentials if notifications are enabled.

## Qdrant setup

Example Docker Compose service (pin a tested version instead of `latest` for deployments):

```yaml
services:
  qdrant:
    image: qdrant/qdrant:<tested-version>
    restart: unless-stopped
    ports:
      - "127.0.0.1:6333:6333"
    volumes:
      - qdrant_storage:/qdrant/storage
    environment:
      QDRANT__SERVICE__API_KEY: ${QDRANT_API_KEY}

volumes:
  qdrant_storage:
```

If n8n and Qdrant are in the same Docker network, `http://qdrant:6333` is normally appropriate. If n8n runs outside that network, configure `qdrant_url` accordingly. Keep the Qdrant API key secret and do not expose Qdrant publicly.

Collections are created by the ingest workflow when needed: `faq`, `fees_limits`, and `compliance`, using cosine distance and 1536-dimensional vectors for `text-embedding-3-small`. If changing the embedding model or dimension, use new collections or rebuild all vectors consistently.

## Data Tables

Create these tables in n8n and select each table in every corresponding Data Table node after importing. Column types matter; timestamp columns are epoch milliseconds stored as numbers.

| Table | Columns |
|---|---|
| `kb_documents` | `doc_id` (string), `title` (string), `collection` (string), `lang` (string), `category` (string), `version` (number), `content_hash` (string), `status` (string), `chunks_count` (number), `last_error` (string), `updated_ms` (number), `cleanup_ts` (number) |
| `kb_qa_log` | `request_id`, `session_id`, `client_key`, `question_hash`, `question_masked`, `decision`, `answer`, `sources`, `language`, `reasons` (string), `ts` (number), `standalone` (number), `groundedness` (number), `best_score` (number), `cache_hit` (number) |
| `kb_unanswered` | `question_hash`, `question_masked`, `language`, `last_request_id`, `status` (string), `count` (number), `first_seen_ms` (number), `last_seen_ms` (number) |
| `kb_alert_state` | `alert_key` (string), `updated_ms` (number) |

`kb_documents.status` may be `pending`, `indexed`, `indexed_with_cleanup_warning`, or `failed`. A cleanup warning means new points were written but stale-point deletion did not complete; the document is eligible for re-index/retry rather than being treated as cleanly indexed.

## Import and configure

1. Start Qdrant and create the four Data Tables above.
2. Import both JSON files separately in **Workflows → Import from file**.
3. In each workflow, open every Data Table node and select its intended table from the dropdown so the instance-specific table ID and schema are loaded. Do not rely on the table name or ID saved in the export: several nodes may contain stale table references. In particular, set `Mark Pending` and `Mark Failed` in the ingest workflow to `kb_documents`, and set `Get Cached Answer` in the ask workflow to `kb_qa_log`. Confirm the other node-to-table mappings against the schemas above.
4. Attach credentials to the relevant nodes:
   - Ingest: Header Auth on `Webhook Ingest`; Qdrant Header Auth (`api-key`) on collection/index/upsert/delete nodes; OpenAI on `Embed Chunks`; Slack on `Slack Ingest Failures`.
   - Ask: OpenAI on embedding and the three model nodes; MongoDB on all three memory nodes; Qdrant Header Auth (`api-key`) on `Qdrant Search`; Slack on alert/escalation nodes.
5. Edit both `Config` nodes. Set the Qdrant URL, model/dimension, thresholds and Slack channel ID. In the Ask workflow, set `Webhook Ask` allowed origins to your real frontend origin (CORS is not authentication).
6. Configure the `Authorization: Bearer ...` Header Auth credential on the ingest webhook. Use a long random token and keep it out of source control.
7. Configure the Ask endpoint behind a trusted reverse proxy or API gateway. It must overwrite incoming `X-Forwarded-For` rather than trust a client-supplied value, enforce a real rate limit, and ideally require authenticated or signed session tokens. The in-workflow Data Table rate check is best-effort and is not atomic under concurrent requests.
8. Add retention for `kb_qa_log`, `kb_unanswered`, and MongoDB collections `kb_memory_router`, `kb_memory_answer`, and `kb_memory_critic`. n8n Data Table cleanup does not clean MongoDB memory.
9. Attach a separate n8n Error Workflow if you need global unhandled-error notifications. These two request workflows do not configure themselves as their own Error Workflow.
10. Test in a non-production environment, then activate each workflow.

## Ingest endpoint

```bash
curl -X POST https://n8n.example.com/webhook/kb/ingest \
  -H "Authorization: Bearer $KB_INGEST_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "documents": [{
      "doc_id": "withdrawal-fees",
      "title": "Withdrawal fees",
      "collection": "fees_limits",
      "lang": "en",
      "category": "fees",
      "version": 3,
      "text": "Withdrawal fees depend on the network..."
    }],
    "force": false,
    "allow_downgrade": false
  }'
```

- `force: true` reindexes even when content is unchanged; it does not permit lowering the stored version.
- `allow_downgrade: true` explicitly permits an older document version. Do not expose this flag to untrusted clients.
- Unchanged cleanly indexed documents are skipped. A metadata or content change triggers reindexing because the content hash includes collection, title, language, category, and text.
- New points are upserted before stale points are deleted. If stale-point deletion fails, the result is `indexed_with_cleanup_warning`, the API returns partial-success status where appropriate, and a later retry can clean up old points.

Typical response status codes: `200` complete/no-op success, `207` partial failure or cleanup warning, `422` all documents rejected, `502` all indexing attempts failed. Verify the exact response in your deployed n8n version after importing.

## Ask endpoint

```bash
curl -X POST https://n8n.example.com/webhook/kb/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"What is the minimum withdrawal for USDT on TRC20?","session_id":"user-session-0001"}'
```

A successful answer returns `decision: "ANSWER"`, an answer containing inline references such as `[1]`, and a `sources` array mapping the references to document metadata. Other decisions include `CLARIFY`, `NO_ANSWER`, `ESCALATE`, `REFUSE_ADVICE`, `SECRET`, and `RATE_LIMITED` (HTTP 429).

The session ID is not an authentication credential. Do not use a client-controlled session ID as proof of identity. For production, use authenticated users or signed, high-entropy session tokens, and scope memory to the verified client identity.

## Decision and safety notes

- Seed phrase, key, token, and password detection is heuristic; it cannot guarantee detection of every secret format. Requests classified as `SECRET` skip `kb_qa_log`; alert messages contain only the request ID and a generic warning.
- Prompt-injection phrase matching is a heuristic. The agent prompts also treat user questions and retrieved documents as untrusted data, but this is not a complete prompt-injection defense.
- Retrieval scores and groundedness thresholds require evaluation on representative questions; defaults are examples, not recommendations.
- HTTP Request batching groups requests into batches; do not assume it guarantees simultaneous parallel execution. Confirm actual concurrency and API limits for the deployed n8n version.
- In-workflow rate limiting uses Data Table history and can race under concurrent requests. Enforce the primary limit at the reverse proxy/API gateway.
- Non-cryptographic hashes are used for document change detection and deduplication, not as security controls.
- Configure a separate error workflow and retention/monitoring policies before production use.
