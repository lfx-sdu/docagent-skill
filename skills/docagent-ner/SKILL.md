---
name: docagent-ner
description: Operates NER Agents API—company/product trace and suggest, globe trace, entity/product pipeline run status list delete and sync-test endpoints. Use for /ner_integration/ner/* supply-chain intelligence flows.
---

# DocuAgent NER (`/ner_integration/ner/*`)

Base: `$DOCAGENT_AGENTS_API_BASE_URL`. Load body schemas from `openapi.json`.

## Interactive trace / suggest

| Method | Path | Role |
|--------|------|------|
| POST | `/ner_integration/ner/company/trace` | Company trace pipeline (`TraceCompanyRequest`) |
| POST | `/ner_integration/ner/company/trace_globe` | Globe clustering (`TraceGlobeRequest`) |
| POST | `/ner_integration/ner/company/suggest` | Entity suggestions (`SuggestEntitiesRequest`) |
| POST | `/ner_integration/ner/product/trace` | Product trace (`TraceProductRequest`) |
| POST | `/ner_integration/ner/product/suggest` | Product suggestions (`SuggestProductsRequest`) |

Minimal company trace example:

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/ner_integration/ner/company/trace" \
  -H "Content-Type: application/json" \
  -d '{"company_query":"Example Corp"}'
```

Minimal product suggestion example:

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/ner_integration/ner/product/suggest" \
  -H "Content-Type: application/json" \
  -d '{"query":"stainless flange"}'
```

## Entity embedding pipeline

| POST | `/ner_integration/ner/entity-pipeline/run` | Background (`PipelineRunRequest`) → `task_id` |
| GET | `/ner_integration/ner/entity-pipeline/status/{task_id}` | Status |
| GET | `/ner_integration/ner/entity-pipeline/tasks` | Query `status`, `limit` |
| POST | `/ner_integration/ner/entity-pipeline/run_sync` | **Testing only**, synchronous |
| DELETE | `/ner_integration/ner/entity-pipeline/tasks/{task_id}` | **Destructive**, confirm |

## Product embedding pipeline

| POST | `/ner_integration/ner/product-pipeline/run` | Background (`ProductPipelineRunRequest`) |
| GET | `/ner_integration/ner/product-pipeline/status/{task_id}` | Status |
| GET | `/ner_integration/ner/product-pipeline/tasks` | Query params |
| POST | `/ner_integration/ner/product-pipeline/run_sync` | Testing only |
| DELETE | `/ner_integration/ner/product-pipeline/tasks/{task_id}` | **Destructive**, confirm |

Poll background jobs with exponential backoff (`docagent-platform`).
