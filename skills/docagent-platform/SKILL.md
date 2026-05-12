---
name: docagent-platform
description: Routes DocuAgent Agents API work by OpenAPI tag and user intent. Use when choosing which skill applies, setting base URL and auth (X-API-Key for ConfigAgent), polling async jobs, or staying endpoint-first against https://api.uat.t4s.lfxdigital.app/agents/v1.
---

# DocuAgent platform (Agents API router)

## Canonical base URL

- Set `DOCAGENT_AGENTS_API_BASE_URL` (UAT: `https://api.uat.t4s.lfxdigital.app/agents/v1`).
- All paths below are relative to that base.
- Discover the full contract: `GET /openapi.json` or [Swagger UI](https://api.uat.t4s.lfxdigital.app/agents/v1/docs).

## Auth

- **`/config_integration/*`**: OpenAPI marks these with `APIKeyHeader`; send `X-API-Key: $DOCAGENT_AGENTS_API_KEY`.
- **Other groups** (`/air8_integration/*`, `/search_integration/*`, `/ner_integration/*`): follow your deployment; OpenAPI may omit `security`. Do not invent headers—use what your environment requires.

## Preflight

```bash
curl -sS "$DOCAGENT_AGENTS_API_BASE_URL/health"
```

## Decision tree (pick domain skill)

1. **Extract / validate documents** (`POST .../validate_and_extract_docs`, execution status) → `docagent-extraction`.
2. **Content checks / rules** (`POST .../check_doc_content`) → `docagent-content-check`.
3. **Export dataframe to blob OR extraction Excel URL** (`export_data_to_blob`, `export-extraction-excel`) → `docagent-export-results`.
4. **Config chat, uploads, shipment mapping, embeddings, global config jobs** → `docagent-config-agent`. Multipart uploads / PDF merge → `docagent-file-prep` (focused recipes).
5. **Poll generic `execution_id`** across Air8 flows → `docagent-results`.
6. **Company research reports** (`search_company_info_and_news`, `get_company_info_and_news_by_id`) → `docagent-company-research`.
7. **Supplier/buyer/factory/product search** (`/search_integration/*`) → `docagent-search`.
8. **NER trace / suggest / pipelines** (`/ner_integration/*`) → `docagent-ner`.
9. **Service health / memory diagnostics only** (`/`, `/health`, `/memory`) → `docagent-admin-kpis`.

## Request discipline

- Prefer **`curl`** or the user’s HTTP client with explicit JSON bodies from `openapi.json` component schemas.
- For **multipart** endpoints, use `-F` / client multipart APIs; never guess field names—read OpenAPI `multipart/form-data` schema.
- On **`422 Validation Error`**, read `detail` and fix the body before retrying.

## Async patterns

- **Air8**: responses include `execution_id` and `status`; poll `GET /air8_integration/check_execution_status?execution_id=...`.
- **Config generation**: `POST /config_integration/generate-global-config` returns `job_id`; poll `GET /config_integration/config-job/{job_id}` until terminal status.
- **Embeddings**: start endpoints return `job_id`; poll `.../shipment-code-embeddings-status/{job_id}` or `.../invoice-code-embeddings-status/{job_id}`.
- **NER pipelines**: start returns `task_id`; poll `.../entity-pipeline/status/{task_id}` or `.../product-pipeline/status/{task_id}`.

Backoff: start at 1–2s, cap around 10–15s, stop after a reasonable wall-clock budget or when status is failed.

## Safety

- **DELETE** on `/search_integration/*/{search_id}` removes data in backing stores per OpenAPI descriptions. Only call with explicit user confirmation.
- Do not log API keys or full document payloads in chat transcripts.

## Endpoint coverage map (all 57 paths)

| Group | Skill(s) |
|-------|----------|
| Health `/`, `/health`, `/memory` | `docagent-admin-kpis` |
| `Air8` | `docagent-extraction`, `docagent-content-check`, `docagent-export-results`, `docagent-company-research`, `docagent-results` (status) |
| `ConfigAgent` | `docagent-config-agent`, `docagent-export-results` (Excel), `docagent-file-prep` |
| `LFSearch` | `docagent-search`, `docagent-results` (GET by id) |
| `NER` | `docagent-ner` |
