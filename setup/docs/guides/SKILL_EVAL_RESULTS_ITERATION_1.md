# SKILL_EVAL_RESULTS_ITERATION_1

Date: 2026-05-20
Tester: Codex agent

## Scope

- `docagent-platform`
- `docagent-extraction`
- `docagent-results`

Method:

1. Verify skill contracts against `SKILL_EVAL_PROMPTS_GUIDE.md`.
2. Run minimal live host checks.

## Runtime checks (live)

- SDU health (`https://api.uat.t4s.lfxdigital.app/agents/v1/health`) -> `200`
- UAT NestJS auth status endpoint (`https://uat.api.doc-agent.lfxdigital.app/v1/execution/admin/auth/status`) -> `403` (reachable without bearer)
- Wrong host check (`https://api.uat.doc-agent.lfxdigital.app/...`) -> DNS failure (`curl: Could not resolve host`)

## Results

### docagent-platform

1. Extraction request with missing config -> **PASS**
   - Evidence: routes extraction to confirmation mode before execution when params are missing.
2. Results check with only execution id -> **PASS**
   - Evidence: explicit SDU fallback route for execution-id-only case.

### docagent-extraction

1. File-only extraction request -> **PASS**
   - Evidence: mandatory execution gate, preflight-first, explicit confirmation, no filename auto-assignment.
2. Confirmed execution request -> **PASS**
   - Evidence: includes full create -> upload -> poll sequence and polling semantics.
3. Cached preference mismatch -> **PASS**
   - Evidence: cache section requires live validation and fallback to preflight when stale.

### docagent-results

1. Recent runs in UAT -> **PASS**
   - Evidence: uses NestJS `uat.api.../v1` for list routes and warns against `api.uat...`.
2. Done status with bearer -> **PASS**
   - Evidence: polls `GET /execution/sdu-extraction-executions/{id}` until output or terminal failure.

## Next step

Run prompt-by-prompt behavior replay and append transcript evidence per test case.
