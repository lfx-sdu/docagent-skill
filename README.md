# DocuAgent Skills

Reusable agent skills for the DocuAgent **Agents API** (`/agents/v1`), compatible with the Vercel Skills CLI. Skills teach agents to call **HTTP endpoints** directly—**not** local repo scripts or backend module paths.

## Prerequisites

- Network access to the Agents API (UAT example below).
- Credentials as required by your environment: many routes have **no** `security` in OpenAPI; **ConfigAgent** routes use **`X-API-Key`**.

### Environment variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `DOCAGENT_AGENTS_API_BASE_URL` | Yes | API root, e.g. `https://api.uat.t4s.lfxdigital.app/agents/v1` |
| `DOCAGENT_AGENTS_API_KEY` | For ConfigAgent | Sent as `X-API-Key` on `/config_integration/*` routes that declare API key auth |

Optional (only if your tenant still uses Azure AD client credentials for a different surface):

- `DOCAGENT_TENANT_ID`, `DOCAGENT_CLIENT_ID`, `DOCAGENT_CLIENT_SECRET`, `DOCAGENT_SCOPE` — **not** used by the Agents OpenAPI doc by default.

### Reference docs

- Swagger UI (UAT): [https://api.uat.t4s.lfxdigital.app/agents/v1/docs](https://api.uat.t4s.lfxdigital.app/agents/v1/docs)
- OpenAPI JSON: `GET $DOCAGENT_AGENTS_API_BASE_URL/openapi.json`

### Preflight (read-only)

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  "$DOCAGENT_AGENTS_API_BASE_URL/health"
```

Expect `200` when the service is healthy.

### ConfigAgent preflight (when using `DOCAGENT_AGENTS_API_KEY`)

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  "$DOCAGENT_AGENTS_API_BASE_URL/config_integration/fetch-threads/<user_id>?limit=1&skip=0"
```

Adjust `<user_id>` to a valid id for your tenant.

## Install

```bash
# install all skills from this repo
npx skills add erictaicp/docagent-skills

# install selected skills
npx skills add erictaicp/docagent-skills --skill docagent-platform --skill docagent-extraction

# install globally for your machine
npx skills add erictaicp/docagent-skills -g

# install to specific agents
npx skills add erictaicp/docagent-skills -a cursor -a codex -a claude-code
```

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
```
