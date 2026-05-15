# USERFLOW - DocuAgent Skills API

This file shows the practical flow to:
- send correct payloads,
- call Extraction and ConfigAgent APIs,
- and retrieve final results safely.

## 1) Setup once

```bash
export DOCAGENT_AGENTS_API_BASE_URL="https://api.uat.t4s.lfxdigital.app/agents/v1"
export DOCAGENT_AGENTS_API_KEY="<config-agent-api-key>"
```

Health check:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  "$DOCAGENT_AGENTS_API_BASE_URL/health"
```

ConfigAgent auth check:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/fetch-threads/<user_id>?limit=1&skip=0"
```

Expected:
- `200`: setup is good.
- `401/403`: wrong or missing API key.
- `422`: invalid path/query values.

## 2) Start extraction (Air8)

Endpoint:
- `POST /air8_integration/validate_and_extract_docs`

Minimum payload:

```json
{
  "file_uri": "https://<storage>/<file>.pdf",
  "order_id": "ORDER-123",
  "nation": "VN",
  "possible_doc_type": ["invoice"]
}
```

Example:

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/validate_and_extract_docs" \
  -H "Content-Type: application/json" \
  -d '{
    "file_uri":"https://<storage>/<file>.pdf",
    "order_id":"ORDER-123",
    "nation":"VN",
    "possible_doc_type":["invoice"],
    "created_by":"integration-user"
  }'
```

Keep the returned `execution_id`. This is your primary correlation key.

## 3) Poll extraction result

Endpoint:
- `GET /air8_integration/check_execution_status?execution_id=<execution_id>`

```bash
curl -sS \
  "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/check_execution_status?execution_id=<execution_id>"
```

Poll every 2-5 seconds until terminal status (`completed` or `failed`).

## 4) Use ConfigAgent API

All ConfigAgent endpoints below require:
- `X-API-Key: $DOCAGENT_AGENTS_API_KEY`

### 4.1 Chat stream

Endpoint:
- `POST /config_integration/config-agent-stream`

Payload:

```json
{
  "message": "Create invoice extraction config",
  "thread_id": "<uuid>",
  "user_id": "user-123"
}
```

Example:

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/config-agent-stream" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  -d '{
    "message":"Create invoice extraction config",
    "thread_id":"<uuid>",
    "user_id":"user-123"
  }'
```

### 4.2 Async global config job

Start job:

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/generate-global-config" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  -d '{"max_attempts":3}'
```

Poll result:

```bash
curl -sS \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/config-job/<job_id>"
```

## 5) Fetch final results

### Extraction results
- `GET /air8_integration/check_execution_status?execution_id=<execution_id>`

### ConfigAgent chat results
- `GET /config_integration/fetch-dialog/<thread_id>`
- `GET /config_integration/fetch-message/<thread_id>/<message_id>`
- `GET /config_integration/fetch-threads/<user_id>?limit=<n>&skip=<n>`

### ConfigAgent job results
- `GET /config_integration/config-job/<job_id>`
- `GET /config_integration/shipment-code-embeddings-status/<job_id>`
- `GET /config_integration/invoice-code-embeddings-status/<job_id>`

## 6) Common payload mistakes and fixes

- `possible_doc_type` is not an array -> send `["invoice"]`, not `"invoice"`.
- Missing `Content-Type: application/json` on POST -> add the header.
- Missing API key on ConfigAgent routes -> add `X-API-Key`.
- Losing IDs (`execution_id`, `thread_id`, `job_id`) -> persist IDs from each POST response before next step.
- Reusing wrong IDs across routes -> use `execution_id` for Air8 status and `job_id` for config jobs.

## 7) Recommended integration pattern

1. Preflight once per run.
2. Start extraction and capture `execution_id`.
3. Poll extraction to terminal.
4. Trigger ConfigAgent action when needed.
5. Poll/fetch by `thread_id`, `message_id`, or `job_id`.
6. Store request payload + returned IDs in logs for traceability.
