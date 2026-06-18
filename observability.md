# Observability (`obskit`) — single source of truth

`gravton-observability` (import `obskit`) is the platform's **OpenTelemetry-native, vendor-neutral**
observability layer. The app emits **OTLP once** (logs + traces + metrics); the **OTel Collector**
routes to swappable backends. Local-first by env; production-ready by config. Top-level uv workspace
member — `agentkit`, Django, Airflow, and the CLI all use it.

## The shape
```
                three workloads                three signals          one OTLP stream      Collector fan-out
   ┌─────────────────────────────┐         ┌───────────────────┐
   │ HTTP (Django/DRF/ASGI)       │         │ logs   (shared)   │
   │ AGENT (AI agent SDK)         │ ──────▶ │ traces (per-wkld) │ ──▶ OTLP :4318 ──▶ OTel Collector ──┬─▶ Langfuse (gen_ai.* spans)
   │ AIRFLOW (batch/DAG jobs)     │         │ metrics(per-wkld) │                                     ├─▶ Sentry (error spans)
   └─────────────────────────────┘         └───────────────────┘                                     ├─▶ CloudWatch (metrics EMF + logs)
                                                                                                      └─▶ X-Ray / Tempo (all spans)
   local: no OTLP endpoint ⇒ console exporters · or point at grafana/otel-lgtm for a full local UI
```

## Why (the problem it solves)
Observability used to be a hand-rolled facade with vendor adapters wired *in-app*. That bakes vendor
choices and credentials into the code. The enterprise-standard, genuinely swappable design is: **the
app speaks only OpenTelemetry/OTLP; the Collector owns the backends.** Swapping Langfuse/Sentry/CloudWatch
is a Collector-YAML change — no app redeploy, no creds in the app, and CloudWatch (which needs AWS
SigV4 for direct OTLP) is handled collector-side.

## Three workloads, three signals
- **Logs are shared** — one structured-JSON pipeline; every line carries `correlation_id` and, when a
  span is active, `trace_id`/`span_id` (the only logs↔traces coupling).
- **Traces + metrics are per-workload** — `configure(Workload.X)` sets a `service.name`
  (`gravton-http`/`-agent`/`-airflow`) so the Collector can route by it:
  - **HTTP** — auto-instrumented request spans, RED metrics (`http.server.request.duration`), errors→Sentry.
  - **AGENT** — `gen_ai.*` spans + token/cost/duration metrics → Langfuse.
  - **AIRFLOW** — Airflow's *native* OTel traces+metrics (`AIRFLOW__TRACES__*`/`AIRFLOW__METRICS__*`);
    in-task `gen_ai` spans auto-nest under the task span (deterministic IDs — no manual traceparent).

## Local-first, production-ready (by env only)
| Env | Result |
|---|---|
| *(nothing set)* | **console exporters** — signals print to stdout / task logs (tests/CI, quick local runs) |
| `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318` | full local UI via the standalone `docker compose -f observability/deploy/docker-compose.observability.yml up` (Grafana LGTM at `:3000`) |
| prod env → gateway Collector | same code; Collector routes to Sentry/Langfuse/CloudWatch |
| `OTEL_SDK_DISABLED=true` | hard off (no providers) |
Per-signal override: `OTEL_TRACES_EXPORTER`/`OTEL_METRICS_EXPORTER`/`OTEL_LOGS_EXPORTER` = `console|otlp|none`.

**One-command full local stack** (Collector + Grafana baked into the dev compose, opt-in profile):
```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4318 AIRFLOW_OTEL_ON=True \
  docker compose -f docker-compose-local.yml --profile observability up
# Django (HTTP), Airflow (native + in-task gen_ai spans), and agent telemetry → otel-collector → Grafana :3000
```
Without the profile / endpoint, those same services fall back to console exporters — no broken connections.

## Concurrency / performance
- **Non-blocking export** — long-lived workloads (HTTP/agent) use Batch processors + a periodic metric
  reader; export happens on **background threads**, so the request/agent hot path never blocks on I/O.
- **No in-app vendor I/O** — the only egress is one OTLP POST to the local Collector; PutMetricData /
  Sentry / Langfuse network calls live in the Collector.
- **async-safe** — OTel + correlation use `contextvars` (propagate across `await`, copied into tasks).
  `obskit.aspan(...)` is an async span; `obskit.gather_in_context(*coros)` fans out parallel sub-agent /
  tool / LLM calls with `asyncio.gather` while keeping every child span nested under the parent.
- **Short-lived processes** (Airflow task, CLI) export synchronously (`SimpleSpanProcessor`) + `flush()`
  on teardown so nothing is lost on exit.

## API (what the app calls)
```python
import obskit
obskit.configure(obskit.Workload.AGENT)                      # once per process; console|otlp by env

with obskit.span("invoke_agent:page", "invoke_agent") as s:  # real OTel span (no-op if unconfigured)
    s.set("gen_ai.conversation.id", scan_id)                 # PII keys rejected → use add_event
    obskit.record_llm_call(s, model="gemini/gemini-2.5-flash",
                           input_tokens=120, output_tokens=30)  # stamps gen_ai.* + cost + token metric

obskit.counter("pages_scanned").add(1, {"dag": "seo_scan"})  # low-cardinality attrs only
obskit.report_error(exc, scan_id=sid)                        # records on span (+ Sentry if extra on)

# async fan-out
async with obskit.aspan("invoke_agent:root", "invoke_agent"):
    results = await obskit.gather_in_context(extract(ctx), judge(ctx))
```
`obskit.tracer()` returns a TracePort to inject into agentkit's `RunContext(trace=...)` so Agent spans
become OTel spans.

## Layout
```
obskit/  config(Workload·Settings·exporter_for) · bootstrap(configure→3 providers) · resource ·
         api(span·record_llm_call·report_error·counter/histogram·tracer) · concurrency(aspan·gather_in_context) ·
         semconv(gen_ai.*) · cost · correlation · logging(JSON+trace ids+OTLP bridge) · metrics(OTel/EMF) ·
         trace(in-process offline) · monitoring · ports · integrations/{django,airflow,sentry}
deploy/  docker-compose.observability.yml · otel-collector.local.yaml · otel-collector.prod.yaml
```

## Dependency hygiene (Python 3.14)
- **Core = OTel only** (`opentelemetry-{api,sdk,exporter-otlp-proto-http}` + semconv); **HTTP/protobuf**,
  not gRPC (dodges grpcio's py3.14 gap, matches Langfuse). Already present transitively via Airflow.
- **No vendor SDK in core.** Langfuse + CloudWatch are *collector-side* (zero Python dep). Sentry is an
  optional `sentry` extra for in-app error capture; auto-instrumentation is the `instrumentation` extra.
- Backends + creds live in `deploy/otel-collector.prod.yaml` (env-substituted), never in the app.

## Wiring (as built)
- **Django**: `obskit.configure(Workload.HTTP)` in settings; `MIDDLEWARE`/`LOGGING`/health via
  `obskit.integrations.django`; optional `instrument_django()` + Sentry.
- **Airflow**: `app/seo/llm/runtime.py` `call_llm` opens an obskit `gen_ai` span + `record_llm_call`;
  `runtime.flush()` in `judge_sample`'s finally. Enable native OTel via `AIRFLOW__TRACES/METRICS__*`
  env (see `obskit.integrations.airflow.NATIVE_OTEL_ENV_EXAMPLE`).
- **agentkit**: inject `obskit.tracer()` into `RunContext`; cost via `obskit.cost`.

## Tests
`observability/tests/` — 52 offline tests (in-memory OTel exporters; no network/keys): exporter
selection, per-workload resource, idempotency/disabled handle, span + gen_ai stamping + cost + token
metric, error status, log↔trace correlation, async `gather_in_context` parenting, the Collector YAML
configs, and the optional Sentry integration (fake SDK). Run the SEO/agentkit suites with
`OTEL_SDK_DISABLED=true` to keep them quiet.
