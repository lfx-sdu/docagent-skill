---
name: docagent-content-check
description: "Implement and troubleshoot DocuAgent content-check execution behavior across backend check-content modules and frontend content-checking result cards. Use when requests involve check execution list/count, check statuses, content-check filters, or `/content-checking` result rendering."
---

# Docagent Content Check

## Primary Backend Areas

- `backend/llm-execution-service/src/sdu-check-content-execution/business/queries`
- `backend/llm-execution-service/src/sdu-check-content-execution/business/services`
- `backend/llm-execution-service/src/sdu-check-content-execution/controller`

## Primary Frontend Areas

- `frontend/app/components/contentCheckingResultsCard`
- `frontend/app/components/contentCheckingResultsCard/contentCheckingResultsListCard`
- Shared `results` models/services

## Workflow

1. Reproduce check-content list/count discrepancy or rendering issue.
2. Inspect controller DTO and defaults first.
3. Compare list query filters with count query filters.
4. Verify service-level status transformations and derived flags.
5. Align frontend types and card rendering states with API response.

## Quality Gates

- Count endpoint and list endpoint return consistent totals.
- Filter combinations do not break pagination.
- Status badges and summary labels reflect true backend state.
- Empty/failed checks render clear, non-blocking UI states.

## Root-Cause Bias

Never stop at cosmetic fixes in cards when mismatch originates in query logic, DTO defaults, or backend status mapping.

