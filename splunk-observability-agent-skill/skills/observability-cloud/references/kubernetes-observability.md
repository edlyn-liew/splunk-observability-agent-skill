# Kubernetes Observability Reference

## Official Documentation

### Splunk K8s Monitoring
- [K8s Navigator](https://help.splunk.com/en/splunk-observability-cloud/monitor-infrastructure/monitor-services-and-hosts/monitor-kubernetes-entities)
- [Helm Chart Installation](https://help.splunk.com/en/splunk-observability-cloud/manage-data/splunk-distribution-of-the-opentelemetry-collector/get-started-with-the-splunk-distribution-of-the-opentelemetry-collector/collector-for-kubernetes/install-with-helm)
- [K8s Data Collection](https://docs.splunk.com/observability/gdi/get-data-in/compute/k8s.html)
- [OTel Operator Auto-Instrumentation](https://docs.splunk.com/Observability/gdi/opentelemetry/auto-instrumentation/auto-instrumentation-operator.html)

### GitHub
- [Helm Chart Repo](https://github.com/signalfx/splunk-otel-collector-chart)
- [Auto-Instrumentation Install Guide](https://github.com/signalfx/splunk-otel-collector-chart/blob/main/docs/auto-instrumentation-install.md)
- [OTel Collector Operator](https://github.com/signalfx/splunk-otel-collector-operator)

## Helm Chart Configuration

### Minimal Install

```bash
helm repo add splunk-otel-collector-chart https://signalfx.github.io/splunk-otel-collector-chart
helm install splunk-otel-collector splunk-otel-collector-chart/splunk-otel-collector \
  --set="splunkObservability.accessToken=<TOKEN>" \
  --set="splunkObservability.realm=<REALM>" \
  --set="clusterName=my-cluster"
```

### Full-Featured values.yaml

```yaml
clusterName: production-cluster
splunkObservability:
  accessToken: "${SPLUNK_ACCESS_TOKEN}"
  realm: us1
  logsEnabled: true
  profilingEnabled: true
environment: production

agent:
  resources:
    limits:
      cpu: 500m
      memory: 512Mi

clusterReceiver:
  resources:
    limits:
      cpu: 250m
      memory: 256Mi

# Optional: send to Splunk Enterprise too
splunkPlatform:
  endpoint: "https://splunk-hec.example.com:8088/services/collector"
  token: "${SPLUNK_HEC_TOKEN}"
  index: "k8s_metrics"
  logsIndex: "k8s_logs"

# Auto-instrumentation
operator:
  enabled: true
```

### Supported K8s Distributions

EKS, GKE, AKS, OpenShift, Rancher, k3s, CNCF-conformant distributions.

## What Gets Collected

### Metrics (via kubeletstats + k8s_cluster receivers)

| Entity | Key Metrics |
|--------|------------|
| **Node** | cpu.utilization, memory.utilization, disk.io, network.io, conditions |
| **Pod** | cpu.usage, memory.usage, restart_count, phase, ready |
| **Container** | cpu.usage, memory.usage, memory.limit, oomkilled |
| **Deployment** | replicas.available, replicas.desired, replicas.updated |
| **DaemonSet** | desired_scheduled, current_scheduled, ready |
| **StatefulSet** | replicas.current, replicas.ready |
| **Job** | active, succeeded, failed |
| **PVC** | capacity, usage |
| **Namespace** | resource quotas, limit ranges |

### Logs (via filelog receiver)

Container stdout/stderr logs collected from `/var/log/pods/` on each node. Auto-enriched with K8s metadata (pod, namespace, container, node).

### Traces (via OTLP receiver)

Applications send traces to the agent DaemonSet's OTLP endpoint (port 4317/4318). Auto-enriched with K8s resource attributes.

## Auto-Instrumentation via Operator

### Setup

```bash
# Enable operator in Helm values
operator:
  enabled: true

# Apply instrumentation CRD
kubectl apply -f - <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: splunk-otel
  namespace: my-app
spec:
  exporter:
    endpoint: http://splunk-otel-collector-agent:4317
  propagators:
    - tracecontext
    - baggage
  env:
    - name: SPLUNK_PROFILER_ENABLED
      value: "true"
EOF
```

### Annotate Pods

```yaml
# Per-pod annotation
metadata:
  annotations:
    instrumentation.opentelemetry.io/inject-java: "true"

# Or per-namespace (instruments all pods)
kubectl annotate namespace my-app instrumentation.opentelemetry.io/inject-java="true"
```

**Language annotations**: `inject-java`, `inject-python`, `inject-nodejs`, `inject-dotnet`, `inject-go`

## K8s Alert Patterns

| Alert | Condition | Severity |
|-------|-----------|----------|
| Pod CrashLoopBackOff | restart_count > 5 in 10 min | Critical |
| Node NotReady | condition.ready = false | Critical |
| Pod Pending > 5 min | phase = Pending for > 5 min | Warning |
| Container OOMKilled | oomkilled event | Major |
| Deployment < desired replicas | available < desired for > 5 min | Major |
| PVC > 90% full | usage/capacity > 0.9 | Warning |
| DaemonSet missing pods | desired != current | Major |
| High pod restart rate | restarts > 3/hour per pod | Warning |

## Platform Team K8s Patterns

### Multi-Cluster Monitoring

Deploy Helm chart per cluster with unique `clusterName`. All clusters report to same Observability Cloud org. Use Navigator to switch between clusters.

### Namespace-Level Cost Attribution

Tag all telemetry with namespace for per-team billing:

```yaml
processors:
  resource:
    attributes:
      - key: k8s.namespace.name
        from_attribute: k8s.namespace.name
        action: upsert
```

### Exclude Non-Production Namespaces

```yaml
# In Helm values — filter out kube-system, monitoring, etc.
agent:
  config:
    processors:
      filter:
        metrics:
          exclude:
            match_type: strict
            resource_attributes:
              - key: k8s.namespace.name
                value: kube-system
```

## Security Team K8s Patterns

1. **RBAC for collector** — use dedicated ServiceAccount with minimal ClusterRole
2. **Network policies** — restrict collector egress to Splunk endpoints only
3. **Secret management** — store tokens in K8s Secrets, reference in Helm values
4. **Log redaction** — use transform processor to remove sensitive fields before export
5. **Namespace isolation** — separate collector configs for sensitive namespaces
6. **Audit pod instrumentation** — track which pods are auto-instrumented
7. **Image scanning** — include collector images in vulnerability scanning pipeline
