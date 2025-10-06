# Distributed Tracing with Tempo on OpenShift

Complete guide to set up distributed tracing using Grafana Tempo, OpenTelemetry, and the new OpenShift Traces UI with multi-tenancy support.

## 📋 Table of Contents
- [Prerequisites](#prerequisites)
- [Architecture Overview](#architecture-overview)
- [Installation Steps](#installation-steps)
- [Instrumentation Guide](#instrumentation-guide)
- [Verification & Troubleshooting](#verification--troubleshooting)
- [References](#references)

---

## Prerequisites

- OpenShift cluster (4.13+)
- Cluster admin access
- S3-compatible storage (AWS S3, MinIO, etc.)
- Basic understanding of OpenTelemetry and distributed tracing

---

## Architecture Overview

```
Applications (Python/Java/Node.js/.NET)
    ↓ (Auto-instrumented via OpenTelemetry)
    ↓ OTLP (HTTP:4318 or gRPC:4317)
    ↓
OpenTelemetry Collector (opentelemetrycollector namespace)
    ↓ (Bearer Token Auth + RBAC)
    ↓ OTLP/gRPC (Port 8090)
    ↓
Tempo Gateway (tracing-system namespace)
    ↓ (Multi-tenancy: dev/prod)
    ↓
Tempo Distributor → Ingester → S3 Storage
    ↑
    └─ Query Frontend ← OpenShift Traces UI
```

---

## Installation Steps

### Step 1: Install Required Operators

Install the following operators from OperatorHub:

1. **Red Hat build of OpenTelemetry** (openshift-opentelemetry-operator namespace)
2. **Tempo Operator** (openshift-tempo-operator namespace)
3. **Cluster Observability Operator** (openshift-cluster-observability-operator namespace)

```bash
# Wait for operators to be ready
oc get csv -n openshift-opentelemetry-operator
oc get csv -n openshift-tempo-operator
oc get csv -n openshift-cluster-observability-operator
```

---

### Step 2: Create Namespaces

```bash
# Create namespace for Tempo
oc new-project tracing-system

# Create namespace for OpenTelemetry Collector
oc new-project opentelemetrycollector
```

---

### Step 3: Configure Object Storage

#### Option A: AWS S3

Create a secret with your S3 credentials:

```bash
oc create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: metrics-tempo-s3
  namespace: tracing-system
stringData:
  endpoint: https://s3.<region>.amazonaws.com
  bucket: <your-bucket-name>
  access_key_id: <your-access-key>
  access_key_secret: <your-secret-key>
type: Opaque
EOF
```

#### Option B: MinIO

```bash
oc create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: metrics-tempo-s3
  namespace: tracing-system
stringData:
  endpoint: http://minio.minio.svc.cluster.local:9000
  bucket: tempo
  access_key_id: <minio-access-key>
  access_key_secret: <minio-secret-key>
type: Opaque
EOF
```

---

### Step 4: Deploy TempoStack with Gateway and Multi-tenancy

**🔴 CRITICAL**: The Gateway and multi-tenancy are **REQUIRED** for the OpenShift Traces UI to work!

```bash
oc create -f - <<EOF
apiVersion: tempo.grafana.com/v1alpha1
kind: TempoStack
metadata:
  name: sample
  namespace: tracing-system
spec:
  storage:
    secret:
      name: metrics-tempo-s3
      type: s3
  storageSize: 10Gi
  resources:
    total:
      limits:
        memory: 2Gi
        cpu: 2000m
  template:
    querier:
      resources:
        limits:
          cpu: "2"
    queryFrontend:
      component:
        resources:
          limits:
            memory: 6Gi
      jaegerQuery:
        enabled: true
        monitorTab:
          enabled: true
          prometheusEndpoint: http://thanos-querier.openshift-monitoring.svc.cluster.local:9094
    # CRITICAL: Gateway must be enabled for Traces UI
    gateway:
      enabled: true
      ingress:
        type: route
  # CRITICAL: Multi-tenancy is required for Traces UI
  tenants:
    mode: openshift
    authentication:
      - tenantName: dev
        tenantId: dev
      - tenantName: prod
        tenantId: prod
EOF
```

**Wait for TempoStack to be ready:**

```bash
# Watch pods starting up
oc get pods -n tracing-system -w

# Check TempoStack status
oc get tempostack sample -n tracing-system
```

Expected pods:
- `tempo-sample-distributor-*`
- `tempo-sample-ingester-*`
- `tempo-sample-querier-*`
- `tempo-sample-query-frontend-*`
- `tempo-sample-compactor-*`
- `tempo-sample-gateway-*` ← **Must be present!**

---

### Step 5: Configure RBAC for Multi-tenancy

**🔴 CRITICAL**: Without RBAC, traces cannot be written or read!

#### 5.1: RBAC for Reading Traces (Users in UI)

```bash
oc create -f - <<EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempostack-traces-reader
rules:
- apiGroups:
  - 'tempo.grafana.com'
  resources:
  - dev
  - prod
  resourceNames:
  - traces
  verbs:
  - 'get'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tempostack-traces-reader
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tempostack-traces-reader
subjects:
- kind: Group
  apiGroup: rbac.authorization.k8s.io
  name: system:authenticated
EOF
```

#### 5.2: RBAC for Writing Traces (OTel Collector)

```bash
oc create -f - <<EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tempostack-traces-write
rules:
- apiGroups:
  - 'tempo.grafana.com'
  resources:
  - dev
  - prod
  resourceNames:
  - traces
  verbs:
  - 'create'
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tempostack-traces-write
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tempostack-traces-write
subjects:
- kind: ServiceAccount
  name: otel-collector
  namespace: opentelemetrycollector
EOF
```

**Verify RBAC:**

```bash
oc get clusterrole tempostack-traces-reader tempostack-traces-write
oc get clusterrolebinding tempostack-traces-reader tempostack-traces-write
```

---

### Step 6: Deploy OpenTelemetry Collector with Bearer Token Auth

**🔴 CRITICAL**: Bearer token authentication is required for Gateway communication!

```bash
oc create -f - <<'EOF'
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel
  namespace: opentelemetrycollector
spec:
  mode: deployment
  replicas: 1
  
  observability:
    metrics:
      enableMetrics: true
  
  config: |
    # ========================================
    # EXTENSIONS - Authentication
    # ========================================
    extensions:
      # Bearer token auth using service account token
      bearertokenauth:
        filename: "/var/run/secrets/kubernetes.io/serviceaccount/token"

    # ========================================
    # RECEIVERS - Accept traces from apps
    # ========================================
    receivers:
      # OTLP receiver (modern standard)
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      
      # Jaeger receiver (for legacy apps)
      jaeger:
        protocols:
          grpc:
            endpoint: 0.0.0.0:14250
          thrift_http:
            endpoint: 0.0.0.0:14268
      
      # Zipkin receiver (for legacy apps)
      zipkin:
        endpoint: 0.0.0.0:9411

    # ========================================
    # PROCESSORS - Process traces
    # ========================================
    processors:
      # Batch traces for efficiency
      batch:
        timeout: 10s
        send_batch_size: 1024
      
      # Prevent out-of-memory errors
      memory_limiter:
        check_interval: 1s
        limit_mib: 512

    # ========================================
    # CONNECTORS - Generate metrics from spans
    # ========================================
    connectors:
      spanmetrics:
        metrics_flush_interval: 15s

    # ========================================
    # EXPORTERS - Send data to backends
    # ========================================
    exporters:
      # Export traces to Tempo Gateway with authentication
      otlp:
        endpoint: "tempo-sample-gateway.tracing-system.svc.cluster.local:8090"
        tls:
          # Use OpenShift service CA
          insecure: false
          ca_file: "/var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt"
        # Bearer token authentication
        auth:
          authenticator: bearertokenauth
        # Tenant header (change to 'prod' if needed)
        headers:
          X-Scope-OrgID: "dev"
      
      # Export span metrics to Prometheus
      prometheus:
        endpoint: 0.0.0.0:8889
        add_metric_suffixes: false
        resource_to_telemetry_conversion:
          enabled: true

    # ========================================
    # SERVICE - Define data flow
    # ========================================
    service:
      # Enable bearer token auth extension
      extensions: [bearertokenauth]
      
      pipelines:
        # Traces pipeline: Receive → Process → Export to Tempo + Generate Metrics
        traces:
          receivers: [otlp, jaeger, zipkin]
          processors: [memory_limiter, batch]
          exporters: [otlp, spanmetrics]
        
        # Metrics pipeline: Receive from spanmetrics → Export to Prometheus
        metrics:
          receivers: [spanmetrics]
          exporters: [prometheus]
EOF
```

**Verify OTel Collector:**

```bash
# Check pod is running
oc get pods -n opentelemetrycollector

# Check logs for bearer token auth
oc logs -n opentelemetrycollector -l app.kubernetes.io/component=opentelemetry-collector | grep bearertokenauth

# Should see:
# "bearertokenauth extension started"
# "refresh token from /var/run/secrets/kubernetes.io/serviceaccount/token"
```

---

### Step 7: Enable OpenShift Traces UI

```bash
oc create -f - <<EOF
apiVersion: observability.openshift.io/v1alpha1
kind: UIPlugin
metadata:
  name: distributed-tracing
spec:
  type: DistributedTracing
EOF
```

**Refresh your OpenShift Console browser** and navigate to:
- **Observe** → **Traces**

You should now see the Traces UI with:
- TempoStack: `sample`
- Tenants: `dev`, `prod`

---

### Step 8: Create Instrumentation Resource

```bash
oc create -f - <<'EOF'
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
  namespace: opentelemetrycollector
spec:
  # Where to send traces - points to OTel Collector
  exporter:
    endpoint: http://otel-collector.opentelemetrycollector.svc.cluster.local:4318
  
  # Trace context propagation formats
  propagators:
    - tracecontext
    - baggage
    - b3
  
  # Sampling configuration (100% sampling)
  sampler:
    type: parentbased_traceidratio
    argument: "1.0"
  
  # Java auto-instrumentation
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
    env:
      - name: OTEL_EXPORTER_OTLP_TIMEOUT
        value: "20000"
      - name: OTEL_TRACES_EXPORTER
        value: otlp
      - name: OTEL_METRICS_EXPORTER
        value: none
      - name: OTEL_LOGS_EXPORTER
        value: none
  
  # Python auto-instrumentation
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
    env:
      - name: OTEL_EXPORTER_OTLP_TIMEOUT
        value: "20"
      - name: OTEL_TRACES_EXPORTER
        value: otlp
      - name: OTEL_METRICS_EXPORTER
        value: none
      - name: OTEL_LOGS_EXPORTER
        value: none
  
  # Node.js auto-instrumentation
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
    env:
      - name: OTEL_EXPORTER_OTLP_TIMEOUT
        value: "20000"
      - name: OTEL_TRACES_EXPORTER
        value: otlp
      - name: OTEL_METRICS_EXPORTER
        value: none
      - name: OTEL_LOGS_EXPORTER
        value: none
  
  # .NET auto-instrumentation
  dotnet:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-dotnet:latest
    env:
      - name: OTEL_EXPORTER_OTLP_TIMEOUT
        value: "20000"
      - name: OTEL_TRACES_EXPORTER
        value: otlp
      - name: OTEL_METRICS_EXPORTER
        value: none
      - name: OTEL_LOGS_EXPORTER
        value: none
EOF
```

**Verify Instrumentation:**

```bash
oc get instrumentation -n opentelemetrycollector
oc describe instrumentation auto-instrumentation -n opentelemetrycollector
```

---

## Instrumentation Guide

### Understanding Auto-Instrumentation

The OpenTelemetry Operator can automatically inject tracing into your applications without code changes by:
1. Adding an init container that downloads the OpenTelemetry SDK
2. Setting environment variables to configure the SDK
3. Instrumenting your application at startup

### Option A: Single-Container Pods (Namespace-Level)

**For Python applications:**
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-python="opentelemetrycollector/auto-instrumentation"
```

**For Java applications:**
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-java="opentelemetrycollector/auto-instrumentation"
```

**For Node.js applications:**
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-nodejs="opentelemetrycollector/auto-instrumentation"
```

**For .NET applications:**
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-dotnet="opentelemetrycollector/auto-instrumentation"
```

### Option B: Multi-Container Pods (Deployment-Level) ⭐ RECOMMENDED

**⚠️ IMPORTANT**: If your pods have multiple containers (e.g., app + oauth-proxy, app + istio-proxy), you **MUST** use deployment-level annotations with container targeting!

#### Step 1: Identify Container Names

```bash
oc get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].name}'
# Example output: kserve-container oauth-proxy
```

#### Step 2: Apply Instrumentation with Container Targeting

**Python application with multiple containers:**
```bash
oc patch deployment <deployment-name> -n <namespace> --type=merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-python": "opentelemetrycollector/auto-instrumentation",
          "instrumentation.opentelemetry.io/python-container-names": "<your-app-container-name>"
        }
      }
    }
  }
}'
```

**Java application:**
```bash
oc patch deployment <deployment-name> -n <namespace> --type=merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-java": "opentelemetrycollector/auto-instrumentation",
          "instrumentation.opentelemetry.io/java-container-names": "<your-app-container-name>"
        }
      }
    }
  }
}'
```

**Real-world example (KServe with oauth-proxy):**
```bash
# Container names: kserve-container, oauth-proxy
# Instrument only kserve-container, not oauth-proxy

oc patch deployment llama-predictor -n model --type=merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-python": "opentelemetrycollector/auto-instrumentation",
          "instrumentation.opentelemetry.io/python-container-names": "kserve-container"
        }
      }
    }
  }
}'
```

### Option C: YAML-Based Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-namespace
spec:
  template:
    metadata:
      annotations:
        # For Python apps with multiple containers
        instrumentation.opentelemetry.io/inject-python: "opentelemetrycollector/auto-instrumentation"
        instrumentation.opentelemetry.io/python-container-names: "app-container"
    spec:
      containers:
      - name: app-container
        image: my-app:latest
      - name: oauth-proxy
        image: oauth-proxy:latest
```

### Restart Applications

After adding instrumentation annotations:

```bash
# Restart specific deployment
oc rollout restart deployment <deployment-name> -n <namespace>

# Or restart all deployments in namespace
oc rollout restart deployment -n <namespace>
```

---

## Verification & Troubleshooting

### Verify Complete Setup

```bash
# 1. Check TempoStack is ready
oc get tempostack sample -n tracing-system
# Status should be "Ready"

# 2. Check all Tempo pods are running
oc get pods -n tracing-system
# All pods should be "Running"

# 3. Check Gateway is present
oc get pods -n tracing-system -l app.kubernetes.io/component=gateway
# Should see tempo-sample-gateway pod

# 4. Check RBAC is configured
oc get clusterrole tempostack-traces-reader tempostack-traces-write
oc get clusterrolebinding tempostack-traces-reader tempostack-traces-write

# 5. Check OTel Collector is running with bearer token auth
oc logs -n opentelemetrycollector -l app.kubernetes.io/component=opentelemetry-collector | grep bearertokenauth
# Should see: "bearertokenauth extension started"

# 6. Check UIPlugin is active
oc get uiplugin distributed-tracing
# Should exist
```

### Verify Instrumentation

```bash
# Check if init container is injected
oc get pod <pod-name> -n <namespace> -o jsonpath='{.spec.initContainers[*].name}'
# Should show: opentelemetry-auto-instrumentation-<language>

# Check environment variables
oc exec -n <namespace> <pod-name> -c <container-name> -- env | grep OTEL
# Should see OTEL_* variables
```

### Check Trace Flow

```bash
# 1. Check OTel Collector logs
oc logs -n opentelemetrycollector -l app.kubernetes.io/component=opentelemetry-collector --tail=50

# 2. Check Gateway logs
oc logs -n tracing-system -l app.kubernetes.io/component=gateway --tail=50
# Should see POST requests, NOT "TLS handshake error"

# 3. Check Distributor logs
oc logs -n tracing-system -l app.kubernetes.io/component=distributor --tail=50

# 4. Check Ingester logs
oc logs -n tracing-system -l app.kubernetes.io/component=ingester --tail=50
```

### View Traces in UI

1. Open **OpenShift Console**
2. Navigate to **Observe** → **Traces**
3. Select **TempoStack**: `sample`
4. Select **Tenant**: `dev`
5. Set time range: **Last 15 minutes**
6. Click **Run Query**

You should see traces in the scatter plot and table!

### Common Issues

#### Issue 1: "No Tempo instances" in Traces UI

**Cause**: Gateway not enabled or multi-tenancy not configured

**Solution**: Verify TempoStack configuration:
```bash
oc get tempostack sample -n tracing-system -o yaml | grep -A 10 gateway
oc get tempostack sample -n tracing-system -o yaml | grep -A 10 tenants
```

Both `gateway.enabled: true` and `tenants.mode: openshift` must be present.

#### Issue 2: "TLS handshake error" in Gateway logs

**Cause**: Missing RBAC or bearer token authentication

**Solution**: 
1. Verify RBAC ClusterRoles exist
2. Verify OTel Collector has bearer token auth configured
3. Restart OTel Collector:
```bash
oc delete pod -l app.kubernetes.io/component=opentelemetry-collector -n opentelemetrycollector
```

#### Issue 3: Init container not injected

**Cause**: Incorrect annotation format or multi-container pod without container targeting

**Solution**:
1. For multi-container pods, use `<language>-container-names` annotation
2. Verify annotation format:
```bash
oc get deployment <name> -n <namespace> -o yaml | grep instrumentation
```

#### Issue 4: Traces not appearing in UI

**Cause**: No application traffic or instrumentation not working

**Solution**:
1. Generate traffic to your application
2. Check OTel Collector logs for trace activity
3. Verify init container is present in pod
4. Check application logs for OpenTelemetry initialization

#### Issue 5: Remove instrumentation

```bash
# Remove namespace annotation
oc annotate namespace <namespace> \
  instrumentation.opentelemetry.io/inject-python-

# Remove deployment annotation
oc patch deployment <deployment-name> -n <namespace> --type=json -p '[
  {"op": "remove", "path": "/spec/template/metadata/annotations/instrumentation.opentelemetry.io~1inject-python"},
  {"op": "remove", "path": "/spec/template/metadata/annotations/instrumentation.opentelemetry.io~1python-container-names"}
]'
```

---

## Key Differences from Basic Setup

This setup includes **critical components** required for the OpenShift Traces UI that are often missing in basic guides:

1. ✅ **Gateway enabled** - Required for Traces UI discovery
2. ✅ **Multi-tenancy** (`mode: openshift`) - Required for Traces UI
3. ✅ **RBAC for reading traces** - Without this, UI shows empty
4. ✅ **RBAC for writing traces** - Without this, OTel can't send traces
5. ✅ **Bearer token authentication** - Required for Gateway communication
6. ✅ **UIPlugin resource** - Enables Traces UI in console
7. ✅ **Container-specific instrumentation** - For multi-container pods

---

## Quick Reference Commands

### Setup
```bash
# Create namespaces
oc new-project tracing-system
oc new-project opentelemetrycollector

# Apply all resources
oc apply -f metrics-tempo-s3_final.yaml      # S3 secret
oc apply -f tempostack.yaml                   # TempoStack (with gateway)
oc apply -f rbac-traces-reader.yaml           # RBAC for users
oc apply -f rbac-traces-writer.yaml           # RBAC for OTel
oc apply -f OpenTelemetry_new                 # OTel Collector (with auth)
oc apply -f instrumentation.yaml              # Instrumentation resource
oc apply -f uiplugin                          # UI Plugin
```

### Instrumentation
```bash
# Single-container (namespace-level)
oc annotate namespace <ns> instrumentation.opentelemetry.io/inject-python="opentelemetrycollector/auto-instrumentation"

# Multi-container (deployment-level)
oc patch deployment <deploy> -n <ns> --type=merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-python": "opentelemetrycollector/auto-instrumentation",
          "instrumentation.opentelemetry.io/python-container-names": "<container>"
        }
      }
    }
  }
}'
```

### Verification
```bash
# Check everything
oc get tempostack -n tracing-system
oc get pods -n tracing-system
oc get clusterrole tempostack-traces-reader tempostack-traces-write
oc get pods -n opentelemetrycollector
oc get uiplugin

# Check logs
oc logs -n opentelemetrycollector -l app.kubernetes.io/component=opentelemetry-collector
oc logs -n tracing-system -l app.kubernetes.io/component=gateway
```

---

## References

- [OpenShift Distributed Tracing with Tempo (Official)](https://docs.openshift.com/container-platform/4.13/observability/distr_tracing/distr_tracing_tempo/distr-tracing-tempo-configuring.html)
- [Cluster Observability Operator](https://docs.openshift.com/container-platform/4.18/html-single/cluster_observability_operator/index)
- [New Traces UI Blog Post](https://developers.redhat.com/articles/2024/07/10/introducing-new-traces-ui-red-hat-openshift-web-console)
- [OpenTelemetry Operator](https://github.com/open-telemetry/opentelemetry-operator)
- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/)
- [Tempo Operator GitHub](https://github.com/grafana/tempo-operator)

---

## Credits

This guide consolidates best practices from:
- Red Hat OpenShift documentation
- Grafana Tempo Operator documentation
- OpenTelemetry Operator documentation
- Real-world production deployments

**Last Updated**: October 2025  
**Tested on**: OpenShift 4.18 on AWS ROSA
