---
name: docagent-admin-kpis
description: Agents API operational diagnostics only—healthcheck roots, GET /health, GET /memory. Use for uptime checks on /agents/v1. Not for doc-agent web admin KPI dashboards (those are outside this OpenAPI bundle).
---

# DocuAgent Agents API observability

The **Agents API** OpenAPI exposes only lightweight runtime introspection—not the NestJS `/admin/kpis` product surface.

Use this skill When you need:

- readiness / health checks against the Agents deployment
- memory stats for troubleshooting (if permitted in your env)

Base: `https://api.uat.t4s.lfxdigital.app/agents/v1`

## Endpoints (no mutating payloads)

```bash
# Root health stub
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/"

# Detailed health (monitoring-oriented)
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/health"

# Process memory diagnostics
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/memory"
```

## Product admin KPI dashboards

Those live on the DocuAgent **web app + execution-service admin modules**, not documented in `/agents/v1/openapi.json`. Automate those via **their** authenticated APIs/UI flows—outside this repo’s Agents skill bundle.
