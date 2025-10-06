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

4. Create opentelemetry.yaml for metrics and trace collection
```
oc create -f opentelemetry.yaml
```
5. Run the job.yaml for sample application
```
oc create -f job.yaml
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
### 7. Apply the Configuration

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

### Option A: Namespace-Level Instrumentation (Recommended)

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

### Option C: Manual Configuration (No Auto-Instrumentation)

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
