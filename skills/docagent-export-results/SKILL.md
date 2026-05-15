---
name: docagent-export-results
description: Exports extraction-related outputs via DocuAgent Agents API—export_data_to_blob (document-processing) and ConfigAgent export-extraction-excel (X-API-Key). Use when users need downloadable blob export from records or extraction Excel URLs (LFX SDU DocuAgent).
---

# DocuAgent export results (Agents API)

## Export dataframe / records to blob

**POST** `export_data_to_blob` (document-processing routes; full URL in example).

OpenAPI schemas: `DataframeExporterRequest`, `DataframeExporterResponse`.

Required: `records_to_export` (map order_id → doc_type), `output_format` (`xlsx`|`csv`|`json`|`tex`|`pkl`|`parquet`), `execution_id`, `created_by`.

```bash
curl -sS -X POST "https://api.uat.t4s.lfxdigital.app/agents/v1/air8_integration/export_data_to_blob" \
  -H "Content-Type: application/json" \
  -d '{
    "records_to_export": {"order-id-1":"invoice"},
    "output_format":"xlsx",
    "execution_id":"<execution-id>",
    "created_by":"<user-or-service>"
  }'
```

## Export extraction Excel (ConfigAgent — API key)

`POST /config_integration/export-extraction-excel`

Schemas: `ExportExtractionExcelRequest`, `ExportExtractionExcelResponse`. Requires `X-API-Key`.

```bash
curl -sS -X POST "https://api.uat.t4s.lfxdigital.app/agents/v1/config_integration/export-extraction-excel" \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $DOCAGENT_AGENTS_API_KEY" \
  -d '{"execution_id":"<execution-id>","created_by":"<user-or-service>"}'
```

Response includes `url` for download when successful.
