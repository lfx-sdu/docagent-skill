---
name: docagent-config-agent
description: Operates Agents API ConfigAgent routes—streaming chat, thread/dialog/message reads, async global-config jobs, shipment mapping, payment Excel processing, embeddings, PDF merge and prepare-upload—using X-API-Key where OpenAPI requires it. Use for /config_integration/* integrations.
---

# DocuAgent config agent (`/config_integration/*`)

**Auth:** Send `X-API-Key: $DOCAGENT_AGENTS_API_KEY` on every operation listed below (OpenAPI `APIKeyHeader`).

Base URL: `$DOCAGENT_AGENTS_API_BASE_URL`.

## Chat / dialogs

| Method | Path | Notes |
|--------|------|--------|
| POST | `/config_integration/config-agent-stream` | Body: `ExecutionAgentRequest` (`message`, `thread_id`, optional `user_id`, `token_id`, `image_urls`, `message_id`) |
| GET | `/config_integration/fetch-dialog/{thread_id}` | Full dialog |
| GET | `/config_integration/fetch-message/{thread_id}/{message_id}` | Single message |
| GET | `/config_integration/fetch-threads/{user_id}` | Query: `limit`, `skip` |

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/config-agent-stream" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  -d '{"message":"...","thread_id":"<uuid>","user_id":"<optional>"}'
```

## Global config generation (async)

- POST `/config_integration/generate-global-config` → `202`, body includes `job_id`, `status`
- GET `/config_integration/config-job/{job_id}` — poll until `completed` or `failed`; `config` when completed

Body for generate: `GenerateGlobalConfigRequest` (`max_attempts` optional, default 3).

## Spreadsheet / domain helpers

- POST `/config_integration/run-shipment-mapping`
- POST `/config_integration/run-shipment-mapping-and-merge`
- POST `/config_integration/process-application-payment-excel`

Bodies: `ShipmentMappingRequest`, `ShipmentMappingRequest`, `ProcessApplicationPaymentRequest` respectively.

## Embeddings jobs

- POST `/config_integration/run-shipment-code-embeddings`
- GET `/config_integration/shipment-code-embeddings-status/{job_id}`
- POST `/config_integration/run-invoice-code-embeddings`
- GET `/config_integration/invoice-code-embeddings-status/{job_id}`

Poll status until `completed`/`failed`/terminal per response schema.

## Multipart uploads (detail in `docagent-file-prep`)

- POST `/config_integration/prepare-and-upload`
- POST `/config_integration/merge-pdf`

Use OpenAPI multipart field names (`execution_id`, optional `files`, optional `base64_inputs`).

**Do not duplicate** multipart recipes here — see `docagent-file-prep` for curl `-F` examples.
