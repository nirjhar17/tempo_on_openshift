## Tempo with Openshift
#Main doc :- https://docs.openshift.com/container-platform/4.13/observability/distr_tracing/distr_tracing_tempo/distr-tracing-tempo-configuring.html

1. Tempo Installation
 -  Install the following operators
    - Tempo
    - Open telemetry
2. Create namespaces tracing-system
```
oc new-project tracing-system
```
3. ### Deploy tempostack in the tracing-system namespace
   ### Create the secret for s3 bucket
   ### create s3 bucket in AWS and make a secret with access key and secret key
```
oc create -f metrics-tempo-s3_final.yaml
oc create -f tempostack.yaml
```

4. Create namespace for OpenTelemetry Collector
```
oc new-project opentelemetrycollector
```

5. Create opentelemetry.yaml for metrics and trace collection
```
oc create -f OpenTelemetry_new
```

6. Tempo configuration for prometheus metrics

### The TempoStack custom resource must specify the following: the Monitor tab is enabled, and the Prometheus endpoint is set to the Thanos querier service to query the data from the user-defined monitoring stack.
1. Enables the monitoring tab in the Jaeger console.
2. The service name for Thanos Querier from user-workload monitoring.
```
kind:  TempoStack
apiVersion: tempo.grafana.com/v1alpha1
metadata:
  name: simplest
spec:
  template:
    queryFrontend:
      jaegerQuery:
        enabled: true
        monitorTab:
          enabled: true  
          prometheusEndpoint: https://thanos-querier.openshift-monitoring.svc.cluster.local:9091 
        ingress:
          type: route

```
### 7. Apply the Instrumentation Configuration

```bash
oc apply -f instrumentation.yaml
```

```bash
# Check Instrumentation resource
oc get instrumentation -n opentelemetrycollector

# Describe it
oc describe instrumentation auto-instrumentation -n opentelemetrycollector
```

## Step 8: Instrument Your Applications

### Option A: Namespace-Level Instrumentation (For Single-Container Pods)

Instrument **all applications** in a namespace:

#### For Java Applications:
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-java="opentelemetrycollector/auto-instrumentation"
```

#### For Python Applications:
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-python="opentelemetrycollector/auto-instrumentation"
```

#### For Node.js Applications:
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-nodejs="opentelemetrycollector/auto-instrumentation"
```

#### For .NET Applications:
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-dotnet="opentelemetrycollector/auto-instrumentation"
```

#### For Multiple Languages in Same Namespace:
```bash
oc annotate namespace <your-namespace> \
  instrumentation.opentelemetry.io/inject-java="opentelemetrycollector/auto-instrumentation" \
  instrumentation.opentelemetry.io/inject-python="opentelemetrycollector/auto-instrumentation" \
  instrumentation.opentelemetry.io/inject-nodejs="opentelemetrycollector/auto-instrumentation"
```

### Option B: Deployment-Level Instrumentation

Instrument a **specific deployment**:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app-namespace
spec:
  template:
    metadata:
      annotations:
        # For Java apps
        instrumentation.opentelemetry.io/inject-java: "opentelemetrycollector/auto-instrumentation"
        
        # For Python apps
        # instrumentation.opentelemetry.io/inject-python: "opentelemetrycollector/auto-instrumentation"
        
        # For Node.js apps
        # instrumentation.opentelemetry.io/inject-nodejs: "opentelemetrycollector/auto-instrumentation"
    spec:
      containers:
      - name: my-app
        image: my-app:latest
```

### Option C: Multi-Container Pods (IMPORTANT!)

⚠️ **When your pods have multiple containers** (e.g., app + oauth-proxy, app + istio-proxy), namespace-level annotations will fail with:
```
ERROR: "incorrect instrumentation configuration - please provide container names for all instrumentations"
```

**Solution**: Use deployment-level annotations with **container-specific targeting**.

#### Step 1: Check Container Names
```bash
# First, identify the container names in your deployment
oc get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].name}'
# Example output: my-app-container oauth-proxy
```

#### Step 2: Apply Container-Specific Instrumentation

##### Python Application with Multiple Containers:
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

##### Java Application with Multiple Containers:
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

##### Node.js Application with Multiple Containers:
```bash
oc patch deployment <deployment-name> -n <namespace> --type=merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-nodejs": "opentelemetrycollector/auto-instrumentation",
          "instrumentation.opentelemetry.io/nodejs-container-names": "<your-app-container-name>"
        }
      }
    }
  }
}'
```

##### .NET Application with Multiple Containers:
```bash
oc patch deployment <deployment-name> -n <namespace> --type=merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-dotnet": "opentelemetrycollector/auto-instrumentation",
          "instrumentation.opentelemetry.io/dotnet-container-names": "<your-app-container-name>"
        }
      }
    }
  }
}'
```

#### Real-World Example:
```bash
# Example: KServe deployment with oauth-proxy sidecar
# Container names: kserve-container, oauth-proxy

# Instrument only the kserve-container (not oauth-proxy)
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

**Key Points:**
- ✅ Use `<language>-container-names` annotation to specify which container to instrument
- ✅ This prevents instrumenting sidecar containers (oauth-proxy, istio-proxy, etc.)
- ✅ Required for KServe, Istio, Service Mesh, and other multi-container deployments
- ✅ The container name must match exactly what's in your deployment spec

### Option D: Manual Configuration (No Auto-Instrumentation)

Configure applications manually to send traces:

#### Environment Variables:
```yaml
env:
  # OTLP gRPC
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.opentelemetrycollector.svc.cluster.local:4317"
  
  # OR OTLP HTTP
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://otel-collector.opentelemetrycollector.svc.cluster.local:4318"
  
  # Service name
  - name: OTEL_SERVICE_NAME
    value: "my-service"
  
  # Traces exporter
  - name: OTEL_TRACES_EXPORTER
    value: "otlp"
```

---

## Step 9: Restart Applications

After adding annotations, restart your applications:

```bash
# Restart all deployments in a namespace
oc rollout restart deployment -n <your-namespace>

# Or restart a specific deployment
oc rollout restart deployment <deployment-name> -n <your-namespace>
```

---

## Step 10: Verify Instrumentation

### Check if Init Container is Injected

```bash
# Get pods in your namespace
oc get pods -n <your-namespace>

# Check for OpenTelemetry init container
oc get pod <pod-name> -n <your-namespace> -o jsonpath='{.spec.initContainers[*].name}'
# Should show: opentelemetry-auto-instrumentation-<language>
```

### Check OpenTelemetry Environment Variables

```bash
# Check environment variables in your application container
oc exec -n <your-namespace> <pod-name> -c <container-name> -- env | grep OTEL

# You should see:
# OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector.opentelemetrycollector.svc.cluster.local:4318
# OTEL_TRACES_EXPORTER=otlp
# OTEL_TRACES_SAMPLER=parentbased_traceidratio
# OTEL_TRACES_SAMPLER_ARG=1.0
```

### Check OTel Collector Logs

```bash
# Check if OTel Collector is receiving traces
oc logs -n opentelemetrycollector deployment/otel-collector -f
```

### Check Tempo Ingester

```bash
# Check if Tempo is receiving traces
oc logs -n tracing-system tempo-simplest-ingester-0 -f
```

### Troubleshoot Operator Issues

```bash
# Check OpenTelemetry Operator logs
oc logs -n openshift-opentelemetry-operator deployment/opentelemetry-operator-controller-manager -f | grep -i error
```

---

## Troubleshooting

### Issue 1: "incorrect instrumentation configuration - please provide container names"

**Cause**: Your pod has multiple containers and namespace-level annotation doesn't specify which to instrument.

**Solution**: Use deployment-level annotations with container names (see Option C: Multi-Container Pods above).

### Issue 2: Init container not injected

**Cause**: Instrumentation resource not found or wrong namespace reference.

**Solution**:
```bash
# Verify Instrumentation resource exists
oc get instrumentation -n opentelemetrycollector

# Check annotation format (must match exactly)
oc get deployment <deployment-name> -n <namespace> -o yaml | grep instrumentation
```

### Issue 3: Traces not appearing in Tempo

**Cause**: Network connectivity or configuration issue.

**Solution**:
```bash
# Test connectivity from OTel Collector to Tempo
oc exec -n opentelemetrycollector deployment/otel-collector -- curl -v tempo-simplest-distributor.tracing-system.svc.cluster.local:4317

# Check OTel Collector logs for errors
oc logs -n opentelemetrycollector deployment/otel-collector | grep -i error
```

### Issue 4: Istio injection conflict

**Cause**: Namespace has `istio-injection: enabled` label but Istio is not installed or misconfigured.

**Solution**:
```bash
# Check if namespace has Istio label
oc get namespace <namespace> -o yaml | grep istio

# Remove Istio injection label if not needed
oc label namespace <namespace> istio-injection-
```

### Issue 5: Remove Instrumentation

```bash
# Remove namespace annotation
oc annotate namespace <namespace> instrumentation.opentelemetry.io/inject-python-

# Remove deployment annotation
oc patch deployment <deployment-name> -n <namespace> --type=json -p '[
  {"op": "remove", "path": "/spec/template/metadata/annotations/instrumentation.opentelemetry.io~1inject-python"},
  {"op": "remove", "path": "/spec/template/metadata/annotations/instrumentation.opentelemetry.io~1python-container-names"}
]'
```

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│  Your Applications (Python/Java/Node.js/.NET)              │
│  - Automatically instrumented via OpenTelemetry Operator   │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ OTLP/HTTP (Port 4318) or OTLP/gRPC (Port 4317)
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  OpenTelemetry Collector (opentelemetrycollector namespace) │
│  - Receives: OTLP, Jaeger, Zipkin                          │
│  - Processes: batch, memory_limiter                         │
│  - Generates: Span metrics                                  │
│  - Exports: Traces to Tempo, Metrics to Prometheus         │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      │ OTLP/gRPC (Port 4317)
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  Tempo Distributor (tracing-system namespace)               │
│  Service: tempo-simplest-distributor                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  Tempo Ingester                                             │
│  Writes traces to S3 (MinIO/AWS)                           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  S3 Storage (AWS/MinIO)                                     │
│  Bucket: tempo-traces                                       │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  Query Traces                                               │
│  - Tempo Query Frontend                                     │
│  - Grafana (with Tempo datasource)                         │
│  - Jaeger UI (via TempoStack)                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Check Container Names
```bash
oc get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[*].name}'
```

### Instrument Single-Container Pod (Namespace-Level)
```bash
oc annotate namespace <namespace> instrumentation.opentelemetry.io/inject-python="opentelemetrycollector/auto-instrumentation"
```

### Instrument Multi-Container Pod (Deployment-Level)
```bash
oc patch deployment <deployment-name> -n <namespace> --type=merge -p '{
  "spec": {
    "template": {
      "metadata": {
        "annotations": {
          "instrumentation.opentelemetry.io/inject-python": "opentelemetrycollector/auto-instrumentation",
          "instrumentation.opentelemetry.io/python-container-names": "<container-name>"
        }
      }
    }
  }
}'
```

### Restart Deployment
```bash
oc rollout restart deployment <deployment-name> -n <namespace>
```

### Verify Init Container
```bash
oc get pod <pod-name> -n <namespace> -o jsonpath='{.spec.initContainers[*].name}'
```

### Check Traces
```bash
# OTel Collector logs
oc logs -n opentelemetrycollector deployment/otel-collector -f

# Tempo ingester logs
oc logs -n tracing-system tempo-simplest-ingester-0 -f
```

---

## References

- [OpenShift Distributed Tracing with Tempo](https://docs.openshift.com/container-platform/4.13/observability/distr_tracing/distr_tracing_tempo/distr-tracing-tempo-configuring.html)
- [OpenTelemetry Operator Documentation](https://github.com/open-telemetry/opentelemetry-operator)
- [Grafana Tempo Documentation](https://grafana.com/docs/tempo/latest/)
