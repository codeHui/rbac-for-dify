[English](rbac_flow_guide.md) | [中文](rbac_flow_guide_cn.md)

# RBAC Flow Guide

## 1. `api/chat-messages` 請求從前端到 Dify 的執行鏈路

下面以使用者在聊天框送出一條消息為例，說明這個請求是怎麼從前端一路走到 Dify 後端 `NEXT_PUBLIC_API_URL` 的。

### Step 1: 前端聊天頁觸發發送

入口在：`app/components/index.tsx`

- `handleSend(message, files)` 會整理：
  - `inputs`
  - `query`
  - `conversation_id`
  - `files`
- 然後調用 `sendChatMessage(data, ...)`

也就是說，真正發起聊天請求的前端業務入口，是主聊天頁的 `handleSend`。

### Step 2: 前端 service 封裝聊天請求

入口在：`service/index.ts`

- `sendChatMessage()` 會把前端整理好的 `body` 再補上：
  - `response_mode: 'streaming'`
- 然後調用：`ssePost('chat-messages', ...)`

這一步的作用是把聊天請求統一收斂到 service 層，並指定這是一個 SSE streaming 請求。

### Step 3: 前端 HTTP 層把請求發到本地 Next.js 後端

入口在：`service/base.ts`

關鍵點有兩個：

1. `ssePost()` 會把 URL 拼成：

```ts
/api/chat-messages
```

也就是說，前端不直接訪問 Dify，而是永遠先打自己這個項目的 Next.js API route。

1. `buildRequestHeaders()` 會從本地儲存中拿當前選中的 agent：

- 讀取：`getStoredSelectedAgentAppIdValue()`
- 寫入 header：`x-dify-app-id`

所以一次聊天請求，除了 cookie 中的登入態以外，還會額外帶一個 agent 標識：

```http
x-dify-app-id: <當前選中的 appId>
```

這個 header 是後端做 agent 選擇和 RBAC 判斷的關鍵輸入。

### Step 4: Next.js API route 接住 `/api/chat-messages`

入口在：`app/api/chat-messages/route.ts`

這個 route 做的事情很簡單：

1. 調用 `getRequestContext(request)`
2. 從 request body 取出：
   - `inputs`
   - `query`
   - `files`
   - `conversation_id`
   - `response_mode`
3. 調用：

```ts
client.createChatMessage(inputs, query, user, responseMode, conversationId, files)
```

其中真正重要的是這裡的 `client` 並不是前端傳來的，而是後端在 `getRequestContext()` 裡根據登入使用者和 agent 權限現場解析出來的。

### Step 5: 後端統一做登入校驗、RBAC、agent 解析

入口在：`app/api/utils/common.ts`

`getRequestContext(request)` 會先調用：

```ts
getAuthorizedAgentConfig(request)
```

這裡是整個後端代理鏈路的核心控制點。

它做了幾件事：

1. `requireAuthenticatedUser(request)`
   - 檢查 cookie 裡的 JWT，或 `Authorization: Bearer <token>`
2. 讀取前端送來的 `x-dify-app-id`
3. 從登入使用者可用的 agent 列表中，選出這次請求要用哪個 agent
4. 如果使用者請求了一個自己沒有權限的 agent，直接丟出 403
5. 找到對應 agent 的 Dify API key，建立 `ChatClient`

### Step 6: JWT 與 RBAC 配置是怎麼解析的

入口在：`utils/auth.ts`

`requireAuthenticatedUser(request)` 的執行鏈如下：

1. `getAuthTokenFromRequest(request)`
   - 先看 `Authorization` header
   - 沒有再看 `auth_token` cookie
2. `verifyAuthToken(token)`
   - 驗證 JWT
3. `buildAuthenticatedUser(config, username)`
   - 讀取 `rbac.json`
   - 找出這個使用者對應的 role
   - 根據 role 裡配置的 `agents` 解析出此使用者可見的 agent 清單

RBAC 配置文件在：`rbac.json`

例如目前配置：

- `admin` role -> `agent-1`, `agent-2`
- `user` role -> `agent-1`

### Step 7: 後端用 server-only 的 agent 配置訪問 Dify

入口在：`config/server.ts`

這裡會把 `.env` 裡的 `NEXT_PUBLIC_AGENT_CONFIGS` 解析成 server 端使用的 agent 配置，包含：

- `id`
- `name`
- `appId`
- `apiKey`

並且還會讀出：

```ts
export const API_URL = `${process.env.NEXT_PUBLIC_API_URL}`.trim()
```

也就是你的 Dify 後端地址，例如：

```dotenv
NEXT_PUBLIC_API_URL=http://127.0.0.1/v1
```

### Step 8: 真正連到 Dify 的位置

真正建立 Dify SDK client 的位置在：`app/api/utils/common.ts`

```ts
const createChatClient = (apiKey: string) => new ChatClient(apiKey, API_URL || undefined)
```

這一行把：

- agent 對應的 `apiKey`
- `config/server.ts` 中讀到的 `API_URL`

一起傳進 `dify-client` 的 `ChatClient`。

然後在 `app/api/chat-messages/route.ts` 裡執行：

```ts
client.createChatMessage(...)
```

到這一步，請求就正式從本項目後端轉發到 Dify 的 `NEXT_PUBLIC_API_URL` 了。

---

## 2. RBAC Controller 是在哪裡做的

最核心的 RBAC 控制點在兩個文件：

### A. `utils/auth.ts`

這裡負責：

- 驗證 JWT
- 讀取 `rbac.json`
- 根據 `username -> role -> agents` 建立 `AuthenticatedUser`

也就是說，這裡負責回答一個問題：

> 這個登入者理論上能用哪些 agent？

關鍵函數：

- `loadRbacConfig()`
- `buildAuthenticatedUser()`
- `resolveAllowedAgents()`
- `requireAuthenticatedUser()`

### B. `app/api/utils/common.ts`

這裡負責：

- 根據當前 HTTP request 裡的 `x-dify-app-id`
- 對照登入者的可用 agent 列表
- 決定本次請求是否允許繼續

也就是說，這裡負責回答另一個問題：

> 這一次 HTTP 請求，能不能實際用戶指定的這個 agent？

關鍵函數：

- `getAuthorizedAgentConfig()`
- `getRequestContext()`

---

## 3. 普通 `user` 登錄後，是否有 `agent2` 權限，是在哪裡判斷的

這個判斷分成兩段。

### 第一段：`user` 這個賬號本身有哪些 agent

位置：`utils/auth.ts`

在 `buildAuthenticatedUser(config, username)` 裡：

1. 先根據使用者名稱找到 `rbac.json` 中的 user
2. 取出它的 role，例如：`user`
3. 再取出 role 對應的 agent 列表

對現在的配置來說：

```json
"user": {
  "agents": ["agent-1"]
}
```

所以普通 `user` 登錄成功後，它的 `authenticatedUser.agents` 只會有 Agent 1，不會有 Agent 2。

### 第二段：這次請求是不是在硬切 `agent2`

位置：`app/api/utils/common.ts`

真正攔截普通使用者訪問 Agent 2 的，是這段邏輯：

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

它的意思是：

1. 從 header 讀出前端請求的 agent appId
2. 在登入使用者可用 agent 列表中查找這個 appId
3. 如果找不到，但請求裡又明確指定了 `requestedAppId`
4. 就直接回：

```ts
403 You do not have permission to access this agent.
```

所以，普通 `user` 如果前端硬傳 Agent 2 的 `appId`：

```http
x-dify-app-id: xxx
```

後端會在 `getAuthorizedAgentConfig()` 這裡直接擋下，不會把請求放行到 Dify。

---

## 4. 一句話總結

- 前端 `handleSend()` -> `service/index.ts` -> `service/base.ts` -> `/api/chat-messages`
- 後端 `app/api/chat-messages/route.ts` -> `getRequestContext()` -> `requireAuthenticatedUser()` -> `getAuthorizedAgentConfig()`
- 通過 RBAC 後，後端用 `ChatClient(apiKey, NEXT_PUBLIC_API_URL)` 去請求 Dify
- 普通 `user` 是否能訪問 Agent 2，最終是在 `app/api/utils/common.ts` 的 `getAuthorizedAgentConfig()` 中做 403 判斷
