---
name: docagent-extraction
description: Start and monitor DocuAgent document extraction from the product flow at /document-extraction. Use when users ask to extract documents, run single-file extraction, merge multiple files, upload to blob, poll execution status, use parent/child field config inheritance, or pass external_context. Prefer NestJS execution API on uat.api.doc-agent.lfxdigital.app/v1; use SDU check_execution_status only as fallback when only execution_id is available and no bearer token.
---

# Document extraction (DocuAgent)

**Source of truth:** `doc-agent` repo (`frontend/app/services/documentExtraction/index.ts`, `frontend/app/components/documentExtractionCard/uploadDocumentCard/ApiFlowPanel.tsx`, `frontend/app/components/documentExtractionCard/index.tsx`).

## Runtime surfaces (UAT)

| Surface | URL | Purpose |
|---------|-----|---------|
| Web app | `https://uat.doc-agent.lfxdigital.app/document-extraction` | UI workflow (Single upload / Batch upload) |
| NestJS execution API | `https://uat.api.doc-agent.lfxdigital.app/v1` | Create execution, blob upload, polling, merge/batch APIs |
| SDU fallback | `https://api.uat.t4s.lfxdigital.app/agents/v1` | Status-only polling by `execution_id` when no bearer |

Do not use `api.uat.doc-agent...` (wrong hostname order). Use `uat.api...`.

## Auth and env

Use Bearer auth for NestJS execution routes:

```env
DOCAGENT_BEARER_TOKEN=<service key from /login or JWT>
DOCAGENT_NESTJS_BASE_URL=https://uat.api.doc-agent.lfxdigital.app/v1
DOCAGENT_ACTOR_EMAIL=<email>  # team SA mode only
```

`DOCAGENT_AGENTS_API_KEY` is not used for extraction execution routes.

## Product flow map (`/document-extraction`)

The UI exposes two modes:

1. **Single upload**
   - `POST /execution/sdu-extraction-executions`
   - `POST /execution/workflows/blob-upload` (multipart)
   - `GET /execution/sdu-extraction-executions/{id}` (poll every 2s)
2. **Multi-file merge (2+ PDF/image files)**
   - `POST /execution/sdu-extraction-executions/merge-and-trigger` (multipart)
   - `GET /execution/sdu-extraction-executions/{id}` (poll every 2s)
3. **Batch upload tab**
   - `POST /execution/sdu-extraction-executions/batch`
   - `POST /execution/sdu-extraction-executions/batch/{batchId}/process`
   - `GET /execution/sdu-extraction-executions/batch/{batchId}`

### Network-verified on page load (UAT)

- `GET /config/sdu-field-configs?page=0&size=1000&sortOrder=desc` (populate field config selector)

## Single-file API flow

### Step 1: create execution

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "field_config_id": "<field_config_id>",
    "order_id": "<order_id>",
    "nation": "<nation>",
    "possible_doc_type": ["<doc_type>"],
    "enable_image_preprocessing": true,
    "validate_doc_type": false,
    "flash_mode": false,
    "use_parent_config": true,
    "external_context": "<optional free text>",
    "parent_config_id": "<optional explicit parent>"
  }' \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/sdu-extraction-executions"
```

Expected response includes `id` and `fileSasUri`.

### Step 2: upload file to blob

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  -F "sasUri=<fileSasUri_from_step_1>" \
  -F "file=@/absolute/path/to/document.pdf" \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/workflows/blob-upload"
```

### Step 3: poll execution

```bash
curl -sS \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/sdu-extraction-executions/<id>"
```

Poll every ~2s until `output` exists.

## Merge-and-trigger flow (2+ files)

Use when the user uploads multiple PDF/image files and wants server-side merge + extraction.

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  -F "field_config_id=<field_config_id>" \
  -F "order_id=<order_id>" \
  -F "nation=<nation>" \
  -F 'possible_doc_type=["<doc_type>"]' \
  -F "enable_image_preprocessing=true" \
  -F "validate_doc_type=false" \
  -F "flash_mode=false" \
  -F "use_parent_config=true" \
  -F "external_context=<optional>" \
  -F "files=@/absolute/path/to/file1.pdf" \
  -F "files=@/absolute/path/to/file2.pdf" \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/sdu-extraction-executions/merge-and-trigger"
```

Then poll `GET .../sdu-extraction-executions/{id}`.

## Inheritance and context rules

- `use_parent_config=true`: resolve parent from child config relationship.
- `parent_config_id`: explicit parent override; takes precedence over `use_parent_config`.
- Omit both if no parent merge is needed.
- `external_context`: optional notes/reference text blended into extraction prompt.

## Polling and failure handling

- Poll interval in product UI: **2 seconds**.
- If `GET .../{id}` returns **422**, treat as failed and stop polling.
- For bearerless status checks with only `execution_id`, use SDU fallback:

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

## Quick decision

| User request | Action |
|--------------|--------|
| "Extract this file" | Single-file 3-step flow |
| "Merge these files and extract" | `merge-and-trigger` flow |
| "Run/process my batch" | Batch create/process/get endpoints |
| "Is extraction done?" with bearer | Poll `GET .../sdu-extraction-executions/{id}` |
| "Is extraction done?" with only execution_id | SDU `check_execution_status` |

