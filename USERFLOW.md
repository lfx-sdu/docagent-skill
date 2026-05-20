# USERFLOW — Check DocuAgent results

**Agents:** read `skills/docagent-results/SKILL.md` first. **Do not** tell the user you cannot access results when you can call SDU with an `execution_id`.

**API root (SDU):** `https://api.uat.t4s.lfxdigital.app/agents/v1`

---

## Flow A — User has `execution_id` (agent in Cursor/CLI)

1. Poll status (no API key, no Azure login):

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

2. Interpret JSON: report `status`; if terminal, summarize any output fields returned.
3. If `processing` / `pending`, wait 2–5s and poll again (reasonable cap).
4. If user needs document preview or full `output` object and SDU response is minimal → point to **Results UI** or NestJS GET with Bearer (Flow C).

---

## Flow B — User has no id (human)

1. Sign in (Azure AD) → **Results** (`/results`).
2. Tab: **Extractions** | **Batches** | **Checks** | **Queue**.
3. Filter by Order ID or Execution ID → open **completed** row.

---

## Flow C — Automation with full records (Bearer JWT)

**Base (UAT):** `https://api.uat.doc-agent.lfxdigital.app/v1`

```bash
curl -sS -H "Authorization: Bearer <token>" \
  "https://api.uat.doc-agent.lfxdigital.app/v1/execution/sdu-extraction-executions?orderId=ORDER-123"
```

Single run:

```bash
curl -sS -H "Authorization: Bearer <token>" \
  "https://api.uat.doc-agent.lfxdigital.app/v1/execution/sdu-extraction-executions/<id>"
```

`DOCAGENT_AGENTS_API_KEY` is **not** used on these routes.

---

## Auth cheat sheet

| Route | Auth |
|-------|------|
| `GET …/air8_integration/check_execution_status` | None |
| `GET …/execution/sdu-extraction-executions` | Bearer JWT |
| `GET/POST …/config_integration/*` | `X-API-Key: $DOCAGENT_AGENTS_API_KEY` |
| Results UI `/results` | Azure AD session |

---

## Preflight

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  "https://api.uat.t4s.lfxdigital.app/agents/v1/health"
```

Expect `200`.
