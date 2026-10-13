# OpenTelemetry Demo with Splunk OTel Collector on GKE

This setup installs both releases in the `otel-demo` namespace. The Splunk
chart runs its normal node-agent DaemonSet and cluster receiver, plus a
two-replica gateway. Every Demo SDK exports OTLP/gRPC or OTLP/HTTP to the
gateway service. The Demo chart's bundled Collector, Jaeger, Prometheus,
Grafana, and OpenSearch are disabled. SDK log export is also disabled because
this configuration targets Splunk Observability Cloud for traces and metrics;
the `/v3/event` entity endpoint is not a general application-log destination.
The gateway's default `logs/entities` route is overridden with a local `nop`
exporter as a second layer of protection, so unexpected OTLP logs are accepted
and discarded rather than filling a remote exporter retry queue.

## 1. Select the GKE cluster

```bash
gcloud container clusters get-credentials "$GKE_CLUSTER" \
  --location "$GKE_LOCATION" \
  --project "$GCP_PROJECT"

kubectl config current-context
kubectl create namespace otel-demo --dry-run=client -o yaml | kubectl apply -f -
```

For GKE Autopilot, change `distribution: gke` to
`distribution: gke/autopilot` in `splunk-otel-values.yaml`.

## 2. Edit the non-secret settings

In `splunk-otel-values.yaml`, replace:

- `REPLACE_WITH_GKE_CLUSTER_NAME`
- `REPLACE_WITH_SPLUNK_REALM` (for example, `us0` or `eu0`)

## 3. Create the Splunk token Secret

Set the token only in your shell; do not add it to a values file or Git.

```bash
read -s -p "Splunk Observability access token: " SPLUNK_ACCESS_TOKEN
echo

kubectl -n otel-demo create secret generic splunk-otel-credentials \
  --from-literal=splunk_observability_access_token="$SPLUNK_ACCESS_TOKEN" \
  --dry-run=client -o yaml | kubectl apply -f -

unset SPLUNK_ACCESS_TOKEN
```

## 4. Install the Splunk Collector first

Run these commands from the directory containing these files:

```bash
helm repo add splunk-otel-collector-chart \
  https://signalfx.github.io/splunk-otel-collector-chart
helm repo add open-telemetry \
  https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm upgrade --install splunk-otel \
  splunk-otel-collector-chart/splunk-otel-collector \
  --version 0.161.0 \
  --namespace otel-demo \
  --values splunk-otel-values.yaml \
  --wait --timeout 10m

kubectl -n otel-demo get service splunk-otel-collector-gateway
kubectl -n otel-demo rollout status deployment/splunk-otel-collector-gateway \
  --timeout=5m
```

## 5. Install the Demo

```bash
helm upgrade --install otel-demo \
  open-telemetry/opentelemetry-demo \
  --version 0.42.0 \
  --namespace otel-demo \
  --values otel-demo-values.yaml \
  --wait --timeout 15m
```

If the Demo was already installed without `OTEL_LOGS_EXPORTER=none`, running
the command above rolls out the corrected environment setting and stops HTTP
429 retries from the gateway's `otlp_http/entities` exporter.

## 6. Verify telemetry routing and open the UI

```bash
# There should be no Demo-owned otel-collector workload/service.
kubectl -n otel-demo get pods,services | grep -E 'splunk-otel|otel-collector'

# Confirm a Demo deployment received the new collector hostname.
kubectl -n otel-demo get deployment frontend -o yaml | \
  grep -A2 OTEL_COLLECTOR_NAME

# Check for receiver/export errors.
kubectl -n otel-demo logs deployment/splunk-otel-collector-gateway \
  --since=10m --all-containers=true | grep -Ei 'error|failed|refused' || true

# Confirm SDK log exporting is disabled for the Demo workloads.
kubectl -n otel-demo get deployment frontend -o yaml | \
  grep -A2 OTEL_LOGS_EXPORTER

# Open the Demo frontend at http://localhost:8080
kubectl -n otel-demo port-forward service/frontend-proxy 8080:8080
```

In Splunk Observability Cloud, look for services such as `frontend`,
`checkout`, and `product-catalog`, and filter by the configured
`k8s.cluster.name`.

## Upgrade or remove

Repeat the two `helm upgrade --install` commands to apply changes. To remove
the demo and collectors:

```bash
helm uninstall otel-demo -n otel-demo
helm uninstall splunk-otel -n otel-demo
kubectl delete namespace otel-demo
```

Deleting the namespace also deletes the token Secret and all namespaced Demo
data. It does not delete the GKE cluster or Splunk data already ingested.
