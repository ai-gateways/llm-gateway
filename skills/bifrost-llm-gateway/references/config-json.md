# Declarative setup with config.json

Use `config.json` for GitOps, containers, and any deployment where configuration should live in version control. Put it in the Bifrost app directory: `~/.config/bifrost` for npx by default (or the folder passed with `-app-dir`), or the volume mounted at `/app/data` for Docker. The schema is at `https://www.getbifrost.ai/schema`.

## Two modes

| Mode | Set | Behavior |
|---|---|---|
| DB-backed (default) | `config_store` omitted or `"enabled": true` | The file seeds a SQLite or PostgreSQL config store. Dashboard and API edits persist, and an entity changes only when you edit it in the file. |
| File-only | `"config_store": {"enabled": false}` | The file is the only source. Dashboard and API config edits are unavailable; restart to apply changes. |

Details: [Configuration Modes](https://docs.getbifrost.ai/quickstart/gateway/setting-up#configuration-modes) and [Source of Truth](https://docs.getbifrost.ai/deployment-guides/config-json/source-of-truth#default-split-mode).

## Example: two providers, weighted keys, retries, one governed key

```json
{
  "$schema": "https://www.getbifrost.ai/schema",
  "client": {
    "enforce_auth_on_inference": true
  },
  "providers": {
    "openai": {
      "keys": [
        {"name": "openai-key-1", "value": "env.OPENAI_API_KEY", "models": ["*"], "weight": 0.7},
        {"name": "openai-key-2", "value": "env.OPENAI_API_KEY_2", "models": ["*"], "weight": 0.3}
      ],
      "network_config": {
        "max_retries": 3,
        "retry_backoff_initial": 100,
        "retry_backoff_max": 10000
      }
    },
    "anthropic": {
      "keys": [
        {"name": "anthropic-key-1", "value": "env.ANTHROPIC_API_KEY", "models": ["*"], "weight": 1.0}
      ]
    }
  },
  "governance": {
    "virtual_keys": [
      {
        "id": "vk-checkout",
        "name": "checkout-service",
        "is_active": true,
        "provider_configs": [
          {"provider": "openai", "allowed_models": ["gpt-4o-mini"], "key_ids": ["*"], "weight": 1},
          {"provider": "anthropic", "allowed_models": ["*"], "key_ids": ["*"], "weight": 1}
        ],
        "rate_limit_id": "rl-checkout"
      }
    ],
    "budgets": [
      {"id": "budget-checkout-monthly", "virtual_key_id": "vk-checkout", "max_limit": 100.00, "reset_duration": "1M"}
    ],
    "rate_limits": [
      {
        "id": "rl-checkout",
        "request_max_limit": 1000, "request_reset_duration": "1h",
        "token_max_limit": 1000000, "token_reset_duration": "1h"
      }
    ]
  }
}
```

Field notes, all from the [config.json governance reference](https://docs.getbifrost.ai/deployment-guides/config-json/governance#virtual-key-fields):

- `providers.<name>.keys[].value` accepts a literal key or `env.NAME`. Each key name must be unique within its provider.
- `weight` on keys splits traffic across keys of one provider; `weight` on `provider_configs` splits traffic across providers for a virtual key, and the other providers become fallbacks.
- A virtual key's `value` can be `env.NAME` (use a value that starts with `sk-bf-`, for example `sk-bf-$(openssl rand -hex 24)`). Omit it and Bifrost generates one; copy it from the dashboard's **Virtual Keys** page.
- `provider_configs` empty means deny all providers unless `allow_all_providers` is `true`.
- `allowed_models: ["*"]` allows all models; entries starting with `regex:` are RE2 patterns.
- `key_ids: ["*"]` allows every provider key; `[]` or omitted denies all.
- A budget has exactly one owner field: `virtual_key_id`, `team_id`, `customer_id`, `provider_config_id`, or `model_config_id`. Reset windows: `30s`, `5m`, `1h`, `1d`, `1w`, `1M`, `1Q`, `1Y`.
- Rate limits attach through `rate_limit_id` on a virtual key or on one of its `provider_configs`.

## Protect the dashboard and management API

```json
{
  "governance": {
    "auth_config": {
      "is_enabled": true,
      "admin_username": "env.BIFROST_ADMIN_USERNAME",
      "admin_password": "env.BIFROST_ADMIN_PASSWORD",
      "disable_auth_on_inference": false
    }
  }
}
```

Creating the first admin account requires a setup token (`BIFROST_SETUP_TOKEN` in the environment, or `setup_token` in `config.json`). See [Protect the Dashboard](https://docs.getbifrost.ai/quickstart/gateway/setting-up#protect-the-dashboard) and [Admin Authentication](https://docs.getbifrost.ai/deployment-guides/config-json/governance#admin-authentication).

## Routing rules (optional)

For conditional routing (by header, team, budget used, or other CEL conditions) with their own fallback lists, declare `governance.routing_rules`. See [Routing Rules](https://docs.getbifrost.ai/providers/routing-rules#configuration) before writing CEL expressions.

## Kubernetes

The Helm chart generates `config.json` from `bifrost.*` values. Some keys change case there, for example `bifrost.client.enforceAuthOnInference`. See the [Helm quick start](https://docs.getbifrost.ai/deployment-guides/helm#step-2-install) and [Helm governance](https://docs.getbifrost.ai/deployment-guides/helm/governance#virtual-keys).
