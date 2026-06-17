# OpenHands Agent Server UI Reference

This file is the long-lived contract reference behind the `openhands-agent-server-ui` skill.

## Intended deployment model

This reference assumes the OpenHands agent-server and any SPA built from this skill are running either:

- on a local machine controlled by the user, or
- inside an authenticated remote environment where the browser client is already considered part of the same trust boundary as the server

This is not guidance for public internet clients, anonymous users, or multi-tenant browser applications.

## Server surfaces

A running OpenHands agent-server exposes three browser-facing surfaces:

1. **REST under `/api`** for conversations, events, tools, files, settings, bash operations, and other control surfaces.
2. **WebSockets under `/sockets`** for live conversation events and bash event streams.
3. **Root/discovery routes** such as `/alive`, `/health`, `/ready`, `/server_info`, `/docs`, `/redoc`, and `/openapi.json`.

If static files are mounted, `/` may redirect to `/static/` or `/static/index.html`. If static files are not mounted, `/` returns server info.

## Discovery checklist

Before writing UI logic, inspect the live server:

- `GET /alive` — process liveness
- `GET /health` — basic health
- `GET /ready` — initialization readiness
- `GET /server_info` — versions, title, usable tools, docs links
- `GET /docs` — interactive FastAPI docs
- `GET /redoc` — ReDoc documentation
- `GET /openapi.json` — machine-readable schema
- `GET /api/tools/` — currently registered tool names

If you expect a richer admin or workspace-management UI, also inspect:

- `POST /api/skills` — merged skill inventory for the workspace
- `POST /api/skills/sync` — force-refresh the public skill catalog cache from the configured source
- `GET /api/skills/marketplace` — installable marketplace catalog with installation status
- `POST /api/skills/install` — install a skill into `~/.openhands/skills/installed/`
- `GET /api/skills/installed` and `GET /api/skills/installed/{skill_name}` — installed-skill inventory and detail
- `PATCH /api/skills/installed/{skill_name}` and `POST /api/skills/installed/{skill_name}/refresh` — enable/disable or refresh an installed skill
- `DELETE /api/skills/installed/{skill_name}` — uninstall a skill
- `POST /api/hooks` — project hook configuration
- `GET /api/profiles`, `GET /api/profiles/{name}`, `POST /api/profiles/{name}`, `DELETE /api/profiles/{name}`, `POST /api/profiles/{name}/rename`, and `POST /api/profiles/{name}/activate` — optional profile-management UI
- `POST /api/mcp/test` — validate one MCP server config before persisting it
- `POST /api/auth/workspace-session` and `DELETE /api/auth/workspace-session` — mint or clear the workspace cookie used for browser embeds
- `GET /api/file/home` — server home directory for file pickers
- `GET /api/file/search_subdirs` — paged directory search for workspace pickers
- `GET /api/workspaces`, `POST /api/workspaces`, `DELETE /api/workspaces`, `POST /api/workspaces/parents`, and `DELETE /api/workspaces/parents` — persist shared workspace shortcuts and parent folders on the server
- `GET /api/git/changes` and `GET /api/git/diff` — optional changes views
- `GET /api/llm/providers`, `GET /api/llm/models`, and `GET /api/llm/models/verified` — optional model pickers
- `GET /api/conversations/{conversation_id}/workspace` and `GET /api/conversations/{conversation_id}/workspace/{file_path:path}` — serve workspace HTML/assets for embeds
- `GET /api/vscode/url`, `GET /api/vscode/status`, and `GET /api/desktop/url` — optional editor or desktop launch surfaces

Record `/server_info.version` early and decide whether your UI should hard-fail, soft-warn, or feature-gate when the server is older than the contract you expect. Handle compatibility explicitly instead of trying to infer partial support.

If the running server and repository docs differ, trust the live server contract first.

## Base URL patterns

### Same-origin UI

- REST base: `${window.location.origin}/api`
- WebSocket base: `ws://${window.location.host}` or `wss://${window.location.host}`

### Separate frontend origin

- REST base: `${SERVER_ORIGIN}/api`
- WebSocket base: `${SERVER_ORIGIN}` with `http` replaced by `ws`
- Cross-origin deployments require the server to allow the frontend origin via CORS.

### Proxy or path-prefix deployments

If the server is exposed behind a shared ingress prefix such as `/runtime/55313`, preserve that prefix for both REST and WebSocket URLs:

- REST base: `${SERVER_ORIGIN}/runtime/55313/api`
- WebSocket base: `${SERVER_ORIGIN}` with `http` replaced by `ws`, then append `/runtime/55313`

Do not derive WebSocket URLs from host alone in these deployments. Preserve the same path prefix used for `/api`.

## Authentication

### REST

When the server is configured with session API keys, endpoints under `/api` require:

```http
X-Session-API-Key: <your-session-api-key>
```

### WebSockets

Browser clients should prefer first-message auth. Open the socket, then immediately send:

```json
{"type":"auth","session_api_key":"<your-session-api-key>"}
```

For this skill's intended local or trusted-environment deployments, a browser-held session key can be acceptable. Do not reuse that pattern for public or multi-tenant browser clients.

Legacy fallbacks exist but are not the preferred browser path:

- `session_api_key` query parameter
- `X-Session-API-Key` WebSocket header for non-browser clients

### Workspace artifacts and embedded HTML

The server has a narrower cookie-based auth path for browser embeds that cannot attach custom headers, such as `<iframe src>` and `<img src>` requests.

- `POST /api/auth/workspace-session` mints a workspace session cookie after validating the `X-Session-API-Key` header and returns `204 No Content`.
- `DELETE /api/auth/workspace-session` clears that cookie and also returns `204 No Content`.
- The cookie is honored only by the workspace static-file routes under `/api/conversations/{conversation_id}/workspace...`.
- The cookie is scoped to `/api/conversations`, marked `HttpOnly`, and uses `SameSite=None`; the server adds `Secure` and `Partitioned` when the request context allows it.
- The rest of `/api` remains header-only to keep the broader CSRF surface closed.

## Key response shapes

### Success response

Many mutating endpoints return:

```json
{ "success": true }
```

### Paged response

List endpoints return:

```json
{
  "items": [],
  "next_page_id": null
}
```

### ConversationInfo highlights

Important UI fields often include:

```json
{
  "id": "uuid",
  "title": "optional title",
  "created_at": "timestamp",
  "updated_at": "timestamp",
  "execution_status": "idle|running|paused|waiting_for_confirmation|finished|error|stuck|deleting",
  "tags": {},
  "workspace": { "kind": "LocalWorkspace", "working_dir": "workspace/project" },
  "agent": {}
}
```

Conversation search/get/start/fork responses trim `agent.agent_context.skills` to `[]` by default. Only request `?include_skills=true` if the UI truly needs the legacy full payload.


### Event shape

Event history and event WebSockets emit serialized event objects with fields such as:

```json
{
  "kind": "ConcreteEventType",
  "id": "event-id",
  "timestamp": "iso-timestamp",
  "source": "user|agent|environment|..."
}
```

Treat `kind` as open-ended. New event variants may appear over time.

### SendMessageRequest over REST

The simplest write path is:

```json
{
  "role": "user",
  "content": [{ "type": "text", "text": "Build a todo app" }],
  "run": true
}
```

## Core REST API

### Discovery and tooling

- `GET /alive`
- `GET /health`
- `GET /ready`
- `GET /server_info`
- `GET /api/tools/`
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
- `DELETE /api/profiles/{name}`
- `POST /api/profiles/{name}/rename`
- `POST /api/profiles/{name}/activate`
- `POST /api/mcp/test`
- `POST /api/auth/workspace-session`
- `DELETE /api/auth/workspace-session`
- `GET /api/llm/providers`
- `GET /api/llm/models`
- `GET /api/llm/models/verified`

### Conversations

- `GET /api/conversations/search?page_id=<cursor>&limit=100&status=<status>&sort_order=<order>`
- `GET /api/conversations/count`
- `GET /api/conversations/{conversation_id}`
- `GET /api/conversations?ids=<uuid>&ids=<uuid>`
- `POST /api/conversations`
- `PATCH /api/conversations/{conversation_id}`
- `DELETE /api/conversations/{conversation_id}`
- `POST /api/conversations/{conversation_id}/run`
- `POST /api/conversations/{conversation_id}/pause`
- `POST /api/conversations/{conversation_id}/interrupt`
- `POST /api/conversations/{conversation_id}/fork`
- `POST /api/conversations/{conversation_id}/condense`
- `GET /api/conversations/{conversation_id}/agent_final_response`
- `POST /api/conversations/{conversation_id}/ask_agent`
- `POST /api/conversations/{conversation_id}/secrets`
- `POST /api/conversations/{conversation_id}/confirmation_policy`
- `POST /api/conversations/{conversation_id}/security_analyzer`
- `POST /api/conversations/{conversation_id}/switch_profile`
- `POST /api/conversations/{conversation_id}/switch_llm`
- `POST /api/conversations/{conversation_id}/switch_acp_model`

Important notes:

- `POST /api/conversations` returns full `ConversationInfo`, not only an ID.
- If the client wants to choose the ID, the field is `conversation_id`, not `id`.
- There is **no dedicated resume endpoint**. To start or resume execution, call `POST /api/conversations/{conversation_id}/run`.
- `POST /api/conversations/{conversation_id}/interrupt` is the immediate cancel path for an in-flight run; `POST /pause` pauses the conversation loop after the current work yields control.

### Events

- `GET /api/conversations/{conversation_id}/events/search`
- `GET /api/conversations/{conversation_id}/events/count`
- `GET /api/conversations/{conversation_id}/events/{event_id}`
- `GET /api/conversations/{conversation_id}/events?event_ids=<id>&event_ids=<id>`
- `POST /api/conversations/{conversation_id}/events`
- `POST /api/conversations/{conversation_id}/events/respond_to_confirmation`

Useful filters on event search include:

- `page_id`
- `limit` up to 100
- `kind`
- `source`
- `body`
- `sort_order=TIMESTAMP|TIMESTAMP_DESC`
- `timestamp__gte`
- `timestamp__lt`

### Files and workspace helpers

- `POST /api/file/upload?path=/absolute/path/in/workspace.txt`
- `GET /api/file/download?path=/absolute/path/in/workspace.txt`
- `GET /api/file/download-trajectory/{conversation_id}`
- `GET /api/file/home`
- `GET /api/file/search_subdirs?path=/absolute/path&limit=100&page_id=<cursor>`
- `GET /api/conversations/{conversation_id}/workspace`
- `GET /api/conversations/{conversation_id}/workspace/{file_path:path}`

Paths must be absolute for the `/api/file/*` helpers. The `/api/conversations/{conversation_id}/workspace...` routes are static-file serving routes rooted at that conversation's local workspace.

### Saved workspaces

- `GET /api/workspaces`
- `POST /api/workspaces`
- `DELETE /api/workspaces?path=/absolute/workspace/path`
- `POST /api/workspaces/parents`
- `DELETE /api/workspaces/parents?path=/absolute/parent/path`

Important notes:

- `GET /api/workspaces` returns both `workspaces` and `workspaceParents`.
- Workspace items use `parentPath` in JSON when a parent folder is present.
- The POST endpoints are idempotent and de-duplicate by `path`.
- Use `/api/file/home` and `/api/file/search_subdirs` to browse filesystem choices, then persist reusable shortcuts with `/api/workspaces...` so every client connected to the same server sees the same saved list.

### Bash / terminal endpoints

- `GET /api/bash/bash_events/search`
- `GET /api/bash/bash_events/{event_id}`
- `GET /api/bash/bash_events/`
- `POST /api/bash/start_bash_command`
- `POST /api/bash/execute_bash_command`
- `DELETE /api/bash/bash_events`

Useful request shape:

```json
{
  "command": "ls -la",
  "cwd": "/workspace/project",
  "timeout": 300
}
```

### Git, editor, and desktop helpers

- `GET /api/git/changes?path=/absolute/path/to/repo`
- `GET /api/git/diff?path=/absolute/path/to/file/or/repo`
- `GET /api/vscode/url?base_url=<optional-browser-base>`
- `GET /api/vscode/status`
- `GET /api/desktop/url?base_url=<optional-browser-base>`

`/api/vscode/status` is a lightweight capability check for whether a VS Code bridge is available before rendering an "Open in VS Code" action.

### Settings endpoints

Useful for admin or advanced configuration UIs:

- `GET /api/settings`
- `PATCH /api/settings`
- `GET /api/settings/agent-schema`
- `GET /api/settings/conversation-schema`
- `GET /api/settings/secrets`
- `PUT /api/settings/secrets`
- `GET /api/settings/secrets/{name}`
- `DELETE /api/settings/secrets/{name}`

### Profiles endpoints

Useful for profile pickers or trusted internal model-management UIs:

- `GET /api/profiles`
- `GET /api/profiles/{name}`
- `POST /api/profiles/{name}`
- `DELETE /api/profiles/{name}`
- `POST /api/profiles/{name}/rename`
- `POST /api/profiles/{name}/activate`

Important notes:

- `GET /api/profiles` returns both a `profiles` array and an `active_profile` field.
- `GET /api/profiles/{name}` defaults to a browser-safer shape where `config.api_key` is `null` and `api_key_set` indicates whether a secret exists.
- `GET /api/profiles/{name}` also supports `X-Expose-Secrets: encrypted` or `plaintext`, mirroring the settings secret-exposure patterns.
- `POST /api/profiles/{name}` saves or overwrites the named profile from a request body shaped like `{ "llm": { ... }, "include_secrets": true }`.
- `POST /api/profiles/{name}/activate` applies the stored LLM config to current agent settings and records that name as `active_profile`.

### MCP helper

- `POST /api/mcp/test`

Important notes:

- `POST /api/mcp/test` validates one candidate MCP server config without persisting it.
- `POST /api/mcp/test` returns HTTP 200 for both success and expected validation failures; use the JSON body's `ok` flag and `error_kind` instead of treating non-2xx status as the only failure signal.

### LLM subscription endpoints

For UIs that support ChatGPT subscription (Plus/Pro) login flows:

- `GET /api/llm/subscription/openai/models`
- `GET /api/llm/subscription/openai/status`
- `POST /api/llm/subscription/openai/device/start`
- `POST /api/llm/subscription/openai/device/poll`
- `POST /api/llm/subscription/openai/logout`

The device-login flow starts with `POST .../device/start` (returns a `user_code` and `verification_uri` for the user to visit), then polls `POST .../device/poll` until the login succeeds or times out. `GET .../status` returns safe connection state without tokens.

### OpenAI-compatible gateway

These routes live at the server root, **not** under `/api`:

- `GET /v1/models`
- `POST /v1/chat/completions`

They expose an OpenAI-compatible chat completions interface backed by the agent server. Useful for integrating with tools or clients that speak the OpenAI protocol.

### Skills and hooks request shapes

Typical request bodies:

```json
{
  "load_public": true,
  "load_user": true,
  "load_project": true,
  "load_org": false,
  "project_dir": "/workspace/project"
}
```

```json
{
  "project_dir": "/workspace/project"
}
```

Use the active conversation's `workspace.working_dir` when you want skills or hooks for that workspace instead of a generic server default.

Installed-skill management uses separate request bodies:

```json
{
  "source": "github:OpenHands/extensions/skills/github",
  "ref": "main",
  "force": false
}
```

```json
{
  "enabled": true
}
```

Useful installed-skill response fields include `name`, `enabled`, `source`, `resolved_ref`, `installed_at`, and `install_path`. The marketplace catalog returns available skills plus installation status so a UI can render install or update actions without re-deriving that state.

## WebSocket API

### Conversation events stream

Main live update channel:

```text
WS /sockets/events/{conversation_id}
```

Supported replay query parameters:

- `resend_mode=all`
- `resend_mode=since&after_timestamp=<ISO-8601 timestamp>`

Best practice:

- store the latest rendered event timestamp
- reconnect with `resend_mode=since`
- de-duplicate by event `id`

The socket can also accept inbound message payloads after authentication, but the server validates those payloads as `Message` objects, not the REST `SendMessageRequest` shape with `run`. For browser SPAs, REST remains the simpler write path.

### Bash events stream

```text
WS /sockets/bash-events
```

Supports:

- first-message auth
- optional `resend_mode=all`
- inbound `ExecuteBashRequest` payloads to start commands

## Security and secret-handling notes

### Browser-safe defaults

Do not default to shipping raw LLM or provider API keys inside browser bundles, local storage, URLs, or logs.

For this skill's intended trust boundary, browser access to a session API key can be acceptable as an internal convenience. Even then, treat it as sensitive: avoid exposing it to unnecessary scripts, logs, screenshots, or storage.

Safer patterns:

1. **Trusted/internal admin UI** — acceptable when the UI and users are in the same trust boundary as the server.
2. **Server-managed settings** — let the server persist settings and secrets instead of asking the browser to own them.
3. **Encrypted round-tripping** — `GET /api/settings` supports `X-Expose-Secrets: encrypted`, and the resulting encrypted values can be sent back in agent settings with `secrets_encrypted: true` for trusted authenticated clients.

### Important trust-model nuance

The agent-server does **not** provide role-based authorization for secret exposure modes. Any client that can authenticate with the session API key is treated as part of the same trust domain.

That means:

- `X-Expose-Secrets: plaintext` is backend-only in practice
- `X-Expose-Secrets: encrypted` is safer for browser round-tripping, but still assumes an authenticated trusted client
- public multi-tenant browser clients should be designed very carefully

## Recommended SPA architecture

Prefer a thin client with:

- `fetch()` for REST writes and initial reads
- one active conversation events WebSocket for the selected conversation
- normalized client state keyed by conversation ID and event ID
- optimistic UI only where it clearly improves UX
- a renderer that branches by `event.kind`

A strong initial feature order is:

1. health and discovery panel
2. conversation list
3. conversation create form
4. conversation detail pane
5. event history view
6. live event stream
7. send message composer
8. status badges
9. confirmation UI
10. optional file transfer
11. optional bash panel

## Maintenance sources in the SDK repo

When refreshing this reference, inspect these files first:

- `openhands-agent-server/openhands/agent_server/api.py`
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
- `openhands-agent-server/openhands/agent_server/tool_router.py`
- `openhands-agent-server/openhands/agent_server/auth_router.py`
- `openhands-agent-server/openhands/agent_server/workspace_router.py`
- `openhands-agent-server/openhands/agent_server/workspaces_router.py`
- `openhands-agent-server/openhands/agent_server/git_router.py`
- `openhands-agent-server/openhands/agent_server/vscode_router.py`
- `openhands-agent-server/openhands/agent_server/desktop_router.py`
- `openhands-agent-server/openhands/agent_server/llm_router.py`
- `openhands-agent-server/openhands/agent_server/openai/router.py`

Also check the current browser-facing examples in `software-agent-sdk`:

- `scripts/agent_server_ui/static/index.html`
- `scripts/agent_server_ui/static/app.js`
- `scripts/websocket_client.html`
- `examples/02_remote_agent_server/11_conversation_fork.py`
- `examples/02_remote_agent_server/12_settings_and_secrets_api.py`
- `examples/02_remote_agent_server/13_workspace_get_llm.py`
- `examples/02_remote_agent_server/14_client_defined_tools.py`
- `examples/02_remote_agent_server/15_openai_compatible_gateway.py`

Treat those client-side examples as implementation inspiration, not as a stronger source of truth than the current router code or live OpenAPI schema.
