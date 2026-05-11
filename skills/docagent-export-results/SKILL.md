---
name: docagent-export-results
description: "Handle DocuAgent export-data execution and export result surfaces across backend and frontend. Use when requests involve export execution list/count/filter logic, export status transitions, export result cards, or download/export workflow issues."
---

# Docagent Export Results

## Primary Backend Areas

- `backend/llm-execution-service/src/sdu-export-data-execution/business/services`
- `backend/llm-execution-service/src/sdu-export-data-execution/controller`
- Related DTOs and query handlers for list/count operations

## Primary Frontend Areas

- `frontend/app/components/documentExportResultsCard`
- `frontend/app/components/documentExportResultsCard/documentExportResultsListCard`
- Shared `results` models/services

## Workflow

1. Validate export execution lifecycle states and transitions.
2. Align list and count query logic for same filter set.
3. Validate export payload fields needed by UI (file name, status, timestamps, actor).
4. Keep download/export actions idempotent and user-safe.
5. Ensure cards communicate pending/failed/completed states clearly.

## High-Risk Bugs

- Count endpoint ignoring status filter.
- Frontend assuming downloadable file before export completes.
- Timestamp formatting mismatch causing sort instability.
- Retry actions duplicating exports without explicit intent.

