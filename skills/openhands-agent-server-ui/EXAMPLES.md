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
