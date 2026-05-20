---
name: docagent-platform
description: Routes DocuAgent work by intent — product UI (NestJS /execution/* on uat.api.doc-agent.lfxdigital.app) vs Agents OpenAPI (/agents/v1 on api.uat.t4s). Use for auth, async polling, and stack choice. Results browsing and hostnames are in docagent-results.
---

# DocuAgent platform router

## Hostnames (UAT)

| Surface | URL |
|---------|-----|
| Web app | `https://uat.doc-agent.lfxdigital.app` |
| NestJS API (`NEXT_PUBLIC_DEV_API` in UAT builds) | `https://uat.api.doc-agent.lfxdigital.app/v1` |
| Agents / SDU OpenAPI | `https://api.uat.t4s.lfxdigital.app/agents/v1` |

**Wrong:** `https://api.uat.doc-agent.lfxdigital.app` — does not resolve. Use **`uat.api`**, not `api.uat`.

**Prod API (typical):** `https://api.doc-agent.lfxdigital.app/v1` — UAT service keys return **401** here.

---

## Two stacks

| Stack | Surface | Base (UAT) | Primary auth |
|-------|---------|------------|--------------|
| **Product** | Next.js (`doc-agent/frontend`) | `https://uat.api.doc-agent.lfxdigital.app/v1` | MSAL JWT, service key (`sa-…`), team SA + `X-Actor-Email` |
| **Agents OpenAPI** | SDU FastAPI ([Swagger](https://api.uat.t4s.lfxdigital.app/agents/v1/docs)) | `https://api.uat.t4s.lfxdigital.app/agents/v1` | `X-API-Key` on `/config_integration/*` |

NestJS **orchestrates** SDU for document runs. The UI triggers and polls **`/execution/*`**, not Agents `check_execution_status`.

OpenAPI wire prefixes: **`Air8`** → `/air8_integration/`, **`ConfigAgent`** → `/config_integration/`, **`LFSearch`** → `/search_integration/`, **`NER`** → `/ner_integration/`.

---

## Product auth (`axiosInstance.ts`)

1. **Team mode:** `Authorization: Bearer <SA key>` + `X-Actor-Email: <email>`  
   **Exceptions** (user JWT required): `/admin/*`, `/teams/:id/sa-key`
2. **User mode:** `Authorization: Bearer <MSAL ID token>`
3. **Service key login:** same Bearer shape (paste on `/login`)
4. **Public:** URLs with `/public/`, `/share/`, `/changelog`

Local dev: frontend `http://localhost:3001/v1`; config may use `NEXT_PUBLIC_CONFIG_API` (3101).

**Config Agent** (separate from Results): `NEXT_PUBLIC_EXTERNAL_AGENT_ENDPOINT` + API key, or gateway `/agents/v1/config_integration/*`.

---

## Automation env (optional, never commit)

| Variable | Purpose |
|----------|---------|
| `DOCAGENT_BEARER_TOKEN` | Service key or JWT for NestJS `/execution/*` |
| `DOCAGENT_NESTJS_BASE_URL` | Default `https://uat.api.doc-agent.lfxdigital.app/v1` |
| `DOCAGENT_ACTOR_EMAIL` | With team SA |
| `DOCAGENT_AGENTS_API_KEY` | **Only** `/config_integration/*` |

---

## Agents API onboarding

- Examples use **`https://api.uat.t4s.lfxdigital.app/agents/v1`** (no env required for base).
- Contract: `GET /openapi.json` or [Swagger UI](https://api.uat.t4s.lfxdigital.app/agents/v1/docs).

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/health"
```

---

## Decision tree

0. **Results / recent runs / queue** → `docagent-results`  
   - Human: `https://uat.doc-agent.lfxdigital.app/results`  
   - curl: `uat.api…/v1/execution/sdu-*` + Bearer  
   - Browser automation: Kimi WebBridge when agent DNS fails  
   - `execution_id` only: SDU `check_execution_status`
1. **Extract / validate** → `docagent-extraction`
2. **Content checks** → `docagent-content-check`
3. **Export Excel** → `docagent-export-results`
4. **Config chat / jobs** → `docagent-config-agent` + `X-API-Key`
5. **Search** → `docagent-search`
6. **NER** → `docagent-ner`
7. **Health / KPIs** → `docagent-admin-kpis`

---

## Async polling

| Job | UI interval | Endpoint | Stop when |
|-----|-------------|----------|-----------|
| Extraction | 2s | `GET …/sdu-extraction-executions/{id}` | `output` or **422** |
| Batch | 2s | `GET …/batch/{id}` | complete |
| Content check | 2s | `GET …/sdu-check-content-executions/{id}` | `output` |
| Queue dashboard | 15s | `GET …/queue-status` | — |
| SDU fallback | 2–5s | `check_execution_status` | terminal |

---

## Request discipline

- Bodies from `openapi.json` schemas.
- Multipart: read OpenAPI `multipart/form-data` — do not guess fields.
- **422:** read `detail` and fix body.

---

## Safety

- **DELETE** on `/search_integration/*/{search_id}` — confirm with user.
- Do not log API keys or full document payloads.

---

## Endpoint coverage (Agents OpenAPI)

| Group | Skill(s) |
|-------|----------|
| Health | `docagent-admin-kpis` |
| Document processing | `docagent-extraction`, `docagent-content-check`, `docagent-results` (SDU status) |
| ConfigAgent | `docagent-config-agent`, `docagent-file-prep` |
| LFSearch | `docagent-search` |
| NER | `docagent-ner` |

**Product Results** (not in Agents Swagger): NestJS `/execution/*` on **`uat.api.doc-agent.lfxdigital.app`** — see `docagent-results`.
