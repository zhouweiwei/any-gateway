# any-gateway

> A tiny, stateless local LLM gateway that routes by model name — point Claude Code once and let the model field decide whether to talk to Anthropic or OpenAI.

A single Go binary that listens on `localhost`, inspects the URL path and (for `/v1/messages`) the model name in the request body, then forwards to a configured upstream. It owns no API keys and holds no per-user state. The gateway can rewrite request headers and bodies when the path requires translation, but it never reads secrets from disk or environment.

## The problem

You like the Claude Code CLI. You sometimes want to point it at GPT-5 — for cost, for speed, for a second opinion, or because Anthropic is having a bad afternoon. The alternatives today are heavy: LiteLLM-proxy is a full Python service with a database; OpenRouter is a paid hosted product; LangChain wrappers replace your CLI entirely. None of them fit the case where the entire problem is "rewrite a few headers and translate one streaming JSON schema."

any-gateway is the smallest thing that closes this gap. Set `ANTHROPIC_BASE_URL=http://localhost:8787` once, then choose your destination per session by setting `ANTHROPIC_MODEL` to either a Claude name (passthrough) or an `openai-*` name (translated). It is also a useful transparent forwarder for `/v1/responses`, with a self-served `/v1/models` that advertises the gateway's curated menu — so it can sit in front of all your LLM traffic for inspection.

## Who this is for

Developers running `claude` (Anthropic's CLI) on their own laptop who occasionally want to drive an OpenAI model from the same CLI, and who do not want to introduce a database, a Python runtime, or an account on a hosted proxy to do so. Secondary: anyone who wants a one-line-per-request log of their LLM traffic during local development.

Not for: production teams needing rate limiting, cost tracking, multi-tenant key vaulting, or remote deployment. The whole design assumes a single user on `localhost`.

## What it does

Three endpoints, one routing rule based on the request body's `model` field.

| Endpoint | Routing rule | Upstream call |
|---|---|---|
| `POST /v1/responses` | none — always OpenAI | `<upstreams.openai>/v1/responses` |
| `POST /v1/messages` | model matches `messages_routes` table | depends on matched route |
| `GET  /v1/models` | none — served from gateway config | none (no upstream call) |

For `/v1/messages`, the gateway parses the request body's `model` field, matches it top-down against the configured `messages_routes`, and acts on the first match. The default route table sends `claude-*` straight through to Anthropic and treats `openai-*` as a translated route to OpenAI's Responses API.

`/v1/models` is served by the gateway itself: it returns the `models` list from `config.yaml` in OpenAI's standard response shape. This presents a unified menu — Claude passthrough names and `openai-*` translated names side-by-side — so a client sees the gateway's curated set of models rather than either upstream's full inventory.

### Primary flows

1. **Claude Code → Anthropic (passthrough).** `ANTHROPIC_BASE_URL=http://localhost:8787`, `ANTHROPIC_MODEL=claude-opus-4-5`. The gateway recognises the Claude model, forwards the request unchanged.
2. **Claude Code → OpenAI (translated).** `ANTHROPIC_BASE_URL=http://localhost:8787`, `ANTHROPIC_MODEL=openai-gpt-5`, with an OpenAI key in `ANTHROPIC_API_KEY`. The gateway sees the `openai-` prefix, strips it, rewrites the auth header, translates the JSON body and SSE stream, and forwards to OpenAI's Responses API.
3. **OpenAI client → OpenAI (passthrough).** Cursor or scripts speaking the Responses API set `OPENAI_BASE_URL=http://localhost:8787/v1`. Calls to `/v1/responses` are forwarded verbatim to `<upstreams.openai>/v1/responses`. Calls to `/v1/models` get the gateway's curated list rather than OpenAI's full inventory — the client only sees what `config.yaml` advertises.

### How translation works (flow 2)

The gateway is stateless: every key, every URL host, every request param flows from the original request. The gateway only mutates what's strictly necessary.

- **Headers.** `x-api-key: <V>` becomes `Authorization: Bearer <V>`. `anthropic-version` is dropped. `Host` is rewritten to the upstream host. Other transport headers pass through.
- **Request body.** The `model` field is rewritten according to the matched route's `rewrite_model` (default `${1}` strips the `openai-` prefix). Anthropic's top-level `system` becomes OpenAI's `instructions`. `messages[]` becomes `input[]` with role-tagged content items. Anthropic `tool_use` and `tool_result` content blocks become OpenAI Responses `function_call` and `function_call_output` items. `max_tokens` becomes `max_output_tokens`.
- **Response stream.** OpenAI Responses SSE events (`response.created`, `response.output_text.delta`, `response.output_item.added`, `response.completed`) are translated into Anthropic Messages SSE events (`message_start`, `content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`, `message_stop`). Tool-call streaming aggregates `input_json` deltas and re-emits them as Anthropic-shaped `tool_use` blocks with stable `index` values.
- **Errors.** OpenAI error JSON is repackaged into Anthropic's error envelope, preserving the HTTP status code, so Claude Code's error handling still triggers.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Language | Go 1.22+ | Best stdlib fit for SSE rewriting and HTTP proxying; same ecosystem as Ollama, openai-forward, one-api. |
| HTTP server | `net/http` | Built-in streaming flush via `http.Flusher`; no framework needed at this size. |
| SSE parsing | `bufio.Scanner` with custom `SplitFunc` | ~10 LOC for line-aware event boundary handling. |
| JSON | `encoding/json` with typed structs | Schemas are small and known; typed structs catch upstream drift early. |
| Config | YAML via `gopkg.in/yaml.v3` | Reads better than TOML for glob keys and route tables. |
| Distribution | `go install` | Single static binary, ~15 MB. Homebrew packaging deferred. |

### Architecture

```
                ┌─────────────────────────────────────────────────────┐
client ─POST──► │  any-gateway :8787                                  │
                │                                                     │
                │  /v1/responses ──────► proxy ──► upstreams.openai   │
                │                                                     │
                │  /v1/messages                                       │
                │   │  parse body → model field                       │
                │   │  match against messages_routes (top-down)       │
                │   │                                                 │
                │   ├─[match: claude-*]─► proxy ──► upstreams.anthropic│
                │   │                                                 │
                │   └─[match: openai-*]                               │
                │       ├─ rewrite header (x-api-key → Bearer)        │
                │       ├─ rewrite body (model + Claude→Responses)    │
                │       ├─ stream translate (SSE ↔ SSE)               │
                │       └────────────────► upstreams.openai           │
                │                                                     │
                │  /v1/models ─────────► gateway models registry      │
                │                         (served from config.yaml)  │
                └─────────────────────────────────────────────────────┘
```

The proxy and translator paths share no state. The translator is a pipeline: parse model → match route → request transform → upstream call → response stream transform.

## Quick start

Requires Go 1.22+ and macOS 13+ (Linux works too).

```bash
# Install
go install github.com/zhouweiwei/any-gateway/cmd/any-gateway@latest

# Configure (defaults below; all fields shown)
cat > config.yaml <<EOF
upstreams:
  openai:    https://api.openai.com
  anthropic: https://api.anthropic.com

models:
  - id: claude-opus-4-5
    owned_by: anthropic
  - id: claude-sonnet-4-6
    owned_by: anthropic
  - id: openai-gpt-5
    owned_by: openai
  - id: openai-gpt-5-mini
    owned_by: openai
  - id: openai-gpt-5-nano
    owned_by: openai

messages_routes:
  - match: "openai-*"
    upstream: openai
    translate: true
    rewrite_model: "\${1}"
  - match: "*"
    upstream: anthropic
EOF

# Run (foreground, Ctrl+C to stop)
any-gateway --config config.yaml
# 2026-05-01T15:32:01Z listening on :8787
```

In another terminal, point your client at the gateway.

```bash
# Claude Code → Anthropic (passthrough)
ANTHROPIC_BASE_URL=http://localhost:8787 \
  ANTHROPIC_MODEL=claude-opus-4-5 \
  ANTHROPIC_API_KEY=sk-ant-... \
  claude

# Claude Code → OpenAI (translated)
ANTHROPIC_BASE_URL=http://localhost:8787 \
  ANTHROPIC_MODEL=openai-gpt-5 \
  ANTHROPIC_API_KEY=sk-openai-... \
  claude

# OpenAI client → OpenAI (passthrough)
OPENAI_BASE_URL=http://localhost:8787/v1 \
  OPENAI_API_KEY=sk-openai-... \
  cursor
```

## Configuration

One file, three sections: upstream URLs, the model menu the gateway advertises, and the `/v1/messages` route table.

```yaml
# config.yaml
upstreams:
  openai:    https://api.openai.com
  anthropic: https://api.anthropic.com

# Models advertised by GET /v1/models. These are the names a client may
# put in its request body's "model" field. Each entry is returned in the
# OpenAI /v1/models response shape: {id, object: "model", owned_by}.
models:
  - id: claude-opus-4-5
    owned_by: anthropic
  - id: claude-sonnet-4-6
    owned_by: anthropic
  - id: openai-gpt-5
    owned_by: openai
  - id: openai-gpt-5-mini
    owned_by: openai
  - id: openai-gpt-5-nano
    owned_by: openai

# POST /v1/messages routing rules. Evaluated top-down against the
# request body's "model" field. First match wins. No match → 400.
messages_routes:
  - match: "openai-*"           # e.g. openai-gpt-5  →  gpt-5 on OpenAI
    upstream: openai
    translate: true             # Claude Messages → OpenAI Responses
    rewrite_model: "${1}"       # ${1} captures whatever * matched
  - match: "*"                  # catch-all fallback
    upstream: anthropic         # forward verbatim to Anthropic
```

Field reference:

- `upstreams.<name>` — base URL for an upstream provider. Referenced by `messages_routes[].upstream`. The gateway preserves the path it was called with (`/v1/messages` stays `/v1/messages`) unless `translate: true`, in which case the path is rewritten to `/v1/responses`.
- `models[]` — the menu returned by `GET /v1/models`. The gateway does not validate that these IDs match `messages_routes` patterns; the user is expected to keep them aligned. Models listed here but not matched by any route will return a 400 if a client tries to use them.
- `messages_routes[].match` — glob pattern matched against the request body's `model` string. `*` is the standard glob wildcard.
- `messages_routes[].upstream` — name of an entry in `upstreams`.
- `messages_routes[].translate` — when `true`, the gateway rewrites the request body and SSE stream from Claude Messages format to OpenAI Responses format.
- `messages_routes[].rewrite_model` — optional. Replaces the model name sent upstream. Use `${1}`, `${2}`, … to refer to globs captured in `match`. If omitted, the original model name is sent upstream.

`/v1/responses` always forwards to `upstreams.openai`. `/v1/models` is served entirely from the `models` config without any upstream call. There is no per-route override for these in v1 — to use multiple OpenAI-compatible backends, run multiple gateways.

API keys, the listen port, and TLS are not configured here. Keys come from each client's request. The port comes from `--port` (default `8787`). TLS is not supported. The gateway has no place to leak secrets to.

## Logging

One line per request to stdout, RFC3339 timestamp, method, path, upstream host, status, latency, response bytes:

```
2026-05-01T15:32:14Z POST /v1/messages model=openai-gpt-5 → api.openai.com/v1/responses 200 412ms 18.2KB
2026-05-01T15:32:18Z POST /v1/messages model=claude-opus-4-5 → api.anthropic.com/v1/messages 200 318ms 12.4KB
2026-05-01T15:32:22Z GET  /v1/models → (local) 200 1ms 412B
```

`--verbose` adds the matched route name and the request byte count. Bodies are never logged.

## Project structure

```
any-gateway/
├── cmd/any-gateway/        # main.go — flag parsing, config loading, server start
├── internal/proxy/         # transparent forwarders for /v1/responses and claude→claude /v1/messages
├── internal/translator/    # Claude Messages ↔ OpenAI Responses request, response, SSE, error translation
├── internal/router/        # parse model field, match messages_routes, dispatch
├── internal/registry/      # GET /v1/models handler — serves the configured models list
├── internal/config/        # YAML loader, glob compilation, route validation
├── internal/translator/testdata/  # golden-file fixtures for translator
├── go.mod
├── go.sum
├── config.example.yaml
└── README.md
```

## Development

```bash
# Run tests (unit + golden-file fixtures for the translator)
go test ./...

# Run with hot-reload during dev
go run ./cmd/any-gateway --config config.yaml --verbose

# Build a release binary
go build -o any-gateway ./cmd/any-gateway
```

The translator has golden-file tests under `internal/translator/testdata/`: real Anthropic request bodies in, expected OpenAI request bodies out, and recorded SSE streams in both directions. Adding a new content-block type means dropping a fixture pair and watching the existing translator code light up.

## Roadmap

### v1 (in scope)

- Pass-through proxy for `POST /v1/responses` to `<upstreams.openai>`.
- Self-served `GET /v1/models` returning the configured `models` list in OpenAI's response shape.
- Model-name dispatch for `POST /v1/messages` via `messages_routes`.
- Pass-through path of `/v1/messages` (claude-* models) to `<upstreams.anthropic>`.
- Full-fidelity translator path of `/v1/messages` (openai-* models) covering streaming, tool calls, system prompt, multi-turn, errors, and token-usage reporting.
- `config.yaml` with named upstreams and glob-pattern route table.
- Foreground binary, stdout logging, `--verbose`, `--port`, `--config` flags.

### Later

- Homebrew formula and launchd plist for `brew services start any-gateway`.
- `--record` flag to dump translated request/response pairs for translator debugging.
- Reverse direction: OpenAI → Claude (an OpenAI client driving Claude Opus).
- Additional upstreams behind named entries in `upstreams` (e.g. `gemini`, `ollama`).
- Merged `/v1/models` response that augments the configured registry with live entries discovered by polling upstream `/v1/models`.
- Optional Chat Completions endpoint for clients that have not migrated to the Responses API.

### Not building (explicit out-of-scope)

- **OpenAI → Claude reverse translation.** Symmetric translator doubles the test surface. Deferred until v1 stabilises.
- **Other upstream providers.** No Gemini, Ollama, Bedrock, Mistral plugin system. Each new upstream is a deliberate v2 addition.
- **Caching, rate limiting, quotas, cost tracking.** All require state and config. The whole gateway is sized to fit in a head.
- **Auth, TLS, multi-user, remote deploy.** Local single-user is the assumed shape; the gateway holds no keys, so multi-tenant isolation is meaningless.
- **OpenAI Chat Completions (`/v1/chat/completions`).** Only the newer Responses API is supported in v1. Clients that only speak Chat Completions are out of scope until they migrate.

## Tradeoffs considered

- **Go over Rust.** Chose Go for the SSE-rewrite + header/body mutation workload because `bufio.Scanner` plus `http.Flusher` plus typed structs is the shortest path to a correct streaming translator. Cost: higher p99 floor than Rust (~50–200 µs per request), invisible at single-user latencies dominated by upstream TTFB.
- **Stateless gateway over keyed daemon.** Chose to forbid the gateway from reading any API key from disk or environment. Keys live in the client's request and are mutated only as bytes flowing through. Cost: cannot rotate keys centrally; cannot rate-limit per-user; cannot run as a shared multi-tenant service. All explicit non-goals.
- **Model-name dispatch over URL-prefix dispatch.** Chose to inspect the request body's `model` field rather than require clients to use distinct URL prefixes per destination. Cost: small JSON parse on every `/v1/messages` request, plus a config table to maintain. Pays back as zero client config — Claude Code points at one URL forever, and the model field controls everything via env or `--model`.
- **OpenAI Responses API over Chat Completions.** Chose Responses for its cleaner mapping to Anthropic's content blocks and forward-looking schema. Cost: Cursor and other Chat-Completions-only clients cannot use the translated path until they migrate; documented as a non-goal.
- **YAML config with named upstreams over hard-coded endpoints.** Chose to put upstream URLs in config so the gateway can target staging endpoints, self-hosted OpenAI-compatible servers (Ollama, vLLM), or alternate Anthropic-compatible endpoints. Cost: one extra config concept. Pays back as the only knob needed when the user wants to swap endpoints.

## Risks

- **Anthropic SSE schema drift.** Anthropic ships new event types without notice. Mitigation: golden-file fixtures under `internal/translator/testdata/` flagged by `go test`; new event types fail closed rather than emit wrong shapes downstream.
- **OpenAI Responses API churn.** The Responses API is newer than Chat Completions and still evolving. Mitigation: pin upstream behaviour via recorded fixtures; surface translator failures as Anthropic-shaped error responses so the client retries cleanly.
- **Tool-call streaming edge cases.** Aggregating `input_json` deltas into stable Anthropic `tool_use` blocks is the gnarliest part of the translator. Edge cases include partial JSON across multiple `input_json_delta` events, abandoned tool calls when the model bails mid-stream, and concurrent tool calls with overlapping indices. Mitigation: explicit state machine with assertions; fixtures cover the known edge cases.
- **Client-side model-name validation.** Claude Code may reject `openai-*` model strings before they reach the gateway. Mitigation: documented escape hatch — choose model names that pass client validation (e.g. embed the disambiguator in a different field) and adjust the `match` patterns accordingly.

## Success metric

I run Claude Code through `http://localhost:8787` against `openai-gpt-5` for a full week of daily work without falling back to direct `claude` or to a different setup, and the route table correctly handles a mix of Claude and OpenAI sessions in the same shell. If that holds, v1 has cleared its bar.

## License

MIT. See `LICENSE`.
