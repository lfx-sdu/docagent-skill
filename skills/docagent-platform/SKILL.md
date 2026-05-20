---
name: docagent-platform
description: Router skill for DocuAgent tasks. Decide between product UI/NestJS execution API vs SDU Agents API, pick the right local skill (docagent-results or docagent-extraction), and apply correct auth/host defaults for UAT.
---

# DocuAgent platform router

Use this skill first to route user intent. Then hand off to a domain skill.

## What exists in this repo now

| Local skill | Use for |
|-------------|---------|
| `docagent-results` | `/results` page, recent runs, queue, checks, share, retries |
| `docagent-extraction` | `/document-extraction` page, single upload, merge-and-trigger, batch create/process |
| `docagent-config-updates` | `/configurations` page, field/checker config CRUD, clone, reparent |

If user asks for domains not yet bundled here (config-agent/search/NER), state they are upstream or create new local skills.

## Hostnames (UAT, verified)

| Surface | URL |
|---------|-----|
| Web app | `https://uat.doc-agent.lfxdigital.app` |
| NestJS API (`NEXT_PUBLIC_DEV_API` in UAT builds) | `https://uat.api.doc-agent.lfxdigital.app/v1` |
| Agents / SDU OpenAPI | `https://api.uat.t4s.lfxdigital.app/agents/v1` |

**Wrong:** `https://api.uat.doc-agent.lfxdigital.app` — does not resolve. Use **`uat.api`**, not `api.uat`.

**Prod API (typical):** `https://api.doc-agent.lfxdigital.app/v1` — UAT service keys return **401** here.

---

## Two API stacks (do not conflate)

| Stack | Surface | Base (UAT) | Primary auth |
|-------|---------|------------|--------------|
| **Product** | Next.js (`doc-agent/frontend`) | `https://uat.api.doc-agent.lfxdigital.app/v1` | MSAL JWT, service key (`sa-…`), team SA + `X-Actor-Email` |
| **Agents OpenAPI** | SDU FastAPI ([Swagger](https://api.uat.t4s.lfxdigital.app/agents/v1/docs)) | `https://api.uat.t4s.lfxdigital.app/agents/v1` | `X-API-Key` on `/config_integration/*` |

NestJS **orchestrates** SDU for document runs. The UI triggers and polls **`/execution/*`**, not Agents `check_execution_status`.

OpenAPI wire prefixes: **`Air8`** → `/air8_integration/`, **`ConfigAgent`** → `/config_integration/`, **`LFSearch`** → `/search_integration/`, **`NER`** → `/ner_integration/`.

---

## Auth model (practical)

1. **Team mode:** `Authorization: Bearer <SA key>` + `X-Actor-Email: <email>`  
   **Exceptions** (user JWT required): `/admin/*`, `/teams/:id/sa-key`
2. **User mode:** `Authorization: Bearer <MSAL ID token>`
3. **Service key login:** same Bearer shape (paste on `/login`)
4. **Public:** URLs with `/public/`, `/share/`, `/changelog`

Local dev: frontend `http://localhost:3001/v1`; config may use `NEXT_PUBLIC_CONFIG_API` (3101).

**Config Agent** (separate from Results): `NEXT_PUBLIC_EXTERNAL_AGENT_ENDPOINT` + API key, or gateway `/agents/v1/config_integration/*`.

---

## Optional automation env (never commit)

| Variable | Purpose |
|----------|---------|
| `DOCAGENT_BEARER_TOKEN` | Service key or JWT for NestJS `/execution/*` |
| `DOCAGENT_NESTJS_BASE_URL` | Default `https://uat.api.doc-agent.lfxdigital.app/v1` |
| `DOCAGENT_ACTOR_EMAIL` | With team SA |
| `DOCAGENT_AGENTS_API_KEY` | **Only** `/config_integration/*` |

---

## Preference cache (routing hint, not source of truth)

For per-user defaults, use local cache outside this repo:

- `~/.cache/docagent-skills/preferences/<user_key>.json`

Router behavior:

1. If cache exists, propose cached extraction defaults.
2. Always validate cached values against live config options before using.
3. If mismatch, ignore cache and route to extraction preflight + confirmation.
4. Never store tokens/keys in cache.

## Routing checklist (what skill to use)

1. **User asks about runs/results/check status/share/queue?**  
   Use `docagent-results`.
2. **User asks to extract docs/upload docs/merge files/batch extraction?**  
   Use `docagent-extraction`.
   - If `field_config_id` / `nation` / `possible_doc_type` are missing or ambiguous, route to **confirm mode** (preflight + user confirmation) before any execution call.
3. **User asks to create/update/delete field/checker configs?**  
   Use `docagent-config-updates`.
3. **User has only `execution_id` and no bearer token?**  
   Use SDU fallback status poll:
   `GET /agents/v1/air8_integration/check_execution_status?execution_id=...`
4. **User asks for Config Agent chat or search/NER APIs?**  
   Not in this local bundle; route to upstream skill set or add new local skill.

---

## Agents API onboarding (when needed)

- Examples use **`https://api.uat.t4s.lfxdigital.app/agents/v1`** (no env required for base).
- Contract: `GET /openapi.json` or [Swagger UI](https://api.uat.t4s.lfxdigital.app/agents/v1/docs).

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/health"
```

---

## Fast playbooks

### Results-oriented request

Use `docagent-results` with this order:
1. `curl` on `uat.api.../v1/execution/...` if bearer exists.
2. If agent DNS/auth blocks, use browser/Kimi WebBridge on `/results`.
3. If only `execution_id`, use SDU `check_execution_status`.

### Extraction-oriented request

Use `docagent-extraction`:
1. **`field_config_id` first** — then list valid **Country** (`nation`) and **Document type** (`possible_doc_type`) from **that** config’s `fieldConfig` (not ISO codes; not values from another config).
2. If user did not explicitly provide config + nation + doc type, run preflight and ask for confirmation before execute.
   - Use the safe list-config parser in `skills/docagent-extraction/SKILL.md` (preflight section) to avoid list-vs-dict response errors.
3. Single upload: create → blob upload → poll by id.
4. Multi-file merge: `merge-and-trigger` → poll by id.
5. Batch tab: create batch → process batch → poll batch.

---

## Polling defaults

| Flow | Interval | Stop condition |
|------|----------|----------------|
| Extraction get-by-id | ~2s | `output` present or `422` |
| Batch get-by-id | ~2s | terminal batch status |
| Queue dashboard | ~15s | user stops monitoring |
| SDU fallback status | 2–5s | terminal status |

---

## Practical anti-patterns to avoid

| Bad | Correct |
|-----|---------|
| `api.uat.doc-agent...` | `uat.api.doc-agent...` |
| Using Agents API to list results | Use NestJS `/execution/*` list/count routes |
| Using `DOCAGENT_AGENTS_API_KEY` for NestJS | Use Bearer token (`sa-…` or JWT) |
| Assuming empty table means no runs | Check login/session first |
| Reusing `nation` / `possible_doc_type` across configs | Re-list pairs per `field_config_id` (`CN` ≠ `china` ≠ `General`) |

---

## Safety

- Never commit `.env` or service keys.
- Redact bearer/API keys in transcripts and copied logs.
- Confirm destructive actions before running `DELETE` or bulk admin mutation endpoints.

For endpoint-level details, use:
- `docagent-results` for `/results` and execution monitoring
- `docagent-extraction` for `/document-extraction` flows
