---
name: docagent-results
description: "Maintain DocuAgent result retrieval flows and shared contracts across extraction, content-checking, and export result pages/cards. Use when users ask to fetch results, fix list/count mismatches, adjust filtering/sorting/pagination, or update shared result models/services."
---

# Docagent Results

## Core Contract Files

- `frontend/app/models/results/index.ts`
- `frontend/app/services/results/index.ts`
- `frontend/app/models/admin/index.ts`

## Result Surfaces

- Extraction result cards and hooks
- Content checking result cards and list cards
- Export result cards and list cards

## Query Consistency Rules

1. Use one canonical filter mapping across list and count endpoints.
2. Keep date/time parsing and timezone assumptions identical between backend and frontend.
3. Keep page size, offset/page index, and sort fields aligned.
4. Ensure default filters produce identical totals across tabs/cards.

## Change Procedure

1. Define expected API request/response shape.
2. Update backend DTO/query/service first if contract is wrong.
3. Update frontend model + service mapping next.
4. Update list/card hooks and render components last.
5. Validate that list length and "total" stay in sync for same query.

## Anti-Patterns

- Fixing only UI with temporary mapping while backend remains inconsistent.
- Duplicating separate filter mappers per component.
- Silent fallback conversions that hide contract breakage.

