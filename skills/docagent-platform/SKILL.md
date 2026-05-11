---
name: docagent-platform
description: "Route DocuAgent platform requests to the correct domain workflow: extraction, results retrieval, content checking, config-agent, export results, and admin KPI operations. Use when users ask to implement, debug, or review behavior across DocuAgent execution services, frontend result pages/cards, or admin dashboards."
---

# Docagent Platform

## Goal

Handle DocuAgent platform work by selecting the correct domain skill first, then applying that skill's workflow deeply enough to fix root causes instead of shipping workarounds.

## Domain Router

Use this decision tree:

1. If request mentions extraction execution, extraction list/count, extraction cards, or document extraction result screens -> use `docagent-extraction`.
2. If request mentions result retrieval APIs, list/count mismatches, pagination/filters, or shared result models/services -> use `docagent-results`.
3. If request mentions check content execution, content checking page/cards, check list/count, or check statuses -> use `docagent-content-check`.
4. If request mentions config-agent, config chat, streaming responses, prompt/configuration UX, or config-related service integration -> use `docagent-config-agent`.
5. If request mentions export data execution, export result cards, download/export statuses, or export list/count -> use `docagent-export-results`.
6. If request mentions admin KPI page, token usage users, admin jobs, or cross-platform admin analytics -> use `docagent-admin-kpis`.

If multiple areas are involved, choose a primary domain and run a secondary pass for cross-module consistency.

## Execution Rules

1. Confirm scope from changed files and affected routes/components.
2. Trace backend query/service/controller flow before changing UI behavior.
3. Keep DTO, query handler, service, model, and frontend contract aligned.
4. Verify list/count/filter/sort semantics stay consistent between backend and frontend.
5. Prefer explicit fixes for root cause over compatibility hacks.

## Validation Checklist

- Run focused lint/test commands for touched services or frontend modules.
- Verify no DTO or model drift between backend payload and frontend typings.
- Re-check affected pages/cards for empty, loading, and error states.
- Report what changed, why it fixes root cause, and what was validated.

