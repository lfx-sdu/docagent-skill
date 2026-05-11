---
name: docagent-config-agent
description: "Build and debug DocuAgent config-agent experiences, including chat/streaming UI, configuration payloads, and backend config integrations. Use when requests mention config-agent route, config chatbot components, streaming output behavior, prompt/config field editing, or configuration API wiring."
---

# Docagent Config Agent

## Primary Frontend Areas

- `frontend/app/(routes)/config-agent`
- `frontend/app/components/configAgentChatbot`
- Theme/layout files that impact config-agent readability

## Primary Backend Areas

- `backend/llm-config-service/src/**` for configuration persistence and integration
- `backend/llm-chat-service/src/**` or external agent integration paths when streaming/chat is involved

## Workflow

1. Identify if issue is transport (streaming/API), contract (DTO), or presentation (UI state).
2. Validate request/response shape and streaming event format first.
3. Update frontend message/render pipeline to handle loading/chunk/complete/error explicitly.
4. Align configuration forms and payload schema with backend expectations.
5. Validate UX clarity: fewer controls, clear hierarchy, calm states, minimal noise.

## Non-Negotiables

- Never hide broken streaming with fake delayed UI updates.
- Keep config-agent interactions deterministic and inspectable.
- Preserve strong typing between frontend models and backend DTOs.
- Maintain accessible keyboard navigation and readable error feedback.

