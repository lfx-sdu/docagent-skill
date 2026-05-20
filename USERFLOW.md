# USERFLOW — Check results and run extraction

**Agents:** read `skills/docagent-results/SKILL.md` first.

**UAT API base (network-verified):** `https://uat.api.doc-agent.lfxdigital.app/v1` — **not** `api.uat.doc-agent…` (NXDOMAIN).

**UAT web app:** `https://uat.doc-agent.lfxdigital.app/results`

---

## Two API stacks

| Stack | Base (UAT) | Used for Results |
|-------|------------|------------------|
| **NestJS (product)** | `https://uat.api.doc-agent.lfxdigital.app/v1` | List, poll, share, queue |
| **Agents / SDU** | `https://api.uat.t4s.lfxdigital.app/agents/v1` | `check_execution_status` only (no list) |

The web app calls **`uat.api…`**, not the SDU Agents host, for `/results`.

---

## Flow A — Browser (human or Kimi WebBridge)

1. Open `https://uat.doc-agent.lfxdigital.app/results` (or `/login` first).
2. Sign in: Azure AD, email, or **service key** (`sa-…` on `/login`).
3. **Extractions** tab: recent runs (50/page, newest first). Empty table before login ≠ no runs.
4. Other tabs: `?tab=batches|orders|queue`.

**Agent tip:** use WebBridge `network` on reload — XHRs go to `uat.api.doc-agent.lfxdigital.app/v1/execution/…`.

---

## Flow B — curl with service key (automation)

```bash
set -a && source .env && set +a   # DOCAGENT_BEARER_TOKEN=sa-…

curl -sS -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  "https://uat.api.doc-agent.lfxdigital.app/v1/execution/sdu-extraction-executions?page=0&size=50&sortBy=createdOn&sortOrder=desc"
```

Optional: `DOCAGENT_NESTJS_BASE_URL`, `DOCAGENT_ACTOR_EMAIL` (team SA).

**Wrong hosts:**

- `api.uat.doc-agent…` → NXDOMAIN
- `uat.doc-agent…/v1/execution/…` → HTML from Next.js, not JSON API

---

## Flow C — Agent has `execution_id` only (no JWT)

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<id>"
```

Poll 2–5s until terminal. For full `output` → Flow B `GET …/sdu-extraction-executions/{id}` or Flow A.

---

## Flow D — Share link (public)

1. `POST …/sdu-extraction-executions/{id}/share` → `shareUrl`
2. Reader: `/share/{token}` or `GET …/execution/public/sdu-extraction-executions/share/{token}`

---

## Flow E — Document extraction (`/document-extraction`)

UI page: `https://uat.doc-agent.lfxdigital.app/document-extraction`

Full agent playbook: `skills/docagent-extraction/SKILL.md`.

### E0. Preflight (do first)

1. Select **`field_config_id`** (config drives everything below).
2. `GET /config/sdu-field-configs/{id}` — list valid **Country → Document type** pairs from `fieldConfig` top-level keys and section names (**not** global: `CN`/`invoice` on one config ≠ `General`/`Invoice` on another).
3. UI **Country** → API `nation`; UI **Document type** → API `possible_doc_type` — both must match **this config only** (re-list if the user changes config).
4. Include `file_name` (ASCII) in create body; use `;filename=<ascii>.pdf` on multipart upload.
5. API runs: prefer `flash_mode: true`, `validate_doc_type: false`.
6. UAT `invoice child 2` / `invoice_child` are empty stubs — use a populated config or the tenant config from the UI.

### E1. Single upload (1 file)

1. `POST /execution/sdu-extraction-executions` (create execution + `fileSasUri`)
2. `POST /execution/workflows/blob-upload` (multipart: `sasUri`, `file`)
3. `GET /execution/sdu-extraction-executions/{id}` every 2s until **HTTP 200** and `output` (202 = still running)

Optional body flags:
- `use_parent_config` (only if parent has field definitions)
- `parent_config_id` (explicit parent override, takes precedence)
- `external_context` (additional context text)

### E2. Merge and trigger (2+ PDF/image files)

1. `POST /execution/sdu-extraction-executions/merge-and-trigger` (multipart; files + fields)
2. `GET /execution/sdu-extraction-executions/{id}` every 2s until `output`

### E3. Batch upload tab

1. `POST /execution/sdu-extraction-executions/batch`
2. `POST /execution/sdu-extraction-executions/batch/{batchId}/process`
3. `GET /execution/sdu-extraction-executions/batch/{batchId}` (poll)

### E4. Failure semantics

- `GET .../{id}` returning **422** = failed execution; stop polling and read the full `message` (often nation/doc-type/config, not a bad PDF).
- “Unsupported file type” with a valid PDF → re-check preflight (E0) and blob upload `success: true`.
- If only `execution_id` is available and no bearer token, fallback:
  `GET https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<id>`

---

## Flow F — Configuration updates (`/configurations`)

UI page: `https://uat.doc-agent.lfxdigital.app/configurations`

### F1. Field configurations tab (`?tab=fieldConfig`)

Load/list:

- `GET /config/sdu-field-configs?page=0&size=50&sortOrder=desc&roots_only=true`
- `GET /config/sdu-field-configs/records/count?roots_only=true`

Edit/create/delete:

- `POST /config/sdu-field-configs`
- `PATCH /config/sdu-field-configs/{id}`
- `DELETE /config/sdu-field-configs/{id}`
- `GET /config/sdu-field-configs/{id}` (editor hydration)
- `GET /config/sdu-field-configs/{id}/children` (tree expansion)
- `PATCH /config/sdu-field-configs/{id}/parent` with `{ "parent_id": "..." }` (reparent root)

### F2. Checker configurations tab (`?tab=checkerConfig`)

Load/list:

- `GET /config/sdu-checker-configs?page=0&size=25&sortOrder=desc`
- `GET /config/sdu-checker-configs/records/count`

Edit/create/delete:

- `POST /config/sdu-checker-configs`
- `PATCH /config/sdu-checker-configs/{id}`
- `DELETE /config/sdu-checker-configs/{id}`
- `GET /config/sdu-checker-configs/{id}`

### F3. Safety and verification

- These endpoints mutate live configuration data; confirm intent before `PATCH`/`DELETE`/reparent.
- Recommended sequence: get current config -> apply update -> re-fetch list/count and get-by-id.
- Use Bearer auth (`DOCAGENT_BEARER_TOKEN` or JWT). Do not use `DOCAGENT_AGENTS_API_KEY` here.

---

## Results page — network on load (UAT)

| GET | Path |
|-----|------|
| Auth | `/execution/admin/auth/status` |
| List | `/execution/sdu-extraction-executions?sortOrder=desc&page=0&size=50&sortBy=createdOn` |
| Count | `/execution/sdu-extraction-executions/records/count` |
| Uptime | `/execution/sdu-extraction-executions/me/job-uptime?days=14` |

**Batches:** `…/sdu-extraction-executions/batch`  
**Checks:** `…/sdu-check-content-executions` (+ count)  
**Queue:** `…/queue-status`

---

## Auth cheat sheet

| Route | Auth |
|-------|------|
| Results UI | Session after Azure / email / service key |
| NestJS `/execution/*` | Bearer (`sa-…` or JWT); team SA + `X-Actor-Email` |
| SDU `check_execution_status` | None |
| `/config_integration/*` | `X-API-Key: $DOCAGENT_AGENTS_API_KEY` |

---

## Preflight

```bash
curl -sS -o /dev/null -w "%{http_code}\n" "https://api.uat.t4s.lfxdigital.app/agents/v1/health"
curl -sS -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  "https://uat.api.doc-agent.lfxdigital.app/v1/execution/admin/auth/status"
```

Expect `200` when token and host match UAT.
