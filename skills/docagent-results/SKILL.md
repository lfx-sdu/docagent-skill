---
name: docagent-results
description: Check DocuAgent extraction, batch, and content-check results. Use when the user says "check results", "see my extraction", "latest runs", "recent runs", "list extractions", "is it done", "find order X", or "results page". UAT NestJS base is https://uat.api.doc-agent.lfxdigital.app/v1 (not api.uat.doc-agent). Prefer browser/Kimi WebBridge on uat.doc-agent.lfxdigital.app/results when the agent host cannot resolve the API, or curl with DOCAGENT_BEARER_TOKEN (service key). Agents OpenAPI has no run list; with only execution_id use SDU check_execution_status. Never refuse with "I can't pull results from this workspace".
---

# Check results (DocuAgent)

**Source of truth:** upstream repo `doc-agent` (sibling checkout e.g. `/Users/erictaicp/work/doc-agent`):

| Area | Path |
|------|------|
| Results API client | `frontend/app/services/results/index.ts` |
| Share | `frontend/app/services/results/share.ts` |
| Types / filters | `frontend/app/models/results/index.ts` |
| GET-by-id + 422 | `frontend/app/services/documentExtraction/index.ts` |
| Auth / Bearer | `frontend/app/utils/axiosInstance.ts` |
| UAT API host (deploy) | `setup/scripts/ops/check-oapi.sh`, `.github/workflows/backend-exec-uat.yaml` → `uat.api.doc-agent.lfxdigital.app` |
| Product UX map | `setup/docs/guides/USERFLOW.md` |

Network capture on UAT confirms runtime behavior; **source code** adds endpoints, types, and edge cases the UI does not always call on first load.

## Hostnames (do not swap)

| Role | UAT | Prod (typical) |
|------|-----|----------------|
| **Web app** | `https://uat.doc-agent.lfxdigital.app` | `https://doc-agent.lfxdigital.app` (or tenant URL) |
| **NestJS execution API** | `https://uat.api.doc-agent.lfxdigital.app/v1` | `https://api.doc-agent.lfxdigital.app/v1` |

**Common mistake:** `https://api.uat.doc-agent.lfxdigital.app` — **does not resolve** (wrong subdomain order). The live UAT API uses **`uat.api`**, not `api.uat`.

**Optional `.env` (never commit):**

```env
DOCAGENT_BEARER_TOKEN=<service key from /login, sa-…>
DOCAGENT_NESTJS_BASE_URL=https://uat.api.doc-agent.lfxdigital.app/v1
DOCAGENT_ACTOR_EMAIL=<email>   # team SA only
```

`DOCAGENT_AGENTS_API_KEY` is **Config Agent only** (`/config_integration/*`) — not for Results.

---

## Two stacks (do not conflate)

| Stack | Who uses it | Base (UAT) | Auth |
|-------|-------------|------------|------|
| **Product / NestJS** | Web app, `curl`, automation | `https://uat.api.doc-agent.lfxdigital.app/v1` | Bearer: MSAL JWT, **service key** (`sa-…`), or team SA + `X-Actor-Email` |
| **Agents / SDU OpenAPI** | Config Agent; **status-only** agent shortcut | `https://api.uat.t4s.lfxdigital.app/agents/v1` | `X-API-Key` on `/config_integration/*` only; **`check_execution_status` has no auth** |

The **Results UI never calls** `/agents/v1/air8_integration/*`. It uses **`/execution/*`** on the NestJS gateway above.

---

## Three ways to get recent runs (pick one)

| Priority | When | How |
|----------|------|-----|
| **1 — Browser** | User has Chrome + WebBridge; or wrong DNS on agent host | Open `https://uat.doc-agent.lfxdigital.app/results` (Kimi WebBridge: navigate → sign in with service key if needed → read table or use `network` tool on `uat.api` XHRs). **Unauthenticated UI shows empty / login — not “no runs”.** |
| **2 — curl** | `DOCAGENT_BEARER_TOKEN` or JWT; host resolves | `GET {DOCAGENT_NESTJS_BASE_URL}/execution/sdu-extraction-executions?page=0&size=50&sortBy=createdOn&sortOrder=desc` + `Authorization: Bearer …` |
| **3 — SDU poll** | Only `execution_id`, no bearer | `GET …/air8_integration/check_execution_status?execution_id=` — **no list** |

Do **not** use `uat.doc-agent…/v1/execution/*` — that URL is the **Next.js frontend**, not the API (returns HTML).

---

## Network-verified: Results page load (UAT)

Captured from `https://uat.doc-agent.lfxdigital.app/results` (Extractions tab). All go to **`uat.api.doc-agent.lfxdigital.app`**.

| Call | Method | Path |
|------|--------|------|
| Auth probe | GET | `/execution/admin/auth/status` |
| List (50, `createdOn` desc) | GET | `/execution/sdu-extraction-executions?sortOrder=desc&page=0&size=50&sortBy=createdOn` |
| Pagination total | GET | `/execution/sdu-extraction-executions/records/count` |
| Job uptime banner | GET | `/execution/sdu-extraction-executions/me/job-uptime?days=14` |

**Other tabs (same API host):**

| Tab | `?tab=` | Extra GETs |
|-----|---------|------------|
| Batches | `batches` | `/execution/sdu-extraction-executions/batch` |
| Checks | `orders` | `/execution/sdu-check-content-executions`, `…/records/count` |
| Queue | `queue` | `/execution/sdu-extraction-executions/queue-status` |

List response is a **JSON array** of executions (`id`, `status`, `requestBody`, `output`, `createdOn`, …). UI maps `status` to COMPLETED / FAILED / QUEUED, etc.

### Endpoint details (per-call, UAT — network + curl)

Use WebBridge `network list` / `network detail` on `uat.api…` after reload or row actions. Below matches live traffic (May 2026).

| Endpoint | Method | Response / notes |
|----------|--------|------------------|
| `/execution/admin/auth/status` | GET | `{ isAdmin, role }` — session probe on load |
| `/execution/sdu-extraction-executions?…` | GET | **Array** of runs. Default query: `page=0&size=50&sortBy=createdOn&sortOrder=desc`. Filters: `orderId`, `executionId` (partial), `fieldConfigId`, `status`, `fromDate`, `toDate`. UI form may send `created_on_from` / `created_on_to` — client maps to `fromDate` / `toDate` in `getDocumentExtractionResults`. List fetch uses **AbortSignal** (cancel stale requests on filter change). |
| `/execution/sdu-extraction-executions/records/count?…` | GET | `{ totalRecordNum }` — same filter query as list |
| `/execution/sdu-extraction-executions/me/job-uptime?days=14` | GET | `{ buckets, totalJobs, totalFailed, fromLabel, toLabel }` — banner above tabs; UI refresh ~60s |
| `/execution/sdu-extraction-executions/queue-status` | GET | `{ activeExecutions, queuedExecutions, queuedExecutionIds, maxConcurrentExecutions, pendingInDatabase, processingInDatabase, totalWaiting, availableSlots, serviceBusQueueDepth, capacityPercentage, systemStatus, isOverCapacity, … }` — Queue tab; UI refresh ~15s |
| `/execution/sdu-extraction-executions/batch?page&size` | GET | `{ pagination: { total, page, size }, data: [...] }` — Batches tab (not a bare array) |
| `/execution/sdu-check-content-executions?…` | GET | Array (may be empty) — Checks tab |
| `/execution/sdu-check-content-executions/records/count` | GET | `{ totalRecordNum }` |
| `/execution/sdu-extraction-executions/{id}` | GET | Full record: `output`, `requestBody`, `createdOn`, `createdBy`, `metadata`, `fileSasUri`, … — used when opening output / polling a run |
| `/execution/sdu-extraction-executions/{id}/share` | GET | Existing share metadata if any (may be empty before share) |
| `/execution/sdu-extraction-executions/{id}/share` | POST | `{ shareToken, shareUrl, expiresAt }` — e.g. `shareUrl` → `https://uat.doc-agent.lfxdigital.app/share/{token}` |
| `/execution/sdu-extraction-executions/{id}/retry` | POST | Retry failed run (UI **Retry** button) |
| `/execution/sdu-export-data-executions` | POST | **Single row:** `{ "execution_id": "<uuid>" }` → **201** `{ id, execution_id, url, status, … }`. **Bulk (Export selected):** `{ "records_to_export": { "<execution_id>": "<order_id>", … }, "output_format": "xlsx" }` per `triggerExportData` in source. |
| `/execution/sdu-export-data-executions` | GET | List export jobs (Exports-related flows in product) |
| `/execution/sdu-extraction-executions/{id}/edited-output` | PATCH | Save edited extraction output |
| `/execution/sdu-extraction-executions/{id}/edited-output/discard` | PATCH | Discard edits |
| `/execution/sdu-extraction-executions/{id}/share` | DELETE | Disable sharing (`disableSharing`) |
| `/execution/sdu-extraction-executions/retry-all-pending` | POST | Bulk retry (60s timeout in client) |
| `/execution/sdu-extraction-executions/mark-all-pending-processing-as-failed` | POST | Queue admin action |
| `/execution/sdu-extraction-executions/mark-all-processing-as-failed` | POST | Queue admin action |
| `/execution/sdu-extraction-executions/batch/{batchId}` | DELETE | Delete batch |

**List item fields (typical):** `id`, `fieldConfigId`, `parentConfigId?`, `status` (`completed` \| `failed` \| `pending` \| …), `description` (fail message), `requestBody`, `output?`, `fileSasUri`, `createdOn`, `lastUpdatedOn`, `createdBy` (`sa-…`).

**GET-by-id vs list:** list rows may omit or truncate `output`; use `GET …/{id}` for full extraction JSON.

### Row actions — UI vs API

| UI control | Extra API? | Behavior |
|------------|------------|----------|
| **View payload** | **No** | Modal **“SDU Extraction Payload”** — built from row `requestBody` + `fileSasUri` (shows `execution_id`, `field_config_id`, `order_id`, `file_uri`, flags). |
| **See fail reason** | **No** | Same payload modal on failed rows; failure text is in row `description` / list payload. |
| **View actions** | Sometimes | Popover (icons); **Share** may `GET …/{id}/share` first. |
| **Share** | **GET** `…/share` then optional **POST** `…/share` | UI checks share status first; POST is used to enable/create share when needed. |
| **Retry** | **POST** `…/{id}/retry` | Failed rows only. |
| **Delete** | **DELETE** `…/{id}` | `deleteDocumentExtractionOutput` in source. |
| **Edit / discard output** | **PATCH** `…/edited-output` (+ `/discard`) | From output modal — not row list load. |
| **Disable share** | **DELETE** `…/{id}/share` | After share enabled. |
| **Export selected** / row export | **POST** `…/sdu-export-data-executions` | Returns downloadable `.xlsx` URL. |

**Toolbar (Extractions):** **Columns**, **Advanced filters** (date range, field config ID, status, created/last updated), **More Actions**, **Export selected** — filtered list re-fetches `GET …/sdu-extraction-executions` + `…/records/count` with query params.

**No extra Agents/SDU calls** for Results browsing — only `uat.api…/execution/*`.

---

## Connectivity troubleshooting

| Symptom | Meaning | What to do |
|--------|---------|------------|
| **NXDOMAIN** on `api.uat.doc-agent…` | Wrong hostname | Use **`uat.api.doc-agent.lfxdigital.app`**, or browser/WebBridge on `uat.doc-agent…/results`. |
| **401** on `api.doc-agent…` with UAT `sa-…` token | Token/env mismatch | Use **UAT** API base; prod token for prod only. |
| **Empty table** in browser | Not signed in | Sign in (Azure / email / **service key** on `/login`). |
| **`.env` only has `DOCAGENT_AGENTS_API_KEY`** | Config Agent only | Add **`DOCAGENT_BEARER_TOKEN`** (service key) for NestJS, or use browser. |
| Agent says “no list API” | Omission | NestJS list exists; SDU has status-only. |

---

## Agent playbook (required)

When the user asks for **recent runs**, **check results**, or **is it done**:

1. **Do not** open with “I can’t pull results from this workspace.”
2. **Try curl first** if `DOCAGENT_BEARER_TOKEN` is available — base **`https://uat.api.doc-agent.lfxdigital.app/v1`** (or `DOCAGENT_NESTJS_BASE_URL`).
3. **If API host fails from agent** → **Kimi WebBridge** (or tell user): `https://uat.doc-agent.lfxdigital.app/results`; service-key login; read table or capture `uat.api` network list.
4. **`execution_id` only, no bearer** → SDU `check_execution_status` immediately (no API key).
5. **`order_id` only** → list with bearer + filter `orderId`, or browser filter on Results.

```bash
# Recent extractions (UAT) — correct host
curl -sS -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  "${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}/execution/sdu-extraction-executions?page=0&size=50&sortBy=createdOn&sortOrder=desc"
```

```bash
# SDU status only (no list, no auth)
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<id>"
```

**UI polling** (not SDU): `GET …/execution/sdu-extraction-executions/{id}` every **2s** until `output` or **422**.

### Anti-patterns

| Bad | Do instead |
|-----|------------|
| `api.uat.doc-agent…` in curl | `uat.api.doc-agent…` |
| `curl` to `uat.doc-agent…/v1/execution/*` | API host is `uat.api…` |
| “No API for latest runs” | NestJS list + browser table |
| “Requires Azure, not service key” | Service key **is** Bearer on NestJS |
| Empty UI = no data | Login first |

---

## Results in the web app (`/results`)

**URL:** `https://uat.doc-agent.lfxdigital.app/results` (UAT)

**Post-login default:** `/results` unless `returnUrl` / trial deep link.

### Tabs (`?tab=`)

| `?tab=` | Label | Content |
|---------|-------|---------|
| `documents` (default) | Extractions | Single-document runs |
| `batches` | Batches | Batch jobs |
| `orders` | Checks | Content-check runs |
| `queue` | Queue | Live queue |

**No Exports tab** — Excel is a row action (`POST …/sdu-export-data-executions`).

### Extractions tab

- **50** rows/page, **`createdOn` desc**
- Filters debounce **500ms**; list + **count** share criteria
- Deep link: `/results?tab=documents&fieldConfigId=<id>&executionId=<id>`

| Filter | API param |
|--------|-----------|
| Order ID | `orderId` |
| Execution ID | `executionId` (partial) |
| Field config | `fieldConfigId` |
| Status | `status` |
| Dates | `fromDate`, `toDate` |

**Status:** `pending` + `queuedAt` → UI shows **QUEUED**. List does **not** auto-poll — use **Queue** tab or refresh.

### Polling intervals (UI)

| Feature | Interval | Endpoint |
|---------|----------|----------|
| Running extraction | 2s | `GET …/sdu-extraction-executions/{id}` |
| Queue tab | 15s | `GET …/queue-status` |
| Job uptime | 60s | `GET …/me/job-uptime?days=14` |

---

## NestJS execution API reference

**Auth** (same as `axiosInstance.ts`):

1. **Team:** `Authorization: Bearer <SA>` + `X-Actor-Email`
2. **User / service key:** `Authorization: Bearer <token>`
3. **Public:** `GET …/execution/public/sdu-extraction-executions/share/{token}`

### Core endpoints

```http
GET {base}/execution/sdu-extraction-executions?page=0&size=50&sortBy=createdOn&sortOrder=desc
GET {base}/execution/sdu-extraction-executions/{id}
GET {base}/execution/sdu-extraction-executions/records/count
GET {base}/execution/sdu-extraction-executions/batch
GET {base}/execution/sdu-check-content-executions
GET {base}/execution/sdu-extraction-executions/queue-status
GET {base}/execution/admin/auth/status
```

| Action | Method | Path |
|--------|--------|------|
| Retry | POST | `…/sdu-extraction-executions/{id}/retry` |
| Export Excel | POST | `…/sdu-export-data-executions` body `{ execution_id }` |
| Share | POST | `…/sdu-extraction-executions/{id}/share` |

---

## SDU status poll (agent shortcut only)

Not used by Results UI.

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<id>"
```

---

## Quick decision

| User wants… | Action |
|-------------|--------|
| Recent runs | Browser `/results` **or** `GET …/sdu-extraction-executions` on **`uat.api…`** |
| Known `execution_id`, no JWT | SDU `check_execution_status` |
| Full output | NestJS `GET …/{id}` or UI row |
| Queue | `?tab=queue` or `queue-status` |
| Config jobs | `docagent-platform` + `X-API-Key` |
