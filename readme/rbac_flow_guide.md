[English](rbac_flow_guide.md) | [中文](rbac_flow_guide_cn.md)

# RBAC Flow Guide

## 1. Request chain from frontend to Dify for `api/chat-messages`

The following uses a user sending a message in the chat box as an example, explaining how the request travels from the frontend all the way to the Dify backend at `NEXT_PUBLIC_API_URL`.

### Step 1: Chat page triggers send

Entry point: `app/components/index.tsx`

- `handleSend(message, files)` assembles:
  - `inputs`
  - `query`
  - `conversation_id`
  - `files`
- Then calls `sendChatMessage(data, ...)`

In other words, the frontend business entry point that actually initiates the chat request is `handleSend` on the main chat page.

### Step 2: Frontend service wraps the chat request

Entry point: `service/index.ts`

- `sendChatMessage()` augments the frontend-assembled `body` with:
  - `response_mode: 'streaming'`
- Then calls: `ssePost('chat-messages', ...)`

This step consolidates chat requests in the service layer and specifies that this is an SSE streaming request.

### Step 3: Frontend HTTP layer sends the request to the local Next.js backend

Entry point: `service/base.ts`

Two key points:

1. `ssePost()` builds the URL as:

```ts
/api/chat-messages
```

That is, the frontend does not call Dify directly; it always hits this project's Next.js API route first.

2. `buildRequestHeaders()` reads the currently selected agent from local storage:

- Read: `getStoredSelectedAgentAppIdValue()`
- Write to header: `x-dify-app-id`

So each chat request, in addition to the login session in the cookie, also carries an agent identifier:

```http
x-dify-app-id: <currently selected appId>
```

This header is the key input for backend agent selection and RBAC checks.

### Step 4: Next.js API route handles `/api/chat-messages`

Entry point: `app/api/chat-messages/route.ts`

This route does the following:

1. Calls `getRequestContext(request)`
2. Reads from the request body:
   - `inputs`
   - `query`
   - `files`
   - `conversation_id`
   - `response_mode`
3. Calls:

```ts
client.createChatMessage(inputs, query, user, responseMode, conversationId, files)
```

The important part is that `client` here is not passed from the frontend; the backend resolves it on the fly in `getRequestContext()` based on the logged-in user and agent permissions.

### Step 5: Backend performs login validation, RBAC, and agent resolution

Entry point: `app/api/utils/common.ts`

`getRequestContext(request)` first calls:

```ts
getAuthorizedAgentConfig(request)
```

This is the core control point of the entire backend proxy chain.

It does the following:

1. `requireAuthenticatedUser(request)`
   - Checks the JWT in the cookie, or `Authorization: Bearer <token>`
2. Reads `x-dify-app-id` sent by the frontend
3. Selects which agent to use for this request from the logged-in user's allowed agent list
4. If the user requests an agent they are not permitted to use, throws 403 immediately
5. Finds the corresponding agent's Dify API key and creates a `ChatClient`

### Step 6: How JWT and RBAC config are resolved

Entry point: `utils/auth.ts`

The execution chain for `requireAuthenticatedUser(request)` is:

1. `getAuthTokenFromRequest(request)`
   - Checks `Authorization` header first
   - Falls back to the `auth_token` cookie
2. `verifyAuthToken(token)`
   - Verifies the JWT
3. `buildAuthenticatedUser(config, username)`
   - Reads `rbac.json`
   - Finds the role for this user
   - Resolves the list of agents visible to this user based on the role's `agents` config

RBAC config file: `rbac.json`

For example, the current config:

- `admin` role -> `agent-1`, `agent-2`
- `user` role -> `agent-1`

### Step 7: Backend uses server-only agent config to access Dify

Entry point: `config/server.ts`

This parses `NEXT_PUBLIC_AGENT_CONFIGS` from `.env` into server-side agent config, including:

- `id`
- `name`
- `appId`
- `apiKey`

It also reads:

```ts
export const API_URL = `${process.env.NEXT_PUBLIC_API_URL}`.trim()
```

Which is your Dify backend address, for example:

```dotenv
NEXT_PUBLIC_API_URL=http://127.0.0.1/v1
```

### Step 8: Where the actual connection to Dify is made

The Dify SDK client is created in: `app/api/utils/common.ts`

```ts
const createChatClient = (apiKey: string) => new ChatClient(apiKey, API_URL || undefined)
```

This line passes:

- The agent's `apiKey`
- `API_URL` read from `config/server.ts`

into `dify-client`'s `ChatClient`.

Then in `app/api/chat-messages/route.ts` it executes:

```ts
client.createChatMessage(...)
```

At this point, the request is formally forwarded from this project's backend to Dify's `NEXT_PUBLIC_API_URL`.

---

## 2. Where RBAC control is enforced

The core RBAC control points are in two files:

### A. `utils/auth.ts`

Responsible for:

- Verifying JWT
- Reading `rbac.json`
- Building `AuthenticatedUser` from `username -> role -> agents`

In other words, this answers the question:

> Which agents can this logged-in user theoretically use?

Key functions:

- `loadRbacConfig()`
- `buildAuthenticatedUser()`
- `resolveAllowedAgents()`
- `requireAuthenticatedUser()`

### B. `app/api/utils/common.ts`

Responsible for:

- Reading `x-dify-app-id` from the current HTTP request
- Comparing it against the logged-in user's allowed agent list
- Deciding whether this request is allowed to proceed

In other words, this answers another question:

> For this specific HTTP request, can the user actually use the agent they specified?

Key functions:

- `getAuthorizedAgentConfig()`
- `getRequestContext()`

---

## 3. Where it is determined whether a regular `user` has `agent2` permission after login

This check happens in two stages.

### Stage 1: Which agents the `user` account itself has

Location: `utils/auth.ts`

In `buildAuthenticatedUser(config, username)`:

1. Find the user in `rbac.json` by username
2. Get their role, e.g. `user`
3. Get the agent list for that role

With the current config:

```json
"user": {
  "agents": ["agent-1"]
}
```

So after a regular `user` logs in successfully, their `authenticatedUser.agents` will only include Agent 1, not Agent 2.

### Stage 2: Whether this request is forcibly targeting `agent2`

Location: `app/api/utils/common.ts`

The logic that actually blocks regular users from accessing Agent 2 is:

```ts
const requestedAppId = request.headers.get(AGENT_ID_HEADER_NAME)
const fallbackAgent = authenticatedUser.agents[0]
const selectedAgent = authenticatedUser.agents.find(agent => agent.appId === requestedAppId) || fallbackAgent

if (!selectedAgent) {
  throw new AuthError(403, 'No agents are assigned to this account.', 'rbac_no_agents')
}

if (requestedAppId && selectedAgent.appId !== requestedAppId) {
  throw new AuthError(403, 'You do not have permission to access this agent.', 'rbac_agent_forbidden')
}
```

What this means:

1. Read the agent appId requested by the frontend from the header
2. Look up this appId in the logged-in user's allowed agent list
3. If not found, but the request explicitly specified `requestedAppId`
4. Return immediately:

```ts
403 You do not have permission to access this agent.
```

So if a regular `user` forcibly sends Agent 2's `appId` from the frontend:

```http
x-dify-app-id: xxx
```

The backend blocks the request in `getAuthorizedAgentConfig()` and does not forward it to Dify.

---

## 4. One-sentence summary

- Frontend: `handleSend()` -> `service/index.ts` -> `service/base.ts` -> `/api/chat-messages`
- Backend: `app/api/chat-messages/route.ts` -> `getRequestContext()` -> `requireAuthenticatedUser()` -> `getAuthorizedAgentConfig()`
- After passing RBAC, the backend calls Dify with `ChatClient(apiKey, NEXT_PUBLIC_API_URL)`
- Whether a regular `user` can access Agent 2 is ultimately decided by the 403 check in `getAuthorizedAgentConfig()` in `app/api/utils/common.ts`
