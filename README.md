# DocuAgent Skills

**Repository:** [github.com/lfx-sdu/docagent-skill](https://github.com/lfx-sdu/docagent-skill)

Markdown skill packs for the DocuAgent **Agents API** (`/agents/v1`). They teach agents which **HTTP** endpoints to call and how—not local Python modules or repo scripts.

---

## Start here

| Step | What to do |
|------|------------|
| 1 | [Install skills](#install) into your agent (Cursor, Copilot, Claude Code, …) via the Skills CLI, **or** clone this repo if your workflow is file-based. |
| 2 | Set [environment variables](#prerequisites). Base URL can default to UAT; API key is needed for ConfigAgent routes. |
| 3 | Follow **[USERFLOW.md](USERFLOW.md)** for end-to-end requests: extraction → polling → ConfigAgent → results. |

**This repo** = skill text under `skills/*/SKILL.md`.  
**Not in this repo** = your API keys, tenant URLs, or PDFs—those stay in env and your storage.

---

## Prerequisites

- Reachable **Agents API** (example UAT host appears in examples below; use your own URL for prod).
- **ConfigAgent** routes expect **`X-API-Key`** when your OpenAPI marks them as API-key auth.

### Environment variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `DOCAGENT_AGENTS_API_BASE_URL` | Optional | API root. If unset, use default `https://api.uat.t4s.lfxdigital.app/agents/v1` |
| `DOCAGENT_AGENTS_API_KEY` | For ConfigAgent | Sent as `X-API-Key` on `/config_integration/*` where applicable |

Optional (only if another surface still uses Azure AD client credentials):

- `DOCAGENT_TENANT_ID`, `DOCAGENT_CLIENT_ID`, `DOCAGENT_CLIENT_SECRET`, `DOCAGENT_SCOPE` — **not** required by the default Agents OpenAPI flow documented here.

### Reference docs

- Swagger UI (UAT): [https://api.uat.t4s.lfxdigital.app/agents/v1/docs](https://api.uat.t4s.lfxdigital.app/agents/v1/docs)
- OpenAPI JSON: `GET $DOCAGENT_AGENTS_API_BASE_URL/openapi.json`

### Preflight (read-only)

```bash
DOCAGENT_AGENTS_API_BASE_URL="${DOCAGENT_AGENTS_API_BASE_URL:-https://api.uat.t4s.lfxdigital.app/agents/v1}"
curl -sS -o /dev/null -w "%{http_code}\n" \
  "$DOCAGENT_AGENTS_API_BASE_URL/health"
```

Expect `200` when the service is healthy.

### ConfigAgent preflight (when using `DOCAGENT_AGENTS_API_KEY`)

```bash
DOCAGENT_AGENTS_API_BASE_URL="${DOCAGENT_AGENTS_API_BASE_URL:-https://api.uat.t4s.lfxdigital.app/agents/v1}"
curl -sS -o /dev/null -w "%{http_code}\n" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/fetch-threads/<user_id>?limit=1&skip=0"
```

Replace `<user_id>` with a valid id for your tenant.

---

## API usage quickstart (payload-first)

1. Validate environment (`/health`, API key preflight if you use ConfigAgent).
2. Start extraction with a valid `DocsValidationRequest` payload.
3. Poll `execution_id` until a terminal status.
4. If needed, run ConfigAgent (`config-agent-stream` or async config job).
5. Fetch outputs via result endpoints (`check_execution_status`, dialog/message/job GETs).

**Full step-by-step payloads and status handling:** [USERFLOW.md](USERFLOW.md).

### Extraction minimal payload

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/air8_integration/validate_and_extract_docs" \
  -H "Content-Type: application/json" \
  -d '{
    "file_uri":"https://<storage>/<file>.pdf",
    "order_id":"ORDER-123",
    "nation":"VN",
    "possible_doc_type":["invoice"]
  }'
```

### ConfigAgent chat payload (requires API key)

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/config-agent-stream" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  -d '{
    "message":"Create extraction config for invoice + packing list",
    "thread_id":"<uuid>",
    "user_id":"<optional>"
  }'
```

---

## Install

Use the [Vercel Skills CLI](https://vercel.com/docs/agent-resources/skills). Discoverability: [skills.sh](https://skills.sh).

Yes, **`npx` install is supported now**. This is the recommended onboarding path.

### Default: install all skills

```bash
npx skills add lfx-sdu/docagent-skill
```

Installs every skill from the default branch of [lfx-sdu/docagent-skill](https://github.com/lfx-sdu/docagent-skill). After install, configure [Prerequisites](#prerequisites); the CLI does not inject your API secrets.

### Fast setup (only API key required for ConfigAgent)

```bash
npx skills add lfx-sdu/docagent-skill
export DOCAGENT_AGENTS_API_KEY="<config-agent-api-key>"
export DOCAGENT_AGENTS_API_BASE_URL="${DOCAGENT_AGENTS_API_BASE_URL:-https://api.uat.t4s.lfxdigital.app/agents/v1}"
```

If you only use Air8 endpoints that do not require API key auth, you can skip `DOCAGENT_AGENTS_API_KEY`.

### Optional: only some skills

```bash
npx skills add lfx-sdu/docagent-skill --skill docagent-platform --skill docagent-extraction
```

### Optional: global install on your machine

```bash
npx skills add lfx-sdu/docagent-skill -g
```

### Optional: target specific agent integrations

```bash
npx skills add lfx-sdu/docagent-skill -a cursor -a codex -a claude-code
```

### Without the CLI: clone the repo

```bash
git clone https://github.com/lfx-sdu/docagent-skill.git
```

Then wire `skills/<skill-name>/SKILL.md` according to your product’s “local skills” rules.

### Public GitHub vs API access

The skill Markdown can live in a **public** repo: it is not a substitute for tenant credentials. Real access control is **`DOCAGENT_AGENTS_API_BASE_URL`**, keys, and your network policies. Never commit `.env` or keys.

### Scaffold a new skill (any project)

```bash
npx skills init          # SKILL.md in the current directory
npx skills init my-skill # my-skill/SKILL.md
```

---

## Included skills

| Skill | OpenAPI tags / scope |
|-------|---------------------|
| `docagent-platform` | Router: pick the right skill; base URL, auth, polling, safety |
| `docagent-extraction` | `Air8` — validate/extract docs, execution status |
| `docagent-content-check` | `Air8` — check doc content, execution status |
| `docagent-export-results` | `Air8` + `ConfigAgent` — export blob, extraction Excel |
| `docagent-config-agent` | `ConfigAgent` — chat, threads, jobs, mapping, embeddings, uploads |
| `docagent-results` | Cross-cutting: execution status, search fetch, config jobs, dialog/message |
| `docagent-admin-kpis` | **Operational only** — `GET /`, `/health`, `/memory` on Agents API *(in-app KPI dashboards live outside this OpenAPI)* |
| `docagent-company-research` | `Air8` — company info/news search and fetch by execution |
| `docagent-file-prep` | `ConfigAgent` — `prepare-and-upload`, `merge-pdf` (multipart) |
| `docagent-search` | `LFSearch` — supplier/buyer/factory/product search and CRUD-style GET/DELETE |
| `docagent-ner` | `NER` — trace, suggest, entity/product pipelines |

---

## Repository layout

```text
skills/
  docagent-platform/SKILL.md
  docagent-extraction/SKILL.md
  docagent-results/SKILL.md
  docagent-content-check/SKILL.md
  docagent-config-agent/SKILL.md
  docagent-export-results/SKILL.md
  docagent-admin-kpis/SKILL.md
  docagent-company-research/SKILL.md
  docagent-file-prep/SKILL.md
  docagent-search/SKILL.md
  docagent-ner/SKILL.md
USERFLOW.md
README.md
```
