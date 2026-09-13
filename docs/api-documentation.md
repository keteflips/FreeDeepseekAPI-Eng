# FreeDeepseekAPI — Complete Documentation

## Overview

This project reverse-engineers the **DeepSeek Web chat API** (`chat.deepseek.com`) and exposes it as OpenAI/Anthropic-compatible local API endpoints. Compatible clients (Claude Code, OpenAI SDK/Responses-style clients, Open WebUI, LiteLLM, Hermes, custom scripts, etc.) can use the DeepSeek web model as if it were a paid API — including tool calling, streaming, reasoning output and multi-session support.

**Server:** local Node.js HTTP server, default `http://127.0.0.1:9655`
**Auth:** a saved DeepSeek Web browser session (`deepseek-auth.json`)
**Model aliases:** `deepseek-chat`, `deepseek-reasoner`, `deepseek-expert`, … (see [§4.3](#43-list-models))

The authoritative sources for current behavior are `server.js`, `README.md` and `.env.example`. This document describes that behavior.

---

## 1. Architecture

```
┌──────────────┐     POST /v1/chat/completions     ┌──────────────────┐
│              │ ──────────────────────────────►    │                  │
│   Client     │    {messages, tools, model,        │  DeepSeek Proxy  │
│  (SDK/agent) │     stream}                        │  (port 9655)     │
│              │ ◄──────────────────────────────    │                  │
│              │    {choices[].message.content      │  Node.js HTTP    │
└──────────────┘     or tool_calls}                 │  Server          │
                                                     │                  │
                                                      └────────┬─────────┘
                                                               │
                                     ┌─────────────────────────┼──────────────┐
                                     │                         │              │
                                     ▼                         ▼              ▼
                           ┌──────────────────┐    ┌──────────────────┐
                           │  PoW Challenge   │    │  Chat Completion │
                           │  /api/v0/chat/   │    │  /api/v0/chat/   │
                           │  create_pow_     │    │  completion      │
                           │  challenge       │    │                  │
                           └──────────────────┘    └──────────────────┘
                                                          │
                                                          ▼
                                                ┌──────────────────┐
                                                │  DeepSeek Web    │
                                                │  chat.deepseek   │
                                                │  .com            │
                                                │  (free web chat) │
                                                └──────────────────┘
```

The proxy pools one or more saved DeepSeek accounts, solves a proof-of-work challenge before every completion, and reuses a DeepSeek `chat_session_id` per agent/session until it is reset.

---

## 2. DeepSeek Web API Endpoints (Reverse-Engineered)

These are the internal endpoints the proxy calls. **Not official** — obtained by reverse-engineering the DeepSeek web app's network traffic. They can change without warning.

### 2.1 Create PoW Challenge

```
POST https://chat.deepseek.com/api/v0/chat/create_pow_challenge

Headers:
  Authorization: Bearer <token>
  x-hif-dliq: <hif_dliq>        (when present in the saved auth)
  x-hif-leim: <hif_leim>        (when present in the saved auth)
  Cookie: ds_session_id=<id>; smidV2=<smidV2>; ...
  Content-Type: application/json
  Origin: https://chat.deepseek.com
  Referer: https://chat.deepseek.com/

Body:
{
  "target_path": "/api/v0/chat/completion"
}

Response:
{
  "data": {
    "biz_data": {
      "challenge": {
        "algorithm": "...",
        "challenge": "...",
        "salt": "...",
        "signature": "...",
        "difficulty": <int>,
        "expire_at": <timestamp>
      }
    }
  }
}
```

### 2.2 Create Chat Session

```
POST https://chat.deepseek.com/api/v0/chat_session/create

Headers: Same as above
Body: {}

Response:
{
  "data": {
    "biz_data": {
      "chat_session": { "id": "uuid-session-id" }
      // some responses put the id directly at data.biz_data.id
    }
  }
}
```

The proxy reads `data.biz_data.chat_session.id` and falls back to `data.biz_data.id`.

### 2.3 Chat Completion (Streaming SSE)

```
POST https://chat.deepseek.com/api/v0/chat/completion

Headers:
  ...same as above...
  X-DS-PoW-Response: <base64 encoded PoW answer>

Body:
{
  "chat_session_id": "uuid",          ← from session/create
  "parent_message_id": <int|null>,    ← threading; null = first message
  "model_type": "default",            ← default | expert | vision
  "prompt": "<user message text>",
  "ref_file_ids": [],
  "thinking_enabled": false,
  "search_enabled": false,
  "action": null,
  "preempt": false
}

Response: Server-Sent Events (SSE). Each `data:` line is a JSON object whose
shape depends on the streamed path. The parser handles, among others:

  {"p":"response/content","v":"more text..."}
  {"p":"response/fragments","v":[{...fragment...}]}
  {"p":"response/fragments/-1/content","v":"appended chars"}
  {"p":"response","v":{...patch operations...}}
  {"p":"response/finish_reason","v":"stop"}
  {"p":"response/status","v":"FINISHED"}
  {"response_message_id":<int>}
  {"finish_reason":"..."}
```

**Key points:**

- `parent_message_id` is an **integer**, not a string — it tracks the conversation tree.
- On the first call, `parent_message_id` is `null`.
- Text arrives as **fragments**. Fragment types include `RESPONSE`/`SEARCH` (assistant output) and `THINK`/`REASONING` (reasoning text). `THINK` fragments become `reasoning_content`; `SEARCH` fragments count as assistant output.
- The parser also handles fragment patches, `response/content` appends and `response/fragments/-1/content` appends.
- `response_message_id` / `response.message_id` becomes the next `parent_message_id`; `messageCount` is incremented.
- A model-error event is surfaced through the error path.

### 2.4 Proof-of-Work (SHA3 Wasm)

Each completion requires solving a PoW challenge using a WASM module. `lib/pow.js` caches the compiled module and enforces a fetch timeout.

```
WASM URL: https://fe-static.deepseek.com/chat/static/sha3_wasm_bg.<hash>.wasm
          (stored as `wasmUrl` in deepseek-auth.json)

Function: wasm_solve(sp, cBytes_ptr, cBytes_len, pBytes_ptr, pBytes_len, difficulty)
Input: challenge bytes + prefix (salt + "_" + expire_at + "_" + challenge)
Output: answer (integer via Float64 view at stack pointer + 8)
```

Steps:

1. Fetch the WASM binary.
2. Instantiate with `{ wbg: {} }` imports.
3. Encode challenge bytes and prefix bytes.
4. Allocate memory and copy the data.
5. Call `wasm_solve()` — returns the answer on success or `0` on failure.
6. Pack `{algorithm, challenge, salt, answer, signature, target_path}` into base64 and send it as `X-DS-PoW-Response`.

---

## 3. Authentication and CORS

- `/`, `/health` and `/readyz` are public probes.
- Every other route requires `Authorization: Bearer <PROXY_API_KEY>` **when a key is configured**.
- The key is read from `PROXY_API_KEY`, or from a file given by `PROXY_API_KEY_FILE`.
- `REQUIRE_PROXY_API_KEY=1` fails startup when neither source is set (enabled by the Containerfile).
- Without a key, non-health endpoints are unauthenticated — do not expose such an instance to the network.
- `PROXY_API_KEY`-protected `/health` additionally returns private status (accounts, agents, in-flight).
- CORS: loopback origins are allowed by default. Add exact remote origins with `PROXY_CORS_ORIGINS=https://ui.example.com,http://192.168.1.20:3000`. A disallowed browser origin gets `403 cors_error`.

---

## 4. Proxy Endpoints

### 4.1 Health Check (liveness, public)

```
GET /
GET /health

Response (public):
{
  "status": "ok",
  "service": "FreeDeepseekAPI",
  "watermark": "..."
}

Response (when authorized or no key configured) adds:
{
  "models": ["deepseek-chat", "deepseek-reasoner", ...],
  "unsupported_models": ["deepseek-expert-search", "deepseek-vision"],
  "agents": <int>,                 // active local agent sessions
  "in_flight": <int>,              // concurrent completions
  "accounts": [
    { "id": "...", "ready": true, "cooldown": false,
      "cooldown_remaining_sec": 0, "failures": 0, "last_used_at": <ts|null> }
  ],
  "config_ready": true,
  "session_reuse": {
    "strategy": "sticky per x-agent-session/user",
    "ttl_minutes": 120,
    "max_messages": 100,
    "reset_all": "POST /reset-session?agent=all"
  }
}
```

Account entries never include auth file paths or file names.

### 4.2 Readiness Probe (public)

```
GET /readyz

200 when at least one account can serve right now, otherwise 503.

{
  "ready": true,
  "ready_accounts": 2,
  "total_accounts": 3
}
```

`/health` is liveness (process is up); `/readyz` is readiness (an uncooldowned account exists).

### 4.3 List Models

```
GET /v1/models

{
  "object": "list",
  "data": [
    {
      "id": "deepseek-chat",
      "object": "model",
      "created": 1700000000,
      "owned_by": "deepseek-web",
      "real_model": "DeepSeek-V4-Flash non-thinking (DeepSeek Web “Fast” / default)",
      "capabilities": { "reasoning": false, "web_search": false, "files": true }
    }
  ]
}
```

Only aliases with `supported: true` are listed. `supported: false` aliases are still described by `/v1/model-capabilities`.

### 4.4 Model Capabilities

```
GET /v1/model-capabilities
GET /api/model-capabilities

{
  "object": "model_capabilities",
  "watermark": "...",
  "data": {
    "deepseek-chat": {
      "id": "deepseek-chat",
      "real_model": "...",
      "model_type": "default",
      "thinking_enabled": false,
      "search_enabled": false,
      "capabilities": { ... },
      "supported": true
    },
    "deepseek-vision": { "supported": false, "unavailable_reason": "..." }
  }
}
```

### 4.5 Chat Completions — Primary API

```
POST /v1/chat/completions

Headers:
  Content-Type: application/json
  Authorization: Bearer <PROXY_API_KEY>   ← required only when a key is configured

Body (OpenAI-compatible):
{
  "model": "deepseek-chat",
  "messages": [
    {"role": "system", "content": "..."},
    {"role": "user", "content": "..."}
  ],
  "tools": [                                 ← optional
    {"type": "function", "function": {"name": "terminal", "description": "...", "parameters": {...}}}
  ],
  "stream": true|false,
  "user": "agent-id",                        ← optional session key
  "session": "agent-id"                      ← optional session key
}

Response (non-stream, stream=false):
{
  "id": "ds-<timestamp>",
  "object": "chat.completion",
  "created": <unix_ts>,
  "model": "deepseek-chat",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "..." | null,
        "reasoning_content": "..." | undefined,
        "tool_calls": [...] | undefined
      },
      "finish_reason": "stop" | "tool_calls" | "length"
    }
  ],
  "usage": {
    "prompt_tokens": <int>,
    "completion_tokens": <int>,
    "total_tokens": <int>,
    "completion_tokens_details": { "reasoning_tokens": <int> }
  }
}

Response (stream, stream=true):
  data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"reasoning_content":"..."}}]}
  data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{"content":"..."}}]}
  data: {"id":"...","object":"chat.completion.chunk","choices":[{"delta":{},"finish_reason":"stop"}]}
  data: [DONE]
```

`prompt_tokens` counts the **client's original input**, not the proxy-expanded prompt (system prompt + injected tool definitions + recovery history). `reasoning_tokens` is an estimate from the extracted reasoning text.

### 4.6 Anthropic Messages Shim — Claude Code / Anthropic SDK

```
POST /v1/messages

Request:
{
  "model": "deepseek-chat",
  "max_tokens": 1024,
  "system": "optional system prompt",
  "messages": [{"role":"user","content":"Hello"}],
  "tools": [
    {
      "name": "get_time",
      "description": "Get current time",
      "input_schema": {"type":"object","properties":{"timezone":{"type":"string"}}}
    }
  ],
  "stream": true|false,
  "metadata": {"user_id":"agent-session-id"}
}

Non-stream response uses Anthropic content blocks:
{
  "type": "message",
  "role": "assistant",
  "content": [{"type":"text","text":"..."}] | [{"type":"tool_use","id":"call_...","name":"...","input":{...}}],
  "stop_reason": "end_turn" | "tool_use",
  "usage": {"input_tokens": <int>, "output_tokens": <int>}
}

Streaming response emits Anthropic-style SSE events:
  event: message_start
  event: content_block_start
  event: content_block_delta
  event: content_block_stop
  event: message_delta
  event: message_stop
```

When reasoning is present, the stream emits it as a text block prefixed with `[reasoning] ... [/reasoning]`.

Claude Code direct backend example:

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:9655"
export ANTHROPIC_AUTH_TOKEN="dummy-key"
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
claude --model deepseek-chat
```

### 4.7 OpenAI Responses API Shim

```
POST /v1/responses

Request:
{
  "model": "deepseek-chat",
  "input": "Hello" | [{"role":"user","content":"Hello"}],
  "instructions": "optional system prompt",
  "tools": [{"type":"function","name":"get_time","parameters":{...}}],
  "stream": true|false
}

Response:
{
  "id": "resp_<timestamp>",
  "object": "response",
  "status": "completed",
  "model": "deepseek-chat",
  "output": [
    { "type": "reasoning", "summary": [{"type":"summary_text","text":"..."}] },
    { "type": "function_call", "id": "fc_...", "call_id": "...", "name": "...", "arguments": "..." },
    { "type": "message", "role": "assistant", "content": [{"type":"output_text","text":"..."}] }
  ],
  "output_text": "...",
  "usage": {
    "input_tokens": <int>,
    "output_tokens": <int>,
    "output_tokens_details": {"reasoning_tokens": <int>}
  }
}

Streaming emits Responses-style events such as:
  event: response.created
  event: response.output_item.added
  event: response.output_item.done
  event: response.content_part.added
  event: response.output_text.delta
  event: response.output_text.done
  event: response.completed
```

### 4.8 Tool Calling Compatibility

The proxy accepts these tool schemas:

- OpenAI Chat Completions: `tools: [{type:"function", function:{name, description, parameters}}]`
- Anthropic Messages: `tools: [{name, description, input_schema}]`
- Responses API: `tools: [{type:"function", name, description, parameters}]`

DeepSeek Web has no native OpenAI tool calls, so the proxy prompt-emulates them. The parser accepts:

- strict JSON: `{"tool_call":{"name":"tool","arguments":{...}}}`
- legacy format: `TOOL_CALL: tool\narguments: {...}`
- fenced JSON blocks with an explicit `tool_call`, `tool_calls`, or `function_call` envelope
- XML-ish `<tool_call>{...}</tool_call>` wrappers
- DeepSeek DSML (`<｜DSML｜tool_calls>...`) and the doubled-bar Web variant (`<｜｜DSML｜｜ Tool Calls>`)

Tool markup is size-bounded and the JSON extractor uses a balanced-brace scan, so incomplete/malformed markup is refused rather than executed.

### 4.9 List Active Sessions

```
GET /v1/sessions

{
  "agents": [
    {
      "agent": "my-agent",
      "session_id": "uuid",
      "message_count": 42,
      "account": "account-id",
      "history_size": 5,
      "age_min": 23
    }
  ],
  "total": 1
}
```

### 4.10 Reset Session

```
POST /reset-session?agent=<agent-id>
POST /reset-session?agent=all

Response (single):
{
  "status": "session_reset",
  "agent": "my-agent",
  "history_preserved": 5,
  "history": "user msg 1 | user msg 2 | ..."
}

Response (all):
{
  "status": "all_sessions_cleared",
  "count": 3
}

404 when the agent has no local session.
```

Sending the literal user message `/new` resets that agent's session and history inline (returns a normal assistant message confirming the reset).

---

## 5. Multi-Agent Session Isolation

### 5.1 How Sessions Are Assigned

The agent id is resolved in this order:

1. `x-agent-session` request header;
2. `session` field in the JSON body;
3. `user` field in the JSON body;
4. otherwise, loopback callers become `dev-agent` and remote callers are keyed by their socket IP.

| Caller | Session key | Example |
|---|---|---|
| Loopback (`127.0.0.1`, `::1`, `::ffff:127.0.0.1`) with no key | `dev-agent` | local scripts |
| Remote IP with no key | remote IP | per-IP isolation |
| Any caller with `x-agent-session` / `session` / `user` | that value | named agents |

**Effect:** each agent gets its own isolated DeepSeek web session. No context leakage between agents.

### 5.2 Configuring Remote Agents

Remote agents should set `x-agent-session` (or `session` / `user`) for named sessions:

```yaml
model:
  base_url: http://127.0.0.1:9655/v1
  model: deepseek-chat
```

### 5.3 Session Data Structure

```javascript
{
  id: "uuid",                    // DeepSeek web session ID (null until created)
  parentMessageId: <int|null>,   // Last message ID for threading
  createdAt: <timestamp|null>,   // Session creation time
  messageCount: 0-100,           // Messages in this session
  accountId: "account-id",       // Sticky account that owns this remote session
  lastActivityAt: <timestamp>,   // Used for idle eviction
  history: [                     // Last 15 exchanges for context recovery
    { user: "...", assistant: "..." }
  ]
}
```

---

## 6. Account Pool

Multiple saved accounts can be pooled:

- `DEEPSEEK_AUTH_PATH` — a single file or a comma-separated list of files;
- `DEEPSEEK_AUTH_DIR` — a directory of `*.json` files (one account per file).

Behavior:

- a new agent/session gets an available account round-robin;
- the chosen account is pinned to the session (`sticky`);
- on `401`/`403`/`429` the account enters cooldown (`DEEPSEEK_ACCOUNT_COOLDOWN_MS`, default 10 minutes; a `Retry-After` header extends it);
- if a session's sticky account is unavailable, the remote session is reset so it is never reused under a different account;
- when every account is cooling down, the request returns `429 rate_limit` with `Retry-After`;
- when no valid account exists, the request returns `503 no_auth`.

---

## 7. Tool Calling Implementation

DeepSeek Web does not natively support function/tool calling, so the proxy implements it via **text injection + parsing**.

### 7.1 Flow

1. **Injection:** tool definitions are converted to text and appended to the system prompt:

```
--- TOOL REQUEST SYSTEM ---
You are an AI that ONLY REASONS and REQUESTS tool executions. You do NOT run any commands yourself.
When you need data from the local server, REQUEST exactly one tool call. Prefer strict JSON:
{"tool_call":{"name":"<function_name>","arguments":{...}}}

Legacy format is also accepted: TOOL_CALL: <function_name>
arguments: <JSON arguments>

Your response will be sent to the local gateway, which executes the command and sends the output back in the next message.

RULES:
1. You ONLY output the tool request — you never run anything yourself
2. Do NOT simulate, guess, or fabricate command output — wait for the actual result
3. The tool runs on <host> (<ip>), the local server — NOT on DeepSeek
4. After the tool executes, the result will be sent to you as a new user/tool message
5. Never add explanation before or after the tool request when requesting a tool
6. Keep arguments compact. Do not include large file contents unless the tool schema requires it.

Available functions:

## terminal
Execute shell commands
Parameters: {...}

--- END TOOL REQUEST SYSTEM ---
REMEMBER: Request tools only with strict JSON or TOOL_CALL legacy format. Never simulate results.
```

Large tool schemas (annotations like `description`/`examples`) are compacted before injection to stay within the prompt budget.

2. **Generation:** the model responds with a tool request when it wants to use a tool.
3. **Parsing:** the proxy detects the markup and extracts the tool name and arguments.
4. **JSON extraction:** a balanced-brace parser handles nested braces and escaped strings.
5. **Conversion:** the parsed call becomes OpenAI `tool_calls` with `finish_reason: "tool_calls"`.
6. **Execution:** the client executes the tool and sends the result back as a `tool` message.

### 7.2 Accepted Formats

```
{"tool_call":{"name":"terminal","arguments":{"command":"hostname -I"}}}
```

```
TOOL_CALL: terminal
arguments: {"command":"hostname -I"}
```

### 7.3 Limitations

- **Unreliable generation** — the model sometimes adds extra text or returns malformed JSON.
- **No native tool support** — unlike the official API, which has structured tool calls.
- **Session drops** — empty responses may occur; the proxy compacts context and retries.

---

## 8. Session Lifecycle & Auto-Recovery

### 8.1 Auto-Reset Triggers

| Condition | Action |
|---|---|
| `messageCount >= MAX_MESSAGE_DEPTH` (100) | Reset the remote session before the next call; keep local history |
| Session age > `SESSION_TTL_MS` (2 hours) | Reset before the next call |
| Upstream HTTP 400/404/500 | Reset the remote session and retry once with a fresh session |
| Empty content response | Compact context and retry up to `DEEPSEEK_MAX_RETRIES` (default 2) |
| Context/content too long | Pre-compact to `DEEPSEEK_MAX_PROMPT_CHARS`, then retry with a smaller budget |
| `/new` user message | Reset that agent's session and clear history |

Idle sessions are evicted after 2× the TTL (default 4 hours).

### 8.2 History Buffer

When a remote session is reset, the proxy preserves the **last 15 exchanges** (capped at 10,000 chars). This recovery context is injected only when the client did not already send multi-turn history:

```
[Previous conversation]
User: what is my IP?
Assistant: Your IP is ...

User: check openvpn accounts
Assistant: TOOL_CALL: terminal
arguments: {"command":"cat /etc/openvpn/server.conf"}

[Continue from here]

<new user prompt>
```

### 8.3 Session Recovery

If the DeepSeek web session expires (HTTP 400/404/500):

1. The current session id is cleared locally.
2. A new session is created via `/api/v0/chat_session/create`.
3. The already-solved PoW answer is reused.
4. The request is retried with `parent_message_id: null` and the recovery prompt.
5. The history buffer is injected as context.

---

## 9. Configuration

### 9.1 Constants (`server.js`)

```javascript
const MAX_HISTORY_LENGTH = 15;              // Keep last 15 exchanges
const MAX_HISTORY_CHARS = 10000;            // Max chars for history buffer
const MAX_MESSAGE_DEPTH = 100;              // Auto-reset after 100 messages
const SESSION_TTL_MS = 2 * 60 * 60 * 1000;  // 2 hours
const MAX_UPSTREAM_PROMPT_CHARS = 80000;    // DEEPSEEK_MAX_PROMPT_CHARS (min 16000)
const MAX_EMPTY_RETRIES = 2;                // DEEPSEEK_MAX_RETRIES (0-10)
const REQUEST_DEADLINE_MS = 120000;         // DEEPSEEK_REQUEST_DEADLINE_MS
const MAX_CONCURRENT = 24;                  // DEEPSEEK_MAX_CONCURRENT
```

The saved auth config:

```javascript
const DS_CONFIG = {
  token: "...",                     // DeepSeek auth token
  hif_dliq: "...",                  // optional custom header
  hif_leim: "...",                  // optional custom header
  cookie: "ds_session_id=...; smidV2=...",  // browser cookies
  wasmUrl: "https://fe-static.deepseek.com/chat/static/sha3_wasm_bg.<hash>.wasm",
};
```

### 9.2 Environment Variables

See `.env.example` for the full list with defaults. Highlights:

- `PORT` (default `9655`), `HOST` (default `127.0.0.1`)
- `PROXY_API_KEY`, `PROXY_API_KEY_FILE`, `REQUIRE_PROXY_API_KEY`
- `PROXY_CORS_ORIGINS`
- `DEEPSEEK_AUTH_PATH`, `DEEPSEEK_AUTH_DIR`
- `DEEPSEEK_ACCOUNT_COOLDOWN_MS`
- `DEEPSEEK_MAX_PROMPT_CHARS`, `DEEPSEEK_MAX_RETRIES`, `DEEPSEEK_REQUEST_DEADLINE_MS`, `DEEPSEEK_MAX_CONCURRENT`
- `NON_INTERACTIVE`, `SKIP_ACCOUNT_MENU`
- `CHROME_PATH`, `DEEPSEEK_CHROME_PORT`, `DEEPSEEK_CHROME_PROFILE`, `DEEPSEEK_KEEP_CHROME_PROFILE`, `DEEPSEEK_REUSE_CHROME`
- `DEEPSEEK_TOKEN`, `DOCTOR_OFFLINE`

### 9.3 Open WebUI / client base URL

```text
http://host.docker.internal:9655/v1   # Open WebUI in Docker
http://localhost:9655/v1              # local
```

If `PROXY_API_KEY` is not set, any API key value is accepted. If it is set, the client must send exactly that bearer token.

---

## 10. Running the Proxy

```bash
# Interactive menu (auth / models / start / quit)
npm start

# Non-interactive (CI, containers)
NON_INTERACTIVE=1 npm start

# Test
curl -s http://127.0.0.1:9655/health
curl -s http://127.0.0.1:9655/readyz
curl -s http://127.0.0.1:9655/v1/models
curl -s http://127.0.0.1:9655/v1/sessions

# Chat
curl -s http://127.0.0.1:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-chat","messages":[{"role":"user","content":"hello"}],"stream":false}'
```

The startup banner prints `[DS-API] Server on http://<HOST>:<PORT> ...` and the active endpoint list.

---

## 11. Error Codes

| HTTP Code | Type | Meaning |
|---|---|---|
| 200 | OK | Response successful |
| 400 | `invalid_model` | Unknown model alias |
| 400 | `unsupported_model` | Known alias not currently usable through the Web API path |
| 400 | `context_length_exceeded` | DeepSeek reported the content/context is too long |
| 401 | `authentication_error` | Missing/invalid proxy API key |
| 403 | `cors_error` | Browser origin not allowed |
| 404 | — | Unknown route, or no local session for the agent |
| 413 | `payload_too_large` | Request body exceeds 10 MB |
| 429 | `rate_limit` | All accounts are cooling down (includes `Retry-After`) |
| 500 | `server_error` | Internal proxy error |
| 502 | `empty_response` (or upstream type) | DeepSeek returned empty content or an upstream error |
| 503 | `no_auth` | No valid DeepSeek auth account |
| 503 | `overloaded` | Too many concurrent completions (`MAX_CONCURRENT`) |
| 504 | `request_timeout` | Request exceeded `REQUEST_DEADLINE_MS` |

Error response format:

```json
{
  "error": {
    "message": "DeepSeek returned empty content",
    "type": "empty_response",
    "agent": "my-agent",
    "failed_session_id": "uuid",
    "message_count": 17,
    "history_length": 5,
    "account": "account-id"
  }
}
```

---

## 12. Known Limitations

| Issue | Cause | Impact |
|---|---|---|
| Empty responses / session drops | DeepSeek web session instability | Conversation interrupted; proxy retries or resets |
| No native tool calling | DeepSeek Web API doesn't support it | Model may generate malformed tool calls |
| Slower than official API | PoW solving + network to DeepSeek | Higher latency per call |
| Session TTL ~2h | DeepSeek web session timeout | Periodic session resets |
| Credentials expire | Browser tokens/cookies change | Re-auth required (`npm run auth`) |
| Shared account quota | Accounts are pooled but limited | Rate limiting across sessions |

---

## 13. Comparison: Web API vs Official API

| Feature | Web API (Proxy) | Official API |
|---|---|---|
| **Cost** | Free (web account) | Paid (per-token) |
| **Tool calling** | Emulated (text injection + parsing) | Native (structured) |
| **Streaming** | Yes | Yes |
| **Reasoning** | Extracted from `THINK` fragments | Native |
| **Reliability** | Medium (session drops) | High (SLA) |
| **Auth** | Saved browser session | API key |
| **PoW** | Required every completion | None |
| **API key needed** | Optional (`PROXY_API_KEY`) | Yes |

---

## 14. Repository Layout

| Path | Purpose |
|---|---|
| `server.js` | The whole proxy/server (HTTP routes, sessions, tool parsing, protocol shims) |
| `lib/pow.js` | Shared WASM proof-of-work solver (also used by `client.js`) |
| `client.js` | Standalone CLI client for the proxy |
| `scripts/auth.js` | Auth manager menu |
| `scripts/deepseek_chrome_auth.js` | Chrome/CDP-based DeepSeek Web login |
| `scripts/auth_import.js` | Import an auth file or browser cookie export |
| `scripts/doctor.js` | Auth/config/PoW diagnostics |
| `scripts/probe_deepseek_models.js` | Probe DeepSeek Web model modes directly |
| `scripts/live_agentic_smoke_tests.mjs` | Live smoke tests against a running proxy |
| `tests/unit.test.js` | Offline unit tests |
| `chrome-extension/` | Separate browser auth helper (not loaded by the server) |
| `Containerfile` | Rootless Podman image (copies `package.json`, `server.js`, `lib/pow.js`) |
| `docs/api-documentation.md` | This document |
