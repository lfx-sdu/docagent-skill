# EXTRACTION_PREFERENCE_CACHE_GUIDE

## Purpose

Store per-user extraction defaults locally. Keep config selection explicit and validated.

## Cache location

- Directory: `~/.cache/docagent-skills/preferences/`
- File: `~/.cache/docagent-skills/preferences/<user_key>.json`

Do not store cache files in this repository.

## JSON schema (example)

```json
{
  "version": 1,
  "updated_at": "2026-05-20T16:30:00Z",
  "tenant": "uat",
  "extraction_defaults": {
    "field_config_id": "ai-gen-9d3d1417-f1aa-413c-ab95-5fe0dd420200",
    "nation": "general",
    "possible_doc_type": [
      "packing_list"
    ],
    "use_parent_config": false,
    "parent_config_id": null,
    "flash_mode": true,
    "validate_doc_type": false
  }
}
```

## Helper functions (copy-paste)

```bash
#!/usr/bin/env bash
set -euo pipefail

CACHE_DIR="${HOME}/.cache/docagent-skills/preferences"
USER_KEY="${DOCAGENT_USER_KEY:-default}"
CACHE_FILE="${CACHE_DIR}/${USER_KEY}.json"

load_pref() {
  mkdir -p "${CACHE_DIR}"
  if [ -f "${CACHE_FILE}" ]; then
    cat "${CACHE_FILE}"
  else
    echo "{}"
  fi
}

save_pref() {
  local field_config_id="$1"
  local nation="$2"
  local doc_type="$3"
  local use_parent_config="${4:-false}"
  local parent_config_id="${5:-null}"
  local flash_mode="${6:-true}"
  local validate_doc_type="${7:-false}"
  local tenant="${8:-uat}"

  mkdir -p "${CACHE_DIR}"

  cat > "${CACHE_FILE}" <<EOF
{
  "version": 1,
  "updated_at": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")",
  "tenant": "${tenant}",
  "extraction_defaults": {
    "field_config_id": "${field_config_id}",
    "nation": "${nation}",
    "possible_doc_type": ["${doc_type}"],
    "use_parent_config": ${use_parent_config},
    "parent_config_id": ${parent_config_id},
    "flash_mode": ${flash_mode},
    "validate_doc_type": ${validate_doc_type}
  }
}
EOF
}

validate_pref() {
  local field_config_id="$1"
  local nation="$2"
  local doc_type="$3"
  local base_url="${DOCAGENT_NESTJS_BASE_URL:-https://uat.api.doc-agent.lfxdigital.app/v1}"

  if [ -z "${DOCAGENT_BEARER_TOKEN:-}" ]; then
    echo "validate_pref: DOCAGENT_BEARER_TOKEN is required" >&2
    return 2
  fi

  curl -sS \
    -H "Authorization: Bearer ${DOCAGENT_BEARER_TOKEN}" \
    "${base_url}/config/sdu-field-configs/${field_config_id}" \
    | python3 -c '
import json, sys
cfg = json.load(sys.stdin)
nation = sys.argv[1]
doc_type = sys.argv[2]
fc = cfg.get("fieldConfig") or {}
sections = fc.get(nation)
if not isinstance(sections, dict):
    print("invalid: nation not found in fieldConfig")
    raise SystemExit(1)
if doc_type not in sections:
    print("invalid: doc_type not found under nation")
    raise SystemExit(1)
print("valid")
' "${nation}" "${doc_type}" >/dev/null
}
```

## Example usage

```bash
if validate_pref "$FIELD_CONFIG_ID" "$NATION" "$DOC_TYPE"; then
  echo "cached preference is valid"
else
  echo "cached preference is stale; run preflight"
fi
```

## Validation rules

1. Read cache and propose defaults to user.
2. Validate cached `field_config_id`, `nation`, and `possible_doc_type` against current selectable options.
   - Reference parser: `skills/docagent-extraction/SKILL.md` preflight "Safe list-config script".
3. If invalid or stale, ignore cache and rerun preflight.
4. Save cache only after user confirms values.

## Security rules

- Never cache `DOCAGENT_BEARER_TOKEN` or API keys.
- Never cache raw document payloads.
- Keep cache local to the user home directory.
