# Grafana Dashboard Import Guide

## Issue with JSON Dashboards

The dashboard JSON files in the `dashboards/` folder have structural issues that prevent direct import into Grafana. Instead, use the manual dashboard creation method below.

## Solution: Create Dashboards Manually in Grafana

### Step 1: Access Grafana

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Open: http://localhost:3000
- Username: `admin`
- Password: `admin` (or get from secret)

### Step 2: Create Dashboard 1 - HPA Autoscaling

1. Click **"+"** → **"Dashboard"** → **"Add new panel"**
2. For each panel below, click **"Add panel"** and configure:

#### Panel 1: Current vs Desired Replicas

**Query:**
```promql
kube_horizontalpodautoscaler_status_current_replicas{namespace="tenant-clusters",horizontalpodautoscaler="tenant3-apiserver-hpa"}
```

**Legend:** `Current Replicas`

**Add second query:**
```promql
kube_horizontalpodautoscaler_status_desired_replicas{namespace="tenant-clusters",horizontalpodautoscaler="tenant3-apiserver-hpa"}
```

**Legend:** `Desired Replicas`

**Panel Settings:**
- Title: "Control Plane Replicas (Current vs Desired)"
- Visualization: Time series
- Y-axis: Count

#### Panel 2: API Server Request Rate

**Query:**
```promql
sum(rate(apiserver_request_total{job="tenant3-apiserver"}[5m]))
```

**Legend:** `Requests/sec`

**Panel Settings:**
- Title: "API Server Request Rate"
- Y-axis: ops (operations per second)

#### Panel 3: CPU Utilization

**Query:**
```promql
sum(rate(container_cpu_usage_seconds_total{namespace="tenant-clusters",pod=~"tenant3-.*"}[5m])) / sum(kube_pod_container_resource_requests{namespace="tenant-clusters",pod=~"tenant3-.*",resource="cpu"}) * 100
```

**Legend:** `CPU %`

**Panel Settings:**
- Title: "CPU Utilization (%)"
- Y-axis: Percent (0-100)

#### Panel 4: Memory Utilization

**Query:**
```promql
sum(container_memory_working_set_bytes{namespace="tenant-clusters",pod=~"tenant3-.*"}) / sum(kube_pod_container_resource_requests{namespace="tenant-clusters",pod=~"tenant3-.*",resource="memory"}) * 100
```

**Legend:** `Memory %`

**Panel Settings:**
- Title: "Memory Utilization (%)"
- Y-axis: Percent (0-100)

#### Panel 5: API Server Latency (p95)

**Query:**
```promql
histogram_quantile(0.95, sum(rate(apiserver_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (verb, le))
```

**Legend:** `{{verb}} - p95`

**Panel Settings:**
- Title: "API Server Latency (p95)"
- Y-axis: seconds

3. Click **"Save dashboard"** → Name: "Kamaji TCP Autoscaling"

### Step 3: Create Dashboard 2 - Datastore Latency

1. Click **"+"** → **"Dashboard"** → **"Add new panel"**

#### Panel 1: API Server → Datastore Roundtrip (p95)

**Query:**
```promql
histogram_quantile(0.95, sum(rate(apiserver_storage_operation_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, resource, le))
```

**Legend:** `{{operation}}/{{resource}}`

**Panel Settings:**
- Title: "API Server → Datastore Roundtrip Latency (p95)"
- Y-axis: seconds

#### Panel 2: Etcd Operation Latency (p95)

**Query:**
```promql
histogram_quantile(0.95, sum(rate(etcd_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, type, le))
```

**Legend:** `{{operation}}/{{type}}`

**Panel Settings:**
- Title: "Etcd Operation Latency (p95)"
- Y-axis: seconds

#### Panel 3: Storage Operation Rate

**Query:**
```promql
sum(rate(apiserver_storage_operation_duration_seconds_count{job="tenant3-apiserver"}[5m])) by (operation, resource)
```

**Legend:** `{{operation}}/{{resource}}`

**Panel Settings:**
- Title: "Storage Operation Rate"
- Y-axis: ops

#### Panel 4: Storage Errors

**Query:**
```promql
sum(rate(apiserver_storage_operation_errors_total{job="tenant3-apiserver"}[5m])) by (operation, resource)
```

**Legend:** `{{operation}}/{{resource}}`

**Panel Settings:**
- Title: "Storage Operation Errors"
- Y-axis: errors/sec

2. Click **"Save dashboard"** → Name: "Kamaji Datastore Latency"

## Verify Metrics Are Available

Before creating dashboards, verify metrics in Prometheus:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Open: http://localhost:9090

Test these queries:

1. **HPA Metrics:**
   ```promql
   kube_horizontalpodautoscaler_status_current_replicas{horizontalpodautoscaler="tenant3-apiserver-hpa"}
   ```
   Should return: Current replica count (3-5)

2. **API Server Metrics:**
   ```promql
   apiserver_request_total{job="tenant3-apiserver"}
   ```
   Should return: Request counters

3. **Datastore Metrics:**
   ```promql
   apiserver_storage_operation_duration_seconds_bucket{job="tenant3-apiserver"}
   ```
   Should return: Storage operation latency buckets

If any query returns "No data", check:

1. **ServiceMonitor exists:**
   ```bash
   kubectl get servicemonitor tenant3-apiserver -n tenant-clusters
   ```

2. **Prometheus is scraping:**
   - In Prometheus UI: Status → Targets
   - Search: tenant3-apiserver
   - Status should be: UP

3. **Service has correct labels:**
   ```bash
   kubectl get svc tenant3 -n tenant-clusters -o yaml | grep -A 5 "labels:"
   ```
   Should have: `monitoring: enabled`

## Troubleshooting

### "No data" in panels

**Check 1: Verify job label**
```promql
# Run in Prometheus
apiserver_request_total
```
Look at the `job` label value. If it's different from "tenant3-apiserver", update all queries to use the correct job name.

**Check 2: Verify namespace**
```bash
kubectl get tcp -A
```
If tenant control plane is in a different namespace, update queries.

**Check 3: Verify HPA name**
```bash
kubectl get hpa -A
```
If HPA has a different name, update queries.

### Metrics exist but panels show errors

**Issue**: Query syntax error

**Solution**: Copy queries exactly as shown above. Common mistakes:
- Missing quotes around label values
- Wrong metric names (check spelling)
- Missing `by` clause in histogram_quantile

### Dashboard doesn't refresh

**Solution**: Set refresh interval
- Dashboard settings (gear icon) → Time options
- Refresh: 10s or 30s

## Quick Reference

**All Prometheus Queries for Copy-Paste:**

```promql
# HPA Current Replicas
kube_horizontalpodautoscaler_status_current_replicas{namespace="tenant-clusters",horizontalpodautoscaler="tenant3-apiserver-hpa"}

# HPA Desired Replicas
kube_horizontalpodautoscaler_status_desired_replicas{namespace="tenant-clusters",horizontalpodautoscaler="tenant3-apiserver-hpa"}

# API Server Request Rate
sum(rate(apiserver_request_total{job="tenant3-apiserver"}[5m]))

# API Server Latency p95
histogram_quantile(0.95, sum(rate(apiserver_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (verb, le))

# Datastore Roundtrip p95
histogram_quantile(0.95, sum(rate(apiserver_storage_operation_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, resource, le))

# Etcd Latency p95
histogram_quantile(0.95, sum(rate(etcd_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, type, le))

# Storage Operation Rate
sum(rate(apiserver_storage_operation_duration_seconds_count{job="tenant3-apiserver"}[5m])) by (operation, resource)

# CPU Utilization
sum(rate(container_cpu_usage_seconds_total{namespace="tenant-clusters",pod=~"tenant3-.*"}[5m])) / sum(kube_pod_container_resource_requests{namespace="tenant-clusters",pod=~"tenant3-.*",resource="cpu"}) * 100

# Memory Utilization
sum(container_memory_working_set_bytes{namespace="tenant-clusters",pod=~"tenant3-.*"}) / sum(kube_pod_container_resource_requests{namespace="tenant-clusters",pod=~"tenant3-.*",resource="memory"}) * 100
```

Save this file for reference when creating dashboards!
