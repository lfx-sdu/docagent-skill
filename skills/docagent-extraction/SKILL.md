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

Team SA: add `X-Actor-Email: <email>` with the service key.

---

## Country and Document type (vary per `field_config_id`)

The UI labels **Country** and **Document type** are **not** global enums. They map to API fields on the execution request and must be chosen **from the selected field config only**.

| UI label | API field | Where it comes from |
|----------|-----------|---------------------|
| **Country** | `nation` | Top-level key under `fieldConfig` for this `field_config_id` |
| **Document type** | `possible_doc_type` | Document **section** under that country (and/or an accepted `doc_family` label — use the section name when unsure) |

### `fieldConfig` shape (per config)

Each config defines its own tree. **Another config’s country/doc-type strings will not work.**

```text
fieldConfig
└── <Country>                 ← send as nation (exact spelling/casing)
    └── <Document section>    ← usual value for possible_doc_type
        ├── doc_family: [...]
        └── key_fields: [...]
```

Example (`sample field config [Immutable]`):

```text
fieldConfig
├── General
│   └── Invoice          → nation: "General",  possible_doc_type: ["Invoice"]
├── China
│   └── Bank Statement   → nation: "China",     possible_doc_type: ["Bank Statement"]
└── India
    └── GST Certificate  → nation: "India",     possible_doc_type: ["GST Certificate"]
```

### Rules agents must follow

1. **Pick `field_config_id` first**, then derive Country + Document type from **that** config’s `GET /config/sdu-field-configs/{id}` response.
2. **Never reuse** `nation` / `possible_doc_type` from a previous run when the user changes config — e.g. `CN` vs `china` vs `General` are different keys on different configs.
3. **Do not assume ISO codes** — `CN`, `US`, `USA` only work if that exact string is a top-level `fieldConfig` key (usually it is not).
4. **`doc_family` typos ≠ API doc type** — e.g. `doc_family` may list `"Inovice"` but `possible_doc_type` may still need `"Invoice"` (agent normalizes typos).
5. **Empty country branch = no types** — e.g. `invoice child 2` has `fieldConfig.china: {}` → `nation: "china"` validates as a country but **no** document types exist there.
6. **Parent/child configs** — valid pairs come from the **effective** schema after merge (`use_parent_config` / `parent_config_id`). If the child’s region is empty, parent fields must supply that region or extraction fails.

### Same document, different configs (UAT examples)

| User intent | Config | `field_config_id` | Valid `nation` | Valid `possible_doc_type` |
|-------------|--------|-------------------|----------------|---------------------------|
| China invoice (demo) | sample field config [Immutable] | `48fec1d9-…` | `General` | `Invoice` |
| China invoice (stub) | invoice child 2 | `6fede801-…` | `china` only (not `CN`) | **none** on UAT — `china` is `{}` |
| Factory audit | FAA General Full v2 | `ai-gen-53c9ecc0-…` | `general` (check config) | `factory_accreditation_assessment` (section key) |
| Shipping variant | tjx_shipvariant | `f9a73eac-…` | from parent `parent_shipdoc` | `invoice` (lowercase — **this config’s** label) |

When the user asks “what country / doc type?”, answer **for the config they selected**, not from memory.

### List valid Country + Document type pairs (required script)

Run after choosing `field_config_id`:

```bash
curl -sS -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  "${DOCAGENT_NESTJS_BASE_URL}/config/sdu-field-configs/<field_config_id>" \
  | python3 -c "
import json, sys
c = json.load(sys.stdin)
fc = c.get('fieldConfig') or {}
print('config:', c.get('name'), '| id:', c.get('id'))
print('parent_id:', c.get('parent_id'))
print()
if not fc:
    print('WARNING: fieldConfig is empty — no valid Country/Document type pairs')
for country, sections in fc.items():
    if not isinstance(sections, dict) or not sections:
        print(f'Country: {country!r}  (no document sections — skip)')
        continue
    for doc_section, body in sections.items():
        if not isinstance(body, dict):
            continue
        families = body.get('doc_family') or []
        has_fields = bool(body.get('key_fields'))
        print(f'Country: {country!r}')
        print(f'  Document type (API): {doc_section!r}  fields={has_fields}')
        if families:
            print(f'    doc_family: {families[:4]}')
print()
print('Use nation = Country column; possible_doc_type = [Document type]')
"
```

---

## Before you run (preflight)

Do these **before** creating an execution. Skipping them causes misleading 422 errors (often reported as “unsupported file type”).

1. **List configs** — `GET /config/sdu-field-configs?page=0&size=100&sortOrder=desc`
2. **Pick `field_config_id`** — match document domain (invoice, FAA, shipping, etc.).
3. **List valid Country + Document type pairs** — run the script above for **that** config only; do not copy values from another config.
4. **Confirm sections have `key_fields`** — empty `{}` regions fail even when the PDF is valid.
5. **Set `file_name`** — ASCII filename in the create body (e.g. `277-invoice.pdf`).
6. **Upload with an ASCII multipart filename** — `file=@/path/doc.pdf;filename=277-invoice.pdf`
7. **Prefer `flash_mode: true`** for API runs (matches successful UAT automation; UI may differ).

### UAT invoice configs (verified)

| Config | ID | Use for invoices? |
|--------|-----|-------------------|
| `invoice child 2` | `6fede801-e1d6-41a3-9cdb-eaaef46b8605` | **No** — `fieldConfig.china` is empty; fails with “no document types registered for china” |
| `invoice_child` | `2da73f95-f3a4-43c6-b20a-46a4d066507c` | **No** — empty; parent mock has no fields |
| `sample field config [Immutable]` | `48fec1d9-fd18-4c8e-8f03-0c4032870927` | **Yes (demo)** — `nation: "General"`, `possible_doc_type: ["Invoice"]` |

For production invoice / 专用发票 fields, use a **populated** config from the user’s tenant (UI dropdown), not empty UAT stubs.

---

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

---

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
    "file_name": "<ascii-filename.pdf>",
    "enable_image_preprocessing": true,
    "validate_doc_type": false,
    "flash_mode": true,
    "use_parent_config": false,
    "external_context": "<optional free text>",
    "parent_config_id": "<optional explicit parent>"
  }' \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/sdu-extraction-executions"
```

Expected response includes `id` and `fileSasUri`.

- Set `use_parent_config: true` only when the **parent** config has non-empty `fieldConfig` (child-only stubs will fail).
- `nation` / `possible_doc_type` must be valid **for this `field_config_id`** (see [Country and Document type](#country-and-document-type-vary-per-field_config_id)).

### Step 2: upload file to blob

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  -F "sasUri=<fileSasUri_from_step_1>" \
  -F "file=@/absolute/path/to/document.pdf;type=application/pdf;filename=<ascii-filename.pdf>" \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/workflows/blob-upload"
```

Expect `{"success":true,...}`. Upload must finish before the agent reads the blob.

### Step 3: poll execution

```bash
curl -sS \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/sdu-extraction-executions/<id>"
```

Poll every **~2s** until `output` is present (**HTTP 200** with `output` array/object). Do not stop at HTTP 202 alone.

---

## Example: China payment form + invoices (UAT demo config)

Payment-application PDFs with embedded 全电子发票 / 专用发票 references:

```json
{
  "field_config_id": "48fec1d9-fd18-4c8e-8f03-0c4032870927",
  "order_id": "TEST-277",
  "nation": "General",
  "possible_doc_type": ["Invoice"],
  "file_name": "277-invoice.pdf",
  "flash_mode": true,
  "validate_doc_type": false,
  "use_parent_config": false
}
```

Do **not** use `nation: "CN"` with `invoice child 2` — that config requires `nation: "china"` and still fails today because `china` has no registered doc types on UAT.

---

## Merge-and-trigger flow (2+ files)

Use when the user uploads multiple PDF/image files and wants server-side merge + extraction. **Requires at least 2 files**; single file uses the 3-step flow above.

```bash
curl -sS -X POST \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  -F "field_config_id=<field_config_id>" \
  -F "order_id=<order_id>" \
  -F "nation=<nation>" \
  -F 'possible_doc_type=["<doc_type>"]' \
  -F "file_name=<ascii-filename.pdf>" \
  -F "enable_image_preprocessing=true" \
  -F "validate_doc_type=false" \
  -F "flash_mode=true" \
  -F "use_parent_config=false" \
  -F "external_context=<optional>" \
  -F "files=@/absolute/path/to/file1.pdf;filename=file1.pdf" \
  -F "files=@/absolute/path/to/file2.pdf;filename=file2.pdf" \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/sdu-extraction-executions/merge-and-trigger"
```

Then poll `GET .../sdu-extraction-executions/{id}`.

---

## Inheritance and context rules

- `use_parent_config=true`: merge parent field definitions from `parent_id` on the child config.
- `parent_config_id`: explicit parent override; takes precedence over `use_parent_config`.
- Omit both or set `use_parent_config: false` when the child is empty and the parent has no fields (common on UAT test configs).
- `external_context`: optional notes/reference text blended into extraction prompt.

---

## Polling and failure handling

- Poll interval in product UI: **2 seconds**.
- Success: **HTTP 200** and non-null `output`.
- In progress: **HTTP 202** — keep polling.
- Failed: **HTTP 422** — read the full `message`; do not guess from the first line.

### 422 messages (read the real cause)

| Message pattern | Likely fix |
|-----------------|------------|
| `Nation 'CN' not found` … `Only 'china' is available` | Wrong country **for that config** — re-list pairs; use `china`, not `CN` |
| `document type 'invoice' is not found under nation 'china'` | Country exists but **no types** under it for this config; pick another config/section or populate `fieldConfig` |
| `document type 'Inovice' was not recognized` … should be `'Invoice'` | Use this config’s document type label (`Invoice`), not a `doc_family` typo |
| User changed config but kept old country/type | Re-run list script on the **new** `field_config_id` — values are not portable across configs |
| `unsupported file type` (with valid PDF/PNG) | Often **config/nation/doc-type** or empty `fieldConfig`; re-run preflight. Re-check blob upload `success: true`. |
| `At least 2 files are required for merge` | Use single-file flow, not `merge-and-trigger` |

SDU `check_execution_status` is **not** a substitute for NestJS poll when you have a bearer token; it may 404 for NestJS-only execution IDs.

Bearerless status-only fallback:

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

After success, link the user to `https://uat.doc-agent.lfxdigital.app/results` and report `execution_id`, `order_id`, and key extracted fields.

---

## Quick decision

| User request | Action |
|--------------|--------|
| "Extract this file" | Preflight config → single-file 3-step flow |
| "Which config / country / doc type?" | `GET` config by id; print valid **(Country → Document type)** pairs for that id only |
| "Merge these files and extract" | `merge-and-trigger` (2+ files) |
| "Run/process my batch" | Batch create/process/get endpoints |
| "Is extraction done?" with bearer | Poll `GET .../sdu-extraction-executions/{id}` until `output` |
| "Is extraction done?" with only execution_id | SDU `check_execution_status` (may 404 for NestJS runs) |
