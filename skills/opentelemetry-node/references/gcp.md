# GCP export (Cloud Trace + Cloud Monitoring)

As of 2026-10. Re-check the linked Google pages: the Telemetry API is recent and its docs have disagreed with each other.

## Export path

- OTLP **http/protobuf** from the SDK straight to `https://telemetry.googleapis.com` (SDK appends `/v1/traces`, `/v1/metrics`). No Collector needed.
- Use the **global** endpoint. Regional endpoints drop metrics whose location doesn't match.
- Packages: `exporter-trace-otlp-proto`, `exporter-metrics-otlp-proto`, `@opentelemetry/resource-detector-gcp`, `google-auth-library`.
- The `@google-cloud/opentelemetry-cloud-trace-exporter`, `…-cloud-monitoring-exporter` and `…-resource-util` packages are **deprecated** (repo archived after 2026-10-30). Never add them.
- **Auth:** async `headers()` on the exporter returning `GoogleAuth.getRequestHeaders()` (ADC; Workload Identity on GKE). Resolve the client lazily inside `headers()` so the preload stays synchronous; Google's sample awaits it before `sdk.start()`, which a sync preload can't do.
- Don't put `x-goog-user-project` in `OTEL_EXPORTER_OTLP_HEADERS`: Google warns it duplicates a header and fails.

Docs: <https://docs.cloud.google.com/trace/docs/migrate-to-otlp-endpoints>, <https://docs.cloud.google.com/stackdriver/docs/reference/telemetry/overview>, migration guide <https://github.com/GoogleCloudPlatform/opentelemetry-operations-js/blob/main/MIGRATION.md>.

## Resource attributes

- `gcpDetector` gives `cloud.*`, `host.*`, `k8s.cluster.name` from the metadata server. It does **not** give namespace or pod, and `host.id` is the **node**: pods on one node collide.
- Metric points are **rejected** without a `location` label (from `cloud.region` / `cloud.availability_zone`) and an `instance` label (from `service.instance.id`). So always add `service.instance.id`: `serviceInstanceIdDetector` (random) and, on GKE, the pod name from the manifest.
- Manifest (downward API). `POD_NAME` / `POD_NAMESPACE` must be listed **before** `OTEL_RESOURCE_ATTRIBUTES` for `$(VAR)` expansion to work:

  ```yaml
  - name: POD_NAME
    valueFrom: { fieldRef: { fieldPath: metadata.name } }
  - name: POD_NAMESPACE
    valueFrom: { fieldRef: { fieldPath: metadata.namespace } }
  - name: OTEL_RESOURCE_ATTRIBUTES
    value: gcp.project_id=<project-id>,k8s.namespace.name=$(POD_NAMESPACE),k8s.pod.name=$(POD_NAME),service.instance.id=$(POD_NAME)
  - name: OTEL_SERVICE_NAME
    value: <service-name>
  ```

- `deployment.environment.name` reached Cloud Trace but was **not** usable as a metric filter. Split envs in dashboards by `namespace` (or project).

## Metric landing

- OTLP metrics land as `prometheus.googleapis.com/<name>/<kind>` on the `prometheus_target` resource and are queried with PromQL, e.g. `http_server_request_duration_seconds_bucket{http_route=…}`. Confirm exact names in Metrics Explorer before writing a dashboard.
- Cumulative and delta are both accepted for sums and histograms, but **delta UpDownCounters are rejected**: keep the SDK default (cumulative).
- `instrumentation-runtime-node` and the mongodb pool metric both arrive with no extra config.

## IAM (Workload Identity)

Roles: **`roles/telemetry.writer`** (traces, metrics, logs via the Telemetry API) + **`roles/monitoring.metricWriter`**. Not `roles/cloudtrace.agent` (legacy Trace API). Possibly `roles/serviceusage.serviceUsageConsumer` on the quota project. APIs to enable: `telemetry.googleapis.com` (required), `cloudtrace.googleapis.com`, `monitoring.googleapis.com`.

Without the grant, export gets 403 and is dropped **silently** (diag logger prints nothing unless `OTEL_LOG_LEVEL` is set).

Per-env checklist. Steps 1–5 read-only; step 6 changes IAM, so run it after review. Paste IDs and role names, never key files or tokens.

```bash
PROJECT=<project-id>; CLUSTER=<cluster>; LOCATION=<zone-or-region>; NS=<namespace>; DEPLOY=<deployment>

# 1. Workload Identity on? (pool prints <project>.svc.id.goog; pools show GKE_METADATA)
gcloud container clusters describe "$CLUSTER" --location "$LOCATION" --project "$PROJECT" \
  --format='value(workloadIdentityConfig.workloadPool)'
gcloud container node-pools list --cluster "$CLUSTER" --location "$LOCATION" --project "$PROJECT" \
  --format='table(name,config.workloadMetadataConfig.mode,config.serviceAccount,config.oauthScopes.list())'

# 2. KSA of the deployment (empty = default)
KSA=$(kubectl -n "$NS" get deploy "$DEPLOY" -o jsonpath='{.spec.template.spec.serviceAccountName}'); echo "KSA=${KSA:-default}"

# 3. Bound GSA? Empty → direct principal:
#    principal://iam.googleapis.com/projects/<PROJECT_NUMBER>/locations/global/workloadIdentityPools/$PROJECT.svc.id.goog/subject/ns/$NS/sa/$KSA
kubectl -n "$NS" get sa "${KSA:-default}" -o jsonpath='{.metadata.annotations.iam\.gke\.io/gcp-service-account}'; echo

# 4. Current roles (set GSA first; an empty $GSA matches every SA — don't trust that output)
gcloud projects get-iam-policy "$PROJECT" --flatten='bindings[].members' \
  --filter="bindings.members:serviceAccount:$GSA" --format='value(bindings.role)'
gcloud projects get-iam-policy "$PROJECT" --flatten='bindings[].members' \
  --filter="bindings.members:$NS/sa/${KSA:-default}" --format='table(bindings.role,bindings.members)'

# 5. APIs
gcloud services list --enabled --project "$PROJECT" \
  --filter='config.name:(cloudtrace.googleapis.com OR monitoring.googleapis.com OR telemetry.googleapis.com)' \
  --format='value(config.name)'

# 6. Grant what's missing (MEMBER = serviceAccount:$GSA or the principal:// from step 3)
for ROLE in roles/telemetry.writer roles/monitoring.metricWriter; do
  gcloud projects add-iam-policy-binding "$PROJECT" --member="$MEMBER" --role="$ROLE" --condition=None
done
gcloud services enable telemetry.googleapis.com cloudtrace.googleapis.com monitoring.googleapis.com --project "$PROJECT"
```

No Workload Identity: pods use the node SA; its OAuth scopes must include `cloud-platform` (`trace.append` may not cover the Telemetry API).

Granting to the namespace's `default` KSA principal lets every pod in that namespace write telemetry. The roles are write-only, so this is usually acceptable; record the trade-off.

## Cost (2026 list prices)

- **Traces:** $0.20 per million spans; first 2.5 M spans/month free per billing account. `spans/month = requests × sample ratio × spans/request`. Exclude health probes (≈0.26 M requests/probe/pod/month at 10 s) and orphan background spans.
- **Metrics:** billed as Prometheus samples, $0.06/M (first tier). `samples/month ≈ series × non-empty buckets × 43,200` per pod at 60 s. `http.server.request.duration` at ~150 route×method×status series ≈ $3/pod/month; runtime + pool metrics negligible; outbound client duration ≈ $0.50/pod/month.
- Metrics dominate; traces are near zero at 10% prod sampling. Cut metrics with fewer buckets (a View) or by dropping `http.client.request.duration`.

## Dashboard

One JSON in the repo, applied by hand with `gcloud monitoring dashboards create --config-from-file=<file>` (`update` for an existing one). Variables: env on `namespace`, optional route regex on `http_route`. Charts that earned their place: top-10 p95 by route, all-routes table (p50/p95/p99, req/s, 5xx%), 5xx ratio, per-pod Mongo connections by `state`, event-loop delay p99, V8 heap used, outbound p95 by `server_address`. Change the JSON only when a metric is added or renamed.
