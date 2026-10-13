# Rollout

Order: local → dev (100%) → overhead gate on dev → staging/uat → prod (reduced ratio). Manifests often live outside the app repo; write the exact env block and commands, and let the owner apply them.

## Per-env config

| Var | local | dev / uat | prod |
|---|---|---|---|
| `OTEL_SDK_DISABLED` | unset (off) | `false` | `false` **after the overhead gate** |
| `OTEL_TRACES_SAMPLER` | — | `traceidratio` | `traceidratio` |
| `OTEL_TRACES_SAMPLER_ARG` | — | `1.0` | `0.1` (tune from cost formula) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4318` when opted in | backend default | backend default |
| `OTEL_RESOURCE_ATTRIBUTES` | — | project, namespace, pod, `service.instance.id` | same |
| `OTEL_LOG_LEVEL` | — | `warn` during rollout | `warn` during rollout |

`traceidratio` vs `parentbased_traceidratio`: parent-based honours an incoming `traceparent` sampled flag, so a load balancer or B2B caller sending one can push the effective prod rate to 100%. Plain ratio keeps the rate you set and still continues the caller's trace ID.

## Shutdown grace

The pod's `terminationGracePeriodSeconds` must exceed the time to flush traces and metrics on shutdown (they flush in parallel). Read the exporter's default request timeout from the installed version (`otlp-exporter-base` → `shared-configuration.js`, `timeoutMillis`: 10 s at 0.223) instead of hard-coding it. With a 10 s timeout a 30 s grace is enough; never raise `OTEL_EXPORTER_OTLP_TIMEOUT` past roughly two-thirds of the grace period without raising the grace period too. Spans emitted while the app is closing may be lost: accepted.

## Verify on dev (the first deployed env)

- Traces arrive under the expected `service.name`, with resource attributes (env, project, cluster, zone).
- Server span has `http.route` as a template; DB spans nest under handlers.
- `http.server.request.duration` is visible with `http_route`; a junk path creates no new series.
- Privacy: no `@`, no query values, no peer IP in any span (check one endpoint with a PII query param, one HTTP-client span to a driver that uses URL params, and a forced duplicate-key error).
- A request-log line links to its trace.
- Export errors: none in pod logs with `OTEL_LOG_LEVEL=warn`.

## Overhead gate (before prod)

Same image throughout, one replica, toggling only `OTEL_SDK_DISABLED`.

- Toggle off/on six times (off, on, off, on, off, on), restarting each time:

  ```bash
  kubectl -n <ns> set env deploy/<api> OTEL_SDK_DISABLED=true
  kubectl -n <ns> rollout status deploy/<api>
  ```

- Load from inside the cluster so network noise stays out: ~200 warmup requests, then 60 s at a steady low rate across 2–3 representative routes (one list, one heavy read, one authenticated). Three runs per state per route; take the median.

  ```bash
  kubectl -n <ns> run hey --rm -it --restart=Never --image=williamyeh/hey -- \
    -z 60s -q 5 -c 1 http://<api-svc>:<port>/<route>
  ```

- Measure p50/p95 from the load tool, pod CPU and RSS from the platform's pod metrics over each window.
- Limits (on vs off, at 100% sampling):
  - p50: ≤ +2 ms or +5%, whichever is larger
  - p95: ≤ +10%
  - CPU: ≤ +15% cores used at the same rate
  - RSS: ≤ +60 MB and under 80% of the memory limit
- On failure, re-measure after each step: drop `runtime-node` and DB query text; lower the ratio; stop and rethink.

## Prod readiness check

Before any release that contains the telemetry code reaches prod:

- IAM granted and APIs enabled (GCP: the checklist in [`gcp.md`](gcp.md)).
- **Manifest drift:** read the prod Deployment's env. If `OTEL_SDK_DISABLED=false` is already set (copied from dev) and the image is pulled by a moving tag (`latest`, `imagePullPolicy: Always`), the next release turns the SDK on immediately, at whatever ratio the manifest says (often a dev `1`), with or without IAM and before the gate. Fix the manifest first: ratio to the prod value, and `OTEL_SDK_DISABLED=true` until the gate passes.
- Running image vs code: confirm whether the running image already contains the telemetry code (log lines with trace fields are the quick tell) so env vars are known to be live or inert.
- Grace period checked against the exporter timeout.

## Privacy follow-ups outside the app

Spans and app logs can be clean while the ingress access log still records full query strings. Note it for the infra owner; it is not fixed in the app.
