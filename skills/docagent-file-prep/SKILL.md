---
name: docagent-file-prep
description: Calls ConfigAgent multipart endpoints prepare-and-upload and merge-pdf using X-API-Key plus execution_id. Use when uploading blobs, merging PDFs, or prepping files before extraction—per OpenAPI Body_* multipart schemas.
---

# DocuAgent file prep (multipart, ConfigAgent)

**Auth:** `X-API-Key: $DOCAGENT_AGENTS_API_KEY`.

Base: `$DOCAGENT_AGENTS_API_BASE_URL`.

OpenAPI models: `Body_prepare_and_upload_endpoint_config_integration_prepare_and_upload_post`, `Body_merge_pdf_endpoint_config_integration_merge_pdf_post` — fields: `execution_id` (required), optional `files` (binary array), optional `base64_inputs` (array of strings).

## Upload / prepare blob URL

`POST /config_integration/prepare-and-upload` — `multipart/form-data`

Behavior (from OpenAPI description): single PDF/Excel/image uploads directly; single `.ai` compresses then uploads; multi PDF/image/.ai merges to one PDF; mixed multi-Excel rejected with 400.

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/prepare-and-upload" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  -F execution_id='<execution-id>' \
  -F files=@./document.pdf
```

Response type: `FileUploadResponse` — includes `blob_url` for downstream `DocsValidationRequest.file_uri`.

## Merge PDFs server-side

`POST /config_integration/merge-pdf` — multipart

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/merge-pdf" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  -F execution_id='<execution-id>' \
  -F files=@./part1.pdf \
  -F files=@./part2.pdf
```

Response: `PdfMergeResponse` with `pdf_base64` and `size_bytes`.
