# installation doc

## **Prerequisite**

We will deploy in the [Kind](https://kind.sigs.k8s.io/) environment.

## Architecture

Prepare 3 clusters: obs-hub, obs-c1, obs-c2.

```yaml
+------------------------+                                  +------------------------+
|         obs-c1         |                                  |         obs-c2         |
|                        |                                  |                        |
|   [Kepler Exporter]    |                                  |   [Kepler Exporter]    |
|   [Kubelet cAdvisor]   |                                  |   [Kubelet cAdvisor]   |
|           |            |                                  |           |            |
|           v            |                                  |           v            |
|   [OTel Collector]     |                                  |   [OTel Collector]     |
+------------------------+                                  +------------------------+
           |                                                              |
           |         (Metrics)                       (Metrics)            |
           +-----------------------------+--------------------------------+
                                         |
                                         v
                           +---------------------------+
                           |        Hub Cluster        |
                           |                           |
                           |    [Kepler Exporter]      |
                           |    [Kubelet cAdvisor]     |
                           |   [Hub OTel Collector]    |
                           |             |             |
                           |             v             |
                           |     [Hub Prometheus]      |
                           +---------------------------+
```

## Install kube-prometheus on Hub

We can install kube-prometheus according to the document: [deploy the prometheus operator](https://sustainable-computing.io/installation/kepler/#deploy-the-prometheus-operator).

## Install OTel on Hub

### Prepare the OTel Operator

This step we will install [OpenTelemetry Operator](https://opentelemetry.io/docs/platforms/kubernetes/operator/#getting-started) on Hub cluster.

```bash
# 1. Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Wait for cert-manager to finish installing...

# 2. Install OTel Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

```

### Create OTel Collector

After installing OTel Operator. Now we can create the OpenTelemetryCollector.

`kubectl apply -f hub-collector.yaml`

*TODO: FIX*
```yaml
# hub-collector.yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: hub
spec:
  image: otel/opentelemetry-collector:latest
  config:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318

    processors:
      attributes:
        actions:
        - key: cluster
          value: hub
          action: insert

    exporters:
      prometheus:
        endpoint: "0.0.0.0:9464"

    service:
      pipelines:
        metrics:
          receivers: [otlp]
          processors: [attributes]
          exporters: [prometheus]
```

This will let the collector listen on ports 4317 and 4318 to receive metrics.

### Change the service type to NodePort

Because the managed clusters will send metrics to the hub cluster, we need to change the OTel collector service to NodePort.

`kubectl patch svc hub-collector -n default -p '{"spec":{"type":"NodePort"}}'`

### Create ServiceMonitor

This will let the kube-prometheus find OTel Collector’s exporter.

`kubectl apply -f otel-collector-smon.yaml`

```yaml
# otel-collector-smon.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      operator.opentelemetry.io/collector-service-type: base
  namespaceSelector:
    matchNames:
      - default
  endpoints:
    - port: prometheus
      path: /metrics
      interval: 15s
```

## Install Kepler on obs-c1, obs-c2

[Deploy Kepler on Kind](https://sustainable-computing.io/installation/kepler/#deploying-kepler-on-a-local-kind-cluster) clusters.

NOTE: We set `OPTS=""` here, because we do not need prometheus stack on obs-c1 and obs-c2.

```bash
git clone --depth 1 git@github.com:sustainable-computing-io/kepler.git
cd ./kepler
make build-manifest OPTS=""
```

Then run the command on obs-c1 and obs-c2 respectively.

```bash
kubectl apply -f _output/generated-manifest/deployment.yaml
```

## Install OTel Operator on obs-c1, obs-c2

Same as [install OTel Operator on hub cluster](#Prepare-the-OTel-Operator).

```bash
# 1. Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Wait for cert-manager to finish installing...

# 2. Install OTel Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

```

## Create OTel Collector on obs-c1

### Grant RBAC permissions

Because grabbing cAdvisor metrics requires RBAC permissions, run`kubectl apply -f c1-otel-collector-rbac.yaml`

```yaml
# c1-otel-collector-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-kubeletstats-role
rules:
- apiGroups: [""]
  resources:
  - "nodes"
  - "nodes/stats"
  - "nodes/proxy"
  - "services"
  - "pods"
  verbs:
  - "get"
  - "list"
  - "watch"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-kubeletstats-binding
subjects:
- kind: ServiceAccount
  name: otel-collector
  namespace: default
roleRef:
  kind: ClusterRole
  name: otel-collector-kubeletstats-role
  apiGroup: rbac.authorization.k8s.io
```

### Create OTel Collector

Here we need to use hub’s OTel Collector’s grpc node port.

`kubectl apply -f c1-collector.yaml`

```yaml
# c1-collector.yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel
spec:
  image: otel/opentelemetry-collector-contrib:latest
  config:
    receivers:
      prometheus:
        config:
          scrape_configs:
            - job_name: 'kepler-pods'
              kubernetes_sd_configs:
                - role: pod
              relabel_configs:
                - source_labels: [__meta_kubernetes_namespace]
                  action: keep
                  regex: kepler

                - source_labels: [__address__, __meta_kubernetes_pod_container_port_number]
                  action: replace
                  regex: ([^:]+):(?:\d+);(\d+)
                  replacement: $${1}:$${2}
                  target_label: __address__

                - source_labels: [__meta_kubernetes_node_name]
                  action: replace
                  target_label: instance
            - job_name: 'kubernetes-cadvisor-central'
              kubernetes_sd_configs:
                - role: node
              scheme: https
              authorization:
                credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
              tls_config:
                ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                insecure_skip_verify: true
              
              relabel_configs:
                - action: labelmap
                  regex: __meta_kubernetes_node_label_(.+)
                - target_label: __address__
                  replacement: kubernetes.default.svc:443
                - source_labels: [__meta_kubernetes_node_name]
                  regex: (.+)
                  target_label: __metrics_path__
                  replacement: /api/v1/nodes/$${1}/proxy/metrics/cadvisor

    processors:
      attributes:
        actions:
        - key: cluster
          value: c1
          action: insert
          
    exporters:
      otlp:
        # hub grpc receiver address.
        endpoint: "172.18.0.4:32333"
        tls:
          insecure: true

    service:
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: [attributes]
          exporters: [otlp]
```

## Create OTel Collector on obs-c2

The steps are almost the same as obs-c1, except for some name changes.

### Grant RBAC permissions

Because grabbing cAdvisor metrics requires RBAC permissions, run`kubectl apply -f c2-otel-collector-rbac.yaml`

```yaml
# c2-otel-collector-rbac.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: otel-collector-kubeletstats-role
rules:
- apiGroups: [""]
  resources:
  - "nodes"
  - "nodes/stats"
  - "nodes/proxy"
  - "services"
  - "pods"
  verbs:
  - "get"
  - "list"
  - "watch"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: otel-collector-kubeletstats-binding
subjects:
- kind: ServiceAccount
  name: otel-collector
  namespace: default
roleRef:
  kind: ClusterRole
  name: otel-collector-kubeletstats-role
  apiGroup: rbac.authorization.k8s.io
```

### Create OTel Collector

`kubectl apply -f c2-collector.yaml`

```yaml
# c2-collector.yaml
apiVersion: opentelemetry.io/v1beta1
kind: OpenTelemetryCollector
metadata:
  name: otel
spec:
  image: otel/opentelemetry-collector-contrib:latest
  config:
    receivers:
      prometheus:
        config:
          scrape_configs:
            - job_name: 'kepler-pods'
              kubernetes_sd_configs:
                - role: pod
              relabel_configs:
                - source_labels: [__meta_kubernetes_namespace]
                  action: keep
                  regex: kepler

                - source_labels: [__address__, __meta_kubernetes_pod_container_port_number]
                  action: replace
                  regex: ([^:]+):(?:\d+);(\d+)
                  replacement: $${1}:$${2}
                  target_label: __address__

                - source_labels: [__meta_kubernetes_node_name]
                  action: replace
                  target_label: instance
            - job_name: 'kubernetes-cadvisor-central'
              kubernetes_sd_configs:
                - role: node
              scheme: https
              authorization:
                credentials_file: /var/run/secrets/kubernetes.io/serviceaccount/token
              tls_config:
                ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
                insecure_skip_verify: true
              
              relabel_configs:
                - action: labelmap
                  regex: __meta_kubernetes_node_label_(.+)
                - target_label: __address__
                  replacement: kubernetes.default.svc:443
                - source_labels: [__meta_kubernetes_node_name]
                  regex: (.+)
                  target_label: __metrics_path__
                  replacement: /api/v1/nodes/$${1}/proxy/metrics/cadvisor

    processors:
      attributes:
        actions:
        - key: cluster
          value: c2
          action: insert
          
    exporters:
      otlp:
        # hub grpc receiver address.
        endpoint: "172.18.0.4:32333"
        tls:
          insecure: true

    service:
      pipelines:
        metrics:
          receivers: [prometheus]
          processors: [attributes]
          exporters: [otlp]
```
