---
name: docagent-config-updates
description: Manage DocuAgent field and checker configurations from /configurations. Use when users ask to create, edit, clone, reparent, delete, or browse field/checker configs, including form-vs-JSON editor updates and list filtering.
---

# Configuration updates (DocuAgent)

Use this skill for the **Configurations** workspace:

- URL: `https://uat.doc-agent.lfxdigital.app/configurations`
- Tabs: `fieldConfig` (Field configurations), `checkerConfig` (Checker configurations)

## Verified UAT bases

| Surface | Base |
|---------|------|
| Web app | `https://uat.doc-agent.lfxdigital.app/configurations` |
| Config API | `https://uat.api.doc-agent.lfxdigital.app/v1/config` |

Do not use `api.uat.doc-agent...` (wrong host order).

## Source-of-truth files

| Concern | Path |
|---------|------|
| Field config API client | `doc-agent/frontend/app/services/fieldConfigs/index.ts` |
| Checker config API client | `doc-agent/frontend/app/services/checkerConfigs/index.ts` |
| Field workspace behavior | `doc-agent/frontend/app/components/fieldConfigsCard/index.tsx` |
| Checker workspace behavior | `doc-agent/frontend/app/components/checkerConfigsCard/index.tsx` |
| Config route + tab query params | `doc-agent/frontend/app/(routes)/configurations/page.tsx` |

## Auth and safety

- Use Bearer token for `/v1/config/*` endpoints (`DOCAGENT_BEARER_TOKEN`, MSAL JWT, or team SA + `X-Actor-Email`).
- `DOCAGENT_AGENTS_API_KEY` is for SDU `/config_integration/*`, not these NestJS config CRUD routes.
- **Production caution:** updates/deletes/reparent mutate live configuration data. Confirm intent before write/delete calls.

## Network-verified on page load (UAT)

Field tab load calls:

- `GET /config/sdu-field-configs?page=0&size=50&sortOrder=desc&roots_only=true`
- `GET /config/sdu-field-configs/records/count?roots_only=true`

Checker tab load calls:

- `GET /config/sdu-checker-configs?page=0&size=25&sortOrder=desc`
- `GET /config/sdu-checker-configs/records/count`

## Field configuration API map

| Action | Method | Path |
|--------|--------|------|
| List | GET | `/config/sdu-field-configs` |
| Count | GET | `/config/sdu-field-configs/records/count` |
| Get by id | GET | `/config/sdu-field-configs/{id}` |
| Create | POST | `/config/sdu-field-configs` |
| Update | PATCH | `/config/sdu-field-configs/{id}` |
| Delete | DELETE | `/config/sdu-field-configs/{id}` |
| Children | GET | `/config/sdu-field-configs/{id}/children` |
| Reparent root | PATCH | `/config/sdu-field-configs/{id}/parent` body `{ "parent_id": "<id>" }` |

### Common field list filters

- `name` (search by config name)
- `id` (exact/partial id search)
- `doc_family`
- `parent_id`
- `roots_only` (default true in UI root list)
- pagination/sort: `page`, `size`, `sortOrder`

## Checker configuration API map

| Action | Method | Path |
|--------|--------|------|
| List | GET | `/config/sdu-checker-configs` |
| Count | GET | `/config/sdu-checker-configs/records/count` |
| Get by id | GET | `/config/sdu-checker-configs/{id}` |
| Create | POST | `/config/sdu-checker-configs` |
| Update | PATCH | `/config/sdu-checker-configs/{id}` |
| Delete | DELETE | `/config/sdu-checker-configs/{id}` |

## UI behavior notes (important for agents)

- `/configurations?tab=fieldConfig|checkerConfig&id=<configId>` deep-links to a specific editor target.
- Field workspace supports:
  - form input and JSON input edit modes
  - clone behavior
  - create child config
  - root reparent flow (modal + parent picker)
- Checker workspace supports form/JSON create/update/clone/delete.
- Sidebar lists are paginated and searchable; list + count calls should stay in sync.

## Safe execution order for updates

1. Fetch current config by id.
2. Diff against requested change.
3. Confirm destructive intent for delete/reparent.
4. Apply `PATCH` or `DELETE`.
5. Re-fetch list/count (and get-by-id when applicable) to verify.

## Practical curl examples (UAT)

```bash
# List field configs (roots)
curl -sS -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  "https://uat.api.doc-agent.lfxdigital.app/v1/config/sdu-field-configs?page=0&size=50&sortOrder=desc&roots_only=true"
```

```bash
# Update a checker config
curl -sS -X PATCH \
  -H "Authorization: Bearer $DOCAGENT_BEARER_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "name": "updated-checker", "rules": [] }' \
  "https://uat.api.doc-agent.lfxdigital.app/v1/config/sdu-checker-configs/<id>"
```

## Quick routing

| User asks… | Use |
|------------|-----|
| “update/create config” | This skill (`docagent-config-updates`) |
| “see extraction results / queue / runs” | `docagent-results` |
| “run extraction/upload docs” | `docagent-extraction` |

