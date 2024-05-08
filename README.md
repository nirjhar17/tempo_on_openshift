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
