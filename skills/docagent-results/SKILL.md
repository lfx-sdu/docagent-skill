---
name: docagent-results
description: How to check document extraction, batch, and content-check results in DocuAgent. Use when a user says "check results", "see my extraction", "is it done", "find order X", or "results page".
---

# Check results (DocuAgent)

## If you are using the web app

1. Sign in to DocuAgent (Azure AD).
2. Open **Results** (`/results`).
3. Pick a tab:
   - **Extractions** — one document / one extraction run
   - **Batches** — many files submitted together
   - **Checks** — content-check (rules) runs
   - **Queue** — what is still waiting or running

### Find a specific run

On **Extractions**, filter by:

| Filter | Use when |
|--------|----------|
| Order ID | You know the business order number |
| Execution ID | You have the id from a trigger response or email |
| Field config | You only want runs for one workflow config |
| Status | You only want `completed`, `failed`, `processing`, etc. |
| Date range | You know roughly when it ran |

Open a row to see extracted fields, validation flags, and the document preview when available.

### Status meanings (Extractions / Checks)

| Status | Meaning |
|--------|---------|
| `pending` | Created, not started yet |
| `queued` | Waiting in queue |
| `processing` | Running now |
| `completed` | Finished — open the row for output |
| `failed` | Error — open the row for reason / retry if offered |

While status is `pending`, `queued`, or `processing`, refresh the list or wait; the UI loads stored results from DocuAgent when done.

### Share a result

From a completed extraction, use **Share** if the product shows it — that creates a share link (see execution service `POST …/:id/share`). Do not guess share URLs.

---

## If you are an agent helping a logged-in user

- Default answer: use **Results** in the app (steps above).
- Do **not** send them to [T4S Agents Swagger](https://api.uat.t4s.lfxdigital.app/agents/v1/docs) to "check results" — that API is for the processing engine, not the Results screen.
- Ask for **execution id** or **order id** if they cannot find the row.

---

## If you must call the API (automation / scripts)

DocuAgent stores results in the **execution service**. You need a **Bearer token** (same login as the UI), not the Config Agent `X-API-Key`.

**Base URL (examples):**

| Environment | Base |
|-------------|------|
| Local | `http://localhost:3000/v1` |
| UAT | `https://api.uat.doc-agent.lfxdigital.app/v1` |

### List extraction results

```http
GET {base}/execution/sdu-extraction-executions?orderId=ORDER-123&status=completed
Authorization: Bearer <token>
```

Useful query params: `orderId`, `executionId`, `fieldConfigId`, `status`, `fromDate`, `toDate`, `page`, `size`, `sortOrder`.

### One extraction by id

```http
GET {base}/execution/sdu-extraction-executions/{id}
Authorization: Bearer <token>
```

Response includes `status`, `requestBody`, and `output` when complete. `202` with a message and no `output` usually means still running.

### List content-check results

```http
GET {base}/execution/sdu-check-content-executions
Authorization: Bearer <token>
```

### List batch results

```http
GET {base}/execution/sdu-extraction-executions/batch
Authorization: Bearer <token>
```

### Queue snapshot (current user)

```http
GET {base}/execution/sdu-extraction-executions/queue-status
Authorization: Bearer <token>
```

---

## SDU-only polling (integrators, not the Results UI)

If you triggered extraction **directly** on the Agents API and only have `execution_id` from that response:

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

That returns SDU job status. The **DocuAgent Results page** uses MongoDB records under `/execution/sdu-extraction-executions`, not this GET, unless you are debugging the pipeline itself.

---

## Quick decision

| You want… | Do this |
|-----------|---------|
| See my extractions in the product | `/results` → **Extractions** |
| See batch or check runs | `/results` → **Batches** or **Checks** |
| Script against DocuAgent | `GET /execution/sdu-extraction-executions` + Bearer |
| Poll raw SDU after direct Agents POST | `GET …/air8_integration/check_execution_status` |
