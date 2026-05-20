---
name: docagent-results
description: Check DocuAgent extraction, batch, and content-check results. Use when the user says "check results", "see my extraction", "is it done", "find order X", or "results page". Agents must run API calls when they have execution_id; never refuse with "I can't pull results from this workspace".
---

# Check results (DocuAgent)

## Agent playbook (required)

When the user asks to **check results**, **see an extraction**, or **is it done**:

1. **Do not** open with "I can't pull your results from here" or blame the workspace/skills repo.
2. **If they gave `execution_id`** (or it appears in chat/logs): **call SDU immediately** — no Azure login, no `DOCAGENT_AGENTS_API_KEY` required:

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

Report `status` back. If `completed` and the response body includes output fields, summarize them. If still running, say so and offer to poll again.

3. **If they gave `order_id` but no `execution_id`**: ask for **one** execution id from the trigger email/UI row, **or** offer the Results UI path below. Do not stop after explaining auth gaps.

4. **`DOCAGENT_AGENTS_API_KEY`** is only for **`/config_integration/*`**. It does **not** list Results rows and is **not** an excuse to skip step 2 when you have `execution_id`.

5. **NestJS Results API** (`GET /execution/sdu-extraction-executions`) needs a **Bearer JWT** (same as the web app). Use it only when the user can supply a token or you are explicitly automating against DocuAgent backend — not as the first reply to "check my results".

6. **DNS / network errors** on `api.uat.doc-agent.lfxdigital.app`: retry or note environment limits; **still use** `api.uat.t4s.lfxdigital.app` for `check_execution_status` when you have `execution_id`.

### Anti-patterns (never do this)

| Bad response | Do instead |
|--------------|------------|
| "This workspace only has skills + API key, not your results" | Run `check_execution_status` if you have `execution_id` |
| "Extraction results live behind Azure AD, not that key" | Keys are unrelated; SDU status GET needs neither key nor JWT |
| "Send me execution id so I can tell you the curl command" | **Run** the curl (or equivalent) and interpret the JSON |
| Default to Swagger / "integrators only" | SDU poll is the normal agent path when `execution_id` is known |
| List UI tabs before trying API when id is already in chat | Poll first, UI second |

---

## Three ways to see results (pick by what you have)

| You have | Auth | What you get |
|----------|------|----------------|
| **`execution_id`** | None (SDU OpenAPI has no global `security`) | Pipeline `status` via Agents API — **use this first in agent chats** |
| **Azure AD session** (human in browser) | User login | Full Results UI: fields, preview, filters — `/results` |
| **Bearer JWT** + order/execution filters | `Authorization: Bearer …` | Full stored record: `output`, `requestBody`, list/filter — NestJS execution service |

Config Agent (`X-API-Key`) is a **fourth** surface for config chat/jobs only — not for listing extraction results.

---

## SDU status poll (Agents API) — primary for agents

**Endpoint:** `GET /air8_integration/check_execution_status?execution_id=<id>`

**Root:** `https://api.uat.t4s.lfxdigital.app/agents/v1`

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

Typical response shape: `execution_id`, `status` (`pending` | `processing` | `completed` | `failed`, per deployment).

- **404 / 500 with "No execution found"** → wrong id, typo, or wrong environment (UAT vs prod).
- **Still running** → backoff 2–5s and poll again; cap total wait per user patience.
- **Completed** → return status; if the payload includes extracted data, present it. For document preview and validation flags, the **Results UI** or NestJS GET-by-id may still be needed.

This is the **same pipeline** the product uses; the Results page also persists a MongoDB record via NestJS for the UI.

---

## Results in the web app (humans)

1. Sign in to DocuAgent (Azure AD).
2. Open **Results** (`/results`).
3. Tabs: **Extractions** | **Batches** | **Checks** | **Queue**.

Filters on **Extractions**: Order ID, Execution ID, field config, status, date range.

| Status | Meaning |
|--------|---------|
| `pending` | Created, not started |
| `queued` | Waiting in queue |
| `processing` | Running |
| `completed` | Done — open row for output |
| `failed` | Error — open row for reason |

Offer this path when the user has **no** `execution_id` and cannot use Bearer automation.

---

## NestJS execution API (full records, automation)

**Auth:** `Authorization: Bearer <token>` (UI login JWT — **not** `DOCAGENT_AGENTS_API_KEY`).

| Environment | Base |
|-------------|------|
| UAT | `https://api.uat.doc-agent.lfxdigital.app/v1` |
| Local | `http://localhost:3000/v1` |

### List extractions

```http
GET {base}/execution/sdu-extraction-executions?orderId=ORDER-123&status=completed
Authorization: Bearer <token>
```

Query params: `orderId`, `executionId`, `fieldConfigId`, `status`, `fromDate`, `toDate`, `page`, `size`, `sortOrder`.

### One extraction by id

```http
GET {base}/execution/sdu-extraction-executions/{id}
Authorization: Bearer <token>
```

Includes `status`, `requestBody`, `output` when complete. `202` without `output` usually means still running.

### Content checks

```http
GET {base}/execution/sdu-check-content-executions
Authorization: Bearer <token>
```

### Batches

```http
GET {base}/execution/sdu-extraction-executions/batch
Authorization: Bearer <token>
```

### Queue (current user)

```http
GET {base}/execution/sdu-extraction-executions/queue-status
Authorization: Bearer <token>
```

If this host does not resolve from the agent environment, say so briefly and **fall back** to SDU `check_execution_status` when `execution_id` is known.

---

## Quick decision

| User wants… | Agent / integrator action |
|-------------|---------------------------|
| Status of a known run | `GET …/check_execution_status?execution_id=` (no API key) |
| Full output + list by order | NestJS `GET /execution/sdu-extraction-executions` + Bearer |
| Browse visually | `/results` in browser |
| Config / threads / global config jobs | `docagent-config-agent` + `X-API-Key` |

---

## Share links

From a completed extraction in the UI, **Share** may create a link (`POST …/:id/share` on execution service). Do not invent share URLs.
