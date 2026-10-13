---
name: opentelemetry-node
description: >
  OpenTelemetry setup for a Node.js / NestJS service: SDK preload, OTLP trace +
  metric export (GCP Telemetry API, Collector, Jaeger), privacy-scrubbing
  exporter, manual spans, log↔trace correlation, per-env rollout. Use when
  adding tracing or latency metrics to a Node service, wiring OTLP export,
  scrubbing PII from spans, or linking logs to traces.
---

Instrument one Node service with the OpenTelemetry SDK so every request gets a trace and a `http.server.request.duration` histogram labelled by route, with no PII exported. Distilled from a shipped NestJS 12 + Express 5 + MongoDB service exporting OTLP direct to GCP; the decisions carry to other stacks, the code templates are Node/TypeScript.

Reference files, reached from the steps:

- [`references/code.md`](references/code.md): copy-ready templates (preload, SDK setup, scrub exporter, `withSpan`, shutdown, log fields, tests, Jaeger).
- [`references/instrumentations.md`](references/instrumentations.md): per-library notes, privacy defaults, the ESM loading gap.
- [`references/gcp.md`](references/gcp.md): GCP Telemetry API export, Workload Identity IAM, metric landing rules, cost.
- [`references/rollout.md`](references/rollout.md): per-env config, sampling, shutdown grace, overhead gate, prod readiness.

## Locked defaults

These are deliberate. Keep them unless the target project has a reason, and say so when deviating.

- **Off unless opted in.** Telemetry starts only when `OTEL_SDK_DISABLED=false`. This inverts the OTel default (unset = on) so tests, CI and local need zero setup and no cloud credentials.
- **Standard `OTEL_*` env vars**, read outside the app's config schema: the SDK starts before DI/config loading.
- **`traceidratio` sampler, not `parentbased_traceidratio`.** A caller's `traceparent` sampled flag cannot force 100%; the trace ID is still continued. Revisit parent-based once you own the upstream (frontend propagation).
- **Individual instrumentation packages**, never `@opentelemetry/auto-instrumentations-node` (39 instrumentations, and its `register` entry installs a SIGTERM handler that races the framework's shutdown).
- **No Collector** unless the platform already runs one: in-process OTLP http/protobuf export.
- **Deny-list scrub exporter** wraps the trace exporter. Every span passes through it.
- **Cumulative metric temporality, 60 s export interval, default histogram buckets, no Views.**

## Steps

### 1. Inventory the target

Record: Node version, framework (Nest/Express/Fastify/plain), whether the compiled output is CJS or ESM (`"type"` in `package.json`, tsconfig `module`), and whether the framework itself ships ESM (Nest ≥12 does). List every client in the request path: DB drivers, ODMs, Redis, HTTP/fetch, queues, analytics stores. Note the health-check route, the logger, and where the app handles shutdown.

Then check versions: `npm view <pkg> version` for each candidate package, and read each instrumentation's supported range against the installed library version ([`instrumentations.md`](references/instrumentations.md) lists the 2026-10 snapshot; ranges move).

**Done when** every request-path client is mapped to an instrumentation (or "covered by http/undici", or "manual span"), and every chosen package's range covers the installed version.

### 2. Choose the export backend

- **GCP (Cloud Trace + Cloud Monitoring):** OTLP direct to `https://telemetry.googleapis.com`, ADC auth. Read [`gcp.md`](references/gcp.md) in full: there are rejection rules and IAM roles that differ from older docs.
- **OTel Collector / Grafana / Honeycomb / Datadog agent:** set `OTEL_EXPORTER_OTLP_ENDPOINT` (and `OTEL_EXPORTER_OTLP_HEADERS` for vendor keys); the same `telemetry.ts` sends traces with the Google auth branch unused. Its metric reader is gated on the Google endpoint, so change that condition ([`code.md` § SDK setup](references/code.md#sdk-setup), note under the block) or no metrics leave the process.
- **Local:** Jaeger all-in-one on `:4318` (traces only; it rejects OTLP metrics, so the template sends metrics only to a real backend).

**Done when** the backend, its endpoint, its auth mechanism and which signals (traces, metrics) it accepts are written down.

### 3. Add the preload and SDK setup

Copy `instrumentation.ts` and `telemetry.ts` from [`code.md`](references/code.md). Wire the preload by the build output found in step 1:

- **CJS output:** the preload is the **first import** of the entrypoint (`main.ts`), as a deep import. Going through a shared-lib barrel that loads DB schemas or the framework would start them before the SDK patches them. If the framework ships ESM (Nest ≥12 imports `express` via the ESM loader), keep the template's `require('express')` after `sdk.start()`: OTel's default hook patches CJS `require` only.
- **ESM output:** a first import is too late, because the loader links the whole graph before any module runs. Start with `node --import ./register.mjs dist/main.js` in the start script **and** the Dockerfile; Node version floor applies.

Details and the probe for which libraries load via ESM: [`instrumentations.md` § ESM](references/instrumentations.md#esm-loading-gap).

Add the shutdown hook ([`code.md` § Shutdown](references/code.md#shutdown)): `sdk.shutdown()` from the framework's own shutdown lifecycle, never a second `process.on('SIGTERM')`.

**Done when** the app boots with `OTEL_SDK_DISABLED` unset and nothing starts, and boots with `OTEL_SDK_DISABLED=false OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318` against local Jaeger and shows a request trace whose server span carries `http.route` as the route **template** (`/v1/items/:id`), with DB child spans under the handler.

### 4. Scrub privacy at the exporter

Copy `ScrubbingSpanExporter` and its spec. It must, for every span:

- strip the query string from `url.full` / `http.url` / `http.target`, and drop `url.query`. Nest's instrumentation writes raw `req.originalUrl` and has no hook; HTTP-client spans for drivers that put params in URLs (ClickHouse `param_*`) leak the same way.
- drop peer addresses (`network.peer.address`, old-semconv `net.peer.ip`, `net.sock.peer.addr`).
- drop the end-user IP (`client.address`, `http.client_ip`) **by default**. Keeping it is a per-project decision; record who made it.
- on exceptions, keep `exception.type`, drop `exception.message` and the first stacktrace line, set `status.message` to the type. Driver errors embed values: Mongo `E11000 dup key: { email: "…" }`.

Then grep the codebase for query params and unique fields that carry PII (emails, tokens, free text) and add a test case for each leak shape found.

**Done when** the spec passes and a local trace of an endpoint with a PII query param and a forced duplicate-key error shows no `@`, no query values and no peer IP in any exported attribute, status or event.

### 5. Add manual spans where auto spans leave blank time

Copy `withSpan(name, fn, attrs?)`. Look at local waterfalls of the slowest endpoints: add a span only where a gap has no auto child (CPU work, fan-out `Promise.all`, cache lookup, fire-and-forget continuation). Guards and single queries are already named by their DB spans.

Convention, add it to the project's `CLAUDE.md`/`AGENTS.md`:

- names `<module>.<step>`, snake_case (`recommendations.score`)
- semantic-convention attribute keys first, else `app.<snake_case>`
- **attribute allowlist**: tenant/org ID or slug, one entity ID (never lists), counts and sizes, booleans, enums, coarse location. User-typed text, user IDs and emails are banned; record `app.has_text_query=true` instead.
- 4xx is a caller error: `app.http_status`, not span ERROR.

**Done when** each slow endpoint's waterfall has no unexplained gap over ~10% of its duration, and every manual attribute is on the allowlist.

### 6. Correlate logs with traces

Add the active span's trace ID, span ID and sampled flag to every structured log line ([`code.md` § Log correlation](references/code.md#log-correlation)). Emit them even for unsampled requests: logs still group per request. GCP field names are `logging.googleapis.com/trace` (`projects/<project-id>/traces/<traceId>`), `…/spanId`, `…/trace_sampled`; other backends want `trace_id` / `span_id`. Redact the same PII query keys in the request-log URL as in step 4: logs leak what spans no longer do.

**Done when** a deployed (or local JSON-mode) request-log line carries the trace ID of its Jaeger/Cloud Trace trace, and a line logged outside a request carries no trace fields.

### 7. Test without a backend

Unit tests use `InMemorySpanExporter` + `SimpleSpanProcessor` on a `BasicTracerProvider`: no SDK start, no network. Required specs: preload switch is off for unset/`true`/empty; scrub exporter (URL, IP, exception); `withSpan` (async success, sync, 4xx, 5xx/plain error, sync throw); log trace fields (present, absent, no project ID). Templates in [`code.md`](references/code.md#tests).

**Done when** the full unit and e2e suites pass with no `OTEL_*` variable set and no cloud credentials.

### 8. Roll out per environment

Follow [`rollout.md`](references/rollout.md): IAM per env, sampler ratio per env, `OTEL_RESOURCE_ATTRIBUTES` from the downward API, grace period, the overhead gate before prod, and the readiness check. Set `OTEL_LOG_LEVEL=warn` during rollout: export failures (403, rejected points) are otherwise silent.

**Done when** each env shows traces and the duration metric with route labels in the backend, prod passed the overhead gate at its target ratio, and no env has the SDK enabled without its IAM grant.
