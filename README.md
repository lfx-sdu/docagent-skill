# DocuAgent Skills

Agent skill packs for **DocuAgent** — how to use the product UI, NestJS execution APIs, and the LFX SDU **Agents API** (`/agents/v1`).

**Upstream:** [github.com/lfx-sdu/docagent-skill](https://github.com/lfx-sdu/docagent-skill)

This repository holds Markdown under `skills/*/SKILL.md`. Skills teach agents which endpoints and workflows to use; they are not Python modules or runnable scripts.

Skills are aligned with the **DocuAgent frontend** (`doc-agent/frontend`): Results at `/results`, NestJS `/execution/*` for list/poll/share, and Agents API only for Config Agent plus the agent-only SDU status poll.

**Agent rule:** When a user asks to check results and provides an **`execution_id`**, poll SDU immediately — do not reply that this repo "only has skills" or that results require Azure AD. For **recent runs** without an id, use NestJS list + Bearer JWT or guide `/results`. See [USERFLOW.md](USERFLOW.md) and `docagent-results`.

---

## Included skills

| Skill | Use when |
|-------|----------|
| [`docagent-platform`](skills/docagent-platform/SKILL.md) | Routing by intent, auth (`X-API-Key` for ConfigAgent), async polling, OpenAPI tags |
| [`docagent-results`](skills/docagent-results/SKILL.md) | Checking extraction, batch, and content-check results in the app or via API |

Additional domain skills (extraction, config agent, search, NER, etc.) may live in the upstream repo; this fork currently keeps the platform router and results guide only.

---

## API bases (two stacks)

| Stack | UAT base | Used for |
|-------|----------|----------|
| **Web app** | `https://uat.doc-agent.lfxdigital.app` | `/results`, login, UI |
| **Product / NestJS** | `https://uat.api.doc-agent.lfxdigital.app/v1` | Results list, poll by id, share, queue |
| **Agents / SDU** | `https://api.uat.t4s.lfxdigital.app/agents/v1` | Config Agent, `check_execution_status` (agent shortcut) |

**Hostname trap:** `api.uat.doc-agent.lfxdigital.app` does **not** resolve. The live UAT API is **`uat.api.doc-agent…`**, not `api.uat…`.

Agents API:

- **Swagger UI:** [https://api.uat.t4s.lfxdigital.app/agents/v1/docs](https://api.uat.t4s.lfxdigital.app/agents/v1/docs)
- **OpenAPI JSON:** `GET …/openapi.json`
- Do **not** append `/docs` to `curl` URLs.

The **Results UI does not call** Agents API for browsing or polling runs. See `docagent-results` for tabs, filters, and auth (MSAL JWT, service key, team SA + `X-Actor-Email`).

---

## Prerequisites

| Variable | Required | Purpose |
|----------|----------|---------|
| `DOCAGENT_AGENTS_API_KEY` | For ConfigAgent routes only | Sent as `X-API-Key` on `/config_integration/*` |
| `DOCAGENT_BEARER_TOKEN` | For Results / NestJS `/execution/*` | Service key (`sa-…`) or JWT — **not** the Agents API key |

**Not** used for `GET /air8_integration/check_execution_status` (no auth). **Not** a substitute: Agents key ≠ NestJS Bearer.

**Automation (optional, never commit):** `DOCAGENT_NESTJS_BASE_URL` (default UAT: `https://uat.api.doc-agent.lfxdigital.app/v1`), `DOCAGENT_BEARER_TOKEN`, `DOCAGENT_ACTOR_EMAIL` for team SA. Or use **Kimi WebBridge** on `uat.doc-agent…/results` when curl from the agent host fails.

No base URL environment variable is required for standard onboarding — substitute the hostname only for non-UAT integrations.

### Check a run by execution id (no API key)

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

### Preflight

```bash
curl -sS -o /dev/null -w "%{http_code}\n" \
  "https://api.uat.t4s.lfxdigital.app/agents/v1/health"
```

Expect `200` when the service is healthy.

---

## Install

Use the [Vercel Skills CLI](https://vercel.com/docs/agent-resources/skills) or clone this repo for file-based skills.

### Install from GitHub (all skills in package)

```bash
npx skills add lfx-sdu/docagent-skill
```

### Install only these skills

```bash
npx skills add lfx-sdu/docagent-skill --skill docagent-platform --skill docagent-results
```

### Fast setup (ConfigAgent)

```bash
npx skills add lfx-sdu/docagent-skill
export DOCAGENT_AGENTS_API_KEY="<config-agent-api-key>"
```

### Clone locally

```bash
git clone https://github.com/lfx-sdu/docagent-skill.git
# or this fork:
git clone https://github.com/erictaicp/docagent-skills.git
```

Point your agent at `skills/<skill-name>/SKILL.md` per your tool’s local-skills rules.

### Target specific agents (optional)

```bash
npx skills add lfx-sdu/docagent-skill -a cursor -a codex -a claude-code
```

---

## Repository layout

```text
skills/
  docagent-platform/SKILL.md   # Router and API discipline
  docagent-results/SKILL.md    # Results UI + execution API + agent playbook
README.md
USERFLOW.md                    # Step-by-step: check results (agent-first)
```

### Troubleshooting agent confusion

If an agent claims there is **no API to list “latest runs”**, point it at **`skills/docagent-results/SKILL.md`**. Common fixes: use **`uat.api.doc-agent…`** (not `api.uat…`), set **`DOCAGENT_BEARER_TOKEN`**, or open **`/results`** via WebBridge.

---

## Security

- Skill text can be public; access control is API keys, network policy, and storage ACLs.
- Never commit `.env` or API keys (`.env` is gitignored).
- Do not log API keys or full document payloads in agent transcripts.
