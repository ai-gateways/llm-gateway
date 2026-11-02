# Moving existing SDK code onto Bifrost

Bifrost exposes provider-compatible routes, so existing code keeps its SDK and request shapes. Change two things: the base URL, and the API key (use a Bifrost virtual key, `sk-bf-...`). Replace `http://localhost:8080` with your Bifrost URL, and include `server.base_path` if Bifrost is served under a path prefix.

| SDK or tool | Base URL | Where the virtual key goes |
|---|---|---|
| OpenAI SDK (Python, JS) | `http://localhost:8080/openai` | `api_key` / `apiKey` |
| Anthropic SDK | `http://localhost:8080/anthropic` | `api_key` / `apiKey` |
| Google GenAI SDK | `http://localhost:8080/genai` | `api_key` |
| LangChain (`ChatOpenAI`) | `http://localhost:8080/langchain` | `openai_api_key` / `openAIApiKey` |
| Plain HTTP | `http://localhost:8080/v1/chat/completions` | `x-bf-vk` header |

Source: [Drop-in Replacement](https://docs.getbifrost.ai/features/drop-in-replacement#how-it-works) and the per-SDK pages linked below.

## OpenAI SDK

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8080/openai", api_key="sk-bf-...")

client.chat.completions.create(model="gpt-4o-mini", messages=[{"role": "user", "content": "Hi"}])
# Other providers through the same client: prefix the model
client.chat.completions.create(model="anthropic/claude-3-sonnet-20240229", messages=[{"role": "user", "content": "Hi"}])
```

```javascript
import OpenAI from "openai";

const client = new OpenAI({ baseURL: "http://localhost:8080/openai", apiKey: "sk-bf-..." });
```

Docs: [OpenAI SDK setup](https://docs.getbifrost.ai/integrations/openai-sdk/overview#setup).

## Anthropic SDK

```python
import anthropic

client = anthropic.Anthropic(base_url="http://localhost:8080/anthropic", api_key="sk-bf-...")
client.messages.create(model="claude-3-sonnet-20240229", max_tokens=1000,
                       messages=[{"role": "user", "content": "Hello!"}])
```

Bifrost blocks unrecognized headers by default. Clients that send `anthropic-beta` need that header added to the allowlist under **Settings, Client Settings**. Docs: [Anthropic SDK setup](https://docs.getbifrost.ai/integrations/anthropic-sdk/overview#setup).

## Google GenAI SDK

```python
from google import genai
from google.genai.types import HttpOptions

client = genai.Client(api_key="sk-bf-...", http_options=HttpOptions(base_url="http://localhost:8080/genai"))
client.models.generate_content(model="gemini-1.5-flash", contents="Hello!")
```

Docs: [Google GenAI SDK](https://docs.getbifrost.ai/integrations/genai-sdk/overview#setup).

## LangChain

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini",
                 openai_api_base="http://localhost:8080/langchain",
                 openai_api_key="sk-bf-...")
```

Docs: [LangChain SDK](https://docs.getbifrost.ai/integrations/langchain-sdk#setup).

## Coding agents and IDEs

**Claude Code.** Claude Code sends `ANTHROPIC_AUTH_TOKEN` as `Authorization: Bearer <token>`, which Bifrost accepts as a virtual key. Test first, then persist the two variables in the `env` block of `~/.claude/settings.json` (never in a committed project settings file):

```bash
export ANTHROPIC_BASE_URL=http://localhost:8080/anthropic
export ANTHROPIC_AUTH_TOKEN=sk-bf-your-virtual-key
curl -sS -w '\n%{http_code}\n' "$ANTHROPIC_BASE_URL/v1/messages" \
  -H "Authorization: Bearer $ANTHROPIC_AUTH_TOKEN" \
  -H "anthropic-version: 2023-06-01" -H "content-type: application/json" \
  -d '{"model": "claude-sonnet-4-6", "max_tokens": 16, "messages": [{"role": "user", "content": "ping"}]}'
```

A `200` confirms the URL, key, and model; `401` means the key was rejected; `403` usually means the key does not allow that provider or model. Docs: [Claude Code static virtual key](https://docs.getbifrost.ai/cli-agents/claude-code#static-virtual-key).

**Cursor.** In Cursor settings under **Models**, put the virtual key in **OpenAI API Key**, turn on **Override OpenAI Base URL**, and enter `https://<your-bifrost-host>/cursor`. Cursor needs a publicly reachable Bifrost URL. Add models as `provider/model`, for example `anthropic/claude-sonnet-4-5-20250929`. Docs: [Cursor setup](https://docs.getbifrost.ai/cli-agents/cursor#setup).

Other tools (Codex CLI, Gemini CLI, OpenCode, Claude Desktop, and more): [CLI agents overview](https://docs.getbifrost.ai/cli-agents/overview#cli-agents).

## After switching

- Run the existing test suite against Bifrost; responses keep the provider's shape.
- On the drop-in routes, responses carry no `extra_fields`; read the serving provider and fallback status from the [response headers](https://docs.getbifrost.ai/providers/response-headers#routing).
