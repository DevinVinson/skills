# OpenHands Agent Server UI Examples

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

## REST helper with session API key

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

## Connect to the conversation events socket

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

Use this only when the browser UI is inside the same trust boundary as the server and can legitimately handle provider credentials.

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
