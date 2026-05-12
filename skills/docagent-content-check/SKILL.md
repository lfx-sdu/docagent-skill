---
name: docagent-content-check
description: Calls Agents API Air8 content checking—POST check_doc_content and poll execution status. Use when users mention document checker rules, checker_config_id, order_id doc checks, or /air8_integration/check_doc_content.
---

# DocuAgent content check (Air8 API)

Base: `$DOCAGENT_AGENTS_API_BASE_URL`.

OpenAPI schemas: `DocumentCheckerRequest`, `DocumentCheckerResponse`, `ExecutionStatusResponse`.

## Start content check

`POST /air8_integration/check_doc_content`

Required: `order_id` (string or array of strings), `doc_types_to_check` (array).

Optional: `external_data`, `checker_config_id`, `rules`, `execution_id`, `created_by`.

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/check_doc_content" \
  -H "Content-Type: application/json" \
  -d '{
    "order_id":"<order-id>",
    "doc_types_to_check":["invoice"],
    "execution_id":"<optional>",
    "created_by":"<optional>"
  }'
```

Returns `execution_id` and `status`.

## Poll status

`GET /air8_integration/check_execution_status?execution_id=<execution_id>`

```bash
curl -sS "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/check_execution_status?execution_id=<execution_id>"
```

Reuse the same backoff guidance as extraction (`docagent-platform` skill).
