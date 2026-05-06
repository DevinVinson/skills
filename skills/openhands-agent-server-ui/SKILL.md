---
name: openhands-agent-server-ui
description: Build browser UIs and single-page apps against a running OpenHands agent-server by using its live REST, OpenAPI, and WebSocket contracts instead of reverse-engineering the repository. Use when the user wants an OpenHands chat UI, dashboard, admin console, or SPA that talks to agent-server.
triggers:
- openhands agent-server ui
- agent-server ui
- agent-server spa
- openhands chat ui
- remote conversation ui
---

# OpenHands Agent Server UI

Use this skill when building or extending a browser UI against an **already running OpenHands agent-server**.

This is a **maintained reference skill for code-writing work**. It is intentionally more durable than a quick everyday skill. Keep it aligned with the running server and the `software-agent-sdk` repo.

## Intended trust boundary

This skill assumes the agent-server and the browser UI are running either:

- on a local machine controlled by the user, or
- inside an authenticated remote environment where the UI users are already trusted and share the same trust boundary as the server

It is **not** a pattern for public, anonymous, or multi-tenant browser deployments.

## First principles

- Trust the running server's `/docs` and `/openapi.json` first if there is any contract drift.
- Use `/server_info` and `/api/tools/` to adapt to the deployment you actually have.
- Keep raw event objects and render by `kind` instead of flattening everything into plain strings.
- Prefer REST for writes and initial reads, and WebSocket for live updates.

## Workflow

### 1. Discover the live server

Inspect these routes before building UI behavior:

- `GET /alive`
- `GET /health`
- `GET /ready`
- `GET /server_info`
- `GET /docs`
- `GET /openapi.json`
- `GET /api/tools/`

### 2. Establish base URLs and auth

- Same-origin SPA: REST base is typically `${window.location.origin}/api`
- Same-origin WebSocket base is typically `ws(s)://${window.location.host}`
- Browser WebSocket auth should use first-message auth with:

```json
{"type":"auth","session_api_key":"<your-session-api-key>"}
```

- REST auth uses `X-Session-API-Key`

### 3. Build the core UX first

Start with:

1. conversation list
2. conversation detail
3. conversation creation
4. event history loading
5. live event stream
6. send message flow
7. status badges
8. confirmation UI

### 4. Add advanced surfaces only when needed

Only add these when the product actually needs them:

- file upload/download
- bash or terminal panel
- settings/admin panels
- profile switching or advanced security controls

## Guardrails

- Do **not** invent SSE or EventSource support. The real-time contract is WebSocket-based.
- Do **not** assume browsers can send custom WebSocket headers. Use first-message auth.
- Do **not** invent a `/resume` endpoint. Resuming uses `POST /api/conversations/{conversation_id}/run`.
- Do **not** assume `POST /api/conversations` returns only an ID. It returns full `ConversationInfo`.
- Do **not** expose raw LLM or provider API keys in browser code unless the UI is inside the intended trusted environment for this skill.
- Treat browser-held session API keys as trusted-environment-only convenience, not as a safe default for public or multi-tenant clients.
- For browser-based settings flows, prefer trusted/internal UIs, server-managed settings, or encrypted settings round-tripping instead of raw plaintext secrets.

## Companion files

- See [REFERENCE.md](./REFERENCE.md) for routes, contracts, auth, and maintenance notes.
- See [EXAMPLES.md](./EXAMPLES.md) for practical JavaScript snippets and payloads.

## Maintenance

When updating this skill, re-check these sources in `software-agent-sdk`:

- `openhands-agent-server/openhands/agent_server/api.py`
- `openhands-agent-server/openhands/agent_server/conversation_router.py`
- `openhands-agent-server/openhands/agent_server/event_router.py`
- `openhands-agent-server/openhands/agent_server/sockets.py`
- `openhands-agent-server/openhands/agent_server/file_router.py`
- `openhands-agent-server/openhands/agent_server/bash_router.py`
- `openhands-agent-server/openhands/agent_server/server_details_router.py`

If repo docs, examples, and the running server disagree, prefer the running server's `/docs` and `/openapi.json`, then the current router source.
