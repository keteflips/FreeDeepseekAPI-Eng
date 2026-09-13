# AGENTS.md

OpenAI/Anthropic-compatible proxy that reverse-engineers the DeepSeek Web chat API (`chat.deepseek.com`). Zero npm dependencies, plain Node >=18, no build step. `server.js` is the whole server (~2.5k lines); `lib/pow.js` is the only shared lib (WASM proof-of-work solver, also used by `client.js`).

## Commands

- `npm test` — syntax-checks every listed entrypoint (`node --check`) then runs `tests/unit.test.js` via the built-in `node --test` runner. This is the full offline suite; there is no separate lint/typecheck.
- Single unit test: `node --test --test-name-pattern="<name>" tests/unit.test.js`.
- `npm run test:live` — smoke tests against a **running** proxy; needs `BASE_URL` and `MODEL`: `BASE_URL=http://127.0.0.1:9655 MODEL=deepseek-chat npm run test:live`.
- `npm start` — interactive menu (auth / models / start / quit). For CI/containers use `NON_INTERACTIVE=1` or `SKIP_ACCOUNT_MENU=1` to start directly.
- `npm run auth` — interactive Chrome-based DeepSeek login, writes `deepseek-auth.json`. `npm run auth:import` imports an existing auth/cookie export. `npm run doctor` (`-- --offline` skips network) diagnoses auth/config and PoW reachability.
- Auth must exist before the server can serve: `deepseek-auth.json` (or `DEEPSEEK_AUTH_DIR`) needs `token`, `cookie`, `wasmUrl`. Never commit it (gitignored; keep `0600`).

## Architecture facts an agent will not infer

- Adding a new syntax-checked file requires editing the explicit `node --check` list in the `package.json` `test` script; it does not glob.
- Multiple DeepSeek accounts are pooled from JSON files (`DEEPSEEK_AUTH_PATH` accepts a comma-separated list; `DEEPSEEK_AUTH_DIR` a directory). Selection is sticky per agent session; accounts cool down on 401/403/429 (`DEEPSEEK_ACCOUNT_COOLDOWN_MS`, default 10m).
- Agent id resolution (`server.js:1915`): `x-agent-session` header, else `params.session`, else `params.user`; with none of those, loopback becomes `dev-agent`, remote callers keyed by IP. Sessions auto-reset at `MAX_MESSAGE_DEPTH` (100) and expire after `SESSION_TTL_MS` (2h idle). Sending the literal user message `/new` resets that agent's session inline. `POST /reset-session?agent=<id|all>` and `GET /v1/sessions` manage them.
- `MODEL_CONFIGS` (`server.js:369`) maps public aliases to DeepSeek Web modes (`model_type` default/expert/vision + `thinking_enabled`/`search_enabled`). Several aliases share one real model; `supported:false` entries (e.g. `deepseek-vision`) are hidden from `/v1/models` but still listed by `/v1/model-capabilities`. Add models there, not in the request handler.
- One POST handler serves three protocols: `/v1/chat/completions` (openai), `/v1/messages` (anthropic), `/v1/responses` (responses). Protocol differences live in `normalizeApiParams` and the `to*Response`/`send*Stream` converters.
- Tool calling is emulated: DeepSeek DSML/JSON markup is parsed back into OpenAI `tool_calls` (block starting `server.js:667`). Prompt length is capped by `DEEPSEEK_MAX_PROMPT_CHARS`/`buildBoundedPrompt`; exceeding it triggers the empty/context-too-long retry path.
- `/health` (liveness, public) and `/readyz` (503 unless an uncooldowned account exists) are distinct. Every other route requires `Authorization: Bearer <PROXY_API_KEY>` when the key is set; `REQUIRE_PROXY_API_KEY=1` fails startup without one. CORS is loopback-only unless `PROXY_CORS_ORIGINS` lists exact origins.
- `lib/pow.js` caches a compiled WASM module and enforces a fetch timeout; a PoW challenge is solved before each completion. `client.js` is a standalone CLI client for the proxy, not part of the server.
- `chrome-extension/` is a separate browser auth helper; not loaded or built by the server or tests.
- Container is non-interactive, read-only, secrets at `/run/secrets/deepseek-auth.json` and `/run/secrets/proxy-api-key`, and copies only `package.json`, `server.js`, `lib/pow.js` (see `Containerfile`). New runtime files must be added to those `COPY` lines or the image breaks.
- `docs/api-documentation.md` mirrors the current implementation (default port 9655, public aliases like `deepseek-chat`). If behavior changes, update it alongside `README.md` and `.env.example`.

## Conventions

- No comments unless they explain a non-obvious workaround; the sparse existing comments encode DeepSeek Web quirks (e.g. `x-client-version=2.0.0` for Expert mode). Preserve them.
- README, docs and all user-facing strings are in English; keep new user-facing text consistent. Code identifiers/comments stay in English. The only intentional non-English strings are the localized DeepSeek error patterns in `server.js` (`isContextTooLongError`) and their test, which must stay multilingual.
- No external dependencies; `package.json` has no `dependencies` field. Anything added must be a Node built-in or vendored.
