# OpenHands Agent Server UI Examples

These examples assume the agent-server and SPA are running either on a local machine or inside an authenticated trusted environment. They are not intended as copy-paste patterns for public or multi-tenant browser deployments.

## Same-origin base URLs

```js
const apiBaseUrl = `${window.location.origin}/api`;
const wsBaseUrl =
  window.location.protocol === 'https:'
    ? `wss://${window.location.host}`
    : `ws://${window.location.host}`;
```

## Separate frontend origin

```js
const SERVER_ORIGIN = 'https://your-agent-server.example.com';
const apiBaseUrl = `${SERVER_ORIGIN}/api`;
const wsBaseUrl = SERVER_ORIGIN.replace(/^http/, 'ws');
```

## Separate frontend origin with proxy path prefix

Use this when the server sits behind an ingress path such as `/runtime/55313`.

```js
const SERVER_ORIGIN = 'https://your-gateway.example.com';
const PATH_PREFIX = '/runtime/55313';
const apiBaseUrl = `${SERVER_ORIGIN}${PATH_PREFIX}/api`;
const wsBaseUrl = `${SERVER_ORIGIN.replace(/^http/, 'ws')}${PATH_PREFIX}`;
```

Do not drop `PATH_PREFIX` when building WebSocket URLs.


## REST helper with session API key

Use this pattern only when the browser client is already inside the same trust boundary as the server, such as a local machine or trusted remote workspace.

```js
const sessionApiKey = window.sessionApiKey ?? null;

async function api(path, options = {}) {
  const headers = new Headers(options.headers || {});

  if (!headers.has('Content-Type') && !(options.body instanceof FormData)) {
    headers.set('Content-Type', 'application/json');
  }
  if (sessionApiKey) {
    headers.set('X-Session-API-Key', sessionApiKey);
  }

  const response = await fetch(`${apiBaseUrl}${path}`, {
    ...options,
    headers,
  });

  if (!response.ok) {
    const text = await response.text();
    throw new Error(`${response.status} ${text}`);
  }

  const contentType = response.headers.get('content-type') || '';
  if (contentType.includes('application/json')) {
    return response.json();
  }
  return response;
}
```

## Mint a workspace session cookie for embeds

Use this only for same-origin or otherwise trusted browser clients that already hold the session API key. This is specifically for workspace artifact routes such as `<iframe src>` and `<img src>`, where the browser cannot attach `X-Session-API-Key` itself.

```js
async function enableWorkspaceEmbeds() {
  const response = await fetch(`${apiBaseUrl}/auth/workspace-session`, {
    method: 'POST',
    headers: {
      'X-Session-API-Key': sessionApiKey,
    },
    credentials: 'include',
  });

  if (!response.ok) {
    throw new Error(await response.text());
  }
}

function workspaceUrl(conversationId, filePath = '') {
  const encodedPath = filePath
    .split('/')
    .filter(Boolean)
    .map(encodeURIComponent)
    .join('/');

  return `${apiBaseUrl}/conversations/${conversationId}/workspace${encodedPath ? `/${encodedPath}` : ''}`;
}

await enableWorkspaceEmbeds();
document.getElementById('artifact-frame').src = workspaceUrl(
  conversationId,
  'reports/index.html',
);
```

That cookie route returns `204 No Content`, and the cookie is only honored by `/api/conversations/{conversation_id}/workspace...`, not by the rest of `/api`.

## Read server info and gate on version

Tie the minimum version to your frontend release instead of guessing at compatibility.

```js
function compareSemver(left, right) {
  const a = left.replace(/^v/, '').split('.').map(Number);
  const b = right.replace(/^v/, '').split('.').map(Number);

  for (let i = 0; i < 3; i += 1) {
    if ((a[i] ?? 0) > (b[i] ?? 0)) return 1;
    if ((a[i] ?? 0) < (b[i] ?? 0)) return -1;
  }
  return 0;
}

async function getServerInfo() {
  const response = await fetch(`${apiBaseUrl.replace(/\/api$/, '')}/server_info`);
  if (!response.ok) throw new Error(await response.text());
  return response.json();
}

const minimumSupportedVersion = '1.17.0';
const serverInfo = await getServerInfo();

if (compareSemver(serverInfo.version, minimumSupportedVersion) < 0) {
  throw new Error(
    `Agent server ${serverInfo.version} is too old for this UI. ` +
      `Need ${minimumSupportedVersion} or newer.`,
  );
}

console.log(serverInfo.usable_tools);
```

## Check deferred-init status

Most UIs only need to detect this state and show that the runtime is starting. The trusted orchestrator, not an ordinary browser user, should normally call `POST /api/init`.

```js
async function getInitStatus() {
  const response = await fetch(`${apiBaseUrl}/init`);
  if (response.status === 404) return { state: 'not_configured' };
  if (!response.ok) throw new Error(await response.text());
  return response.json();
}

const initStatus = await getInitStatus();
if (initStatus.state === 'dormant' || initStatus.state === 'initializing') {
  console.log('runtime is starting', initStatus);
}
```

```js
async function initializeDormantServer(initApiKey, payload) {
  const response = await fetch(`${apiBaseUrl}/init`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-Init-API-Key': initApiKey,
    },
    body: JSON.stringify(payload),
  });
  if (!response.ok) throw new Error(await response.text());
  return response.json();
}
```

## Connect to the conversation events socket

This example assumes the session API key is being supplied by a trusted local or authenticated environment.

```js
function connectConversationEvents(conversationId, afterTimestamp = null) {
  const params = new URLSearchParams();
  if (afterTimestamp) {
    params.set('resend_mode', 'since');
    params.set('after_timestamp', afterTimestamp);
  }

  const url = `${wsBaseUrl}/sockets/events/${conversationId}${params.toString() ? `?${params}` : ''}`;
  const ws = new WebSocket(url);

  ws.addEventListener('open', () => {
    if (sessionApiKey) {
      ws.send(JSON.stringify({
        type: 'auth',
        session_api_key: sessionApiKey,
      }));
    }
  });

  ws.addEventListener('message', (event) => {
    const payload = JSON.parse(event.data);
    console.log('event', payload.kind, payload);
  });

  return ws;
}
```

## Search workspace subdirectories

```js
const home = await api('/file/home');
const subdirs = await api(
  `/file/search_subdirs?path=${encodeURIComponent(home.home)}&limit=20`,
);
console.log(subdirs.items);
```

## Persist saved workspaces and parent folders

```js
const saved = await api('/workspaces');
console.log(saved.workspaces, saved.workspaceParents);

await api('/workspaces/parents', {
  method: 'POST',
  body: JSON.stringify({
    parents: [
      {
        id: '/workspace',
        name: 'Workspace root',
        path: '/workspace',
      },
    ],
  }),
});

await api('/workspaces', {
  method: 'POST',
  body: JSON.stringify({
    workspaces: [
      {
        id: '/workspace/project',
        name: 'project',
        path: '/workspace/project',
        parentPath: '/workspace',
      },
    ],
  }),
});
```

## Load skills for a workspace

```js
const projectDir = '/workspace/project'; // or activeConversation.workspace?.working_dir

const skills = await api('/skills', {
  method: 'POST',
  body: JSON.stringify({
    load_public: true,
    load_user: true,
    load_project: true,
    load_org: false,
    project_dir: projectDir,
  }),
});

console.log(skills.skills.map((skill) => skill.name));
```

## Load sub-agents for a workspace

```js
const subAgents = await api('/sub-agents', {
  method: 'POST',
  body: JSON.stringify({
    load_user: true,
    load_project: true,
    load_builtin: true,
    project_dir: projectDir,
  }),
});

console.log(subAgents.agents.map((agent) => ({
  name: agent.name,
  builtin: agent.is_builtin,
  tools: agent.tools,
})));
```

## Inspect marketplace skills

```js
const marketplace = await api('/skills/marketplace');
console.log(marketplace.skills.map((skill) => ({
  name: skill.name,
  installed: skill.installed,
  source: skill.source,
})));
```

## Install or enable a skill

```js
const installedSkill = await api('/skills/install', {
  method: 'POST',
  body: JSON.stringify({
    source: 'github:OpenHands/extensions/skills/github',
    ref: 'main',
  }),
});

await api(`/skills/installed/${encodeURIComponent(installedSkill.name)}`, {
  method: 'PATCH',
  body: JSON.stringify({ enabled: true }),
});
```

## Inspect, install, and load a plugin

```js
const plugins = await api('/plugins', {
  method: 'POST',
  body: JSON.stringify({
    load_user: true,
    load_project: true,
    project_dir: projectDir,
  }),
});
console.log(plugins.plugins.map((plugin) => plugin.name));

const marketplacePlugins = await api('/plugins/marketplace');
console.log(marketplacePlugins.plugins.map((plugin) => ({
  name: plugin.name,
  installed: plugin.installed,
  source: plugin.source,
  repoPath: plugin.repo_path,
})));

const installedPlugin = await api('/plugins/install', {
  method: 'POST',
  body: JSON.stringify({
    source: 'github:OpenHands/extensions/plugins/city-weather',
    ref: 'main',
    force: false,
  }),
});

await api(`/conversations/${conversationId}/load_plugin`, {
  method: 'POST',
  body: JSON.stringify({ plugin_ref: installedPlugin.name }),
});
```

## List and activate saved profiles

```js
const profiles = await api('/profiles');
console.log(profiles.active_profile, profiles.profiles);

await api(`/profiles/${encodeURIComponent('gpt-5')}/activate`, {
  method: 'POST',
});
```

## List, materialize, and activate agent profiles

Agent profiles are separate from LLM profiles. Activation uses the profile's stable `id` and only updates the active pointer.

```js
const agentProfiles = await api('/agent-profiles');
console.log(agentProfiles.active_agent_profile_id, agentProfiles.profiles);

const profileName = agentProfiles.profiles[0]?.name;
if (profileName) {
  const diagnostics = await api(
    `/agent-profiles/${encodeURIComponent(profileName)}/materialize`,
    { method: 'POST' },
  );
  console.log(diagnostics.valid, diagnostics);
}

const profileId = agentProfiles.profiles[0]?.id;
if (profileId) {
  await api(`/agent-profiles/${encodeURIComponent(profileId)}/activate`, {
    method: 'POST',
  });
}
```

## Test an MCP server config before saving it

```js
const probe = await api('/mcp/test', {
  method: 'POST',
  body: JSON.stringify({
    name: 'github',
    server: {
      type: 'stdio',
      command: 'npx',
      args: ['-y', '@modelcontextprotocol/server-github'],
    },
    timeout: 15,
  }),
});

if (!probe.ok) {
  console.warn(`MCP validation failed: ${probe.error_kind} ${probe.error}`);
} else {
  console.log('MCP tools', probe.tools);
}
```

## Run MCP OAuth setup

Use this when an MCP config uses OAuth. The callback URL should be the final localhost redirect URL from the OAuth browser flow.

```js
const started = await api('/mcp/oauth/start', {
  method: 'POST',
  body: JSON.stringify({
    name: 'oauth-server',
    server: {
      type: 'http',
      url: 'https://mcp.example.com/mcp',
      auth: { strategy: 'oauth2' },
    },
    timeout: 15,
  }),
});

if (started.ok) {
  window.open(started.authorization_url, '_blank', 'noopener,noreferrer');
}

const status = await api(`/mcp/oauth/status/${encodeURIComponent(started.job_id)}`);
console.log(status.status, status.ok);

await api(`/mcp/oauth/callback/${encodeURIComponent(started.job_id)}`, {
  method: 'POST',
  body: JSON.stringify({
    callback_url: 'http://localhost:12345/callback?code=...',
  }),
});
```


## Search conversations

```js
const page = await api('/conversations/search?limit=50');
console.log(page.items, page.next_page_id);
```

## Get one conversation

```js
const conversation = await api(`/conversations/${conversationId}`);
console.log(conversation.execution_status);
```

If the UI truly needs the full inlined skill payload, opt in explicitly:

```js
const conversationWithSkills = await api(
  `/conversations/${conversationId}?include_skills=true`,
);
console.log(conversationWithSkills.agent.agent_context.skills);
```


## Ask the agent a side question without mutating history

```js
const answer = await api(`/conversations/${conversationId}/ask_agent`, {
  method: 'POST',
  body: JSON.stringify({
    question: 'Summarize the current plan in one sentence.',
  }),
});

console.log(answer.response);
```

## Read the agent's final response summary

```js
const finalResponse = await api(
  `/conversations/${conversationId}/agent_final_response`,
);
console.log(finalResponse.response);
```

## Switch a conversation to a saved profile

```js
await api(`/conversations/${conversationId}/switch_profile`, {
  method: 'POST',
  body: JSON.stringify({ profile_name: 'gpt-5' }),
});
```

## Start, stop, and resume a goal loop

```js
await api(`/conversations/${conversationId}/goal`, {
  method: 'POST',
  body: JSON.stringify({
    objective: 'Finish the remaining implementation and verify it.',
    max_iterations: 5,
  }),
});

await api(`/conversations/${conversationId}/goal/stop`, { method: 'POST' });
await api(`/conversations/${conversationId}/goal/resume`, { method: 'POST' });
```

## Navigate a conversation branch to an event

```js
const updatedConversation = await api(`/conversations/${conversationId}/navigate`, {
  method: 'POST',
  body: JSON.stringify({ event_id: targetEventId }),
});

console.log(updatedConversation.leaf_event_id);
```

## Switch an ACP conversation model

```js
await api(`/conversations/${conversationId}/switch_acp_model`, {
  method: 'POST',
  body: JSON.stringify({ model: 'gpt-5.5' }),
});
```

## Create a conversation for a trusted internal UI

Use this only when the browser UI is inside the same trust boundary as the server and can legitimately handle provider credentials, such as a local machine or authenticated internal workspace.

```js
const response = await api('/conversations', {
  method: 'POST',
  body: JSON.stringify({
    agent: {
      llm: {
        model: 'anthropic/claude-sonnet-4-5-20250929',
        api_key: '<trusted-only-api-key>',
      },
      tools: [
        { name: 'terminal' },
        { name: 'file_editor' },
        { name: 'task_tracker' },
      ],
    },
    workspace: {
      kind: 'LocalWorkspace',
      working_dir: 'workspace/project',
    },
    initial_message: {
      role: 'user',
      content: [{ type: 'text', text: 'Help me build a UI' }],
      run: true,
    },
  }),
});

console.log('conversation id', response.id);
```

## Safer settings round-trip pattern

If the server is already managing settings, fetch encrypted settings for authenticated trusted clients, then send them back with `secrets_encrypted: true`.

```js
async function getEncryptedSettings() {
  const headers = new Headers();
  headers.set('X-Session-API-Key', sessionApiKey);
  headers.set('X-Expose-Secrets', 'encrypted');

  const response = await fetch(`${apiBaseUrl}/settings`, { headers });
  if (!response.ok) throw new Error(await response.text());
  return response.json();
}
```

```js
const settings = await getEncryptedSettings();

const response = await api('/conversations', {
  method: 'POST',
  body: JSON.stringify({
    agent: settings.agent_settings,
    workspace: {
      kind: 'LocalWorkspace',
      working_dir: 'workspace/project',
    },
    secrets_encrypted: true,
    initial_message: {
      role: 'user',
      content: [{ type: 'text', text: 'Start a new task' }],
      run: true,
    },
  }),
});
```

## Patch one MCP server in settings

Use these endpoints when a settings UI needs to update one MCP server without replacing the whole `mcp_config` object.

```js
await api(`/settings/mcp/${encodeURIComponent('github')}`, {
  method: 'POST',
  body: JSON.stringify({
    transport: 'stdio',
    command: 'npx',
    args: ['-y', '@modelcontextprotocol/server-github'],
  }),
});

await api(`/settings/mcp/${encodeURIComponent('github')}`, {
  method: 'PATCH',
  body: JSON.stringify({
    env: { GITHUB_TOKEN: '<trusted-only-token>' },
  }),
});

await api(`/settings/mcp/${encodeURIComponent('github')}`, {
  method: 'DELETE',
});
```

## Send a user message

```js
await api(`/conversations/${conversationId}/events`, {
  method: 'POST',
  body: JSON.stringify({
    role: 'user',
    content: [{ type: 'text', text: 'Continue' }],
    run: true,
  }),
});
```

## Interrupt immediately

```js
await api(`/conversations/${conversationId}/interrupt`, { method: 'POST' });
```

Use `/interrupt` when the UI needs to cancel an in-flight run right away. `/pause` is cooperative and takes effect once the current work yields control.

## Pause and resume

```js
await api(`/conversations/${conversationId}/pause`, { method: 'POST' });
await api(`/conversations/${conversationId}/run`, { method: 'POST' });
```

There is no `/resume` endpoint. Resume uses `/run`.

## Respond to confirmation requests

```js
await api(`/conversations/${conversationId}/events/respond_to_confirmation`, {
  method: 'POST',
  body: JSON.stringify({
    accept: true,
    reason: 'Approved from UI',
  }),
});
```

## Upload a file

```js
async function uploadFile(file, absolutePath) {
  const form = new FormData();
  form.append('file', file);

  return api(`/file/upload?path=${encodeURIComponent(absolutePath)}`, {
    method: 'POST',
    body: form,
    headers: sessionApiKey ? { 'X-Session-API-Key': sessionApiKey } : {},
  });
}
```

Do not manually set `Content-Type` for `FormData`.

## Check VS Code launcher availability

```js
const vscodeStatus = await api('/vscode/status');
if (vscodeStatus.enabled && vscodeStatus.running) {
  const { url } = await api('/vscode/url');
  console.log('open vscode at', url);
}
```

## Execute a bash command through REST

```js
const output = await api('/bash/execute_bash_command', {
  method: 'POST',
  body: JSON.stringify({
    command: 'ls -la',
    cwd: '/workspace/project',
    timeout: 300,
  }),
});
```

## Browser bash-events socket

This example also assumes a trusted local or authenticated environment.

```js
function connectBashEvents() {
  const url = `${wsBaseUrl}/sockets/bash-events`;
  const ws = new WebSocket(url);

  ws.addEventListener('open', () => {
    if (sessionApiKey) {
      ws.send(JSON.stringify({
        type: 'auth',
        session_api_key: sessionApiKey,
      }));
    }
  });

  ws.addEventListener('message', (event) => {
    const payload = JSON.parse(event.data);
    console.log('bash event', payload);
  });

  return ws;
}
```

## List commits and archive a workspace

```js
const commits = await api(
  `/git/commits?path=${encodeURIComponent('/workspace/project')}&limit=20`,
);
console.log(commits.commits, commits.has_more);

const changes = await api(
  `/git/commits/${encodeURIComponent(commits.commits[0].sha)}/changes?path=${encodeURIComponent('/workspace/project')}`,
);
console.log(changes);

const archiveResponse = await api(
  `/file/archive?path=${encodeURIComponent('/workspace/project')}&format=tar.gz`,
);
const archiveBlob = await archiveResponse.blob();
console.log(archiveBlob.size);
```

## Use OpenAI-compatible chat completions

```js
async function openAICompatibleChat(messages, existingConversationId = null) {
  const modelsResponse = await fetch(`${apiBaseUrl.replace(/\/api$/, '')}/v1/models`, {
    headers: { Authorization: `Bearer ${sessionApiKey}` },
  });
  if (!modelsResponse.ok) throw new Error(await modelsResponse.text());
  const models = await modelsResponse.json();
  const model = models.data[0]?.id;
  if (!model) throw new Error('No OpenAI-compatible models are configured.');

  const headers = new Headers({
    'Content-Type': 'application/json',
    Authorization: `Bearer ${sessionApiKey}`,
  });
  if (existingConversationId) {
    headers.set('X-OpenHands-ServerConversation-ID', existingConversationId);
  }

  const response = await fetch(`${apiBaseUrl.replace(/\/api$/, '')}/v1/chat/completions`, {
    method: 'POST',
    headers,
    body: JSON.stringify({
      model,
      messages,
      stream: false,
    }),
  });
  if (!response.ok) throw new Error(await response.text());

  return {
    conversationId: response.headers.get('X-OpenHands-ServerConversation-ID'),
    completion: await response.json(),
  };
}
```

## Start and poll OpenAI subscription sign-in

```js
const models = await api('/llm/subscription/openai/models');
console.log(models.models);

const signIn = await api('/llm/subscription/openai/device/start', {
  method: 'POST',
});

console.log(signIn.user_code, signIn.verification_uri);

const status = await api('/llm/subscription/openai/device/poll', {
  method: 'POST',
  body: JSON.stringify({ device_code: signIn.device_code }),
});

console.log(status.connected);
```
