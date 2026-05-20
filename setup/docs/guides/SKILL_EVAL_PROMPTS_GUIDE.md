# SKILL_EVAL_PROMPTS_GUIDE

## Scope

Test these skills:

- `docagent-platform`
- `docagent-extraction`
- `docagent-results`

## Run sequence

1. Pick a test prompt.
2. Run the skill behavior.
3. Check output against `Expected behavior`.
4. Mark fail if any `Fail signals` appear.
5. Record pass/fail and notes.

---

## docagent-platform

### Test 1 - Missing extraction config

**Prompt**

`/docagent-extraction @/Users/me/Downloads/sample.pdf`

**Expected behavior**

- Routes to extraction workflow.
- Does not auto-run execution APIs.
- Performs preflight and asks for explicit `field_config_id`, `nation`, `possible_doc_type`.
- Defaults to UI handoff at `/document-extraction` with a confirmed value pack.

**Fail signals**

- Auto-selects config from filename.
- Starts create/upload/poll without confirmation.

### Test 2 - Results check with execution id only

**Prompt**

`check result for execution_id=abc-123`

**Expected behavior**

- Uses SDU fallback status poll path when only execution_id is available.
- Does not incorrectly claim "cannot check from this workspace".

**Fail signals**

- Refuses without attempting fallback.
- Routes to wrong API stack for this case.

---

## docagent-extraction

### Test 1 - File-only request

**Prompt**

`extract this file @/Users/me/Downloads/packing_list.pdf`

**Expected behavior**

- Runs preflight only first.
- Lists candidate configs and valid nation/doc_type pairs.
- Requires explicit confirmation before execution.
- Provides UI-first handoff with confirmed values.

**Fail signals**

- Silent config inference from filename.
- Reuses stale run params without confirmation.

### Test 2 - Confirmed run request

**Prompt**

`run now with field_config_id=<id>, nation=<n>, doc_type=<d>, order_id=ORD-1`

**Expected behavior**

- Executes create -> upload -> poll.
- Reports execution_id/order_id/output status clearly.

**Fail signals**

- Skips upload or stops polling at 202.
- Uses wrong host (`api.uat.doc-agent...`).

### Test 3 - Stale cached preference

**Prompt**

`use my cached extraction defaults`

**Expected behavior**

- Loads cache as suggestion.
- Validates cached config+nation+doc_type against current options.
- Falls back to preflight when stale.

**Fail signals**

- Uses stale cache without validation.

---

## docagent-results

### Test 1 - Recent runs

**Prompt**

`show my latest extraction runs in UAT`

**Expected behavior**

- Uses NestJS results routes on `uat.api.doc-agent.../v1`.
- Falls back to UI (`/results`) when host-side curl is blocked.

**Fail signals**

- Attempts Agents API for list endpoints.
- Uses wrong host order (`api.uat...`).

### Test 2 - Done status with bearer

**Prompt**

`is extraction <id> done?`

**Expected behavior**

- Polls `GET /execution/sdu-extraction-executions/{id}` until output or terminal failure.

**Fail signals**

- One-shot response without proper poll logic.

---

## Log template

```text
Date:
Tester:
Skill:
Test:
Pass/Fail:
Notes:
```
