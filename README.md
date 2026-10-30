# LLM Gateway: What It Is and How It Works

**TL;DR**

- An LLM gateway sits between applications and model providers, exposes one API, and centralizes routing, credentials, budgets, guardrails, and logging.
- An LLM gateway retries and falls back to another model or provider when a call returns a 429 or 5xx error.
- An LLM gateway meters usage in tokens and cost, so platform teams can cap spend per team, application, or key.
- An LLM gateway complements an API gateway: the API gateway handles general HTTP traffic at the edge, and the LLM gateway handles model calls behind it.
- LLM gateways differ most in added latency under load, deployment options, license, and whether they also govern MCP tool and agent traffic.

An LLM gateway is a service that sits between applications and large language model providers, giving every application one API for model access while centralizing routing, credentials, budgets, guardrails, and logging. Teams usually adopt one once they call more than one provider, run more than one AI application, or need a record of who sent which prompt and what it cost. This guide explains how an LLM gateway works, how it differs from an API gateway, and what to check when choosing one.

## What Is an LLM Gateway?

An LLM gateway is a reverse proxy built for model traffic. Applications send requests in one format, usually the OpenAI-compatible chat completions API, and the gateway authenticates, translates, and forwards each request to the right provider. Because every call passes through it, the gateway is the one place where policy, cost tracking, and logging apply to all applications at once.

![Three caller types send requests through one LLM gateway, which forwards them to hosted model APIs, cloud AI platforms, or self-hosted models and exports telemetry](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-llm-gateway/llm-gateway-architecture.png)

*Figure 1: Applications integrate once with the gateway; provider choice, keys, and telemetry move into shared infrastructure.*

Without a gateway, each application integrates provider SDKs directly and keeps its own keys, retries, and logging. That works for one prototype. When five services call three providers, every policy change is made five times and no one has a complete view of usage.

Callers are typically product features, internal tools, AI agents, and coding assistants. Upstreams fall into three groups:

- **Hosted model APIs** such as OpenAI, Anthropic, Google Gemini, and Mistral
- **Cloud AI platforms** such as AWS Bedrock, Azure OpenAI, and Google Vertex AI
- **Self-hosted inference servers** such as vLLM, SGLang, and Ollama

## How an LLM Gateway Works

An LLM gateway processes each request as a short pipeline. It authenticates the caller, checks budgets and rate limits, applies input guardrails, looks up a cache, selects a provider and model, and forwards the call. On the way back it applies output checks, records token usage and cost, and emits logs, metrics, and traces.

![A request passes left to right through authentication, budget and rate limit checks, guardrails, cache lookup, and routing, with fallback to a second provider on 429 or 5xx errors](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-llm-gateway/llm-gateway-request-pipeline.png)

*Figure 2: Cheap checks run first, so a rejected or cached request never reaches a provider or costs tokens.*

- **Authentication.** The caller presents a gateway-issued key instead of a provider key. The gateway maps it to a team, application, or user and holds the provider credentials itself.
- **Budgets and rate limits.** Limits are counted in tokens and dollars as well as requests, because providers bill and throttle by tokens. Provider APIs such as [OpenAI's](https://developers.openai.com/api/docs/guides/rate-limits) enforce per-minute request and token limits and return HTTP 429 when one is exceeded.
- **Guardrails.** Input and output checks detect personal data, secrets, or prompt injection, then block, redact, or flag the request. The [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/) lists the risks most guardrail policies target.
- **Caching.** Exact-match or semantic caching answers a repeated or similar prompt without a provider call.
- **Routing and failover.** Rules select the provider and model by weight, cost, latency, or request attributes. On a 429, a 5xx, or a timeout, the gateway retries and then moves to the next model in the fallback chain.
- **Observability.** Each request produces a log entry with tokens, cost, latency, and caller identity, plus metrics and traces. The [OpenTelemetry semantic conventions for generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) standardize the attribute names.

## Core LLM Gateway Features

Most LLM gateways share a common core: a unified API across providers, central credential management, routing with fallbacks, load balancing across keys, token-based budgets and rate limits, caching, guardrails, and request logging. Products differ in implementation depth, added latency, and where they can run.

| Feature | What it does | Problem it addresses |
|---|---|---|
| Unified API | Accepts one request format and translates it to each provider's API | Provider-specific SDKs and code paths in every application |
| Credential management | Stores provider keys centrally and issues scoped keys to callers | Provider keys copied into many services and laptops |
| Routing and fallbacks | Selects a model per request and switches on errors | Outages and rate-limit errors from a single provider |
| Load balancing | Spreads traffic across keys, regions, and deployments | Per-key rate limits capping throughput |
| Budgets and rate limits | Caps tokens and spend per team, application, or key | Unattributed spend and runaway usage |
| Caching | Returns stored responses for repeated or similar prompts | Paying more than once for the same answer |
| Guardrails | Inspects prompts and responses against policy | Personal data, secrets, and unsafe output reaching providers or users |
| Observability | Logs every request and exports metrics and traces | No record of who called which model and what it cost |

## LLM Gateway vs API Gateway

An API gateway manages general HTTP traffic: TLS termination, authentication, request-count rate limits, and routing to backend services. An LLM gateway manages model traffic specifically: it understands tokens, streamed responses, provider request formats, and model fallbacks. Most organizations run both, with the API gateway at the edge and the LLM gateway behind it.

| Concern | API gateway | LLM gateway |
|---|---|---|
| Unit of metering | Requests | Tokens and cost, plus requests |
| Routing target | Backend services | Model providers and specific models |
| Failure handling | Retries and circuit breaking across service instances | Fallback across providers and model families |
| Payload awareness | Headers, paths, and schemas | Prompts, completions, tool calls, and streamed tokens |
| Caching | HTTP response caching by URL | Exact or semantic caching by prompt |
| Security policy | Authentication, IP rules, schema validation | Prompt and response guardrails, personal data and secrets detection |
| Cost visibility | Not tracked | Spend per team, key, model, and provider |

Some API gateways now ship AI plugins, which blurs the line. A practical test is whether the product meters tokens, translates between provider formats, and falls back across providers; if it does all three, it is functioning as an LLM gateway.

## Why Teams Adopt an LLM Gateway

Teams adopt an LLM gateway when direct provider integrations start to repeat across applications. Common triggers are a provider outage that took a feature offline, spend that cannot be attributed to a team, provider keys spread across many services, and a compliance requirement to log and inspect every prompt and response.

![Without a gateway, each application carries its own keys, retries, and logging to reach the providers; with a gateway, both applications share one layer that holds them](https://articles-images-cdn.t3.tigrisfiles.io/diagrams/ai-gateways-llm-gateway/llm-gateway-without-vs-with.png)

*Figure 3: The gateway replaces per-application copies of keys, retries, and logging with one shared control point.*

- **Reliability.** Provider rate limits and outages become a routing decision instead of an error returned to the user.
- **Cost control.** Spend is attributed and capped in one place, and caching removes repeat calls before they are billed.
- **Security and compliance.** Provider keys live in one system, and every prompt and response passes the same guardrails and audit log. The [NIST AI Risk Management Framework](https://www.nist.gov/itl/ai-risk-management-framework) asks organizations to document and monitor AI use, and gateway logs are a direct source for that record.
- **Model flexibility.** Adding or replacing a model becomes a configuration change, not a code change in every service.

## LLM Gateway Deployment Models

An LLM gateway runs in one of three ways: as a hosted service operated by a vendor, as a self-hosted service inside the company's own cloud account or data center, or as a library embedded in each application. The choice determines where prompts, responses, and logs are processed and stored.

- **Hosted.** The vendor operates the gateway and prompts transit its infrastructure. Setup is fast; data residency depends on the vendor's regions and retention policy.
- **Self-hosted.** The gateway runs in the company's VPC, Kubernetes cluster, or data center. Prompts, responses, and logs stay inside that boundary, and air-gapped deployments with no outbound telemetry are possible.
- **Embedded library.** Routing logic runs inside each application process. There is no extra network hop, but policy is enforced per application, which reintroduces the duplication a gateway removes.

A self-hosted gateway is on the path of every model call, so check for clustering, health-aware load balancing, and restart-free configuration changes.

## How to Evaluate an LLM Gateway

Evaluate an LLM gateway on what it adds to every request and what it lets you control. The key questions are how much latency it adds under production load, where it can be deployed, what its license allows, whether it governs tool and agent traffic as well as model calls, and how it connects to existing identity and monitoring systems.

- **Added latency under load.** Look for a published benchmark that states per-request overhead at a stated throughput and instance size, and reproduce it. Microsecond-level overhead at thousands of requests per second is achievable; millisecond-level overhead compounds in agent loops that make dozens of model calls per task.
- **Deployment and data boundary.** Confirm the gateway runs inside your VPC or on-premises, including air-gapped, and that no prompts, responses, or telemetry leave your infrastructure.
- **License.** A permissive license such as Apache 2.0 or MIT lets you self-host, audit, and modify the code. Check whether governance features sit in a separately licensed directory or a paid tier.
- **Coverage beyond model calls.** Agents also call tools through the [Model Context Protocol](https://modelcontextprotocol.io/specification) and other agents through [A2A](https://a2a-protocol.org/latest/specification/). A gateway that applies one policy to model, MCP tool, and agent traffic avoids running three separate control points.
- **Governance depth.** Scoped keys, budgets and rate limits at the organization, team, and user level, role-based access control, and single sign-on through the identity provider you already use, such as Okta or Microsoft Entra ID.
- **Observability.** Native OpenTelemetry tracing, Prometheus metrics, and log export to the systems you already run.
- **Reliability.** Clustering with no single leader node, circuit breaking, adaptive load balancing, and automatic fallbacks across providers and regions.
- **Compatibility.** An OpenAI-compatible API and drop-in support for your existing SDKs, so migration is a base URL change.

The companion guides explain [how an MCP gateway governs tool calls](https://github.com/ai-gateways/mcp-gateway) and [how an agent gateway governs A2A traffic](https://github.com/ai-gateways/agent-gateway). Before choosing, run a load test with your own prompts and providers, and force a provider to return 429 errors to confirm traffic shifts to the fallback without client-visible errors.

## Open-Source LLM Gateway Projects

Several open-source LLM gateway projects can be self-hosted. The table lists facts each project publishes in its own repository: language, license, traffic covered, overhead benchmark, and deployment options. Benchmark figures are each project's own and use different test setups, so they are not directly comparable.

| Project | Language | License | Traffic covered | Published overhead benchmark | Deployment |
|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost) | Go | Apache 2.0 | LLM, MCP tools, A2A agents | About 11 µs added per request at 5,000 RPS on one t3.xlarge | Self-hosted single binary via npx, Docker, or Kubernetes; in-VPC, on-premises, or air-gapped |
| [LiteLLM](https://github.com/BerriAI/litellm) | Python, with a Rust core | MIT, except the `enterprise/` directory | LLM, MCP tools, A2A agents | 8 ms P95 latency at 1,000 RPS | Self-hosted proxy (Docker) or Python SDK |
| [Agent Router](https://github.com/theagentrouter/agent-router) (formerly Envoy AI Gateway) | Go | Apache 2.0 | LLM, MCP tools | Not published in the repository | Kubernetes with Envoy Gateway, or a standalone CLI |
| [Kong Gateway](https://github.com/Kong/kong) | Lua | Apache 2.0 | HTTP APIs, LLM, MCP tools | Not published in the repository | Docker, Kubernetes ingress controller, or bare metal |

## Frequently Asked Questions

### What is the difference between an LLM gateway and an AI gateway?

The two terms describe largely the same category. "LLM gateway" emphasizes routing and governing calls to large language models. "AI gateway" is broader and often covers embedding, image, and speech models, and increasingly MCP tool calls and agent-to-agent traffic. Check which traffic types a product actually governs rather than relying on its label.

### Is an LLM gateway the same as an LLM proxy?

No. An LLM proxy forwards requests to a provider, usually translating the request format. An LLM gateway includes that proxy function and adds governance: scoped keys, token budgets, rate limits, guardrails, cross-provider fallbacks, and centralized logging. A proxy solves integration; a gateway also solves control and visibility across many applications.

### Do I need an LLM gateway if I only use one model provider?

Often yes. A single provider still enforces rate limits, has outages, and bills by tokens. A gateway balances load across that provider's keys or regions, attributes spend to teams, applies guardrails, and keeps provider keys out of application code. It also turns adding a second provider into a configuration change.

### How much latency does an LLM gateway add?

It depends on the implementation. Published benchmarks for open-source gateways range from microseconds to several milliseconds per request at high throughput. Model responses usually take hundreds of milliseconds, but gateway overhead accumulates in agent workflows that chain many calls, so measure it at your expected request rate.

### Can an LLM gateway run on premises?

Yes, if it is self-hostable. Several open-source gateways run as a container, a single binary, or a Kubernetes deployment inside a private network. For regulated workloads, confirm the gateway runs fully air-gapped, sends no telemetry to its vendor, and stores logs only in systems you control.

### How does an LLM gateway reduce LLM costs?

Budgets and rate limits stop runaway usage per team or key. Caching serves repeated or similar prompts without a provider call. Routing rules send simple requests to cheaper models. Per-team cost attribution shows which applications drive spend, which is usually the first step toward reducing it.

## Related

- [MCP Gateway](https://github.com/ai-gateways/mcp-gateway): how an MCP gateway registers, secures, and observes tool calls from AI applications and agents
- [Agent Gateway](https://github.com/ai-gateways/agent-gateway): how an agent gateway governs agent-to-agent (A2A) traffic
- [AI Gateways](https://github.com/ai-gateways): the full set of guides in this organization
