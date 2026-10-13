# Code templates

TypeScript, CJS build, NestJS + Express + MongoDB. Replace `<service-name>`, `/healthz` and the instrumentation list with the target's (step 1 inventory). Relative imports use `.js` extensions (nodenext); drop them if the target uses classic resolution.

Templates live as fenced code, not `.ts` files, so the skill folder never gets swept into a project's `tsc` / lint / jest run.

## Contents

- [Preload](#preload) · [SDK setup](#sdk-setup) · [Shutdown](#shutdown)
- [Scrub exporter](#scrub-exporter) · [withSpan](#withspan) · [Log correlation](#log-correlation)
- [Tests](#tests) · [Local Jaeger + env](#local-jaeger--env)

## Packages

```bash
npm i @opentelemetry/api @opentelemetry/core @opentelemetry/sdk-node \
  @opentelemetry/sdk-trace-base @opentelemetry/sdk-metrics @opentelemetry/resources \
  @opentelemetry/exporter-trace-otlp-proto @opentelemetry/exporter-metrics-otlp-proto \
  @opentelemetry/instrumentation-http @opentelemetry/instrumentation-undici \
  @opentelemetry/instrumentation-runtime-node
# per stack: instrumentation-express, instrumentation-nestjs-core, instrumentation-mongodb, …
# GCP only: @opentelemetry/resource-detector-gcp google-auth-library
```

`@opentelemetry/api` is a direct dependency even though `sdk-node` pulls it in: app code imports it for manual spans and log fields.

## Preload

`<lib>/telemetry/instrumentation.ts`, imported as line 1 of `main.ts`:

```ts
// main.ts
// Must stay the first import: the OTel SDK has to patch http/express/db drivers
// before they load. Deep import on purpose — the shared barrel loads the ODM.
import '<lib>/telemetry/instrumentation.js';
import { NestFactory } from '@nestjs/core';
```

```ts
// instrumentation.ts
// Preload: only main.ts imports this file, so tests never start the SDK.
import { config as loadDotenv } from 'dotenv';
import { isTelemetryEnabled, startTelemetry } from './telemetry.js';

// .env is otherwise loaded later by the app's config loader; the OTEL_* switch
// must be visible now. Never overrides real env vars (deployed manifests win).
loadDotenv({ quiet: true });

if (isTelemetryEnabled()) startTelemetry();
```

Drop the `dotenv` lines if the project has no `.env`. Never export `telemetry.ts` / `instrumentation.ts` from a barrel that loads the framework or ODM.

## SDK setup

`<lib>/telemetry/telemetry.ts`, side-effect free:

```ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-proto';
import { OTLPMetricExporter } from '@opentelemetry/exporter-metrics-otlp-proto';
import { PeriodicExportingMetricReader } from '@opentelemetry/sdk-metrics';
import { BatchSpanProcessor } from '@opentelemetry/sdk-trace-base';
import {
  envDetector,
  resourceFromAttributes,
  serviceInstanceIdDetector,
} from '@opentelemetry/resources';
import { gcpDetector } from '@opentelemetry/resource-detector-gcp';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import {
  ExpressInstrumentation,
  ExpressLayerType,
} from '@opentelemetry/instrumentation-express';
import { NestInstrumentation } from '@opentelemetry/instrumentation-nestjs-core';
import { MongoDBInstrumentation } from '@opentelemetry/instrumentation-mongodb';
import { UndiciInstrumentation } from '@opentelemetry/instrumentation-undici';
import { RuntimeNodeInstrumentation } from '@opentelemetry/instrumentation-runtime-node';
import { GoogleAuth } from 'google-auth-library';
import { ScrubbingSpanExporter } from './scrubbing-span-exporter.js';

const GOOGLE_TELEMETRY_ENDPOINT = 'https://telemetry.googleapis.com';
const DEFAULT_SERVICE_NAME = '<service-name>';
const HEALTH_PATH = '/healthz';

// Off unless OTEL_SDK_DISABLED=false (deviation from the OTel default, where
// unset means on) — keeps tests, CI and local quiet with no setup.
export function isTelemetryEnabled(env: NodeJS.ProcessEnv = process.env) {
  return env['OTEL_SDK_DISABLED']?.trim().toLowerCase() === 'false';
}

let sdk: NodeSDK | undefined;

export async function shutdownTelemetry(): Promise<void> {
  await sdk?.shutdown();
}

function googleAuthHeaders(): () => Promise<Record<string, string>> {
  // Resolved lazily on first export so the preload stays synchronous.
  const auth = new GoogleAuth({
    scopes: 'https://www.googleapis.com/auth/cloud-platform',
  });
  return async () => Object.fromEntries(await auth.getRequestHeaders());
}

export function startTelemetry(): void {
  const endpoint = (
    process.env['OTEL_EXPORTER_OTLP_ENDPOINT'] ?? GOOGLE_TELEMETRY_ENDPOINT
  ).replace(/\/$/, '');
  const isGoogle = new URL(endpoint).hostname.endsWith('googleapis.com');
  const headers = isGoogle ? googleAuthHeaders() : undefined;

  sdk = new NodeSDK({
    resource: resourceFromAttributes({
      'service.name': DEFAULT_SERVICE_NAME,
      'deployment.environment.name': process.env['APP_ENV'] ?? 'unknown',
    }),
    // Later detectors win: OTEL_RESOURCE_ATTRIBUTES / OTEL_SERVICE_NAME from
    // the manifest override the random instance id and the defaults.
    resourceDetectors: isGoogle
      ? [serviceInstanceIdDetector, gcpDetector, envDetector]
      : [serviceInstanceIdDetector, envDetector],
    spanProcessors: [
      new BatchSpanProcessor(
        new ScrubbingSpanExporter(
          new OTLPTraceExporter({ url: `${endpoint}/v1/traces`, headers }),
        ),
      ),
    ],
    // Metrics only to a real backend; local Jaeger rejects OTLP metrics.
    metricReaders: isGoogle
      ? [
          new PeriodicExportingMetricReader({
            exporter: new OTLPMetricExporter({
              url: `${endpoint}/v1/metrics`,
              headers,
            }),
            exportIntervalMillis: 60_000,
          }),
        ]
      : [],
    instrumentations: [
      new HttpInstrumentation({
        // Probes would dominate trace volume and add metric points.
        ignoreIncomingRequestHook: (req) =>
          req.url?.startsWith(HEALTH_PATH) ?? false,
        // No orphan root traces from startup / background calls.
        requireParentforOutgoingSpans: true,
      }),
      new ExpressInstrumentation({
        // Route is still recorded; only the per-layer spans go.
        ignoreLayersType: [
          ExpressLayerType.MIDDLEWARE,
          ExpressLayerType.REQUEST_HANDLER,
        ],
      }),
      new NestInstrumentation(),
      new MongoDBInstrumentation(), // enhancedDatabaseReporting stays false
      new UndiciInstrumentation({ requireParentforSpans: true }),
      new RuntimeNodeInstrumentation(),
    ],
  });
  sdk.start();

  // CJS build + ESM framework only (Nest ≥12 imports express via the ESM
  // loader, which the CJS require hook misses). A CJS require here patches the
  // shared module.exports in place, so the framework gets the instrumented
  // copy. See instrumentations.md § ESM loading gap.
  // eslint-disable-next-line @typescript-eslint/no-require-imports
  require('express');
}
```

Non-GCP backend whose metrics endpoint is real: change the `metricReaders` condition from `isGoogle` to "not local Jaeger" (e.g. an `OTEL_METRICS_EXPORTER !== 'none'` check), and drop `gcpDetector` / `google-auth-library`.

## Shutdown

NestJS (requires `app.enableShutdownHooks()` in `main.ts`):

```ts
// Flushes pending spans/metrics on SIGTERM; no-op when telemetry is off.
@Injectable()
class TelemetryShutdown implements OnApplicationShutdown {
  onApplicationShutdown(): Promise<void> {
    return shutdownTelemetry();
  }
}
// add TelemetryShutdown to AppModule providers
```

Nest gives no ordering guarantee between this hook and other modules' shutdown hooks (e.g. the Mongoose connection close), so spans from the last DB calls during shutdown may be lost: accepted. Plain Node/Express, where you own the order: call `await shutdownTelemetry()` at the end of the existing SIGTERM handler, after `server.close()` and DB disconnect.

## Scrub exporter

`scrubbing-span-exporter.ts`:

```ts
import type { Attributes } from '@opentelemetry/api';
import type { ExportResult } from '@opentelemetry/core';
import type {
  ReadableSpan,
  SpanExporter,
  TimedEvent,
} from '@opentelemetry/sdk-trace-base';

// Deny-list privacy scrub applied to every span before export. Spans carry no
// query strings, peer/client IPs or error message text (driver errors embed
// values, e.g. Mongo E11000 dup-key emails).
const DROPPED_KEYS = new Set([
  'url.query',
  'network.peer.address',
  // end-user IP: drop by default; remove these two only on an explicit decision
  'client.address',
  'http.client_ip',
  // old semconv, in case an instrumentation still emits it
  'net.peer.ip',
  'net.sock.peer.addr',
]);
const URL_KEYS = new Set(['url.full', 'http.url', 'http.target']);

function scrubAttributes(attributes: Attributes): Attributes {
  const out: Attributes = {};
  for (const [key, value] of Object.entries(attributes)) {
    if (DROPPED_KEYS.has(key)) continue;
    out[key] =
      URL_KEYS.has(key) && typeof value === 'string'
        ? value.split('?')[0]
        : value;
  }
  return out;
}

function scrubEvent(event: TimedEvent): TimedEvent {
  if (event.name !== 'exception') return event;
  const type = event.attributes?.['exception.type'];
  const stack = event.attributes?.['exception.stacktrace'];
  return {
    ...event,
    attributes: {
      ...(type !== undefined ? { 'exception.type': type } : {}),
      // first stack line is "<Type>: <message>" — the message is what leaks
      ...(typeof stack === 'string'
        ? { 'exception.stacktrace': stack.split('\n').slice(1).join('\n') }
        : {}),
    },
  };
}

export function scrubSpan(span: ReadableSpan): ReadableSpan {
  const events = span.events.map(scrubEvent);
  const exceptionType = events.find((e) => e.name === 'exception')
    ?.attributes?.['exception.type'];
  return {
    name: span.name,
    kind: span.kind,
    spanContext: () => span.spanContext(),
    parentSpanContext: span.parentSpanContext,
    startTime: span.startTime,
    endTime: span.endTime,
    status: {
      code: span.status.code,
      ...(span.status.message !== undefined && exceptionType !== undefined
        ? { message: String(exceptionType) }
        : {}),
    },
    attributes: scrubAttributes(span.attributes),
    links: span.links,
    events,
    duration: span.duration,
    ended: span.ended,
    resource: span.resource,
    instrumentationScope: span.instrumentationScope,
    droppedAttributesCount: span.droppedAttributesCount,
    droppedEventsCount: span.droppedEventsCount,
    droppedLinksCount: span.droppedLinksCount,
  };
}

export class ScrubbingSpanExporter implements SpanExporter {
  constructor(private readonly inner: SpanExporter) {}

  export(
    spans: ReadableSpan[],
    resultCallback: (result: ExportResult) => void,
  ): void {
    this.inner.export(spans.map(scrubSpan), resultCallback);
  }

  shutdown(): Promise<void> {
    return this.inner.shutdown();
  }

  forceFlush(): Promise<void> {
    return this.inner.forceFlush?.() ?? Promise.resolve();
  }
}
```

Why an exporter wrapper and not a span processor: `sdk-trace-base` 2.x has no `onEnding` hook, and `onEnd` receives a read-only span. Why not only `redactedQueryParams*` on the http instrumentation: those lists don't cover the Nest span's `url.full`, HTTP-client spans of other drivers, or exception text.

`ReadableSpan` field names drift between SDK majors (`parentSpanId` → `parentSpanContext` in 2.x). If the target's SDK differs, let `tsc` list the missing fields and match them.

## withSpan

`with-span.ts`, safe to export from the shared barrel (imports only `@opentelemetry/api` and the framework's exception type):

```ts
import { HttpException } from '@nestjs/common';
import { SpanStatusCode, trace } from '@opentelemetry/api';
import type { Attributes, Span } from '@opentelemetry/api';

const tracer = trace.getTracer('app');

// Manual span around one request step. A no-op with the SDK off. Sync or
// async fn: the span ends after the promise settles. 4xx HttpExceptions are
// caller errors, not span errors — only app.http_status.
export function withSpan<T>(
  name: string,
  fn: (span: Span) => T,
  attrs?: Attributes,
): T {
  return tracer.startActiveSpan(name, { attributes: attrs }, (span) => {
    const fail = (err: unknown): void => {
      if (err instanceof HttpException && err.getStatus() < 500) {
        span.setAttribute('app.http_status', err.getStatus());
      } else {
        span.recordException(err instanceof Error ? err : String(err));
        span.setStatus({ code: SpanStatusCode.ERROR });
      }
    };
    let result: T;
    try {
      result = fn(span);
    } catch (err) {
      fail(err);
      span.end();
      throw err;
    }
    if (result instanceof Promise) {
      return result
        .catch((err: unknown) => {
          fail(err);
          throw err;
        })
        .finally(() => span.end()) as T;
    }
    span.end();
    return result;
  });
}
```

Not NestJS: replace the `HttpException` check with the project's own status-bearing error, e.g. `typeof (err as { status?: unknown }).status === 'number' && (err as { status: number }).status < 500`.

Tagging the enclosing span from deeper code (e.g. a cache method): `trace.getActiveSpan()?.setAttribute('app.cache_hit', hit)`.

## Log correlation

Add to the JSON log writer (not the human-readable local mode):

```ts
import { TraceFlags, isSpanContextValid, trace } from '@opentelemetry/api';

// From OTEL_RESOURCE_ATTRIBUTES (set by the manifest). Missing → omit `trace`.
const gcpProjectId = process.env['OTEL_RESOURCE_ATTRIBUTES']
  ?.split(',')
  .map((kv) => kv.split('='))
  .find(([k]) => k?.trim() === 'gcp.project_id')?.[1]
  ?.trim();

// No-op when the SDK is off (no active span).
function traceFields(): Record<string, unknown> {
  const ctx = trace.getActiveSpan()?.spanContext();
  // Ignored routes (health check) carry an all-zero, invalid context
  if (!ctx || !isSpanContextValid(ctx)) return {};
  return {
    ...(gcpProjectId
      ? {
          'logging.googleapis.com/trace': `projects/${gcpProjectId}/traces/${ctx.traceId}`,
        }
      : {}),
    'logging.googleapis.com/spanId': ctx.spanId,
    'logging.googleapis.com/trace_sampled':
      (ctx.traceFlags & TraceFlags.SAMPLED) !== 0,
  };
}

// in emit(): JSON.stringify({ severity, time, ...fields, ...traceFields() })
```

Non-GCP: emit `trace_id`, `span_id`, `trace_flags` (or the backend's documented keys) instead. Reading the project from env avoids a metadata-server call at construct time.

If request logging happens in an interceptor's rxjs `tap`, the AsyncLocalStorage context normally survives; verify the request-log line carries the same trace ID as the server span. Fallback: capture `getActiveSpan()` at the top of `intercept()`.

## Tests

Shared setup: in-memory exporter, no SDK start.

```ts
// scrubbing-span-exporter.spec.ts
import { SpanStatusCode } from '@opentelemetry/api';
import {
  BasicTracerProvider,
  InMemorySpanExporter,
  SimpleSpanProcessor,
} from '@opentelemetry/sdk-trace-base';
import { ScrubbingSpanExporter } from './scrubbing-span-exporter.js';

describe('ScrubbingSpanExporter', () => {
  const memory = new InMemorySpanExporter();
  const tracer = new BasicTracerProvider({
    spanProcessors: [new SimpleSpanProcessor(new ScrubbingSpanExporter(memory))],
  }).getTracer('test');

  afterEach(() => memory.reset());

  it('strips query strings and IPs, keeps user agent and route', () => {
    tracer
      .startSpan('GET /v1/audit-logs', {
        attributes: {
          'url.full': 'https://api.example.com/v1/audit-logs?actor_email=a%40b.com',
          'url.query': 'actor_email=a%40b.com',
          'url.path': '/v1/audit-logs',
          'client.address': '203.0.113.7',
          'http.client_ip': '203.0.113.8',
          'network.peer.address': '10.0.0.1',
          'net.peer.ip': '10.0.0.2',
          'net.sock.peer.addr': '10.0.0.3',
          'user_agent.original': 'Mozilla/5.0',
          'http.route': '/v1/audit-logs',
        },
      })
      .end();

    const [span] = memory.getFinishedSpans();
    expect(span.attributes).toEqual({
      'url.full': 'https://api.example.com/v1/audit-logs',
      'url.path': '/v1/audit-logs',
      'user_agent.original': 'Mozilla/5.0',
      'http.route': '/v1/audit-logs',
    });
    expect(JSON.stringify(span.attributes)).not.toMatch(/@|%40|203\.0|10\.0\.0\./);
  });

  it('removes exception message text (E11000 dup key email)', () => {
    const err = new Error(
      'E11000 duplicate key error collection: users index: email_1 dup key: { email: "a@b.com" }',
    );
    err.name = 'MongoServerError';
    const span = tracer.startSpan('mongodb.insert');
    span.recordException(err);
    span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
    span.end();

    const [exported] = memory.getFinishedSpans();
    expect(exported.status).toEqual({
      code: SpanStatusCode.ERROR,
      message: 'MongoServerError',
    });
    const event = exported.events.find((e) => e.name === 'exception');
    expect(event?.attributes?.['exception.type']).toBe('MongoServerError');
    expect(event?.attributes?.['exception.message']).toBeUndefined();
    expect(JSON.stringify(exported.events)).not.toContain('@');
    expect(exported.spanContext().spanId).toBe(span.spanContext().spanId);
  });
});
```

```ts
// with-span.spec.ts
import { BadRequestException, InternalServerErrorException } from '@nestjs/common';
import { SpanStatusCode, trace } from '@opentelemetry/api';
import {
  BasicTracerProvider,
  InMemorySpanExporter,
  SimpleSpanProcessor,
} from '@opentelemetry/sdk-trace-base';
import { withSpan } from './with-span.js';

describe('withSpan', () => {
  const memory = new InMemorySpanExporter();

  beforeAll(() => {
    // withSpan's tracer is the global proxy tracer, so register a provider.
    trace.setGlobalTracerProvider(
      new BasicTracerProvider({ spanProcessors: [new SimpleSpanProcessor(memory)] }),
    );
  });
  afterEach(() => memory.reset());
  afterAll(() => trace.disable());

  it('ends the span with attrs on async success', async () => {
    await expect(
      withSpan('a.b', () => Promise.resolve('ok'), { 'app.offer_count': 3 }),
    ).resolves.toBe('ok');
    const [span] = memory.getFinishedSpans();
    expect(span.name).toBe('a.b');
    expect(span.attributes['app.offer_count']).toBe(3);
    expect(span.status.code).toBe(SpanStatusCode.UNSET);
  });

  it('works for sync fn and passes the span in', () => {
    expect(withSpan('a.sync', (s) => (s.setAttribute('app.x', 1), 7))).toBe(7);
    expect(memory.getFinishedSpans()[0].attributes['app.x']).toBe(1);
  });

  it('4xx: sets app.http_status, no error, rethrows', async () => {
    const err = new BadRequestException('nope');
    await expect(withSpan('a.b', () => Promise.reject(err))).rejects.toBe(err);
    const [span] = memory.getFinishedSpans();
    expect(span.attributes['app.http_status']).toBe(400);
    expect(span.status.code).toBe(SpanStatusCode.UNSET);
    expect(span.events).toHaveLength(0);
  });

  it.each([
    ['5xx HttpException', () => new InternalServerErrorException('x')],
    ['plain Error', () => new Error('boom')],
  ])('%s: recorded as ERROR, rethrown', async (_n, make) => {
    const err = make();
    await expect(withSpan('a.b', () => Promise.reject(err))).rejects.toBe(err);
    const [span] = memory.getFinishedSpans();
    expect(span.status.code).toBe(SpanStatusCode.ERROR);
    expect(span.events.some((e) => e.name === 'exception')).toBe(true);
  });

  it('sync throw ends the span and rethrows', () => {
    expect(() =>
      withSpan('a.sync', () => {
        throw new Error('boom');
      }),
    ).toThrow('boom');
    expect(memory.getFinishedSpans()).toHaveLength(1);
  });
});
```

```ts
// telemetry.spec.ts
import { trace } from '@opentelemetry/api';
import { isTelemetryEnabled, shutdownTelemetry } from './telemetry.js';

describe('telemetry preload', () => {
  it('is off unless OTEL_SDK_DISABLED=false', () => {
    expect(isTelemetryEnabled({})).toBe(false);
    expect(isTelemetryEnabled({ OTEL_SDK_DISABLED: 'true' })).toBe(false);
    expect(isTelemetryEnabled({ OTEL_SDK_DISABLED: '' })).toBe(false);
    expect(isTelemetryEnabled({ OTEL_SDK_DISABLED: 'false' })).toBe(true);
    expect(isTelemetryEnabled({ OTEL_SDK_DISABLED: ' FALSE ' })).toBe(true);
  });

  it('starts nothing on import', async () => {
    // No SDK registered → the API hands out non-recording spans.
    const span = trace.getTracer('t').startSpan('s');
    expect(span.isRecording()).toBe(false);
    span.end();
    await expect(shutdownTelemetry()).resolves.toBeUndefined();
  });
});
```

Log-correlation spec: make a span active with `trace.setSpanContext(context.active(), { traceId, spanId, traceFlags })` + `context.with(...)` (or `jest.spyOn(trace, 'getActiveSpan')`), then assert: fields present and `trace_sampled` matches the flags; unsampled → `false`; no active span → no fields; `INVALID_SPAN_CONTEXT` → no fields; no project ID → no `trace` field but `spanId` kept.

## Local Jaeger + env

```yaml
# docker-compose.yaml — opt-in: docker compose --profile tracing up -d jaeger
# UI at http://localhost:16686
  jaeger:
    image: jaegertracing/jaeger:<pinned-version>
    profiles: ['tracing']
    ports:
      - '16686:16686' # UI
      - '4318:4318' # OTLP/HTTP
```

```bash
# .env.example
# Telemetry is off unless OTEL_SDK_DISABLED=false. Local opt-in with Jaeger:
#   docker compose --profile tracing up -d jaeger   → UI http://localhost:16686
# Deployed envs set these in the manifest.
# OTEL_SDK_DISABLED=false
# OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
# OTEL_TRACES_SAMPLER=traceidratio
# OTEL_TRACES_SAMPLER_ARG=1.0
# OTEL_LOG_LEVEL=warn
```

No console exporter: it floods stdout and breaks structured logging.
