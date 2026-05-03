# any-gateway

> A tiny, stateless local LLM gateway that routes by model name — point Claude Code once and let the model field decide whether to talk to Anthropic or OpenAI.

## About

A single Go binary on `localhost` that forwards LLM traffic between clients (Claude Code, Cursor, scripts) and upstreams (Anthropic, OpenAI). Stateless: no API keys on disk, no caching, no shared memory between requests.

Routing is narrow: if the request body's `model` starts with `openai-`, the gateway translates Anthropic Messages into OpenAI Responses format and forwards to OpenAI. Everything else forwards to Anthropic unchanged. Outgoing requests can additionally pass through an opt-in chain of request-side plugins, starting with a header-injection plugin.

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
| 1 | OpenAI passthrough | ❌ |
| 2 | Request-side plugin system (with header-injection plugin) | ❌ |
| 3 | Claude request forward | ❌ |
| 4 | Anthropic → OpenAI translation | ❌ |

## License

MIT. See `LICENSE`.
