# Instrumentations

Snapshot from 2026-10 (Node 22, `@opentelemetry/sdk-node` 0.223, stable SDK 2.12). Re-check every version and supported range with `npm view <pkg> version` and the package's `InstrumentationNodeModuleDefinition` (in `build/src/instrumentation.js`) before trusting the table.

## Picks

| Target | Package @ snapshot | Supported range | Notes |
|---|---|---|---|
| http/https server + client | `instrumentation-http` 0.223 | core | Server span, `http.server.request.duration`, `http.client.request.duration` |
| express | `instrumentation-express` 0.71 | `>=4 <6` | **Only source of `http.route`** for the http span and metric |
| `@nestjs/core` | `instrumentation-nestjs-core` 0.69 | `>=4 <13` | Controller + handler spans |
| mongodb driver | `instrumentation-mongodb` 0.76 | `>=4 <8` | Query spans + `db.client.connection.count` pool metric |
| undici / global `fetch` | `instrumentation-undici` 0.33 | `undici >=5.12` | Uses `diagnostics_channel`: loader-agnostic |
| event loop, heap, GC | `instrumentation-runtime-node` 0.36 | — | `nodejs.eventloop.delay.*`, V8 heap, GC; negligible cost |
| mongoose | `instrumentation-mongoose` 0.69 | `>=5.9.7 <10` | **Skip by default**: wraps every mongodb span in a second span (≈2× DB spans) for only the model-level op name |
| ioredis | `instrumentation-ioredis` 0.71 | `>=2 <7` | Add only if Redis is in the request path |
| pg / mysql2 / fastify / koa / graphql | matching `instrumentation-*` | check | Same rules: privacy defaults below, range check |

Not covered by any instrumentation: raw SMTP (nodemailer), custom TCP protocols. Add a manual span if they matter. ClickHouse (`@clickhouse/client` ≥1.20) has a native `tracer` option (`createClient({ tracer: trace.getTracer('clickhouse') })`); without it, its HTTP calls still appear as http client spans.

Typical span count per read request with the picks above: 1 server + 2 Nest + 0–1 express + N mongo ≈ 6–10. Express middleware spans + mongoose double it. Spans per request is the main trace-cost lever.

## `http.route` chain

- `instrumentation-http` puts `http.route` on the server span and on the duration metric, reading it from `RPCMetadata` in context.
- Only `instrumentation-express` sets `RPCMetadata.route` (the matched route template). It does so **before** checking `ignoreLayersType`, so ignoring `middleware` and `request_handler` layers drops the spans but keeps the route.
- `instrumentation-nestjs-core` sets `http.route` only on its own span.
- Unmatched paths (404 from a not-found middleware) get no `http.route`, so scanner traffic collapses to one series per method×status. Leave it that way; never put the raw URL on a metric.

**Hard gate:** `http.route` present (as a template) on the server span **and** the `http.server.request.duration` metric. If it is missing, Express was not patched: see the ESM gap.

## ESM loading gap

OTel's default hook (`require-in-the-middle`) patches CJS `require` only. A library imported through the ESM loader is never patched.

Check: does the framework ship ESM (Nest ≥12 does, `@nestjs/platform-express` does `import express from 'express'`)? Is the build output CJS or ESM? Probe from the repo root with a `-r` script that wraps `Module.prototype.require` and logs names, then `require` the framework adapter: anything missing from the log is loaded via ESM.

Fixes, simplest first:

1. **CJS build, ESM framework:** call `require('express')` right after `sdk.start()` in the preload. The CJS require goes through the hook and patches the cached `module.exports` in place; the framework's ESM import receives the same object. Works because express itself is CJS.
2. **ESM build (`"type": "module"` or `module: nodenext` emitting ESM):** register the ESM hook before anything else: `node --import ./register.mjs dist/main.js`, where `register.mjs` calls `module.register('@opentelemetry/instrumentation/hook.mjs', import.meta.url)` then starts the SDK. The synchronous `import-in-the-middle` hook variant needs Node ≥22.22.3 / ≥24.11.1. Check the prod image's Node version.
3. **Fallback:** a tiny middleware/interceptor that sets `getRPCMetadata(context.active()).route = req.route.path`.

Other cases seen: mongoose imported via ESM but also `require`d somewhere at runtime gets patched by luck (prototype patches apply before the first query); mongodb is CJS-required by mongoose, so it is patched; Nest's own instrumentation re-requires its files for majors ≥12; http and undici are always fine.

## Preload placement

- **First-line import in `main.ts`** (recommended): TS emits `require`s in source order for CJS, so it is equivalent to `node -r`, needs no Dockerfile or manifest change, and works under `nest start --watch` with no flag.
- `node --require ./dist/.../instrumentation.js dist/main` also works.
- Avoid `NODE_OPTIONS=--require …` for watch mode: it loads into the CLI and tsc watcher processes too, and `deleteOutDir` deletes the preload on each rebuild.

## Privacy defaults (checked in source)

| Instrumentation | Captures by default | Do |
|---|---|---|
| mongodb | `db.query.text` with every leaf value replaced by `?` (shape kept) | Keep `enhancedDatabaseReporting: false` (default). `dbStatementSerializer: () => undefined` drops shape too |
| mongoose | `db.query.text` only if a serializer is set | Don't set one |
| ioredis | `db.statement` with a limited number of args | `dbStatementSerializer: (cmd) => cmd` |
| http server | `url.full`, `url.path`, `url.query` (only cloud-signature params redacted), `client.address` from `Forwarded`/`X-Forwarded-For`, `network.peer.address`, `user_agent.original`. No headers unless `headersToSpanAttributes` | Never put `authorization`, `cookie`, `x-api-key` in `headersToSpanAttributes`. Scrub the rest in the exporter |
| nestjs-core | `url.full = req.originalUrl`, raw, query included, no hook to change it | Exporter scrub (the reason the wrapper is required) |
| undici/fetch | `url.full` with query string | Exporter scrub. API keys belong in headers, not query params |
| all, on error | `recordException(e)` + `setStatus({ message: e.message })` | Exporter scrub of status and exception events |

`redactedQueryParams` / `redactedQueryParamsServer` on the http instrumentation are exact-match, case-sensitive, independent lists. Optional defence-in-depth; they don't replace the exporter.

## Known noise (accept, don't patch)

- `instrumentation-nestjs-core` marks its controller/handler spans ERROR on **any** thrown exception, 4xx included. Error dashboards should use the metric's `error.type` (5xx only) and the root HTTP span status (unset for 4xx).
- Mongo `aggregate`-based counts name the collection `$cmd`.
- One `Create Nest App` trace per pod start: keep it, it shows cold-start time.
- Guard time is visible as the gap between the Nest controller span start and the handler span start, with the guard's DB calls named; a dedicated guard span adds nothing.
