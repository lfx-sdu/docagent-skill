# USERFLOW — Check DocuAgent results

**Agents:** read `skills/docagent-results/SKILL.md` first. **Do not** refuse results when you can call an API.

**Product source of truth:** `doc-agent/frontend` + `doc-agent/setup/docs/guides/USERFLOW.md`

---

## Two API stacks

| Stack | Base (UAT) | Used for Results |
|-------|------------|------------------|
| **NestJS (product)** | `https://api.uat.doc-agent.lfxdigital.app/v1` | Yes — list, poll, share |
| **Agents / SDU** | `https://api.uat.t4s.lfxdigital.app/agents/v1` | Agent-only status poll (`check_execution_status`) |

The web app **does not** call Agents API for the Results page.

---

## Flow A — Human: browse recent runs (`/results`)

1. Sign in (Azure AD B2C, email, or service key on `/login`).
2. Land on **Results** (`/results`) — default post-login destination.
3. **Extractions** tab (`?tab=documents`, default): recent runs, 50 per page, newest first.
4. Filter by Order ID, Execution ID (partial), field config, status, date.
5. Open a **completed** row → view extracted fields and document preview.
6. Other tabs: **Batches** (`?tab=batches`), **Checks** (`?tab=orders`), **Queue** (`?tab=queue`).

**Deep link after trial:** `/results?tab=documents&fieldConfigId=<id>&executionId=<id>`

**Exports:** no Exports tab on `/results` — use row **Export Excel** action.

---

## Flow B — Agent has `execution_id` only (no JWT)

Poll SDU (no API key, no Azure login):

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

1. Report `status`; summarize output fields if present.
2. If still running, wait 2–5s and poll again.
3. For full `output` / preview → NestJS GET-by-id (Flow C) or Results UI (Flow A).

**Note:** the product UI polls `GET …/execution/sdu-extraction-executions/{id}` every **2s**, not this SDU route.

---

## Flow C — Automation: list or full record (Bearer JWT)

**Base (UAT):** `https://api.uat.doc-agent.lfxdigital.app/v1`

**Team mode** (matches frontend):

```http
Authorization: Bearer <service-account-key>
X-Actor-Email: <user@email>
```

**User mode:**

```http
Authorization: Bearer <MSAL ID token>
```

### Recent extractions

```bash
curl -sS -H "Authorization: Bearer <token>" \
  "https://api.uat.doc-agent.lfxdigital.app/v1/execution/sdu-extraction-executions?page=0&size=50&sortBy=createdOn&sortOrder=desc"
```

### Single run (same as UI polling)

```bash
curl -sS -H "Authorization: Bearer <token>" \
  "https://api.uat.doc-agent.lfxdigital.app/v1/execution/sdu-extraction-executions/<id>"
```

Poll every **2s** until `output` exists or **422** (failed).

`DOCAGENT_AGENTS_API_KEY` is **not** used on these routes.

---

## Flow D — Share link (public)

1. Owner: `POST …/execution/sdu-extraction-executions/{id}/share` → `shareUrl`
2. Reader: open `/share/{token}` (no login)
3. API: `GET …/execution/public/sdu-extraction-executions/share/{token}`

---

## Auth cheat sheet

| Route | Auth |
|-------|------|
| Results UI `/results` | Azure AD / email / service key session |
| `GET …/execution/sdu-extraction-executions` | Bearer JWT or team SA + `X-Actor-Email` |
| `GET …/air8_integration/check_execution_status` | None (agent shortcut) |
| `GET/POST …/config_integration/*` | `X-API-Key: $DOCAGENT_AGENTS_API_KEY` |
| Public share GET | None |

---

## Preflight (SDU)

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  "https://api.uat.t4s.lfxdigital.app/agents/v1/health"
```

Expect `200`.
