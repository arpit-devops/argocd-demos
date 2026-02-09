# Prerequisites for Kamaji TCP Autoscaling

These applications install the required components for HPA and monitoring.

## Components

### 1. kube-prometheus-stack
- **Namespace**: `monitoring`
- **Includes**:
  - Prometheus Operator
  - Prometheus Server
  - Grafana
  - AlertManager
  - Node Exporter
  - Kube State Metrics
- **Purpose**: Provides ServiceMonitor CRD and metrics collection

### 2. metrics-server
- **Namespace**: `kube-system`
- **Purpose**: Provides resource metrics API for HPA

## Installation Order

1. Apply prerequisites first:
   ```bash
   kubectl apply -f prerequisites/
   ```

2. Wait for prerequisites to be ready:
   ```bash
   # Wait for Prometheus Operator
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=prometheus-operator -n monitoring --timeout=300s
   
   # Wait for metrics-server
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=metrics-server -n kube-system --timeout=300s
   ```

3. Verify installation:
   ```bash
   # Check ServiceMonitor CRD
   kubectl get crd servicemonitors.monitoring.coreos.com
   
   # Check metrics-server
   kubectl get deployment metrics-server -n kube-system
   
   # Test metrics API
   kubectl top nodes
   kubectl top pods -n tenant-clusters
   ```

4. Access Grafana:
   ```bash
   kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
   # Username: admin
   # Password: admin
   ```

## Post-Installation

Once prerequisites are ready, the tenant1 application will automatically sync and deploy:
- TenantControlPlane with autoscaling
- ServiceMonitor for metrics
- HorizontalPodAutoscaler

Then you can run the load test to visualize autoscaling.
