# Kamaji TCP Control Plane with Autoscaling

This setup configures a Kamaji Tenant Control Plane with horizontal autoscaling capabilities and comprehensive monitoring.

## Components

### 1. **tcp.yaml** - Tenant Control Plane
- Initial replicas: 3
- Resource requests/limits configured for HPA
- Profiling enabled for metrics collection
- Labels for ServiceMonitor discovery

### 2. **hpa.yaml** - Horizontal Pod Autoscaler
- **Min replicas**: 3
- **Max replicas**: 5
- **Triggers**:
  - CPU utilization: 60%
  - Memory utilization: 70%
- **Scale-up behavior**: Fast response (60s stabilization)
- **Scale-down behavior**: Conservative (300s stabilization)

### 3. **servicemonitor.yaml** - Prometheus Monitoring
- Scrapes API server metrics every 30s
- Captures key metrics:
  - `apiserver_request_total`
  - `apiserver_request_duration_seconds`
  - `apiserver_current_inflight_requests`
  - CPU, memory, and goroutine metrics

### 4. **load-test-job.yaml** - Load Generator
- Creates artificial load on the API server
- Runs 5 parallel pods for 10 minutes
- Generates continuous GET requests (nodes, pods, services)
- Triggers HPA scale-up from 3 to 5 replicas

### 5. **prometheus-queries.yaml** - Monitoring Queries
- Pre-configured PromQL queries for:
  - Request rates and latency
  - Resource utilization
  - HPA status (current vs desired replicas)
  - Inflight requests

### 6. **grafana-dashboard.json** - Visualization Dashboard
- Real-time autoscaling visualization
- Panels for:
  - Replica count (current vs desired)
  - API server request rate
  - CPU/Memory utilization
  - Request latency (p95)
  - Inflight requests

## Prerequisites

1. **Prometheus Operator** installed in the cluster
2. **Metrics Server** for HPA to function
3. **ServiceMonitor CRD** available

## Deployment Steps

### 1. Verify Prerequisites
```bash
# Check if Prometheus Operator is installed
kubectl get crd servicemonitors.monitoring.coreos.com

# Check if metrics-server is running
kubectl get deployment metrics-server -n kube-system

# Verify HPA can get metrics
kubectl get apiservice v1beta1.metrics.k8s.io
```

### 2. Deploy via ArgoCD (GitOps)
The manifests will be automatically synced by ArgoCD from the repository.

```bash
# Check ArgoCD application status
kubectl get application tenant1 -n argocd

# Force sync if needed
argocd app sync tenant1
```

### 3. Verify Deployment
```bash
# Check TCP control plane
kubectl get tcp tenant3 -n tenant-clusters

# Check HPA status
kubectl get hpa tenant3-apiserver-hpa -n tenant-clusters

# Check ServiceMonitor
kubectl get servicemonitor tenant3-apiserver -n tenant-clusters

# Verify initial replicas
kubectl get pods -n tenant-clusters -l app=tenant3-control-plane
```

### 4. Run Load Test
```bash
# Apply the load test job
kubectl apply -f load-test-job.yaml

# Watch the load test progress
kubectl logs -f job/apiserver-load-test -n tenant-clusters

# Monitor HPA scaling in real-time
watch kubectl get hpa tenant3-apiserver-hpa -n tenant-clusters
```

### 5. Visualize Autoscaling

#### Option A: Using kubectl
```bash
# Watch HPA status
watch kubectl get hpa tenant3-apiserver-hpa -n tenant-clusters

# Watch pod count
watch kubectl get pods -n tenant-clusters -l app=tenant3-control-plane

# Check HPA events
kubectl describe hpa tenant3-apiserver-hpa -n tenant-clusters
```

#### Option B: Using Prometheus
```bash
# Port-forward to Prometheus
kubectl port-forward -n monitoring svc/prometheus-k8s 9090:9090

# Open browser: http://localhost:9090
# Use queries from prometheus-queries.yaml
```

#### Option C: Using Grafana
```bash
# Port-forward to Grafana
kubectl port-forward -n monitoring svc/grafana 3000:3000

# Open browser: http://localhost:3000
# Import grafana-dashboard.json
```

## Expected Behavior

### During Load Test:
1. **Baseline (0-2 min)**: 3 replicas running, low CPU/memory
2. **Load starts (2-3 min)**: CPU utilization increases above 60%
3. **Scale-up triggered (3-4 min)**: HPA detects high utilization
4. **Scaling in progress (4-5 min)**: New pods are created (4th pod)
5. **Continued load (5-7 min)**: If load persists, 5th pod is created
6. **Max replicas reached (7-10 min)**: 5 pods running, load distributed
7. **Load test ends (10 min)**: Request rate drops
8. **Scale-down begins (15 min)**: After 5-min stabilization window
9. **Return to baseline (20 min)**: Back to 3 replicas

### Key Metrics to Watch:
- **HPA Current Replicas**: Should go from 3 → 4 → 5
- **CPU Utilization**: Should spike above 60% then stabilize
- **API Request Rate**: Should increase significantly during load test
- **Request Latency**: May increase slightly under load

## Troubleshooting

### HPA Not Scaling
```bash
# Check HPA status
kubectl describe hpa tenant3-apiserver-hpa -n tenant-clusters

# Verify metrics are available
kubectl top pods -n tenant-clusters

# Check metrics-server logs
kubectl logs -n kube-system -l k8s-app=metrics-server
```

### ServiceMonitor Not Scraping
```bash
# Check Prometheus targets
# Port-forward and visit: http://localhost:9090/targets

# Verify service labels match
kubectl get svc -n tenant-clusters -l app=tenant3-control-plane

# Check Prometheus operator logs
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus-operator
```

### Load Test Not Working
```bash
# Check job status
kubectl get jobs -n tenant-clusters

# Check pod logs
kubectl logs -n tenant-clusters -l app=load-test

# Verify service endpoint
kubectl get svc tenant3 -n tenant-clusters
```

## Cleanup

```bash
# Delete load test job
kubectl delete job apiserver-load-test -n tenant-clusters

# HPA and ServiceMonitor will be managed by ArgoCD
# To remove everything, delete the ArgoCD application
argocd app delete tenant1
```

## Notes

- The HPA uses both CPU and memory metrics for scaling decisions
- Scale-up is fast (60s) to handle sudden load spikes
- Scale-down is slow (300s) to prevent flapping
- ServiceMonitor assumes Prometheus Operator with label selector `prometheus: kube-prometheus`
- Adjust HPA thresholds based on your workload patterns
