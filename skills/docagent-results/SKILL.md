---
name: docagent-results
description: Cross-cutting retrieval and polling for DocuAgent Agents API—execution_id status for document-processing jobs, LFSearch GET by search_id, ConfigAgent threads/messages/jobs/embeddings statuses. Use when correlating IDs across extraction, search, config, or NER tasks (LFX SDU DocuAgent).
---

# DocuAgent results (IDs & GET patterns)

Everything here is **GET** unless noted. Compose URLs with `https://api.uat.t4s.lfxdigital.app/agents/v1`.

## Document processing — execution lifecycle

Poll any async document-processing job using:

**GET** `check_execution_status?execution_id=<execution_id>` (full URL in recipe below).

Company research retrieval:

**GET** `get_company_info_and_news_by_id?execution_id=<execution_id>`

(Starting company research: **POST** per OpenAPI—for example `search_company_info_and_news` under the same document-processing route group as extraction.)

## LFSearch — fetch by `search_id`

Replace `{kind}` with `product_search`, `supplier_search`, `buyer_search`, or `factory_search`.

| GET | `/search_integration/{kind}/{search_id}` | Result payload |
| GET | `/search_integration/{kind}_execution/{search_id}` | Execution record |

**DELETE** `/search_integration/{kind}/{search_id}` — destructive; confirm with user before calling.

## ConfigAgent — requires `X-API-Key`

| GET | `/config_integration/fetch-dialog/{thread_id}` |
| GET | `/config_integration/fetch-message/{thread_id}/{message_id}` |
| GET | `/config_integration/fetch-threads/{user_id}?limit=&skip=` |
| GET | `/config_integration/config-job/{job_id}` |
| GET | `/config_integration/shipment-code-embeddings-status/{job_id}` |
| GET | `/config_integration/invoice-code-embeddings-status/{job_id}` |

## NER pipelines — poll task status

| GET | `/ner_integration/ner/entity-pipeline/status/{task_id}` |
| GET | `/ner_integration/ner/entity-pipeline/tasks?status=&limit=` |
| GET | `/ner_integration/ner/product-pipeline/status/{task_id}` |
| GET | `/ner_integration/ner/product-pipeline/tasks?status=&limit=` |

## Practices

- Pass through **`execution_id`**, **`search_id`**, **`thread_id`**, **`job_id`**, **`task_id`** from POST responses verbatim.
- For list endpoints with `limit`/`skip`, keep pagination deterministic when scraping all pages.

## Quick retrieval recipes

Extraction status:

```bash
curl -sS "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/check_execution_status?execution_id=<execution_id>"
```

ConfigAgent dialog:

```bash
curl -sS \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  "https://api.uat.t4s.lfxdigital.app/agents/v1/config_integration/fetch-dialog/<thread_id>"
```

ConfigAgent async job:

```bash
curl -sS \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  "https://api.uat.t4s.lfxdigital.app/agents/v1/config_integration/config-job/<job_id>"
```
