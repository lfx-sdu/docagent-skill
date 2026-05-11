# DocuAgent Skills

Reusable agent skills for the DocuAgent platform, compatible with the Vercel Skills CLI.

## Install

```bash
# install all skills from this repo
npx skills add <owner>/docagent-skills

# install selected skills
npx skills add <owner>/docagent-skills --skill docagent-platform --skill docagent-extraction

# install globally for your machine
npx skills add <owner>/docagent-skills -g

# install to specific agents
npx skills add <owner>/docagent-skills -a cursor -a codex -a claude-code
```

## Included Skills

- `docagent-platform` - router/orchestrator for DocuAgent domain skills
- `docagent-extraction` - extraction execution workflows
- `docagent-results` - shared result retrieval contracts and consistency
- `docagent-content-check` - content-check execution workflows
- `docagent-config-agent` - config-agent chat/streaming/config UX
- `docagent-export-results` - export execution and export result surfaces
- `docagent-admin-kpis` - admin KPI and jobs analytics flows

## Repository Layout

```text
skills/
  docagent-platform/SKILL.md
  docagent-extraction/SKILL.md
  docagent-results/SKILL.md
  docagent-content-check/SKILL.md
  docagent-config-agent/SKILL.md
  docagent-export-results/SKILL.md
  docagent-admin-kpis/SKILL.md
```

