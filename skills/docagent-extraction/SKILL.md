---
name: docagent-extraction
description: Calls DocuAgent Agents API Air8 extraction endpoints—validate/extract documents and merged field-config preview—with execution-status polling via execution_id. Use when users mention document extraction, validate_and_extract_docs, field_config_id, or extraction pipeline against /agents/v1.
---

# DocuAgent extraction (Air8 API)

Base: `$DOCAGENT_AGENTS_API_BASE_URL` — paths below are appended.

Schemas: `$DOCAGENT_AGENTS_API_BASE_URL/openapi.json` (`DocsValidationRequest`, `MergeConfigRequest`, …).

## Start extraction / validation

`POST /air8_integration/validate_and_extract_docs`

Body fields (minimum from OpenAPI): `file_uri`, `order_id`, `nation`, `possible_doc_type` (array). Optional includes `execution_id`, `created_by`, `field_config_id`, `parent_config_id`, preprocessing flags (`enable_image_preprocessing`, `rotate_upright`, …), `external_context`, `prompt_template_id`, etc.

Payload checklist before sending:
- `possible_doc_type` must be an array (for example `["invoice"]`).
- `file_uri` must be an accessible URI for the backend runtime.
- Keep `order_id` stable if you need deterministic traceability.
- Pass `execution_id` only when intentionally correlating with an existing run.

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/validate_and_extract_docs" \
  -H "Content-Type: application/json" \
  -d '{
    "file_uri":"<blob-url-or-uri>",
    "order_id":"<order-id>",
    "nation":"<nation>",
    "possible_doc_type":["invoice"],
    "execution_id":"<optional-stable-exec-id>",
    "created_by":"<user-or-service-id>"
  }'
```

Response (typical): `execution_id`, `status`. Poll until terminal.

## Poll execution status

`GET /air8_integration/check_execution_status?execution_id=<id>`

```bash
curl -sS "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/check_execution_status?execution_id=<execution_id>"
```

Simple polling pattern:

```bash
while true; do
  response="$(curl -sS "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/check_execution_status?execution_id=<execution_id>")"
  echo "$response"
  echo "$response" | rg '"status":"(completed|failed)"' >/dev/null && break
  sleep 3
done
```

## Preview merged parent/child config

`POST /air8_integration/preview_merged_config`

Requires `parent_config_id` and `child_config_id`.

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/preview_merged_config" \
  -H "Content-Type: application/json" \
  -d '{"parent_config_id":"<id>","child_config_id":"<id>"}'
```

## Quality bar

- Reuse known `execution_id` when correlating uploads, extraction, downstream export, or search.
- Prefer OpenAPI-required fields explicitly set; omitting optional flags still applies server defaults documented in schemas.
