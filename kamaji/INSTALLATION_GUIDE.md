# Kamaji Installation Guide with HPA Autoscaling

This guide will walk you through installing Kamaji on a fresh Kubernetes cluster and setting up HPA autoscaling for tenant control planes.

## Prerequisites

Before you begin, ensure you have:

- A Kubernetes cluster (v1.24+)
- `kubectl` configured to access your cluster
- `helm` installed (v3.0+)
- ArgoCD installed on your cluster
- Git repository access (for GitOps)

## Step 1: Verify Your Cluster

First, make sure you're connected to the right cluster:

```bash
# Check current context
kubectl config current-context

# Verify cluster is accessible
kubectl get nodes
```

## Step 2: Install Prerequisites

### 2.1 Install Metrics Server

Metrics Server is required for HPA to get CPU and memory metrics.

```bash
# Create ArgoCD Application for metrics-server
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-server
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://kubernetes-sigs.github.io/metrics-server/
    chart: metrics-server
    targetRevision: 3.11.0
    helm:
      values: |
        args:
          - --kubelet-insecure-tls
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

# Wait for metrics-server to be ready
kubectl wait --for=condition=available --timeout=300s deployment/metrics-server -n kube-system

# Verify metrics are available
kubectl top nodes
```

### 2.2 Install Prometheus Operator (kube-prometheus-stack)

This provides Prometheus, Grafana, and monitoring capabilities.

```bash
# Create ArgoCD Application for kube-prometheus-stack
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 55.5.0
    helm:
      values: |
        prometheus:
          prometheusSpec:
            serviceMonitorSelectorNilUsesHelmValues: false
        grafana:
          adminPassword: admin
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

# Wait for Prometheus and Grafana to be ready
kubectl wait --for=condition=available --timeout=600s deployment/kube-prometheus-stack-operator -n monitoring
kubectl wait --for=condition=available --timeout=600s deployment/kube-prometheus-stack-grafana -n monitoring

# Verify Prometheus is running
kubectl get pods -n monitoring | grep prometheus
```

## Step 3: Install Kamaji

### 3.1 Create Kamaji ArgoCD Project

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: kamaji
  namespace: argocd
spec:
  description: Kamaji Hosted Control Plane Manager
  sourceRepos:
    - 'https://github.com/arpit-devops/argocd-demos.git'
    - 'https://clastix.github.io/charts'
  destinations:
    - namespace: kamaji-system
      server: https://kubernetes.default.svc
    - namespace: tenant-clusters
      server: https://kubernetes.default.svc
    - namespace: monitoring
      server: https://kubernetes.default.svc
    - namespace: kube-system
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
EOF
```

### 3.2 Install Kamaji Operator

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kamaji
  namespace: argocd
spec:
  project: kamaji
  source:
    repoURL: https://clastix.github.io/charts
    chart: kamaji
    targetRevision: 1.0.0
  destination:
    server: https://kubernetes.default.svc
    namespace: kamaji-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

# Wait for Kamaji operator to be ready
kubectl wait --for=condition=available --timeout=300s deployment/kamaji-controller-manager -n kamaji-system

# Verify Kamaji CRDs are installed
kubectl get crd | grep kamaji
```

### 3.3 Verify Scale Subresource is Available

This is critical for HPA to work with Kamaji:

```bash
# Check if TenantControlPlane has scale subresource
kubectl get crd tenantcontrolplanes.kamaji.clastix.io -o yaml | grep -A 5 "subresources:"

# You should see:
# subresources:
#   scale:
#     specReplicasPath: .spec.controlPlane.deployment.replicas
#     statusReplicasPath: .status.kubernetesResources.deployment.replicas
```

## Step 4: Create Tenant Control Plane with HPA

### 4.1 Create TenantControlPlane

```bash
kubectl apply -f - <<EOF
apiVersion: kamaji.clastix.io/v1alpha1
kind: TenantControlPlane
metadata:
  name: tenant3
  namespace: tenant-clusters
  labels:
    app: tenant3-control-plane
spec:
  controlPlane:
    deployment:
      replicas: 3
      extraArgs:
        apiServer:
          - --profiling=true
      resources:
        apiServer:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 1000m
            memory: 1Gi
        controllerManager:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        scheduler:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
      service:
        additionalMetadata:
          labels:
            app: tenant3-control-plane
            monitoring: enabled
        serviceType: ClusterIP
  kubernetes:
    version: v1.26.1
  networkProfile:
    dnsServiceIPs:
      - 10.96.0.10
    podCIDR: 10.244.0.0/16
    serviceCIDR: 10.96.0.0/16
EOF

# Wait for tenant control plane to be ready
kubectl wait --for=condition=Ready --timeout=600s tcp/tenant3 -n tenant-clusters

# Verify pods are running
kubectl get pods -n tenant-clusters
```

### 4.2 Create HorizontalPodAutoscaler

**IMPORTANT**: HPA must target the TenantControlPlane CRD (not the Deployment) to use Kamaji's scale subresource.

```bash
kubectl apply -f - <<EOF
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tenant3-apiserver-hpa
  namespace: tenant-clusters
spec:
  scaleTargetRef:
    apiVersion: kamaji.clastix.io/v1alpha1
    kind: TenantControlPlane
    name: tenant3
  minReplicas: 3
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
        - type: Pods
          value: 1
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 120
      selectPolicy: Min
EOF

# Verify HPA is working
kubectl get hpa -n tenant-clusters

# You should see:
# NAME                    REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS
# tenant3-apiserver-hpa   TenantControlPlane/tenant3   25%/60%, 30%/70%   3         5         3
```

## Step 5: Test HPA Autoscaling

### 5.1 Create Load Test Job

This job generates API requests to trigger autoscaling:

```bash
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: apiserver-load-test
  namespace: tenant-clusters
spec:
  parallelism: 8
  completions: 8
  backoffLimit: 0
  template:
    metadata:
      labels:
        app: load-test
    spec:
      restartPolicy: Never
      containers:
        - name: load-generator
          image: bitnami/kubectl:latest
          command:
            - /bin/bash
            - -c
            - |
              set -e
              
              echo "Starting BALANCED API server load test..."
              echo "Target: tenant3 API server"
              echo "Pod: \$HOSTNAME"
              
              # Get the service endpoint
              API_SERVER="tenant3.tenant-clusters.svc.cluster.local:6443"
              TOKEN=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
              
              # Generate balanced sustained load for 10 minutes
              END_TIME=\$(($(date +%s) + 600))
              COUNTER=0
              
              while [ \$(date +%s) -lt \$END_TIME ]; do
                # Controlled parallel batches - 6 requests at a time
                for i in {1..6}; do
                  kubectl --server=https://\${API_SERVER} \\
                    --insecure-skip-tls-verify \\
                    --token=\$TOKEN \\
                    get nodes --request-timeout=2s 2>/dev/null || true &
                  
                  kubectl --server=https://\${API_SERVER} \\
                    --insecure-skip-tls-verify \\
                    --token=\$TOKEN \\
                    get pods -A --request-timeout=2s 2>/dev/null || true &
                  
                  kubectl --server=https://\${API_SERVER} \\
                    --insecure-skip-tls-verify \\
                    --token=\$TOKEN \\
                    get services -A --request-timeout=2s 2>/dev/null || true &
                done
                
                # Wait for batch to complete
                wait
                
                COUNTER=\$((COUNTER + 1))
                if [ \$((COUNTER % 20)) -eq 0 ]; then
                  echo "[\$HOSTNAME] Load test running... \$(date) - Completed \$COUNTER batches"
                fi
                
                # Balanced delay - not too fast, not too slow
                sleep 0.3
              done
              
              echo "[\$HOSTNAME] Load test completed! Total batches: \$COUNTER"
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
EOF
```

### 5.2 Monitor Autoscaling

Watch HPA scale the control plane:

```bash
# Monitor HPA in real-time
watch kubectl get hpa tenant3-apiserver-hpa -n tenant-clusters

# Monitor pod count
watch 'kubectl get pods -n tenant-clusters | grep tenant3'

# Check HPA events
kubectl describe hpa tenant3-apiserver-hpa -n tenant-clusters

# View load test logs
kubectl logs -f job/apiserver-load-test -n tenant-clusters --max-log-requests=8
```

### 5.3 Expected Behavior

**Timeline:**

- **T+0-60s**: CPU climbing from 25% → 60%+
- **T+60-120s**: HPA triggers scale-up (3 → 4 replicas)
- **T+120-180s**: 4th pod starts and becomes ready
- **T+180-300s**: If CPU stays high, scales to 5 replicas
- **T+600s+**: Load test completes, CPU drops
- **T+900s+**: After 5 min cooldown, scales down to 3 replicas

**Verification:**

```bash
# Check TCP spec was updated by HPA
kubectl get tcp tenant3 -n tenant-clusters -o jsonpath='{.spec.controlPlane.deployment.replicas}'

# Check deployment has correct replicas
kubectl get deployment tenant3 -n tenant-clusters

# Verify new pods are running
kubectl get pods -n tenant-clusters | grep tenant3
```

## Step 6: Configure ArgoCD for HPA Compatibility

If using GitOps with ArgoCD, configure it to ignore the replicas field:

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tenant1
  namespace: argocd
spec:
  project: kamaji
  source:
    repoURL: https://github.com/arpit-devops/argocd-demos.git
    targetRevision: main
    path: kamaji/tcp
  destination:
    server: https://kubernetes.default.svc
    namespace: tenant-clusters
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  ignoreDifferences:
    - group: kamaji.clastix.io
      kind: TenantControlPlane
      jsonPointers:
        - /spec/controlPlane/deployment/replicas
EOF
```

This prevents ArgoCD from resetting the replica count when HPA scales.

## Troubleshooting

### HPA shows "unknown" metrics

**Problem**: `kubectl get hpa` shows `<unknown>/60%`

**Solution**: Ensure resource requests are defined in TenantControlPlane:

```bash
kubectl get tcp tenant3 -n tenant-clusters -o yaml | grep -A 10 "resources:"
```

All containers (apiServer, controllerManager, scheduler) must have CPU/memory requests.

### HPA not scaling

**Problem**: HPA shows metrics but doesn't scale

**Solution**: Verify HPA is targeting TenantControlPlane (not Deployment):

```bash
kubectl get hpa tenant3-apiserver-hpa -n tenant-clusters -o yaml | grep -A 3 "scaleTargetRef:"

# Should show:
# scaleTargetRef:
#   apiVersion: kamaji.clastix.io/v1alpha1
#   kind: TenantControlPlane
#   name: tenant3
```

### Pods crash during load test

**Problem**: Load test pods show OOMKilled or API server pods crash

**Solution**: Reduce load test intensity:
- Decrease `parallelism` (try 5 instead of 8)
- Increase `sleep` duration (try 0.5 instead of 0.3)
- Reduce parallel requests per batch (try 4 instead of 6)

### ArgoCD keeps resetting replicas

**Problem**: HPA scales to 4, but ArgoCD resets to 3

**Solution**: Add `ignoreDifferences` to ArgoCD Application (see Step 6)

## Summary

You now have:

✅ Kamaji installed with scale subresource support  
✅ TenantControlPlane with resource requests for HPA  
✅ HPA targeting TenantControlPlane CRD  
✅ Load test to trigger autoscaling  
✅ ArgoCD configured to not interfere with HPA  

The HPA will automatically scale your tenant control plane from 3 to 5 replicas based on CPU and memory utilization.

## Next Steps

- Set up observability (see OBSERVABILITY_GUIDE.md)
- Configure custom metrics for more advanced autoscaling
- Try KEDA for event-driven autoscaling
- Monitor datastore latency and performance
