# Kamaji Grafana Dashboards

This directory contains Grafana dashboards for monitoring Kamaji TCP control planes.

## Available Dashboards

### 1. grafana-dashboard.json
**Main Autoscaling & Performance Dashboard**

Monitors:
- HPA status (current vs desired replicas)
- API server request rate and latency
- CPU and memory utilization
- Inflight requests
- Basic datastore metrics

**Use case**: Overall control plane health and autoscaling visualization

---

### 2. datastore-latency-dashboard.json
**Dedicated Datastore Latency Dashboard**

Focuses on API Server ↔ Datastore performance:
- **API Server → Datastore Roundtrip Latency** (p95, p99)
- **Etcd Operation Latency** (p95, p99)
- **Storage Operation Latency by Type** (GET, LIST, CREATE, UPDATE)
- **Operation Rates** (ops/sec)
- **Storage Errors**
- **Average Roundtrip Time** (stat panel)

**Use case**: Deep dive into datastore performance, troubleshooting slow queries

**Key Metrics:**
- Roundtrip latency includes: serialization + network + etcd processing + deserialization
- Etcd latency is direct etcd operation time
- Healthy: <20ms (p95), <50ms (p99)
- Warning: 50-100ms
- Critical: >100ms

---

### 3. kine-dashboard.json
**Kine Datastore Dashboard** (for non-etcd backends)

Monitors Kine-specific metrics when using:
- SQLite
- PostgreSQL
- MySQL
- Other SQL backends

**Panels:**
- **Kine SQL Query Latency** (p95, p99)
- **Connection Pool Usage** (in-use, idle, max)
- **Query Rate by Operation**
- **Transaction Latency**
- **Database Size**
- **Query Errors**
- **Backend Type** (shows which SQL backend is in use)
- **Connection Wait Time**
- **Storage Backend Roundtrip** (via API server metrics)

**Use case**: Monitor Kine performance when not using etcd

**Note**: Kine metrics may not be available if using native etcd. Check backend type first.

---

## How to Import Dashboards

### Method 1: Grafana UI

1. Access Grafana:
   ```bash
   kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
   ```
   Open: http://localhost:3000 (admin/admin)

2. Import dashboard:
   - Click "+" → "Import"
   - Click "Upload JSON file"
   - Select dashboard file
   - Choose Prometheus datasource
   - Click "Import"

### Method 2: ConfigMap (GitOps)

Create a ConfigMap with the dashboard:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: kamaji-dashboards
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  datastore-latency.json: |
    <paste dashboard JSON here>
```

Grafana sidecar will automatically load it.

---

## Dashboard Selection Guide

**I want to see autoscaling in action:**
→ Use `grafana-dashboard.json`

**API server is slow, need to troubleshoot:**
→ Use `datastore-latency-dashboard.json`

**Using Kine with PostgreSQL/MySQL/SQLite:**
→ Use `kine-dashboard.json`

**Want to see roundtrip latency specifically:**
→ Use `datastore-latency-dashboard.json` - Panel #1 & #2

**Need to check if datastore is the bottleneck:**
→ Compare panels in `datastore-latency-dashboard.json`:
  - If etcd latency is low but roundtrip is high → serialization/network issue
  - If both are high → datastore is slow

---

## Prometheus Queries Reference

### API Server → Datastore Roundtrip (p95)
```promql
histogram_quantile(0.95, sum(rate(apiserver_storage_operation_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, resource, le))
```

### Etcd Operation Latency (p95)
```promql
histogram_quantile(0.95, sum(rate(etcd_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, type, le))
```

### Kine Query Latency (p95)
```promql
histogram_quantile(0.95, sum(rate(kine_sql_query_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, le))
```

### Storage Operation Rate
```promql
sum(rate(apiserver_storage_operation_duration_seconds_count{job="tenant3-apiserver"}[5m])) by (operation, resource)
```

---

## Troubleshooting

### Dashboard shows "No data"

**Check ServiceMonitor:**
```bash
kubectl get servicemonitor tenant3-apiserver -n tenant-clusters
kubectl describe servicemonitor tenant3-apiserver -n tenant-clusters
```

**Check Prometheus targets:**
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```
Visit: http://localhost:9090/targets
Look for "tenant3-apiserver" - should be UP

**Check if metrics are exposed:**
```bash
kubectl port-forward -n tenant-clusters svc/tenant3 6443:6443
curl -k https://localhost:6443/metrics | grep -E "apiserver_storage|etcd_request"
```

### Kine metrics not showing

Kine metrics are only available when using Kine as the datastore backend (not native etcd).

**Check datastore type:**
```bash
kubectl get tcp tenant3 -n tenant-clusters -o yaml | grep -A 5 dataStore
```

If using default etcd, use `datastore-latency-dashboard.json` instead.

### High latency alerts

**If roundtrip latency > 100ms:**
1. Check etcd latency - if also high, datastore is slow
2. Check network latency between API server and datastore
3. Check datastore resource usage (CPU, memory, disk I/O)
4. Consider scaling datastore or optimizing queries

**If only specific operations are slow:**
- LIST operations are naturally slower (fetching multiple objects)
- CREATE/UPDATE may be slow if datastore is under heavy write load
- GET should be fast (<10ms typically)

---

## Customization

All dashboards use the job label `tenant3-apiserver`. To monitor a different tenant:

1. Edit dashboard JSON
2. Find/replace: `job="tenant3-apiserver"` → `job="your-tenant-apiserver"`
3. Re-import dashboard

Or use Grafana variables for multi-tenant monitoring.
