---
name: docagent-search
description: Operates LFSearch Agents API—supplier,buyer,factory,product search POST plus GET result/execution and DELETE lifecycle. Use for /search_integration/* discovery and retrieval of search_id.
---

# DocuAgent LF Search (`/search_integration/*`)

Base: `$DOCAGENT_AGENTS_API_BASE_URL`. Schemas: `SearchRequest`, `NewSearchRequest`, `SearchResponse`.

## Start searches

| Endpoint | Purpose |
|---------|---------|
| `POST /search_integration/supplier_search` | Supplier search (`SearchRequest`) |
| `POST /search_integration/buyer_search` | Buyer search |
| `POST /search_integration/factory_search` | Factory search |
| `POST /search_integration/supplier_search_debug` | Debug variant |
| `POST /search_integration/buyer_search_debug` | Debug variant |
| `POST /search_integration/factory_search_debug` | Debug variant |
| `POST /search_integration/product_search` | Product (`NewSearchRequest`: optional `execution_id`, `created_by`, `query`, `images`) |
| `POST /search_integration/product_search_debug_new` | Debug product |

`SearchRequest` fields include `execution_id`, optional `created_by`, optional `query`, optional `image` (legacy single image).

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/search_integration/supplier_search" \
  -H "Content-Type: application/json" \
  -d '{"execution_id":"<execution-id>","created_by":"<user>","query":"<text>","image":"<url-or-path-or-base64>"}'
```

```bash
curl -sS -X POST "$DOCAGENT_AGENTS_API_BASE_URL/search_integration/product_search" \
  -H "Content-Type: application/json" \
  -d '{"execution_id":"<execution-id>","created_by":"<user>","query":"<text>","images":["<uri1>"]}'
```

Responses include `execution_id` and `status`. For retrieval routes below, substitute `{search_id}` with the identifier your integration uses—the OpenAPI paths name this parameter `search_id` even when its value echoes an `execution_id` from earlier steps.

| GET | `/search_integration/product_search/{search_id}` |
| GET | `/search_integration/product_search_execution/{search_id}` |
| GET | `/search_integration/supplier_search/{search_id}` |
| GET | `/search_integration/supplier_search_execution/{search_id}` |
| GET | `/search_integration/buyer_search/{search_id}` |
| GET | `/search_integration/buyer_search_execution/{search_id}` |
| GET | `/search_integration/factory_search/{search_id}` |
| GET | `/search_integration/factory_search_execution/{search_id}` |

Example:

```bash
curl -sS "$DOCAGENT_AGENTS_API_BASE_URL/search_integration/supplier_search/<search_id>"
```

## Deletes (destructive)

Only with explicit confirmation:

```bash
curl -sS -X DELETE "$DOCAGENT_AGENTS_API_BASE_URL/search_integration/product_search/<search_id>"
```

Same pattern for supplier, buyer, factory path segments.
