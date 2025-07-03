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

This step we will install OpenTelemetry Operator on Hub cluster.

1. Install cert-manager: `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml`
2. Install OTel Operator: `kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml`

### Create OTel Collector

Now we can create the OpenTelemetryCollector.

`kubectl apply -f hub-collector.yaml`

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

Run the command on obs-c1 and obs-c2 respectively.

`kubectl apply -f kepler_deploy.yaml`

```yaml
# kepler-deploy.yaml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/warn: privileged
    security.openshift.io/scc.podSecurityLabelSync: "false"
    sustainable-computing.io/app: kepler
  name: kepler
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    sustainable-computing.io/app: kepler
  name: kepler-sa
  namespace: kepler
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    sustainable-computing.io/app: kepler
  name: prometheus-k8s
  namespace: kepler
rules:
- apiGroups:
  - ""
  resources:
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    sustainable-computing.io/app: kepler
  name: kepler-clusterrole
rules:
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  - nodes/proxy
  - nodes/stats
  - pods
  verbs:
  - get
  - watch
  - list
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    sustainable-computing.io/app: kepler
  name: prometheus-k8s
  namespace: kepler
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: prometheus-k8s
subjects:
- kind: ServiceAccount
  name: prometheus-k8s
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    sustainable-computing.io/app: kepler
  name: kepler-clusterrole-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kepler-clusterrole
subjects:
- kind: ServiceAccount
  name: kepler-sa
  namespace: kepler
---
apiVersion: v1
data:
  BIND_ADDRESS: 0.0.0.0:9102
  CPU_ARCH_OVERRIDE: ""
  ENABLE_EBPF_CGROUPID: "true"
  ENABLE_GPU: "true"
  ENABLE_PROCESS_METRICS: "false"
  ENABLE_QAT: "false"
  EXPOSE_CGROUP_METRICS: "false"
  EXPOSE_HW_COUNTER_METRICS: "true"
  EXPOSE_IRQ_COUNTER_METRICS: "true"
  KEPLER_LOG_LEVEL: "1"
  KEPLER_NAMESPACE: kepler
  METRIC_PATH: /metrics
  MODEL_CONFIG: |
    CONTAINER_COMPONENTS_ESTIMATOR=false
  PROMETHEUS_SCRAPE_INTERVAL: 30s
  REDFISH_PROBE_INTERVAL_IN_SECONDS: "60"
  REDFISH_SKIP_SSL_VERIFY: "true"
kind: ConfigMap
metadata:
  labels:
    sustainable-computing.io/app: kepler
  name: kepler-cfm
  namespace: kepler
---
apiVersion: v1
data:
  redfish.csv: |
    eW91cl9rdWJlbGV0X25vZGVfbmFtZSxyZWRmaXNoX3VzZXJuYW1lLHJlZGZpc2hfcGFzc3
    dvcmQsaHR0cHM6Ly9yZWRmaXNoX2lwX29yX2hvc3RuYW1lCg==
kind: Secret
metadata:
  labels:
    sustainable-computing.io/app: kepler
  name: redfish-4kh9d7bc7m
  namespace: kepler
type: Opaque
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kepler-exporter
    sustainable-computing.io/app: kepler
  name: kepler-exporter
  namespace: kepler
spec:
  clusterIP: None
  ports:
  - name: http
    port: 9102
    targetPort: http
  selector:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: kepler-exporter
    sustainable-computing.io/app: kepler
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    sustainable-computing.io/app: kepler
  name: kepler-exporter
  namespace: kepler
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: kepler-exporter
      sustainable-computing.io/app: kepler
  template:
    metadata:
      labels:
        app.kubernetes.io/component: exporter
        app.kubernetes.io/name: kepler-exporter
        sustainable-computing.io/app: kepler
    spec:
      containers:
      - args:
        - /usr/bin/kepler -v=1 -redfish-cred-file-path=/etc/redfish/redfish.csv
        command:
        - /bin/sh
        - -c
        env:
        - name: NODE_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        image: quay.io/sustainable_computing_io/kepler:latest
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 9102
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 60
          successThreshold: 1
          timeoutSeconds: 10
        name: kepler-exporter
        ports:
        - containerPort: 9102
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 400Mi
        securityContext:
          privileged: true
        volumeMounts:
        - mountPath: /lib/modules
          name: lib-modules
          readOnly: true
        - mountPath: /sys
          name: tracing
          readOnly: true
        - mountPath: /proc
          name: proc
        - mountPath: /var/run
          name: var-run
        - mountPath: /etc/kepler/kepler.config
          name: cfm
          readOnly: true
        - mountPath: /etc/redfish
          name: redfish
          readOnly: true
      dnsPolicy: ClusterFirstWithHostNet
      hostPID: true
      serviceAccountName: kepler-sa
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/control-plane
        operator: Exists
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Exists
      volumes:
      - hostPath:
          path: /lib/modules
          type: Directory
        name: lib-modules
      - hostPath:
          path: /sys
          type: Directory
        name: tracing
      - hostPath:
          path: /proc
          type: Directory
        name: proc
      - hostPath:
          path: /var/run
          type: Directory
        name: var-run
      - configMap:
          name: kepler-cfm
        name: cfm
      - name: redfish
        secret:
          secretName: redfish-4kh9d7bc7m
```

## Install OTel Operator on obs-c1, obs-c2

This step we will install OpenTelemetry Operator on obs-c1 and obs-c2 clusters.

1. Install cert-manager: `kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml`
2. Install OTel Operator: `kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml`

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
