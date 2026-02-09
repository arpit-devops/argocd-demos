# Kamaji Observability Guide

This guide explains how to set up complete observability for Kamaji tenant control planes, including autoscaling metrics, API server performance, and datastore latency monitoring.

## Overview

We'll set up monitoring for three key areas:

1. **HPA Autoscaling Observability** - Monitor HPA decisions and replica scaling
2. **API Server Observability** - Track request rates, latency, and performance
3. **Datastore Observability** - Monitor etcd/Kine latency and roundtrip times

## Prerequisites

- Kamaji installed with TenantControlPlane running
- kube-prometheus-stack installed (Prometheus + Grafana)
- kubectl access to your cluster

## Part 1: Enable Metrics Collection

### Step 1: Create ServiceMonitor

ServiceMonitor tells Prometheus to scrape metrics from the tenant control plane API server.

```bash
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: tenant3-apiserver
  namespace: tenant-clusters
  labels:
    app: tenant3-control-plane
spec:
  selector:
    matchLabels:
      app: tenant3-control-plane
      monitoring: enabled
  endpoints:
    - port: api
      interval: 30s
      scheme: https
      tlsConfig:
        insecureSkipVerify: true
      metricRelabelings:
        - action: keep
          regex: 'apiserver_request_total|apiserver_request_duration_seconds.*|apiserver_current_inflight_requests|process_cpu_seconds_total|process_resident_memory_bytes|go_goroutines|etcd_request_duration_seconds.*|apiserver_storage_.*|kine_.*'
          sourceLabels:
            - __name__
EOF
```

**What this does:**
- Scrapes metrics every 30 seconds from the API server
- Collects API server metrics (requests, latency, inflight)
- Collects datastore metrics (etcd, storage operations, Kine)
- Uses insecure TLS (for internal cluster communication)

### Step 2: Verify Prometheus is Scraping

```bash
# Port-forward to Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &

# Open Prometheus UI
# Visit: http://localhost:3000

# Check targets
# Go to: Status → Targets
# Search for: tenant3-apiserver
# Status should be: UP (green)
```

### Step 3: Test Metrics Availability

Run these queries in Prometheus to verify metrics are available:

**API Server Metrics:**
```promql
# Request rate
rate(apiserver_request_total{job="tenant3-apiserver"}[5m])

# Request latency (p95)
histogram_quantile(0.95, sum(rate(apiserver_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (verb, le))

# Current inflight requests
apiserver_current_inflight_requests{job="tenant3-apiserver"}
```

**Datastore Metrics:**
```promql
# Etcd operation latency (p95)
histogram_quantile(0.95, sum(rate(etcd_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, type, le))

# Storage operation latency (p95)
histogram_quantile(0.95, sum(rate(apiserver_storage_operation_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, resource, le))

# Storage operation rate
sum(rate(apiserver_storage_operation_duration_seconds_count{job="tenant3-apiserver"}[5m])) by (operation, resource)
```

**HPA Metrics:**
```promql
# Current replicas
kube_horizontalpodautoscaler_status_current_replicas{horizontalpodautoscaler="tenant3-apiserver-hpa"}

# Desired replicas
kube_horizontalpodautoscaler_status_desired_replicas{horizontalpodautoscaler="tenant3-apiserver-hpa"}

# CPU utilization
kube_horizontalpodautoscaler_status_current_metrics_average_utilization{horizontalpodautoscaler="tenant3-apiserver-hpa", metric_name="cpu"}
```

If these queries return data, Prometheus is successfully scraping metrics! ✅

## Part 2: Import Grafana Dashboards

### Step 1: Access Grafana

```bash
# Port-forward to Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80 &

# Get Grafana admin password
kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d
echo

# Open Grafana UI
# Visit: http://localhost:3000
# Username: admin
# Password: (from command above)
```

### Step 2: Import Dashboard 1 - HPA Autoscaling Dashboard

This dashboard shows HPA behavior and API server performance.

**Import Steps:**

1. In Grafana, click **"+"** → **"Import"**
2. Click **"Upload JSON file"**
3. Select: `dashboards/grafana-dashboard.json`
4. **Datasource**: Select **Prometheus** (default)
5. Click **"Import"**

**Dashboard Panels:**

- **Current Replicas** - Real-time replica count
- **Desired Replicas** - HPA target replica count
- **API Server Request Rate** - Requests per second
- **API Server Latency (p95)** - 95th percentile latency
- **CPU Utilization** - Current vs target CPU
- **Memory Utilization** - Current vs target memory
- **Inflight Requests** - Active API requests
- **Go Routines** - API server goroutine count

**Expected Data:**
- You should see replica count (3-5)
- Request rate will show spikes during load tests
- CPU/Memory will show percentage values
- Latency should be <100ms for healthy API server

### Step 3: Import Dashboard 2 - Datastore Latency Dashboard

This dashboard monitors API server → datastore roundtrip latency and etcd performance.

**Import Steps:**

1. Click **"+"** → **"Import"**
2. Click **"Upload JSON file"**
3. Select: `dashboards/datastore-latency-dashboard.json`
4. **Datasource**: Select **Prometheus**
5. Click **"Import"**

**Dashboard Panels:**

- **Panel #1: API Server → Datastore Roundtrip Latency (p95)** - Total time for storage operations
- **Panel #2: API Server → Datastore Roundtrip Latency (p99)** - 99th percentile
- **Panel #3: Etcd Operation Latency (p95)** - Direct etcd operation time
- **Panel #4: Etcd Operation Latency (p99)** - 99th percentile
- **Panel #5-8: Storage Operation Latency by Type** - GET, LIST, CREATE, UPDATE operations
- **Panel #9: Etcd Request Rate** - Operations per second
- **Panel #10: Storage Operations Rate** - By operation type
- **Panel #11: Storage Operation Errors** - Error count
- **Panel #12: Average Roundtrip Time** - Single stat panel
- **Panel #13: Total Operations/sec** - Single stat panel

**Expected Data:**
- Roundtrip latency: 5-20ms (healthy)
- Etcd latency: 2-10ms (healthy)
- GET operations: fastest (5-10ms)
- LIST operations: slower (10-50ms)
- Error rate: should be 0 or very low

**What is "Roundtrip Latency"?**

Roundtrip latency measures the total time for an API server storage operation:
1. API server serializes object
2. Network transmission to etcd
3. Etcd processes request
4. Network transmission back
5. API server deserializes response

This is the most important metric for understanding end-to-end datastore performance.

### Step 4: Import Dashboard 3 - Kine Dashboard (Optional)

Only needed if using Kine (PostgreSQL/MySQL/SQLite) instead of etcd.

**Import Steps:**

1. Click **"+"** → **"Import"**
2. Click **"Upload JSON file"**
3. Select: `dashboards/kine-dashboard.json`
4. **Datasource**: Select **Prometheus**
5. Click **"Import"**

**Dashboard Panels:**

- **SQL Query Latency (p95/p99)** - Database query performance
- **Connection Pool Usage** - Active vs idle connections
- **Query Rate by Type** - SELECT, INSERT, UPDATE, DELETE
- **Transaction Latency** - Transaction commit time
- **Database Size** - Storage usage
- **Query Errors** - Failed queries

**Note**: If using etcd (default), this dashboard will show "No Data" - this is expected.

## Part 3: Verify All Metrics Are Working

### Verification Checklist

Run these commands to verify each metric type:

#### 1. HPA Metrics ✅

```bash
# Check HPA status
kubectl get hpa tenant3-apiserver-hpa -n tenant-clusters

# Should show:
# TARGETS: 25%/60%, 30%/70%  (not <unknown>)
# REPLICAS: 3-5
```

**Prometheus Query:**
```promql
kube_horizontalpodautoscaler_status_current_replicas{horizontalpodautoscaler="tenant3-apiserver-hpa"}
```

**Expected**: Returns current replica count (3-5)

#### 2. API Server Request Metrics ✅

```bash
# Generate some API requests
kubectl --kubeconfig=/path/to/tenant3-admin-kubeconfig get nodes
kubectl --kubeconfig=/path/to/tenant3-admin-kubeconfig get pods -A
```

**Prometheus Query:**
```promql
rate(apiserver_request_total{job="tenant3-apiserver"}[5m])
```

**Expected**: Returns request rate (e.g., 0.5-10 req/sec)

#### 3. API Server Latency Metrics ✅

**Prometheus Query:**
```promql
histogram_quantile(0.95, sum(rate(apiserver_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (verb, le))
```

**Expected**: Returns latency values (e.g., 0.01-0.1 seconds)

#### 4. Etcd Latency Metrics ✅

**Prometheus Query:**
```promql
histogram_quantile(0.95, sum(rate(etcd_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, type, le))
```

**Expected**: Returns etcd operation latency (e.g., 0.002-0.01 seconds)

#### 5. Storage Operation Metrics ✅

**Prometheus Query:**
```promql
histogram_quantile(0.95, sum(rate(apiserver_storage_operation_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, resource, le))
```

**Expected**: Returns storage operation latency by type (GET, LIST, CREATE, UPDATE)

#### 6. Storage Operation Rate ✅

**Prometheus Query:**
```promql
sum(rate(apiserver_storage_operation_duration_seconds_count{job="tenant3-apiserver"}[5m])) by (operation, resource)
```

**Expected**: Returns operations per second by type

### Troubleshooting "No Data" in Dashboards

If dashboards show "No Data":

**Problem 1: ServiceMonitor not created**

```bash
# Check if ServiceMonitor exists
kubectl get servicemonitor tenant3-apiserver -n tenant-clusters

# If not found, create it (see Step 1)
```

**Problem 2: Prometheus not scraping**

```bash
# Check Prometheus targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &

# Visit: http://localhost:9090/targets
# Search for: tenant3-apiserver
# Status should be: UP

# If DOWN, check:
# 1. Service has correct labels (app: tenant3-control-plane, monitoring: enabled)
# 2. ServiceMonitor selector matches service labels
# 3. Port name is "api" in the service
```

**Problem 3: Metrics not exposed**

```bash
# Check if API server is exposing metrics
kubectl port-forward -n tenant-clusters svc/tenant3 6443:6443 &

# Try to access metrics endpoint (will fail due to auth, but should connect)
curl -k https://localhost:6443/metrics

# Should return: "Unauthorized" (not connection refused)
```

**Problem 4: Wrong job label**

```bash
# Check what job label Prometheus is using
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &

# In Prometheus UI, run:
apiserver_request_total

# Check the "job" label value
# Update dashboard queries if job name is different from "tenant3-apiserver"
```

**Problem 5: Dashboard JSON syntax error**

```bash
# Validate JSON files
cat dashboards/grafana-dashboard.json | jq .
cat dashboards/datastore-latency-dashboard.json | jq .
cat dashboards/kine-dashboard.json | jq .

# If errors, the JSON is malformed
```

## Part 4: Understanding the Metrics

### HPA Autoscaling Metrics

**Key Metrics:**
- `kube_horizontalpodautoscaler_status_current_replicas` - Current number of pods
- `kube_horizontalpodautoscaler_status_desired_replicas` - Target number of pods
- `kube_horizontalpodautoscaler_status_current_metrics_average_utilization` - Current CPU/memory %

**What to Watch:**
- **Gap between current and desired** - Indicates scaling in progress
- **Frequent scaling** - May need to adjust thresholds or stabilization windows
- **No scaling despite high CPU** - Check HPA target is TenantControlPlane (not Deployment)

### API Server Performance Metrics

**Key Metrics:**
- `apiserver_request_total` - Total requests (counter)
- `apiserver_request_duration_seconds` - Request latency (histogram)
- `apiserver_current_inflight_requests` - Active requests (gauge)

**Healthy Values:**
- Request rate: 1-100 req/sec (depends on workload)
- Latency p95: <100ms
- Latency p99: <500ms
- Inflight requests: <100

**Warning Signs:**
- Latency p95 >500ms - API server overloaded
- Inflight requests >500 - Too many concurrent requests
- High error rate - Check API server logs

### Datastore Latency Metrics

**Key Metrics:**
- `etcd_request_duration_seconds` - Direct etcd operation time
- `apiserver_storage_operation_duration_seconds` - Full roundtrip time (API server → etcd → API server)

**Healthy Values:**
- Etcd latency p95: <10ms
- Roundtrip latency p95: <20ms
- GET operations: 5-10ms
- LIST operations: 10-50ms

**Warning Signs:**
- Etcd latency >50ms - Etcd overloaded or network issues
- Roundtrip >100ms - Serialization overhead or network latency
- High error rate - Etcd connectivity issues

**Difference between Etcd and Roundtrip:**
- **Etcd latency**: Time etcd takes to process the request
- **Roundtrip latency**: Etcd latency + serialization + network + deserialization

## Part 5: Setting Up Alerts (Optional)

Create Prometheus alerts for critical conditions:

```bash
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: tenant3-alerts
  namespace: tenant-clusters
spec:
  groups:
    - name: tenant3-apiserver
      interval: 30s
      rules:
        - alert: HighAPIServerLatency
          expr: histogram_quantile(0.95, sum(rate(apiserver_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (verb, le)) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "API server latency is high"
            description: "API server p95 latency is {{ \$value }}s (threshold: 0.5s)"
        
        - alert: HighEtcdLatency
          expr: histogram_quantile(0.95, sum(rate(etcd_request_duration_seconds_bucket{job="tenant3-apiserver"}[5m])) by (operation, type, le)) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Etcd latency is high"
            description: "Etcd p95 latency is {{ \$value }}s (threshold: 0.05s)"
        
        - alert: HPAMaxedOut
          expr: kube_horizontalpodautoscaler_status_current_replicas{horizontalpodautoscaler="tenant3-apiserver-hpa"} >= kube_horizontalpodautoscaler_spec_max_replicas{horizontalpodautoscaler="tenant3-apiserver-hpa"}
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "HPA has reached maximum replicas"
            description: "HPA is at max replicas ({{ \$value }}), consider increasing maxReplicas"
EOF
```

## Summary

You now have complete observability for:

✅ **HPA Autoscaling**
- Current vs desired replicas
- CPU and memory utilization
- Scaling events and timeline

✅ **API Server Performance**
- Request rate and latency
- Inflight requests
- Error rates

✅ **Datastore Latency**
- API server → datastore roundtrip time
- Etcd operation latency
- Storage operation breakdown (GET, LIST, CREATE, UPDATE)
- Operation rates and errors

✅ **Kine Backend** (if applicable)
- SQL query latency
- Connection pool usage
- Transaction performance

All metrics are collected by Prometheus and visualized in Grafana dashboards!

## Quick Reference

**Access Grafana:**
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# Visit: http://localhost:3000 (admin/admin)
```

**Access Prometheus:**
```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
# Visit: http://localhost:9090
```

**Check ServiceMonitor:**
```bash
kubectl get servicemonitor tenant3-apiserver -n tenant-clusters
```

**Verify metrics:**
```bash
# In Prometheus, run:
apiserver_request_total{job="tenant3-apiserver"}
```

**Dashboard locations:**
- `dashboards/grafana-dashboard.json` - HPA autoscaling
- `dashboards/datastore-latency-dashboard.json` - Datastore performance
- `dashboards/kine-dashboard.json` - Kine SQL backend
