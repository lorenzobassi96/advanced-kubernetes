# Assignment Solution — Telemetry (lbassi)

This folder contains my solution to the [Telemetry assignment](../README.md#assignment-telemetry):
deploy the demo app, confirm Prometheus scrapes its metrics, and build a Grafana
dashboard that visualizes and correlates them.

## Metrics exposed by the app

From [assignment/app/app.py](../assignment/app/app.py) the app exports three metrics on `:8000/metrics`:

| Metric | Type | Meaning |
| --- | --- | --- |
| `demo_app_requests_total` | Counter | Total processed requests |
| `demo_app_request_latency_seconds` | Histogram | Per-request latency (exposes `_bucket`, `_sum`, `_count`) |
| `demo_app_memory_usage_mb` | Gauge | Current RSS memory in MB |

The pod is scraped automatically thanks to the `prometheus.io/scrape: "true"` and
`prometheus.io/port: "8000"` annotations set in the chart.

## 1. Deploy the app

```bash
helm install assignment-app oci://ghcr.io/unusualfor/demo-app -n assignment --create-namespace

# Verify it is running
kubectl get pods -n assignment -l app=demo-app
kubectl get svc  -n assignment demo-app
```

## 2. Confirm the metrics endpoint

```bash
CLUSTER_IP=$(kubectl get svc -n assignment demo-app -o jsonpath='{.spec.clusterIP}')
curl http://$CLUSTER_IP:8000/metrics | grep demo_app
```

You should see `demo_app_requests_total`, `demo_app_request_latency_seconds_*`
and `demo_app_memory_usage_mb`.

## 3. Confirm Prometheus scrapes the app

```bash
export PNODE_PORT=$(kubectl get -n default -o jsonpath="{.spec.ports[0].nodePort}" services prometheus-server)
export PNODE_IP=$(kubectl get nodes -o jsonpath="{.items[0].status.addresses[0].address}")

# The target must appear and be "up"
curl -s http://$PNODE_IP:$PNODE_PORT/api/v1/targets | jq '.data.activeTargets[] | select(.labels.namespace=="assignment") | {job:.labels.job, health:.health}'
```

Or open the UI: `http://$PNODE_IP:$PNODE_PORT/targets` and filter for *assignment*.

## 4. PromQL queries used in the dashboard

| Panel | PromQL |
| --- | --- |
| Request rate (req/s) | `rate(demo_app_requests_total[1m])` |
| Average latency (s) | `rate(demo_app_request_latency_seconds_sum[5m]) / rate(demo_app_request_latency_seconds_count[5m])` |
| p95 latency (s) | `histogram_quantile(0.95, rate(demo_app_request_latency_seconds_bucket[5m]))` |
| Memory usage (MB) | `demo_app_memory_usage_mb` |
| Correlation: rate vs latency | both `rate(demo_app_requests_total[1m])` and the average-latency query on the same panel |

## 5. Import the Grafana dashboard

The ready-made dashboard lives in [grafana-dashboard.json](grafana-dashboard.json). It has
four panels, and the last one puts request rate and latency together on the same
time axis to relate the two metrics (as required by the assignment).

1. Open Grafana → **Dashboards → New → Import**.
2. Upload [grafana-dashboard.json](grafana-dashboard.json) (or paste its contents).
3. When prompted, select your **Prometheus** data source.
4. Click **Import**.

To generate load while you watch the graphs, the app already produces one request
every ~2 s on its own, so the panels populate automatically after a couple of minutes.

## Clean up

```bash
helm uninstall assignment-app -n assignment
```
