---
name: docagent-extraction
description: "Implement and debug DocuAgent extraction execution workflows end-to-end across backend execution modules and frontend extraction result views. Use when requests involve extraction list/count/filter issues, extraction execution states, extraction API contracts, or `/document-extraction` and extraction result cards."
---

# Docagent Extraction

## Primary Backend Areas

- `backend/llm-execution-service/src/sdu-extraction-execution/business/queries`
- `backend/llm-execution-service/src/sdu-extraction-execution/business/services`
- `backend/llm-execution-service/src/sdu-extraction-execution/controller`

## Primary Frontend Areas

- `frontend/app/components/documentResultsCard`
- `frontend/app/components/documentResultsCard/components/documentExtractionResultsListCard`
- `frontend/app/components/documentResultsCard/hooks`
- `frontend/app/models/results`
- `frontend/app/services/results`

## Workflow

1. Reproduce the issue in extraction list/count/filter/sort behavior.
2. Trace API path: controller DTO -> business query/service -> repository/query builder.
3. Verify execution status mapping and total-count semantics match list query.
4. Align frontend model/service parsing with backend response fields.
5. Fix shared contracts first, then adjust presentation components.

## Common Failure Patterns

- List query and count query apply different filters.
- DTO defaults diverge from frontend filter defaults.
- Status enums differ between backend payload and frontend type union.
- Pagination indexing mismatch (page vs offset assumptions).
- Frontend keeps stale state after filter/sort updates.

## Done Criteria

- Extraction list/count are consistent for same filters.
- Sorting and pagination are deterministic.
- Empty, loading, and failure states remain usable.
- Changed DTOs/models/services are all type-safe and aligned.

