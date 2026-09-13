# FreeDeepseekAPI

<p align="center">
  <strong>Local OpenAI-compatible API proxy for DeepSeek Web Chat</strong>
</p>

<p align="center">
  <a href="https://github.com/ForgetMeAI/FreeDeepseekAPI/blob/main/LICENSE"><img alt="License MIT" src="https://img.shields.io/badge/license-MIT-green.svg" /></a>
  <img alt="Node.js 18 plus" src="https://img.shields.io/badge/node-18%2B-339933.svg" />
  <img alt="No npm dependencies" src="https://img.shields.io/badge/dependencies-0-blue.svg" />
  <img alt="OpenAI compatible" src="https://img.shields.io/badge/OpenAI-compatible-111111.svg" />
</p>

<p align="center">
  <a href="#-quick-start">Quick Start</a> •
  <a href="#-features">Features</a> •
  <a href="#-request-examples">Examples</a> •
  <a href="#-models">Models</a> •
  <a href="#-endpoints">Endpoints</a> •
  <a href="#-open-webui">Open WebUI</a>
</p>

FreeDeepseekAPI spins up a local API server for **DeepSeek Web Chat** (`chat.deepseek.com`) and lets you connect DeepSeek Web to Open WebUI, LiteLLM, Hermes, Claude Code, OpenAI SDK-style clients and other OpenAI-compatible tools.

The project works through your regular logged-in DeepSeek account in a separate Chrome profile. The local server accepts API requests and then talks to DeepSeek Web itself using the saved browser session.

> ⚠️ This is an experimental web-chat proxy. DeepSeek may change its internal Web API without warning. For production use cases the official paid DeepSeek API is more reliable.

ForgetMeAI: https://t.me/forgetmeai

---

## Navigation

- [What This Gives You](#-what-this-gives-you)
- [Features](#-features)
- [Quick Start](#-quick-start)
- [Windows Setup](#-windows-setup)
- [Linux / Chromium Setup](#-linux--chromium-setup)
- [VPS / Headless Setup](#-vps--headless-setup)
- [Rootless Podman](#-rootless-podman)
- [Diagnostics / doctor](#-diagnostics--doctor)
- [Session Reuse and Chat Reset](#-session-reuse-and-chat-reset)
- [Multi-account pool](#-multi-account-pool)
- [Console Auth Ideas](#-console-auth-ideas)
- [Verifying It Works](#-verifying-it-works)
- [Request Examples](#-request-examples)
  - [Chat Completions](#chat-completions)
  - [Reasoning](#reasoning)
  - [Web search](#web-search)
  - [Streaming](#streaming)
  - [Anthropic Messages API](#anthropic-messages-api)
  - [OpenAI Responses API](#openai-responses-api)
  - [Tool calling](#tool-calling)
- [Models](#-models)
- [Endpoints](#-endpoints)
- [Open WebUI](#-open-webui)
- [Refresh Login](#-refresh-login)
- [Project Status](#-project-status)

---

## ✨ What This Gives You

- Use DeepSeek Web as a local API endpoint.
- Connect DeepSeek to Open WebUI and other OpenAI-compatible clients.
- Get regular JSON responses or streaming SSE.
- Use reasoning models with a separate `reasoning_content`.
- Work with the Anthropic Messages API shim for Claude Code / the Anthropic SDK.
- Use the OpenAI Responses API shim for newer OpenAI/Codex-style clients.
- Keep separate web sessions for different agents/users.

## 🚀 Features

- **OpenAI-compatible API:** `POST /v1/chat/completions`
- **Anthropic-compatible shim:** `POST /v1/messages`
- **OpenAI Responses shim:** `POST /v1/responses`
- **Streaming:** SSE chunks and regular non-stream JSON responses
- **Reasoning output:** separate `reasoning_content` for thinking models
- **Tool calling:** parsing of OpenAI tools, Anthropic tools and Responses function tools
- **Model capabilities:** `GET /v1/model-capabilities` with alias → real web mode
- **Agent sessions:** a separate DeepSeek session per `user` / agent id
- **Session recovery:** automatic reset of stale chains/sessions
- **Zero dependencies:** Node.js 18+, no npm dependencies

---

## ⚡ Quick Start

```bash
git clone https://github.com/ForgetMeAI/FreeDeepseekAPI.git
cd FreeDeepseekAPI
npm run auth
npm start
```

`npm run auth` opens the authorization menu:

1. choose option `1`;
2. log in to DeepSeek in the separate Chrome profile;
3. send a short message like `ok`;
4. return to the terminal and press Enter.

`npm start` shows the startup menu:

- `1` — authenticate / refresh DeepSeek login
- `2` — show models and statuses
- `3` — start the proxy
- `4` — quit

For headless/CI startup without the menu:

```bash
NON_INTERACTIVE=1 npm start
# or
SKIP_ACCOUNT_MENU=1 npm start
```

By default the server listens on:

```text
http://localhost:9655
```

By default the proxy is reachable only from this computer. To allow network
access, explicitly set the address and a separate proxy key:

```bash
HOST=0.0.0.0 PROXY_API_KEY='replace-with-a-long-random-value' npm start
```

After that, pass the key as `Authorization: Bearer <key>`. Without
`PROXY_API_KEY`, non-health endpoints remain unauthenticated, so do not
expose such an instance to the network.

Browser requests are allowed from loopback origins. If your UI is served from
another address, add its exact origin as a comma-separated list, for example
`PROXY_CORS_ORIGINS=https://ui.example.com,http://192.168.1.20:3000`.

---

## 🪟 Windows Setup

```powershell
git clone https://github.com/ForgetMeAI/FreeDeepseekAPI.git
cd FreeDeepseekAPI
npm run auth
npm start
```

If Chrome is installed in a non-standard location, specify the path explicitly:

```powershell
$env:CHROME_PATH="C:\Program Files\Google\Chrome\Application\chrome.exe"
npm run auth
```

If Chrome is not found, `npm run auth` now prints ready-made instructions for Windows/macOS/Linux instead of a cryptic stack trace.

---

## 🐧 Linux / Chromium Setup

```bash
git clone https://github.com/ForgetMeAI/FreeDeepseekAPI.git
cd FreeDeepseekAPI
CHROME_PATH=$(which chromium) npm run auth
npm start
```

If Chromium has a different name:

```bash
CHROME_PATH=$(which chromium-browser) npm run auth
# or
CHROME_PATH=$(which google-chrome) npm run auth
```

---

## 🖥 VPS / Headless Setup

The most reliable flow without Chrome on the server:

1. On the home PC that has a GUI/Chrome:

```bash
npm run auth
```

2. Copy `deepseek-auth.json` to the VPS:

```bash
scp deepseek-auth.json user@your-vps:/opt/FreeDeepseekAPI/deepseek-auth.json
```

3. On the VPS, import/verify the file and set safe permissions:

```bash
cd /opt/FreeDeepseekAPI
npm run auth:import -- --input ./deepseek-auth.json
npm run doctor -- --offline
```

4. Start the proxy without the interactive menu:

```bash
NON_INTERACTIVE=1 npm start
```

You can import not only an existing `deepseek-auth.json` but also a browser cookie export:

```bash
DEEPSEEK_TOKEN="<token>" npm run auth:import -- --input ./cookies.json
```

> Important: `deepseek-auth.json` is access to your DeepSeek Web login. Do not commit it, do not publish it, keep it with `0600` permissions.

---

## 🦭 Rootless Podman

The container is intended only for non-interactive proxy startup. Perform browser
authorization on the host with `npm run auth`: the auth scripts and
`deepseek-auth.json` are not copied into the image.

Run Podman as a regular user, without `sudo`.

1. Build the local image:

```bash
podman build --tag localhost/free-deepseek-api:local --file Containerfile .
```

2. Pass the DeepSeek auth and a separate proxy key via Podman secrets:

```bash
podman secret create --replace free-deepseek-auth ./deepseek-auth.json

printf 'Proxy API key: '
IFS= read -r -s PROXY_API_KEY
printf '\n'
printf '%s' "$PROXY_API_KEY" |
  podman secret create --replace free-deepseek-proxy-key -
```

Use a long random key. The value stays in the current shell's `PROXY_API_KEY`
variable so you can test the API; it does not end up in the image or the
Podman command line.

3. Start the container with minimal privileges:

```bash
podman run --detach \
  --name free-deepseek-api \
  --publish 127.0.0.1:9655:9655 \
  --secret free-deepseek-auth,target=deepseek-auth.json,uid=1000,gid=1000,mode=0400 \
  --secret free-deepseek-proxy-key,target=proxy-api-key,uid=1000,gid=1000,mode=0400 \
  --read-only \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  localhost/free-deepseek-api:local
```

Inside the container, `NON_INTERACTIVE=1`, `HOST=0.0.0.0` and the paths to both
secrets are set in advance. `REQUIRE_PROXY_API_KEY=1` prevents the container from
starting if the key secret is missing or empty. On the host the port is published
only on `127.0.0.1`; do not remove that address without a separate network
firewall/access policy.

4. Check liveness, account readiness and a protected endpoint:

```bash
podman healthcheck run free-deepseek-api
curl --fail http://127.0.0.1:9655/readyz
curl --fail \
  -H "Authorization: Bearer $PROXY_API_KEY" \
  http://127.0.0.1:9655/v1/models
```

The built-in healthcheck probes the local `/health` (whether the process is
alive). `/readyz` additionally returns `503` if no DeepSeek auth account is
currently ready to serve requests. Container diagnostics:

```bash
podman logs free-deepseek-api
podman inspect --format '{{.State.Health.Status}}' free-deepseek-api
```

Stop and remove the container together with the saved Podman secrets:

```bash
podman stop free-deepseek-api
podman rm free-deepseek-api
podman secret rm free-deepseek-auth free-deepseek-proxy-key
unset PROXY_API_KEY
```

When rotating auth or the proxy key, replace the corresponding secret and recreate
the container so the behavior does not depend on the Podman version.

---

## 🩺 Diagnostics / doctor

```bash
npm run doctor
# without network requests to DeepSeek:
npm run doctor -- --offline
```

`doctor` checks:

- whether `deepseek-auth.json` / `DEEPSEEK_AUTH_DIR` is found;
- whether the JSON is valid;
- whether `token`, `cookie`, `wasmUrl` are present;
- whether file permissions are safe on macOS/Linux (`0600`);
- during a normal run — whether the DeepSeek PoW endpoint is reachable.

If you see `data.biz_data is null`, `fetch failed`, `401/403/429`, or Hermes/OpenCode cannot see the models — run `npm run doctor` first.

---

## ♻️ Session Reuse and Chat Reset

FreeDeepseekAPI does not create a new DeepSeek chat for every HTTP request without reason. The logic is:

- one `x-agent-session`, `session` or `user` → one DeepSeek chat session;
- if a session id already exists, the proxy reuses it and continues the chain via `parent_message_id`;
- auto-reset happens on TTL, a DeepSeek session error or an overly long message chain;
- local history is kept as a short context so a new DeepSeek session can continue the conversation.
- long agent requests are bounded before sending by `DEEPSEEK_MAX_PROMPT_CHARS` (80,000 characters by default): the start of the task, fresh tool results and the tool adapter are preserved;
- if the client already sent multi-turn history, the local recovery history is not added a second time;
- an empty response is retried at most `DEEPSEEK_MAX_RETRIES` times (2 by default), and the context is reduced on each retry.

Explicitly set an agent/session:

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-agent-session: my-agent" \
  -d '{"model":"deepseek-chat","messages":[{"role":"user","content":"Hello"}]}'
```

List active sessions:

```bash
curl http://localhost:9655/v1/sessions
```

Reset one session:

```bash
curl -X POST "http://localhost:9655/reset-session?agent=my-agent"
```

Reset all sessions:

```bash
curl -X POST "http://localhost:9655/reset-session?agent=all"
```

Why chats still show up in DeepSeek Web: the proxy works through the internal Web Chat API, and DeepSeek stores the real chat sessions on its side. This is normal for a web proxy. The goal of session reuse is to avoid spawning new chats unnecessarily and to reset cleanly only when the chain has gone stale/broken.

---

## 👥 Multi-account pool

You can connect multiple auth files. The correct model is: sticky account per agent/session — the proxy does not switch accounts within a live DeepSeek session. If an account gets `401/403/429` and goes into cooldown, the session is safely reset and a new request may move to another available account.

Option 1 — a directory with auth files:

```bash
mkdir -p accounts
cp deepseek-auth-main.json accounts/main.json
cp deepseek-auth-backup.json accounts/backup.json
chmod 600 accounts/*.json
DEEPSEEK_AUTH_DIR=./accounts NON_INTERACTIVE=1 npm start
```

Option 2 — a list of files:

```bash
DEEPSEEK_AUTH_PATH="./accounts/main.json,./accounts/backup.json" NON_INTERACTIVE=1 npm start
```

How the pool works:

- a new agent/session gets an available account round-robin;
- the chosen account is pinned to the session (`sticky`);
- on `401`, `403`, `429` the account goes into cooldown;
- if a session's sticky account goes into cooldown, the old DeepSeek session is reset so a rate-limited/expired account is not hammered;
- account status is visible in `/health` without auth file paths and without file names;
- auth files must be stored with `0600` permissions.

Configure the cooldown:

```bash
DEEPSEEK_ACCOUNT_COOLDOWN_MS=600000 npm start
```

---

## 🔑 Console Auth Ideas

The password flow from PR #3 can be done, but it is safer not to store the password and not to make it the default. A proper implementation:

1. `npm run auth:console` asks for email/phone and password via a hidden prompt.
2. The password is held only in process memory, not written to files/logs/history.
3. The script replays the Web login flow via `fetch`/CDP: it gets a captcha/verify challenge, gives the human a link/code, and waits for confirmation.
4. After a successful login, only a standard-format `deepseek-auth.json` is saved.
5. If DeepSeek asks for captcha/2FA, the script honestly says "open the link, complete the check, press Enter" rather than trying to bypass the protection.
6. For a VPS, the `auth:console --no-save-password --output deepseek-auth.json` mode is preferable.

Minimal safe MVP: console auth interactive only, without an env password. An acceptable automation variant: `DEEPSEEK_EMAIL=... npm run auth:console`, but the password is still entered via a hidden prompt.

---

## ✅ Verifying It Works

```bash
curl http://localhost:9655/
curl http://localhost:9655/readyz
curl http://localhost:9655/v1/models
curl http://localhost:9655/v1/model-capabilities
```

If everything is fine, `/health` returns the server status, the list of supported aliases and `config_ready: true`, and `/readyz` reports at least one ready account.

---

## 🧪 Request Examples

### Chat Completions

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "Hi! Answer in one sentence."}],
    "stream": false
  }'
```

### Reasoning

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-reasoner",
    "messages": [{"role": "user", "content": "Briefly: why is the sky blue?"}],
    "stream": false
  }'
```

For reasoning models the API returns the chain of thought separately from the final answer:

- non-stream: `choices[0].message.reasoning_content`
- stream: `choices[0].delta.reasoning_content`
- usage: `usage.completion_tokens_details.reasoning_tokens`

`reasoning_tokens` is an approximate estimate based on the extracted DeepSeek Web `THINK` text, because the web stream does not expose official token usage for reasoning separately.

### Web search

```bash
curl -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat-search",
    "messages": [{"role": "user", "content": "Find a fresh fact about DeepSeek and answer briefly."}],
    "stream": false
  }'
```

### Streaming

```bash
curl -N -X POST http://localhost:9655/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [{"role": "user", "content": "Write a short joke."}],
    "stream": true
  }'
```

### Anthropic Messages API

```bash
curl -X POST http://localhost:9655/v1/messages \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "max_tokens": 512,
    "messages": [{"role": "user", "content": "Answer exactly OK"}],
    "stream": false
  }'
```

For Claude Code you can point the backend directly:

```bash
export ANTHROPIC_BASE_URL="http://127.0.0.1:9655"
export ANTHROPIC_AUTH_TOKEN="dummy-key"
export CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY=1
claude --model deepseek-chat
```

### OpenAI Responses API

```bash
curl -X POST http://localhost:9655/v1/responses \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "input": "Answer exactly OK",
    "stream": false
  }'
```

### Tool calling

FreeDeepseekAPI accepts:

- OpenAI `tools`;
- Anthropic `tools`;
- Responses API function tools.

The proxy asks DeepSeek to return a strict JSON tool call, but it can also parse fallback formats:

- `TOOL_CALL:`
- fenced JSON with an explicit `tool_call`, `tool_calls`, or `function_call` envelope
- `<tool_call>...</tool_call>`
- DeepSeek DSML (`<｜DSML｜tool_calls>...`) and the Web variant with `<｜｜DSML｜｜ Tool Calls>`

---

## 🧠 Models

`GET /v1/models` returns only the aliases that are currently verified and working through this proxy.

### Working aliases

| Alias | Web mode | Reasoning | Web search | Comment |
| --- | --- | --- | --- | --- |
| `deepseek-chat` | `Fast` / `default` | no | no | basic chat |
| `deepseek-v3` | `Fast` / `default` | no | no | compatible alias |
| `deepseek-default` | `Fast` / `default` | no | no | compatible alias |
| `deepseek-reasoner` | `Fast` / `default` | yes | no | `thinking_enabled=true` |
| `deepseek-r1` | `Fast` / `default` | yes | no | R1-compatible alias |
| `deepseek-chat-search` | `Fast` / `default` | no | yes | web search |
| `deepseek-default-search` | `Fast` / `default` | no | yes | web search alias |
| `deepseek-reasoner-search` | `Fast` / `default` | yes | yes | reasoning + search |
| `deepseek-r1-search` | `Fast` / `default` | yes | yes | R1-compatible + search |
| `deepseek-expert` | `Expert` / `expert` | no | no | Expert mode |
| `deepseek-v4-pro` | `Expert` / `expert` | yes | no | Expert + reasoning |

Full mapping:

```bash
curl http://localhost:9655/v1/model-capabilities
```

According to the official DeepSeek V4 Preview page, `deepseek-chat` and `deepseek-reasoner` are currently routed to `deepseek-v4-flash` non-thinking/thinking. The direct stream at `chat.deepseek.com` does not expose the exact checkpoint name (`model: ""`), so the proxy records both the web mode (`default` / `Fast`) and the current official routing (`DeepSeek-V4-Flash`).

The current DeepSeek Web remote config output shows these web modes:

- `default` / UI `Fast` — works; supports `thinking_enabled` and `search_enabled`.
- `expert` / UI `Expert` — works through the current web contract (`x-client-version=2.0.0`) and supports `thinking_enabled`. `/v1/models` exposes `deepseek-expert` without reasoning and `deepseek-v4-pro` as Expert + reasoning.
- `vision` / UI `Recognition` — visible in the remote config, but the direct Web API currently returns `backend_err_by_model` (`Vision is temporarily unavailable`). That is why `deepseek-vision` is hidden from `/v1/models`.

Search for Expert is unavailable per the remote config, so `deepseek-expert-search` remains unsupported.

---

## 🔌 Endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/` or `/health` | proxy status (liveness) |
| `GET` | `/readyz` | readiness: 503 unless an uncooldowned account exists |
| `GET` | `/v1/models` | list of working OpenAI-compatible aliases |
| `GET` | `/v1/model-capabilities` | full mapping of aliases, real model, capabilities |
| `POST` | `/v1/chat/completions` | OpenAI-compatible Chat Completions |
| `POST` | `/v1/messages` | Anthropic Messages API shim |
| `POST` | `/v1/responses` | OpenAI Responses API shim |
| `GET` | `/v1/sessions` | active local agent sessions |
| `POST` | `/reset-session?agent=<id>` | reset one session |
| `POST` | `/reset-session?agent=all` | reset all sessions |

---

## 🖥 Open WebUI

Base URL for Open WebUI in Docker:

```text
http://host.docker.internal:9655/v1
```

For a local run without Docker:

```text
http://localhost:9655/v1
```

If `PROXY_API_KEY` is not set, you can specify any API key. If the key is set,
the client must send exactly that key — the proxy checks the bearer token before
accessing models, sessions and completions.

---

## 🔐 Refresh Login

```bash
npm run auth
npm start
```

If DeepSeek starts responding with `401`, `403` or asks for a new PoW/session — repeat `npm run auth` and refresh the saved browser session.

Local authorization files must not end up on GitHub:

- `deepseek-auth.json`
- `.chrome-profile-deepseek/`
- `.env`

They are already added to `.gitignore`.

---

## 🧪 Tests

Project syntax check:

```bash
npm test
```

Live smoke tests against a running local proxy:

```bash
BASE_URL=http://127.0.0.1:9655 MODEL=deepseek-chat npm run test:live
```

---

## 📌 Project Status

FreeDeepseekAPI is an experimental web-chat proxy for local use and integrations. It depends on the current DeepSeek Web Chat contract, so changes on DeepSeek's side may require updating the auth/session logic or the model mapping.

If something stops working:

1. refresh the login via `npm run auth`;
2. check `/v1/model-capabilities`;
3. repeat the request on a fresh session;
4. if the problem persists — DeepSeek has likely changed its internal Web API.

---

<p align="center">
  <strong>ForgetMeAI</strong> · <a href="https://t.me/forgetmeai">Telegram</a>
</p>
