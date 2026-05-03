# any-gateway

> A tiny, stateless local LLM gateway that routes by model name — point Claude Code once and let the model field decide whether to talk to Anthropic or OpenAI.

## About

A single Go binary that listens on `localhost`, inspects the request body's `model` field, and forwards to the matching upstream. It owns no API keys and holds no per-user state — every key, host, and request parameter flows from the original request, mutated only when the path requires translation.

The gateway sits between an LLM client (Claude Code, Cursor, scripts) and the upstream providers (Anthropic, OpenAI). The routing rule is intentionally narrow: only the `openai-` prefix triggers anything. If the request body's `model` field starts with `openai-`, the gateway rewrites headers, translates the JSON body and SSE stream from Anthropic's Messages format to OpenAI's Responses format, and forwards to OpenAI. Every other `POST /v1/messages` request — regardless of model name — is forwarded to Anthropic unchanged. For OpenAI clients hitting `/v1/responses` directly, the gateway forwards verbatim.

Outgoing upstream requests can additionally pass through an opt-in chain of request-side plugins — small filters that mutate headers, body, or query before the request leaves the gateway. Plugins ship disabled; each one is enabled individually in `config.yaml`. The first shipped plugin injects a configured header into every upstream request; further plugins are added as discrete features.

## Features

- Forwards `POST /v1/messages` to Anthropic by default, preserving headers, streaming, and the original model name.
- Translates `POST /v1/messages` into an OpenAI Responses call when the model name starts with `openai-` — the prefix is stripped and the rest is sent as the OpenAI model.
- Covers the full translation surface: request body, SSE stream, tool calls, system prompt, multi-turn, errors, and token-usage reporting.
- Forwards `POST /v1/responses` verbatim to OpenAI for clients that already speak the Responses API.
- Runs a configurable chain of request-side plugins that can mutate the outgoing upstream request (headers, body, query). Every plugin is opt-in — the gateway ships with all plugins disabled.
- Includes a header-injection plugin out of the box: enable it in config and the gateway adds a configured header (name + value) to every upstream request.
- Stateless by construction: no API keys on disk or in environment, no caching, no rate limits, no shared memory between requests.

## Roadmap & status

| # | Step | Status |
|---|------|--------|
| 1 | Anthropic default forward | ❌ |
| 2 | OpenAI passthrough | ❌ |
| 3 | Anthropic → OpenAI translation | ❌ |
| 4 | Request-side plugin system (with header-injection plugin) | ❌ |

## License

MIT. See `LICENSE`.
