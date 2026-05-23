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
- Use `/server_info`, `/api/tools/`, and, when relevant, `/api/skills`, `/api/hooks`, `/api/profiles`, `/api/workspaces`, and `/api/mcp/test` to adapt to the deployment you actually have.
- Keep raw event objects and render by `kind` instead of flattening everything into plain strings.
- Prefer REST for writes and initial reads, and WebSocket for live updates.
- Do not assume the server is always mounted at the origin root. Reverse proxies may expose `/api` and `/sockets` under a shared path prefix.
- Surface server-version incompatibility early by checking `/server_info.version` before enabling richer UI features.

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

Capture the server version from `/server_info` and decide up front whether your UI should hard-fail, soft-warn, or feature-gate when the server is older than the contract you expect. Handle compatibility explicitly instead of guessing from partial failures.

If you want richer admin or workspace-management surfaces, also inspect these optional endpoints:

- `POST /api/skills`
- `POST /api/skills/sync`
- `GET /api/skills/marketplace`
- `POST /api/skills/install`
- `GET /api/skills/installed`
- `GET /api/skills/installed/{skill_name}`
- `PATCH /api/skills/installed/{skill_name}`
- `DELETE /api/skills/installed/{skill_name}`
- `POST /api/skills/installed/{skill_name}/refresh`
- `POST /api/hooks`
- `GET /api/profiles`
- `GET /api/profiles/{name}`
- `POST /api/profiles/{name}`
- `POST /api/profiles/{name}/activate`
- `POST /api/mcp/test`
- `POST /api/cloud-proxy`
- `POST /api/auth/workspace-session`
- `DELETE /api/auth/workspace-session`
- `GET /api/file/home`
- `GET /api/file/search_subdirs`
- `GET /api/workspaces`
- `POST /api/workspaces`
- `DELETE /api/workspaces`
- `POST /api/workspaces/parents`
- `DELETE /api/workspaces/parents`
- `GET /api/git/changes`
- `GET /api/git/diff`
- `GET /api/llm/providers`
- `GET /api/llm/models`
- `GET /api/llm/models/verified`
- `GET /api/conversations/{conversation_id}/workspace`
- `GET /api/conversations/{conversation_id}/workspace/{file_path:path}`
- `GET /api/vscode/url`
- `GET /api/vscode/status`
- `GET /api/desktop/url`

### 2. Establish base URLs and auth

- Same-origin SPA mounted at the server root: REST base is typically `${window.location.origin}/api`
- Same-origin WebSocket base is typically `ws(s)://${window.location.host}`
- If the server is exposed behind a proxy path such as `/runtime/<port>`, preserve that same prefix for both REST and WebSocket URLs.
- Browser WebSocket auth should use first-message auth with:

```json
{"type":"auth","session_api_key":"<your-session-api-key>"}
```

- REST auth uses `X-Session-API-Key`
- Workspace artifact routes under `/api/conversations/{conversation_id}/workspace...` can also use the workspace session cookie minted by `POST /api/auth/workspace-session`; those auth endpoints return `204 No Content`, and the cookie is not honored by the rest of `/api`

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
- file or workspace picker via `/api/file/home` and `/api/file/search_subdirs`
- persisted workspace shortcuts via `/api/workspaces` and `/api/workspaces/parents`
- skills or hook inspection via `/api/skills` and `/api/hooks`
- installed-skill or marketplace UI via `/api/skills/marketplace`, `/api/skills/install`, and `/api/skills/installed...`
- profile management via `/api/profiles...`
- MCP configuration validation via `POST /api/mcp/test`
- same-origin cloud forwarding via `POST /api/cloud-proxy`
- git changes or diff panels via `/api/git/changes` and `/api/git/diff`
- model or provider pickers via `/api/llm/providers`, `/api/llm/models`, and `/api/llm/models/verified`
- VS Code or desktop launchers via `/api/vscode/url`, `/api/vscode/status`, and `/api/desktop/url`
- bash or terminal panel
- settings/admin panels
- advanced security controls

## Guardrails

- Do **not** invent SSE or EventSource support. The real-time contract is WebSocket-based.
- Do **not** assume browsers can send custom WebSocket headers. Use first-message auth.
- Do **not** invent a `/resume` endpoint. Resuming uses `POST /api/conversations/{conversation_id}/run`.
- Do **not** assume `POST /api/conversations` returns only an ID. It returns full `ConversationInfo`.
- Do **not** assume conversation fetches include the full inlined skill catalog. `ConversationInfo` responses trim `agent.agent_context.skills` to `[]` unless you opt in with `?include_skills=true`.
- Use `POST /api/conversations/{conversation_id}/interrupt` when the UX needs an immediate stop for an in-flight LLM/tool request; `POST /pause` waits for the current call to finish.
- Do **not** expose raw LLM or provider API keys in browser code unless the UI is inside the intended trusted environment for this skill.
- Treat browser-held session API keys as trusted-environment-only convenience, not as a safe default for public or multi-tenant clients.
- For browser-based settings flows, prefer trusted/internal UIs, server-managed settings, or encrypted settings round-tripping instead of raw plaintext secrets.

## Companion files

- See [REFERENCE.md](./REFERENCE.md) for routes, contracts, auth, and maintenance notes.
- See [EXAMPLES.md](./EXAMPLES.md) for practical JavaScript snippets and payloads.

## Maintenance

When updating this skill, re-check these sources in `software-agent-sdk`:

- `openhands-agent-server/openhands/agent_server/api.py`
- `openhands-agent-server/openhands/agent_server/auth_router.py`
- `openhands-agent-server/openhands/agent_server/conversation_router.py`
- `openhands-agent-server/openhands/agent_server/event_router.py`
- `openhands-agent-server/openhands/agent_server/sockets.py`
- `openhands-agent-server/openhands/agent_server/file_router.py`
- `openhands-agent-server/openhands/agent_server/bash_router.py`
- `openhands-agent-server/openhands/agent_server/server_details_router.py`
- `openhands-agent-server/openhands/agent_server/settings_router.py`
- `openhands-agent-server/openhands/agent_server/skills_router.py`
- `openhands-agent-server/openhands/agent_server/hooks_router.py`
- `openhands-agent-server/openhands/agent_server/profiles_router.py`
- `openhands-agent-server/openhands/agent_server/mcp_router.py`
- `openhands-agent-server/openhands/agent_server/cloud_proxy_router.py`
- `openhands-agent-server/openhands/agent_server/git_router.py`
- `openhands-agent-server/openhands/agent_server/vscode_router.py`
- `openhands-agent-server/openhands/agent_server/desktop_router.py`
- `openhands-agent-server/openhands/agent_server/llm_router.py`
- `openhands-agent-server/openhands/agent_server/models.py`
- `openhands-agent-server/openhands/agent_server/openapi.py`
- `openhands-agent-server/openhands/agent_server/workspace_router.py`
- `openhands-agent-server/openhands/agent_server/workspaces_router.py`

Also re-check the current browser-facing examples in `software-agent-sdk`:

- `scripts/agent_server_ui/static/index.html`
- `scripts/agent_server_ui/static/app.js`
- `examples/02_remote_agent_server/11_conversation_fork.py`
- `examples/02_remote_agent_server/12_settings_and_secrets_api.py`
- `examples/02_remote_agent_server/13_workspace_get_llm.py`

If repo docs, examples, and the running server disagree, prefer the running server's `/docs` and `/openapi.json`, then the current router source.
