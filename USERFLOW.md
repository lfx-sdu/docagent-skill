# USERFLOW — Check DocuAgent results

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
