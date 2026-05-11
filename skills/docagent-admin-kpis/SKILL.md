---
name: docagent-admin-kpis
description: "Build and debug DocuAgent admin KPI and jobs analytics across backend admin modules and frontend `/admin/kpis` views. Use when users ask about platform-wide metrics, token usage top users, admin job list behavior, KPI data aggregation, or admin dashboard UI/UX updates."
---

# Docagent Admin Kpis

## Primary Backend Areas

- `backend/llm-execution-service/src/admin/admin-jobs`
- `backend/llm-execution-service/src/sdu-token-usage` and token usage aggregates
- Any admin query/service joining extraction, check, and export metrics

## Primary Frontend Areas

- `frontend/app/(routes)/admin/kpis/page.tsx`
- `frontend/app/components/adminKpis/TokenUsageTopUsers.tsx`
- `frontend/app/components/adminJobList`

## Workflow

1. Confirm KPI definitions before changing formulas or labels.
2. Trace aggregation source queries and date/window filters.
3. Keep backend metric fields and frontend chart/table schemas aligned.
4. Validate admin-role access behavior remains unchanged.
5. Review KPI UI density and readability (calm hierarchy, clear labels).

## Validation Checklist

- KPI totals are reproducible from source queries.
- Time range and timezone behavior are explicit.
- Top-users and job lists remain stable under sort/filter changes.
- Frontend loading/empty/error states are informative but quiet.

