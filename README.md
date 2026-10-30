# LLM Observability

LLM observability is the practice of capturing and analyzing the prompts, responses, token usage, cost, latency, errors, and tool calls of large language model applications so teams can debug, operate, and govern them in production. Application monitoring shows that a request took four seconds; LLM observability shows which model answered, what it cost, and which tools it called.

This guide explains what to capture, how traces, metrics, and logs divide the work, how the OpenTelemetry GenAI conventions standardize the data, where to capture it, and how to keep sensitive content under control.

## Contents

- [What is LLM observability?](#what-is-llm-observability)
- [What to capture](#what-to-capture)
- [Traces, metrics, and logs](#traces-metrics-and-logs)
- [OpenTelemetry for LLM applications](#opentelemetry-for-llm-applications)
- [Gateway-layer vs SDK instrumentation](#gateway-layer-vs-sdk-instrumentation)
- [Cost and usage attribution](#cost-and-usage-attribution)
- [Privacy and data retention](#privacy-and-data-retention)
- [How to evaluate an LLM observability setup](#how-to-evaluate-an-llm-observability-setup)
- [Open-source LLM observability projects](#open-source-llm-observability-projects)
- [Related guides](#related-guides)

## What is LLM observability?

LLM observability is the telemetry layer for model-backed software. It records each model call, tool call, and agent step with enough context to answer operational questions: why a request was slow, why it failed, what it cost, which team sent it, and what the model actually said. It applies the traces, metrics, and logs of application monitoring to LLM traffic.

```text
application, agent, or coding tool
            |
            v
      capture point     (SDK inside the app, or a gateway on the request path)
            |
            +--> traces    one request, step by step
            +--> metrics   tokens, cost, latency over time
            +--> logs      full request and response records
            |
            v
      existing backends (tracing, metrics, log storage, data warehouse)
```

LLM traffic differs from ordinary API traffic in four ways:

- **Cost varies per request.** Providers bill by tokens, so two calls to the same endpoint can differ in cost by orders of magnitude.
- **Output is nondeterministic.** Debugging often requires the actual prompt and response, not only a status code.
- **Work spans many calls.** One agent task can make dozens of model calls and tool calls, which only make sense when grouped into one trace or session.
- **Content is sensitive.** Prompts and responses often contain personal data, source code, or credentials, so capturing them needs explicit policy.

Observability records what happened; evaluation, which scores whether an answer was correct, consumes the same traces to answer a different question.

## What to capture

An LLM observability setup should capture request metadata, token usage, cost, latency, errors, and the identity of the caller for every model call, plus tool calls and agent operations where they occur. Prompt and response content is the most useful field for debugging and the most sensitive, so it is usually captured under a separate, explicit policy.

| Field | Examples | Why it matters |
|---|---|---|
| Request metadata | [Requested model](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-request-model), [provider](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-provider-name), temperature, max tokens | Explains behavior differences between requests |
| Caller identity | API key, application, team, user | Required for cost attribution and audit |
| Token usage | [Input tokens](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-usage-input-tokens), [output tokens](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-usage-output-tokens), [cached input tokens](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-usage-cache-read-input-tokens) | Drives cost and reveals prompt growth |
| Cost | Price per request, computed from tokens and the model's current rates | Budgeting and chargeback |
| Latency | Total duration, time to first token, time between tokens on streams | User experience and capacity planning |
| Errors and retries | Status, error type such as rate limit or timeout, retries, fallback model used | Separates provider incidents from application bugs |
| Tool calls | [Tool name](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-tool-name), arguments, result status | Shows what an agent did, not only what it said |
| Agent context | [Agent name](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-agent-name), task ID, [conversation ID](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-conversation-id) | Groups many calls into one task or session |
| Content | [Input messages](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-input-messages), [output messages](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-output-messages) | Debugging and quality review, subject to privacy policy |

The [response model](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/#gen-ai-response-model) is worth recording separately, because aliases and fallbacks mean the model that answered is not always the one requested.

## Traces, metrics, and logs

Traces, metrics, and logs answer different questions about LLM traffic. A trace shows the steps of one request or agent task. Metrics aggregate tokens, cost, latency, and errors over time for dashboards and alerts. Logs keep the complete record of each request for search, audit, and debugging. A production setup needs all three, linked by shared IDs.

A trace for one agent task looks like this (durations and token counts are illustrative):

```text
trace: one user request
  invoke_agent   research agent ............. 8.2 s
    chat         model call 1 ............... 1.9 s   1,240 input / 310 output tokens
    execute_tool search (MCP) ............... 0.6 s
    chat         model call 2 ............... 2.4 s   2,860 input / 540 output tokens
```

- **Traces** are made of spans. Each model call, tool execution, and agent invocation becomes a span with its own timing and attributes, nested under the task that caused it.
- **Metrics** are counters and histograms: requests, tokens, cost, latency, time to first token, and errors, labeled by model, provider, and team. Keep high-cardinality values such as user IDs or prompt hashes out of metric labels, because [every unique label combination is a new time series](https://prometheus.io/docs/practices/naming/#labels).
- **Logs** hold one record per request with full metadata and, where policy allows, content. They serve search and audit.

A latency alert from a metric should lead to the traces behind it, and a trace to the logged request.

## OpenTelemetry for LLM applications

OpenTelemetry is the vendor-neutral standard for traces, metrics, and logs, and its GenAI semantic conventions define the span names, attributes, and metrics for model calls, tool execution, agents, MCP, and A2A. Using them means any OpenTelemetry-compatible backend can read LLM telemetry without a custom schema.

The GenAI conventions now live in their own repository, [open-telemetry/semantic-conventions-genai](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md#spans), and cover:

- **Model calls.** [Inference spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md#inference) for chat and completion requests, plus spans for embeddings and retrieval.
- **Tools.** The [execute tool span](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md#execute-tool-span) for function and tool calls.
- **Agents.** The [invoke agent client span](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-agent-spans.md#invoke-agent-client-span) and related spans for agent creation, workflows, and planning.
- **Protocols.** Dedicated conventions for [Model Context Protocol spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/mcp.md#spans) and [Agent2Agent spans](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/a2a.md#spans).
- **Metrics.** Client metrics such as [`gen_ai.client.operation.duration`](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-metrics.md#metric-gen_aiclientoperationduration) and token metrics such as [`gen_ai.client.inference.usage.input_tokens`](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-token-metrics.md#metric-gen_aiclientinferenceusageinput_tokens).

The conventions are still marked Development, so names can change between releases; pin the convention version your instrumentation emits.

## Gateway-layer vs SDK instrumentation

LLM telemetry can be captured inside the application with an SDK or on the network path with a gateway. SDK instrumentation sees in-process steps such as retrieval and prompt assembly. Gateway-layer capture sees every request from every application, with consistent identity, cost, and policy data. Most setups use both.

| Aspect | Gateway-layer capture | SDK instrumentation |
|---|---|---|
| Code changes | None beyond pointing the client at the gateway | Add an SDK or instrumentation library per application and language |
| Coverage | Every application, agent, and coding tool routed through the gateway | Only the applications that were instrumented |
| In-app steps | Not visible unless the app propagates trace context | Retrieval, prompt templates, and function code appear as spans |
| Cost and tokens | Computed once, with one pricing table for all callers | Computed per SDK, sometimes with different pricing data |
| Policy context | Records the key, team, budget, guardrail outcome, and fallback used | Usually unaware of gateway policy decisions |
| Performance | Adds a network hop, so gateway overhead matters | Runs in process, adding work to the application |

The two join through trace context: when the application sends a [W3C `traceparent` header](https://www.w3.org/TR/trace-context/#x3-2-traceparent-header) with each model request, the gateway's spans attach to the application's trace. Gateway capture also covers traffic that cannot be instrumented, such as third-party coding agents, when it is routed through the gateway. The [LLM gateway request pipeline](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works) shows where this capture happens.

## Cost and usage attribution

Cost attribution assigns LLM spend to the teams, applications, keys, users, models, and providers that caused it. It requires two things at the capture point: the caller's identity and a current price for each model, including separate rates for cached input tokens.

```text
input tokens x input price  +  output tokens x output price  =  cost per request
cost per request  ->  summed by key, team, application, model, provider, session
```

- **Attach identity at the edge.** Scoped keys or identity headers let every request carry a team and application without relying on each developer to tag calls.
- **Keep pricing current.** Prices differ by model, region, and batch or cached usage; compute cost from token counts with a maintained pricing table.
- **Count cached tokens separately.** Prompt caching lowers the price of [cached input tokens](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-token-metrics.md#metric-gen_aiclientinferenceusagecache_readinput_tokens), so cost math that treats every input token alike overstates spend.
- **Roll up by session.** An agent task's cost is the sum of every model call it made; grouping by conversation or session ID shows which workflows are expensive.

Once spend is visible per key or team, the same gateway can enforce budgets on it.

## Privacy and data retention

Prompts and responses are the most useful and most sensitive part of LLM telemetry. The OpenTelemetry GenAI conventions state that instrumentations should not capture instructions, inputs, or outputs by default and should make content capture an explicit opt-in. A privacy-aware setup decides what content to keep, where it lives, who can read it, and for how long.

The [content capture guidance](https://github.com/open-telemetry/semantic-conventions-genai/blob/main/docs/gen-ai/gen-ai-spans.md#full-buffered-content) lists three patterns: record no content, record content on span attributes, or store content externally and keep references on the spans. Production systems usually combine them with these controls:

- **Metadata-only mode.** Tokens, cost, latency, errors, and identity answer most operational questions at far lower risk.
- **Redaction before storage.** Detect and mask personal data and secrets before content reaches logs or exporters. [Sensitive information disclosure](https://genai.owasp.org/llmrisk/llm022025-sensitive-information-disclosure/#:~:text=Sensitive%20information%20can%20affect%20both%20the%20LLM%20and%20its%20application%20context) is one of the OWASP Top 10 risks for LLM applications, and telemetry stores are a common place for it to leak.
- **Data location.** Self-hosted capture and storage keep prompts, responses, and logs inside your own network instead of a vendor's.
- **Access control.** Separate permission to view metrics from permission to read raw prompts and responses.
- **Retention limits.** Keep content for days or weeks and aggregated metrics longer.

## How to evaluate an LLM observability setup

Evaluate an LLM observability setup on coverage, standards, where data goes, and what it costs to run. The strongest setups capture every model, tool, and agent call without per-application code, emit standard OpenTelemetry data, export to the tools a team already uses, keep content inside its own infrastructure, and add no meaningful latency to requests.

- **Coverage without code changes.** Gateway capture reaches every application, coding agent, and internal tool routed through it.
- **Standards.** Native OTLP export using the GenAI semantic conventions, plus Prometheus-compatible metrics, so data lands in existing tracing and metrics backends.
- **Export to existing tools.** Built-in exporters for the APM tool, log platform, message queue, or data warehouse already in use avoid a separate silo.
- **One trace across hops.** Model calls, [MCP tool calls](https://github.com/ai-gateways/mcp-gateway#how-to-evaluate-an-mcp-gateway), and [agent operations](https://github.com/ai-gateways/agent-gateway#observability-for-multi-agent-workflows) should share session and trace IDs so a whole task can be followed in one view.
- **Data boundary.** Self-hosted storage, a switch to disable content logging while keeping usage metadata, and redaction applied before export.
- **Overhead.** Logging and export should run asynchronously, off the request path, and per-request overhead should itself be exposed as a metric.
- **Streaming detail.** Time to first token and time between tokens, not only total duration.
- **Cost accuracy.** Maintained pricing for every model and provider, with cached and batch tokens priced correctly.

## Open-source LLM observability projects

The projects below capture at the gateway, instrument application code, or provide a backend for analyzing traces. The table lists facts each project publishes in its own repository; one uses a source-available license.

| Project | Language, license | Approach | OpenTelemetry | What it captures | Deployment |
|---|---|---|---|---|---|
| [Bifrost](https://github.com/maximhq/bifrost/blob/dev/docs/features/observability/default.mdx) | Go, Apache 2.0 | AI gateway that records every request routed through it, with no application code changes | Exports OTLP traces and metrics over HTTP or gRPC in the GenAI semantic conventions format; Prometheus `/metrics` | Model calls with tokens, cost, and latency; tool execution; A2A agent operations; its own per-request overhead | Self-hosted; request logs in SQLite or PostgreSQL |
| [Langfuse](https://github.com/langfuse/langfuse#-core-features) | TypeScript, MIT except the `ee/` directories | Tracing and evaluation platform; applications send traces through SDKs or the API | Repository includes an OTLP ingestion endpoint | Traces, prompt management, evaluations, datasets | Self-hosted with Docker Compose or Kubernetes (Helm); built on ClickHouse |
| [OpenLLMetry](https://github.com/traceloop/openllmetry#-what-do-we-instrument) | Python, Apache 2.0 | Instrumentation libraries and SDK built on OpenTelemetry | Standard OpenTelemetry instrumentations that export to any OpenTelemetry backend or collector | LLM provider calls, vector databases, and frameworks | Python package (`traceloop-sdk`) |
| [Arize Phoenix](https://github.com/Arize-ai/phoenix#trace-your-application) | Python, Elastic License 2.0 (source-available) | Tracing and evaluation application with OpenInference auto-instrumentation | OpenTelemetry-based instrumentation | Traces and evaluations | `pip`, Docker, or Kubernetes (Helm) |
| [OpenLIT](https://github.com/openlit/openlit#-opentelemetry-native-genai-semantic-conventions) | TypeScript, Apache 2.0 | OpenTelemetry-native platform with SDK auto-instrumentation | OTLP endpoint; OpenTelemetry-native tracing | LLM calls, tool calls, MCP requests, agent steps, cost, and GPU usage | Self-hosted with Docker Compose; Python SDK (`openlit`) |

Details reflect each repository in October 2026; confirm current capabilities in each project's documentation before choosing.

Bifrost's request logging, OTLP export, and Prometheus metrics are in its open-source core, while its native Datadog, Kafka, BigQuery, Pub/Sub, and Splunk connectors and object-storage log exports are part of its Enterprise tier.

## Related guides

- [How an LLM gateway works](https://github.com/ai-gateways/llm-gateway#how-an-llm-gateway-works): the request pipeline where gateway-layer capture happens
- [Observability for multi-agent workflows](https://github.com/ai-gateways/agent-gateway#observability-for-multi-agent-workflows): joining agent, model, and tool calls into one session
- [How to evaluate an MCP gateway](https://github.com/ai-gateways/mcp-gateway#how-to-evaluate-an-mcp-gateway): audit and tracing requirements for tool calls
- [The three kinds of AI traffic](https://github.com/ai-gateways/ai-gateway#the-three-kinds-of-ai-traffic): how LLM, MCP, and agent gateways fit together
- [Audit logs and compliance mapping](https://github.com/ai-gateways/ai-governance#audit-logs-and-compliance-mapping): turning request logs into compliance evidence
- [Guardrail actions](https://github.com/ai-gateways/ai-guardrails#guardrail-actions): redaction and content checks applied before data is stored
- [AI Gateways](https://github.com/ai-gateways): all guides in this organization
