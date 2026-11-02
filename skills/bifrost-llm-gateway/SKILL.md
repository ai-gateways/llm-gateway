---
name: bifrost-llm-gateway
description: Set up Bifrost, the open-source LLM gateway by Maxim AI, in front of OpenAI, Anthropic, AWS Bedrock, Azure OpenAI, Vertex AI, Gemini, and other model providers. Gives every app one OpenAI-compatible endpoint, central provider API keys, automatic fallbacks, weighted load balancing, and virtual keys with budgets and rate limits. Use when the user wants an LLM gateway or AI gateway, one unified or OpenAI-compatible API for many models, provider failover, LLM rate limiting or spend caps, to migrate OpenAI, Anthropic, Google GenAI, or LangChain SDK code by changing the base URL, or when an existing API gateway cannot meter tokens or fail over between model providers.
compatibility: Runs Bifrost with npx (Node.js) or Docker and needs network access to the model providers.
metadata:
  author: Maxim AI
  version: "1.0"
---

# Bifrost LLM gateway

Bifrost is an open-source AI gateway (Apache 2.0) built by Maxim AI. As an LLM gateway it sits between applications and model providers: apps call one OpenAI-compatible endpoint, and Bifrost holds the provider keys, picks a key and provider, retries, falls back, and enforces per-key budgets and rate limits.

Use this skill to install Bifrost, connect providers, move existing SDK code onto it, and add failover and spend controls. For MCP tool traffic use `bifrost-mcp-gateway`; for A2A agent traffic use `bifrost-agent-gateway`; for all three behind one gateway use `bifrost-ai-gateway`.

## When to use

- One API for several model providers, or an OpenAI-compatible endpoint for non-OpenAI models.
- Failover when a provider returns errors, times out, or rate limits.
- Provider API keys spread across services that should live in one place.
- Budgets and token or request rate limits per app, team, or customer.
- An existing API gateway (NGINX, Kong, a cloud API gateway) is already in place. Keep it at the edge for TLS and network policy, and put Bifrost behind it: an API gateway meters requests, while an LLM gateway meters tokens and cost, translates provider formats, and falls back across providers. See [LLM gateway vs API gateway](https://github.com/ai-gateways/llm-gateway#llm-gateway-vs-api-gateway).

## Steps

### 1. Start Bifrost

Export provider keys in the shell that starts Bifrost. Keys written as `env.NAME` are read from the Bifrost process environment, so with Docker pass them with `-e`.

```bash
export OPENAI_API_KEY="..."
export ANTHROPIC_API_KEY="..."

# Option A: npx
npx -y @maximhq/bifrost

# Option B: Docker, with data persisted in ./data
docker run -p 8080:8080 -v $(pwd)/data:/app/data \
  -e OPENAI_API_KEY -e ANTHROPIC_API_KEY maximhq/bifrost
```

The dashboard and API share one port: `http://localhost:8080`. These unpinned commands are for local evaluation. Before exposing Bifrost beyond localhost, pin a version, set a dashboard password with a setup token, and enable TLS ([Protect the Dashboard](https://docs.getbifrost.ai/quickstart/gateway/setting-up#protect-the-dashboard)).

### 2. Add providers

Use the dashboard (**Model Providers**) or the management API. Two keys with weights 0.7 and 0.3 split OpenAI traffic 70/30 across accounts:

```bash
curl -X POST http://localhost:8080/api/providers \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "openai",
    "keys": [
      {"name": "openai-key-1", "value": "env.OPENAI_API_KEY", "models": ["*"], "weight": 0.7},
      {"name": "openai-key-2", "value": "env.OPENAI_API_KEY_2", "models": ["*"], "weight": 0.3}
    ],
    "network_config": {"max_retries": 3, "retry_backoff_initial": 100, "retry_backoff_max": 10000}
  }'

curl -X POST http://localhost:8080/api/providers \
  -H "Content-Type: application/json" \
  -d '{
    "provider": "anthropic",
    "keys": [{"name": "anthropic-key-1", "value": "env.ANTHROPIC_API_KEY", "models": ["*"], "weight": 1.0}]
  }'
```

Each key name must be unique within its provider. Retry backoff values are in milliseconds. For GitOps or no-UI deployments, declare the same settings in `config.json` instead: see [references/config-json.md](references/config-json.md).

### 3. Send a request

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "openai/gpt-4o-mini", "messages": [{"role": "user", "content": "Hello, Bifrost!"}]}'
```

Models are named `provider/model`. A bare name such as `gpt-4o-mini` is resolved to a provider through the model catalog.

### 4. Add fallbacks

Per request, list `provider/model` fallbacks in the body. Each fallback gets its own retry budget, and they run in order until one succeeds:

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-4o-mini",
    "messages": [{"role": "user", "content": "Explain quantum computing in simple terms"}],
    "fallbacks": ["anthropic/claude-3-5-sonnet-20241022"]
  }'
```

The response's `extra_fields.routing_info` names the provider that served the request and sets `is_fallback`. To get failover without changing request bodies (for example in drop-in SDK code), give a virtual key one `provider_configs` entry per provider that serves the model, with weights: Bifrost picks a provider by weight and adds the rest as fallbacks ([Governance-based Routing](https://docs.getbifrost.ai/providers/provider-routing#governance-based-routing)).

### 5. Issue virtual keys with budgets and rate limits

A virtual key (`sk-bf-...`) is what apps send instead of a provider key. Provider access is deny-by-default, so list each provider the key may use (or set `"allow_all_providers": true`).

```bash
curl -s -X POST http://localhost:8080/api/governance/virtual-keys \
  -H "Content-Type: application/json" \
  -d '{
    "name": "checkout-service",
    "provider_configs": [
      {"provider": "openai", "weight": 1.0, "allowed_models": ["gpt-4o-mini"], "key_ids": ["*"]}
    ],
    "budgets": [{"max_limit": 100.00, "reset_duration": "1M"}],
    "rate_limit": {
      "token_max_limit": 100000, "token_reset_duration": "1h",
      "request_max_limit": 100, "request_reset_duration": "1m"
    },
    "is_active": true
  }' | jq -r '.virtual_key.value'
```

Budgets are in USD. Reset durations: `1m`, `1h`, `1d`, `1w`, `1M`, `1Q`, `1Y`. Budgets also exist on teams, customers, and per provider config; a request must pass every level ([Budget and Limits](https://docs.getbifrost.ai/features/governance/budget-and-limits#budget-checking-flow)).

Then require a virtual key on every inference call:

```bash
curl -X PUT http://localhost:8080/api/config \
  -H "Content-Type: application/json" \
  -d '{"client_config": {"enforce_auth_on_inference": true}}'
```

If dashboard authentication is on, add your admin credentials (`Authorization: Basic ...` or a session token) to every `/api/*` call.

### 6. Point existing code at Bifrost

Change only the base URL and use the virtual key as the API key:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8080/openai",
    api_key="sk-bf-...",  # Bifrost virtual key
)
client.chat.completions.create(
    model="anthropic/claude-3-5-sonnet-20241022",
    messages=[{"role": "user", "content": "Hello"}],
)
```

Base URLs for the Anthropic, Google GenAI, and LangChain SDKs, plus Claude Code and Cursor, are in [references/sdk-migration.md](references/sdk-migration.md).

## Verify

1. `curl -s http://localhost:8080/health` returns a success response.
2. `curl -s http://localhost:8080/v1/models -H "x-bf-vk: $BIFROST_VK"` lists only the models the key allows.
3. A chat completion sent with `x-bf-vk: $BIFROST_VK` returns `200`, and a call without a key returns an auth error once enforcement is on.
4. The request appears in the dashboard's request logs under the provider and model that served it.
5. Send a request with a bad primary model and a valid fallback; `extra_fields.routing_info.is_fallback` is `true`.

## Gotchas

- `401` means the virtual key was rejected; `403` usually means the key does not allow that provider or model.
- A virtual key with empty `provider_configs` and `allow_all_providers` off blocks every provider.
- When both dashboard auth and inference auth are on, `Authorization` carries the admin credential, so send the virtual key in `x-bf-vk` ([Authentication and Virtual Keys](https://docs.getbifrost.ai/features/governance/virtual-keys#authentication-and-virtual-keys)).
- One open-source node keeps provider configs, budgets, and usage in memory; multiple OSS nodes sharing one Postgres database are not supported. Cluster mode with state sync is a Bifrost Enterprise feature ([Budget and Limits overview](https://docs.getbifrost.ai/features/governance/budget-and-limits#overview)).
- Adaptive load balancing (routing on real-time performance metrics) is a Bifrost Enterprise feature; weighted keys and virtual-key weights are open source.

## Reference

- [references/config-json.md](references/config-json.md): declarative `config.json` for providers, retries, virtual keys, budgets, and rate limits.
- [references/sdk-migration.md](references/sdk-migration.md): base URLs for each SDK and coding agent.
- Bifrost docs: [Provider Configuration](https://docs.getbifrost.ai/quickstart/gateway/provider-configuration#multi-provider-setup), [Retries and Fallbacks](https://docs.getbifrost.ai/features/retries-and-fallbacks#fallbacks), [Virtual Keys](https://docs.getbifrost.ai/features/governance/virtual-keys#configuration), [Drop-in Replacement](https://docs.getbifrost.ai/features/drop-in-replacement#zero-code-changes).
- Source: [maximhq/bifrost](https://github.com/maximhq/bifrost#quick-start).
