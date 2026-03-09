# 📊 Monitoring — Prometheus & Grafana

## The Observability Pillars

```
Metrics  → Prometheus (what's happening numerically)
Logs     → Loki / ELK  (what happened textually)
Traces   → Jaeger / Tempo (where time is spent)
```

---

## Prometheus

### Core Concepts
| Concept | Description |
|---------|-------------|
| **Metric types** | Counter, Gauge, Histogram, Summary |
| **Scrape** | Prometheus pulls metrics from targets |
| **Exporter** | Exposes metrics in Prometheus format |
| **PromQL** | Query language for metrics |
| **Alert rule** | Fires when a PromQL condition is true for a duration |

### Metric Types
```
Counter   → always increases (requests_total, errors_total)
Gauge     → goes up/down (memory_usage, active_connections)
Histogram → bucketed observations (request_duration_seconds)
Summary   → like histogram, calculates quantiles client-side
```

### PromQL Cheatsheet
```promql
# Instant query
http_requests_total

# Filter by label
http_requests_total{job="api", status="500"}

# Rate (per-second avg over 5 min window)
rate(http_requests_total[5m])

# Error rate %
rate(http_requests_total{status=~"5.."}[5m]) /
rate(http_requests_total[5m]) * 100

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod)

# Memory per namespace
sum(container_memory_working_set_bytes) by (namespace)

# Predict disk full (4 hours)
predict_linear(node_filesystem_free_bytes[1h], 4*3600)
```

---

## Alert Rules
```yaml
# alerts.yml
groups:
  - name: app.rules
    rules:
      - alert: HighErrorRate
        expr: |
          rate(http_requests_total{status=~"5.."}[5m]) /
          rate(http_requests_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.job }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: PodCrashLooping
        expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} is crash looping"

      - alert: DiskSpaceLow
        expr: |
          (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk space < 15% on {{ $labels.instance }}"
```

---

## Grafana

### Dashboard-as-Code with Grafonnet/JSON
Key dashboard panels to always include:
- **Request rate** — `rate(http_requests_total[5m])`
- **Error rate** — filter 5xx, show as %
- **Latency p50/p95/p99** — histogram_quantile
- **Pod restarts** — kube_pod_container_status_restarts_total
- **CPU/Memory** — node_cpu / container_memory

---

## 🛠️ Hands-On Project: Full K8s Monitoring Stack

**What you'll deploy:** Prometheus + Grafana + Alertmanager on Kubernetes via Helm

```bash
# Add helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (all-in-one)
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values values.yaml

# Access Grafana
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
# Default: admin / prom-operator
```

**Sample `values.yaml`:**
```yaml
grafana:
  adminPassword: "changeme123"
  ingress:
    enabled: true
    hosts:
      - grafana.example.com

prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          resources:
            requests:
              storage: 50Gi

alertmanager:
  config:
    route:
      receiver: slack-alerts
    receivers:
      - name: slack-alerts
        slack_configs:
          - api_url: "${SLACK_WEBHOOK}"
            channel: "#alerts"
            title: "{{ .CommonAnnotations.summary }}"
```

📁 See [`examples/`](./examples/) for complete setup.

---

## 🎯 Key Interview Questions

1. **Difference between a Counter and a Gauge?**  
   Counter only increases (use `rate()` to get per-second change). Gauge goes up and down (query directly).

2. **What is the `rate()` function in PromQL?**  
   Calculates per-second average increase of a counter over a time window. Always use `rate()` not `irate()` for alerting.

3. **What is the `for` clause in an alert rule?**  
   The alert must be true continuously for this duration before firing — prevents flapping alerts.

4. **How does Prometheus discover targets?**  
   Static config, file-based discovery, or service discovery (K8s, Consul, EC2). In K8s, it scrapes annotated pods/services.

5. **What's the RED method for monitoring services?**  
   Rate (requests/sec), Errors (error rate), Duration (latency). USE method is for resources: Utilization, Saturation, Errors.
